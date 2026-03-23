# Backend-Aone Agent 后端架构文档

> 本文档描述 `backend-aone` 的完整技术架构，涵盖各层次模块职责、数据流、并发模型和用户隔离设计。

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [第一层：API 接入层](#2-第一层api-接入层)
3. [第二层：并发网关层](#3-第二层并发网关层)
4. [第三层：Agent 核心层](#4-第三层agent-核心层)
5. [第四层：中间件层](#5-第四层中间件层)
6. [第五层：工具层](#6-第五层工具层)
7. [第六层：管理器层](#7-第六层管理器层)
8. [第七层：存储与沙箱层](#8-第七层存储与沙箱层)
9. [第八层：模型配置层](#9-第八层模型配置层)
10. [用户数据隔离设计](#10-用户数据隔离设计)
11. [核心请求流程](#11-核心请求流程)

---

## 1. 整体架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                          前端 / 客户端                                │
│                    (React + Ant Design X)                            │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ HTTP / SSE
┌──────────────────────────────▼──────────────────────────────────────┐
│                     第一层：API 接入层                                │
│              FastAPI  server.py / events.py / sse_handler.py        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                     第二层：并发网关层                                │
│         Gateway / CommandQueue / TaskScheduler / ActivityMonitor    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                     第三层：Agent 核心层                              │
│          TaskClawAgent / AgentStream / Prompt / AgentState          │
└────────┬──────────────────────────────────────────────┬────────────┘
         │                                              │
┌────────▼────────┐                          ┌──────────▼──────────┐
│  第四层：中间件  │                          │   第五层：工具层     │
│  UserData       │                          │  内置工具 + MCP工具  │
│  Memory         │                          │  + Skills            │
│  ToolError      │                          └─────────────────────┘
└─────────────────┘
         │
┌────────▼──────────────────────────────────────────────────────────┐
│                     第六层：管理器层                                │
│       ConfigManager / SessionManager / McpManager / SkillManager  │
└────────┬──────────────────────────────────────────────────────────┘
         │
┌────────▼──────────────────────────────────────────────────────────┐
│                   第七层：存储与沙箱层                               │
│         OSSStorage / CheckpointerManager / Sandbox(E2B/Daytona)   │
└────────┬──────────────────────────────────────────────────────────┘
         │
┌────────▼──────────────────────────────────────────────────────────┐
│                   第八层：模型配置层                                 │
│           LLMFactory / ActiveLLMConfig / LLMPreset                │
└───────────────────────────────────────────────────────────────────┘
```

---

## 2. 第一层：API 接入层

### 模块结构

```
server/
├── server.py       # FastAPI 主服务，路由定义，AgentManager
├── events.py       # SSE 事件类型常量定义
└── sse_handler.py  # Agent 流式事件 → SSE 格式转换
```

### 路由分组

```
GET  /                          健康检查

# 会话管理
POST   /api/session             创建新会话
GET    /api/sessions            获取会话列表
GET    /api/session/{id}        获取会话详情
DELETE /api/session/{id}        删除会话

# 核心任务（SSE 流式）
POST   /api/task                流式任务处理（主链路）
POST   /api/chat                非流式聊天（兼容旧接口）

# 模型管理
GET    /api/models/presets      获取所有预设模型
GET    /api/models/active       获取当前激活模型
POST   /api/models/switch       切换模型

# MCP 服务器管理
GET    /api/mcp/list            列出 MCP 服务器
POST   /api/mcp/add             添加 MCP 服务器
PUT    /api/mcp/update/{name}   更新 MCP 服务器
DELETE /api/mcp/delete/{name}   删除 MCP 服务器
POST   /api/mcp/reload          热更新 MCP 工具

# 技能管理
GET    /api/skills              列出技能
POST   /api/skills/create       创建技能
POST   /api/skills/upload       上传 zip/tgz 技能包
PUT    /api/skills/{name}       更新技能
DELETE /api/skills/delete/{name} 删除技能
POST   /api/skills/reload       热更新技能

# 网关 & 调度
GET    /api/gateway/stats       网关状态
GET    /api/gateway/tasks       调度任务列表
POST   /api/gateway/tasks       注册调度任务
DELETE /api/gateway/tasks/{name} 注销调度任务

# 文件 / OSS
GET    /api/files/url           获取文件访问 URL
POST   /api/files/sync          同步本地文件到 OSS
```

### SSE 事件类型

```mermaid
graph LR
    A[Agent 推理事件] --> B{事件类型}
    B --> C[token / content\n流式 token]
    B --> D[tool_start\n工具开始调用]
    B --> E[tool_end\n工具调用结束]
    B --> F[todos\n待办事项更新]
    B --> G[ask_clarify\n澄清问题暂停]
    B --> H[state\nAgent 状态快照]
    B --> I[metadata\n会话元数据]
    B --> J[on_chain_end\n任务完成]
    B --> K[error\n错误事件]
```

### AgentManager 设计

`AgentManager` 是 API 层的核心单例，负责多用户 Agent 实例的生命周期管理：

```mermaid
classDiagram
    class AgentManager {
        +_user_agents: dict[str, TaskClawAgent]
        +_user_session_managers: dict[str, SessionManager]
        +_oss_storage: OSSStorageService
        +initialize()
        +get_or_create_agent(user_id) TaskClawAgent
        +create_session(user_id) str
        +change_user(user_id) str
        +get_mcp_manager(user_id) McpManager
        +get_skill_manager(user_id) SkillManager
        +reload_mcp_tools(user_id) int
        +reload_skills(user_id) int
        +close_all()
    }
```

---

## 3. 第二层：并发网关层

### 模块结构

```
gateway/
├── gateway.py          # 核心网关，并发控制入口
├── command_queue.py    # 用户级命令队列（串行化同用户请求）
├── activity_monitor.py # 系统空闲状态监控
├── task_scheduler.py   # 定时/空闲触发的后台任务调度
├── schedule_loader.py  # 从 schedule.md 加载任务指令
└── handler_factory.py  # 注册预设 handler（stream / stream_collector）
```

### 并发控制模型

```mermaid
graph TD
    A[用户请求] --> B{全局并发检查\nglobal_max_concurrent}
    B -->|超限| C[排队等待]
    B -->|通过| D{用户并发检查\nuser_max_concurrent}
    D -->|超限| E[用户队列等待]
    D -->|通过| F[执行 Handler]
    F --> G{Handler 类型}
    G --> H[stream\n流式任务]
    G --> I[stream_collector\n后台收集任务]
    H --> J[SSE 推流给前端]
    I --> K[结果写入文件]
```

### 后台调度流程

```mermaid
sequenceDiagram
    participant S as TaskScheduler
    participant M as ActivityMonitor
    participant L as ScheduleLoader
    participant A as TaskClawAgent

    S->>L: 加载 schedule.md 任务定义
    loop 每隔 interval 秒
        S->>M: 检查系统是否空闲
        M-->>S: idle=true/false
        alt 空闲模式 & 系统空闲
            S->>A: 执行后台任务
        else 定时模式
            S->>A: 执行后台任务
        end
    end
```

---

## 4. 第三层：Agent 核心层

### 模块结构

```
agents/
├── taskclaw_agent.py  # Agent 主体（生命周期、推理、工具、沙箱）
├── agent_stream.py    # 流式事件迭代器
└── prompt.py          # 系统 Prompt（任务工作流指令）
```

### TaskClawAgent 核心状态

```mermaid
classDiagram
    class TaskClawAgent {
        +config: AppConfig
        +active_llm_config: ActiveLLMConfig
        +sandbox_config: SandboxConfig
        +user_id: str
        +backend_root_dir: str
        +agent: DeepAgent (compiled graph)
        +_mcp_tools: List[BaseTool]
        +_builtin_tools: List[BaseTool]
        +_skills: List[str]
        +_checkpointer_manager: CheckpointerManager
        +initialize()
        +chat(message, thread_id) str
        +stream_with_state(message, thread_id) AsyncIterator
        +resume_with_state(answer, thread_id) AsyncIterator
        +switch_model(new_config)
        +get_active_llm_config() ActiveLLMConfig
        +reload_mcp_tools() int
        +update_skills(paths)
        +close()
    }
```

### Agent 初始化流程

```mermaid
flowchart TD
    A[TaskClawAgent.__init__] --> B[加载 ConfigManager]
    B --> C[build_default_active_config\n从 config.yaml 推断初始 LLM]
    C --> D[initialize 异步初始化]
    D --> E[_load_mcp_tools\n连接 MCP 服务器]
    E --> F[create_builtin_tools\n创建内置工具]
    F --> G{vision_model 已配置?}
    G -->|是| H[create_vision_llm\n创建视觉 LLM]
    G -->|否| I[跳过]
    H --> J[get_skills_list\n加载技能列表]
    I --> J
    J --> K[create_llm_from_active_config\n创建主 LLM]
    K --> L[CheckpointerManager.initialize\n初始化 SQLite Checkpointer]
    L --> M[创建 Middleware 链\nUserData + Memory + ToolError]
    M --> N[create_deep_agent\n编译 DeepAgents 图]
    N --> O[初始化沙箱 Backend\nE2B / Daytona / Local]
    O --> P[Agent 就绪]
```

### 流式推理流程

```mermaid
sequenceDiagram
    participant C as Client
    participant S as server.py
    participant G as Gateway
    participant A as TaskClawAgent
    participant D as DeepAgents
    participant T as Tools/MCP

    C->>S: POST /api/task (SSE)
    S->>G: handle_task(stream_handler)
    G->>A: stream_with_state(message, thread_id)
    A->>D: agent.astream_events(input, config)
    loop 推理循环
        D->>D: LLM 推理
        D-->>A: on_chat_model_stream (token)
        A-->>S: yield token 事件
        S-->>C: SSE data: token
        D->>T: 工具调用
        T-->>D: 工具结果
        D-->>A: on_tool_start / on_tool_end
        A-->>S: yield tool 事件
        S-->>C: SSE data: tool_start/end
    end
    D-->>A: 推理完成
    A-->>S: yield on_chain_end
    S-->>C: SSE data: [DONE]
```

---

## 5. 第四层：中间件层

### 中间件执行顺序

```mermaid
flowchart LR
    A[用户消息] --> B[UserDataMiddleware\n用户目录初始化/隔离]
    B --> C[MemoryMiddleware\n加载长期记忆注入 Prompt]
    C --> D[DeepAgents 推理]
    D --> E[ToolErrorHandlingMiddleware\n捕获工具异常返回 ToolMessage]
    E --> F[MemoryMiddleware\n更新记忆]
    F --> G[流式输出]
```

### 各中间件职责

| 中间件 | 触发时机 | 核心职责 |
|--------|----------|----------|
| `UserDataMiddleware` | 每次请求前 | 确保 `user-data/{user_id}/` 目录存在，初始化用户工作区 |
| `MemoryMiddleware` | 请求前/后 | 从 `memory/` 读取历史摘要注入系统 Prompt；推理后更新记忆 |
| `ToolErrorHandlingMiddleware` | 工具调用异常时 | 捕获异常，构造结构化 `ToolMessage` 返回，防止 Agent 崩溃 |

---

## 6. 第五层：工具层

### 工具分类

```mermaid
graph TD
    A[Agent 可用工具] --> B[内置工具 Builtin Tools]
    A --> C[MCP 工具 External Tools]
    A --> D[Skills 技能]

    B --> B1[ask_clarify_tool\n向用户发起澄清问题]
    B --> B2[cron_tool\n注册/管理定时任务]
    B --> B3[image_understanding_tool\n调用视觉 LLM 理解图片]
    B --> B4[time_tool\n获取当前时间]

    C --> C1[aone-coop\nAone 协作 MCP]
    C --> C2[aone-km\nAone 知识库 MCP]
    C --> C3[aone-log-downloader\n日志下载 MCP]
    C --> C4[... 用户自定义 MCP]

    D --> D1[Markdown 技能文件\n注入系统 Prompt]
    D --> D2[脚本技能\n可执行工具]
```

### MCP 连接协议支持

```mermaid
graph LR
    A[McpManager] --> B{连接类型}
    B --> C[stdio\n子进程模式\ncommand + args]
    B --> D[http\nSSE 传输\nurl + headers]
    B --> E[streamable-http\n流式 HTTP 传输\nurl + headers]
```

### ask_clarify 澄清流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant S as server.py
    participant A as Agent
    participant T as ask_clarify_tool

    A->>T: 调用 ask_clarify(question)
    T-->>S: 发出 ask_clarify SSE 事件
    S-->>U: SSE: ask_clarify {question}
    Note over U,S: 任务暂停，等待用户回复
    U->>S: POST /api/task?resume_answer=xxx
    S->>A: resume_with_state(answer, thread_id)
    A->>A: 注入用户回复，继续推理
    A-->>U: 继续流式输出
```

---

## 7. 第六层：管理器层

### 模块结构

```
managers/
├── config_manager.py   # 配置加载（全局/用户私有双层）
├── session_manager.py  # 会话元数据持久化（SQLite）
├── mcp_manager.py      # MCP 服务器配置 CRUD
└── skill_manager.py    # 技能文件管理（本地 + OSS）
```

### ConfigManager 配置加载优先级

```mermaid
flowchart TD
    A[ConfigManager.load] --> B{用户私有 config.yaml 存在?}
    B -->|是| C[加载用户私有 config.yaml]
    B -->|否| D[加载全局 config.yaml]
    C --> E{config.local.yaml 存在?}
    D --> E
    E -->|是| F[深度合并 local 配置]
    E -->|否| G[跳过]
    F --> H[解析 ${ENV_VAR} 环境变量]
    G --> H
    H --> I[Pydantic 校验]
    I --> J[返回 AppConfig]
```

### SessionManager 数据模型

```mermaid
erDiagram
    SESSION {
        string id PK
        string user_id
        string model
        string status
        datetime created_at
        datetime updated_at
    }
```

### SkillManager 技能同步流程

```mermaid
flowchart LR
    A[全局 skills/\n默认技能模板] -->|首次初始化 copy| B[user-data/uid/skills/\n用户私有技能]
    C[OSS 云端技能] -->|sync_skills_from_oss\n增量同步| B
    D[用户上传 zip/tgz] -->|create_skill_from_bytes| B
    B -->|get_enabled_skill_paths| E[Agent 加载技能]
```

---

## 8. 第七层：存储与沙箱层

### 存储架构

```mermaid
graph TD
    A[Agent 产生的数据] --> B{OSS 是否配置?}
    B -->|是| C[OSSStorageService\n阿里云 OSS]
    B -->|否| D[本地文件系统\nuser-data/uid/]

    E[会话状态 Checkpoint] --> F[CheckpointerManager\nSQLite .db/sessions.db]
    F --> G[LangGraph AsyncSqliteSaver\n支持多会话并发]
```

### 沙箱执行架构

```mermaid
graph TD
    A[Agent 代码执行请求] --> B{sandbox.enabled?}
    B -->|false| C[VirtualShellBackend\n本地文件系统执行]
    B -->|true| D{provider 类型}
    D --> E[E2BProvider\nE2B 云沙箱]
    D --> F[DaytonaProvider\nDaytona 远程沙箱]
    D --> G[LocalProvider\n本地沙箱]
    E --> H[OssSyncBackend 包装\n执行结果自动同步 OSS]
    F --> H
    G --> H
```

---

## 9. 第八层：模型配置层

### LLM 预设体系

```mermaid
graph LR
    A[LLMPreset 枚举] --> B[CODE_LLM\nqwen3-coder:30b]
    A --> C[CODE_NEXT_LLM\nqwen3-coder-next:latest]
    A --> D[QWEN35_LLM\nqwen3.5:latest]
    A --> E[GLM_LLM\nglm-4.7-flash:latest]
    A --> F[GLM5_LLM\nglm-5:cloud]
    A --> G[KIMI_LLM\nkimi-k2.5:cloud]
    A --> H[QWEN_PLUS\nqwen-plus-latest\nDashScope]
    A --> I[CUSTOM\n用户自定义]

    B --> J[AMAP 内部服务\nhttps://pre-map-data-ai.amap.com/v1]
    C --> J
    D --> J
    E --> J
    F --> J
    G --> J
    H --> K[DashScope\nhttps://dashscope.aliyuncs.com]
```

### LLM 创建优先级

```mermaid
flowchart TD
    A[create_llm_from_active_config] --> B[ActiveLLMConfig\n用户显式指定的值]
    B -->|未指定| C[LLM_PRESET_DEFAULTS\n预设默认值]
    C -->|未配置| D[ModelConfig\nconfig.yaml 全局配置]
    D -->|未配置| E[环境变量\nOPENAI_BASE_URL\nOPENAI_API_KEY]
    E --> F[ChatOpenAI 实例]
```

### 模型切换流程

```mermaid
sequenceDiagram
    participant U as 用户/前端
    participant S as server.py
    participant A as TaskClawAgent
    participant C as ConfigManager

    U->>S: POST /api/models/switch {preset, model_name}
    S->>S: 校验 preset 枚举值
    S->>A: agent.switch_model(new_config)
    A->>A: 更新 active_llm_config
    A->>A: create_llm_from_active_config\n重建 LLM 实例
    A->>A: 重新编译 DeepAgents 图
    A-->>S: 切换成功
    alt preset == CUSTOM
        S->>C: save_custom_model\n持久化到用户私有 config.yaml
    end
    S-->>U: {status: ok, active_model: {...}}
```

---

## 10. 用户数据隔离设计

### 目录结构

```
app/backend-aone/
├── config.yaml                    # 全局配置（模板）
├── skills/                        # 全局默认技能（模板）
└── user-data/
    ├── .registry/
    │   └── task_table.json        # 全局调度任务注册表
    └── {user_id}/
        ├── config.yaml            # 用户私有配置（覆盖全局）
        ├── skills/                # 用户私有技能
        │   ├── skill-a/
        │   └── skill-b/
        ├── .db/
        │   └── sessions.db        # 用户会话 + Checkpoint SQLite
        ├── memory/                # 用户长期记忆
        ├── schedule.md            # 用户定时任务定义
        └── sessions/              # 用户任务产出文件
```

### 多用户隔离模型

```mermaid
graph TD
    subgraph 用户A
        A1[AgentManager._user_agents\nuser_a → TaskClawAgent_A]
        A2[user-data/user_a/config.yaml]
        A3[user-data/user_a/.db/sessions.db]
        A4[user-data/user_a/skills/]
        A5[ActiveLLMConfig_A\n独立模型配置]
    end

    subgraph 用户B
        B1[AgentManager._user_agents\nuser_b → TaskClawAgent_B]
        B2[user-data/user_b/config.yaml]
        B3[user-data/user_b/.db/sessions.db]
        B4[user-data/user_b/skills/]
        B5[ActiveLLMConfig_B\n独立模型配置]
    end

    subgraph 共享资源
        C1[全局 config.yaml\n默认配置模板]
        C2[全局 skills/\n默认技能模板]
        C3[OSSStorageService\n共享云存储]
        C4[Gateway\n全局并发控制]
    end

    C1 -.->|首次 copy| A2
    C1 -.->|首次 copy| B2
    C2 -.->|首次 copy| A4
    C2 -.->|首次 copy| B4
```

---

## 11. 核心请求流程

### 完整 SSE 任务请求链路

```mermaid
sequenceDiagram
    participant U as 前端用户
    participant API as FastAPI server.py
    participant GM as AgentManager
    participant GW as Gateway
    participant AG as TaskClawAgent
    participant MW as Middleware 链
    participant DA as DeepAgents
    participant MCP as MCP 工具
    participant OSS as OSS 存储

    U->>API: POST /api/task?task_content=xxx&user_id=yyy
    API->>API: 参数校验
    API->>GM: get_or_create_agent(user_id)
    GM-->>API: TaskClawAgent 实例
    API->>GM: create_session(user_id)
    GM-->>API: session_id
    API-->>U: SSE: metadata {session_id}

    API->>GW: handle_task(stream_handler)
    GW->>GW: 并发检查（全局 + 用户级）
    GW->>AG: stream_with_state(message, thread_id)

    AG->>MW: UserDataMiddleware.before()
    MW->>MW: MemoryMiddleware.before()\n加载历史记忆注入 Prompt
    MW->>DA: astream_events(input, config)

    loop Agent 推理循环
        DA->>DA: LLM 推理（ChatOpenAI）
        DA-->>AG: token 事件
        AG-->>API: yield SSE token
        API-->>U: SSE: data token

        opt 需要工具调用
            DA->>MCP: 调用 MCP 工具
            MCP-->>DA: 工具结果
            DA-->>AG: tool_start / tool_end 事件
            AG-->>API: yield SSE tool 事件
            API-->>U: SSE: data tool_start/end
        end

        opt 需要澄清
            DA->>AG: ask_clarify 事件
            AG-->>U: SSE: ask_clarify {question}
            Note over U,AG: 任务暂停
            U->>API: POST /api/task?resume_answer=xxx
            API->>AG: resume_with_state(answer)
        end
    end

    DA-->>AG: 推理完成
    MW->>MW: MemoryMiddleware.after()\n更新长期记忆
    AG-->>API: yield on_chain_end
    API-->>U: SSE: on_chain_end
    API-->>U: SSE: [DONE]

    opt OSS 已配置
        AG->>OSS: 同步任务产出文件
    end
```

---

## 附录：技术栈

| 层次 | 技术选型 |
|------|----------|
| Web 框架 | FastAPI + Uvicorn |
| Agent 推理框架 | DeepAgents (基于 LangGraph) |
| LLM 接入 | LangChain OpenAI (`ChatOpenAI`) |
| 数据校验 | Pydantic v2 |
| 会话持久化 | SQLite (AsyncSqliteSaver) |
| 外部工具协议 | MCP (stdio / http / streamable-http) |
| 云存储 | 阿里云 OSS |
| 沙箱执行 | E2B / Daytona / Local |
| 流式传输 | SSE (Server-Sent Events) |
| 配置管理 | YAML + 环境变量 (`${VAR}` 语法) |
| 前端 | React + Ant Design X |
