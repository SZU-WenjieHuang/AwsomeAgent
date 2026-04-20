
## 1.1 React 架构（Reasoning + Acting）

**React** 不是指前端框架 React，而是 **Re**asoning（推理）+ **Act**ing（行动）的缩写。

### 核心思想

```
┌─────────────────────────────────────────┐
│           React 循环架构                 │
├─────────────────────────────────────────┤
│  Thought（思考）→ Action（行动）→ Observation（观察） │
│       ↑                                    │
│       └────────────────────────────────────┘
└─────────────────────────────────────────┘
```

### 工作流程

| 步骤 | 说明 | 示例 |
|------|------|------|
| **Thought** | 分析当前状态，制定计划 | "用户要查天气，我需要调用天气API" |
| **Action** | 执行具体动作（调用工具） | 调用 `get_weather("北京")` |
| **Observation** | 观察执行结果 | 获取到天气数据：晴天，25°C |
| **循环** | 基于观察继续思考下一步 | 整理数据，回复用户 |

ReAct 是目前 Agent 开发中最主流的架构模式，LangChain、AutoGPT 等项目都采用了这种设计思想。



## 1.2 Context Engine 架构（上下文压缩引擎）

**代表框架：Hermes**

这是 Hermes 最核心的创新，也是面试里很少被提到的。

**核心思路**：把"上下文管理"从 prompt 工程里抽出来，做成一个**可插拔的引擎层**。

```python
# context_engine.py — 抽象基类
class ContextEngine(ABC):
    last_prompt_tokens: int = 0
    threshold_percent: float = 0.75
    protect_first_n: int = 3   # 保护头部消息（system prompt）
    protect_last_n: int = 6    # 保护尾部消息（最近上下文）

    @abstractmethod
    def should_compress(self, prompt_tokens: int = None) -> bool:
        """判断是否需要压缩"""

    @abstractmethod
    def compress(self, messages, current_tokens) -> List[Dict]:
        """执行压缩，返回新的消息列表"""
```

**ContextCompressor 的具体算法**（`context_compressor.py`）：

```
1. 先做工具输出剪枝（cheap pre-pass，不调 LLM）
2. 保护头部 N 条消息（system prompt + 首轮）
3. 按 token 预算保护尾部（最近 ~20K tokens）
4. 用辅助 LLM 对中间段做结构化摘要
5. 多次压缩时，迭代更新之前的摘要（不是重新生成）
```

**摘要前缀的设计细节**：

```python
SUMMARY_PREFIX = (
    "[CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted "
    "into the summary below. This is a handoff from a previous context "
    "window — treat it as background reference, NOT as active instructions. "
    "Do NOT answer questions or fulfill requests mentioned in this summary..."
)
```

> **工程洞察**：这个前缀解决了一个真实问题——模型看到历史摘要后会"重新执行"里面的任务。加了 `NOT as active instructions` 之后，模型才能正确把摘要当背景信息而非待执行指令。

---

## 1.3 Smart Model Routing（智能模型路由）

**代表框架：Hermes**

不是所有请求都需要最强的模型。Hermes 实现了一个**基于规则的轻量路由层**：

```python
# smart_model_routing.py
_COMPLEX_KEYWORDS = {
    "debug", "implement", "refactor", "analyze",
    "architecture", "docker", "kubernetes", "subagent", ...
}

def choose_cheap_model_route(user_message, routing_config):
    # 消息超过 160 字符 → 用主模型
    if len(text) > max_chars: return None
    # 包含代码块 → 用主模型
    if "```" in text or "`" in text: return None
    # 包含复杂关键词 → 用主模型
    if words & _COMPLEX_KEYWORDS: return None
    # 否则路由到 cheap_model
    return cheap_model_route
```

**这是一个 Cost-Aware Agent 架构**，核心逻辑是：

```
简单问候/查询 → cheap model（省钱）
复杂推理/工具调用 → primary model（保质量）
```

---

## 1.4 Mixture of Agents（MoA 多模型聚合）

**代表框架：Hermes**（`mixture_of_agents_tool.py`）

基于论文 [arXiv:2406.04692](https://arxiv.org/abs/2406.04692)，Hermes 把 MoA 做成了一个**工具**，让 Agent 自己决定什么时候用多模型聚合：

```python
# 参考模型并行生成
REFERENCE_MODELS = [
    "anthropic/claude-opus-4.6",
    "google/gemini-3-pro-preview",
    "openai/gpt-5.4-pro",
    "deepseek/deepseek-v3.2",
]
AGGREGATOR_MODEL = "anthropic/claude-opus-4.6"

# 聚合 prompt（来自论文）
AGGREGATOR_SYSTEM_PROMPT = """You have been provided with a set of responses 
from various open-source models... synthesize these responses into a single, 
high-quality response... recognizing that some of it may be biased or incorrect."""
```

**架构流程**：

```
用户问题
    ↓
4个参考模型并行生成（temperature=0.6，鼓励多样性）
    ↓
聚合模型合并（temperature=0.4，聚焦一致性）
    ↓
最终高质量答案
```

> **关键设计**：参考模型用高 temperature 保证多样性，聚合模型用低 temperature 保证稳定性。这不是简单投票，而是让最强模型做"裁判"。

---

## 1.5 Supervisor + SubAgent 架构（主从任务委派）

**代表框架：DeerFlow**

DeerFlow 实现了一套完整的**主 Agent 委派子 Agent** 的架构，核心在 `task_tool.py` 和 `executor.py`：

```python
# task_tool.py — 主 Agent 通过 task 工具委派任务
@tool("task", parse_docstring=True)
async def task_tool(
    description: str,    # 任务描述（3-5词）
    prompt: str,         # 详细任务说明
    subagent_type: str,  # general-purpose | bash
    max_turns: int = None,
) -> str:
    # 防止递归嵌套：子 Agent 不能再创建子 Agent
    tools = get_available_tools(subagent_enabled=False)
    executor = SubagentExecutor(config=config, tools=tools, ...)
```

**SubagentExecutor 的并发设计**（`executor.py`）：

```python
# 三个独立线程池，职责分离
_scheduler_pool = ThreadPoolExecutor(max_workers=3)   # 调度
_execution_pool = ThreadPoolExecutor(max_workers=3)   # 执行
_isolated_loop_pool = ThreadPoolExecutor(max_workers=3)  # 隔离事件循环

@dataclass
class SubagentResult:
    task_id: str
    trace_id: str          # 分布式追踪，链接父子 Agent 日志
    status: SubagentStatus # PENDING/RUNNING/COMPLETED/FAILED/CANCELLED/TIMED_OUT
    ai_messages: list      # 完整执行轨迹
    cancel_event: threading.Event  # 支持取消
```

**子 Agent 的配置隔离**（`general_purpose.py`）：

```python
GENERAL_PURPOSE_CONFIG = SubagentConfig(
    disallowed_tools=["task", "ask_clarification", "present_files"],
    # 禁止子 Agent 再委派任务 → 防止无限递归
    model="inherit",   # 继承父 Agent 的模型
    max_turns=100,
)
```

---

## 1.6 Guardrail Middleware 架构（护栏中间件）

**代表框架：DeerFlow + Hermes**

这是生产级 Agent 最重要但最少被讨论的架构层。

**DeerFlow 的 GuardrailMiddleware**（`guardrails/middleware.py`）：

```python
class GuardrailMiddleware(AgentMiddleware[AgentState]):
    """在工具执行前拦截，评估是否允许执行"""

    def wrap_tool_call(self, request, handler):
        gr = self._build_request(request)
        decision = self.provider.evaluate(gr)  # 同步评估
        
        if not decision.allow:
            # 返回错误 ToolMessage，让 Agent 自适应
            return self._build_denied_message(request, decision)
        return handler(request)  # 放行
```

**Hermes 的危险命令检测**（`approval.py`）：

```python
DANGEROUS_PATTERNS = [
    (r'\brm\s+-[^\s]*r', "recursive delete"),
    (r'\bDROP\s+(TABLE|DATABASE)\b', "SQL DROP"),
    (r':\(\)\s*\{\s*:\s*\|\s*:\s*&\s*\}\s*;\s*:', "fork bomb"),
    (r'\b(curl|wget)\b.*\|\s*(ba)?sh\b', "pipe remote content to shell"),
    # 防止 Agent 自杀
    (r'\b(pkill|killall)\b.*\b(hermes|gateway)\b', "kill hermes process"),
    ...
]
```

**两个框架的护栏设计对比**：

| 维度 | DeerFlow | Hermes |
|---|---|---|
| **拦截层** | 工具调用前中间件 | 命令执行前正则检测 |
| **失败策略** | `fail_closed`（默认拒绝） | 人工审批 / 自动审批 |
| **反馈方式** | 返回错误 ToolMessage | 阻断 + 提示 |
| **智能化** | 规则引擎 | 辅助 LLM 判断低风险命令 |

---

## 1.7 Event-Driven DAG 架构（事件驱动有向无环图）

**代表框架：LangGraph**

LangGraph 的底层是 **Pregel 计算模型**（来自 Google 的图计算论文），这是四个框架里最底层、最通用的架构。

**核心抽象**（`graph/state.py`）：

```python
class StateGraph(Generic[StateT, ContextT, InputT, OutputT]):
    """节点通过读写共享状态通信
    每个节点签名：State -> Partial<State>
    每个 state key 可以注册 reducer 函数
    """
```

**Checkpoint 机制**（`pregel/_checkpoint.py`）：

```python
def create_checkpoint(checkpoint, channels, step, *, id=None) -> Checkpoint:
    """每一步都创建快照，支持：
    - 时间旅行（回放任意历史状态）
    - 断点续跑（从任意 checkpoint 恢复）
    - Human-in-the-loop（在任意节点暂停等待人工确认）
    """
    return Checkpoint(
        v=LATEST_VERSION,
        ts=datetime.now(timezone.utc).isoformat(),
        id=id or str(uuid6(clock_seq=step)),
        channel_values=values,
        channel_versions=checkpoint["channel_versions"],
    )
```

**Channel 系统**（支持多种数据流模式）：

```
LastValue    → 最新值覆盖（普通状态）
BinaryOperatorAggregate → 聚合（如消息列表 append）
NamedBarrierValue → 屏障同步（等所有并行节点完成）
Topic        → 发布订阅
EphemeralValue → 临时值（不持久化）
```

**`_algo.py` 里的并行调度**：

```python
# Pregel 的核心：每一轮找出所有可以并行执行的节点
# 依赖关系通过 channel 版本号追踪
# 没有依赖的节点 → 并行执行
# 有依赖的节点 → 等待上游 channel 更新
```

---

### 三、架构对比总结

```
┌─────────────────────────────────────────────────────────────────┐
│                    四框架架构定位对比                              │
├──────────────┬──────────────────────────────────────────────────┤
│   Hermes     │ 单 Agent + 强上下文工程                            │
│              │ ContextEngine（可插拔压缩）                        │
│              │ Smart Routing（成本感知）                          │
│              │ MoA（多模型聚合工具）                              │
│              │ SQLite 状态持久化 + FTS5 全文搜索                  │
├──────────────┼──────────────────────────────────────────────────┤
│   DeerFlow   │ Supervisor + SubAgent（主从委派）                  │
│              │ Guardrail Middleware（护栏层）                     │
│              │ 基于 LangGraph Pregel 的 DAG 执行                  │
│              │ 任务隔离 + 并发线程池                              │
├──────────────┼──────────────────────────────────────────────────┤
│  LangGraph   │ 通用 DAG 框架（基础设施层）                        │
│              │ Pregel 计算模型                                    │
│              │ Checkpoint 时间旅行                                │
│              │ Channel 系统（多种数据流模式）                     │
│              │ Human-in-the-loop 原生支持                        │
├──────────────┼──────────────────────────────────────────────────┤
│  OpenClaw    │ 多模态 + 分布式设备协作                            │
│              │ Plugin 扩展系统                                    │
│              │ 安全沙箱（exec allowlist）                         │
│              │ 多 Channel 接入（Telegram/Discord/Slack）          │
└──────────────┴──────────────────────────────────────────────────┘
```

