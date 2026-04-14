
# 01. Skills 太多导致上下文爆炸——核心解法框架

### 问题本质

每个 Skill 有三层内容，全量注入代价极高：

```
Layer 1: name + description     ← 轻量，几十 token
Layer 2: SKILL.md 完整文档      ← 中等，几百 token
Layer 3: Tools schema（多个）   ← 最重，每个 tool ~500-1000 token
```

全量注入 = Layer1 + Layer2 + Layer3 × N 个 Skill，直接爆炸。

---

## 两大主流解法

### 方案一：多步懒加载（Hermes / OpenClaw / LangChain）

**核心思想：按需、分层、逐步注入，用多次 API 调用换 token 节省。**

```
Pass 1 ── 只注入 Layer 1（name + description）
           ↓ 模型决定用哪个 Skill
Pass 2 ── 注入该 Skill 的 Layer 2（SKILL.md）
           ↓ 模型决定用哪个 Tool
Pass 3 ── 注入该 Tool 的完整 schema（Layer 3）
           ↓ 执行
```

**各家的具体实现差异：**

| 框架 | Pass 1 的实现方式 | 特点 |
|------|-----------------|------|
| **OpenClaw** | system prompt 只放 `<available_skills>` 名称列表，调 `skill_view(name)` 懒加载 SKILL.md | 最直接，skill 目录是静态文本 |
| **HermesAgent** | 只发 name + 一行描述 + 一个特殊工具 `request_tool(name)`，模型主动请求完整 schema | 两阶段，tool schema 也懒加载 |
| **LangChain BigTool** | 所有 skill/tool 的 description **预先 embedding** 存向量库，Pass 1 用用户 query 做语义搜索召回 top-k | 不是顺序三步，而是向量检索直接跳到最相关的 skill，精度更高 |

**LangChain BigTool 的特殊性：** 它把 Pass 1 从"模型自己选"变成了"向量检索选"，官方数据显示 tool 选择准确率提升 **3 倍**，且对 skill 数量几乎无上限（支持 1000+ tools）。

**代价：** 每次工具调用多 1-2 次 API round trip，有额外延迟。

---

### 方案二：Agent 隔离（DeerFlow）

**核心思想：不在单 Agent 里解决 skill 太多的问题，而是从架构层拆掉这个问题。**

```
Supervisor Agent
    ├── Sub-Agent A（只配 skill: 搜索、爬虫）
    ├── Sub-Agent B（只配 skill: 代码执行、文件操作）
    └── Sub-Agent C（只配 skill: 数据分析、报告生成）
```

每个子 Agent 的上下文窗口只装自己负责的那几个 skill，**根本不存在"skill 太多"的问题**。

DeerFlow 的具体做法：新建 Custom Agent 时 skill 和 tool 默认为空，**按需显式配置**，不默认加载所有权限。

**代价：** 架构复杂度高，子 Agent 间的任务路由、状态同步、失败处理都要额外设计。

---

## 两种方案的选型判断

| 维度 | 多步懒加载 | Agent 隔离 |
|------|-----------|-----------|
| **适用场景** | 单 Agent 需要访问大量 skill，但每次只用少数几个 | 任务本身可以清晰拆分为子任务，子任务能力差异大 |
| **工程复杂度** | 低，改 prompt 构建逻辑即可 | 高，需要设计路由、状态同步、仲裁机制 |
| **延迟影响** | 多 1-2 次 API round trip | 子 Agent 并行时延迟可控 |
| **skill 数量上限** | LangChain BigTool 可到 1000+，其他方案 ~100 | 理论无上限，取决于子 Agent 数量 |
| **调试难度** | 低，单链路可追踪 | 高，多 Agent 状态难以追踪 |

**实践建议：** 大多数场景先用懒加载，skill 数量 < 50 时 OpenClaw/HermesAgent 的方案够用；skill 数量 > 100 或需要语义精准匹配时上 LangChain BigTool 的向量检索；只有任务本身天然可拆分时才考虑 Agent 隔离。



# 02. Skill description 相似导致误召回

### 根本问题

> **只靠 name + description 做路由，信息量天然不足，准确率会系统性崩塌。**

阿里巴巴 SkillRouter 论文（2026，~80K skills 大规模实验）给出了量化结论：

- 把 skill body 从路由信号中移除 → 准确率下降 **31~44 个百分点**
- description 质量再高也救不了，问题出在**信息量本身不够**，而不是描述写得好不好

---

### 两大主流解法

#### 方案一：粗筛 → 精筛（LangChain + 阿里 SkillRouter）

**核心思路：** 不在第一步就强行区分，而是先召回 top-k 候选，再用更丰富的信息做精细重排。

```
用户 query
    ↓
粗筛：name + description → 向量检索召回 top-k（LangChain BigTool）
    ↓
精筛：加载完整 skill body → rerank 模型重排（阿里 SkillRouter）
    ↓
选出最终 skill
```

**各家实现差异：**
- **LangChain BigTool**：把所有 skill 的 description 预先 embedding 存向量库，用 query 做语义搜索召回 top-k，再让模型在 top-k 里精细区分
- **阿里 SkillRouter**：用完整 skill body（不只是 description）做 embedding，0.6B encoder 召回 + 0.6B reranker 精排，两个关键训练技巧：
  - **False-negative filtering**：两个 skill 内容高度相似时，不能把其中一个当负样本，否则模型学坏
  - **Listwise reranking**：不是两两比较，而是在候选列表里做整体排序，更适合 skill 高度同质化的场景

---

#### 方案二：增加信息丰富度（HermesAgent + OpenClaw）

**核心思路：** 不只靠 name + description，在 skill 定义层就强制补充更多判别信号，让路由有更多信息可用。

**强制结构化 skill 定义（HermesAgent SKILL.md 规范）：**
```
- 适用场景（when to use）       ← 正样本
- 禁止场景（when NOT to use）   ← 负样本，关键！
- 输入参数约束
- 前置条件
- 预期输出格式
- 调用示例（正例 + 反例）
```

**OpenClaw 的补充：** 不依赖 description 做路由，而是用**实际加载的 skill 名称**做精确能力索引，coordinator 看到的是每个 Agent 实际持有的 skill 列表，而不是模糊描述。

**执行前兜底（HermesAgent）：** 模型选定 skill 后，执行前做 schema 校验——如果当前输入不满足该 skill 的前置条件，直接中断，强制重选，不让错误扩散。

---

### 两种方案的本质区别

| | 粗筛→精筛 | 增加信息丰富度 |
|--|---------|-------------|
| **解决层次** | 路由算法层 | skill 定义层 |
| **适用规模** | 大规模（1000+ skills，高度同质化） | 中小规模（<100 skills） |
| **工程成本** | 高（需要向量库 + rerank 模型） | 低（改 skill 定义规范即可） |
| **核心信号** | skill 完整 body 的语义 | 正负样本 + 约束条件 |
| **代表** | LangChain BigTool、阿里 SkillRouter | HermesAgent、OpenClaw |

---

### 实践建议

> **两种方案不是互斥的，最稳健的做法是叠加使用：**
> 1. 先把 skill 定义做结构化（加正负样本、约束条件）——这是基础，成本低
> 2. 再在路由层做粗筛→精筛——这是规模化的保障
> 3. 最后在执行前加 schema 校验兜底——这是最后一道防线



#03. Skill，和 Tool 的区别是什么？

**Tool：**
- 原子能力（如查数据库、读文件、发请求、执行命令）
- 最低层操作接口

**Skill：**
- 稳定的能力模块，由多个 Tool 和工作流策略组成
- 如"审阅合同风险"、"生成测试补丁"、"整理会议纪要"
- 包含任务边界、输入输出约束、错误处理逻辑
- 复用性更高

**工程视角：**
- 只暴露 Tool：模型每次重新组合，稳定性差
- 封装成 Skill：模型在更高抽象层操作，方便版本管理和效果评估

**例子：**
- Crash Analysis 根因分析例子