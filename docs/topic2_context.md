# 主题 2：Context —— 模型看到的"整个世界"

> 学习顺序：**Agent → Context → Tool → Memory → LLM Provider → Session → Skill → SubAgent**
> 本章目标：看懂系统提示词是怎么从一堆"片段（section）"拼成模型看到的那一整段文字；理解 section 注册机制、cache boundary 分层。
> 前置：主题 1 已掌握 `AgentDefinition.get_system_prompt()` 只是"候选提示词"——本章回答"父 agent 的系统提示词是谁拼的"。
> 阅读代码前先跑一遍：`cd clawcodex-ascend && uv run clawcodex-dev -p "请介绍一下你自己"`，看它怎么描述自己的身份和规则。

---

## 0. 承上启下：从前一章到本章

### 0.1 前一章回顾：主题 1（Agent）学了什么

**主题 1 的核心问题**：agent 是什么？——答案是"**配置包**"：给同一个引擎（`query()` 主循环）换上不同的配置（提示词 + 工具 + 模型），就有不同行为的 agent。

| 维度 | 主题 1 学到的 | 一句话 |
|------|--------------|--------|
| **数据结构** | `AgentDefinition` | 一个 agent = 一张"程序卡"（agent_type / when_to_use / tools / model / get_system_prompt） |
| **注册机制** | `AgentRegistry` + `@register` 装饰器 | 导入即注册，同名 last-wins，`find()` O(1) 查找 |
| **来源分层** | 5 条通道 + 优先级阶梯 | built-in < plugin < clawcodex_ext < extensions < user < project < managed |
| **提示词** | `get_agent_system_prompt()` 三级降级 | fork 继承 > 自带 > 默认兜底 |
| **两条链路** | 主链路（CLI→桥接层→query）vs 子链路（Agent 工具→run_agent→query） | 都汇入同一个引擎 |
| **主循环** | `_query_impl` 的 7 个 Phase | 推理→行动→观察，每轮重建快照 |

**主题 1 的设计架构与理念**：

| 设计 | 理念 |
|------|------|
| **配置与执行分离** | `AgentDefinition` 只是数据（定义时），引擎 `query()` 才是执行（运行时）——"定义时/运行时"视角区分贯穿全文 |
| **导入即注册** | 装饰器副作用触发注册，`import` 一次 `find()` 就可用——解耦：引擎不认识具体 agent，只认注册表 |
| **安全底线不由调用者决定** | `_normalise_disallowed` 自动合并全局禁用工具——黑名单是"强制叠加"不是"可选" |
| **快照隔离** | `QueryState` 每轮重建，`create_subagent_context` 子 agent 隔离——"能干活但不干扰父" |
| **钩子解耦** | 7 个 `_goal_*` 翻译函数 + `TerminalHolder`——主循环零感知外部系统 |

> **一句话记住主题 1**：agent = 配置包；引擎 = 同一个 `query()`；一切靠"注册表 + 钩子"解耦。

### 0.2 本章设计架构：Context（上下文）

**本章的核心问题**：主题 1 里 `get_agent_system_prompt()` 只是子 agent 的"候选提示词"，那**主 agent 的系统提示词——模型真正看到的整段文字——是谁拼的？** 答案是本章的 `context_system/`。

**设计架构（一条流水线）**：

```
                    context_system/（提示词组装流水线）
                    ┌─────────────────────────────────────┐
  各模块的数据 ────►│  section 注册表（插件入口）            │
  工具/目录/git     │    register_section(id, builder, ...) │
  记忆/技能/子agent │         │                            │
                    │         ▼                            │
                    │  19 个内置 builder + 注册的 builder   │
                    │    （每个产出一段 section）            │
                    │         │                            │
                    │         ▼                            │
                    │  build_full_system_prompt_blocks()   │
                    │    排序 → 分组 → 打缓存标记 → block   │
                    └──────────────┬──────────────────────┘
                                   ▼
                    QueryParams.system_prompt → query() → 模型
```

**本章的设计理念（3 条核心）**：

| 理念 | 含义 | 代码体现 |
|------|------|---------|
| **1. 拼装不写死，section 插件化** | 系统提示词不是一段写死的文字，而是**每次运行时动态拼出来的**；想加内容？注册一个 section 就行，不改拼装主代码 | `register_section`（section_registry.py:164）+ `collect_new_sections`（拼装末尾收集） |
| **2. 缓存按"变化频率"分层** | 前缀缓存命中省 90% token——把"几乎不变"的放前面（GLOBAL）、"会话稳定"的中间（SESSION）、"每次变"的放后面（REQUEST） | `CacheScope` 三档 + `_emit_group` 组尾打 `cache_control` |
| **3. 关注点分离，各模块只注册不侵入** | 记忆模块注册 `memory` section、技能模块注册 `skills` section——**各模块只提供 builder，拼装主代码不认识它们** | `consult_section_builders` / `collect_new_sections` 双路径 |

**与主题 1 的架构呼应**：

| 主题 1（Agent） | 主题 2（Context） |
|----------------|-------------------|
| agent 注册表（`AgentRegistry`） | section 注册表（`_registry`） |
| `@register` 装饰器注册 agent | `register_section()` 注册 section |
| 定义时（数据）vs 运行时（执行） | 注册时（builder 声明）vs 拼装时（builder 调用） |
| 导入即注册 | 懒安装（`ensure_eager_extensions_installed`） |

> **一句话记住本章**：Context = 提示词组装流水线——**section 插件化拼装 + cache boundary 分层**。前面主题 1 学的"配置包"在这里变成"积木拼装"。

---

## 1. 它是什么

**Context = agent 每次思考时"看到"的全部内容**。模型一次只能看一段固定长度的文字，这段文字就是上下文。它包含：系统提示词（身份和规则）、项目背景、记忆、用户消息、工具结果、历史对话。

**核心问题：上下文是怎么拼出来的？** 答案在 `context_system/`——它像一个"提示词组装流水线"，把很多片段按顺序拼成最终的 system prompt。

**核心心法一句话：**

> **系统提示词 = 一条"提示词组装流水线"的产物。流水线上每个"工位"（section）负责一段内容，按固定顺序拼装，再按"变化频率"分组打上缓存标记。**

换句话说，模型看到的不是"一段写死的文字"，而是**每次运行时动态拼出来的**——工作目录变了、git 状态变了、记忆变了，拼出来的提示词也跟着变。

### 和主题 1 的关系

主题 1 学过：`get_agent_system_prompt()`（子 agent 专属）只有三级降级（fork 继承 / 自带 / 默认）。但**主 agent 的系统提示词是谁拼的？** 答案是本章的 `build_full_system_prompt_blocks()`——它才是"总装线"，把 intro / system / doing_tasks / tool_docs / env / memory 等几十个 section 拼成完整提示词。

### 为什么先学它

主题 3-8（Tool/Memory/Provider/Session/Skill/SubAgent）的内容，**最终都要通过 section 进到模型看到的文字里**。理解 Context 的 section 机制，后面每个模块学完都能对号入座："这个模块是注册了哪个 section 进去的"。

---

## 2. 代码在哪

主题 2 只需要看 4 个文件（按阅读顺序）：

| 文件 | 看什么 | 重点标注 |
|------|--------|----------|
| `clawcodex_ext/context_system/prompt_assembly.py` | **组装主入口**：`build_full_system_prompt_blocks()` + 19 个 `_build_*_section` | **必读**，先读这个 |
| `clawcodex_ext/context_system/section_registry.py` | `register_section()`：往提示词里"注册一段内容"的插件机制 | 必读 |
| `clawcodex_ext/context_system/system_prompt_cache.py` | `SystemPromptSection` / `CacheScope`（GLOBAL/SESSION/REQUEST） | 扫读 |
| `clawcodex_ext/context_system/builder.py` | 老版 `build_context_prompt()`：看它拼了哪几段 | 对照看 |

> 为什么先读 `prompt_assembly.py`？因为它定义了"系统提示词由哪些 section、按什么顺序拼"——这是本章的**数据契约**。

### 顺带认识：context_system 全家桶

| 文件 | 职责 |
|------|------|
| `prompt_assembly.py` | **主入口**：拼装所有 section（本章重点） |
| `section_registry.py` | section 注册表 + `register_section()` 插件机制 |
| `system_prompt_cache.py` | `SystemPromptSection` / `CacheScope`（缓存分层） |
| `builder.py` | 老版 API `build_context_prompt()`（日期/目录/git/CLAUDE.md） |
| `claude_md.py` / `clawcodex_md.py` | 项目说明文件（CLAUDE.md/CLAWCODEX.md）怎么读进上下文 |
| `git_context.py` | git 状态（当前分支/未提交改动）怎么进上下文 |
| `memory_prefetch.py` | 记忆预取（把相关记忆塞进上下文） |
| `microcompact.py` | 上下文微压缩（前面压缩流水线第 3 层） |
| `context_analyzer.py` | 上下文分析（token 统计等） |
| `workspace_snapshot.py` | 工作区快照 |
| `cache_boundary.py` | 缓存边界常量 |

---

## 3. 核心概念与代码语法

### 3.1 组装主入口：`build_full_system_prompt_blocks`

**伪代码先行**——想象拼装过程：

```python
# 伪代码：把一堆 section 拼成完整系统提示词
def 拼系统提示词(cwd, tools, agents, skills, mcp_servers, ...):
    sections = []
    sections += [intro, system, doing_tasks, actions, using_tools, tone, efficiency]  # 静态基础
    sections += [tool_docs, env, memory, mcp, agents, skills, output_style]          # 动态模块
    sections += [proactive, plan_mode, non_interactive, tool_restrictions, ...]      # 条件模块
    sections.sort(按 order)
    按缓存范围分组: global / session / request
    返回 block 列表（每组末尾带 cache_control）
```

**再看真实代码**（`prompt_assembly.py:684-921`）——核心拼装循环（节选 765-840）：

```python
def build_full_system_prompt_blocks(*, cwd, tools, tool_registry, agents, skills,
                                    mcp_servers, output_style, plan_mode, ...):
    sections: list[SystemPromptSection] = []

    # 逐个调用内置 builder，拼成 sections 列表
    intro = _build_intro_section(use_cache, ctx)          # 简介（你是谁）
    if intro:
        sections.append(intro)
    system = _build_system_section(use_cache, ctx)        # 系统规则
    if system:
        sections.append(system)
    tasks = _build_doing_tasks_section(use_cache, ctx)    # 做事准则
    ...
    tool_docs = _build_tool_docs_section(tools, tool_registry, use_cache, ctx)  # 工具文档
    env_section = _build_env_section(cwd, use_cache, ctx)                        # 环境（目录/git）
    memory_section = _build_memory_section(ctx)                                  # 记忆
    agent_section = _build_agent_section(agents, use_cache, ctx)                 # 可用子 agent
    skill_section = _build_skill_section(skills, use_cache, ctx)                 # 技能列表
    ...
    # 收集注册表里新增的 section（插件扩展）
    for new_sec in collect_new_sections(ctx):
        sections.append(new_sec)

    # 按 order 排序（保证两次调用字节级一致 → 缓存稳定）
    sections.sort(key=lambda s: s.order)

    # 按缓存范围分组（核心！见 3.3）
    global_sections = [s for s in sections if s.cache_scope is CacheScope.GLOBAL and s.content]
    session_sections = [s for s in sections if s.cache_scope is CacheScope.SESSION and s.content]
    request_sections = [s for s in sections if s.cache_scope is CacheScope.REQUEST and s.content]

    blocks = []
    if global_sections:
        _emit_group(global_sections, CacheScope.GLOBAL)   # 每组末尾带 cache_control
    blocks.append({"type": "text", "text": SYSTEM_PROMPT_DYNAMIC_BOUNDARY})  # 动态边界标记
    if session_sections:
        _emit_group(session_sections, CacheScope.SESSION)
    if request_sections:
        _emit_group(request_sections, CacheScope.REQUEST)
    if append_system_prompt:
        blocks.append({"type": "text", "text": append_system_prompt})  # 主 agent 身份追加
    return blocks
```

#### 语法拆解

| 语法 | 说明 |
|------|------|
| `def build_full_system_prompt_blocks(*, cwd, ...)` | `*` 号：所有参数**只能按关键字传**，强制可读性 |
| `sections.append(...)` | 每个 builder 返回一个 section，非空就加入列表 |
| `sections.sort(key=lambda s: s.order)` | **按 order 属性排序**——保证拼装顺序稳定 |
| `_emit_group(group, scope)` | 内部辅助函数：把一组 section 转成 block 列表，**组内最后一个 block 打 cache_control** |
| `-> list[dict[str, Any]]` | 返回 block 列表（`{"type": "text", "text": ..., "cache_control"?: ...}`） |

> **返回值**：一个 `list[dict]`，每个 dict 是一个文本块。这个列表直接作为 API 的 `system` 参数发给模型——**模型看到的就是这些块拼接成的文字**。

### 3.2 section 注册机制：插件式加内容

**伪代码先行**：

```python
# 伪代码：往提示词里加一段内容
注册一段内容(
    id="my_section",              # 唯一标识
    builder=生成我的内容的函数,     # 给它上下文，返回文字（返回 None = 不显示）
    order=50,                     # 排序位置（可选，默认查表）
    cache_scope=SESSION,          # 缓存范围（可选，默认查表）
)
```

**再看真实代码**（`section_registry.py:164-200`）：

```python
_registry: dict[str, RegisteredSection] = {}

def register_section(
    id: str,                                    # 唯一标识
    *,
    builder: Callable[[dict[str, Any]], str | None],  # 生成内容的函数
    order: int | None = None,                   # 排序位置
    cache_scope: SectionScope | None = None,     # 缓存范围
    tags: list[str] | None = None,               # 标签
) -> RegisteredSection:
    sec = RegisteredSection(
        id=id,
        builder=builder,
        order=order if order is not None else _CANONICAL_ORDER.get(id, 50),
        cache_scope=cache_scope if cache_scope is not None else _CANONICAL_SCOPE.get(id, SectionScope.SESSION),
        tags=set(tags or []),
    )
    _registry[id] = sec          # 存进注册表（同名覆盖，last-wins）
    return sec
```

**核心机制**（`section_registry.py:94-143` 的 `_CANONICAL_ORDER` / `_CANONICAL_SCOPE`）：

```python
_CANONICAL_ORDER = {
    # 静态模块（0-6）
    "intro": 0, "system": 1, "doing_tasks": 2, "actions": 3,
    "using_tools": 4, "tone_style": 5, "output_efficiency": 6,
    # 动态模块（10-95）
    "tool_docs": 10, "environment": 20, "memory": 25, "mcp": 30,
    "agents": 40, "skills": 50, "output_style": 60, "proactive": 65,
    "plan_mode": 70, "non_interactive": 80, "tool_restrictions": 90,
    "iteration_meta": 95,
}

_CANONICAL_SCOPE = {
    "intro": GLOBAL, "system": GLOBAL, ..., "tool_docs": SESSION,
    "environment": REQUEST, "memory": REQUEST, ...
}
```

#### 语法拆解

| 语法 | 说明 |
|------|------|
| `builder: Callable[[dict[str, Any]], str | None]` | builder 是函数类型：接收 `runtime_ctx` 字典，返回字符串或 None（None = 抑制该 section） |
| `_registry[id] = sec` | 字典赋值即注册——**同名覆盖**（last-wins） |
| `_CANONICAL_ORDER.get(id, 50)` | 没给 order 就从规范表查，查不到默认 50 |
| `_CANONICAL_SCOPE.get(id, SectionScope.SESSION)` | 没给 scope 就从规范表查，查不到默认 SESSION |

> **两种注册语义**（`section_registry.py:174-175` 注释）：
> - `id` 匹配内置 section（如 `"intro"`）→ **覆盖**内置内容
> - `id` 未知 → **注入**一个新 section
> 
> 这就是"插件机制"：**想往提示词里加内容？注册一个 section 就行，不用改拼装主代码**。

### 3.3 cache boundary：按"变化频率"分组

**为什么需要**：LLM API 有**提示词缓存**——同一段前缀重复发送时，命中的部分按低价计费（省 90% token 成本）。所以把"几乎不变的"放前面（能缓存），"每次变的"放后面（不能缓存，或缓存很短）。

**三种 CacheScope**（`system_prompt_cache.py`）：

| Scope | 含义 | 典型 section | 变化频率 |
|-------|------|-------------|---------|
| `GLOBAL` | 全局静态 | intro / system / doing_tasks / actions / using_tools / tone | 几乎不变 |
| `SESSION` | 会话级 | tool_docs / mcp / agents / skills / output_style | 会话内稳定 |
| `REQUEST` | 请求级 | environment / memory / proactive / plan_mode / non_interactive | 每次请求都变 |

**拼装结果的分组结构**（`prompt_assembly.py:710-714` 注释）：

```text
[global blocks…, ⟨cache_control 5m⟩,   ← 全局静态，可缓存
 __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__,   ← 动态边界标记（硬编码分割线）
 session blocks…, ⟨cache_control 5m⟩,  ← 会话级
 request blocks…, ⟨cache_control 5m⟩,  ← 请求级（每次变）
 (optional append_system_prompt block)] ← 主 agent 身份（不缓存）
```

**`_emit_group` 的核心**（`prompt_assembly.py:874-901`）：

```python
def _emit_group(group, scope):
    for idx, section in enumerate(group):
        block = {"type": "text", "text": section.content}
        block["_cache_scope"] = scope.value    # 内部标记（发送前剥离）
        block["_section_id"] = section.id      # 调试用（prompt_dump 可还原映射）
        if idx == len(group) - 1:              # ← 组内最后一个 block 打缓存标记
            block["cache_control"] = {"type": "ephemeral", "ttl": ttl}
        blocks.append(block)
```

> **关键**：每组**只有最后一个 block** 带 `cache_control`——因为前缀缓存是"从开头到某处"，把标记打在组尾，整组静态内容就都进缓存了。

### 3.4 内置 builder 一览（19 个）

`prompt_assembly.py` 里每个 `_build_*_section` 是一个"工位"：

| builder | section id | 内容 | scope |
|---------|-----------|------|-------|
| `_build_intro_section` | intro | "你是 Claude Code..."（身份简介） | GLOBAL |
| `_build_system_section` | system | 系统规则 | GLOBAL |
| `_build_doing_tasks_section` | doing_tasks | 做事准则（# Doing tasks） | GLOBAL |
| `_build_actions_section` | actions | 行动准则（# Executing actions） | GLOBAL |
| `_build_using_tools_section` | using_tools | 工具使用指南 | GLOBAL |
| `_build_tone_style_section` | tone_style | 语气/风格 | GLOBAL |
| `_build_output_efficiency_section` | output_efficiency | 输出效率 | GLOBAL |
| `_build_tool_docs_section` | tool_docs | 工具文档（name + description） | SESSION |
| `_build_env_section` | environment | 环境（cwd/git 状态） | REQUEST |
| `_build_memory_section` | memory | 记忆内容 | REQUEST |
| `_build_memory_store_section` | memory_store | 记忆存储说明 | REQUEST |
| `_build_mcp_section` | mcp | MCP 服务器 | SESSION |
| `_build_mcp_instructions_section` | mcp_instructions | MCP 使用说明 | SESSION |
| `_build_agent_section` | agents | 可用子 agent 列表 | SESSION |
| `_build_skill_section` | skills | 技能列表 | SESSION |
| `_build_output_style_section` | output_style | 输出样式 | SESSION |
| `_build_proactive_section` | proactive | 主动行为 | REQUEST |
| `_build_plan_mode_section` | plan_mode | 计划模式（条件） | REQUEST |
| `_build_non_interactive_section` | non_interactive | 非交互模式（条件） | REQUEST |

---

## 4. 架构关系：Context 在整个程序里的位置

> **附属文档（详细版）**：[Context 拼装调用链完整解析](call_chain_context.md)——按拼装链路分 5 章（build_effective_system_prompt → build_full_system_prompt_blocks → 各 builder → 注册机制），含反问题。

### 4.1 三句话定位

1. **Context 是"拼装线"，不是"存储"**：它不存数据，只负责把各模块的数据（git/记忆/工具/技能）**拼成模型看到的文字**。
2. **谁调用它**：`agent_loop_compat.py` 的 `build_effective_system_prompt()`（主题 1 学过）→ `build_full_system_prompt_blocks()` → 返回 block 列表 → 作为 `QueryParams.system_prompt` 传给 `query()`。
3. **各模块通过 section 接入**：记忆模块注册 `memory` section，技能模块注册 `skills` section——**不改拼装主代码，只注册**。

### 4.2 与主循环的配合

```
query() 主循环（引擎）
  │
  ├─ 拼上下文（本章）─────────────────────┐
  │   build_effective_system_prompt()      │
  │   → build_full_system_prompt_blocks()  │
  │   → [intro, system, ..., memory, ...]  │
  │   → block 列表（带 cache_control）      │
  │                                        │
  │   ↓                                    │
  │   QueryParams.system_prompt = blocks   │
  │                                        │
  ├─ 调模型（主题5 Provider）──────────────┤
  │   把 system_prompt + messages + tools  │
  │   发给 LLM                             │
  └─ 模型看到 = 拼好的完整提示词 ───────────┘
```

#### 4.2.1 拼好的提示词怎么流进引擎（5 层传递链）

上面图里"QueryParams.system_prompt = blocks"到底怎么发生的？答案是一条 **5 层传递链**——`build_full_system_prompt_blocks()` 拼出的 block 列表，每层原样透传，最终成为模型看到的主 agent 提示词：

```
build_full_system_prompt_blocks()     ← ① 拼装（prompt_assembly.py:684）
  → 返回 block 列表
        │
        ▼
build_effective_system_prompt()       ← ② 包装（agent_loop_compat.py:249）
  → 内部调 build_full_system_prompt_blocks
  → 返回 effective_system_prompt
        │
        ▼
headless.py:2088-2104                ← ③ 主 agent 身份追加
  effective_system_prompt = build_effective_system_prompt(...)
  if options.append_system_prompt:   ← 追加主 agent 身份（_resolve_startup_agent 注入的）
      effective_system_prompt.append({"type": "text", "text": options.append_system_prompt})
        │
        ▼
_run_one_agent_loop()                 ← ④ 装配（headless.py:2017）
  → run_query_as_agent_loop(          ← headless.py:2190
        system_prompt=effective_system_prompt,   ← headless.py:2195
        ...)
        │
        ▼
_make_params(messages)                ← ⑤ 进 QueryParams（agent_loop_compat.py:697）
  return QueryParams(
      system_prompt=system_prompt,    ← agent_loop_compat.py:700（闭包捕获）
      ...)
        │
        ▼
query(params) → _query_impl(params)   ← ⑥ 引擎
  current_system_prompt = params.system_prompt    ← query.py:2418
  ...
  await _call_model_sync(system_prompt=current_system_prompt, ...)  ← 发给模型
```

**每层的关键细节**：

| 层 | 干什么 | 返回值/关键点 |
|----|--------|--------------|
| ① `build_full_system_prompt_blocks` | 拼 19 个 section → block 列表 | `list[dict]`，带 `cache_control` |
| ② `build_effective_system_prompt` | 包装，可追加 output style | 调 `build_full_system_prompt_blocks`（agent_loop_compat.py:61） |
| ③ headless 追加身份 | **主 agent 身份在这里**：`options.append_system_prompt` | 是 `_resolve_startup_agent` 注入的（主题 1 学过） |
| ④ `_run_one_agent_loop` → `run_query_as_agent_loop` | 透传 `system_prompt=effective_system_prompt` | headless.py:2195 |
| ⑤ `_make_params` | 闭包捕获 `system_prompt` 参数 → `QueryParams.system_prompt` | agent_loop_compat.py:700（每轮复用，只更新 messages） |
| ⑥ `_query_impl` | `current_system_prompt = params.system_prompt` → 发给 `_call_model_sync` | query.py:2418，**模型看到的就是它** |

**关键机制：闭包传递（第 ⑤ 步的精髓）**

`_make_params` 是**嵌套闭包**——它不接收 `system_prompt` 参数，而是**直接捕获外层 `run_query_as_agent_loop` 的局部变量 `system_prompt`**：

```python
def run_query_as_agent_loop(*, system_prompt, ...):   # ← 外层参数
    ...
    def _make_params(messages):                        # ← 嵌套函数
        return QueryParams(
            system_prompt=system_prompt,               # ← 捕获外层！不用传参
            ...
        )
    while True:
        params = _make_params(next_messages)           # 每轮调用，复用同一份 system_prompt
```

**这保证了**：
1. 每轮调模型用的**都是同一份拼好的提示词**（不会每轮重拼）
2. 只有 `messages` 每轮更新，`system_prompt` 保持不变（直到会话级上下文变化触发重拼）

> **一句话**：`build_full_system_prompt_blocks()` 拼出的 block 列表 → `build_effective_system_prompt()` 包装 → **headless 追加主 agent 身份**（`append_system_prompt`）→ `run_query_as_agent_loop(system_prompt=...)` 透传 → `_make_params` **闭包捕获**进 `QueryParams.system_prompt` → `_query_impl` 里 `current_system_prompt = params.system_prompt` → 传给 `_call_model_sync` 发给模型。**拼装一次、闭包复用、每轮原样发送**。

### 4.3 主题 1 的呼应

> 主题 1 留的问题："**父 agent 的系统提示词是谁拼的？**" 答案就是本章：`build_effective_system_prompt()` → `build_full_system_prompt_blocks()`。而 `AgentDefinition.get_system_prompt()`（主题 1）只是**子 agent 专属**的候选，主 agent 走的是这条总装线。

---

## 5. 引出下一章：Tool（工具）

本章你已经看到 `_build_tool_docs_section` 把"工具文档"拼进提示词。但问题来了：

**模型的工具描述（name + description + input_schema）是从哪来的？工具是怎么被执行的？**

答案就是主题 3：**Tool（工具）**。

> **预告**：`clawcodex_ext/tool_system/` 是工具系统——`ToolRegistry`（注册表）、`Tool`（工具定义：name/input_schema/...）、`build_tool`（工厂函数）。`_build_tool_docs_section` 读的就是注册表里的工具，把它们序列化成模型能看的 schema。**工具的"声明"在 Tool 系统，工具的"展示"在 Context 系统**——两者通过 `tool_registry` 连接。

留一个问题给下一章：如果工具系统改了某个工具的 description，模型下次看到的是什么？（提示：Context 每次拼装时重新读注册表 → 动态反映变更。）

---

## 6. 练习与反问

### 动手练习

1. 打开 `prompt_assembly.py`，找出 `build_full_system_prompt_blocks` 里**按顺序**调用了哪些 `_build_*_section`（数一数：至少 19 个）。对照 3.4 的表格。

2. 打开 `section_registry.py`，回答：
   - `_CANONICAL_ORDER` 里 `memory` 的 order 是多少？（25）
   - 如果我注册一个 `id="my_plugin"` 的 section 但不给 order，它排在哪？（查表没有 → 默认 50）
   - 如果我注册 `id="intro"`，会发生什么？（覆盖内置 intro）

3. 运行 `cd clawcodex-ascend && uv run clawcodex-dev -p "介绍一下你自己"`，观察：
   - 它是否提到了"环境"（cwd/git）？
   - 它是否提到了可用工具或子 agent？

4. **对照练习**：打开 `prompt_assembly.py:760-840`，试着把"哪几个 builder 是 GLOBAL scope、哪几个是 REQUEST scope"对应到 3.3 的表格。

### 反问习题（自测）

**Q1.** `build_full_system_prompt_blocks` 为什么要按 `order` 排序 sections？

<details><summary>答案</summary>
保证两次调用（相同输入）产出**字节级一致**的 block 列表——这是缓存稳定性的前提。如果顺序每次都变，缓存永远命中不了。
</details>

**Q2.** 为什么每组 section 只有**最后一个 block** 带 `cache_control`？

<details><summary>答案</summary>
前缀缓存是"从开头到某处"的。把缓存标记打在组尾，整组静态内容（组内所有 block）都进入缓存前缀，下次请求命中省 90% token 成本。
</details>

**Q3.** `builder` 返回 `None` 表示什么？为什么要这样设计？

<details><summary>答案</summary>
返回 None = **抑制该 section**（不显示）。比如 `_build_plan_mode_section` 只在 plan_mode=True 时返回内容，否则返回 None——让条件性内容按需出现，不需要在拼装主代码里写 if。
</details>

**Q4.** 记忆系统怎么加进上下文的？（提示：`_build_memory_section`）

<details><summary>答案</summary>
记忆模块实现一个 builder，`register_section(id="memory", builder=...)` 注册进去。拼装时 `build_full_system_prompt_blocks` 调用 `_build_memory_section`（内置）或 `consult_section_builders("memory")`（注册的），把记忆内容拼进提示词。**不改拼装主代码，只注册**。
</details>

**Q5.** 主 agent 的身份提示词（`append_system_prompt`）为什么不缓存？

<details><summary>答案</summary>
它来自 `_resolve_startup_agent` 解析的 AgentDefinition（主题 1 学过），每次启动可能不同；而且它拼在**最后**（request 组之后），前缀缓存到不了它。所以作为独立的未缓存 block 追加。
</details>

---

## 7. 本章小结（一张表）

| 你要知道的事 | 一句话 |
|-------------|--------|
| Context 是什么 | 模型看到的全部内容 = 系统提示词 + 项目背景 + 记忆 + 历史 |
| 拼装主入口 | `build_full_system_prompt_blocks()`（prompt_assembly.py:684） |
| 核心机制 | section 插件：`register_section(id, builder, order, cache_scope)` |
| 19 个内置 builder | intro/system/doing_tasks/.../memory/skills/agents 等 |
| 顺序怎么定 | `_CANONICAL_ORDER` 表（静态 0-6，动态 10-95） |
| 缓存怎么分层 | CacheScope：GLOBAL（静态）/ SESSION（会话）/ REQUEST（每次变） |
| 组尾缓存标记 | 每组最后一个 block 带 `cache_control`（前缀缓存命中省 90%） |
| 主 agent 提示词 | `append_system_prompt` 追加在最后（不缓存） |
| 和谁配合 | `build_effective_system_prompt` → QueryParams.system_prompt → query() |
| 下一章预告 | `_build_tool_docs_section` 读的注册表 → 主题 3 Tool 系统 |

---

*下一主题：[主题 3：Tool（工具）](learning_plan.md#主题-3tool工具)—— 工具的声明、注册、分发。*
