# 从可观测到可理解:用 UModel 构建 Agent 原生的代码知识图谱

**作者**: 元乙  
**发布时间**: 2026年4月19日  
**来源**: 阿里技术 ATA  
**浏览量**: 317 | **点赞**: 32 | **评论**: 7

---

## 背景

近年来,AI Agent(Cursor、Copilot、Claude Code、Codex 等)已经深度参与软件开发。从代码补全到跨文件重构,从 Bug 定位到架构设计,Agent 能力越来越强。从 Prompt Engineering 到 Context Engineering 到 Harness Engineering,驾驭 AI 的方式也在不断进化,Agent 的能力边界也在不断扩展。

然而,当我们把一个真实的企业级项目交给 Agent 时,一个被忽视的问题开始浮现:**Agent 真的理解你的项目吗?**

当前 Agent 理解代码的方式正在分化成两个明确的流派:

- **无索引流派**:Claude Code 信奉 Unix 哲学,完全不做预索引:用 `grep`、`rg`、`glob` 实时搜索文件系统。Anthropic 内部测试发现 agentic search 性能全面超越 RAG,by a lot。简洁、实时、无隐私问题,但每次会话从零开始,且对大仓库代价高昂。

- **CodeIndex 流派**:Cursor、Windsurf、Copilot 走向量索引路线:用 tree-sitter 做语义切片,生成 embedding 存入向量数据库(如 Turbopuffer),再用 Merkle tree 做增量同步。Qodo 和 Augment Code 更进一步,在向量索引之上叠加了代码依赖图和 commit 历史索引。

两个流派各有所长,但面对以下问题时还是难以很好工作:

- 我要改 pkg/a2a 的 Adapter 接口,影响范围是什么?
  - 向量相似度搜不到依赖链,grep 逐文件搜效率低且不完整
- 生产上 vibeops-xxx SLO 破线了,有大量的 pending request,什么原因,是代码变更吗?
  - 代码索引只覆盖代码域,运维域的数据不在图里
- 项目里有没有跨越架构边界的异常依赖?
  - 没有架构层级建模,跨越边界无从定义

这些问题的共性是:**它们需要确定性的结构关系、跨域的实体关联、以及时间维度的变更历史。**

这和可观测领域的发展轨迹惊人相似:从看日志到统一建模,可观测经历了从零散数据到 UModel 知识图谱的进化。而代码理解,即使最先进的 CodeIndex 方案,也还停留在帮 Agent 找到相关代码片段的阶段:找到了片段,但不理解结构。

![代码理解的挑战](https://oss-ata.alibaba.com/article/2026/04/da526ce9-0b83-437b-ad0a-4c952d72fae8)

---

## 代码理解的五种范式

在展开技术方案之前,有必要先厘清当前代码理解的完整图谱。五种范式,代表了从无状态搜索到有状态推理的演进变化。

![代码理解的五种范式](https://oss-ata.alibaba.com/article/2026/04/977c876a-8c92-4066-b1a3-61b53bc95c35)

### 范式一:Agentic Search(Claude Code 流派)

Claude Code 是当前最极致的无索引路线。Anthropic 创始工程师 Boris Cherny 公开分享过这个决策的来龙去脉:Claude Code 早期版本用过 RAG + 本地向量库,但内部测试发现 agentic search 全面胜出:by a lot, and this was surprising。

它的工作方式纯粹到优美:

```
Agent 收到问题
  → Glob: 按文件名模式匹配(近零 token 成本)
  → Grep (ripgrep): 按内容正则搜索(低 token 成本)
  → Read: 读取完整文件(高 token 成本)
  → 判断 → 下一轮搜索或给出答案
```

工具按 token 成本分层,Agent 自主决定搜索策略,像一个经验丰富的开发者在终端里 `rg` + `cat` 排查问题。这种 Unix 哲学的方法有几个真实的优势:

- **零预处理**:不需要索引构建时间,打开项目即可工作
- **永远新鲜**:没有索引过期问题,每次搜索都是实时文件系统状态
- **隐私友好**:代码不离开本地,不生成 embedding,不上传到任何服务器
- **简单可靠**:依赖链极短:Agent + 文件系统 + ripgrep,没有向量数据库可以挂掉

但这种方法的天花板同样明确:

- **无结构感知**:`rg HandleRequest` 能找到所有出现位置,但分不清哪些是定义、哪些是调用、哪些是注释。Agent 要靠自己阅读代码来判断。
- **每次从零开始**:上一个会话分析过的依赖关系,下一个会话全部作废。没有持久化的知识积累。
- **规模受限**:200 个文件的 TypeScript 项目没问题;5 万个文件的企业级单仓,agentic search 可能需要 30+ 轮工具调用和几万 token 才能拼凑出全局依赖图。实际上根本不可能拼出完整的全局图:只能拼出与当前任务相关的局部视图。
- **无法做全局分析**:无法回答列出所有跨架构层级的调用,因为架构层级本身就没有被建模。

### 范式二:CodeIndex / 向量索引(Cursor、Windsurf、Copilot 流派)

这是当前 AI IDE 的主流技术路线。以 Cursor 为例,其技术架构已有大量公开分析:

```
代码仓库
  → tree-sitter 解析为 AST
  → 按语义单元切片(函数、类、逻辑块)
  → 生成向量 embedding
  → 存入 Turbopuffer 向量数据库
  → Merkle Tree 跟踪变更,增量同步
```

Cursor 在工程上做到了几个精巧的优化:每 10 分钟用 Merkle Tree 根哈希比对检测变更,只重新嵌入变化的文件;团队成员间 92% 的代码库相似度可以复用索引,将新成员的首次索引从数分钟降到秒级;通过 .cursorignore 控制索引范围。

Windsurf(Codeium)用类似的 RAG 架构:768 维向量嵌入 + 自研 M-Query 检索:但额外叠加了 Cascade 上下文引擎,跟踪编辑历史、终端命令、导航模式等会话状态。GitHub Copilot 在 2025 年 3 月将语义搜索索引做到了秒级完成。

CodeIndex 的真正价值是语义搜索:Agent 不用知道精确的函数名,用自然语言描述意图就能找到相关代码。这是 grep 做不到的。

但 CodeIndex 有一个根本性的局限:**向量相似度是文本层面的近似匹配,不是结构层面的关系推理。**

- `import pkg/a2a` 在代码里是一条确定性的依赖关系,但在向量空间里只是一个文本片段的相似度信号
- 找到所有直接或间接依赖 pkg/a2a 的模块需要图遍历,不是向量检索
- 这个接口变更的影响沿调用链传播了几跳需要确定性的 calls 关系,不是语义相似度
- Augment Code 的评测显示,Cursor 在 50+ 文件的跨文件重构中会出现不一致:前 30 个文件改对了,后 20 个因为上下文窗口溢出而产生错误

CodeIndex 本质上是更聪明的搜索引擎:帮 Agent 找到正确的代码片段塞入上下文,但不替 Agent 做结构化推理。

### 范式三:Code Graph + RAG 混合(Qodo、Augment Code 流派)

Qodo 和 Augment Code 代表了 CodeIndex 的下一个演化方向:在向量索引之上叠加代码结构图。

Qodo 的技术栈显得尤其认真:

- 自研 Qodo-Embed-1 代码嵌入模型(1.5B 参数在 CoIR 基准上超越 7B 竞品),通过合成数据训练来捕获语法、变量依赖、控制流、API 用法等代码特有语义
- 客户端构建代码图:函数、类、模块及其调用图、继承关系、跨语言链接
- 服务端维护向量数据库 + 设计文档 + 架构图 + PR/commit 历史
- AST 感知的切片策略:递归切分 AST 节点,回填 import 语句和类定义等关键上下文

Augment Code 的 Context Engine 走得更远:

- 跨仓库的语义索引,理解服务间如何连接和依赖
- 索引范围超出代码:commit 历史(为什么改)、codebase patterns、外部文档、ticket、甚至tribal knowledge
- 2025 年发布 Context Lineage,索引 commit 历史和 diff 摘要,让 Agent 理解架构决策的演变
- 通过 MCP 协议开放给任何兼容的 Agent,基准测试显示 30-80% 的质量提升

这一流派的关键进步是:代码不只是文本,还是一个有结构的图。Augment 尤其体现了理解需要上下文,上下文需要历史的洞察。

但即使是最先进的 Code Graph + RAG 混合方案,仍有几个系统性的边界:

- **图的范围限于代码域**:知道 A 调用 B,但不知道 B 对应的服务在生产环境出过什么告警。代码图和运维图是割裂的。
- **图的查询能力有限**:服务于 RAG 检索的图,通常支持邻居查找和短路径查询,但不支持任意深度的图遍历、模式匹配、聚合分析。
- **IDE 局部,非团队全局**:索引绑定在开发者的 IDE 实例上。一个人分析过的结构洞察,另一个人无法直接复用。
- **缺乏标准化的时序维度**:Augment 的 Context Lineage 开始做 commit 历史,但构建日志、部署日志、测试日志、事件日志:这些完整的时间记忆还不在图中。

### 范式四:CodeWiki / LLM 文档(DeepWiki 流派)

DeepWiki(GitHub 15.7k star,Cognition AI / Devin 背后的团队出品)代表了另一种路线:代码仓库 → LLM → 精美 Wiki 文档。只需把 URL 中的 github.com 替换为 deepwiki.com,就能看到自动生成的架构图、模块文档、函数注解。

对人类开发者快速了解陌生项目,这是极好的体验。DeepWiki 还支持通过 .devin/wiki.json 配置文件控制生成范围,通过 MCP Server 提供 ask_question、read_wiki_structure、read_wiki_contents 等工具接口。

但文档本质上是为人类阅读而优化的线性叙事:

- **难验证**:LLM 生成的描述可能含有幻觉,而在代码理解中,一条错误的A 调用 B比没有信息更危险
- **难遍历**:文档无法回答列出所有调用 X 的函数这类图遍历查询
- **难推理**:不支持多跳分析:改了 A,沿 calls 关系走 3 跳,影响到哪些入口点
- **难维护**:改一行代码就要全量重生成。虽然 DeepWiki 支持 badge 触发自动刷新,但每次都是全量 LLM 调用,成本和延迟都不低
- **不可编程**:MCP 接口本质上是问文档一个问题,不是在图上执行一次查询

CodeWiki 和 CodeIndex 的关系,类似于数据库领域里物化视图和查询引擎的关系:文档是预计算好的视图,回答预设问题很快,但无法回答视图之外的临时查询。

### 范式五:代码知识图谱(我们的选择)

五种范式可以按一条轴线排列:从"无状态搜索"到"有状态推理"。

| | Agentic Search | CodeIndex | Code Graph+RAG | CodeWiki | 知识图谱 |
|---|---|---|---|---|---|
| **代表** | Claude Code | Cursor、Windsurf、Copilot | Qodo、Augment Code | DeepWiki | UModel |
| **核心技术** | grep / rg | 向量相似度 | 结构图 + 向量 | LLM 文档 | Entity + Log + Link |
| **索引策略** | 零预处理 | 语义切片 + embedding | AST 依赖图 + embedding | 全量 LLM 生成 | AST 确定性提取 + LLM 增强 |
| **状态持久化** | 每次从零 | Merkle Tree 增量同步 | 部分增量 | 全量重生成 | SHA256 增量构建 |
| **覆盖域** | 纯代码域 | 纯代码域 | 代码域 | 代码域 | 代码 + 运维 + 需求 |

如果说 Agentic Search 是每次现场勘察,CodeIndex 是带着高清地图勘察,Code Graph+RAG 是地图上标注了公路和铁路,CodeWiki 是请人写了一本地方志:那我们要建的是一个活的 GIS 系统:可以查询任意两点之间的路径,可以叠加实时交通数据,可以标注每条路的通行历史,可以随着地形变化持续更新,支持任意维度的空间分析。

**关键差异不是更好的搜索,而是三个维度的系统性结合:**

1. **确定性 vs 概率性**:CodeIndex 给你的是最可能相关的代码片段(向量相似度),Code Graph 给你的是AST 解析出的结构关系(但查询能力受限于 RAG 框架),我们给你的是AST 确定性提取 + SPL/graph-match 任意查询:置信度 1.0 的关系 + 图灵完备的查询语言。

2. **代码域 vs 跨域**:从 Agentic Search 到 Code Graph+RAG,所有方案都止步于代码域。这个模块调用了哪些函数:可以回答。这个模块对应的生产服务上周出过几次告警:无法回答。UModel 的 EntitySetLink 可以将 code.module 连接到 ops.service、event.alert、req.issue,Agent 沿着链路推理不需要跳出图谱。

3. **快照 vs 时间线**:CodeIndex 是当前代码的快照索引。Code Graph 开始加入 commit 历史。我们提供完整的时间维度:commit_log、build_log、deploy_log、test_log、incident_log,每种 LogSet 通过 DataLink 关联到 EntitySet。Agent 不只知道现在的结构是什么,还知道它是怎么演变到这里的和它在生产上表现如何。

---

## 从个人 Wiki 到代码 Wiki:同一套范式,不同的确定性

在之前的文章《如何用 UModel 构建一个会成长的个人 Wiki》中,我们探讨了一个核心问题:如果我们真的要做一个可长期维护、可持续演进、对 Agent 友好的知识系统,底层应该长什么样?

那篇文章的结论是:真正让知识系统长期成立的,不只是 Agent,也不是 Markdown,而是一个足够清晰的建模层。UModel 提供了这个建模层:用 Set 表达对象,用 Link 表达关系,用 Field 约束语义。

个人 Wiki 的流程是:原始资料 → LLM 抽取实体和关系 → 对齐与规范化 → UModel 结构层 → Wiki 页面。整个抽取过程完全依赖 LLM,因此每条关系天然带有不确定性:张城和元乙是不是同一个人?这篇文章和那个项目有没有关系?都需要 LLM 来判断,需要对齐层来修正。

**代码领域有一个根本不同:代码的结构关系是确定性的。**

`import pkg/a2a` 就是导入了 pkg/a2a,`func (s *Server) HandleRequest()` 就是 Server 类型的方法:这些不需要 LLM 推断,AST 解析即可确定,置信度 = 1.0。

这意味着代码 Wiki 可以在个人 Wiki 的范式之上,引入一个模型层的确定性保证:

```
个人 Wiki:   原始资料 → [LLM 抽取] → 对齐 → UModel → Wiki 页面
                          ↑ 全程依赖 LLM,置信度 0.4-0.9


代码 Wiki:   代码仓库 → [AST 确定性提取] + [LLM 语义增强] → UModel → CLI 查询
                          ↑ 结构关系确定(1.0)   ↑ 摘要/归属补充(0.6-0.9)
```

这一层确定性对 Agent 的推理至关重要:当 Agent 做 RCA 时,它需要信任调用链上的每一跳。如果某条 calls 关系是 LLM 猜的,整个推理链就不可靠。而 AST 提取的关系是确定性事实,Agent 可以无条件信任。

同时,代码 Wiki 保留了个人 Wiki 的 LLM 增强能力:模块摘要、文档-代码关联、组件归属这些语义层信息仍由 LLM 生成,标注为 INFERRED,Agent 可以选择性采信。

---

## Entity + Log + Link:不只是结构图

UModel 在可观测领域的核心设计是用 Set 和 Link 组成的图来描述 IT 世界:EntitySet 描述实体现态,LogSet 描述时序事件,MetricSet 描述度量指标,Link 将它们连接成网络。

当我们把同样的建模方法论应用到代码领域,获得的不只是一张结构图。

### Entity:代码的当前结构

5 种 EntitySet 描述代码的现态,通过 repo_id 复合主键支持多仓库共存:

| EntitySet | 主键 | 核心字段 |
|---|---|---|
| `code.module` | repo_id + module_path | repo_id, loc, summary, sha256 |
| `code.function` | repo_id + qualified_name | repo_id, signature, importance, visibility |
| `code.type` | repo_id + qualified_name | repo_id, type_kind, signature |
| `arch.component` | repo_id + component_name | repo_id, component_type, description |
| `doc.document` | repo_id + doc_path | repo_id, doc_title, doc_type |

repo_id 参与主键计算(Entity ID = md5(repo_id:pk_value)),使得不同仓库的同名模块不会冲突,同一个图谱可以同时容纳多个项目。

6 种 EntitySetLink 描述结构关系:contains、imports、calls、extends、describes、belongs_to。每条关系标注 __confidence__ 和 __extraction_method__(EXTRACTED / INFERRED / AMBIGUOUS)。

### Log:代码的变化历史

这是 Code-WIKI 和所有纯图谱工具的关键分水岭。

在可观测领域,我们不只看 Pod 当前的状态(Entity),还要看它的日志和指标趋势。代码也一样:只看结构不看历史,等于只看了一帧截图。

代码领域的 Log 远不止 Git Commit:

| LogSet | 关联实体 | 记录内容 | 回答的问题 |
|---|---|---|---|
| `code.commit_log` | module, function | author, message, diff stat | 谁改了什么?为什么改? |
| `code.review_log` | module | reviewer, comments, 关联 issue | 审查决策链路 |
| `code.build_log` | module | pipeline status, duration, error | 构建是否健康? |
| `code.deploy_log` | module, service | environment, version, rollback | 何时到达生产? |
| `code.test_log` | function | pass/fail, coverage, duration | 测试覆盖和健康趋势 |
| `code.incident_log` | module, service | severity, 关联告警 | 引发过什么生产问题? |

Log 的价值在于和 Entity 的关联查询:

- 这个模块最近一周被谁修改过? → commit_log WHERE module_path = X AND time > now()-7d
- 上次部署之后有没有新增 incident? → deploy_log JOIN incident_log ON time_window
- 引入这个依赖之后,构建时间变长了吗? → build_log GROUP BY week,交叉 commit_log 的依赖变更时间

每条 LogSet 通过 DataLink 关联到对应的 EntitySet,Agent 可以从 Entity 跳转到 Log,或从 Log 回溯到 Entity。

### 跨域关联:代码不是一座孤岛

代码从来不是独立存在的。它服务于需求,通过 CICD 到达生产,在运行时产生可观测数据,出了问题又回到代码来定位。当前的工具链中,每个环节都是一座孤岛:需求在 Jira 里,代码在 Git 里,构建在 Jenkins 里,运行在 K8s 里,告警在监控系统里。

当生产告警响起时,从告警追溯到代码变更,中间要跳转多少个系统、人工关联多少条信息?

UModel 的价值在于:所有这些实体可以活在同一个图里。

---

## 技术架构:双轨提取 + 图谱构建

### 整体流水线

### DETECT:增量变更检测

每个文件计算 SHA256 内容指纹,和上次构建的缓存比对。对于 vibeops-agents(~2375 个 Go 文件),增量构建通常只需处理几十个变更文件,耗时从全量的数分钟降到秒级。

### EXTRACT:AST + LLM 双轨

**AST 轨道(tree-sitter)**:基于 PEG 的增量解析器,支持 40+ 语言,通过 tags.scm 规则跨语言一致地提取定义、引用、结构关系、导入关系、调用关系、继承关系。所有提取结果置信度 = 1.0。

值得注意的是,Cursor 等 CodeIndex 方案也使用 tree-sitter:但它们用 tree-sitter 做语义切片(把代码切成适合 embedding 的块),我们用 tree-sitter 做结构提取(提取定义、引用、调用、继承等确定性关系)。同一个解析器,服务于完全不同的目标:前者产出向量,后者产出图谱。

**LLM 轨道**:模块摘要(Agent 上下文注入片段,不是人读的文档)、文档-代码关联、组件归属。每条标注 __extraction_method__: INFERRED + 置信度。Agent 可按场景选择信任阈值:RCA 高置信度优先,探索场景可放宽。

### RESOLVE:跨文件符号解析

单文件 AST 无法解决跨文件引用。RESOLVE 处理:

- Go import github.com/org/repo/pkg/a2a → module_path `pkg/a2a`
- 方法 receiver type `(s *Server)` → 归属 code.type `pkg/server.Server`
- 调用 `s.HandleRequest()` → pkg/server.Server.HandleRequest
- 接口实现 `type Adapter struct implements Handler` → extends 关系

确定性解析,不依赖 LLM。

### BUILD:图谱组装 + 架构发现

架构发现不是简单的社区检测:Louvain/Leiden 发现的是聚类,不是架构。完整流程:

```
Step 1: 图构建
  module 为节点,imports + calls + extends 为有向边
  边权重:calls > imports > extends

Step 2: 层次分析
  计算依赖方向性:A→B 且 B↛A → A 在 B 上层
  识别入度 = 0 的顶层入口和出度 = 0 的底层基础设施

Step 3: 社区发现
  Leiden 算法在有向图上发现功能簇
  resolution 参数控制粒度(~150 模块 → ~15 组件)

Step 4: 标注与命名
  基于依赖方向标注层次:API/Gateway、Service/Business、Infrastructure/Utility
  LLM 命名和描述,与项目文档交叉验证
```

输出是有层次、有方向、有命名的架构视图。Agent 可以据此判断这个调用是否跨越了架构层次。

### SYNC:同步到 UModel

```
Entity 写入: starops umodel post-logs → __entity logstore
Topo 写入:   starops umodel post-logs → __topo logstore
Schema 同步: starops umodel sync(注册 EntitySet/Link 定义)
```

UModel 后端基于 SLS 存储引擎,继承了高吞吐写入、秒级查询、graph-match 图遍历、SQL 聚合、全文检索等能力。

### SERVE:查询的工程细节

实践中摸索出的关键模式:

**两步查询**:graph-match 返回 entity_id,不含业务字段。所有图遍历查询先走拓扑获取 id 集合,再批量拉取业务字段:

```
Step 1: .topo | graph-match (n1:code@code.module {__entity_id__: '<id>'})
              -[e]->(n2) project n1, e, n2

Step 2: .entity with(domain='code', name='code.module', ids=['id1','id2',...])
```

**聚合走 SLS 直查**:热点分析等统计查询直接 SQL 查 __topo logstore:

```
SELECT __dest_entity_id__, count(1) as import_count
FROM log WHERE __relation_type__ = 'imports'
GROUP BY __dest_entity_id__
ORDER BY import_count DESC LIMIT 20
```

在当前多仓库规模(~11000 实体、~19000 条边,含 vibeops-agents 和 starops-cli 两个项目)下,单次查询端到端延迟百毫秒。

---

## Agent 交互层:CLI + Skill

### CLI 设计

Agent 的推理是渐进式的:先搜索,看到结果再决定下一步。CLI 的 search → context → impact 天然匹配这个模式,且支持批量执行和管道组合。

```
code-wiki query <子命令>     # 图谱查询
  ├── search <keyword>       # 实体搜索
  ├── context <name>         # 符号完整上下文
  ├── impact <path>          # 变更影响分析
  ├── callers / callees      # 调用链
  ├── deps / rdeps           # 依赖 / 反向依赖

code-wiki check <子命令>     # 治理检查
  ├── arch                   # 架构违规扫描
  └── hotspots               # 耦合热点

code-wiki ingest             # 构建/更新图谱
code-wiki status             # 健康检查
```

子命令按 Agent 意图组织。Agent 不需要知道底层是 graph-match 还是 SLS SQL:看影响范围就用 impact。

### 输出格式:为 Agent 上下文窗口优化

默认 --format brief 输出为 Agent 的 token 预算优化:

```
$ code-wiki query context pkg/a2a

Module: pkg/a2a
  LOC: 1,247 | Language: Go | Component: a2a-protocol
  Summary: A2A protocol implementation for agent-to-agent communication

Types (17): TaskStore(struct), A2AServer(struct), AgentCard(struct), ...
Functions (52): HandleA2ARequest[entry], StartA2AServer[entry], ...
Reverse dependencies (9): pkg/api/handler, pkg/server, cmd/vibeops-agents, ...
Component crossings: → api, → scheduler
```

一个 query context 的输出 < 500 tokens。需要完整数据时用 --format json。

### Skill:场景化使用指南

配合 CLI 的 Agent Skill 按场景组织,Agent 不需要学习 SPL 语法:

```
## RCA:从告警追到代码
code-wiki query search <关键词>       # 定位模块
code-wiki query context <module>      # 理解结构
code-wiki query callers <function>    # 追踪调用链

## 开发:改代码前评估影响
code-wiki query impact <module>       # 影响范围
code-wiki query deps / rdeps          # 依赖关系

## 治理:定期检查
code-wiki check arch                  # 架构违规
code-wiki check hotspots              # 热点分析
```

---

## 实战案例

上述是我们 Agent 相关代码库,通过 UModel 构建出的模块和类型的关联关系,VibeCoding 了一个简单的 Demo 便于读者理解

### 案例一:Agent 自主完成变更影响评估

我们向一个子 Agent 下达任务:

评估修改 vibeops-agents 项目中 pkg/a2a 模块的影响范围,包括受影响的模块和入口点、上下游依赖、架构风险。

子 Agent 仅凭 code-wiki CLI 和 Skill,未读取任何源代码文件,完成了以下推理:

```
$ code-wiki query search a2a
Found 9 entities matching a2a:
  [module] pkg/a2a                    LOC:1247  A2A protocol core
  [module] pkg/a2a/adapter            LOC:834   A2A adapter implementations
  [module] pkg/a2a/taskstore          LOC:567   A2A task state management
  ...

$ code-wiki query context pkg/a2a
Module: pkg/a2a
  Types (17): TaskStore, A2AServer, AgentCard, ...
  Functions (52): HandleA2ARequest[entry], StartA2AServer[entry], ...
  Reverse dependencies (9): pkg/api/handler, pkg/server, cmd/vibeops-agents, ...

$ code-wiki query impact pkg/a2a
Impact of pkg/a2a:
  Direct dependents (9): pkg/api/handler, pkg/server, cmd/vibeops-agents, ...
  Entry points affected: HandleA2ARequest, StartA2AServer, main
  Component crossings: a2a-protocol → api, a2a-protocol → server

$ code-wiki check hotspots
Top hotspots by import count:
  1. pkg/a2a/adapter    (48 imports)  ← 高耦合警告
  2. pkg/util/logger     (35 imports)
  3. pkg/config          (32 imports)
```

Agent 最终输出:

```
=== A2A 协议修改影响评估 ===

影响范围:
- 直接依赖模块:9 个(pkg/api/handler, pkg/server 等)
- 受影响入口点:3 个(HandleA2ARequest, StartA2AServer, main)
- 跨组件边界:2 处(a2a-protocol → api, a2a-protocol → server)

风险点:
- pkg/a2a/adapter 被 48 个模块导入,全项目耦合度最高
- bus factor = 1,建议增加 reviewer

建议执行顺序:
1. 先修改 pkg/a2a 核心协议定义
2. 同步更新 pkg/a2a/adapter 的接口实现
3. 修改 pkg/api/handler 的调用方式
4. 更新 pkg/server 的初始化逻辑
5. 回归测试 cmd/vibeops-agents 入口点
```

全程 5 条命令,每条 1-3 秒,总耗时不到 15 秒。Agent 未读取任何源代码文件,仅通过图谱查询完成了完整的影响评估。

### 案例二:RCA:从告警到代码根因

生产告警:`service-vibeops error_rate > 5%`

```
# 1. 从运维实体定位代码模块
$ code-wiki query context pkg/server
Module: pkg/server
  Functions (23): StartServer[entry], handleRequest, applyMiddleware, ...
  Dependencies (12): pkg/a2a, pkg/config, pkg/auth, ...

# 2. 追踪调用链,定位可能出错的下游
$ code-wiki query callees pkg/server.handleRequest
Callees of handleRequest:
  pkg/auth.ValidateToken          [component: auth]
  pkg/a2a.HandleA2ARequest        [component: a2a-protocol]
  pkg/scheduler.DispatchTask      [component: scheduler]

# 3. 结合 commit_log 发现 a2a 模块 2 小时前有变更
#    author=xxx, message=refactor adapter interface

# 4. 确认变更影响
$ code-wiki query impact pkg/a2a
Impact of pkg/a2a:
  Direct dependents (9): pkg/api/handler, pkg/server, ...
  Entry points affected: HandleA2ARequest, StartA2AServer

# → 根因:a2a 接口重构影响了 server 调用链,检查接口兼容性
```

### 案例三:架构治理:发现架构腐化

```
# 1. 扫描架构违规
$ code-wiki check arch
Architecture violations:
  pkg/util/logger calls pkg/api/handler.GetRequestID
    [utility → api] utility 层不应调用 api 层
  pkg/config calls pkg/scheduler.GetDefaultConfig
    [infra → service] 基础设施层不应依赖业务层

# 2. 发现耦合热点
$ code-wiki check hotspots
Top hotspots:
  1. pkg/a2a/adapter      48 imports  [HIGH]
  2. pkg/util/logger       35 imports  [NORMAL]
  3. pkg/scheduler/queue   28 imports  [MEDIUM]

# 3. 深入分析高耦合模块
$ code-wiki query rdeps pkg/a2a/adapter
Reverse dependencies (48):
  pkg/api/* (12 modules), pkg/server/* (8 modules), pkg/scheduler/* (6 modules), ...

# Agent 建议拆分为 adapter/protocol, adapter/transform, adapter/routing
```

---

## 展望

### 全方位数字化评测

我们计划构建一套标准化的代码理解评测基准,覆盖影响分析、调用链追踪、架构违规检测、RCA 根因定位等核心场景,在不同规模的真实代码库上,对比 Model + Bash(Agentic Search)、Model + CodeWiki(LLM 文档)、Model + UModel(知识图谱)三种范式在准确率、召回率、推理步数、Token 消耗等维度的表现。

用 SWE-bench 式的量化评测,让每种范式的能力边界可度量、可复现。在此之上,基于 Benchmark 分数优化整体的技术架构,包括迭代升级相关的 Skills + CLI。

### Agent 自主维护

Agent 不只是图谱的消费者,也可以成为维护者:

- 代码结构变更后,关联的 LLM 推断关系标记为需要重新评估
- 定期巡检孤立实体、缺失关系、过期数据
- 上述能力之上,还需要有一套 Verify、质量评估体系,让自维护可控

### 架构守护门禁

集成到 CI 流程,PR 时自动运行:

```
code-wiki ingest --incremental        # 增量更新图谱
code-wiki check arch                  # 架构违规检查
code-wiki query impact <changed_files> # 变更影响分析
```

### 从可观测到可理解

从可观测数据的建模到代码知识的建模,从用 Entity + Log 描述运行系统,到用 Entity + Log 描述代码系统:UModel 正在从观测 IT 系统走向理解构建 IT 系统的代码和过程。

当 Agent 同时理解了代码的结构、历史以及生产表现,真正意义上的 AI 原生软件工程才成为可能。

---

## 附录

下述是用 Cursor + Opus 4.6 分析 starops-cli 项目重构策略的两次对话,从 Agent 的理解程度来看,叠加了 UModel 的效果要好非常多,一些重要的结构问题都能很好的发现。而原生的分析更多还是停留在表面,废话很多,实际性内容很少。

### 提示词

```
@pkg 现在这个包太乱了,已经不太适合维护了,请根据我们项目的目的和现状,帮我看看如何重构,给我一个你认为最佳的重构方式。告诉我如何做即可。
```

### Cursor + Opus 4.6 (CodeIndex + SubAgents)

```
现在我已经完整了解了 `pkg/` 的全貌。让我整理分析结果并给出重构方案。

---

## `pkg/` 重构方案

### 一、现状诊断

`pkg/` 当前有 **17 个顶层包 + observe 下 10 个子包**,共 93 个文件。核心问题不是「包太多」,而是以下 5 个结构性问题:

**问题 1:职责泄漏 — `auth` 包承载了 observe 的配置**

```go:41:55:pkg/auth/auth.go
type ObserveRerankerConfig struct {
	Endpoint string `json:"endpoint,omitempty" yaml:"endpoint,omitempty"`
	APIKey   string `json:"api_key,omitempty" yaml:"api_key,omitempty"`
	Model    string `json:"model,omitempty" yaml:"model,omitempty"`
}

type ObserveConfig struct {
	Workspace string                `json:"workspace,omitempty" yaml:"workspace,omitempty"`
	Reranker  ObserveRerankerConfig `json:"reranker,omitempty" yaml:"reranker,omitempty"`
}

type Profile struct {
	AuthConfig `yaml:",inline"`
	Observe    ObserveConfig `json:"observe,omitempty" yaml:"observe,omitempty"`
}
```

`ObserveConfig` / `ObserveRerankerConfig` 是 observe 引擎的专属配置,却定义在 `auth` 里,导致 `auth` 从「认证凭据管理」膨胀为「全局配置中心」。

**问题 2:命名冲突 — 三对 "影子包"**

| 全局包 | observe 子包 | 冲突点 |
|--------|-------------|--------|
```

### Cursor + Opus4.6 + UModel

```
好的,我现在已经对整个 `pkg` 包的架构、依赖关系、职责划分有了非常清晰的理解。下面是我的分析和重构方案。

---

## `pkg` 包重构方案

### 一、现状诊断

当前 `pkg/` 有 **18 个子包**(含 `observe/` 下 8 个子子包),总计约 **17,400 行代码**。核心问题:

#### 1. 两套平行的响应/错误/退出机制

这是最严重的问题。目前存在两条互不兼容的输出路径:

| 路径 | 使用者 | 响应类型 | 退出方式 |
|------|--------|----------|----------|
| **CLI 路径**: `pkg/response` + `pkg/hint` + `pkg/output` | cmd/sls, cmd/cms, cmd/umodel, cmd/tool | `response.ErrorResponse` | `response.ExitWithError()` 直接 `os.Exit` |
| **Engine 路径**: `pkg/observe/response.go` + `pkg/observe/output/` + `pkg/exitcode` | cmd/observe/* | `observe.Response` | 返回 `*Response`,由调用方决定退出码 |

**具体冗余**:
- `pkg/exitcode` 只有 15 行,且 `hint.ExitCodeForType()` 已是退出码单一真相源——`exitcode` 完全多余
- `pkg/observe/output/output.go`(65 行)是 `pkg/output` 的薄适配器,仅为 `observe.Response` 做了一层包装
- `pkg/observe/response.go` 的 `Response` 结构和 `pkg/response` 的 `ErrorResponse` 结构本质相同,都是 `{success, error, suggest}` 信封

#### 2. `pkg/observe` 是一个臃肿的整体

`observe` 包本身(顶层文件)承担了太多职责:

| 文件 | 职责 | 行数 |
|------|------|------|
| `executor.go` | 工具执行引擎 | 196 |
| `route.go` | 后端路由分发(SLS/CMS/PromQL/Topo/DataSet) | 401 |
| `entity_resolve.go` | 实体 ID 拓扑解析 | 225 |
| `template.go` | 查询模板渲染 + 13 个模板函数 | 304 |
| `resolve.go` | 参数解析辅助函数 | 111 |
| `response.go` | 响应信封定义 | 69 |
```

---

## 评论

**评论者**: 槿叶  
**评论内容**: "#1 4月20日 大量的runtime branch怎么解决呢"

---

## 关键要点总结

### 为什么向量相似度搜索不到依赖链?

文章中明确指出了这个问题的根本原因:

1. **向量相似度是文本层面的近似匹配,不是结构层面的关系推理**
   - `import pkg/a2a` 在代码里是一条确定性的依赖关系,但在向量空间里只是一个文本片段的相似度信号
   - 找到所有直接或间接依赖 pkg/a2a 的模块需要图遍历,不是向量检索

2. **CodeIndex 的本质局限**
   - CodeIndex 本质上是更聪明的搜索引擎:帮 Agent 找到正确的代码片段塞入上下文,但不替 Agent 做结构化推理
   - 这个接口变更的影响沿调用链传播了几跳需要确定性的 calls 关系,不是语义相似度

3. **为什么逐文件grep不现实**
   - agentic search 可能需要 30+ 轮工具调用和几万 token 才能拼凑出全局依赖图
   - 实际上根本不可能拼出完整的全局图:只能拼出与当前任务相关的局部视图
   - 规模受限:5 万个文件的企业级单仓代价高昂

### 解决方案:代码知识图谱

文章提出的解决方案是构建**代码知识图谱**,具有以下优势:

1. **确定性 vs 概率性**: AST 确定性提取 + SPL/graph-match 任意查询:置信度 1.0 的关系 + 图灵完备的查询语言

2. **代码域 vs 跨域**: EntitySetLink 可以将 code.module 连接到 ops.service、event.alert、req.issue

3. **快照 vs 时间线**: 提供完整的时间维度:commit_log、build_log、deploy_log、test_log、incident_log
