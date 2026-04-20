基于我们之前深入分析 OpenClaw 源码的对话，来做一个完整的总结。

---

## 什么是 Harness？

### 核心定义

**Harness 是 Agent 的可插拔运行时执行引擎**——它封装了"一次完整对话尝试"所需的一切：从组装 system prompt、注册工具、管理 session，到驱动 LLM 对话循环、执行工具调用、处理流式响应。

名字来自"马具"的隐喻：**给不同的"马"（模型厂商/执行引擎）套上统一的"缰绳"（接口）**，让上层代码用同一套方式驾驭所有执行引擎。

### Harness 的统一接口（来自 `types.ts`）

```typescript
type AgentHarness = {
  id: string;         // 唯一标识，如 "pi"、"codex"
  label: string;      // 显示名
  supports(ctx): AgentHarnessSupport;   // "我能不能处理这个 provider/model？"
  runAttempt(params): Promise<Result>;  // "执行一次完整对话"
  compact?(params): Promise<Result>;    // 压缩上下文（可选）
  reset?(params): void;                 // 重置会话（可选）
  dispose?(): void;                     // 销毁资源（可选）
};
```

上层的 `run.ts` 只调 `harness.runAttempt()`，完全不关心底层是谁。

---

### Harness 管理什么？（`runAttempt` 的完整职责）

```
一次 runAttempt() 的生命周期：

1. 沙箱 & 工作区     → 创建目录、解析沙箱上下文、获取写锁
2. Skills 加载       → 读取 Skill 定义，生成注入 system prompt 的文本
3. Bootstrap 上下文  → 读取工作区文件，分析 token 预算
4. 工具注册          → 内置工具 + MCP 工具 + LSP 工具，适配不同厂商格式
5. System Prompt 组装 → Skills + 工具描述 + 沙箱信息 + 运行时信息 → 完整 prompt
6. Session 管理      → 打开/修复会话文件，创建 Agent Session
7. Stream 函数包装   → 7 层洋葱皮（thinking 清理、工具 ID 修复、超时检测等）
8. 对话执行          → session.prompt() → LLM 响应 → tool_call → 执行 → 回传 → 循环
9. 后处理            → 上下文引擎维护、usage 统计、释放锁
```

---

### Harness 不管什么？（由外层 `run.ts` 负责）

| 职责 | 负责方 |
|---|---|
| 模型解析、认证 | `run.ts` |
| 重试循环（while true） | `run.ts` |
| 超时/溢出后的 compact 决策 | `run.ts` |
| 认证轮换（auth profile rotation） | `run.ts` |
| 模型降级 fallback（FailoverError） | `run.ts` |
| Harness 选择与降级 | `selection.ts` |

> **类比**：Harness 是"打一局比赛"，`run.ts` 是"决定要不要换人重打"。

---

## 三个具体例子

### 例子 1：PI Harness（内置通用引擎）

```typescript
// builtin-pi.ts
{
  id: "pi",
  supports: () => ({ supported: true, priority: 0 }),  // 什么都能跑，优先级最低
  runAttempt: runEmbeddedAttempt,  // 指向 attempt.ts 那个 1000+ 行的大函数
}
```

- **执行模式**：客户端直接调模型 API，自己管 session、自己组装 prompt、自己处理 stream
- **适用场景**：所有通用场景，是兜底方案
- **特点**：通过 7 种 stream-wrappers 适配不同厂商（OpenAI、Anthropic、Google、Moonshot、MiniMax、Bedrock、OpenRouter...）

---

### 例子 2：Codex Harness（远程执行引擎）

```typescript
// extensions/codex/harness.ts
{
  id: "codex",
  supports: (ctx) => ctx.provider === "codex"
    ? { supported: true, priority: 100 }   // 优先级 100，远高于 PI
    : { supported: false },
  runAttempt: runCodexAppServerAttempt,    // 完全不同的执行函数！
  compact: maybeCompactCodexAppServerSession,
  reset: clearCodexAppServerBinding,
  dispose: clearSharedCodexAppServerClient,
}
```

- **执行模式**：把消息交给 Codex App Server（独立后端进程），服务端自己调模型、管上下文；工具调用时服务端通过 JSON-RPC 回调客户端执行
- **适用场景**：模型调用和对话循环在远端服务器上的场景
- **和 PI 的本质区别**：PI 是"我自己调 API"，Codex 是"我把消息交给服务端，服务端帮我调"

---

### 例子 3：Anthropic Stream Wrapper（PI 内部的厂商适配）

这不是一个独立的 Harness，而是 PI Harness 内部针对 Anthropic 的特殊处理：

```typescript
// extensions/anthropic/stream-wrappers.ts
// 针对 Anthropic 的特殊处理：
// - 注入 beta header（fine-grained-tool-streaming、interleaved-thinking）
// - 1M context 模型（claude-opus-4、claude-sonnet-4）自动加 context-1m beta
// - OAuth token 自动切换 beta 集合
// - service tier 控制（auto / standard_only）
```

---

## 整体架构关系

```
用户消息
  ↓
run.ts（重试循环 + 模型解析 + 认证管理）
  ↓
selection.ts（Harness 选择）
  ├── provider = "codex" → Codex Harness（priority=100）
  │     └── runCodexAppServerAttempt()  ← 远端服务器执行
  │
  └── provider = 其他 → PI Harness（priority=0，兜底）
        └── runEmbeddedAttempt()  ← 本地执行
              ├── 工具注册（内置 + MCP + LSP）
              ├── System Prompt 组装
              ├── Stream 函数包装（按厂商选不同 wrapper）
              │     ├── anthropic/stream-wrappers.ts
              │     ├── google-stream-wrappers.ts
              │     ├── openai-stream-wrappers.ts
              │     ├── moonshot-stream-wrappers.ts
              │     └── ...
              └── session.prompt() → LLM 对话循环
```

---

### 一句话总结

> **Harness = Agent 的运行时操作系统**。它抽象了"如何跟模型对话"这件事，让上层代码只调一个 `runAttempt()`，不管底层是直接调 API（PI）、还是交给远端服务器（Codex）、还是走完全不同的协议——Harness 自己搞定一切。