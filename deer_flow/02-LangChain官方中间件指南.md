是的，`before_agent()`、`before_model()`、`after_model()` 等是 LangChain 官方定义的 Agent 中间件 Hook 点。让我为您查找官方文档。
我已经找到了官方文档。让我为您总结 LangChain Agent Middleware 的官方 Hook 点。

---

## LangChain Agent Middleware 官方 Hook 点

是的，`before_agent()`、`before_model()`、`after_model()` 等都是 **LangChain 官方定义的 Agent 中间件 Hook 点**。

### 官方文档地址

| 文档类型 | 链接 |
|---------|------|
| **自定义中间件指南** | https://docs.langchain.com/oss/python/langchain/middleware/custom |
| **AgentMiddleware API 参考** | https://reference.langchain.com/python/langchain/agents/middleware/types/AgentMiddleware |
| **博客介绍** | https://blog.langchain.com/agent-middleware/ |

---

### 官方 Hook 点分类

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        LangChain Agent Middleware Hooks                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  【Node-style hooks】- 在特定执行点顺序运行                                         │
│  ┌────────────────────┬──────────────────────────────────────────────────────┐  │
│  │ Hook               │ 触发时机                                              │  │
│  ├────────────────────┼──────────────────────────────────────────────────────┤  │
│  │ before_agent()     │ Agent 启动前（每次调用执行一次）                        │  │
│  │ before_model()     │ 每次调用 LLM 模型前                                   │  │
│  │ after_model()      │ 每次 LLM 模型返回后                                   │  │
│  │ after_agent()      │ Agent 执行完成后（每次调用执行一次）                    │  │
│  └────────────────────┴──────────────────────────────────────────────────────┘  │
│                                                                                 │
│  【Wrap-style hooks】- 包装模型/工具调用                                            │
│  ┌────────────────────┬──────────────────────────────────────────────────────┐  │
│  │ Hook               │ 作用                                                  │  │
│  ├────────────────────┼──────────────────────────────────────────────────────┤  │
│  │ wrap_model_call()  │ 包装每次模型调用（可用于重试、缓存、转换）               │  │
│  │ wrap_tool_call()   │ 包装每次工具调用（可用于错误处理、拦截）                │  │
│  └────────────────────┴──────────────────────────────────────────────────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### 两种使用方式

#### 1. 装饰器方式（简单场景）

```python
from langchain.agents.middleware import before_model, after_model, AgentState
from langchain.messages import AIMessage
from langgraph.runtime import Runtime
from typing import Any

@before_model(can_jump_to=["end"])
def check_message_limit(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    if len(state["messages"]) >= 50:
        return {
            "messages": [AIMessage("Conversation limit reached.")],
            "jump_to": "end"
        }
    return None

@after_model
def log_response(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    print(f"Model returned: {state['messages'][-1].content}")
    return None
```

#### 2. 类方式（复杂场景，DeerFlow 使用的方式）

```python
from langchain.agents.middleware import AgentMiddleware, AgentState, hook_config
from langchain.messages import AIMessage
from langgraph.runtime import Runtime
from typing import Any

class MessageLimitMiddleware(AgentMiddleware):
    def __init__(self, max_messages: int = 50):
        super().__init__()
        self.max_messages = max_messages

    @hook_config(can_jump_to=["end"])
    def before_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        if len(state["messages"]) >= self.max_messages:
            return {
                "messages": [AIMessage("Conversation limit reached.")],
                "jump_to": "end"
            }
        return None

    def after_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        print(f"Model returned: {state['messages'][-1].content}")
        return None
```

---

### DeerFlow 中间件与官方 Hook 的对应关系

| DeerFlow 中间件 | 使用的官方 Hook | 作用 |
|----------------|----------------|------|
| `ThreadDataMiddleware` | `before_agent()` | 初始化线程目录 |
| `UploadsMiddleware` | `before_agent()` | 注入上传文件信息 |
| `SandboxMiddleware` | `before_agent()` | 获取沙箱环境 |
| `DanglingToolCallMiddleware` | `wrap_model_call()` | 修复悬垂工具调用 |
| `DeferredToolFilterMiddleware` | `wrap_model_call()` | 过滤延迟工具 |
| `SummarizationMiddleware` | `before_model()` | 上下文压缩（LangChain内置） |
| `ViewImageMiddleware` | `before_model()` | 注入图像数据 |
| `TodoMiddleware` | `before_model()` | 任务列表提醒 |
| `TitleMiddleware` | `after_model()` | 生成对话标题 |
| `SubagentLimitMiddleware` | `after_model()` | 限制子Agent并发 |
| `LoopDetectionMiddleware` | `after_model()` | 检测循环调用 |
| `ToolErrorHandlingMiddleware` | `wrap_tool_call()` | 工具错误处理 |
| `ClarificationMiddleware` | `wrap_tool_call()` | 拦截澄清请求 |
| `MemoryMiddleware` | `after_agent()` | 队列记忆更新 |

---

### 关键概念：State 更新机制

根据官方文档：

- **Node-style hooks** (`before_agent`, `before_model`, `after_model`, `after_agent`): 直接返回 `dict`，通过 graph 的 reducers 应用到 agent state

- **Wrap-style hooks** (`wrap_model_call`, `wrap_tool_call`): 
  - 模型调用：返回 `ExtendedModelResponse` 包含 `Command` 来注入状态更新
  - 工具调用：直接返回 `Command`

```python
# Node-style 返回 dict
def before_model(self, state, runtime) -> dict | None:
    return {"messages": [HumanMessage(content="...")]}

# Wrap-style 模型调用返回 ExtendedModelResponse
def wrap_model_call(self, request, handler):
    response = handler(request)
    return ExtendedModelResponse(
        model_response=response,
        command=Command(update={"key": "value"})
    )

# Wrap-style 工具调用返回 Command
def wrap_tool_call(self, request, handler):
    return Command(update={"messages": [ToolMessage(...)]})
```

---

### 总结

DeerFlow 的中间件系统完全基于 **LangChain 官方 Agent Middleware 架构**，所有 Hook 点都是官方标准接口。这种设计的好处：

1. **标准化**：遵循 LangChain 生态规范
2. **可组合**：多个中间件可以链式组合
3. **灵活**：支持装饰器和类两种方式
4. **强大**：可以拦截和控制 Agent 执行的各个阶段





---------------------------------------------------------------------------

根据 LangChain 官方文档，让我为您详细解释 `node-style` 和 `wrap-style` 的区别：

---

## Node-style vs Wrap-style 对比

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              核心区别一览                                               │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   特性              Node-style                    Wrap-style                            │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   执行方式          在特定执行点顺序运行              包装/包裹目标调用                   │
│   控制粒度          在固定点干预                      完全控制目标调用过程                │
│   调用次数          固定执行一次                      可决定调用 0次/1次/多次             │
│   适用场景          日志、验证、状态更新              重试、缓存、错误处理、拦截            │
│   返回值            dict 状态更新                     ModelResponse/Command/ToolMessage   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 一、Node-style Hooks

### 特点
- **在特定执行点运行**：在 Agent 执行流程的预定义节点上运行
- **顺序执行**：多个中间件按顺序链式执行
- **单次执行**：每个 hook 在该执行点只执行一次

### 四个官方 Hook 点

```
Agent 执行流程：
─────────────────────────────────────────────────────────►

  │                    │                    │              │
  ▼                    ▼                    ▼              ▼
┌─────────┐      ┌─────────┐        ┌─────────┐      ┌─────────┐
│before_  │      │before_  │        │after_   │      │after_   │
│agent()  │ ──▶  │model()  │  ──▶   │model()  │ ──▶  │agent()  │
│         │      │  LLM    │        │         │      │         │
│初始化    │      │调用前    │        │调用后    │      │清理     │
└─────────┘      └─────────┘        └─────────┘      └─────────┘

    ◄────────────── 每次模型调用循环 ──────────────►
```

### 代码示例

```python
from langchain.agents.middleware import AgentMiddleware, AgentState
from langgraph.runtime import Runtime
from typing import Any

class MyNodeMiddleware(AgentMiddleware):
    
    def before_agent(self, state: AgentState, runtime: Runtime) -> dict | None:
        """Agent 启动前执行一次"""
        print("Agent 开始执行")
        return {"some_flag": True}  # 返回状态更新
    
    def before_model(self, state: AgentState, runtime: Runtime) -> dict | None:
        """每次调用 LLM 前执行"""
        print(f"准备调用模型，当前 {len(state['messages'])} 条消息")
        return None  # 无需更新状态
    
    def after_model(self, state: AgentState, runtime: Runtime) -> dict | None:
        """每次 LLM 返回后执行"""
        last_msg = state["messages"][-1]
        print(f"模型返回: {last_msg.content[:100]}...")
        return {"model_call_count": state.get("call_count", 0) + 1}
    
    def after_agent(self, state: AgentState, runtime: Runtime) -> dict | None:
        """Agent 执行完成后执行一次"""
        print("Agent 执行完成")
        return None
```

---

## 二、Wrap-style Hooks

### 特点
- **包装目标调用**：包裹模型调用或工具调用
- **完全控制**：可以决定是否调用、调用几次、如何转换结果
- **拦截能力**：可以短路（不调用）、重试（多次调用）

### 两个官方 Hook

```
┌─────────────────────────────────────────────────────────────────┐
│                    wrap_model_call()                            │
│                                                                 │
│   你的代码        handler(request)         你的代码              │
│      │                 │                      │                 │
│      ▼                 ▼                      ▼                 │
│   ┌─────┐         ┌─────────┐            ┌─────┐               │
│   │准备  │  ──▶   │实际LLM  │   ──▶      │处理  │               │
│   │工作  │         │调用     │            │结果  │               │
│   └─────┘         └─────────┘            └─────┘               │
│                                                                 │
│   可做：重试、缓存、修改请求、记录日志、转换响应                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    wrap_tool_call()                             │
│                                                                 │
│   你的代码        handler(request)         你的代码              │
│      │                 │                      │                 │
│      ▼                 ▼                      ▼                 │
│   ┌─────┐         ┌─────────┐            ┌─────┐               │
│   │准备  │  ──▶   │实际工具 │   ──▶      │处理  │               │
│   │工作  │         │执行     │            │结果  │               │
│   └─────┘         └─────────┘            └─────┘               │
│                                                                 │
│   可做：错误处理、拦截、权限检查、结果转换                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 代码示例：重试逻辑

```python
from langchain.agents.middleware import (
    AgentMiddleware, 
    ModelRequest, 
    ModelResponse,
    ToolCallRequest,
    ToolMessage,
    Command
)
from typing import Callable

class RetryMiddleware(AgentMiddleware):
    """Node-style 无法实现的重试逻辑"""
    
    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """包装模型调用，实现重试"""
        for attempt in range(3):
            try:
                return handler(request)  # 调用实际模型
            except Exception as e:
                if attempt == 2:
                    raise  # 最后一次失败，抛出异常
                print(f"重试 {attempt + 1}/3: {e}")

class ErrorHandlingMiddleware(AgentMiddleware):
    """工具错误处理"""
    
    def wrap_tool_call(
        self,
        request: ToolCallRequest,
        handler: Callable[[ToolCallRequest], ToolMessage]
    ) -> ToolMessage | Command:
        """包装工具调用，捕获错误"""
        try:
            return handler(request)  # 调用实际工具
        except Exception as e:
            # 返回错误消息而不是抛出异常
            return ToolMessage(
                content=f"工具执行失败: {str(e)}",
                tool_call_id=request.tool_call.get("id"),
                name=request.tool_call.get("name"),
                status="error"
            )
```

---

## 三、关键区别对比

### 1. 控制能力

| 场景 | Node-style | Wrap-style |
|------|-----------|------------|
| 只想在特定点记录日志 | ✅ `after_model` | ❌ 过度设计 |
| 需要重试失败的调用 | ❌ 无法控制 | ✅ `wrap_model_call` |
| 需要缓存模型响应 | ❌ 无法拦截 | ✅ `wrap_model_call` |
| 工具出错时返回友好错误 | ❌ 无法捕获 | ✅ `wrap_tool_call` |
| 拦截特定工具调用 | ❌ 无法阻止 | ✅ `wrap_tool_call` |

### 2. 返回值差异

```python
# Node-style: 直接返回 dict，自动合并到 state
def before_model(self, state, runtime) -> dict | None:
    return {"messages": [HumanMessage(content="...")]}

# Wrap-style 模型: 返回 ModelResponse 或 ExtendedModelResponse
def wrap_model_call(self, request, handler):
    response = handler(request)
    return response  # 必须返回 ModelResponse

# Wrap-style 工具: 返回 ToolMessage 或 Command
def wrap_tool_call(self, request, handler):
    return ToolMessage(content="...", tool_call_id="...")  # 或 Command
```

### 3. 执行流程差异

```
Node-style 执行流程：
─────────────────────────────────────────────────►
     │              │              │
     ▼              ▼              ▼
  Middleware1    Middleware2    Middleware3
  (before_model) (before_model) (before_model)
     │              │              │
     └──────────────┴──────────────┘
                    │
                    ▼
                实际调用 LLM


Wrap-style 执行流程：
─────────────────────────────────────────────────►
                    │
                    ▼
            ┌───────────────┐
            │  Middleware1  │  ◄── 最外层
            │  (wrap_call)  │
            │  ┌─────────┐  │
            │  │Middleware2│ │  ◄── 中间层
            │  │(wrap_call)│ │
            │  │ ┌─────┐ │ │
            │  │ │实际 │ │ │
            │  │ │调用 │ │ │  ◄── 最内层
            │  │ └─────┘ │ │
            │  └─────────┘ │
            └───────────────┘
                    │
                    ▼
                返回结果
```

---

## 四、DeerFlow 中的实际应用

| 中间件 | 类型 | 为什么选这个类型 |
|--------|------|----------------|
| `ThreadDataMiddleware` | Node (`before_agent`) | 只需在开始时初始化一次 |
| `TitleMiddleware` | Node (`after_model`) | 在模型返回后生成标题 |
| `LoopDetectionMiddleware` | Node (`after_model`) | 检查模型响应，无需拦截 |
| `DanglingToolCallMiddleware` | Wrap (`wrap_model_call`) | 需要修改请求消息列表 |
| `ToolErrorHandlingMiddleware` | Wrap (`wrap_tool_call`) | 需要捕获工具异常 |
| `ClarificationMiddleware` | Wrap (`wrap_tool_call`) | 需要拦截并阻止工具执行 |

---

## 五、简单记忆口诀

```
Node-style = "在固定时间点做某事"
           → 初始化、记录、检查、更新状态

Wrap-style = "包装控制某个调用"
           → 重试、缓存、拦截、错误处理、转换
```

选择原则：**能用 Node-style 解决的不要用 Wrap-style**，Wrap-style 更强大但也更复杂。