# 主题 3：Tool —— agent 的"手"

> 学习顺序：**Agent → Context → Tool → Memory → LLM Provider → Session → Skill → SubAgent**
> 本章目标：看懂工具系统——`Tool` 数据类、`build_tool` 工厂、`ToolRegistry` 注册与分发、47 个内置工具；理解"模型只会动嘴，工具才是手"。
> 前置：主题 2 学过 `_build_tool_docs_section` 把工具文档拼进提示词——本章回答"工具的 schema 从哪来、怎么被调用"。
> 阅读代码前先跑一遍：`cd clawcodex-ascend && uv run clawcodex-dev -p "帮我读一下当前目录的 README.md"`，观察它怎么用 Read 工具。

---

## 0. 承上启下：从前一章到本章

### 0.1 前一章回顾：主题 2（Context）学了什么

**主题 2 的核心问题**：模型看到的系统提示词是谁拼的？——答案是 `context_system/` 的**提示词组装流水线**。

| 维度 | 主题 2 学到的 | 一句话 |
|------|--------------|--------|
| **拼装入口** | `build_effective_system_prompt()` | headless/TUI 路径的总封装 |
| **拼块核心** | `build_full_system_prompt_blocks()` | 收集→排序→分组→打缓存标记 |
| **section 机制** | `register_section(id, builder, ...)` | 插件式加内容，不改拼装主代码 |
| **缓存分层** | CacheScope（GLOBAL/SESSION/REQUEST） | 按变化频率分组，组尾打 cache_control |
| **19 个 builder** | intro/system/doing_tasks/.../skills | 每个产出一段 section |

**主题 2 的设计理念**：**拼装不写死，section 插件化**；**缓存按变化频率分层**；**关注点分离，各模块只注册不侵入**。

> **一句话记住主题 2**：Context = 提示词组装流水线——section 插件化拼装 + cache boundary 分层。

### 0.2 本章设计架构：Tool（工具）

**本章的核心问题**：主题 2 里 `_build_tool_docs_section` 把"工具文档"拼进提示词——但**工具的 schema 从哪来？模型说"调工具"后工具怎么被执行？**

**设计架构（声明 → 注册 → 分发）**：

```
① 声明:  build_tool(name, input_schema, call, ...)   build_tool.py:144
            └─ 造一个 Tool 对象（名字/schema/实现函数/权限...）
                  │
② 注册:  ToolRegistry.register(tool)                registry.py:241
            └─ 存进 _tools 列表 + _by_name 字典（+别名）
                  │
③ 展示:  ToolRegistry.get_tools() → _build_tool_docs_section
            └─ 序列化成模型能看的 schema → 拼进提示词（主题2）
                  │
④ 分发:  ToolRegistry.dispatch(call, context)        registry.py:341
            └─ 模型返回 tool_use → 校验/权限 → 调 call() → 返回结果
```

**本章的设计理念（3 条核心）**：

| 理念 | 含义 | 代码体现 |
|------|------|---------|
| **1. 工具 = 数据 + 函数** | Tool 是 dataclass：声明时给名字/schema/实现，运行时才调 call | `build_tool` 工厂 + `Tool` dataclass |
| **2. 注册即插拔** | 注册进 ToolRegistry 就能被模型看到、被分发；不注册就不可见 | `register()` + `_by_name` |
| **3. 分发带安全门** | dispatch 前有 5 道检查（schema 校验/权限/plan-mode/validate） | `dispatch()` 的完整流程 |

**与主题 2 的架构呼应**：

| 主题 2（Context） | 主题 3（Tool） |
|------------------|----------------|
| section 注册表（`_registry`） | 工具注册表（`ToolRegistry`） |
| `register_section(id, builder)` | `register(tool)` |
| 拼装时收集 section | 分发时按名字找工具 |
| section 插件机制 | 工具即插即拔 |

> **一句话记住本章**：Tool = agent 的"手"——**声明（build_tool）→ 注册（register）→ 展示（get_tools→提示词）→ 分发（dispatch）**。模型只负责"点单"（tool_use），工具系统负责"出餐"（找工具→校验→执行）。

---

## 1. 它是什么

**Tool（工具）= agent 用来"影响世界"的接口**。模型本身不会读文件、不会执行命令——它只能输出文本。工具让模型能"动手"：模型返回一个 `tool_use`（说"我要用 Read 读文件"），宿主（工具系统）执行真实的文件读取，把结果喂回给模型。

**核心心法一句话：**

> **模型只会"动嘴"（输出文本），工具才是"手"（真实执行）。工具 = 名字 + 输入 schema + 执行函数（call）+ 权限规则。**

### 和主题 2 的关系

主题 2 学过 `_build_tool_docs_section`（prompt_assembly.py）把工具序列化进提示词。但那时只看到"展示"环节。本章补全全链路：**工具的 schema 从哪来（build_tool）、怎么被组织（register）、模型点单后怎么执行（dispatch）**。

### 为什么先学它

主题 4-8（Memory/Provider/Session/Skill/SubAgent）**都要通过工具落地**——记忆工具、模型调用工具、Skill 工具、子 agent 工具。理解工具系统的"声明-注册-分发"，后面每个模块学完都能对号入座："这个能力是注册了哪个工具进去的"。

---

## 2. 代码在哪

主题 3 只需要看 4 个文件（按阅读顺序）：

| 文件 | 看什么 | 重点标注 |
|------|--------|----------|
| `clawcodex_ext/tool_system/build_tool.py` | `Tool` 数据类 + `build_tool` 工厂函数 | **必读**，先读这个 |
| `clawcodex_ext/tool_system/registry.py` | `ToolRegistry` 注册表 + `register()` + `dispatch()` | 必读 |
| `clawcodex_ext/tool_system/tools/` | **47 个内置工具**（read.py/write.py/bash/...） | 挑 2-3 个看 |
| `clawcodex_ext/tool_system/tools/__init__.py` | 内置工具怎么被装配 | 扫读 |

> 为什么先读 `build_tool.py`？因为它定义了"一个工具长什么样"——这是整个工具体系的**数据契约**。

### 顺带认识：tool_system 全家桶

| 文件/目录 | 职责 |
|-----------|------|
| `build_tool.py` | **Tool 数据类 + build_tool 工厂**（本章重点） |
| `registry.py` | **ToolRegistry**：注册/查找/分发（本章重点） |
| `context.py` | `ToolContext`（环境背包，主题 1 学过） |
| `tools/` | **47 个内置工具**（read/write/glob/grep/bash/agent/skill...） |
| `loader.py` / `defaults.py` | 工具加载 / 默认工具集 |
| `renderers.py` | 工具结果渲染 |
| `schema_validation.py` | 输入 schema 校验 |
| `tool_search.py` | 工具检索（按意图找工具） |
| `errors.py` / `tool_timeout.py` | 工具错误 / 超时控制 |
| `protocol.py` | 工具系统协议（抽象接口） |
| `tools/mcp.py` | **通用 MCP 工具**（`MCPTool`：server+tool 参数调用） |
| `services/mcp/` | **MCP 服务端**：client 连接、tool_wrapper 包装、connection_manager、oauth 等 31 文件 |

---

## 3. 核心概念与代码语法

### 3.1 `Tool` 数据类：一个工具长什么样

**伪代码先行**——工具的核心字段：

```python
# 伪代码：一个工具 = 名字 + schema + 实现 + 元信息
tool = Tool(
    名字="read_file",                    # name：模型点单用的名字
    输入格式={"path": "string"},          # input_schema：模型怎么填参数
    执行函数=读取文件内容,                 # call：真正干活的函数
    描述="读取指定文件",                  # description：模型看到的介绍
    权限检查=检查是否允许读,               # check_permissions
    只读=True,                           # is_read_only：只读工具
)
```

**再看真实代码**（`build_tool.py:65-114`）：

```python
@dataclass
class Tool:
    name: str                            # 名字（模型点单用）
    input_schema: Mapping[str, Any]      # 输入 schema（模型填参数的格式）
    call: Callable[[dict[str, Any], ToolContext], ToolResult]   # 执行函数（核心！）
    prompt: Callable[..., str]           # 提示词（工具使用说明）
    description: Callable[[dict[str, Any]], str]   # 描述（模型看到的）
    map_result_to_api: Callable[[Any, str], dict[str, Any]]    # 结果转 API 格式
    check_permissions: Callable[[dict[str, Any], ToolContext], PermissionResult]  # 权限检查
    is_enabled: Callable[[], bool]       # 是否启用
    is_concurrency_safe: Callable[[dict[str, Any]], bool]  # 能否并行（分区用）
    is_read_only: Callable[[dict[str, Any]], bool]        # 是否只读
    is_destructive: Callable[[dict[str, Any]], bool]      # 是否破坏性
    user_facing_name: Callable[[dict[str, Any] | None], str]  # 用户显示名
    to_auto_classifier_input: Callable[[dict[str, Any]], Any]  # 自动分类输入

    aliases: tuple[str, ...] = ()        # 别名（模型可用别名点单）
    search_hint: str | None = None       # 搜索提示
    max_result_size_chars: int | float = 20_000  # 结果大小上限
    strict: bool = False                 # 严格模式
    should_defer: bool = False           # 是否延迟加载
    always_load: bool = False            # 是否总加载
    is_mcp: bool = False                 # 是否 MCP 工具
    is_lsp: bool = False                 # 是否 LSP 工具

    validate_input: ... = None           # 输入校验（可选）
    get_path: ... = None                 # 路径提取（可选）
    ...
```

#### 语法拆解

| 语法 | 说明 |
|------|------|
| `@dataclass` | 自动生成 `__init__`/`__repr__` 等——字段声明即构造参数 |
| `Callable[[dict, ToolContext], ToolResult]` | call 是函数类型：接收 (输入dict, 工具上下文) → 返回 ToolResult |
| `Callable[[dict[str, Any]], bool]` | 元信息都是函数（延迟求值）——**不是 bool 值，是返回 bool 的函数** |
| `field(...)` 没有，默认值直接写 | `aliases: tuple = ()` 不可变默认值可以直接写 |

> **关键**：`is_read_only`/`is_enabled` 等是**函数**不是 bool——运行时才求值（可以依赖输入动态判断）。

### 3.2 `build_tool` 工厂：怎么造一个工具

**伪代码先行**：

```python
# 伪代码：build_tool 是"工具工厂"——填必填，其余有默认
tool = build_tool(
    name="read_file",
    input_schema={"path": {"type": "string"}},
    call=read_file_impl,          # 必填：实现
    description="读取文件",        # 可选：不给就用名字
    is_read_only=lambda _: True,  # 可选：不给有默认
)
```

**再看真实代码**（`build_tool.py:144-240`）——核心段：

```python
def build_tool(
    *, name: str,                        # 名字（必填）
    input_schema: Mapping[str, Any],     # 输入格式（必填）
    call: Callable[[dict[str, Any], ToolContext], ToolResult],  # 实现（必填）
    prompt: Callable[..., str] | str | None = None,   # 提示词（可选）
    description: Callable[[dict[str, Any]], str] | str | None = None,  # 描述（可选）
    map_result_to_api: ... | None = None,
    ...
    is_enabled: Callable[[], bool] | None = None,
    is_read_only: Callable[[dict[str, Any]], bool] | None = None,
    ...
) -> Tool:
    # ① 处理 prompt：字符串/None/函数 → 统一成函数
    if isinstance(prompt, str):
        _p = prompt
        def prompt_fn() -> str:
            return _p                   # 字符串 → 包成函数
    elif prompt is None:
        def prompt_fn():
            return ""                   # None → 空提示词
    else:
        prompt_fn = prompt              # 函数 → 直接用

    # ② 处理 description：同理统一成函数（None → 用名字）
    if isinstance(description, str):
        _d = description
        def desc_fn(_input):
            return _d
    elif description is None:
        def desc_fn(_input):
            return name                 # None → 用名字当描述
    else:
        desc_fn = description

    # ③ 组装 Tool 对象（缺省字段用 TOOL_DEFAULTS 兜底）
    return Tool(
        name=name,
        input_schema=input_schema,
        call=call,
        prompt=prompt_fn,
        description=desc_fn,
        map_result_to_api=map_result_to_api or _default_map_result_to_api(name),  # 兜底
        ...
        is_enabled=is_enabled or TOOL_DEFAULTS["is_enabled"],       # 兜底: lambda: True
        is_concurrency_safe=is_concurrency_safe or TOOL_DEFAULTS["is_concurrency_safe"],  # 兜底: False
        is_read_only=is_read_only or TOOL_DEFAULTS["is_read_only"], # 兜底: False
        is_destructive=is_destructive or TOOL_DEFAULTS["is_destructive"],  # 兜底: False
        ...
    )
```

#### 语法拆解

| 语法 | 说明 |
|------|------|
| `*, name, ...` | 所有参数只能按关键字传 |
| `prompt: ... \| str \| None` | 三种形态都接受——**统一成函数**（多态适配） |
| `is_enabled or TOOL_DEFAULTS["is_enabled"]` | 缺省兜底：给了用给的，没给用默认（`lambda: True`） |
| `TOOL_DEFAULTS`（117-124 行） | 默认行为表：is_enabled=True、is_read_only=False 等 |

#### 用法小示例

```python
>>> from clawcodex_ext.tool_system.build_tool import build_tool
>>> from clawcodex_ext.tool_system.protocol import ToolResult   # 结果类型（可选 import）

>>> # 造一个最简单的工具（call 接收 (输入dict, 工具上下文) → 返回 ToolResult）
>>> my_tool = build_tool(
...     name="greet",
...     input_schema={"name": {"type": "string"}},
...     call=lambda tool_input, ctx: ToolResult(name="greet", output=f"Hello {tool_input['name']}!"),
...     description="Say hello to someone",
... )
>>> my_tool.name
'greet'
>>> my_tool.is_read_only({})      # 没给 → 默认 False
False
>>> my_tool.description({})       # 给了 → 用给的
'Say hello to someone'
```

### 3.3 `ToolRegistry`：注册与查找

**伪代码先行**：

```python
# 伪代码：注册表 = 列表 + 字典（名字→工具）
注册表:
    _tools:  list[Tool]      # 按注册顺序
    _by_name: dict[str, Tool]  # 名字（小写）→ 工具，含别名

注册(tool):  _tools.append(tool); _by_name[name.lower()] = tool; 别名也登记
查找(name):  return _by_name.get(name.lower())
```

**再看真实代码**（`registry.py:229-260`）：

```python
class ToolRegistry:
    def __init__(self, tools=None):
        self._tools: Tools = []                  # 列表（按注册顺序）
        self._by_name: dict[str, Tool] = {}      # 字典（名字小写 → 工具）
        self.disabled_servers: set[str] = set()  # 禁用的 MCP 服务器
        if tools:
            for tool in tools:
                self.register(tool)              # 构造时批量注册

    def register(self, tool: Tool) -> None:
        key = tool.name.lower()                  # 名字转小写（大小写不敏感）
        if key in self._by_name:
            raise ValueError(f"duplicate tool name: {tool.name}")  # 重复名 → 报错
        self._tools.append(tool)                 # 加入列表
        self._by_name[key] = tool                # 加入字典
        for alias in tool.aliases:               # 别名也登记
            alias_key = alias.lower()
            if alias_key in self._by_name:
                raise ValueError(...)            # 别名冲突 → 报错
            self._by_name[alias_key] = tool
```

> **注意**：`register` 遇到重复名字**抛 ValueError**（不像 agent 注册表 last-wins）——工具名冲突是硬错误，因为分发时歧义会致命。

#### 用法小示例

```python
>>> from clawcodex_ext.tool_system.registry import ToolRegistry
>>> from clawcodex_ext.tool_system.build_tool import build_tool

>>> reg = ToolRegistry()
>>> reg.register(build_tool(name="read_file", input_schema={}, call=lambda i, c: ToolResult(...)))
>>> reg.register(build_tool(name="write_file", input_schema={}, call=lambda i, c: ToolResult(...)))

>>> reg.find_tool_by_name("read_file").name   # 查找（模块级函数，需传列表）
>>> reg.get("read_file").name                 # 或 ToolRegistry.get() 方法（registry.py:304）
'read_file'
>>> [t.name for t in reg.get_tools()]         # 列出全部（注意：get_tools 是模块级函数）
['read_file', 'write_file']
```

### 3.4 `dispatch`：模型点单后怎么执行

**伪代码先行**——分发是"安全门 + 执行器"：

```python
# 伪代码：分发
def dispatch(call, context):
    1. 按名字找工具（找不到 → 返回 error）
    2. schema 校验 + 语义强转（"true" → True）
    3. plan-mode 检查（破坏性工具在 plan 模式禁止）
    4. validate_input 校验（可选）
    5. 权限检查（deny → 拒绝；ask → 问用户；allow → 放行）
    6. 调 tool.call(input, context) → 结果
    7. 包装成 ToolResult 返回
```

**再看真实代码**（`registry.py:341-512`）——核心段：

```python
def dispatch(self, call: ToolCall, context: ToolContext) -> ToolResult:
    # ① 找工具（支持宏解析）
    tool = resolve_tool_for_context(context, call.name, base_registry=self)
    if tool is None:
        return ToolResult(name=call.name, output={"error": f"unknown tool: {call.name}"},
                          is_error=True, tool_use_id=call.tool_use_id)   # 找不到 → error

    # ② schema 校验 + 语义强转（"true"→True, "30"→30）
    coerced_input = validate_tool_input(tool.name, call.input, tool.input_schema)
    if coerced_input is not call.input:
        call = ToolCall(name=call.name, input=coerced_input, ...)   # 用强转后的输入

    # ③ plan-mode 防御（破坏性工具在 plan 模式禁止动项目文件）
    if context.plan_mode and tool.name in _DESTRUCTIVE_TOOLS:
        if not _is_temp_path(tool.name, call.input, context):
            return ToolResult(..., is_error=True)   # 拒绝

    # ④ validate_input 自定义校验（可选）
    if tool.validate_input is not None:
        validation = tool.validate_input(call.input, context)
        if not validation.result:
            return ToolResult(..., is_error=True)   # 校验失败

    # ⑤ 权限检查
    decision = has_permissions_to_use_tool(tool, call.input, context.permission_context, ...)
    if decision.behavior == "deny":
        return ToolResult(..., is_error=True)       # 拒绝
    if decision.behavior == "ask":
        final, ... = handle_permission_ask(...)     # 问用户
        if final.behavior == "deny":
            return ToolResult(..., is_error=True)   # 用户拒绝

    # ⑥ 执行！
    try:
        result = _invoke_tool_call(tool, call.input, context)   # 调 tool.call()
    finally:
        ...
    return result                                   # 返回 ToolResult
```

**5 道安全门**（分发 = 安全门 + 执行）：

| 关卡 | 检查 | 不通过返回 |
|------|------|-----------|
| ① 找工具 | 名字在注册表？ | `unknown tool` error |
| ② schema 强转 | 输入格式合法？ | 校验失败 |
| ③ plan-mode | 破坏性工具在 plan 模式？ | 拒绝 |
| ④ validate_input | 自定义校验通过？ | 校验失败 |
| ⑤ 权限 | 允许/拒绝/询问？ | deny → error |

> **安全门顺序有讲究**：先找工具 → 再校验输入 → 再权限——**权限是最后一道门**，因为它最贵（可能弹窗问用户）。

### 3.5 MCP：扩展工具的"外挂源"

**MCP（Model Context Protocol）= 让 agent 连上外部能力的标准协议**。前面学的 47 个内置工具（read/write/bash...）是"自带的手"，而 MCP 是"外接的手"——数据库、GitHub API、浏览器、企业系统，只要实现 MCP 协议就能被 agent 调用。

#### 3.5.1 两种 MCP 工具形态

项目里有**两种** MCP 使用方式（`tool_system/tools/mcp.py` + `services/mcp/tool_wrapper.py`）：

| 形态 | 名字 | 作用 | 代码 |
|------|------|------|------|
| **通用 MCP 工具** | `MCP`（单个） | 手动指定 server + tool 调用 | `tools/mcp.py:312` 的 `MCPTool` |
| **per-server 包装工具** | `mcp__server__tool`（每工具一个） | 服务器暴露的每个工具包装成独立 Tool | `services/mcp/tool_wrapper.py` |

**命名格式**（`mcp_string_utils.py:55`）：

```python
def build_mcp_tool_name(server_name: str, tool_name: str) -> str:
    return f"mcp__{server_name}__{tool_name}"     # 如 mcp__github__search_code
```

> `mcp__<服务器>__<工具>` 三段式——模型看到这个名字就知道"这是 github 服务器上的 search_code 工具"。`registry.disabled_servers` 可以用前缀隐藏整台服务器的工具（registry.py:339）。

#### 3.5.2 通用 MCP 工具（MCPTool）

**它是个特殊的 Tool**——`server`/`tool` 是**输入参数**，不是名字的一部分（`tools/mcp.py:312-348`）：

```python
MCPTool: Tool = build_tool(
    name="MCP",
    input_schema={
        "type": "object",
        "properties": {
            "server": {"type": "string", "description": "要调用的 MCP 服务器名"},
            "tool":   {"type": "string", "description": "该服务器上的工具名"},
            "input":  {"type": "object", "description": "传给工具的参数（JSON）"},
        },
        "required": ["server", "tool"],          # server + tool 必填
    },
    call=_mcp_call,                               # 真正执行（见下）
    prompt=MCP_TOOL_PROMPT,                       # 使用说明（含示例）
    description="Call a tool exposed by a connected MCP server.",
    map_result_to_api=_mcp_map_result_to_api,     # 结果格式（字符串/block 直传）
    validate_input=_validate_mcp_input,           # 输入校验
    max_result_size_chars=100_000,                # 结果上限（防撑爆上下文）
    is_mcp=True,
)
```

**模型怎么用**（`MCP_TOOL_PROMPT` 里的示例，mcp.py:76-78）：

```json
// 模型返回的 tool_use:
{"name": "MCP", "input": {"server": "github", "tool": "search_code", "input": {"query": "bug fix"}}}
// 或无参数:
{"name": "MCP", "input": {"server": "myserver", "tool": "list_items"}}
```

#### 3.5.3 `_mcp_call`：真正执行（mcp.py:197-237）

```python
def _mcp_call(tool_input: dict[str, Any], context: ToolContext) -> ToolResult:
    server = tool_input["server"]              # ① 取服务器名
    tool_name = tool_input["tool"]             # ② 取工具名
    args = tool_input.get("input") or {}       # ③ 取参数（没有给空字典）
    # ④ 基本校验（server/tool 非空、args 是字典）
    if not isinstance(server, str) or not server:
        raise ToolInputError("server must be a non-empty string")
    ...
    # ⑤ 从上下文拿 MCP 客户端（连接管理在 context.mcp_clients）
    client = context.mcp_clients.get(server)
    if client is None:
        return ToolResult(name="MCP", output={"error": f"mcp server not connected: {server}"},
                          is_error=True)       # 服务器没连上 → error

    # ⑥ 调 MCP 客户端（异步，可能在专用事件循环上跑）
    async def _async_call():
        raw = client.call_tool(tool_name, args)   # 返回值: 任意（MCP 协议响应）
        if inspect.isawaitable(raw):
            raw = await raw
        return raw
    coro = _async_call()
    manager_loop = getattr(context, "mcp_manager_loop", None)
    if manager_loop is not None and not manager_loop.is_closed():
        result = _run_on_loop(coro, manager_loop)   # 在 MCP 管理循环上跑
    else:
        result = _run_async(coro)                    # 或新建循环跑
    # ⑦ 序列化结果（content blocks → dict 列表）
    out = _serialize_mcp_result(result)
    return ToolResult(name="MCP", output={"server": server, "tool": tool_name, "output": out})
```

**关键点**：
- **客户端从哪来**：`context.mcp_clients`（主题 1 学过 ToolContext 的 `mcp_clients` 字段）——服务器连接由 MCP 管理器维护，工具执行时按 `server` 名取出客户端
- **事件循环处理**：MCP stdio 传输属于特定事件循环，所以用 `_run_on_loop`（在管理循环上 `run_until_complete`，mcp.py:240-254）或 `_run_async`（新建循环，mcp.py:257-276）——**同步的 dispatch 环境调异步的 MCP**

#### 3.5.4 per-server 包装（tool_wrapper.py）

服务器连上后，它暴露的**每个工具**被包装成独立 Tool（`tool_wrapper.py`）：

```python
# tool_wrapper.py 的核心逻辑（节选）
def wrap_mcp_tool(server, schema) -> Tool:
    return build_tool(
        name=build_mcp_tool_name(server.name, schema.name),  # mcp__github__search_code
        input_schema=schema.input_schema,                    # 服务器给的 schema
        call=make_call(server, schema),                      # 调服务器
        validate_input=...,                                   # jsonschema 校验（WI-8.1）
        max_result_size_chars=MAX_RESULT_SIZE_CHARS,          # 100,000（WI-8.4）
        is_mcp=True,
    )
```

**注意**：`tool_wrapper.py` 从 `src.tool_system.build_tool` 导入（59 行）——说明 MCP 包装部分用的是**上游 vendored** 的 build_tool，与 `clawcodex_ext` 的是同一个概念。

#### 3.5.5 MCP 工具怎么进 ToolRegistry

`assemble_tool_pool`（registry.py:566-618）把内置 + MCP 拼在一起：

```python
def assemble_tool_pool(registry, permission_context, mcp_tools=None):
    """组装发给 API 的完整工具池。内置优先，MCP 在后。
    内置工具排前面、MCP 排后面——内置同名优先（先注册者胜）。"""
    all_tools = get_all_base_tools(registry)         # ① 内置工具
    allowed = filter_tools_by_deny_rules(all_tools, permission_context)  # ② 过滤拒绝规则
    result = [t for t in allowed if t.is_enabled()]  # ③ 只留启用的
    if not mcp_tools:
        return result
    allowed_mcp = filter_tools_by_deny_rules(mcp_tools, permission_context)  # ④ MCP 也过滤
    allowed_mcp.sort(key=lambda t: t.name)           # ⑤ MCP 按名字排序
    for t in allowed_mcp:
        if not any(tool_matches_name(x, t.name) for x in result):
            result.append(t)                         # ⑥ 内置没有的同名才加（内置优先）
    return result
```

**完整流程**：

```
MCP 服务器（github / database / browser...）
  │ 连接（connection_manager 维护，存进 context.mcp_clients）
  ▼
每台服务器暴露的工具列表（list_tools）
  │
  ├─ 通用路径: 一个 MCP 工具（手动指定 server+tool）    mcp.py:312
  └─ 包装路径: 每工具一个 mcp__server__tool             tool_wrapper.py
        │
        ▼
assemble_tool_pool(registry, perm_ctx, mcp_tools)     registry.py:566
  │ 内置 + MCP 拼接（内置优先，MCP 排后）
  ▼
发给模型（tools= 参数）→ 模型看到 mcp__xxx__yyy 工具
  │
  ▼
模型调 mcp__github__search_code → dispatch → per-server wrapper 的 call
  │
  └─ client.call_tool("search_code", args) → MCP 服务器 → 结果返回
```

#### 3.5.6 MCP 在提示词里的体现（呼应主题 2）

主题 2 学过两个 MCP 相关 section（`_build_mcp_section` / `_build_mcp_instructions_section`）：
- `mcp` section（SESSION scope）：列出连接的 MCP 服务器（模型知道有哪些"外挂"）
- `mcp_instructions` section（SESSION scope）：服务器自带的工具使用说明

**所以 MCP 的完整链路跨两个系统**：
- **Context 系统**（主题 2）：`mcp`/`mcp_instructions` section 告诉模型"有哪些 MCP 服务器 + 怎么用"
- **Tool 系统**（本章）：`MCPTool` / `mcp__server__tool` 让模型"真正调用"

#### 3.5.7 具体参数示例

```python
# 假设连了 github 服务器（暴露 search_code / list_repos 两个工具）
# 模型看到的工具列表（tools= 参数）:
tools = [
    ...内置 47 个...,
    Tool(name="mcp__github__list_repos", ...),    # per-server 包装
    Tool(name="mcp__github__search_code", ...),   # per-server 包装
    Tool(name="MCP", ...),                        # 通用 MCP 工具（兜底）
]

# 模型调 search_code:
{"name": "mcp__github__search_code", "input": {"query": "bug fix", "repo": "myapp"}}

# dispatch 后:
#   wrapper.call → client.call_tool("search_code", {"query": "bug fix", ...})
#   → GitHub MCP 服务器 → 返回结果 → 序列化 → 回填 messages
```

**一句话**：MCP = 工具系统的"外挂源"——服务器连上后，它的工具要么包装成 `mcp__server__tool` 独立工具、要么走通用 `MCP` 工具（server+tool 参数），最终通过 `assemble_tool_pool` 拼进工具池发给模型。**内置 47 个工具是"自带的手"，MCP 是"外接的手"。**

---

## 4. 架构关系：Tool 在整个程序里的位置

### 4.1 三句话定位

1. **Tool 是"手"，模型是"嘴"**：模型输出 `tool_use`（说"我要读文件"），`dispatch` 执行真实操作，结果喂回模型。
2. **ToolRegistry 是"工具库"**：`build_tool` 声明工具 → `register` 入库 → `get_tools` 给模型看（拼进提示词）→ `dispatch` 执行。
3. **工具系统与 Context 系统互补**：Context 负责"展示"（工具 schema 拼进提示词），Tool 系统负责"执行"（dispatch 调 call）——**展示和执行分离**。

### 4.2 与主循环的配合

```
query() 主循环（引擎）
  │
  ├─ 拼上下文（主题2）───────────────┐
  │   _build_tool_docs_section()      │   ← 从 ToolRegistry.get_tools() 拿工具
  │   → 工具 schema 拼进提示词         │      序列化成模型能看的
  │                                   │
  ├─ 调模型（主题5 Provider）─────────┤
  │   模型返回 tool_use(name="Read",  │
  │     input={"path": "config.yaml"})│
  │                                   │
  └─ 执行工具（本章）─────────────────┘
      _run_tools_partitioned()         ← 主循环 Phase 4
        └─ ToolRegistry.dispatch(call, context)   ← 本章核心
              ├─ 找工具 → 校验 → 权限
              └─ tool.call(input, context) → ToolResult
                    └─ 结果回填 messages（主题1 Phase 6）
```

### 4.3 主题 2 的呼应

> 主题 2 留的问题："如果工具系统改了某个工具的 description，模型下次看到的是什么？" 答案：**`_build_tool_docs_section` 每次拼装时重新读 `ToolRegistry.get_tools()`，动态反映变更**——你改了工具描述，下轮拼提示词就更新。

---

## 5. 引出下一章：Memory（记忆）

本章你已经看到工具是"手"。但问题来了：

**模型每次对话都是"失忆"的——它只看到当前 messages，记不住跨会话的事。agent 怎么"记住"用户偏好、项目历史？**

答案就是主题 4：**Memory（记忆）**。

> **预告**：`clawcodex_ext/memdir/` 是记忆系统——用 Markdown 文件存记忆（`~/.claude/memory/`），模型通过工具读写。主题 2 学过 `_build_memory_section` 把记忆拼进提示词；本章的工具系统里有 `memory.py` 工具（读写记忆）。**记忆 = 文件 + 工具 + 提示词 section 三者的配合**。

留一个问题给下一章：如果记忆存在文件里，模型怎么"想起"该读哪个文件？（提示：记忆预取 memory_prefetch.py 会按相关性挑选记忆塞进上下文。）

---

## 6. 练习与反问

### 动手练习

1. 打开 `build_tool.py`，找出 `Tool` 的**必填字段**（没有默认值的）和**可选字段**（有默认值的）。必填只有哪几个？

2. 打开 `registry.py` 的 `register`，回答：
   - 重复工具名会怎样？（抛 ValueError）
   - 别名冲突会怎样？（也抛 ValueError）
   - 为什么工具名冲突是硬错误，而 agent 注册表是 last-wins？（提示：分发歧义）

3. 打开 `tool_system/tools/read.py`，看它怎么用 `build_tool` 声明 Read 工具——找出 name、input_schema、call、is_read_only 的值。

4. 运行 `cd clawcodex-ascend && uv run clawcodex-dev -p "帮我数一下当前目录有几个文件"`，观察：
   - 模型调了哪个工具？（可能是 Bash 或 Glob）
   - 工具结果怎么显示的？

### 反问习题（自测）

**Q1.** `Tool.is_read_only` 是 bool 还是函数？为什么？

<details><summary>答案</summary>
是**函数**（`Callable[[dict], bool]`）不是 bool——运行时才求值，可以依赖输入动态判断（比如"读 /tmp 临时文件"可能算只读，读项目文件不算）。`build_tool` 不传时默认 `lambda _input: False`（不认为只读）。
</details>

**Q2.** `build_tool` 的 description 不传会怎样？

<details><summary>答案</summary>
`desc_fn` 返回 `name`（工具名）当描述。模型看到的工具介绍就是工具名本身。所以建议传 description——名字太短，模型不知道什么时候该用。
</details>

**Q3.** `ToolRegistry.register` 遇到重复名为什么抛异常，而不是 last-wins？

<details><summary>答案</summary>
工具分发是"按名字精确查找"（`_by_name`），重名会歧义——模型说调 `read_file`，系统不知道该执行哪个。所以重名是硬错误（fail fast），而不是静默覆盖。agent 注册表 last-wins 是因为 agent 是可选的（找得到就行），工具是必需的（歧义致命）。
</details>

**Q4.** dispatch 的安全门顺序为什么是"找工具 → 校验 → 权限"，权限放最后？

<details><summary>答案</summary>
权限检查最贵（可能弹窗问用户）。先做便宜的检查（找工具/校验输入）过滤掉明显错误，最后才问权限——避免"为无效调用弹窗"。
</details>

**Q5.** 工具系统改了 description，模型下次看到的是什么？

<details><summary>答案</summary>
`_build_tool_docs_section` 每次拼装时重新读 `ToolRegistry.get_tools()`——所以下次拼提示词就是新描述。工具的"展示"（Context）和"执行"（Tool 系统）分离，改一边不影响另一边。
</details>

---

## 7. 本章小结（一张表）

| 你要知道的事 | 一句话 |
|-------------|--------|
| Tool 是什么 | agent 的"手"——模型输出 tool_use，工具系统执行 |
| 核心数据类 | `Tool`（build_tool.py:65）：name/schema/call/权限/元信息 |
| 怎么造工具 | `build_tool(name, input_schema, call, ...)`（144）——必填 3 个，其余默认 |
| 怎么注册 | `ToolRegistry.register(tool)`（241）——重名抛异常 |
| 怎么查找 | `find_tool_by_name(name)` —— `_by_name` 字典 O(1) |
| 怎么执行 | `dispatch(call, context)`（341）——5 道安全门 + 调 call |
| 内置工具 | 47 个（tools/ 目录：read/write/glob/grep/bash/agent/skill...） |
| 和谁配合 | Context 展示（_build_tool_docs_section）↔ Tool 执行（dispatch） |
| 下一章预告 | 记忆工具 → 主题 4 Memory（文件+工具+section 配合） |

---

*下一主题：[主题 4：Memory（记忆）](learning_plan.md#主题-4memory记忆)—— agent 怎么记住跨会话的事。*
