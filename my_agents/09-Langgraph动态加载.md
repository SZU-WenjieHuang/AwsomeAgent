```mermaid
sequenceDiagram
    autonumber
    participant User as 用户请求
    participant TCA as TaskClawAgent
    participant CBT as create_builtin_tools()
    participant CCST as create_call_skill_tool()
    participant CDA as create_deep_agent()
    participant LG as LangGraph 节点图
    participant CST as call_skill_tool
    participant LST as _load_skill_tools()
    participant Cache as _skill_tools_cache

    Note over User,Cache: 阶段一：Agent 初始化（首次请求触发）

    User->>TCA: 首次请求
    activate TCA
    TCA->>TCA: initialize()
    
    Note over TCA: Step 1: 加载 MCP 工具
    TCA->>TCA: _load_mcp_tools()
    
    Note over TCA,CBT: Step 2: 创建内置工具
    TCA->>CBT: create_builtin_tools(skills_dir=...)
    activate CBT
    
    CBT->>CBT: create_ask_clarify_tool()
    CBT->>CBT: create_cron_tools()
    CBT->>CBT: create_time_tool()
    CBT->>CBT: create_skill_reader_tool()
    
    Note over CBT,CCST: 创建 call_skill_tool 元工具
    CBT->>CCST: create_call_skill_tool(skills_dir)
    activate CCST
    
    Note over CCST: 只创建 StructuredTool 实例<br/>不加载任何 Skill 工具<br/>不建立任何索引
    CCST-->>CBT: return call_skill_tool 实例
    deactivate CCST
    
    CBT-->>TCA: return [ask_clarify, cron, time, skill, call_skill_tool]
    deactivate CBT
    
    Note over TCA: self._builtin_tools = [...]
    
    Note over TCA,CDA: Step 3: 创建 LangGraph Agent
    TCA->>TCA: _get_execution_tools()
    TCA->>CDA: create_deep_agent(tools=execution_tools)
    activate CDA
    
    Note over CDA,LG: 工具绑定到 LLM
    CDA->>LG: model.bind_tools(tools)
    CDA->>LG: StateGraph.add_node("tools", tool_node)
    CDA-->>TCA: return CompiledStateGraph
    deactivate CDA
    
    Note over TCA: self.agent = CompiledStateGraph
    TCA-->>User: Agent 初始化完成
    deactivate TCA

    Note over User,Cache: ═══════════════════════════════════════════════════

    Note over User,Cache: 阶段二：用户实际调用 Skill 工具（首次调用某 Skill）

    User->>TCA: "分析崩溃 80298980"
    activate TCA
    TCA->>LG: agent.astream(messages)
    activate LG
    
    Note over LG: LLM 推理，输出 tool_calls
    LG->>LG: llm_node: 生成 tool_calls
    
    Note over LG: tool_calls = [{<br/>  name: "call_skill_tool",<br/>  args: {skill_name: "crash_analysis", tool_name: "crash_router", ...}<br/>}]
    
    LG->>CST: tool_node: 执行 call_skill_tool.invoke(args)
    activate CST
    
    CST->>LST: _load_skill_tools("crash_analysis")
    activate LST
    
    LST->>Cache: 检查缓存: "crash_analysis" in cache?
    
    alt 缓存未命中（首次调用）
        Cache-->>LST: 未找到
        
        Note over LST: 动态导入模块
        LST->>LST: importlib.import_module("skills.crash_analysis")
        
        Note over LST: 查找工厂函数
        LST->>LST: 找到 create_crash_tools()
        
        Note over LST: 创建所有工具实例
        LST->>LST: tools_list = create_crash_tools()
        
        Note over LST,Cache: 存入缓存
        LST->>Cache: _skill_tools_cache["crash_analysis"] = {<br/>  "crash_router": Tool,<br/>  "crash_code": Tool,<br/>  "crash_extractor": Tool,<br/>  ...<br/>}
    else 缓存命中（后续调用）
        Cache-->>LST: 返回已缓存的 tools_dict
    end
    
    LST-->>CST: return tools_dict
    deactivate LST
    
    Note over CST: 从 tools_dict 取出目标工具
    CST->>CST: tool = tools_dict["crash_router"]
    CST->>CST: result = tool.invoke(tool_args)
    CST-->>LG: return "[crash_router] 分析结果..."
    deactivate CST
    
    LG->>LG: 继续 llm_node 处理结果
    LG-->>TCA: yield 流式事件
    deactivate LG
    
    TCA-->>User: 返回分析结果
    deactivate TCA

    Note over User,Cache: ═══════════════════════════════════════════════════

    Note over User,Cache: 阶段三：后续调用同一 Skill（缓存命中）

    User->>TCA: "继续分析另一个崩溃"
    activate TCA
    TCA->>LG: agent.astream(messages)
    activate LG
    LG->>CST: call_skill_tool.invoke(...)
    activate CST
    CST->>LST: _load_skill_tools("crash_analysis")
    activate LST
    LST->>Cache: 检查缓存
    Cache-->>LST: 命中！直接返回 tools_dict
    deactivate LST
    Note over CST: 无需 importlib，直接使用缓存
    CST-->>LG: return 结果
    deactivate CST
    LG-->>TCA: yield 流式事件
    deactivate LG
    TCA-->>User: 返回结果
    deactivate TCA
```

## 关键时间点总结

| 时间点 | 发生什么 | 位置 |
|-------|---------|------|
| **Agent 初始化时** | `call_skill_tool` 元工具被创建并加入 `builtin_tools` | `create_builtin_tools()` |
| **Agent 初始化时** | `call_skill_tool` 被绑定到 LLM (`bind_tools`) | `create_deep_agent()` |
| **Agent 初始化时** | **不建立任何 Skill 索引**，`_skill_tools_cache = {}` 为空 | - |
| **首次调用某 Skill 时** | `importlib.import_module()` 动态导入 Skill 模块 | `_load_skill_tools()` |
| **首次调用某 Skill 时** | 调用 `create_xxx_tools()` 创建所有工具实例 | `_load_skill_tools()` |
| **首次调用某 Skill 时** | 存入 `_skill_tools_cache` 缓存 | `_load_skill_tools()` |
| **后续调用同一 Skill** | 直接从缓存取，无任何导入操作 | `_load_skill_tools()` |

## 结论

```
Agent 初始化时：
┌─────────────────────────────────────┐
│ builtin_tools = [                   │
│   ask_clarify,                      │
│   cron_xxx,                         │
│   time,                             │
│   skill,                            │
│   call_skill_tool  ← 只有这个壳子    │
│ ]                                   │
├─────────────────────────────────────┤
│ _skill_tools_cache = {}  ← 完全空的 │
└─────────────────────────────────────┘

首次调用 crash_analysis 后：
┌─────────────────────────────────────┐
│ _skill_tools_cache = {              │
│   "crash_analysis": {               │
│     "crash_router": <Tool>,         │
│     "crash_code": <Tool>,           │
│     "crash_extractor": <Tool>,      │
│     ...                             │
│   }                                 │
│ }                                   │
└─────────────────────────────────────┘
```

**答案**：Agent 初始化时**完全没有索引**，`_skill_tools_cache` 是空的。只有在**首次调用某个 Skill 时**才会动态导入并缓存该 Skill 的所有工具。