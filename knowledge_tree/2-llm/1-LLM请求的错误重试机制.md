## OpenClaw：LLM 错误处理的相关文件与代码分析

OpenClaw 的 LLM 错误处理是一套**多层级、精细化的工业级体系**，涉及 6 个核心文件：

---

### 文件 1：`src/agents/pi-embedded-helpers/types.ts` — 错误类型定义

这是整个错误处理体系的**类型基石**，只有 12 行，但定义了所有下游模块共用的 `FailoverReason` 联合类型：

```typescript
export type FailoverReason =
  | "auth"            // 认证失败（401/403，可重试）
  | "auth_permanent"  // 永久性认证失败（API Key 被撤销，不可重试）
  | "format"          // 请求格式错误（400/422）
  | "rate_limit"      // 速率限制（429）
  | "overloaded"      // 服务过载（529）
  | "billing"         // 计费问题（402）
  | "timeout"         // 超时/服务不可用（408/500/502/503/504）
  | "model_not_found" // 模型不存在（404）
  | "session_expired" // 会话过期（410）
  | "unknown";        // 未知错误
```

**设计亮点**：将 `auth` 和 `auth_permanent` 分开，前者可以通过轮转 Profile 恢复，后者直接终止。这种区分决定了后续 failover 策略的走向。

---

### 文件 2：`src/agents/pi-embedded-helpers/failover-matches.ts` — 错误模式匹配库

这是一个**纯粹的错误文本模式匹配库**（约 200 行），不包含任何业务逻辑，只负责回答"这段错误文本属于哪种错误？"。

**核心数据结构** — `ERROR_PATTERNS` 对象，按错误类型组织了大量正则和字符串匹配模式：

```typescript
const ERROR_PATTERNS = {
  rateLimit: [
    /rate[_ ]limit|too many requests|429/,
    /throttling(?:exception)?/i,
    "exceeded your current quota",
    "resource_exhausted",
    // ... 共 16 个模式
  ],
  overloaded: [
    /overloaded_error|"type"\s*:\s*"overloaded_error"/i,
    "overloaded",
    "high demand",
    // 注意：service_unavailable 只有和 overload/capacity 同时出现才算 overloaded
  ],
  timeout: [
    "timeout", "timed out", "socket hang up",
    /\beconn(?:refused|reset|aborted)\b/i,  // Node.js 网络错误码
    /\bepipe\b/i, /\benotfound\b/i,
    /without sending (?:any )?chunks?/i,     // 流式响应中断
    // ... 共 30+ 个模式
  ],
  billing: [
    /["']?(?:status|code)["']?\s*[:=]\s*402\b/i,
    "insufficient credits", "insufficient balance",
    ZAI_BILLING_CODE_1311_RE,  // 智谱 AI 特定错误码
    // ... 共 12 个模式
  ],
  auth: [
    "incorrect api key", "unauthorized", "forbidden",
    /\b401\b/, /\b403\b/,
    ZAI_AUTH_CODE_1113_RE,  // 智谱 AI 特定错误码
    // ... 共 20+ 个模式
  ],
};
```

**关键导出函数**：

```typescript
// 每个函数都是对 ERROR_PATTERNS 某个分组的封装
export function isRateLimitErrorMessage(raw: string): boolean;
export function isTimeoutErrorMessage(raw: string): boolean;
export function isBillingErrorMessage(raw: string): boolean;
export function isAuthErrorMessage(raw: string): boolean;
export function isOverloadedErrorMessage(raw: string): boolean;
```

**设计亮点**：
- `isBillingErrorMessage()` 对超长错误文本（>512 字符）只检查硬性 402 模式，避免误匹配
- `isAuthPermanentErrorMessage()` 和 `isAuthErrorMessage()` 分开，前者只匹配高置信度模式（如 "key revoked"），后者匹配更宽泛的模式

---

### 文件 3：`src/agents/pi-embedded-helpers/errors.ts` — 错误分类引擎（核心）

这是整个体系的**大脑**（约 600 行），负责将原始错误信号（HTTP 状态码 + 错误消息 + Provider 信息）综合分类为 `FailoverReason`。

**核心入口函数** — `classifyFailoverSignal()`：

```typescript
export function classifyFailoverSignal(signal: FailoverSignal): FailoverClassification | null {
  const inferredStatus = inferSignalStatus(signal);
  // 1. 先从消息文本分类
  const messageClassification = signal.message
    ? classifyFailoverClassificationFromMessage(signal.message, signal.provider)
    : null;
  // 2. 再结合 HTTP 状态码分类（状态码优先级更高）
  const statusClassification = classifyFailoverClassificationFromHttpStatus(
    inferredStatus, signal.message, messageClassification,
  );
  if (statusClassification) return statusClassification;
  // 3. 尝试从错误码分类（如 RESOURCE_EXHAUSTED）
  const codeReason = classifyFailoverReasonFromCode(signal.code);
  if (codeReason) return toReasonClassification(codeReason);
  // 4. 最后回退到消息文本分类结果
  return messageClassification;
}
```

**HTTP 状态码分类逻辑** — `classifyFailoverClassificationFromHttpStatus()`：

这个函数的精妙之处在于**状态码和消息文本的交叉验证**：

```typescript
// HTTP 402 不是简单归为 billing
if (status === 402) {
  // Anthropic Claude Max 的 402 实际是 rate_limit
  // 如果消息包含 "usage limit, try again later" → rate_limit
  // 如果消息包含 "insufficient credits" → billing
  return toReasonClassification(message ? classify402Message(message) : "billing");
}

// HTTP 401/403 也不是简单归为 auth
if (status === 401 || status === 403) {
  if (isAuthPermanentErrorMessage(message)) return "auth_permanent";
  // OpenRouter 的 "Key limit exceeded" 虽然是 401/403，但实际是 billing
  if (messageReason === "billing") return "billing";
  return "auth";
}

// HTTP 400/422 是最模糊的
if (status === 400 || status === 422) {
  // 先检查消息文本，可能是 rate_limit/auth/billing/model_not_found
  if (messageClassification) return messageClassification;
  // 都不是才归为 format
  return "format";
}
```

**消息文本分类逻辑** — `classifyFailoverClassificationFromMessage()`：

这个函数有一个**严格的优先级链**（约 30 个 if 分支），顺序非常重要：

```
1. 排除非 LLM 错误（图片尺寸/大小错误）→ 返回 null
2. 会话过期 → session_expired
3. 模型不存在 → model_not_found
4. 上下文溢出 → context_overflow（特殊类型，不是 FailoverReason）
5. 402 文本信号 → billing 或 rate_limit
6. OpenRouter Key 限额 → billing
7. 周期性用量限制 → rate_limit 或 billing
8. 速率限制 → rate_limit
9. 服务过载 → overloaded
10. 瞬态 HTTP 错误 → timeout
11. 计费错误 → billing（必须在 auth 之前，避免 "insufficient balance" 被误判为 auth）
12. 永久性认证错误 → auth_permanent
13. 认证错误 → auth
14. Anthropic 未知错误 → timeout（Provider 特定）
15. OpenRouter Provider 返回错误 → timeout（Provider 特定）
16. 服务器错误 → timeout
17. JSON API 内部错误 → timeout（但排除 billing/auth 的 api_error）
18. 格式错误 → format
19. 超时文本 → timeout
20. Provider 特定模式 → 对应 reason
```

**另一个重要函数** — `formatAssistantErrorText()`：

将原始错误转换为**用户友好的提示**，按优先级匹配：

```typescript
// 工具不存在 → 检查是否是沙箱策略阻止
// 磁盘空间不足 → 专用提示
// OAuth 刷新失败 → "Re-authenticate this provider"
// 上下文溢出 → "Try /reset or use a larger-context model"
// 推理约束错误 → "Use /think minimal"
// 流式响应无效 → "Please try again"
// 消息角色顺序错误 → "Use /new to start a fresh session"
// 速率限制/过载 → 格式化的限流提示
// 传输层错误 → 格式化的网络错误提示
// 超时 → "LLM request timed out"
// 计费 → Provider 特定的计费错误提示
// 兜底 → 截断到 600 字符展示
```

---

### 文件 4：`src/agents/pi-embedded-helpers/provider-error-patterns.ts` — Provider 特定模式

这个文件处理**各家 Provider 独特的错误格式**（约 140 行）：

```typescript
// Provider 特定的上下文溢出模式
export const PROVIDER_CONTEXT_OVERFLOW_PATTERNS: readonly RegExp[] = [
  /\binput token count exceeds the maximum number of input tokens\b/i,  // AWS Bedrock
  /\binput exceeds the maximum number of tokens\b/i,                    // Google Vertex
  /\bollama error:\s*context length exceeded/i,                          // Ollama
  /\btotal tokens?.*exceeds? (?:the )?(?:model(?:'s)? )?(?:max|maximum|limit)/i, // Cohere
  /\b(?:request|prompt) \(\d[\d,]*\s*tokens?\) exceeds (?:the )?available context size\b/i, // llama.cpp
];

// Provider 特定的 failover 模式
export const PROVIDER_SPECIFIC_PATTERNS: readonly ProviderErrorPattern[] = [
  { test: /\bthrottlingexception\b/i, reason: "rate_limit" },           // AWS Bedrock
  { test: /\bconcurrency limit(?: has been)? reached\b/i, reason: "rate_limit" },
  { test: /\bmodelnotreadyexception\b/i, reason: "overloaded" },       // AWS Bedrock
  { test: /model(?:_is)?_deactivated/i, reason: "model_not_found" },   // Groq
];
```

**还支持 Provider Plugin Hook 扩展**：

```typescript
export function classifyProviderSpecificError(errorMessage: string): FailoverReason | null {
  // 1. 先尝试 Provider Plugin Hook（动态扩展）
  const pluginReason = runtimeHooks?.classifyProviderFailoverReasonWithPlugin({
    context: { errorMessage },
  });
  if (pluginReason) return pluginReason;
  // 2. 回退到静态模式匹配
  for (const pattern of PROVIDER_SPECIFIC_PATTERNS) {
    if (pattern.test.test(errorMessage)) return pattern.reason;
  }
  return null;
}
```

---

### 文件 5：`src/infra/retry.ts` — 通用重试引擎

这是一个**与业务无关的通用重试工具**（约 130 行），被 LLM 请求和其他 HTTP 请求共用。

**核心函数** — `retryAsync()`：

```typescript
export async function retryAsync<T>(
  fn: () => Promise<T>,
  attemptsOrOptions: number | RetryOptions = 3,
  initialDelayMs = 300,
): Promise<T> {
  // 支持两种调用方式：
  // 1. 简单模式：retryAsync(fn, 3, 300)
  // 2. 完整模式：retryAsync(fn, { attempts: 3, shouldRetry, retryAfterMs, onRetry })
```

**退避策略**：

```typescript
// 优先使用 Provider 返回的 retry_after 值
const retryAfterMs = options.retryAfterMs?.(err);
const hasRetryAfter = typeof retryAfterMs === "number" && Number.isFinite(retryAfterMs);

// 有 retry_after → 取 max(retryAfter, minDelay)
// 没有 → 指数退避 minDelay * 2^(attempt-1)
const baseDelay = hasRetryAfter
  ? Math.max(retryAfterMs, minDelayMs)
  : minDelayMs * 2 ** (attempt - 1);

// 加 Jitter 防雷群
delay = applyJitter(delay, jitter);
// Jitter 实现：delay * (1 + random(-jitter, +jitter))
```

---

### 文件 6：`src/agents/pi-embedded-runner/run/helpers.ts` — Failover 策略参数

定义了 **Profile 轮转和重试迭代的上限**：

```typescript
export const DEFAULT_OVERLOAD_FAILOVER_BACKOFF_MS = 0;
export const DEFAULT_MAX_OVERLOAD_PROFILE_ROTATIONS = 1;   // 过载时最多换 1 次 API Key
export const DEFAULT_MAX_RATE_LIMIT_PROFILE_ROTATIONS = 1; // 限流时最多换 1 次 API Key

const BASE_RUN_RETRY_ITERATIONS = 24;
const RUN_RETRY_ITERATIONS_PER_PROFILE = 8;
const MIN_RUN_RETRY_ITERATIONS = 32;
const MAX_RUN_RETRY_ITERATIONS = 160;

// 根据可用 Profile 数量动态计算最大重试次数
export function resolveMaxRunRetryIterations(profileCandidateCount: number): number {
  const scaled = BASE_RUN_RETRY_ITERATIONS +
    Math.max(1, profileCandidateCount) * RUN_RETRY_ITERATIONS_PER_PROFILE;
  return Math.min(MAX_RUN_RETRY_ITERATIONS, Math.max(MIN_RUN_RETRY_ITERATIONS, scaled));
}
// 1 个 Profile → max(32, 24+8) = 32 次
// 5 个 Profile → max(32, 24+40) = 64 次
// 20 个 Profile → min(160, 24+160) = 160 次
```

### 文件 7：`src/agents/pi-embedded-runner/run/failover-observation.ts` — Failover 可观测性

记录每次 failover 决策的**结构化日志**：

```typescript
export type FailoverDecisionLoggerInput = {
  stage: "prompt" | "assistant";                              // 错误发生在哪个阶段
  decision: "rotate_profile" | "fallback_model" | "surface_error"; // 做了什么决策
  failoverReason: FailoverReason | null;                      // 错误分类
  provider: string;
  model: string;
  profileId?: string;
  fallbackConfigured: boolean;                                // 是否配置了备用模型
  timedOut?: boolean;
  aborted?: boolean;
};
```

三种决策：
- **`rotate_profile`**：切换到下一个 Auth Profile（换 API Key）
- **`fallback_model`**：降级到备用模型
- **`surface_error`**：向用户展示错误信息（所有恢复手段用尽）

---

### OpenClaw 错误处理流程总结

```
LLM 请求失败
    │
    ▼
┌─ failover-matches.ts ──────────────────────────────┐
│  文本模式匹配：isRateLimitErrorMessage()           │
│                isBillingErrorMessage()              │
│                isTimeoutErrorMessage()              │
│                isOverloadedErrorMessage()           │
│                isAuthErrorMessage()                 │
└────────────────────────┬───────────────────────────┘
                         │
                         ▼
┌─ errors.ts ────────────────────────────────────────┐
│  综合分类：classifyFailoverSignal()                │
│    = HTTP 状态码 × 消息文本 × Provider 特定模式    │
│  输出：FailoverReason                              │
└────────────────────────┬───────────────────────────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
     rate_limit      timeout      billing
     overloaded      auth         auth_permanent
            │            │            │
            ▼            ▼            ▼
┌─ helpers.ts ───────────────────────────────────────┐
│  Failover 决策：                                   │
│  - rotate_profile（换 API Key，最多 1 次）         │
│  - fallback_model（降级备用模型）                  │
│  - surface_error（展示友好错误）                   │
│  重试上限：32~160 次（按 Profile 数量动态调整）    │
└────────────────────────┬───────────────────────────┘
                         │
                         ▼
┌─ retry.ts ─────────────────────────────────────────┐
│  通用重试：指数退避 + Jitter + retryAfter          │
│  默认 3 次，300ms 起步，30s 上限                   │
└────────────────────────┬───────────────────────────┘
                         │
                         ▼
┌─ failover-observation.ts ──────────────────────────┐
│  结构化日志：记录每次 failover 决策                │
│  stage / decision / reason / provider / model      │
└────────────────────────────────────────────────────┘
```

---

## DeerFlow：LLM 错误处理的相关文件与代码分析

DeerFlow 的错误处理分散在**中间件链**中，涉及 4 个中间件，核心文件是 `deer_flow/01-deer-flow中间件.md` 中记录的实现。

---

### 中间件 1：`ToolErrorHandlingMiddleware` — 工具错误捕获

**触发时机**：`wrap_tool_call()` 阶段（工具调用包装）

```python
def wrap_tool_call(self, request, handler):
    try:
        return handler(request)  # 执行工具
    except GraphBubbleUp:
        raise  # LangGraph 控制流信号，不拦截！
    except Exception as e:
        # 将异常转换为错误 ToolMessage
        return ToolMessage(
            content=f"Error executing tool: {str(e)}",
            tool_call_id=request.tool_call["id"],
            status="error"
        )
```

**代码分析**：
- **`GraphBubbleUp` 例外**：这是 LangGraph 的控制流信号（如 `Command(goto=END)`），不是真正的错误，必须向上传播
- **所有其他异常统一捕获**：不区分错误类型，统一转为 `status="error"` 的 `ToolMessage`
- **让 LLM 自主决策**：错误信息作为工具返回值反馈给 LLM，LLM 可以选择重试、换方法、或告知用户
- **保留 `tool_call_id`**：确保 LLM 能将错误与对应的工具调用关联

**与 OpenClaw 的关键差异**：DeerFlow 不做错误分类，不做自动恢复，完全依赖 LLM 的"智能"来处理错误。

---

### 中间件 2：`LoopDetectionMiddleware` — 循环检测

**触发时机**：`after_model()` 阶段（模型调用后）

```python
def _track_and_check(self, state, runtime):
    call_hash = _hash_tool_calls(tool_calls)  # MD5 哈希
    history.append(call_hash)                  # 滑动窗口（最近 20 次）
    
    count = history.count(call_hash)
    if count >= self.hard_limit:    # 5 次 → 强制停止
        return _HARD_STOP_MSG, True
    elif count >= self.warn_threshold:  # 3 次 → 注入警告
        return _WARNING_MSG, False
```

**代码分析**：
- **Hash 去重**：对 `tool_calls` 的 `name + args` 计算 MD5，相同的调用会产生相同的 hash
- **两级阈值**：3 次软警告（注入提示消息让 LLM 换策略），5 次硬停止（直接移除 `tool_calls`，强制文本回复）
- **线程安全**：使用 `threading.Lock`，支持 LRU 淘汰（最多 100 个线程的历史）
- **典型场景**：LLM 反复调用同一个失败的工具（如 `read_file` 读取不存在的文件），LoopDetection 会打破这个死循环

---

### 中间件 3：`DanglingToolCallMiddleware` — 悬垂工具调用修复

**触发时机**：`wrap_model_call()` 阶段（模型调用包装）

```python
def wrap_model_call(self, request, handler):
    for msg in messages:
        if msg.type == "ai" and msg.tool_calls:
            for tc in msg.tool_calls:
                if tc.id not in existing_tool_msg_ids:
                    # 插入占位消息
                    patched.append(ToolMessage(
                        content="[Tool call was interrupted...]",
                        tool_call_id=tc.id,
                        status="error"
                    ))
    return handler(request.override(messages=patched))
```

**代码分析**：
- **问题场景**：用户在 Agent 执行过程中取消请求，导致 AI 发出了 `tool_calls` 但没有收到对应的 `ToolMessage`
- **修复方式**：扫描消息历史，为每个没有对应 `ToolMessage` 的 `tool_call` 插入一个 `status="error"` 的占位消息
- **为什么需要**：大多数 LLM Provider（如 OpenAI、Anthropic）要求 `tool_calls` 和 `ToolMessage` 必须成对出现，否则会返回 400 错误

---

### 中间件 4：`SummarizationMiddleware` — 上下文溢出预防

**触发时机**：`before_model()` 阶段（模型调用前）

```python
# 配置示例
summarization:
  trigger:
    type: "token_count"
    threshold: 100000      # Token 数超过 10 万时触发
  keep:
    type: "last_n"
    count: 10              # 保留最近 10 条消息
  model_name: "gpt-4o-mini" # 用轻量模型生成摘要
```

**代码分析**：
- **主动预防**：不等 Provider 返回 Context Overflow 错误，而是在发送前主动检测并压缩
- **与 OpenClaw 的差异**：OpenClaw 是**事后检测**（收到错误后提示用户 `/reset`），DeerFlow 是**事前预防**（自动压缩）

---

### DeerFlow 错误处理流程总结

```
LLM 请求/工具调用
    │
    ▼
┌─ SummarizationMiddleware (before_model) ───────────┐
│  主动预防 Context Overflow                         │
│  Token 数超阈值 → 自动压缩历史消息                 │
└────────────────────────┬───────────────────────────┘
                         │
                         ▼
┌─ DanglingToolCallMiddleware (wrap_model_call) ─────┐
│  修复中断导致的消息不完整                          │
│  缺失 ToolMessage → 插入占位错误消息               │
└────────────────────────┬───────────────────────────┘
                         │
                         ▼
                   LLM 模型调用
                         │
                         ▼
┌─ LoopDetectionMiddleware (after_model) ────────────┐
│  检测重复工具调用循环                              │
│  3 次 → 软警告 / 5 次 → 硬停止                    │
└────────────────────────┬───────────────────────────┘
                         │
                         ▼
                    工具执行
                         │
                         ▼
┌─ ToolErrorHandlingMiddleware (wrap_tool_call) ─────┐
│  捕获所有工具异常                                  │
│  异常 → ToolMessage(status="error")                │
│  反馈给 LLM 自主决策                               │
└────────────────────────────────────────────────────┘
```

---

### 核心差异总结

| 维度 | **OpenClaw** | **DeerFlow** |
|------|-------------|-------------|
| **错误分类** | 10 种 FailoverReason + 20+ Provider 特定模式，**系统自动分类** | 不分类，**统一捕获** |
| **恢复策略** | Profile 轮转 → 模型降级 → 用户提示，**系统自动恢复** | 错误反馈给 LLM，**LLM 自主决策** |
| **Context Overflow** | 事后检测，提示用户 `/reset` | 事前预防，`SummarizationMiddleware` 自动压缩 |
| **重试** | 自研 `retryAsync()`，指数退避 + Jitter + retryAfter | 依赖 LangChain `BaseChatModel` 内置 `max_retries` |
| **循环检测** | 工具级 loop detection config | `LoopDetectionMiddleware`，hash + 滑动窗口 |
| **代码量** | ~1200 行（6 个文件） | ~100 行（4 个中间件的错误处理部分） |
| **适用场景** | 多租户生产环境、多 Provider 切换 | 快速原型、单用户场景 |

简单来说：**OpenClaw 是"系统替你做决策"，DeerFlow 是"让 LLM 替你做决策"**。前者更可靠但更复杂，后者更简洁但依赖 LLM 的推理能力。

需要我进一步深入某个具体模块的实现细节吗？现在我已经收集了所有关键代码文件的内容。下面为你详细阐述两个项目处理 LLM 错误的相关文件和代码分析。

