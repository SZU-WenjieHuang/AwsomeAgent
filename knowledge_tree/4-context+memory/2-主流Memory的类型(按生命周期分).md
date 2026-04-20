好的，基于之前对 DeerFlow、Hermes、LangChain、LangGraph、OpenClaw 以及 MemGPT/Letta 源码的深度分析，我来按照 **Memory 的生命周期** 维度，结构化地整理主流 Agent 的 Memory 类型体系。

---

## 主流 Agent Memory 类型 — 按生命周期分层

---

### 总览：Memory 生命周期金字塔

```
┌──────────────────────────────────────────────────────────────┐
│                    ⑤ 组织/全局级 Memory                      │
│         生命周期：永久 │ 跨用户、跨 Agent 共享                 │
├──────────────────────────────────────────────────────────────┤
│                    ④ 用户级 Memory                           │
│         生命周期：长期 │ 跨会话持久化                          │
├──────────────────────────────────────────────────────────────┤
│                    ③ Agent 级 Memory                         │
│         生命周期：长期 │ Agent 自身的知识和经验                 │
├──────────────────────────────────────────────────────────────┤
│                    ② 会话级 Memory                           │
│         生命周期：单次会话 │ 会话结束即可丢弃                   │
├──────────────────────────────────────────────────────────────┤
│                    ① 轮次级 Memory (Working Memory)          │
│         生命周期：单轮推理 │ 推理完成即消失                     │
└──────────────────────────────────────────────────────────────┘
```

---

### ① 轮次级 Memory（Working Memory / In-Context）

**生命周期：** 单轮推理，推理完成即消失

**存储介质：** 上下文窗口

**本质：** 当前上下文窗口中的全部 token，是模型"此刻能看到的一切"

| 子类型 | 说明 | 主流做法 |
|--------|------|---------|
| **System Prompt** | 角色定义、规则、约束 | 所有框架都有，固定注入 |
| **Memory 注入块** | 从长期记忆中检索后注入的内容 | DeerFlow 的 `format_memory_for_injection()`，Hermes 的 `_system_prompt_snapshot`，Letta 的 Core Memory Block |
| **当前轮用户输入** | 用户本轮的消息 | 直接作为 user message |
| **工具调用结果** | 本轮工具返回的 observation | 作为 tool/function message 注入 |
| **思考链** | 模型的 reasoning/thought | ReAct 模式下的 thought 部分 |

**各框架实现对比：**

| 框架 | 实现方式 |
|------|---------|
| **LangChain** | `load_memory_variables()` 返回的 dict 注入到 prompt 模板变量中 |
| **LangGraph** | 图节点的 State 就是 working memory，每个节点读写 channel |
| **DeerFlow** | `format_memory_for_injection()` 将长期记忆格式化后注入 `<memory>` 标签 |
| **Hermes** | `format_for_system_prompt()` 返回冻结快照，会话内不变 |
| **Letta/MemGPT** | Core Memory Block 始终 pin 在上下文窗口中，Agent 可通过 function call 自行修改 |

**关键设计决策：**
- **Token 预算控制**：DeerFlow 用 tiktoken 精确计数，按置信度排序逐条加入直到预算耗尽
- **Prefix Cache 保护**：Hermes 的"冻结快照"模式——会话内 system prompt 不变，保护 KV cache 性能
- **主动 vs 被动**：Letta 让 Agent 自己管理 Core Memory（主动），其他框架多为系统自动注入（被动）

---

### ② 会话级 Memory（Session / Conversation Memory）

**生命周期：** 单次会话（从用户开始对话到会话结束）

**存储介质：** 进程内存， checkpoints会落盘到数据库 SQLite / PostgreSQL

**本质：** 维持多轮对话的连贯性，会话结束后可选择性保留或丢弃

| 子类型 | 说明 | 存储方式 |
|--------|------|---------|
| **消息缓冲区** | 最近 N 轮的原始消息 | 内存列表 |
| **滑动窗口** | 只保留最近 k 轮，更早的丢弃 | 内存列表 + 截断 |
| **递增摘要** | 每轮用 LLM 将新消息与已有摘要合并 | 内存字符串 |
| **摘要+缓冲混合** | 最近 k 轮保留原文，更早的压缩成摘要 | 内存字符串 + 列表 |
| **Token 预算缓冲** | 按 token 数而非轮次截断 | 内存列表 + token 计数 |
| **任务状态** | 当前任务的规划、进度、中间变量 | 结构化 state |

**各框架实现对比：**

| 框架 | 类/机制 | 策略 |
|------|---------|------|
| **LangChain** | `ConversationBufferMemory` | 全量存储，最简单 |
| **LangChain** | `ConversationBufferWindowMemory` | 滑动窗口，保留最近 k 轮 |
| **LangChain** | `ConversationSummaryMemory` | 每轮调 LLM 做递增摘要 |
| **LangChain** | `ConversationSummaryBufferMemory` | 摘要 + 缓冲区混合 |
| **LangChain** | `ConversationTokenBufferMemory` | 按 token 数截断 |
| **LangGraph** | `Checkpoint` | 图执行的状态快照，支持时间旅行和回放 |
| **DeerFlow** | `MemoryUpdateQueue` | 会话结束后异步提取有价值信息写入长期记忆 |
| **Hermes** | `session_search_tool` | 会话转录存盘，可跨会话搜索历史 |
| **Letta/MemGPT** | Recall Memory（温存储） | 完整对话历史自动持久化，可搜索检索 |
| **OpenClaw** | `ConversationStore` | 仅存储会话路由信息（非语义记忆） |

**关键设计决策：**

```
                    全量存储
                   ┌────────┐
                   │ 简单    │ ← LangChain Buffer
                   │ 但贵    │
                   └────┬───┘
                        │ 对话变长
                        ▼
              ┌─────────────────┐
              │ 滑动窗口 / Token │ ← LangChain Window/Token
              │ 截断旧消息       │
              └────────┬────────┘
                       │ 丢失重要历史
                       ▼
              ┌─────────────────┐
              │ 摘要 + 缓冲区    │ ← LangChain SummaryBuffer
              │ 旧的压缩，新的保留│    Letta 的消息驱逐+递归摘要
              └────────┬────────┘
                       │ 需要更精细控制
                       ▼
              ┌─────────────────┐
              │ 状态快照 + 检索   │ ← LangGraph Checkpoint
              │ 完整状态可回溯    │    Letta Recall Memory
              └─────────────────┘
```

**Letta 的消息驱逐策略（源自 MemGPT 论文）：**
- 上下文窗口满时，驱逐约 70% 的消息
- 被驱逐的消息经过**递归摘要**——与之前的摘要合并
- 越老的消息在摘要中的影响力越小
- 原始消息存入 Recall Memory，可通过搜索找回

---

### ③ Agent 级 Memory（Agent Self-Knowledge）

**生命周期：** 长期持久化，跨会话存在，属于 Agent 自身的知识

**存储介质：** 本地文件（md/json），数据库 (SQLite + FTS5) 支持全文检索， 向量数据库（支持语义检索）

**本质：** Agent 在工作中积累的经验、发现的规律、学到的教训

| 子类型 | 说明 | 典型内容 |
|--------|------|---------|
| **环境知识** | Agent 发现的运行环境事实 | "这台机器用的是 macOS"、"项目用 Python 3.11" |
| **工具经验** | 工具使用中的发现 | "这个 API 超时阈值是 30s"、"用 jq 解析比 python 快" |
| **纠正记录** | 从错误中学到的教训 | "不要用 rm -rf，用 trash 命令"（含 sourceError） |
| **项目约定** | 项目特有的规范 | "这个仓库的测试用 pytest，不用 unittest" |
| **技能/流程** | 稳定的操作流程 | "部署流程：先 build → 再 test → 最后 deploy" |

**各框架实现对比：**

| 框架 | 存储位置 | 机制 |
|------|---------|------|
| **Hermes** | `MEMORY.md` 文件 | Agent 通过 `memory` tool 主动写入，§ 分隔条目，字符数限制 2200 |
| **DeerFlow** | `memory.json` 的 `facts[]` 数组 | LLM 自动提取，按 category 分类（knowledge/behavior/correction），置信度排序 |
| **Letta** | Core Memory 的 Agent Persona Block | Agent 通过 function call 自行编辑，始终在上下文中 |
| **LangChain** | 无内置支持 | 需开发者自行实现 |
| **LangGraph** | `Store` 的 Agent 命名空间 | `store.put(("agents", "agent_1"), "knowledge", {...})` |

**Hermes 的 Agent Memory 写入规则（源码中的 schema description）：**

```
WHEN TO SAVE (proactively):
- 发现环境信息（OS、工具、项目结构）
- 学到约定、API 特性、工作流
- 识别到稳定的、未来有用的事实

DO NOT SAVE:
- 任务进度、会话结果、临时 TODO
- 琐碎/显而易见的信息
- 原始数据转储
```

**DeerFlow 的纠正检测机制（源码 `updater.py`）：**

```python
# 当检测到用户纠正 Agent 时
if correction_detected:
    correction_hint = (
        "Pay special attention to what the agent got wrong, "
        "record the correct approach as a fact with category "
        '"correction" and confidence >= 0.95'
    )
# 注入到 prompt 中的 fact 会带上 sourceError：
# - [correction | 0.95] 用 trash 命令删除文件 (avoid: 之前用了 rm -rf)
```

---

### ④ 用户级 Memory（User Profile / Preferences）

**生命周期：** 长期持久化，跨会话、跨任务存在，绑定到特定用户

**存储介质：** 本地文件（md/json），数据库 (SQLite + FTS5) 支持全文检索， 向量数据库（支持语义检索）

**本质：** Agent 对用户的认知——偏好、身份、习惯、沟通风格

| 子类型 | 说明 | 典型内容 |
|--------|------|---------|
| **身份信息** | 用户的基本属性 | 姓名、角色、公司、时区 |
| **偏好** | 用户的喜好和习惯 | "喜欢用 Python"、"偏好简洁回复" |
| **沟通风格** | 交互方式偏好 | "双语能力"、"喜欢代码示例" |
| **工作上下文** | 当前工作状态 | 正在做的项目、使用的技术栈 |
| **当前关注** | 近期焦点 | 3-5 个并行关注的主题 |
| **历史轨迹** | 时间线上的活动 | 近期/早期/长期背景 |

**各框架实现对比：**

| 框架 | 存储结构 | 更新机制 |
|------|---------|---------|
| **DeerFlow** | `memory.json` 的 `user` 区域（workContext / personalContext / topOfMind）+ `history` 区域（recentMonths / earlierContext / longTermBackground） | **LLM 自动提取**：每次对话后异步调用 LLM 分析对话，提取用户信息更新到对应字段 |
| **Hermes** | `USER.md` 文件 | **Agent 主动写入**：通过 `memory` tool 的 `target="user"` 操作，字符数限制 1375 |
| **Hermes/Honcho** | Honcho 服务端 | **辩证查询**：通过 `dialectic_query` 合成用户表征，支持 user representation + peer card |
| **Hermes/Mem0** | Mem0 Platform API | **服务端 LLM 提取**：`sync_turn()` 将对话发送到 Mem0，服务端自动提取事实 |
| **LangChain** | `ConversationEntityMemory` | **LLM 实体提取**：每轮用 LLM 提取实体并更新实体摘要 |
| **Letta** | Core Memory 的 User Block | **Agent 自主管理**：Agent 通过 function call 编辑 User Block 内容 |
| **LangGraph** | `Store` 的用户命名空间 | `store.put(("users", "user_123"), "profile", {...})` |

**DeerFlow 的用户记忆结构（源码 `storage.py`）：**

```json
{
  "user": {
    "workContext":      { "summary": "核心贡献者，16k+ stars 项目...", "updatedAt": "..." },
    "personalContext":  { "summary": "双语能力，对 AI Agent 感兴趣...", "updatedAt": "..." },
    "topOfMind":        { "summary": "正在研究 DeerFlow 架构，同时跟踪 MCP 协议发展...", "updatedAt": "..." }
  },
  "history": {
    "recentMonths":       { "summary": "最近 1-3 个月的详细活动...", "updatedAt": "..." },
    "earlierContext":     { "summary": "3-12 个月前的重要模式...", "updatedAt": "..." },
    "longTermBackground": { "summary": "核心专长、长期兴趣...", "updatedAt": "..." }
  }
}
```

**关键设计差异：**

| 维度 | 被动提取（DeerFlow/Mem0） | 主动写入（Hermes/Letta） |
|------|------------------------|------------------------|
| **触发方式** | 对话结束后自动分析 | Agent 在对话中主动调用 tool |
| **提取质量** | 依赖 LLM 理解能力 | 依赖 Agent 判断什么值得记 |
| **成本** | 每次对话额外一次 LLM 调用 | 无额外成本（复用对话中的 tool call） |
| **实时性** | 异步延迟（防抖队列） | 即时写盘 |
| **覆盖面** | 可能遗漏隐含信息 | 可能遗漏 Agent 没注意到的信息 |

---

### ⑤ 组织/全局级 Memory（Shared / Organizational Knowledge）

**生命周期：** 永久，跨用户、跨 Agent 共享

**存储介质：** 本地文件（md/json），数据库 (SQLite + FTS5) 支持全文检索， 向量数据库（支持语义检索）

**本质：** 组织级别的知识资产，所有 Agent 和用户共享

| 子类型 | 说明 | 典型内容 |
|--------|------|---------|
| **知识库** | 结构化/非结构化文档 | 产品文档、API 文档、FAQ |
| **向量索引** | 语义检索的 embedding 库 | RAG 的向量数据库 |
| **知识图谱** | 实体关系网络 | 组织架构、数据血缘、概念关系 |
| **元数据目录** | 治理过的数据资产 | 表定义、字段含义、数据所有者 |

**各框架实现对比：**

| 框架 | 实现方式 |
|------|---------|
| **LangChain** | `VectorStoreRetrieverMemory` — 向量检索相关历史；`ConversationKGMemory` — 知识图谱 |
| **LangGraph** | `Store` 的全局命名空间 + 向量搜索 |
| **Hermes/Holographic** | SQLite + FTS5 + HRR 向量符号架构，支持组合推理和矛盾检测 |
| **DeerFlow** | 计划中的 TF-IDF 上下文感知检索（尚未合并） |
| **Letta** | Archival Memory（冷存储）— 长期向量存储，支持 vector DB 和 graph DB |

---

