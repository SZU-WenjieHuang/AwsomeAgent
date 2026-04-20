好的，直接输出！

---

## Hermes Agent 四个自进化闭环 —— 源码级深度分析

---

### 🔄 闭环一：Memory Nudge — 记忆自主沉淀

#### 核心代码流程

**第一步：初始化计数器**（`run_agent.py` L1136-1146）

```python
self._memory_nudge_interval = 10        # 每 10 轮对话触发一次
self._memory_flush_min_turns = 6        # 至少 6 轮才允许 flush
self._turns_since_memory = 0            # 当前已累积轮数
```

这两个参数都可以通过 `config.yaml` 的 `memory.nudge_interval` 和 `memory.flush_min_turns` 配置。

**第二步：每轮对话累加计数**（`run_agent.py` L7862-7878）

```python
self._user_turn_count += 1
self._turns_since_memory += 1

if self._turns_since_memory >= self._memory_nudge_interval:
    _should_review_memory = True
    self._turns_since_memory = 0  # 重置计数器
```

关键点：**计数器在 `memory` 工具被主动调用时也会重置**（L7003）：
```python
if function_name == "memory":
    self._turns_since_memory = 0
```
这意味着如果用户主动让 Agent 存记忆，nudge 计时器会重新开始，避免重复审查。

**第三步：后台 fork Agent 执行审查**（`run_agent.py` L2169-2260）

```python
def _spawn_background_review(self, messages_snapshot, review_memory, review_skills):
    def _run_review():
        review_agent = AIAgent(
            model=self.model,
            max_iterations=8,        # 最多 8 步，防止失控
            quiet_mode=True,         # 静默执行
        )
        review_agent._memory_store = self._memory_store      # 共享记忆存储
        review_agent._memory_nudge_interval = 0              # 禁止嵌套 nudge
        review_agent._skill_nudge_interval = 0               # 禁止嵌套 nudge

        review_agent.run_conversation(
            user_message=prompt,                              # 审查 prompt
            conversation_history=messages_snapshot,           # 传入完整对话快照
        )
    t = threading.Thread(target=_run_review, daemon=True)
    t.start()
```

**第四步：记忆写入的安全防护**（`memory_tool.py` L55-85）

写入前会扫描内容是否包含注入攻击：

```python
_MEMORY_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'you\s+are\s+now\s+', "role_hijack"),
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET|PASSWORD)', "exfil_curl"),
    (r'authorized_keys', "ssh_backdoor"),
    # ...
]
```

还会检测不可见 Unicode 字符（零宽空格等），防止隐式注入。

**第五步：双文件存储架构**（`memory_tool.py` L100-130）

```python
class MemoryStore:
    def __init__(self, memory_char_limit=2200, user_char_limit=1375):
        self.memory_entries: List[str] = []    # MEMORY.md — Agent 自身笔记
        self.user_entries: List[str] = []      # USER.md — 用户画像
```

两个文件有**独立的字符上限**，写满后必须先 replace/remove 旧条目才能添加新的。这是一个**有界记忆**设计，强制 Agent 做信息筛选而不是无脑堆积。

**第六步：Frozen Snapshot 模式**

```python
# 系统 prompt 中注入的是 load 时的冻结快照
self._system_prompt_snapshot = {"memory": "...", "user": "..."}
# 中途写入只更新磁盘文件，不改变当前会话的 system prompt
# 下次会话启动时才会加载新内容
```

这个设计保护了 **Anthropic 的 prefix cache**——如果中途改 system prompt，缓存就失效了，成本会飙升。

#### 举例

> **场景**：你和 Hermes 聊了 10 轮，期间提到"我更喜欢用 TypeScript 而不是 JavaScript"、"部署一般用 Docker"。
>
> **触发**：第 10 轮结束后，`_turns_since_memory` 达到 10，触发后台审查。
>
> **后台 Agent 行为**：fork 出的 review agent 读完整对话，调用 `memory(action="add", target="user", content="Prefers TypeScript over JavaScript")`，写入 `USER.md`。
>
> **下次会话**：你开新对话时，`load_from_disk()` 加载 `USER.md`，system prompt 里就有了"用户偏好 TypeScript"，Agent 默认用 TS 写代码。

---

### 🔄 闭环二：Skill Auto-Creation — 技能自主生成

#### 核心代码流程

**第一步：追踪工具调用迭代次数**（`run_agent.py` L8121-8124）

```python
# 在 Agent Loop 的每次迭代中
if (self._skill_nudge_interval > 0
        and "skill_manage" in self.valid_tool_names):
    self._iters_since_skill += 1
```

注意：这里计数的是 **工具调用迭代次数**（每次 API 返回 tool_calls 算一次），而不是用户对话轮数。这意味着只有"复杂任务"（需要多次工具调用）才会触发。

**第二步：任务完成后检查阈值**（`run_agent.py` L10588-10591）

```python
if (self._skill_nudge_interval > 0
        and self._iters_since_skill >= self._skill_nudge_interval
        and "skill_manage" in self.valid_tool_names):
    _should_review_skills = True
    self._iters_since_skill = 0
```

**第三步：审查 Prompt 的精妙设计**（`run_agent.py` L2145-2153）

```python
_SKILL_REVIEW_PROMPT = (
    "Review the conversation above and consider saving or updating a skill "
    "if appropriate.\n\n"
    "Focus on: was a non-trivial approach used to complete a task that required "
    "trial and error, or changing course due to experiential findings along the "
    "way, or did the user expect or desire a different method or outcome?\n\n"
    "If a relevant skill already exists, update it with what you learned. "
    "Otherwise, create a new skill if the approach is reusable.\n"
    "If nothing is worth saving, just say 'Nothing to save.' and stop."
)
```

这个 prompt 有三个关键判断条件：
1. **试错过程**：是否经历了 trial and error
2. **路线变更**：是否因为实际发现而改变了方法
3. **用户纠正**：用户是否期望不同的方法或结果

这三个条件精准地筛选出"值得沉淀"的经验，而不是把每个任务都存成 skill。

**第四步：Skill 创建的完整流程**（`skill_manager_tool.py` L300-345）

```python
def _create_skill(name, content, category=None):
    # 1. 校验名称（小写字母+数字+连字符，≤64字符）
    err = _validate_name(name)
    # 2. 校验 YAML frontmatter（必须有 name + description）
    err = _validate_frontmatter(content)
    # 3. 校验大小（≤100K 字符）
    err = _validate_content_size(content)
    # 4. 检查名称冲突
    existing = _find_skill(name)
    # 5. 创建目录 + 原子写入 SKILL.md
    _atomic_write_text(skill_md, content)
    # 6. 安全扫描 — 不通过则回滚删除
    scan_error = _security_scan_skill(skill_dir)
    if scan_error:
        shutil.rmtree(skill_dir, ignore_errors=True)
```

**第五步：Skill 的 patch 更新**（`skill_manager_tool.py` L382-465）

当已有 Skill 需要改进时，Agent 不需要重写整个文件，而是用 `patch` 做精准修改：

```python
def _patch_skill(name, old_string, new_string, file_path=None):
    # 使用 fuzzy_match 引擎做模糊匹配
    from tools.fuzzy_match import fuzzy_find_and_replace
    new_content, match_count, _strategy, match_error = fuzzy_find_and_replace(
        content, old_string, new_string, replace_all
    )
    # 修改后重新校验 frontmatter
    err = _validate_frontmatter(new_content)
    # 原子写入 + 安全扫描
    _atomic_write_text(target, new_content)
    scan_error = _security_scan_skill(skill_dir)
    if scan_error:
        _atomic_write_text(target, original_content)  # 回滚！
```

**第六步：Skill 创建后立即刷新缓存**（`skill_manager_tool.py` L645-649）

```python
if result.get("success"):
    from agent.prompt_builder import clear_skills_system_prompt_cache
    clear_skills_system_prompt_cache(clear_snapshot=True)
```

这确保新创建的 Skill 立即出现在 skills_list 中，下次对话就能被发现和加载。

#### 举例

> **场景**：你让 Hermes 帮你在一个新项目里配置 ESLint + Prettier + Husky。Agent 先试了一种配置方式，发现和项目的 monorepo 结构冲突，改了方案，又遇到 Prettier 和 ESLint 规则冲突，最终找到了正确的配置顺序。整个过程用了 15 次工具调用。
>
> **触发**：`_iters_since_skill` 达到 10（默认阈值），任务完成后触发 skill 审查。
>
> **后台 Agent 行为**：review agent 分析对话，发现这是一个"经历了试错和路线变更"的复杂任务，调用：
> ```
> skill_manage(action="create", name="eslint-prettier-monorepo",
>   content="---\nname: eslint-prettier-monorepo\ndescription: Configure ESLint + Prettier in monorepo...\n---\n## Procedure\n1. Install in root...\n2. Use flat config...\n## Pitfalls\n- Prettier must run AFTER ESLint...")
> ```
>
> **下次使用**：你在另一个 monorepo 项目里说"帮我配 linting"，Agent 通过 `skills_list()` 发现 `eslint-prettier-monorepo` skill，加载后直接按正确路径执行，跳过之前的试错。

---

### 🔄 闭环三：Session Search — 跨会话经验回溯

#### 核心代码流程

**第一步：FTS5 全文检索**（`session_search_tool.py` L297-380）

```python
def session_search(query, role_filter=None, limit=3, db=None, current_session_id=None):
    # FTS5 搜索，取 50 条匹配
    raw_results = db.search_messages(
        query=query,
        role_filter=role_list,
        limit=50,
    )
    # 按 session 分组，去重，排除当前会话
    # 解析 delegation 链，找到根 session
    resolved_sid = _resolve_to_parent(raw_sid)
```

关键设计：**排除当前会话**（Agent 已经有这个上下文了）和**排除子 Agent 会话**（delegation 产生的子会话归并到父会话）。

**第二步：智能截断**（`session_search_tool.py` L100-170）

```python
def _truncate_around_matches(full_text, query, max_chars=100_000):
    # 策略1：完整短语匹配
    phrase_pat = re.compile(re.escape(query_lower))
    # 策略2：所有关键词在 200 字符内共现
    # 策略3：单个关键词位置（兜底）
    # 选择覆盖最多匹配位置的窗口
    # 偏向 25% 前文 + 75% 后文
    ws = max(0, candidate - max_chars // 4)
```

这个截断算法非常精巧——它不是简单地取前 N 个字符，而是**找到匹配密度最高的区域**，然后取一个窗口。

**第三步：LLM 摘要而非原文注入**（`session_search_tool.py` L175-200）

```python
async def _summarize_session(conversation_text, query, session_meta):
    system_prompt = (
        "You are reviewing a past conversation transcript to help recall what happened. "
        "Summarize the conversation with a focus on the search topic. Include:\n"
        "1. What the user asked about or wanted to accomplish\n"
        "2. What actions were taken and what the outcomes were\n"
        "3. Key decisions, solutions found, or conclusions reached\n"
        "4. Any specific commands, files, URLs, or technical details\n"
        "5. Anything left unresolved or notable\n"
    )
    response = await async_call_llm(
        task="session_search",
        messages=[...],
        temperature=0.1,     # 低温度，保证事实准确
        max_tokens=10000,
    )
```

用 **auxiliary model**（通常是 Gemini Flash 这种便宜快速的模型）做摘要，而不是把原始对话塞进主模型的上下文。这样做的好处：
- 主模型的上下文窗口不会被历史对话占满
- 摘要是**聚焦于搜索主题**的，而不是全量信息

#### 举例

> **场景**：三周前你让 Hermes 帮你解决了一个 Docker 网络问题——容器之间无法通过 hostname 互访。今天你在另一个项目里遇到了类似问题。
>
> **触发**：你说"我之前好像解决过 Docker 容器网络互访的问题，帮我回忆一下"。Agent 调用 `session_search(query="Docker 容器网络互访")`。
>
> **执行过程**：
> 1. FTS5 在 SQLite 中搜索包含"Docker"、"容器"、"网络"的历史消息
> 2. 找到 3 周前的会话，取出对话内容
> 3. `_truncate_around_matches` 围绕匹配位置截取 100K 字符窗口
> 4. Gemini Flash 生成摘要："用户遇到 Docker Compose 中容器无法通过 service name 互访的问题。原因是容器在不同的 network 中。解决方案：在 docker-compose.yml 中显式定义一个共享 network，所有需要互访的 service 都加入该 network..."
>
> **结果**：Agent 拿到摘要后，直接按照上次的解决方案来处理当前问题，不需要重新排查。

---

### 🔄 闭环四：GEPA 进化优化 — 离线技能进化

#### 核心架构（`hermes-agent-self-evolution` 仓库）

这是一个**独立仓库**，不在 hermes-agent 内部运行，而是**作用于** hermes-agent。

**优化循环**：

```
1. SELECT TARGET — 选择要优化的 Skill
2. BUILD EVALUATION DATASET — 构建评测数据集
   ├── Source A: 强模型合成测试用例（15-30 个）
   ├── Source B: SessionDB 挖掘真实使用数据
   ├── Source C: 手工标注黄金集
   └── Source D: 技能特定的自动评估
3. WRAP AS DSPy MODULE — 把 Skill 包装成可优化的 DSPy 模块
4. RUN GEPA OPTIMIZER — 遗传进化
   ├── 读取执行轨迹（traces）理解"为什么失败"
   ├── 提出针对性变异（不是随机变异）
   └── 只需 3 个样本就能工作
5. EVALUATE & COMPARE — 在 holdout 集上对比
6. DEPLOY via PR — 人工审核后合并
```

**GEPA 的核心优势**：它不是盲目变异文本，而是**读取 batch_runner 产生的执行轨迹**，理解 Agent 在哪一步出了问题、为什么选错了工具、为什么遗漏了关键步骤，然后做**反射性改进**。

**五层约束门控**：

```
Candidate Variant
│
├──► pytest (2550+ tests, 必须 100% 通过)     ← GATE 1
├──► TBLite fast (20 tasks, ~20min)           ← GATE 2
├──► 任务特定评测集                             ← FITNESS
│
▼ Top 3 Candidates Only
├──► Full TBLite (100 tasks)                  ← GATE 3
├──► YC-Bench (长对话连贯性)                    ← GATE 4
│
▼ Best Candidate → PR with metrics
```

**五阶段进化路线**：

| 阶段 | 目标 | 风险 | 状态 |
|------|------|------|------|
| Phase 1 | Skill 文件（SKILL.md） | 最低 | ✅ 已实现 |
| Phase 2 | Tool descriptions（工具描述文本） | 低 | 🔲 计划中 |
| Phase 3 | System prompt 各段落 | 中高 | 🔲 计划中 |
| Phase 4 | Tool 实现代码（Python） | 最高 | 🔲 计划中 |
| Phase 5 | 持续自动化循环 | — | 🔲 计划中 |

**评分机制**：LLM-as-Judge + Rubric

```
对于大多数 Skill，没有二元对错——质量是主观的。
评分维度：
1. Agent 是否遵循了 Skill 的流程？(0-1)
2. 输出是否正确/有用？(0-1)
3. 是否简洁（在 token 预算内）？(0-1)
```

#### 举例

> **场景**：`github-code-review` 这个 Skill 在实际使用中，经常漏掉 SQL 注入类的安全问题。
>
> **Phase 1 执行过程**：
> 1. **构建评测集**：用 Claude Opus 生成 20 个代码审查测试用例，其中 5 个包含 SQL 注入漏洞
> 2. **基线评估**：用当前 Skill 跑 batch_runner，发现 SQL 注入检出率只有 40%
> 3. **GEPA 优化**：
>    - GEPA 读取执行轨迹，发现 Agent 在审查时跳过了数据库交互层的代码
>    - 提出变异：在 Skill 的 "Procedure" 部分增加 "Always check database query construction for parameterized queries vs string concatenation"
>    - 生成 10 个候选变体
> 4. **评估**：最佳变体在 holdout 集上 SQL 注入检出率提升到 80%
> 5. **约束检查**：Skill 大小仍在 15KB 以内，pytest 全部通过，TBLite 无回归
> 6. **部署**：自动创建 PR，diff 清晰展示改了什么，人工 review 后合并
>
> **成本**：整个优化过程约 $2-10（纯 API 调用，无需 GPU）

---

### 🧠 四个闭环的协同关系

```
在线层（Runtime，每次对话都在运行）
┌─────────────────────────────────────────────────────┐
│                                                     │
│  闭环一 Memory Nudge ←──── 闭环三 Session Search    │
│  "记住用户是谁"            "回忆过去做过什么"         │
│       │                         │                   │
│       ▼                         ▼                   │
│  MEMORY.md / USER.md    历史会话摘要                 │
│       │                         │                   │
│       └────────┬────────────────┘                   │
│                ▼                                    │
│        System Prompt（下次会话生效）                  │
│                                                     │
│  闭环二 Skill Auto-Creation                         │
│  "把经验沉淀成可复用的操作手册"                       │
│       │                                             │
│       ▼                                             │
│  ~/.hermes/skills/xxx/SKILL.md                      │
│                                                     │
└─────────────────────────────────────────────────────┘
                    │
                    ▼ Skill 文件作为优化目标
离线层（Evolution，定期/手动运行）
┌─────────────────────────────────────────────────────┐
│                                                     │
│  闭环四 GEPA 进化优化                                │
│  "用遗传算法让 Skill 文本越来越好"                    │
│                                                     │
│  SessionDB ──► 评测数据 ──► GEPA 变异 ──► PR        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**关键洞察**：

1. **闭环一和闭环二共享同一个触发机制**（`_spawn_background_review`），但用不同的计数器和不同的 prompt。Memory 按对话轮数触发，Skill 按工具调用次数触发——这精准地区分了"用户信息积累"和"复杂任务完成"两种场景。

2. **闭环三为闭环一和闭环二提供输入**。Session Search 找到的历史经验，可能触发 Agent 主动更新 Memory 或改进 Skill。

3. **闭环四以闭环二的产出为优化目标**。Agent 在线创建的 Skill 质量参差不齐（毕竟是 LLM 生成的），GEPA 在离线阶段对这些 Skill 做系统性优化。

4. **所有闭环都有安全兜底**：Memory 有注入扫描，Skill 有安全审计 + 原子写入 + 回滚机制，GEPA 有五层约束门控 + 人工 PR review。

这就是为什么 Hermes 的 slogan 是 **"The agent that grows with you"** —— 它不是一个静态的工具，而是一个会随着使用越来越强的系统。四个闭环从不同时间尺度（实时、会话间、离线）持续改进 Agent 的能力。