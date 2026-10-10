---
title: Hermes 工具系统深度拆解
source: https://poizon.feishu.cn/wiki/X490wH4a3iwAxHkthgMc8htInCb?from=from_copylink
document_id: QR07ddgdBomxIbxNeR3cVIBonVf
revision_id: 6
captured_at: 2026-10-10
---
# Hermes 工具系统深度拆解

> 目标读者：第一次接触 Hermes Agent，但想真正搞懂“工具系统为什么这样设计、怎么设计、生命周期是什么、字段有什么用、面试怎么讲”的工程同学。

## 本次讲解覆盖范围

我会优先讲：

1. 工具系统在 Hermes 中承担什么职责
2. 工具从“代码文件”到“模型可调用 schema”的完整链路
3. 工具从“模型返回 tool_calls”到“执行结果回填模型”的完整链路
4. `ToolEntry`、`registry.register()`、`get_tool_definitions()`、`handle_function_call()` 等核心字段和函数
5. 为什么要有 toolset、check_fn、缓存、generation、tool_search、插件/MCP 动态注册
6. 一个工具从创建、注册、暴露、调用、不可用、注销的生命周期
7. 小白类比、理解校准、面试题

这次先不展开但已识别的关键支线：

- 具体工具内部实现：例如 browser、terminal、cronjob 每个都很大，可以单独拆。
- 模型适配层：不同 provider 对 tool schema 和 tool_calls 的格式有差异，这里只讲工具系统主链路。
- 审批/安全细节：这里只讲它们在链路中的位置，完整安全策略可以另开一篇。

## 一句话总览

Hermes 的工具系统不是“模型想调用哪个函数，主循环里写一堆 if/else 去执行哪个函数”。

它更像一个小型插件运行时：

```Plaintext
flowchart TD
    A["tools/*.py 工具文件"] --> B["registry.register() 自注册"]
    B --> C["ToolRegistry 保存 ToolEntry"]
    C --> D["model_tools.get_tool_definitions()"]
    D --> E["toolset / check_fn / config 过滤"]
    E --> F["OpenAI-style function schema"]
    F --> G["模型 API 请求"]
    G --> H["模型返回 tool_calls"]
    H --> I["conversation_loop 校验名称和参数"]
    I --> J["tool_executor 调度执行"]
    J --> K["model_tools.handle_function_call()"]
    K --> L["registry.dispatch() 找 handler"]
    L --> M["工具 handler 执行"]
    M --> N["tool result message 回填给模型"]
```

小白类比：

> 你可以把 Hermes 想成一家大型公司。模型是“项目经理”，工具是“各部门员工”，registry 是“公司通讯录”，toolset 是“部门权限分组”，schema 是“每个员工的服务说明书”，tool_executor 是“派单系统”。项目经理不能随便凭空叫人干活，它只能根据通讯录里公开的服务说明书发工单。

## 阅读导航

如果你想快速建立全局理解，先读：

1. `一句话总览`
2. `第一层：工具如何注册`
3. `第二层：工具如何变成模型可调用的 schema`
4. `第四层：模型返回 tool_calls 后发生什么`
5. `第十层：完整生命周期`

如果你要准备面试，重点读：

1. `为什么这样设计`
2. `代价和风险`
3. `增强章节 E：风险与防护`
4. `面试官追问`
5. `增强章节 F：更完整的面试题`

如果你要改代码，重点读：

1. `增强章节 A：工具注册表的工程细节`
2. `增强章节 B：toolset 过滤和 schema 缓存`
3. `增强章节 C：模型 tool_calls 的执行细节`
4. `增强章节 D：插件和 MCP 动态工具`

## 目录地图

```Plaintext
hermes-agent-main/
  tools/
    registry.py              # 中央工具注册表：保存工具元数据、生成 schema、分发执行
    skill_manager_tool.py    # skill_manage 工具示例
    file_tools.py            # read_file / write_file / patch / search_files 示例
    terminal_tool.py         # terminal 工具示例
    tool_search.py           # 工具太多时的渐进式暴露机制
    mcp_tool.py              # MCP 动态工具注册
  model_tools.py             # 工具系统编排层：发现工具、生成模型 schema、dispatch 入口
  toolsets.py                # 工具集定义：web/file/browser/skills 等
  agent/
    agent_init.py            # Agent 初始化时加载工具，生成 valid_tool_names
    conversation_loop.py     # 模型返回 tool_calls 后的校验与循环控制
    tool_executor.py         # 顺序/并发执行工具，处理中断、guardrails、结果回填
    tool_dispatch_helpers.py # 构造 tool result message 等辅助逻辑
  hermes_cli/
    plugins.py               # 插件注册工具、hook、平台能力等
```

## 第一层：工具如何注册

### 代码片段 A：工具注册表的职责

位置：tools/registry.py

```Python
class ToolEntry:
    """Metadata for a single registered tool."""

    __slots__ = (
        "name", "toolset", "schema", "handler", "check_fn",
        "requires_env", "is_async", "description", "emoji",
        "max_result_size_chars", "dynamic_schema_overrides",
    )
```

中文注释版理解：

```Python
class ToolEntry:
    # 一个 ToolEntry 就是一张“工具档案卡”
    # name: 工具名，模型调用时用这个名字
    # toolset: 工具属于哪个工具集，例如 skills/file/browser
    # schema: 给模型看的 JSON Schema，说明参数怎么填
    # handler: 真正执行工具的 Python 函数
    # check_fn: 当前环境是否可用，例如 API key/依赖是否存在
    # requires_env: 这个工具依赖哪些环境变量
    # is_async: handler 是否是 async
    # description: 给 UI 或文档用的描述
    # emoji: CLI 展示工具执行状态用
    # max_result_size_chars: 这个工具返回结果最多保留多少字符
    # dynamic_schema_overrides: 每次生成 schema 时动态覆盖字段
```

这段设计的关键点是：Hermes 没有只保存 `name -> function`，而是保存“工具运行所需的完整元数据”。这样工具不只是能执行，还能被筛选、展示、限流、动态改 schema、做可用性检查。

### ToolEntry 字段逐个解释

| 字段 | 来自哪里 | 用在哪里 | 为什么需要 |
|-|-|-|-|
| `name` | `registry.register(name=...)` | 模型 tool_call、dispatch 查找 | 工具唯一标识 |
| `toolset` | 注册时指定 | toolset 过滤、权限控制 | 让不同场景只暴露部分工具 |
| `schema` | 工具文件内定义 | 给模型 API 的 tools 参数 | 告诉模型怎么调用 |
| `handler` | 注册时传入函数 | `registry.dispatch()` | 真正执行业务逻辑 |
| `check_fn` | 可选 | `get_definitions()` | 当前不可用就别暴露给模型 |
| `requires_env` | 可选 | UI/doctor/工具配置 | 告诉用户缺什么配置 |
| `is_async` | 可选 | `dispatch()` | async handler 需要事件循环桥接 |
| `description` | schema 或注册参数 | UI/工具列表 | 人类可读说明 |
| `emoji` | 注册参数 | CLI 进度展示 | 提升可读性 |
| `max_result_size_chars` | 注册参数 | 工具结果预算 | 防止撑爆上下文 |
| `dynamic_schema_overrides` | 可选 callable | `get_definitions()` | schema 随配置变化 |

### 代码片段 B：工具模块如何自注册

以 `skill_manage` 为例，位置：tools/skill_manager_tool.py

```Python
registry.register(
    name="skill_manage",
    toolset="skills",
    schema=SKILL_MANAGE_SCHEMA,
    handler=lambda args, **kw: skill_manage(
        action=args.get("action", ""),
        name=args.get("name", ""),
        content=args.get("content"),
        category=args.get("category"),
        file_path=args.get("file_path"),
        file_content=args.get("file_content"),
        old_string=args.get("old_string"),
        new_string=args.get("new_string"),
        replace_all=args.get("replace_all", False),
        absorbed_into=args.get("absorbed_into")),
    emoji="📝",
)
```

中文注释版理解：

```Python
registry.register(
    name="skill_manage",          # 模型看到并调用的工具名
    toolset="skills",             # 归到 skills 工具集，方便统一启用/禁用
    schema=SKILL_MANAGE_SCHEMA,   # 参数说明书，给模型 API 用
    handler=lambda args, **kw: skill_manage(
        action=args.get("action", ""),
        name=args.get("name", ""),
        # 其余字段同样从 args 中取出
    ),
                                  # 真正执行时，从 args 取字段，再调用 Python 函数
    emoji="📝",                   # 展示用，不影响逻辑
)
```

这就是“工具不是 skill”的原因：`skill_manage` 是代码里的工具；skill 是它管理的 Markdown 知识文件。

### 代码片段 C：自动发现工具模块

位置：tools/registry.py

```Python
def discover_builtin_tools(tools_dir: Optional[Path] = None) -> List[str]:
    """Import built-in self-registering tool modules and return their module names."""
    tools_path = Path(tools_dir) if tools_dir is not None else Path(__file__).resolve().parent
    module_names = [
        f"tools.{path.stem}"
        for path in sorted(tools_path.glob("*.py"))
        if path.name not in {"__init__.py", "registry.py", "mcp_tool.py"}
        and _module_registers_tools(path)
    ]

    imported: List[str] = []
    for mod_name in module_names:
        try:
            importlib.import_module(mod_name)
            imported.append(mod_name)
        except Exception as e:
            logger.warning("Could not import tool module %s: %s", mod_name, e)
    return imported
```

逐段拆解：

1. 扫描 `tools/*.py`。
2. 排除 `__init__.py`、`registry.py`、`mcp_tool.py`。
3. 只导入顶层包含 `registry.register(...)` 的文件。
4. Python 模块被 import 时，文件底部的 `registry.register()` 自动执行。
5. 工具就进入中央注册表。

为什么要先用 AST 检测 `_module_registers_tools()`？

因为不是 `tools/` 下每个文件都是工具模块。有些是辅助模块。如果全量 import，可能触发重依赖、网络初始化或副作用。Hermes 只导入真正顶层注册工具的文件，减少启动风险。

## 第二层：工具如何变成模型可调用的 schema

### 代码片段 D：Agent 初始化时加载工具

位置：agent/agent_init.py

```Python
agent.tools = _ra().get_tool_definitions(
    enabled_toolsets=enabled_toolsets,
    disabled_toolsets=disabled_toolsets,
    quiet_mode=agent.quiet_mode,
)

agent.valid_tool_names = set()
if agent.tools:
    agent.valid_tool_names = {tool["function"]["name"] for tool in agent.tools}
```

中文解释：

1. Agent 初始化时不直接读取 registry，而是调用 `get_tool_definitions()`。
2. 返回值是模型 API 可接受的工具 schema 列表。
3. `valid_tool_names` 是一份“本 session 真正允许调用的工具名集合”。
4. 后面模型如果幻觉出不存在或未授权工具，就会被 `valid_tool_names` 拦住。

### 代码片段 E：toolset 如何控制工具暴露

位置：model_tools.py

```Python
def get_tool_definitions(
    enabled_toolsets: Optional[List[str]] = None,
    disabled_toolsets: Optional[List[str]] = None,
    quiet_mode: bool = False,
    skip_tool_search_assembly: bool = False,
) -> List[Dict[str, Any]]:
    ...
    result = _compute_tool_definitions(
        enabled_toolsets,
        disabled_toolsets,
        quiet_mode,
        skip_tool_search_assembly=skip_tool_search_assembly,
    )
```

位置：toolsets.py

```Python
TOOLSETS = {
    "skills": {
        "description": "Access, create, edit, and manage skill documents with specialized instructions and knowledge",
        "tools": ["skills_list", "skill_view", "skill_manage"],
        "includes": []
    },
    "file": {
        "description": "File manipulation tools: read, write, patch (with fuzzy matching), and search (content + files)",
        "tools": ["read_file", "write_file", "patch", "search_files"],
        "includes": []
    },
}
```

根据真实代码组装逻辑还原，一个 session 可能传入：

```JSON
{
  "enabled_toolsets": ["skills", "file"],
  "disabled_toolsets": ["browser"]
}
```

最终先解析为候选工具：

```JSON
{
  "tools_to_include": [
    "skills_list",
    "skill_view",
    "skill_manage",
    "read_file",
    "write_file",
    "patch",
    "search_files"
  ]
}
```

再经过 registry 可用性检查，才生成最终模型 schema。

### 为什么要有 toolset

如果没有 toolset，只有“所有工具都给模型”或“手动列工具名”，会有几个问题：

1. 权限太粗：Telegram、cron、子 Agent、主 CLI 的权限不应该一样。
2. prompt 太大：工具 schema 很占上下文。
3. 安全风险：不该让 webhook 或自动任务拿到 terminal/file 等高危工具。
4. 配置难维护：用户不想一个个工具勾选。

toolset 是“工具权限包”。例如：

```Plaintext
webhook-safe 工具集：只给 web_search / web_extract / vision / clarify
skills 工具集：只给 skills_list / skill_view / skill_manage
browser 工具集：给浏览器操作工具
```

## 第三层：registry 如何生成 OpenAI-style function schema

### 代码片段 F：`registry.get_definitions()`

位置：tools/registry.py

```Python
def get_definitions(self, tool_names: Set[str], quiet: bool = False) -> List[dict]:
    result = []
    check_results: Dict[Callable, bool] = {}
    entries_by_name = {entry.name: entry for entry in self._snapshot_entries()}
    for name in sorted(tool_names):
        entry = entries_by_name.get(name)
        if not entry:
            continue
        if entry.check_fn:
            if entry.check_fn not in check_results:
                check_results[entry.check_fn] = _check_fn_cached(entry.check_fn)
            if not check_results[entry.check_fn]:
                continue
        schema_with_name = {**entry.schema, "name": entry.name}
        if entry.dynamic_schema_overrides is not None:
            overrides = entry.dynamic_schema_overrides()
            if isinstance(overrides, dict):
                schema_with_name.update(overrides)
        result.append({"type": "function", "function": schema_with_name})
    return result
```

逐段拆解：

1. `tool_names` 是 toolset 解析后的候选工具名。
2. `entries_by_name` 从 registry 快照拿工具档案。
3. 如果工具有 `check_fn`，先判断当前环境是否支持。
4. `check_fn` 失败则不暴露给模型。
5. schema 必须带 `name`。
6. 如果工具 schema 需要根据运行时配置变化，则调用 `dynamic_schema_overrides()`。
7. 最终包成：

```JSON
{
  "type": "function",
  "function": {
    "name": "skill_manage",
    "description": "...",
    "parameters": {
      "type": "object",
      "properties": {}
    }
  }
}
```

### check_fn 的意义

例如一个浏览器工具可能依赖 Playwright，一个 Home Assistant 工具依赖 token，一个 terminal 后端可能依赖 Docker/Modal。

如果环境不满足，最好的策略不是“暴露给模型，然后执行时报错”，而是“根本不让模型看到它”。这样可以减少幻觉调用和失败循环。

### 缓存和 generation

Hermes 有两层缓存：

1. `registry.py` 里 `check_fn` 结果 TTL 约 30 秒。
2. `model_tools.py` 里 `get_tool_definitions()` 按参数和 `registry._generation` 缓存。

`registry._generation` 的作用：

```Plaintext
每当 register / deregister / alias 变化
  generation += 1
  旧工具 schema 缓存自动失效
```

小白类比：

> 通讯录每改一次版本号就 +1。你手里的旧通讯录如果版本号不对，就要重新拿新版。

## 第四层：模型返回 tool_calls 后发生什么

### 代码片段 G：conversation loop 校验工具名

位置：agent/conversation_loop.py

```Python
if assistant_message.tool_calls:
    for tc in assistant_message.tool_calls:
        if tc.function.name not in agent.valid_tool_names:
            repaired = agent._repair_tool_call(tc.function.name)
            if repaired:
                tc.function.name = repaired
    invalid_tool_calls = [
        tc.function.name for tc in assistant_message.tool_calls
        if tc.function.name not in agent.valid_tool_names
    ]
```

中文解释：

1. 模型返回 tool_calls 后，Hermes 不会马上执行。
2. 先检查工具名是不是本 session 允许的。
3. 如果工具名轻微拼错，尝试修复。
4. 如果仍然无效，就构造错误 tool message 反馈给模型，让模型自己纠正。
5. 连续无效达到上限后，本轮停止为 partial，避免无限循环。

根据真实代码逻辑还原，模型可能返回：

```JSON
{
  "tool_calls": [
    {
      "id": "call_123",
      "function": {
        "name": "skill_managee",
        "arguments": "{\"action\":\"patch\"}"
      }
    }
  ]
}
```

Hermes 会尝试修复成：

```JSON
{
  "function": {
    "name": "skill_manage"
  }
}
```

如果修不好，会回给模型：

```JSON
{
  "role": "tool",
  "name": "skill_managee",
  "tool_call_id": "call_123",
  "content": "Tool 'skill_managee' does not exist. Available tools: ..."
}
```

### 代码片段 H：执行器调用 dispatcher

位置：agent/tool_executor.py

```Python
function_result = _ra().handle_function_call(
    function_name,
    function_args,
    effective_task_id,
    tool_call_id=tool_call.id,
    session_id=agent.session_id or "",
    turn_id=getattr(agent, "_current_turn_id", "") or "",
    api_request_id=getattr(agent, "_current_api_request_id", "") or "",
    enabled_tools=list(agent.valid_tool_names) if agent.valid_tool_names else None,
    skip_pre_tool_call_hook=True,
    enabled_toolsets=getattr(agent, "enabled_toolsets", None),
    disabled_toolsets=getattr(agent, "disabled_toolsets", None),
)
```

关键参数解释：

| 参数 | 来源 | 作用 |
|-|-|-|
| `function_name` | 模型 tool_call | 要执行的工具 |
| `function_args` | tool_call.arguments JSON 解析后 | 工具参数 |
| `effective_task_id` | 当前任务 ID | 隔离 terminal/browser 等状态 |
| `tool_call_id` | 模型返回 | 回填 tool message 时对应原调用 |
| `session_id` | 当前会话 | 日志、hook、观测 |
| `turn_id` | 当前轮次 | 观测与追踪 |
| `api_request_id` | 当前 API 请求 | 观测与追踪 |
| `enabled_tools` | `valid_tool_names` | execute_code 等沙箱工具范围 |
| `enabled_toolsets` | 当前 session 配置 | tool_search bridge 范围控制 |
| `disabled_toolsets` | 当前 session 配置 | 同上 |

## 第五层：`handle_function_call()` 和真正 dispatch

### 代码片段 I：分发入口

位置：model_tools.py

```Python
def handle_function_call(
    function_name: str,
    function_args: Dict[str, Any],
    task_id: Optional[str] = None,
    # 省略 session_id、turn_id、enabled_toolsets 等追踪和权限参数
) -> str:
    function_args = coerce_tool_args(function_name, function_args)
    if not isinstance(function_args, dict):
        function_args = {}
    # 处理 tool_search bridge、plugin hook、审批、observability 后再 dispatch
    result = _dispatch(function_args)
```

这里先做参数类型修正。例如模型常把数字、布尔、数组写成字符串：

```JSON
{
  "limit": "10",
  "replace_all": "true",
  "paths": "README.md"
}
```

`coerce_tool_args()` 会参考工具 schema，把它尽量修成：

```JSON
{
  "limit": 10,
  "replace_all": true,
  "paths": ["README.md"]
}
```

### 代码片段 J：registry.dispatch

位置：tools/registry.py

```Python
def dispatch(self, name: str, args: dict, **kwargs) -> str:
    entry = self.get_entry(name)
    if not entry:
        return json.dumps({"error": f"Unknown tool: {name}"})
    try:
        if entry.is_async:
            from model_tools import _run_async
            return _run_async(entry.handler(args, **kwargs))
        return entry.handler(args, **kwargs)
    except Exception as e:
        logger.exception("Tool %s dispatch error: %s", name, e)
        raw = f"Tool execution failed: {type(e).__name__}: {e}"
        ...
        return json.dumps({"error": sanitized})
```

逐段拆解：

1. 根据 `name` 找 `ToolEntry`。
2. 找不到就返回 JSON 错误。
3. 如果是 async handler，用 `_run_async()` 桥接。
4. 否则直接调用 `entry.handler(args, **kwargs)`。
5. 所有异常都捕获，并清洗错误文本，避免异常内容污染模型上下文。

## 第六层：工具结果如何回到模型

模型调用工具不是一次性结束。工具执行结果会作为新的 `tool` message 加回 messages，然后模型继续推理。

根据 OpenAI-style tool call 结构，一个工具结果大致是：

```JSON
{
  "role": "tool",
  "name": "skill_manage",
  "tool_call_id": "call_123",
  "content": "{\"success\": true, \"message\": \"Skill patched.\"}"
}
```

为什么必须有 `tool_call_id`？

因为一轮里模型可能同时调用多个工具。`tool_call_id` 用来告诉模型：这个结果对应刚才哪个调用。

## 第七层：顺序执行、并发执行和中断

Hermes 的 `agent/tool_executor.py` 同时支持：

- `execute_tool_calls_sequential`
- `execute_tool_calls_concurrent`

并发执行的意义：

1. 模型一轮可能发出多个互不依赖的工具调用。
2. 例如同时读多个文件、查多个状态、跑多个轻量查询。
3. 并发可以减少等待时间。

但并发也有代价：

1. 写文件类工具并发可能冲突。
2. terminal/process 类工具需要 task_id 隔离。
3. 结果必须按原 tool_call 顺序回填，否则模型对不上。
4. 中断、异常和 hook 都要能正确处理。

Hermes 因此在执行前会做：

- interrupt 检查
- tool_search unwrap
- guardrail 检查
- pre/post hook
- 结果预算控制
- tool result message 构造

## 第八层：tool_search 为什么存在

当 MCP 和插件工具很多时，如果把所有工具 schema 都塞给模型，会有三个问题：

1. prompt token 爆炸。
2. 模型更容易调用错工具。
3. 工具列表动态变化时，维护成本高。

Hermes 的做法是：核心工具直接暴露，非核心的大量工具可以被折叠到三个桥接工具后面：

```Plaintext
tool_search   # 搜索可用工具
tool_describe # 查看某个工具 schema
tool_call     # 调用被折叠的真实工具
```

小白类比：

> 不把全公司几千个员工的详细简历都塞给项目经理，只给一个“人事系统”。项目经理先搜“谁会修空调”，再看这个人的服务说明，最后通过系统派单。

关键安全点：

`tool_call` 不能绕过 session 权限。Hermes 在 `handle_function_call()` 和 `tool_executor` 里都会检查 scoped deferrable names，确保受限 session 不能通过桥接工具调用本不该拥有的工具。

## 第九层：插件和 MCP 动态工具

Hermes 的内置工具之外，还有两类动态扩展：

1. plugin 工具：由 `hermes_cli/plugins.py` 注册。
2. MCP 工具：由 `tools/mcp_tool.py` 根据外部 MCP server 动态注册。

动态扩展需要解决的问题：

| 问题 | Hermes 的处理 |
|-|-|
| 外部工具和内置工具重名 | 默认拒绝 shadow，除非显式 `override=True` |
| MCP 工具列表变化 | 支持 deregister 后重新注册 |
| 工具集名称不稳定 | 支持 toolset alias |
| schema 缓存过期 | registry mutation 时 `_generation += 1` |
| 工具不可用 | check_fn / startup discovery / 动态刷新 |

## 第十层：完整生命周期

### 生命周期图

```Plaintext
stateDiagram-v2
    [*] --> CodeDefined: 工具文件定义 schema + handler
    CodeDefined --> Registered: registry.register()
    Registered --> Discoverable: discover_builtin_tools/import plugin/MCP
    Discoverable --> Filtered: get_tool_definitions 解析 toolset
    Filtered --> Available: check_fn 通过
    Filtered --> Hidden: check_fn 失败或 toolset 禁用
    Available --> Exposed: schema 进入 agent.tools
    Exposed --> Called: 模型返回 tool_call
    Called --> Validated: 工具名/JSON/权限校验
    Validated --> Executing: tool_executor 调度
    Executing --> Dispatched: registry.dispatch()
    Dispatched --> Completed: handler 返回 JSON/string
    Completed --> Returned: tool message 回填模型
    Returned --> [*]
    Registered --> Deregistered: MCP 刷新/插件卸载/动态注销
    Deregistered --> [*]
```

### 生命周期每一步说明

| 阶段 | 发生位置 | 输入 | 输出 | 失败时 |
|-|-|-|-|-|
| 定义 | `tools/*.py` | schema、handler | Python 模块 | schema 写错会影响模型调用 |
| 注册 | `registry.register()` | name/toolset/schema/handler | ToolEntry | 重名可能被拒绝 |
| 发现 | `discover_builtin_tools()` | tools 目录 | import 模块 | import 失败记录 warning |
| 过滤 | `get_tool_definitions()` | enabled/disabled toolsets | 候选工具名 | unknown toolset warning |
| 可用性检查 | `registry.get_definitions()` | check_fn | 最终 schema | check_fn false 则隐藏 |
| 暴露 | `agent_init.py` | schema list | `agent.tools` / `valid_tool_names` | 无工具则模型无工具可用 |
| 调用 | 模型 API | tool_calls | function name/args | 模型可能幻觉 |
| 校验 | `conversation_loop.py` | tool_calls | validated calls | 错误回给模型重试 |
| 执行 | `tool_executor.py` | validated calls | function_result | interrupt/guardrail 可中止 |
| 分发 | `model_tools.py` / `registry.py` | name/args | handler result | 异常转 JSON error |
| 回填 | `tool_dispatch_helpers.py` 等 | result | tool message | 超预算会裁剪/持久化 |
| 注销 | `registry.deregister()` | tool name | registry 变化 | generation 更新 |

## 为什么这样设计

### 1. 解耦

工具定义、工具选择、模型调用、工具执行分开：

- `tools/*.py` 只关心“我是什么工具”
- `registry.py` 只关心“有哪些工具”
- `model_tools.py` 只关心“哪些工具给模型看、怎么 dispatch”
- `conversation_loop.py` 只关心“模型返回了什么”
- `tool_executor.py` 只关心“如何执行和回填”

好处是新增工具不用改主循环，插件/MCP 也能接入。

### 2. 权限边界

toolset 让 Hermes 可以按平台/任务限制工具：

- webhook 只给安全工具
- cron 禁用交互类工具
- 子 Agent 只给必要工具
- gateway 平台可单独配置工具集

### 3. 降低模型幻觉

不可用工具不暴露，工具太多就折叠进 tool_search，schema 描述会根据实际可用工具动态改写。

### 4. 支持长生命周期进程

Gateway 是长期运行的。如果每轮都重新探测 Docker、MCP、Playwright、插件，会慢且不稳定。所以需要缓存、generation、TTL、动态刷新。

### 5. 可观测和可治理

tool_call_id、turn_id、api_request_id、pre/post hook、duration_ms 让工具调用可以被日志、插件、审批、安全策略观察。

## 代价和风险

| 设计 | 好处 | 代价 |
|-|-|-|
| 自注册 | 扩展方便 | import 有副作用风险 |
| toolset | 权限清晰 | 配置复杂度上升 |
| check_fn 缓存 | 性能好 | 环境变化最多延迟 30 秒感知 |
| tool_search | 节省 token | 多一层桥接，调试复杂 |
| 动态 MCP | 扩展强 | schema 质量不可控 |
| 并发执行 | 更快 | 写操作冲突风险 |
| 自动参数修正 | 容错强 | 可能掩盖模型输出质量问题 |

## 理解校准

你的理解可以先这样表述：

> Hermes 工具系统就是把 Python 函数注册成模型可调用的 JSON Schema，模型调用后再根据名字找到函数执行。

这个理解对的地方：

- 工具确实最终会变成 JSON Schema 给模型。
- 模型确实通过 name + arguments 调用。
- 执行时确实靠 registry 找 handler。

不完整的地方：

- 它不只是函数映射，还有 toolset 权限、check_fn 可用性、schema 动态化、缓存、MCP/插件、guardrails、并发执行、结果预算。
- 模型看到的工具列表不等于 registry 里所有工具。
- tool_search 可能让模型先看不到真实工具，而是通过桥接搜索后调用。

它会失效的场景：

- 工具被注册了但 toolset 没启用，模型看不到。
- 工具 check_fn 失败，模型看不到。
- MCP 工具刷新后旧 schema 缓存失效，工具集合会变。
- 受限子 Agent 不能通过 tool_call 桥接越权。

## 面试官追问

### 问题 1：Hermes 为什么不用一个大 if/else 来执行工具？

面试官想考什么：

- 是否理解注册表模式和插件化扩展。

简洁回答：

- 因为工具数量多、来源多，包含内置工具、插件工具、MCP 动态工具。用 registry 可以把工具定义、发现、过滤、执行解耦，新增工具只需自注册，不需要改 agent 主循环。

小白版解释：

- 大 if/else 像老板记住每个员工手机号；registry 像公司通讯录。公司人多、还会有人入职离职时，通讯录比老板脑子可靠。

### 问题 2：`schema` 和 `handler` 分别是什么？

标准答案：

- `schema` 是给模型看的调用说明，告诉模型工具名、描述和参数结构。`handler` 是 Python 侧真正执行工具逻辑的函数。

容易答错：

- 以为 schema 会执行代码。schema 不执行，它只是说明书。

### 问题 3：为什么要有 `check_fn`？

标准答案：

- `check_fn` 用来判断当前环境是否满足工具运行条件。不满足时工具不进入模型 schema，减少无效调用和失败循环。

深入答案：

- 对长期运行的 gateway，check_fn 结果还做 TTL 缓存，避免每轮重复探测外部依赖。

### 问题 4：`registry._generation` 有什么用？

标准答案：

- 它是工具注册表的版本号。register、deregister、alias 变化时递增，用来让 `get_tool_definitions()` 的缓存自动失效。

小白版解释：

- 通讯录更新了版本号，旧通讯录就不能继续当最新版用。

### 问题 5：tool_search 解决什么问题？

标准答案：

- 当插件/MCP 工具很多时，直接暴露全部 schema 会占用大量上下文并增加误调用。tool_search 把非核心工具折叠到搜索/描述/调用桥接工具后面，实现渐进式暴露。

容易答错：

- 以为 tool_search 是普通 web search。它搜索的是工具目录，不是互联网。

### 问题 6：模型调用不存在的工具，Hermes 怎么处理？

标准答案：

- `conversation_loop.py` 先用 `valid_tool_names` 校验，尝试 `_repair_tool_call()` 修复轻微名称错误；修复失败则回填错误 tool message 给模型重试，超过次数停止为 partial。

### 问题 7：为什么执行结果要作为 `role=tool` 的 message 回给模型？

标准答案：

- 因为工具调用是多轮推理的一部分。模型需要看到工具结果，才能决定下一步继续调用工具还是输出最终回答。`tool_call_id` 用来对应一轮里的多个工具调用。

## 小白完整类比：餐厅点菜系统

把 Hermes 工具系统想成餐厅：

| Hermes | 餐厅类比 |
|-|-|
| 模型 | 服务员 |
| 用户任务 | 客人需求 |
| 工具 | 厨房、收银、外卖、仓库等部门 |
| schema | 菜单和点单格式 |
| registry | 餐厅后台系统 |
| toolset | 今日开放的窗口 |
| check_fn | 这道菜今天有没有食材 |
| tool_call | 服务员下单 |
| handler | 厨房实际做菜 |
| tool result | 出餐结果 |
| tool_search | 问询台：先查有哪些隐藏服务 |

如果今天没有牛肉，菜单上就不该出现牛肉面。否则服务员点了，厨房才说没有，会浪费一轮沟通。

这就是 `check_fn` 的意义。

如果餐厅有 5000 道隐藏菜，不可能把整本菜单塞给服务员。服务员先问问询台“有没有低糖甜品”，再决定点哪道。

这就是 `tool_search` 的意义。

## 增强章节 A：工具注册表的工程细节

这一节把 `ToolRegistry.register()` 讲得更细。它不是简单地 `self._tools[name] = handler`，而是在处理四个工程问题：

1. 命名冲突
2. 工具集归属
3. 可用性检查
4. 缓存失效

### 代码片段 K：register 的冲突处理

位置：tools/registry.py

```Python
existing = self._tools.get(name)
if existing and existing.toolset != toolset:
    both_mcp = (
        existing.toolset.startswith("mcp-")
        and toolset.startswith("mcp-")
    )
    if both_mcp:
        logger.debug("MCP tool overwrite is allowed during refresh")
    elif override:
        logger.info("Explicit tool override")
    else:
        logger.error("Reject accidental cross-toolset shadowing")
        return

self._tools[name] = ToolEntry(
    name=name,
    toolset=toolset,
    schema=schema,
    handler=handler,
    check_fn=check_fn,
    requires_env=requires_env or [],
    is_async=is_async,
    description=description or schema.get("description", ""),
    emoji=emoji,
    max_result_size_chars=max_result_size_chars,
    dynamic_schema_overrides=dynamic_schema_overrides,
)
if check_fn and toolset not in self._toolset_checks:
    self._toolset_checks[toolset] = check_fn
self._generation += 1
```

中文注释版理解：

```Python
existing = self._tools.get(name)

# 如果已经有同名工具，并且来自不同 toolset，就进入冲突处理
if existing and existing.toolset != toolset:
    # MCP 工具之间允许覆盖，因为 MCP server refresh 时可能重新注册同名工具
    both_mcp = existing.toolset.startswith("mcp-") and toolset.startswith("mcp-")

    if both_mcp:
        pass
    elif override:
        # 插件明确说“我要覆盖”，才允许
        pass
    else:
        # 默认拒绝，防止插件或外部工具偷偷冒充内置工具
        return

# 真正写入注册表
self._tools[name] = ToolEntry(...)

# 记录 toolset 级别的可用性检查
if check_fn and toolset not in self._toolset_checks:
    self._toolset_checks[toolset] = check_fn

# 注册表版本号 +1，让工具 schema 缓存失效
self._generation += 1
```

为什么默认拒绝跨 toolset 同名覆盖？

因为工具名是模型调用时的唯一入口。如果一个插件也注册 `terminal`，但实现变成“把命令发到远端服务器”，而 Hermes 静默允许覆盖，那就会出现严重安全和调试问题。

所以 Hermes 的策略是：

- 内置工具和插件/MCP 重名：默认拒绝。
- 插件确实要替换：必须显式 `override=True`。
- MCP-to-MCP 重名：允许，因为动态刷新时可能自然发生。

### 三个真实注册例子

#### 例子 1：skill_manage

位置：tools/skill_manager_tool.py

```Python
registry.register(
    name="skill_manage",
    toolset="skills",
    schema=SKILL_MANAGE_SCHEMA,
    handler=lambda args, **kw: skill_manage(
        action=args.get("action", ""),
        name=args.get("name", ""),
        content=args.get("content"),
        old_string=args.get("old_string"),
        new_string=args.get("new_string"),
    ),
    emoji="📝",
)
```

特点：

- 用 lambda 做参数适配，把模型传来的 `args` 字典转成业务函数参数。
- 没有 `check_fn`，说明只要 skills toolset 启用，它通常可用。
- 属于 `skills` toolset，和 `skills_list`、`skill_view` 一起被管理。

#### 例子 2：file tools

位置：tools/file_tools.py

```Python
registry.register(name="read_file", toolset="file", schema=READ_FILE_SCHEMA, handler=_handle_read_file, check_fn=_check_file_reqs, emoji="📖", max_result_size_chars=100_000)
registry.register(name="write_file", toolset="file", schema=WRITE_FILE_SCHEMA, handler=_handle_write_file, check_fn=_check_file_reqs, emoji="✍️", max_result_size_chars=100_000)
registry.register(name="patch", toolset="file", schema=PATCH_SCHEMA, handler=_handle_patch, check_fn=_check_file_reqs, emoji="🔧", max_result_size_chars=100_000)
registry.register(name="search_files", toolset="file", schema=SEARCH_FILES_SCHEMA, handler=_handle_search_files, check_fn=_check_file_reqs, emoji="🔎", max_result_size_chars=100_000)
```

特点：

- 一个模块注册多个工具。
- 共享 `toolset="file"`。
- 共享 `_check_file_reqs()`。
- 每个工具设置 `max_result_size_chars=100_000`，避免大文件结果直接塞爆上下文。

#### 例子 3：terminal

位置：tools/terminal_tool.py

```Python
registry.register(
    name="terminal",
    toolset="terminal",
    schema=TERMINAL_SCHEMA,
    handler=_handle_terminal,
    check_fn=check_terminal_requirements,
    emoji="💻",
    max_result_size_chars=100_000,
)
```

位置：tools/terminal_tool.py

```Python
def check_terminal_requirements() -> bool:
    config = _get_env_config()
    env_type = config["env_type"]

    if env_type == "local":
        return True
    elif env_type == "docker":
        docker = find_docker()
        if not docker:
            return False
        result = subprocess.run([docker, "version"], capture_output=True, timeout=5)
        return result.returncode == 0
```

这说明 terminal 工具虽然是核心能力，但也不是永远暴露。配置成 Docker 后端时，如果 Docker 不可用，模型根本看不到 terminal 工具。

## 增强章节 B：toolset 过滤和 schema 缓存

### enabled_toolsets 与 disabled_toolsets

`enabled_toolsets` 和 `disabled_toolsets` 的关系不是平级“二选一”，而是：

```Plaintext
先用 enabled_toolsets 做白名单
再用 disabled_toolsets 做最终减法
```

位置：model_tools.py

```Python
if enabled_toolsets is not None:
    for toolset_name in effective_enabled_toolsets:
        if validate_toolset(toolset_name):
            resolved = resolve_toolset(toolset_name)
            tools_to_include.update(resolved)
else:
    from toolsets import get_all_toolsets
    for ts_name in get_all_toolsets():
        tools_to_include.update(resolve_toolset(ts_name))
```

位置：model_tools.py

```Python
if disabled_toolsets:
    for toolset_name in disabled_toolsets:
        if validate_toolset(toolset_name):
            resolved = resolve_toolset(toolset_name)
            tools_to_include.difference_update(resolved)
```

为什么 disabled 要最后减？

因为 toolset 可以组合。例如某个“大礼包 toolset”包含 `terminal`，用户又显式禁用 `terminal`。如果禁用不在最后执行，大礼包可能把 terminal 又加回来。

理解成数学集合：

```Plaintext
最终工具 = resolve(enabled_toolsets 或 all_toolsets) - resolve(disabled_toolsets)
```

### 特殊场景：Kanban worker 强制工具

位置：model_tools.py

```Python
if os.environ.get("HERMES_KANBAN_TASK") and "kanban" not in effective_enabled_toolsets:
    effective_enabled_toolsets.append("kanban")
```

意思是：如果当前进程是 kanban worker，就算用户配置里没显式启用 kanban，也必须给它完成任务、阻塞任务、心跳等生命周期工具。否则 worker 无法向调度器报告状态。

### schema 缓存 key

位置：model_tools.py

```Python
cache_key = (
    frozenset(enabled_toolsets) if enabled_toolsets is not None else None,
    frozenset(disabled_toolsets) if disabled_toolsets else None,
    registry._generation,
    cfg_fp,
    bool(os.environ.get("HERMES_KANBAN_TASK")),
    bool(skip_tool_search_assembly),
)
```

每个字段为什么在 key 里：

| 字段 | 为什么影响工具 schema |
|-|-|
| `enabled_toolsets` | 白名单不同，工具不同 |
| `disabled_toolsets` | 黑名单不同，工具不同 |
| `registry._generation` | 动态注册/注销后工具不同 |
| `cfg_fp` | 配置文件变化会影响动态 schema |
| `HERMES_KANBAN_TASK` | kanban worker 会强制加工具 |
| `skip_tool_search_assembly` | 返回折叠前或折叠后的工具清单不同 |

`cfg_fp` 是配置文件的 `(mtime_ns, size)`。它不是直接解析配置内容做 hash，成本更低，足够用于缓存失效。

## 增强章节 C：模型 tool_calls 的执行细节

### 代码片段 L：工具名修复

位置：agent/agent_runtime_helpers.py

```Python
lowered = tool_name.lower()
if lowered in agent.valid_tool_names:
    return lowered
normalized = _norm(tool_name)
if normalized in agent.valid_tool_names:
    return normalized
...
matches = get_close_matches(lowered, agent.valid_tool_names, n=1, cutoff=0.7)
if matches:
    return matches[0]
```

模型可能把工具名写成：

```Plaintext
BrowserClick_tool
browser-click
PatchTool
skillManage
```

Hermes 会尝试：

1. 小写化
2. `-` / 空格转 `_`
3. CamelCase 转 snake_case
4. 去掉 `_tool` / `-tool` / `tool` 后缀
5. fuzzy match

为什么要修复而不是直接失败？

因为工具名错误通常是模型格式小失误，不是用户任务失败。能自动修复，就能减少无意义重试。

### JSON 参数错误处理

位置：agent/conversation_loop.py

```Python
if isinstance(args, (dict, list)):
    tc.function.arguments = json.dumps(args)
elif not args or not args.strip():
    tc.function.arguments = "{}"
else:
    json.loads(args)
```

处理策略：

| 情况 | 处理 |
|-|-|
| arguments 已经是 dict/list | 转成 JSON 字符串 |
| arguments 是空字符串 | 当作 `{}` |
| arguments 是合法 JSON 字符串 | 正常进入执行 |
| arguments 非法且像截断 | 停止为 partial，避免执行半截危险参数 |
| arguments 非法但非截断 | 让模型重试，最多 3 次 |

### 顺序执行 vs 并发执行

位置：agent/tool_dispatch_helpers.py

```Python
if len(tool_calls) <= 1:
    return False
if any(name in _NEVER_PARALLEL_TOOLS for name in tool_names):
    return False
...
if tool_name in _PATH_SCOPED_TOOLS:
    if any(_paths_overlap(scoped_path, existing) for existing in reserved_paths):
        return False
...
if tool_name not in _PARALLEL_SAFE_TOOLS:
    if not _is_mcp_tool_parallel_safe(tool_name):
        return False
```

并发不是“只要多个工具就一起跑”，而是先问：

1. 是否只有一个工具？一个就没必要并发。
2. 是否包含永远不能并发的工具？
3. 文件路径是否重叠？
4. 工具是否被标记为 parallel safe？
5. MCP 工具是否允许并发？

为什么谨慎？

因为并发读多个文件很安全，但并发写同一个文件、并发执行交互工具、并发操作同一个浏览器页面，都可能产生竞态。

### 代码片段 M：并发执行但按原顺序回填

位置：agent/tool_executor.py

```Python
with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
    f = executor.submit(propagate_context_to_thread(_run_tool), i, tc, name, args)
...
results[index] = (function_name, function_args, result, duration, is_error, False)
...
messages.append(make_tool_result_message(name, _tool_content, tc.id))
```

关键点：

- 工具可以并发完成。
- 结果存进 `results[index]`。
- 最后按原始 tool_call index 回填。

为什么要按原顺序？

因为模型上一条 assistant message 里的 tool_calls 是有顺序的。按原顺序回填可以减少 provider 适配差异，也让模型更容易对齐“哪个结果对应哪个调用”。

### 代码片段 N：tool result message

位置：agent/tool_dispatch_helpers.py

```Python
return {
    "role": "tool",
    "name": name,
    "tool_name": name,
    "content": wrapped,
    "tool_call_id": tool_call_id,
}
```

字段解释：

| 字段 | 作用 |
|-|-|
| `role` | 告诉模型这是一条工具结果消息 |
| `name` | OpenAI/tool adapter 使用的工具名 |
| `tool_name` | Hermes 内部 session DB 使用 |
| `content` | 工具结果文本，可能被包装/裁剪/落盘 |
| `tool_call_id` | 对应 assistant tool_calls 里的 id |

### 大结果为什么要落盘

位置：tools/tool_result_storage.py

大结果不应该直接完整塞回模型上下文，否则一次 `search_files` 或 `read_file` 就可能把上下文撑爆。

Hermes 的策略是：

1. 单个大结果：`maybe_persist_tool_result()` 写到临时文件，只回填摘要和路径。
2. 一轮多个中等结果总量太大：`enforce_turn_budget()` 再把最大结果落盘。
3. 模型后续可用 `read_file` 分页读取需要部分。

这比“直接截断”更好，因为信息没有丢，只是延迟读取。

## 增强章节 D：插件和 MCP 动态工具

### 插件如何注册工具

位置：hermes_cli/plugins.py

```Python
def register_tool(
    self,
    name,
    toolset,
    schema,
    handler,
    check_fn=None,
    override=False,
):
    from tools.registry import registry
    registry.register(
        name=name,
        toolset=toolset,
        schema=schema,
        handler=handler,
        check_fn=check_fn,
        # 还会传入 requires_env、is_async、description、emoji 等元数据
        override=override,
    )
    self._manager._plugin_tool_names.add(name)
```

插件不是直接改全局变量，而是在自己的 `register(ctx)` 中通过 `PluginContext.register_tool()` 接入统一 registry。

为什么 manifest 里的 `provides_tools` 不够？

- `plugin.yaml` 里的 `provides_tools` 只是声明“我可能提供这些工具”。
- 真正可调用需要运行时提供 `schema + handler + check_fn`。
- 插件可以根据环境、配置、依赖决定是否实际注册。

### MCP 如何动态注册工具

位置：tools/mcp_tool.py

```Python
def _register_server_tools(name, server, config):
    toolset_name = f"mcp-{name}"
    ...
    schema = _convert_mcp_schema(name, mcp_tool)
    registry.register(
        name=schema["name"],
        toolset=toolset_name,
        schema=schema,
        handler=_make_tool_handler(name, mcp_tool.name, server.tool_timeout),
        check_fn=_make_check_fn(name),
    )
    registry.register_toolset_alias(name, toolset_name)
```

MCP 工具来自远端 server 的 `list_tools()`。Hermes 做了几步转换：

1. 连接 MCP server。
2. 调用 `list_tools()` 拿远端工具列表。
3. 把 MCP tool schema 转成 Hermes/OpenAI-style schema。
4. 为每个远端工具生成 handler。
5. 注册到 `registry`。
6. 注册 toolset alias，例如真实 toolset 是 `mcp-github`，用户可以用 `github`。

### MCP 工具列表变化

位置：tools/mcp_tool.py

```Python
case ToolListChangedNotification():
    self._schedule_tools_refresh()
```

Hermes 收到 `notifications/tools/list_changed` 后，不在通知 handler 里同步刷新，而是调度后台刷新。

为什么？

MCP 通常走 JSON-RPC/stdin/stdout。如果在通知处理里同步 `list_tools()`，可能和当前流上的请求互相等待，导致卡死。后台刷新更安全。

刷新时：

位置：tools/mcp_tool.py

```Python
stale_tool_names = old_tool_names - {...fresh names...}
for tool_name in stale_tool_names:
    registry.deregister(tool_name)
    _forget_mcp_tool_server(tool_name)

self._registered_tool_names = _register_server_tools(self.name, self, self._config)
```

也就是：

1. 新列表里没有的旧工具：注销。
2. 新列表里还存在或新增的工具：重新注册。
3. 注册表 generation 改变，schema 缓存失效。

### deregister 不只是删除工具

位置：tools/registry.py

```Python
entry = self._tools.pop(name, None)
...
if not toolset_still_exists:
    self._toolset_checks.pop(entry.toolset, None)
    self._toolset_aliases = {
        alias: target
        for alias, target in self._toolset_aliases.items()
        if target != entry.toolset
    }
self._generation += 1
```

如果一个 MCP server 的最后一个工具没了，只删工具名还不够，还要：

- 清理 toolset check。
- 清理 alias。
- generation +1。

否则 UI 或模型可能看到“幽灵工具集”。

## 增强章节 E：风险与防护

| 风险 | 发生场景 | 防护 |
|-|-|-|
| 插件覆盖内置工具 | 插件注册同名工具 | 默认拒绝跨 toolset 覆盖，除非 `override=True` |
| 工具不可用仍暴露 | 缺 Docker/API key/依赖 | `check_fn` 过滤 |
| 工具列表变化但 schema 缓存没变 | MCP refresh/plugin load | `registry._generation` |
| 受限 session 通过 tool_call 越权 | tool_search bridge 调隐藏工具 | scoped deferrable names 检查 |
| 工具错误文本污染模型 | 异常里带结构标签/提示注入 | `_sanitize_tool_error()` 清洗 |
| 大结果撑爆上下文 | read/search 返回超大内容 | tool result storage 落盘 |
| 并发写冲突 | 多工具同时写同一路径 | parallel safety/path overlap 检查 |
| MCP prompt injection | 远端工具描述恶意 | MCP 描述扫描和工具结果边界包装 |

### 代码片段 O：错误清洗

位置：model_tools.py

```Python
_TOOL_ERROR_ROLE_TAG_RE = re.compile(
    r'</?(?:tool_call|function_call|result|response|output|input|system|assistant|user)>',
    re.IGNORECASE,
)
...
def _sanitize_tool_error(error_msg: str) -> str:
    sanitized = _TOOL_ERROR_ROLE_TAG_RE.sub("", error_msg)
    ...
    return f"[TOOL_ERROR] {sanitized}"
```

为什么工具错误也要清洗？

工具错误会进入模型上下文。如果异常字符串里包含类似 `</tool_call><system>...` 的结构化片段，虽然 JSON 层不会让它真的逃逸协议，但模型会“读到”这些文本，可能被干扰。清洗是防御纵深。

## 增强章节 F：更完整的面试题

### 问题 8：`enabled_toolsets=None` 和 `enabled_toolsets=[]` 有什么区别？

标准答案：

- `None` 表示不限制，默认从所有 toolset 取工具。`[]` 表示显式空白名单，普通 toolset 一个都不给。这个区别对平台配置很重要。

小白解释：

- `None` 是“没说限制，默认全开放”；`[]` 是“明确说今天不开任何窗口”。

### 问题 9：为什么 disabled_toolsets 要在最后做集合减法？

标准答案：

- 因为 enabled toolset 可能是组合工具集。最后减法才能保证用户显式禁用的工具不会被组合工具集重新带回来。

### 问题 10：为什么 tool_search 的 `tool_call` unwrap 后还要做 scope gate？

标准答案：

- 因为 unwrap 会把桥接工具变成底层真实工具调用。如果不检查底层工具是否属于当前 session 授权范围，受限子 Agent 或 webhook session 可能通过桥接工具调用未授权工具。

### 问题 11：为什么非法 JSON 参数不一定马上失败？

标准答案：

- 因为模型输出工具参数时常有格式小错误。Hermes 会区分空字符串、可修复格式、疑似截断和真正非法。可修复就修，非危险错误让模型重试，疑似截断才停止，避免执行半截参数。

### 问题 12：并发执行工具为什么仍要按原始顺序回填？

标准答案：

- tool result 需要对应 assistant message 里的 tool_calls。并发完成顺序不可控，但原始调用顺序是协议上下文的一部分。按原顺序回填减少模型和 provider adapter 混淆。

### 问题 13：为什么大工具结果不直接截断？

标准答案：

- 直接截断会丢信息。Hermes 把大结果落盘，只回填摘要和文件路径，让模型后续按需分页读取，既保留信息又控制上下文。

### 问题 14：MCP 工具列表变化为什么要后台刷新？

标准答案：

- MCP 通常是 JSON-RPC 流。如果在 `tools/list_changed` 通知处理里同步刷新，可能阻塞同一条通信流。后台刷新可以避免卡住 MCP server。

### 问题 15：`plugin.yaml.provides_tools` 为什么不能直接让工具可调用？

标准答案：

- manifest 只是声明插件可能提供哪些工具。真正调用需要运行时注册 `schema`、`handler`、`check_fn` 到 registry。这样插件才能根据环境和配置动态决定实际提供什么。

## 后续深挖建议

1. 单独拆 `terminal` 工具：权限、审批、后台进程、task_id 隔离。
2. 单独拆 `skill_manage` 工具：技能创建、patch、frontmatter、curator 统计。
3. 单独拆 MCP 动态注册：外部 server schema 如何变成 Hermes tool。
4. 单独拆 tool_executor：并发执行、guardrail、中断、结果预算。
