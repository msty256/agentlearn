# 主题 1：Agent —— agent 就是"配置包"

> 学习顺序：**Agent → Context → Tool → Memory → LLM Provider → Session → Skill → SubAgent**
> 本章目标：看懂 `AgentDefinition` 数据结构、预置 agent 的定义、注册机制、系统提示词怎么生成，并理解"同一个引擎、换配置 = 换行为"。
> 阅读代码前先跑一遍：`cd clawcodex-ascend && uv run clawcodex-dev -p "你好"`（或任意一句问候），看看它怎么响应。

---

## 1. 它是什么

"Agent"在这个项目里有两层意思，先分清：

| 层 | 意思 | 类比 |
|----|------|------|
| 第 1 层：整个程序 | 你运行 `clawcodex-dev`，这个程序整体就是一个 agent —— 能听懂任务、调用工具、完成任务 | 一个"员工" |
| 第 2 层：agent 定义 | 程序里预置了多个不同"性格/能力"的 agent（主 agent、Explore 探索员、Plan 规划师、code-reviewer 代码审查员…）。每个 agent 是一个 `AgentDefinition` **数据对象** | 员工的"岗位说明书" |

本章学第 2 层。**核心心法一句话：**

> **agent 就是一个"配置包"——给同一个引擎（`query()` 主循环）换上不同的配置（提示词 + 工具 + 模型），就有不同行为的 agent。**

换句话说，引擎是固定的"机器"，`AgentDefinition` 是插进机器的"程序卡"。

### 为什么先学它

后面 7 个主题（Context / Tool / Memory / Provider / Session / Skill / SubAgent）讲的都是"这个引擎的零件"。先认识 `AgentDefinition` 这个"总装清单"，后面每个零件学完都能回来对号入座——它就是那张把一切串起来的索引表。

---

## 2. 代码在哪

主题 1 只需要看 4 个文件（按阅读顺序）：

| 文件 | 看什么 | 重点标注 |
|------|--------|----------|
| `clawcodex_ext/agent/agent_definitions.py` | `AgentDefinition` 数据结构 + 4 个内置 agent | **必读**，先读这个 |
| `clawcodex_ext/agent/registry.py` | `AgentRegistry` 注册表 + `register` 装饰器 | 必读 |
| `clawcodex_ext/agent/prompt.py` | `get_agent_system_prompt()` 生成系统提示词 | 扫读 |
| `clawcodex_ext/agent/run_agent.py` | 子 agent 执行器（第 8 章细讲） | **只看 `run_agent()` 的注释和参数表**，别深入 |

> 为什么先读 `agent_definitions.py`？因为它定义了"一个 agent 由哪些字段组成"——这是整个 agent 体系的**数据契约**，其他文件都是围绕它转的。

### 顺带认识：agent 模块全家桶

`clawcodex_ext/agent/` 目录下还有几十个文件，现在不用全看，但知道分工有好处：

| 文件/目录 | 职责 | 哪个主题学 |
|-----------|------|-----------|
| `agent_definitions.py` | AgentDefinition + 内置 agent | **本章** |
| `registry.py` | 注册表 | **本章** |
| `prompt.py` | 系统提示词生成 | **本章** |
| `policy.py` | 提示词"积木"（identity/规范/工具集） | 本章扩展 |
| `_bundled_agents/` | 装饰器注册的扩展 agent 示例 | 本章扩展 |
| `run_agent.py` | 子 agent 执行器 | 主题 8 |
| `subagent_context.py` | 子 agent 隔离上下文 | 主题 8 |
| `session.py` / `conversation.py` / `transcript.py` | 会话、消息历史、存档 | 主题 6 |
| `background_runner.py` / `fork_subagent.py` | 后台子 agent / fork 模式 | 主题 8 |

---

## 3. 核心概念与代码语法

### 3.1 `AgentDefinition`：一个 agent 长什么样

文件：`agent_definitions.py:55-94`

```python
@dataclass
class AgentDefinition:
    """Definition for an agent that can be spawned by the Agent tool."""

    agent_type: str                       # 名字，唯一标识（子 agent 用它被点名）
    when_to_use: str                      # 什么时候用这个 agent（给"爸爸"看的介绍）
    tools: list[str] | None = None        # 工具白名单；None 或 ['*'] = 全部工具
    source: AgentSource = "built-in"      # 来源：built-in / user / plugin / clawcodex_ext ...
    base_dir: str = "built-in"            # 加载目录标记
    model: str | None = None              # 建议模型；None→继承父，'inherit'→强制继承
    provider: str | None = None           # 模型服务商（anthropic/openai/...）
    permission_mode: PermissionMode | None = None  # 权限模式
    max_turns: int | None = None          # 最多跑多少轮（防死循环）
    background: bool = False              # 是否后台运行
    color: str | None = None              # UI 显示颜色
    memory: str | None = None             # 记忆配置
    omit_clawcodex_md: bool = False       # 是否跳过项目说明文件
    disallowed_tools: list[str] | None = None  # 工具黑名单
    hooks: dict[str, Any] | None = None   # 生命周期钩子
    skills: list[str] | None = None       # 可用技能列表
    isolation: Literal["worktree", "remote"] | None = None  # 隔离方式
    get_system_prompt: Callable[..., str] = field(default=lambda: "")  # 提示词生成函数
    ...
```

#### 语法拆解（初学者注意）

1. **`@dataclass`**：Python 标准库装饰器。自动根据字段生成 `__init__`、`__repr__`、`__eq__`，省得手写样板代码。字段声明就是构造参数。

2. **类型注解**（`str`、`list[str] | None`、`Callable[..., str]`）：
   - `tools: list[str] | None` = "类型是字符串列表，**或者** None"（`|` 是 3.10+ 的联合类型写法）。
   - `get_system_prompt: Callable[..., str]` = "一个可调用对象，任意参数，返回字符串"——**注意它是函数类型，不是字符串**，这点最容易看错。

3. **`field(default=lambda: "")`**：dataclass 里可变默认值不能直接写 `= ""` 的位置用 `field()` 包一层；`lambda: ""` 让每个实例都有独立的默认函数（避免共享同一个可变对象）。

4. **`Literal[...]`**：类型必须是列出的值之一（`"worktree"` 或 `"remote"`），编译器/IDE 能帮你查错。

5. **`__post_init__`**（`agent_definitions.py:91-94`）：dataclass 的"初始化钩子"，`__init__` 之后自动调用。这里的作用是兼容：`omit_claude_md` 和 `omit_clawcodex_md` 是同一个意思的两种拼法，只要任一个为真就两个都置真。

```python
def __post_init__(self) -> None:
    if self.omit_claude_md or self.omit_clawcodex_md:
        self.omit_claude_md = True
        self.omit_clawcodex_md = True
```

> **关键字段速记**：`agent_type`（身份证）、`when_to_use`（简介）、`tools`（白名单）、`disallowed_tools`（黑名单）、`model`（建议模型）、`get_system_prompt`（提示词函数）。其他字段用到再查。

### 3.2 预置 agent：四个内置"岗位"

文件：`agent_definitions.py:167-351`

项目自带 4 个内置 agent（`get_built_in_agents()` 返回，`agent_definitions.py:374-379`）：

| agent_type | 一句话（when_to_use 精简版） | tools | model |
|-----------|------------------------------|-------|-------|
| `general-purpose` | 通用 agent，研究/搜索/多步任务 | `["*"]` 全部 | 不指定（用默认子 agent 模型） |
| `Explore` | 快速探索代码库的只读专家 | 全工具但 `disallowed_tools` 禁了写类工具 | `haiku`（便宜小模型） |
| `Plan` | 软件架构师，只规划不写代码 | 同上（只读） | `inherit`（继承父） |
| `verification` | 代码验证，后台运行，红色 UI | 全工具但禁写 | `inherit` |

看一个具体的定义（`agent_definitions.py:225-246`）：

```python
EXPLORE_AGENT = AgentDefinition(
    agent_type="Explore",
    when_to_use=(
        "Fast agent specialized for exploring codebases. Use this when you need to "
        "quickly find files by patterns (eg. \"src/components/**/*.tsx\"), search code "
        'for keywords (eg. "API endpoints"), or answer questions about the codebase '
        '(eg. "how do API endpoints work?"). When calling this agent, specify the '
        'desired thoroughness level: "quick" for basic searches, "medium" for '
        'moderate exploration, or "very thorough" for comprehensive analysis across '
        "multiple locations and naming conventions."
    ),
    disallowed_tools=["Agent", "ExitPlanMode", "Edit", "Write", "NotebookEdit"],
    source="built-in",
    base_dir="built-in",
    omit_clawcodex_md=True,
    model="haiku",
    get_system_prompt=_explore_system_prompt,
)
```

#### 语法拆解

1. **`when_to_use` 为什么这么长？** 它不只是给人看的注释——它是**喂给"父 agent"的广告词**。父 agent（比如主 agent）遇到"要找代码"的任务时，模型会读这些描述决定派谁去。所以写的是"告诉调用者什么时候该选我"。

2. **`disallowed_tools` 比 `tools` 更常用**：Explore 是只读 agent，与其列"能用哪些"，不如列"绝不能碰哪些"（写文件、再委派子 agent 都禁止）。黑名单语义：默认全工具，减去这些。

3. **`model="haiku"` 是"建议值"不是"硬编码"**：代码注释明确说，`get_agent_model` 会先尝试在 session 的 provider 上找 haiku；**如果当前 provider 不支持（比如 DeepSeek），自动回退继承**。这就是"跨 provider 安全"的设计。

4. **系统提示词是一个函数**：`get_system_prompt=_explore_system_prompt`。注意这里传的是**函数对象**（不带括号），不是调用结果。因为提示词可能依赖运行时状态，所以延迟到真正要用时再调用。

看 `_explore_system_prompt` 的开头（`agent_definitions.py:183-220`）——它是纯文本，但注意它做了什么：**用全大写强调只读**：

```python
def _explore_system_prompt(**_kwargs: Any) -> str:
    return (
        "You are a file search specialist for Claw Codex. You excel at thoroughly "
        "navigating and exploring codebases.\n\n"
        "=== CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===\n"
        "This is a READ-ONLY exploration task. You are STRICTLY PROHIBITED from:\n"
        "- Creating new files (no Write, touch, or file creation of any kind)\n"
        ...
    )
```

> 提示：`**_kwargs` 表示"接收任意关键字参数但不在乎"——这样签名对谁都能调用，方便统一接口。

#### 对比：general-purpose 用共享片段拼提示词

`agent_definitions.py:158-164` 展示了**提示词复用**模式：

```python
def _general_purpose_system_prompt(**_kwargs: Any) -> str:
    return (
        f"{_SHARED_PREFIX} When you complete the task, respond with a concise "
        "report covering what was done and any key findings — the caller will relay this to "
        f"the user, so it only needs the essentials.\n\n{_SHARED_GUIDELINES}"
    )
```

`_SHARED_PREFIX` 和 `_SHARED_GUIDELINES`（`agent_definitions.py:104-127`）是模块级常量，被多个 agent 拼用——这就是"提示词工程"里的**模板复用**：公共部分抽成常量，差异部分拼接。

### 3.3 注册机制：agent 怎么"上线"

文件：`registry.py:102-248`

内置 agent 是模块级变量，但要能被系统找到，必须**注册**进 `AgentRegistry`。注册表是**进程级单例**（类变量）：

```python
class AgentRegistry:
    _definitions: list[AgentDefinition] = []   # 按注册顺序
    _by_type: dict[str, AgentDefinition] = {}  # 名字 → 定义，O(1) 查找

    @classmethod
    def find(cls, agent_type: str) -> AgentDefinition | None:
        """Look up an agent by its agent_type."""
        return cls._by_type.get(agent_type)
```

#### 语法拆解

1. **`@classmethod`**：方法第一个参数是 `cls`（类本身）而不是 `self`（实例）。调用方式 `AgentRegistry.find("Explore")`，不需要先创建实例。类变量 `_definitions` 是所有实例共享的——所以是"全局注册表"。

2. **两条数据通路**：
   - `_definitions`（列表）：保留注册顺序，用于"列出所有 agent"（`all()`）。
   - `_by_type`（字典）：名字直接查定义，用于"按名字找"（`find()`）。
   - 注册时（`_add()`，`registry.py:229-248`）两个结构**同步维护**，还要处理"同名覆盖"（后注册的顶掉先注册的，`last-wins`）。

3. **`find()` 返回 `AgentDefinition | None`**：可能找不到，返回 None。调用方必须处理 None 的情况（这是 Python 类型系统里最常见的"可能为空"表达）。

#### 两种注册方式

**方式一：装饰器**（扩展 agent 常用，`registry.py:156-225`）：

```python
@AgentRegistry.register(
    "code-reviewer",                       # agent_type
    when_to_use="Use after writing code to get an independent review.",
    tools=["Read", "Glob", "Grep", "Bash"],
    disallowed_tools=["Edit", "Write"],
    permission_mode="default",
)
def _code_reviewer_prompt() -> str:
    return build_agent_prompt(identity=IDENTITY_CODE_REVIEWER, ...)
```

真实例子在 `_bundled_agents/code_reviewer.py:40-55`。装饰器内部：
1. 接收 `agent_type`、`when_to_use` 等参数，**返回一个装饰器函数**（`registry.py:198`）。
2. 装饰器接收被装饰的函数 `prompt_fn`，把它包成 `AgentDefinition` 的 `get_system_prompt`（`registry.py:221`）。
3. 调用 `cls._add(agent)` 注册（`registry.py:223`）。

**方式二：直接注册已构建的 AgentDefinition**（`registry.py:145-154`）：

```python
AgentRegistry.register_definition(my_agent_def)
```

#### 装饰器语法拆解（初学者重点）

```python
def register(
    cls,
    agent_type: str,
    *,
    when_to_use: str,
    ...
) -> Callable[[Callable[..., str]], "AgentDefinition"]:
    def decorator(prompt_fn: Callable[..., str]) -> "AgentDefinition":
        agent = AgentDefinition(..., get_system_prompt=prompt_fn)
        return cls._add(agent)
    return decorator
```

- **`*` 号**：表示它后面的参数**只能按关键字传**（不能 `register("x", "y")` 位置传参），强制可读性。
- **返回类型 `Callable[[Callable[..., str]], "AgentDefinition"]`**："返回一个函数，这个函数接收一个函数、返回 AgentDefinition"。也就是"装饰器工厂 → 装饰器 → 定义"三层。
- **闭包**：`decorator` 内部能访问外层 `register` 的参数（`agent_type`、`when_to_use`…），Python 在调用 `@AgentRegistry.register(...)` 时先生成闭包环境，随后 `@` 把函数喂给 `decorator`。

> **注册的副作用时机**：`clawcodex_ext/agent/__init__.py:28-30` 的 docstring 说得很清楚——**导入 `clawcodex_ext.agent` 就会触发所有 `_bundled_agents` 的装饰器副作用**（eager registration）。这就是"导入即注册"模式。

### 3.4 黑名单合并：注册时就把工具算清楚

`registry.py:83-99` 有个细节函数 `_normalise_disallowed`：

```python
def _normalise_disallowed(disallowed: list[str] | None) -> list[str] | None:
    if disallowed is None:
        return list(ALL_AGENT_DISALLOWED_TOOLS)   # 没给黑名单 → 至少禁这些
    seen: set[str] = set(disallowed)
    merged: list[str] = list(disallowed)
    for tool in ALL_AGENT_DISALLOWED_TOOLS:       # 全局禁用的工具永远叠加
        if tool not in seen:
            merged.append(tool)
            seen.add(tool)
    return merged
```

`ALL_AGENT_DISALLOWED_TOOLS`（`constants.py:52-64`）是所有 agent 都禁用的"危险工具"：`TaskOutput`、`ExitPlanMode`、`Agent`（防递归）、`AskUserQuestion` 等。

> **设计意图**：即使你自定义 agent 时只写了 2 个禁用工具，注册时也会自动把全局禁用列表并进去——**安全底线不由调用者决定**。用集合（`set`）去重，避免重复。

### 3.5 系统提示词怎么生成

文件：`prompt.py:236-263`

```python
def get_agent_system_prompt(
    agent_definition: AgentDefinition,
    parent_system_prompt: "str | list | None" = None,
) -> "str | list":
    from clawcodex_ext.agent.constants import DEFAULT_AGENT_PROMPT, FORK_SUBAGENT_TYPE

    # 规则 1：fork 模式直接继承父的系统提示词
    if agent_definition.agent_type == FORK_SUBAGENT_TYPE and parent_system_prompt:
        return parent_system_prompt

    # 规则 2：用自己的提示词生成函数
    prompt = agent_definition.get_system_prompt()
    if prompt:
        reminder = agent_definition.critical_system_reminder
        if reminder and reminder not in prompt:
            return f"{prompt}\n\n{reminder}"   # 追加"关键提醒"
        return prompt

    # 规则 3：兜底用默认提示词
    return DEFAULT_AGENT_PROMPT
```

三个规则逐层降级，理解优先级：

| 优先级 | 条件 | 用谁的提示词 |
|--------|------|-------------|
| 1 | fork 类型且有父提示词 | 直接继承父的（fork = "借用父的脑子"） |
| 2 | agent 有自己的 `get_system_prompt()` 且返回非空 | 自己的，必要时追加强制提醒（`critical_system_reminder`） |
| 3 | 都没有 | `DEFAULT_AGENT_PROMPT`（`constants.py:110-116`）兜底 |

> **提醒**：`get_system_prompt()` 返回空字符串时走兜底——所以 `AgentDefinition` 的默认 `field(default=lambda: "")` 不是摆设。

---

## 4. 架构关系：Agent 在整个程序里的位置

### 4.1 三句话定位

1. **Agent 是"配置"，不是"代码"**：`query()` 主循环（`clawcodex_ext/query/query.py`）才是引擎本体，agent 只是喂给它的参数。
2. **Agent 由"爸爸"来用**：真正触发子 agent 的是 **Agent 工具**（`tool_system/tools/agent.py`），模型在对话里说"我要派个 Explore"，工具系统收到后调用 `run_agent()`。
3. **Agent 决定"隔离边界"**：每个子 agent 通过 `create_subagent_context()`（`subagent_context.py`）拿到**独立副本**的上下文——独立的工具状态、权限、abort 控制器，防止干扰父 agent。

### 4.2 与主循环的配合（呼应运行时流程）

`runtime_flow.md` 里讲过的完整链路在这里收口：**模型说"调 Agent 工具" → Agent 工具 → run_agent() → query() 递归**。

```text
用户输入
   │
   ▼
query() 主循环（引擎）
   │  ├─ 拼上下文（主题2 Context）────── 读 agent 的 get_system_prompt()
   │  ├─ 调模型（主题5 Provider）
   │  └─ 模型返回 tool_use ──► Tool 系统（主题3）
   │                                └─ 是 Agent 工具？
   │                                     ▼
   │                             run_agent()（run_agent.py）
   │                              ├─ 按 agent_type 查 AgentRegistry.find()
   │                              ├─ 解析权限/工具/模型
   │                              ├─ create_subagent_context()  ← 隔离
   │                              └─ 递归调 query() ←── 子 agent 的完整引擎
   ▼
Session.save()（主题6 存档）
```

### 4.3 配合的关键代码点（扫一眼即可）

`run_agent.py:242-253` 的 docstring 已经把 5 步说清了：

```python
async def run_agent(params: RunAgentParams) -> AsyncGenerator[Message, None]:
    """Run an agent's query loop and yield messages.

    1. Resolves model, tools, system prompt, and permission mode
    2. Creates an isolated subagent context
    3. Runs the query loop via the existing query() function
    4. Yields messages as they arrive
    5. Cleans up on completion or abort
    """
```

其中"按 agent_type 查定义"这步就是 `AgentRegistry.find()` 干的。**所以注册表是"配置"和"运行时"之间的桥梁**：定义阶段用 `@register` 登记，运行阶段用 `find()` 取回。

---

## 5. 引出下一章：Context（上下文）

主题 1 里你已经见过两个"提示词"概念，但它们**还没被拼起来**：

- `AgentDefinition.get_system_prompt()` —— 一个 agent 的"自我介绍"（身份、规则、只读约束…）
- `critical_system_reminder` —— 追加的"关键提醒"

问题来了：**这些片段怎么变成模型真正看到的那一长段文字？** 模型并不知道"有一段系统提示词叫 Explore 的自我介绍"，它只看到**一整段**拼好的文本。

答案就是主题 2：**Context（上下文）**。

> **预告**：`clawcodex_ext/context_system/` 是一个"提示词组装流水线"。系统提示词只是其中一个 `section`（片段），还有项目背景、git 状态、记忆、工具描述……都要拼进去。核心机制是 `section_registry.py` 的 `register_section()`——**想往提示词里加内容？注册一个 section 就行，不用改组装主代码**。

留一个问题给下一章：如果 `get_agent_system_prompt()` 是"子 agent 专属提示词"，那**父 agent（主 agent）的系统提示词是谁拼的？** 这就是 Context 系统的活。

---

## 6. 练习与反问

### 动手练习

1. 打开 `agent_definitions.py`，找出 `EXPLORE_AGENT` 和 `PLAN_AGENT`，对比它们的：
   - `when_to_use` 有什么不同（什么时候该派谁）？
   - `disallowed_tools` 是否一样？为什么 Plan 也要禁写工具？
   - `model` 分别是 `haiku` 和 `inherit`，含义有什么差别？

2. 打开 `registry.py`，在代码里回答：
   - `find("Explore")` 会走哪条数据通路？（`_definitions` 还是 `_by_type`？）
   - 如果两个模块都注册了同名 agent，谁会赢？（提示：看 `_add` 的 `last-wins` 逻辑）

3. 打开 `_bundled_agents/code_reviewer.py`，对比它和 `EXPLORE_AGENT` 的注册方式——**为什么一个是装饰器、一个是直接构建 AgentDefinition？**（提示：谁拥有 `get_system_prompt` 的定义权）

4. 运行 `cd clawcodex-ascend && uv run clawcodex-dev -p "帮我查一下当前目录有哪些 python 文件"`，观察：
   - 主 agent 有没有调用 Agent 工具？
   - 如果调了，`subagent_type` 传的是什么？为什么选它？

### 反问习题（自测）

**Q1.** `AgentDefinition.model` 字段存的是"模型名"，但为什么代码里叫它"建议值"而不是"硬编码"？

<details><summary>答案</summary>
因为运行时要过 `get_agent_model()` 解析：当前 provider 支持才用，不支持就回退继承父的模型（如 DeepSeek 不支持 haiku）。`model` 只是"想要什么"，不是"必须是什么"。
</details>

**Q2.** `get_system_prompt` 字段是**函数**而不是字符串，为什么？

<details><summary>答案</summary>
提示词可能依赖运行时状态（父提示词、技能列表、LKB 开关），延迟到调用时才生成。且不同 agent 的提示词生成逻辑不同（有拼接、有继承、有兜底），用函数才能表达这种"多态"。
</details>

**Q3.** 为什么 `_by_type` 和 `_definitions` 要**两份**数据结构存同样的 agent？

<details><summary>答案</summary>

`_by_type`（字典）用于按名字 O(1) 查找，`_definitions`（列表）用于保留注册顺序、整体列出。一个服务"点菜"，一个服务"看菜单"，各司其职。同步维护靠 `_add()` 一个入口。
</details>

**Q4.** Explore agent 的 `when_to_use` 那么长，读者（父 agent 的模型）真的会读吗？长描述会不会浪费 token？

<details><summary>答案</summary>

会读——`when_to_use` 会被 `format_agent_line()` 拼进 Agent 工具的 description（`prompt.py:73-79`），模型决策派谁时就靠它。至于 token：`MAX_INLINE_TOOL_DISPLAY=20`（`constants.py:125`）限制工具名内联展示，但 `when_to_use` 是决策依据不截断。这是"决策质量"和"token 成本"的取舍。
</details>

**Q5.** 现在你能说出：**"同一个引擎、换配置 = 换行为"** 在代码里具体指什么和什么的关系吗？

<details><summary>答案</summary>

引擎 = `query()` 主循环（`query/query.py`），配置 = `AgentDefinition`。同一个 `query()` 收到不同的 `system_prompt`/`tools`/`model`，就跑出不同行为（Explore 只读、Plan 只规划、code-reviewer 只审查）。主题 8 你会看到 `run_agent()` 就是"把 AgentDefinition 翻译成 QueryParams，再调同一个 query()"。
</details>

---

## 7. 本章小结（一张表）

| 你要知道的事 | 一句话 |
|-------------|--------|
| Agent 是什么 | 一个"配置包"（AgentDefinition），不是执行代码 |
| 最核心的字段 | `agent_type` / `when_to_use` / `tools` / `disallowed_tools` / `model` / `get_system_prompt` |
| 怎么注册 | `@AgentRegistry.register(...)` 装饰器 或 `register_definition()`，导入即注册 |
| 怎么查找 | `AgentRegistry.find("Explore")`，O(1) |
| 系统提示词谁生成 | `get_agent_system_prompt()`：fork 继承 > 自己的函数 > 默认兜底 |
| 黑名单怎么算 | 自己的 + 全局强制禁用的（`ALL_AGENT_DISALLOWED_TOOLS`）自动合并 |
| 和谁配合 | Agent 工具触发 → `run_agent()` → `create_subagent_context()` 隔离 → 递归 `query()` |
| 下一章预告 | `AgentDefinition` 的提示词只是 Context 流水线的**一个 section** |

---

*下一主题：[主题 2：Context（上下文）](learning_plan.md#主题-2context上下文)—— 系统提示词怎么和项目背景、git 状态、记忆拼成一整段。*

