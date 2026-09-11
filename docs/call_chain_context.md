# 附属文档：Context 拼装调用链完整解析

> 所属：主题 2（Context）学习文档的附属资料
> 目标读者：想彻底搞懂"系统提示词从哪来、怎么拼、怎么流进引擎"的初学者
> 学习方法：**先整体后局部**——先看拼装总览（整体），再钻进每个函数（局部）；代码块带详细注释（作用 + 返回值）。
> 前置知识：主题 1 的 `call_chain_main_agent.md`（知道主 agent 调用链）；主题 2 主文档（知道 section 概念）。

---

## 第 0 章：拼装总览——系统提示词从哪来

**核心问题**：模型看到的整段系统提示词，是谁、在哪、怎么拼出来的？

**一句话答案**：`context_system/` 的**提示词组装流水线**——各模块的数据（工具/目录/git/记忆/技能）通过 builder 变成 section，`build_full_system_prompt_blocks()` 拼装排序分组，最终成为 `QueryParams.system_prompt` 发给模型。

**完整调用链（一图流）**：

```
headless / TUI（前端）
  │  调用
  ▼
build_effective_system_prompt(style_prompt, tool_context, ...)   agent_loop_compat.py:249
  │  ├─ 非 coordinator 路径:
  │  │    build_full_system_prompt_blocks(cwd, skills, ...)      prompt_assembly.py:684
  │  │      ├─ 19 个内置 _build_*_section（每个产出 section 或 None）
  │  │      ├─ collect_new_sections(ctx)（注册表里新加的）
  │  │      ├─ 排序 → 按 CacheScope 分组
  │  │      └─ _emit_group 转 block（组尾打 cache_control）
  │  ├─ coordinator 路径: 用协调器提示词替换基础块
  │  └─ 末尾: build_context_prompt() 拼 workspace+git+CLAWCODEX.md（不缓存）
  ▼
返回 block 列表 → 存到 effective_system_prompt
  │
  ▼
_run_one_agent_loop → run_query_as_agent_loop(system_prompt=effective_system_prompt)  headless.py:2190
  │
  ▼
_make_params → QueryParams(system_prompt=...)       agent_loop_compat.py:700（闭包捕获）
  │
  ▼
query() → _query_impl → _call_model_sync(system_prompt=...)   query.py:2418 → 发给模型
```

**另一条路径（engine/REPL）**：`engine.py` 的 `_build_system_prompt_parts()`（engine.py:147）——当调用方没预设 system_prompt 时，它做 canonical 拼装，结果一样进 `QueryParams.system_prompt`。

**记忆口诀**：**拼装（build_effective_system_prompt）→ 拼块（build_full_system_prompt_blocks）→ 建参（_make_params）→ 发送（_call_model_sync）**。

---

## 第 1 章：`build_effective_system_prompt` —— 拼装入口（封装器）

### 1.1 这一层是干嘛的（整体）

**它是 headless/TUI 路径的"总封装"**——负责：
1. 调 `build_full_system_prompt_blocks` 拿基础块（intro/system/.../skills）
2. 末尾**追加 workspace+git+CLAWCODEX.md** 作为不缓存的尾部块
3. 处理 coordinator 模式（协调器提示词替换基础块）

**为什么要这层**（docstring 269-275 行解释）：TUI/headless 走 `query()` 时 `params.system_prompt` **原样透传**——如果没人拼，模型就**收不到任何基础指令**。这个 helper 恢复了基础块。

**伪代码**：

```python
def build_effective_system_prompt(style_prompt, tool_context, *, provider, mcp_servers, query_source):
    if 是 coordinator 模式:
        blocks = [协调器提示词块]          # 替换整个基础块集
    else:
        skills = 查技能列表()
        blocks = build_full_system_prompt_blocks(   # ← 核心：基础块
            cwd=..., output_style="default",
            append_system_prompt=style_prompt,      # 输出样式追加
            query_source=..., provider=..., mcp_servers=..., skills=skills,
        )
    context_prompt = build_context_prompt(workspace_root, cwd)  # workspace+git+CLAWCODEX.md
    if context_prompt:
        blocks += [{"type": "text", "text": context_prompt, "_cache_scope": "request"}]
    return blocks
```

### 1.2 真实代码（非 coordinator 主路径，359-427）

```python
# ① 查技能列表（best-effort——查失败就 None，不致命）
try:
    from ..command_system import get_skill_tool_commands
    skills = get_skill_tool_commands(cwd)     # 返回值: 技能命令列表或 None
except Exception:
    skills = None                              # 查询失败 → 无技能

# ② 核心：调基础拼装（返回值: block 列表）
blocks = build_full_system_prompt_blocks(
    cwd=cwd,                                   # 工作目录
    output_style="default",                    # 样式后面追加（engine 路径的约定）
    append_system_prompt=style_prompt,         # 输出样式作为"追加提示词"传入
    query_source=query_source,                 # 来源标签（影响缓存 TTL 选择）
    provider=provider,                         # 供 GLOBAL 缓存 scope 门控
    mcp_servers=mcp_servers,                   # MCP 服务器（无则 None）
    skills=skills,                             # 技能列表
)
```

**注意**：这里**故意不传** `tools`/`tool_registry`（docstring 287-290 行）——工具 schema 走 API 的 `tools=` 参数，如果这里再拼工具文档就**双重发送**了。

```python
# ③ 尾部：拼 workspace+git+CLAWCODEX.md（不缓存）
try:
    context_prompt = build_context_prompt(     # 返回值: str（含 ## Project Instructions）
        tool_context.workspace_root,           # 项目根目录
        cwd=tool_context.cwd,                  # 当前目录
    )
except Exception:
    context_prompt = ""                        # 失败给空串

# ④ 追加为尾部块（REQUEST scope = 每次变 → 不缓存）
if context_prompt.strip():
    blocks = blocks + [
        {
            "type": "text",
            "text": context_prompt,
            "_cache_scope": CacheScope.REQUEST.value,   # 标记"请求级"（DeepSeek 会挪尾部）
        }
    ]

# ⑤ 返回完整 block 列表（返回值: list[dict]，每个 dict 一个文本块）
return blocks
```

### 1.3 Python 语法讲解

| 语法 | 说明 |
|------|------|
| `style_prompt: str` 非关键字参数 | 第一个位置参数（headless 传 `_style_prompt`） |
| `*, provider, mcp_servers, query_source` | `*` 后只能按关键字传 |
| `blocks = blocks + [...]` | 列表拼接（**新建列表**，不是原地 append）——保持原 blocks 不变 |
| `_cache_scope` 下划线字段 | 内部元数据，发送前剥离（Anthropic 路径） |

### 1.4 具体参数示例

```python
# headless.py:2088 的真实调用:
effective_system_prompt = build_effective_system_prompt(
    _style_prompt,                    # "输出样式提示词"（可能为空串）
    tool_context,                     # 工具上下文（含 workspace_root/cwd）
    provider=provider,                # 已建好的 BaseProvider
)

# 返回的 blocks（简化）:
[
    {"type": "text", "text": "你是 Claude Code..."},                    # 基础块（来自 build_full_...）
    {"type": "text", "text": "# Doing tasks\n..."},
    ...
    {"type": "text", "text": "## Project Instructions\n（CLAWCODEX.md 内容）",
     "_cache_scope": "request"},                                        # 尾部不缓存块
]
```

---

## 第 2 章：`build_full_system_prompt_blocks` —— 拼块核心（总装线）

### 2.1 这一层是干嘛的（整体）

**它是"总装线"**——把几十个 section 拼成 block 列表。4 步：**收集 → 排序 → 分组 → 打缓存标记**。

### 2.2 真实代码（核心段，765-921）

```python
def build_full_system_prompt_blocks(*, cwd, tools, tool_registry, agents, skills,
                                    mcp_servers, output_style, plan_mode, ...):
    # ① ctx 准备（透传字典，见主文档 3.1）
    ctx: dict[str, Any] = runtime_ctx or {}   # 调用方传的，没传用空字典
    ctx.setdefault("cwd", cwd)                # 至少保证有 cwd 键

    # ② 收集 sections（19 个内置 builder + 注册表新增）
    sections: list[SystemPromptSection] = []
    intro = _build_intro_section(use_cache, ctx)      # 返回值: section 或 None
    if intro:
        sections.append(intro)                        # 非 None 才加入
    system = _build_system_section(use_cache, ctx)
    if system:
        sections.append(system)
    # ...（省略 doing_tasks/actions/using_tools/tone/efficiency）...
    tool_docs = _build_tool_docs_section(tools, tool_registry, use_cache, ctx)  # 工具文档
    env_section = _build_env_section(cwd, use_cache, ctx)                        # 环境
    memory_section = _build_memory_section(ctx)                                  # 记忆
    agent_section = _build_agent_section(agents, use_cache, ctx)                 # 子 agent
    skill_section = _build_skill_section(skills, use_cache, ctx)                 # 技能
    # ...（省略 mcp/plan_mode/non_interactive 等条件性 builder）...

    # ③ 注册表新增的 section（插件注入）
    for new_sec in collect_new_sections(ctx):         # 返回值: list[SystemPromptSection]
        sections.append(new_sec)

    # ④ 排序（保证字节级稳定 → 缓存稳定）
    sections.sort(key=lambda s: s.order)              # 按 order 升序

    # ⑤ 按 CacheScope 分组
    global_sections  = [s for s in sections if s.cache_scope is CacheScope.GLOBAL  and s.content]
    session_sections = [s for s in sections if s.cache_scope is CacheScope.SESSION and s.content]
    request_sections = [s for s in sections if s.cache_scope is CacheScope.REQUEST and s.content]

    # ⑥ 转 block（组尾打缓存标记）
    blocks: list[dict[str, Any]] = []
    if global_sections:
        _emit_group(global_sections, CacheScope.GLOBAL)   # 每组最后一个 block 带 cache_control
    blocks.append({"type": "text", "text": SYSTEM_PROMPT_DYNAMIC_BOUNDARY})  # 动态边界标记
    if session_sections:
        _emit_group(session_sections, CacheScope.SESSION)
    if request_sections:
        _emit_group(request_sections, CacheScope.REQUEST)
    if append_system_prompt:
        blocks.append({"type": "text", "text": append_system_prompt})  # 主 agent 身份（不缓存）

    return blocks   # 返回值: list[dict]（{"type":"text","text":..., "cache_control"?:...}）
```

### 2.3 每步返回值

| 步骤 | 关键调用 | 返回值 |
|------|---------|--------|
| ① ctx 准备 | `runtime_ctx or {}` | dict（默认只有 cwd） |
| ② 收集 | `_build_*_section(...)` | `SystemPromptSection` 或 None（None 不加入） |
| ③ 注册注入 | `collect_new_sections(ctx)` | `list[SystemPromptSection]` |
| ④ 排序 | `sections.sort(key=...)` | 无（原地排序） |
| ⑤ 分组 | 列表推导 | 3 个 list（GLOBAL/SESSION/REQUEST） |
| ⑥ 转 block | `_emit_group(group, scope)` | 无（追加到外层 blocks） |

### 2.4 关键细节

- **`use_cache` 参数**：传给每个 builder——builder 内部决定"这次从缓存读还是重新生成"
- **`custom_system_prompt` 分支**（731-758 行）：如果调用方给了自定义提示词，**跳过整个 section 拼装**，只返回一个不缓存块
- **`append_system_prompt`**：追加在最后（不缓存）——主 agent 身份走这里

---

## 第 3 章：各 `_build_*_section` —— 拼装的"工位"

### 3.1 通用模式（每个 builder 都这样）

```python
def _build_xxx_section(参数..., runtime_ctx=None) -> SystemPromptSection | None:
    # ① 先 consult 注册表（看有没有覆盖）——注意：不是所有 builder 都有这步
    override = consult_section_builders("xxx", runtime_ctx)   # 返回值: section 或 None
    if override is not None:
        return override                      # 注册的覆盖了内置

    # ② 默认逻辑：生成内容
    content = 生成内容()
    if not content:
        return None                          # 内容为空 → 抑制（不显示）

    # ③ 打包返回
    return SystemPromptSection(
        id="xxx",
        content=content,
        cache_scope=CacheScope.???,
        order=???,
    )
```

### 3.2 两个典型例子

**① `_build_env_section`（1334-1373）——环境信息**：

```python
def _build_env_section(cwd, use_cache, runtime_ctx=None):
    override = consult_section_builders("environment", runtime_ctx)   # 查覆盖
    if override is not None:
        return override

    target = cwd or os.getcwd()              # 没给 cwd 用当前目录
    parts = ["# Environment"]
    parts.append(f"- CWD: {target}")         # 工作目录
    parts.append(f"- OS: {platform.system()} {platform.release()}")   # 系统
    parts.append(f"- Date: {_get_session_start_date_iso()}")          # 会话开始日期（缓存安全）
    parts.append(f"- Shell: {os.environ.get('SHELL', 'unknown')}")    # shell
    ...
    content = "\n".join(parts)
    # 返回值: REQUEST scope（环境每次请求都可能变，如 cwd/date）
    return SystemPromptSection(id="environment", content=content,
                               cache_scope=CacheScope.REQUEST, order=20)
```

> **注意 Date 的缓存安全设计**（1350-1357 行注释）：REQUEST 组末尾带 cache_control，如果 Date 用"每秒钟的时间戳"会**每轮 bust 缓存**（4 个缓存点之一浪费）。所以用 `_get_session_start_date_iso()`——**会话开始时的日期**，会话内稳定。

**② `_build_memory_section`（1376-1409）——记忆**：

```python
def _build_memory_section(runtime_ctx=None):
    # 先 consult 注册表（P119-A）——memory 的注册 builder 在这被消费
    override = consult_section_builders("memory")
    if override is not None:
        return override                      # 注册的覆盖了默认

    # 默认: 读 MEMORY.md
    content = load_memory_prompt()           # 读自动记忆文件
    if not content:
        return None                          # 没记忆 → 抑制
    # 返回值: REQUEST scope（MEMORY.md 会话中会被模型改写）
    return SystemPromptSection(id="memory", content=content,
                               cache_scope=CacheScope.REQUEST, order=25)
```

### 3.3 builder 一览表（作用 + 返回值 + scope）

| builder | section | 内容 | scope | 返回 None 时 |
|---------|---------|------|-------|-------------|
| `_build_intro_section` | intro | "你是 Claude Code..." | GLOBAL | 极少（身份总有） |
| `_build_system_section` | system | 系统规则 | GLOBAL | 极少 |
| `_build_doing_tasks_section` | doing_tasks | 做事准则 | GLOBAL | 极少 |
| `_build_tool_docs_section` | tool_docs | 工具文档 | SESSION | 无工具时 |
| `_build_env_section` | environment | 目录/OS/日期/shell | REQUEST | 极少 |
| `_build_memory_section` | memory | 记忆内容 | REQUEST | 无记忆时 |
| `_build_mcp_section` | mcp | MCP 服务器 | SESSION | 无 MCP 时 |
| `_build_agent_section` | agents | 子 agent 列表 | SESSION | 无 agent 时 |
| `_build_skill_section` | skills | 技能列表 | SESSION | 无技能时 |
| `_build_plan_mode_section` | plan_mode | 计划模式 | REQUEST | 非 plan 模式时 |
| `_build_non_interactive_section` | non_interactive | 非交互模式 | REQUEST | 交互模式时 |

> **规律**：`None` 返回 = "该 section 当前不适用"——builder 用返回值告诉拼装线"这轮别放我"。

---

## 第 4 章：section 注册机制 —— 插件怎么接入

### 4.1 注册与消费（两条路径）

```
register_section(id, builder, order, cache_scope, tags)   section_registry.py:164
  └─ 存进 _registry 字典（id → RegisteredSection）
        │
        ├─ id 匹配内置（如 "memory"）
        │     └─ 内置 builder 调 consult_section_builders(id) 消费   section_registry.py:214
        │           └─ 返回: SystemPromptSection 或 None（覆盖/回退默认）
        │
        └─ id 不在内置表（如 "weekday"）
              └─ 拼装末尾 collect_new_sections(ctx) 消费          section_registry.py:248
                    └─ 返回: list[SystemPromptSection]（注入新 section）
```

### 4.2 真实注册示例（memory，scope_aware_prompt.py）—— 完整步骤拆解

以 memory 模块的真实代码为例，讲透"一个 section 从写 builder 到被拼进 prompt"的**每一步**。这是你自己写项目时最值得照抄的模板。

#### 4.2.1 全景：这个文件做了什么

`scope_aware_prompt.py` 做的事：**让"记忆提示词"按 scope（user/project/local）分层拼接**。它完全不改上游拼装代码，只通过注册一个 `memory` section 的 builder 接入。整个文件只有 3 个层次：

```text
① 配置层:  _default_memory_scopes = ["user", "project", "local"]     ← 默认哪些 scope
② 业务层:  build_scope_aware_memory_prompt(scopes) -> str|None        ← 真正干活
③ 注册层:  install_memory_extension() -> register_section(...)        ← 接入拼装线
```

#### 4.2.2 第一步：配置层——模块级常量（scope_aware_prompt.py:44-50）

```python
# 合法的 scope 集合（用于校验）
VALID_MEMORY_SCOPES: frozenset[str] = frozenset({"user", "project", "local"})

# 默认启用的 scope（可在启动时用 set_default_memory_scopes() 覆盖）
_default_memory_scopes: list[str] = ["user", "project", "local"]
```

**为什么先写配置**：你的 builder 通常需要"参数"。**builder 的契约是 `(runtime_ctx) -> str | None`——它只接收 runtime_ctx 一个参数**，所以额外的配置必须通过**模块级变量**或**闭包**传入，不能靠 builder 参数。

> **你写项目时**：如果你的 section 需要配置（比如"显示哪些数据源"），照抄这个模式——模块级常量 + `set_xxx()` 覆盖函数。

#### 4.2.3 第二步：业务层——真正的 builder 函数（scope_aware_prompt.py:84-126）

```python
def build_scope_aware_memory_prompt(memory_scopes: list[str] | None = None) -> str | None:
    """构建"按 scope 分层的记忆提示词"。
    返回值: str（拼好的记忆文字）或 None（无可显示的记忆）"""
    if not memory_scopes:
        return None                      # ① 无 scope → 直接返回 None（抑制）

    validated = _validate_scopes(memory_scopes)   # ② 过滤非法 scope（警告但不抛错）
    if not validated:
        return None                      # ③ 全非法 → 返回 None

    try:
        from clawcodex_ext.memdir.memdir import load_memory_prompts  # 懒导入
    except ImportError:
        return None                      # ④ 依赖不可用 → 返回 None

    try:
        prompts = load_memory_prompts(memory_scopes=validated)  # 返回值: list[str]
    except Exception:
        logger.exception("Failed to load scope-aware memory prompts")
        return None                      # ⑤ 加载失败 → 返回 None

    if not prompts:
        return None                      # ⑥ 没记忆 → 返回 None

    combined = "\n\n".join(prompts)      # ⑦ 多个 scope 的记忆拼成一段
    return combined                      # 返回值: str（非空）
```

**关键观察——builder 的"退出策略"**：这个函数有 **6 个返回 None 的分支**，只有最后 1 个返回 str。这是 builder 的**核心设计**：

| 返回 | 含义 | 拼装线处理 |
|------|------|-----------|
| `str`（非空） | "有内容，显示我" | 拼进 prompt |
| `None` | "当前无内容/不适用" | 回退默认（不覆盖）或抑制（不注入） |
| `""`（空串） | "显式清空"（一般不直接用） | 等于删掉该 section |

> **你写项目时**：你的 builder 一定要**仔细设计 None 分支**——"什么时候不显示"和"什么时候显示"同样重要。比如"天气 section"：今天有数据返回文字，查不到就返回 None（而不是返回"查询失败"的废话）。

#### 4.2.4 第三步：注册层——install 函数（scope_aware_prompt.py:129-140）

```python
def install_memory_extension() -> None:
    """注册 scope-aware 记忆 builder 到拼装注册表。幂等——可多次调用。"""
    # ① 懒导入 register_section（避免模块加载期的循环依赖）
    from clawcodex_ext.context_system.section_registry import register_section

    # ② 包一层闭包——适配 builder 契约 (runtime_ctx) -> str|None
    def _wrapped(_ctx: dict) -> str | None:
        # 注意: _ctx 是 runtime_ctx（当前几乎只有 cwd）
        #       这里用不到它，所以命名 _ctx（下划线 = 忽略）
        prompt = build_scope_aware_memory_prompt(_default_memory_scopes)
        return prompt                    # str | None（None = 不覆盖默认）

    # ③ 注册——覆盖内置 memory section
    register_section("memory", builder=_wrapped)
```

**逐步解释**：

| 行 | 作用 | 为什么 |
|----|------|--------|
| ① 懒导入 `register_section` | 函数内 import | 避免循环依赖（`section_registry` 可能反向依赖本模块） |
| ② 定义 `_wrapped` 闭包 | 适配 `(runtime_ctx)->str|None` 契约 | `build_scope_aware_memory_prompt` 的签名是 `(memory_scopes)`，不匹配 builder 契约——用闭包包一层，把模块级配置 `_default_memory_scopes` 传进去 |
| ③ `register_section("memory", builder=_wrapped)` | 注册 | `id="memory"` 匹配内置 → 走 **consult 覆盖**路径 |

**`_wrapped` 为什么这样写**（闭包的价值）：

```python
# 不这样写（签名不匹配）:
def _wrapped(_ctx):                      # builder 契约: 接收 runtime_ctx
    return build_scope_aware_memory_prompt(_default_memory_scopes)   # ← 闭包捕获 _default_memory_scopes
```

**`_wrapped` 捕获了外层 `_default_memory_scopes`**——即使后来有人调 `set_default_memory_scopes(["user"])` 改了模块级变量，`_wrapped` 每次调用都读**当前值**。这就是闭包 vs 硬编码的区别。

> **你写项目时**：如果你的 builder 需要参数，两种适配方式：
> 1. **闭包捕获**（memory 用的）：`def _wrapped(_ctx): return my_builder(my_config)`——配置在模块级
> 2. **从 runtime_ctx 读**：`def _wrapped(ctx): return my_builder(ctx.get("my_key"))`——配置由调用方塞进 runtime_ctx

#### 4.2.5 第四步：它怎么被消费（回到 `_build_memory_section`）

注册后，拼装时 `_build_memory_section`（prompt_assembly.py:1391-1394）会查注册表：

```python
def _build_memory_section(runtime_ctx=None):
    # 先查注册表——memory 的注册 builder 在这被消费
    override = consult_section_builders("memory")     # 返回值: section 或 None
    if override is not None:
        return override                               # ← 注册的 _wrapped 的结果在这返回

    # 默认逻辑（注册不存在时走这里）
    content = load_memory_prompt()                    # 读 MEMORY.md
    ...
```

`consult_section_builders`（section_registry.py:214）内部：

```python
def consult_section_builders(section_id, runtime_ctx=None):
    sec = _registry.get(section_id)      # ① 查注册表
    if sec is None:
        return None                      # 没注册 → None
    try:
        content = sec.builder(runtime_ctx or {})   # ② 调注册的 builder（即 _wrapped）
    except Exception:
        return None                      # builder 抛异常 → None（回退默认）
    if content is None:
        return None                      # ③ builder 返回 None → 回退默认
    # ④ builder 返回 str → 包装成 section 返回
    return SystemPromptSection(id=section_id, content=content, ...)
```

**完整闭环**：

```
install_memory_extension()
  └─ register_section("memory", builder=_wrapped)     # 注册（一次）
        └─ _registry["memory"] = RegisteredSection(...)
              │
              ▼ （每次拼装）
        _build_memory_section(ctx)
          └─ consult_section_builders("memory")       # 查表
                └─ _wrapped(ctx) → build_scope_aware_memory_prompt(...)
                      └─ 返回 str（拼好的记忆）或 None（无记忆）
                            │ str → SystemPromptSection(memory, content, REQUEST, 25)
                            └─ None → 回退默认（读 MEMORY.md）
```

#### 4.2.6 自己写一个项目的完整模板（照抄版）

```python
# my_section.py —— 你要写的新 section
from __future__ import annotations

# ① 配置层（可选）
_default_show_detail: bool = True          # 你的 section 的配置

def set_show_detail(flag: bool) -> None:   # 配置覆盖函数（可选）
    """启动时调用，覆盖默认配置。"""
    global _default_show_detail
    _default_show_detail = flag

# ② 业务层（核心 builder）
def build_my_content(show_detail: bool) -> str | None:
    """生成你的 section 内容。
    返回值: str（显示内容）或 None（不显示）"""
    data = 获取数据()                        # 你的数据来源
    if not data:
        return None                         # 无数据 → 抑制（关键！）
    lines = ["# My Section", f"- data: {data}"]
    if show_detail:                          # 配置影响内容
        lines.append(f"- detail: {更多细节()}")
    return "\n".join(lines)

# ③ 注册层（install 函数）
def install_my_section() -> None:
    """注册到拼装注册表。幂等。"""
    from clawcodex_ext.context_system.section_registry import register_section

    def _wrapped(_ctx: dict) -> str | None:  # 适配 builder 契约
        return build_my_content(_default_show_detail)   # 闭包捕获配置

    register_section(
        id="my_section",                    # 全新 id → 走 collect 注入路径（必收集）
        builder=_wrapped,
        order=35,                           # 排序位置（可查 _CANONICAL_ORDER 选空位）
        cache_scope=SectionScope.SESSION,   # 缓存范围（可选，默认查表→SESSION）
        tags=["custom"],                    # 标签（可选）
    )

# ④ 接入调度（二选一）
# 方式A（推荐，官方模式）: 在 clawcodex_ext/__init__.py 的
#   ensure_eager_extensions_installed() 里加:
#     from my_section import install_my_section
#     install_my_section()
# 方式B（简单）: 保证模块被 import（如你的扩展包 __init__.py 里
#   import my_section 即可触发模块级注册）
```

**验证你的 section 生效**（自己测试）：

```python
# 测试脚本（临时跑一下）:
from clawcodex_ext.context_system.section_registry import register_section, collect_new_sections
from clawcodex_ext.context_system import prompt_assembly

register_section(
    id="my_section",
    builder=lambda _ctx: "# My Section\n- test data",
    order=35,
)

blocks = prompt_assembly.build_full_system_prompt_blocks(cwd="/tmp", tools=[], agents=[], skills=[])
texts = [b["text"] for b in blocks]
assert any("# My Section" in t for t in texts), "section 没拼进去!"
print("✅ my_section 已进入 prompt")
```

#### 4.2.7 你写项目时最容易踩的 5 个坑

| 坑 | 现象 | 解法 |
|----|------|------|
| **builder 返回空串 `""`** | section 消失（或被 collect 过滤） | 无内容时返回 `None` 而不是 `""` |
| **id 用了内置名但内置不 consult** | 注册了但永不生效 | 新功能用**全新 id**（走 collect 必收集） |
| **注册太晚** | 本轮拼装没看到你的 section | 注册必须在首次 `build_full_system_prompt_blocks` 前 |
| **builder 抛异常** | consult/collect 捕获后返回 None → section 静默消失 | builder 内部 try/except，返回 None 兜底 |
| **配置写死在 builder 里** | 改配置要改代码 | 用模块级变量 + set_xxx()（照抄 memory 模式） |

### 4.3 谁触发注册（懒安装调度）

```
ensure_eager_extensions_installed()      clawcodex_ext/__init__.py:97
  └─ install_memory_extension()          __init__.py:117
        └─ register_section("memory", ...)
```

**关键**：`_eager_extensions_installed` 标志保证**只装一次**（幂等）；在**所有 src 模块加载完后**调用（避免循环导入）。

### 4.4 新增一个 section 的操作（4 道关卡回顾）

| 关卡 | 检查 | 不通过会怎样 |
|------|------|-------------|
| 1. id 是否匹配内置 | 匹配内置 → 走 consult（**不一定被消费**）；不匹配 → 走 collect（必收集） | 内置 id 但 builder 不 consult → 白注册 |
| 2. builder 返回 None | None = 不覆盖/不注入 | section 不出现 |
| 3. collect 的 `if content:` | 空串/None 过滤 | 空 section 不加入 |
| 4. 注册时机 | 必须在首次拼装前 | 注册晚 → 本轮不生效 |

---

## 第 5 章：反问题（自测）

**Q1.** `build_effective_system_prompt` 为什么不传 `tools` 给 `build_full_system_prompt_blocks`？

<details><summary>答案</summary>
工具 schema 已经通过 API 的 `tools=` 参数发给模型（`_call_model_sync(tools=...)`）。如果这里再拼工具文档 section，就会**双重发送**——浪费 token 且可能不一致。docstring 287-290 行明确"deliberately NOT passed"。
</details>

**Q2.** `_build_env_section` 的 Date 为什么用"会话开始日期"而不是当前时间？

<details><summary>答案</summary>
REQUEST 组的末尾 block 带 cache_control（4 个缓存点之一）。如果 Date 用每秒时间戳，会**每轮 bust 缓存**——写一个 25% 的缓存条目却从没被读回。会话开始日期在会话内稳定，既给了模型日期信息又不破坏缓存。
</details>

**Q3.** `build_effective_system_prompt` 末尾的 workspace/git/CLAWCODEX.md 块为什么标记 REQUEST scope 且不缓存？

<details><summary>答案</summary>
它是"实时工作区快照"——嵌入了 git status 和文件统计，agent 一编辑文件就变。如果放进缓存前缀，一次编辑就 bust 整个前缀（DeepSeek 的自动前缀缓存）。标记 REQUEST 让 DeepSeek 把它挪到尾部；Anthropic 路径发送前剥离标记，行为不变。
</details>

**Q4.** 两条拼装路径（`build_effective_system_prompt` vs `engine._build_system_prompt_parts`）什么关系？

<details><summary>答案</summary>
`build_effective_system_prompt` 是 TUI/headless（cutover）路径——它们预设了 system_prompt，query() 原样透传，所以需要这个 helper 补基础块。`engine._build_system_prompt_parts` 是 engine/REPL 路径——调用方没预设时它做 canonical 拼装。两条路径最终都进 `QueryParams.system_prompt`，**殊途同归**。
</details>

**Q5.** 注册一个 `id="intro"` 的 section，会生效吗？

<details><summary>答案</summary>
**大概率不生效**。`_build_intro_section` 内部如果没调 `consult_section_builders("intro")`，注册表里的 intro 永远不会被消费（关卡 1）。目前明确 consult 的是 `memory`（`_build_memory_section` 1392 行）。所以新加 section 建议用**全新 id**（走 collect 必收集路径）。
</details>

---

*下一站：主题 3（Tool）—— 工具的声明、注册、分发。*
