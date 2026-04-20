

## Hermes Agent 为什么能"自进化"？—— Self-Evolving Loop 深度解析

Hermes Agent 的自进化能力不是一个单一机制，而是由 **四个闭环系统** 协同构成的。我从源码中逐一拆解：


### 🔄 闭环一：Memory Nudge — 记忆自主沉淀

**源码位置**：`run_agent.py` L1136-1146, L7868-7878

Agent 内部维护了一个 **turn 计数器** `_turns_since_memory`，每隔 N 轮对话（默认 10 轮，可配置 `nudge_interval`），系统会自动触发一次 **记忆审查**：

```
用户对话 → 计数器累加 → 达到阈值 → 触发 _should_review_memory = True
```

审查时，Agent 会 fork 出一个后台 `AIAgent`（`_spawn_background_review`，L2169），用这段 prompt 驱动：

> *"Review the conversation above... Has the user revealed things about themselves — their persona, desires, preferences, or personal details worth remembering?"*

关键设计：
- **后台线程执行**，不阻塞用户交互
- 直接写入 `MEMORY.md`（Agent 自身笔记）和 `USER.md`（用户画像）
- 写入前有 **注入检测**（`_scan_memory_content`），防止 prompt injection 污染记忆

**效果**：Agent 越用越懂你，会自动记住你的偏好、工作习惯、纠正过的行为模式。

---

### 🔄 闭环二：Skill Auto-Creation — 技能自主生成

**源码位置**：`run_agent.py` L1235-1239, L8121-8124, L10588-10611

这是最核心的自进化机制。Agent 维护另一个计数器 `_iters_since_skill`，追踪 **工具调用迭代次数**。当一次对话中工具调用次数达到阈值（默认 10 次），系统判定这是一个"复杂任务"，触发技能审查：

```
复杂任务完成 → 工具调用计数达标 → 触发 _should_review_skills = True
→ 后台 fork Agent → 审查对话 → 自动创建/更新 Skill
```

审查 prompt（`_SKILL_REVIEW_PROMPT`，L2145）：

> *"Was a non-trivial approach used to complete a task that required trial and error, or changing course due to experiential findings along the way?"*

通过 `skill_manage` 工具（`skill_manager_tool.py`），Agent 可以：
- **`create`**：从零创建新 Skill（SKILL.md + 目录结构）
- **`patch`**：针对性修改已有 Skill（find-and-replace）
- **`edit`**：整体重写 Skill
- **`write_file`**：添加参考文档、脚本等辅助文件

**效果**：Agent 完成一个复杂任务后，会自动把"怎么做"沉淀成可复用的 Skill 文件。下次遇到类似任务，直接加载 Skill 而不是重新摸索。

---

### 🔄 闭环三：Session Search — 跨会话经验回溯

**源码位置**：`tools/session_search_tool.py`

Agent 的所有历史对话都存储在 SQLite 中，通过 **FTS5 全文索引** 支持跨会话搜索。当 Agent 遇到新问题时，可以主动搜索过去的对话：

```
新问题 → FTS5 检索历史会话 → 取 Top-N 匹配 → LLM 摘要 → 注入当前上下文
```

关键设计：
- 不是把原始对话塞进上下文，而是用 **Gemini Flash 做摘要**，只提取相关信息
- 截断策略（`_truncate_around_matches`）：围绕匹配位置取窗口，最大 100K 字符

**效果**：Agent 能"回忆"过去的经验，避免重复犯错。

---

### 🔄 闭环四：GEPA 进化优化 — 离线技能进化

**源码位置**：独立仓库 `hermes-agent-self-evolution`

这是更高层次的进化——用 **DSPy + GEPA（Genetic-Pareto Prompt Evolution）** 对 Skill 文件做遗传进化优化：

```
读取当前 Skill → 生成评测数据集 → GEPA 变异候选 → 执行评估 → 约束门控 → 最优变体 → PR 提交
```

GEPA 的核心能力是 **读取执行轨迹（traces）来理解"为什么失败"**，然后做针对性改进，而不是盲目变异。

约束门控包括：
- 全量测试必须通过
- Skill 大小 ≤ 15KB
- 语义不能偏离原始目的
- 所有变更必须经过人工 PR review

**效果**：Skill 文件本身会越来越好，不需要人工手动优化 prompt。

---

### 🧠 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    Hermes Self-Evolving Loop                │
│                                                             │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │ 用户对话  │───▶│  Agent Loop   │───▶│  任务完成/响应   │  │
│  └──────────┘    └──────┬───────┘    └────────┬─────────┘  │
│                         │                      │            │
│              ┌──────────▼──────────┐           │            │
│              │  Turn/Iter 计数器    │           │            │
│              │  memory_nudge: 10轮  │           │            │
│              │  skill_nudge: 10次   │           │            │
│              └──────────┬──────────┘           │            │
│                         │ 达到阈值              │            │
│              ┌──────────▼──────────┐           │            │
│              │ _spawn_background_  │◀──────────┘            │
│              │     review()        │  后台线程               │
│              └────┬─────────┬──────┘                        │
│                   │         │                               │
│          ┌────────▼──┐  ┌───▼────────┐                     │
│          │ Memory写入 │  │ Skill创建/  │                     │
│          │ MEMORY.md  │  │ 更新/patch  │                     │
│          │ USER.md    │  │ SKILL.md    │                     │
│          └────────┬──┘  └───┬────────┘                     │
│                   │         │                               │
│          ┌────────▼─────────▼──────┐                       │
│          │  下次会话 System Prompt  │◀── Session Search     │
│          │  注入更新后的记忆+技能    │    (FTS5 跨会话召回)   │
│          └─────────────────────────┘                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  离线进化层 (hermes-agent-self-evolution)            │   │
│  │  DSPy + GEPA → 变异 Skill → 评估 → 最优 → PR       │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

### 💡 为什么这个设计很巧妙？

| 设计点 | 传统 Agent | Hermes Agent |
|--------|-----------|-------------|
| **记忆** | 手动管理或无 | 自动 nudge 沉淀，有注入防护 |
| **技能** | 人工编写 prompt | Agent 自己从经验中提炼 Skill，还能 patch 改进 |
| **跨会话** | 每次从零开始 | FTS5 + LLM 摘要回溯历史经验 |
| **进化** | 人工调优 prompt | GEPA 遗传算法自动进化 Skill 文本 |
| **安全** | 无 | 记忆写入有注入扫描，Skill 创建有安全审计 |

**核心洞察**：Hermes 把 Agent 的"学习"拆成了 **在线（runtime）** 和 **离线（evolution）** 两层。在线层通过 nudge 机制让 Agent 在使用中自动积累经验；离线层通过 GEPA 对积累的 Skill 做系统性优化。这形成了一个真正的 **闭环自进化系统**。

---

需要我进一步深入某个具体模块的实现细节吗？比如 GEPA 的进化算法、Memory 的冲突消解策略、或者 Skill 的安全扫描机制？ 🤓