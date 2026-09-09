# 附属文档：主 agent 调用链完整解析

> 所属：主题 1（Agent）学习文档的第 1 章附属资料
> 目标读者：了解一些 Python、想彻底搞懂"一条命令从敲下去到主循环"的初学者
> 学习方法：**先整体后局部**——每章开头先看"这一层是干嘛的"（整体），再钻进函数看"怎么写的"（局部）；每个函数都配"伪代码 → 真实代码 → Python 语法讲解 → 具体参数示例"。
> 前置知识：只需要知道 Python 的基本语法（函数、类、import）。

---

## 第 0 章：序章 —— 从"敲一个字符"到 `main()` 的完整旅程

在讲 6 层调用链之前，先回答一个最基础的问题：**我明明只敲了一个命令，程序是怎么"活"起来的？是谁把它变成了 `main()` 函数在跑？**

这一章讲清楚"终端 → 命令名 → 可执行文件 → Python 解释器 → `main()`"的完整链条。

### 0.1 先纠正一个直觉：不是"逐字符触发"

很多初学者以为"敲一个字符程序就开始动了"——**不是的**。终端的工作方式：

```text
你敲:  c l a w c o d e x - d e v ␣ - p ␣ " 你 好 "
        │ 每个字符都只是"显示在屏幕上"（shell 的 readline 缓存）
        │ 程序还没启动！
        ▼
你按:  回车 (Enter)
        │ shell 收到整行文本 "clawcodex-dev -p \"你好\""
        ▼
shell 开始干活: 拆词 → 找命令 → 启动进程
```

**关键结论**：**回车之前，程序一行都没跑**。终端里你看到的字符只是 shell 的"输入缓存"，回车才是"触发信号"。

### 0.2 shell 拿到命令后干了什么（整体）

你按下回车后，shell（bash/zsh）按这 4 步处理：

```
回车
  │
  ▼
① 拆词: "clawcodex-dev" + ["-p", "你好"]
  │
  ▼
② 找命令: 在 PATH 目录列表里搜 "clawcodex-dev" 可执行文件
  │    PATH=/usr/local/bin:/usr/bin:~/.local/bin:/c/Users/xxx/.local/bin:...
  │    └─ 找到 ~/.local/bin/clawcodex-dev（wrapper 脚本）
  │
  ▼
③ 启动进程: shell fork 出一个子进程，执行那个文件
  │
  ▼
④ 传参: 把 ["-p", "你好"] 作为参数传给进程
  │    → Python 收到 sys.argv = ["clawcodex-dev", "-p", "你好"]
```

**核心**：`clawcodex-dev` 不是一个程序，而是一个**入口点（entry point）**——它指向 Python 里的 `main()` 函数。

### 0.3 入口点声明：`pyproject.toml` 的魔法

项目用 `pyproject.toml` 声明"命令名 → 函数"的映射（`pyproject.toml:160-161`）：

```toml
[project.scripts]
clawcodex-dev = "clawcodex_ext.cli.main:main"
```

**这一行的含义**（这是整个链条的"源头"）：

| 部分 | 含义 |
|------|------|
| `[project.scripts]` | 声明"我要生成命令行工具"的节 |
| `clawcodex-dev` | **命令名**——用户敲的命令 |
| `clawcodex_ext.cli.main` | **模块路径**——`clawcodex_ext/cli/main.py` |
| `:main` | **函数名**——模块里的 `main()` 函数 |

读法：**"当用户敲 `clawcodex-dev` 时，调用 `clawcodex_ext.cli.main` 模块里的 `main()` 函数"**。

**Python 语法讲解**——`"包.模块:函数"` 的路径格式：

```text
"clawcodex_ext.cli.main:main"
   │          │    │    │
   │          │    │    └─ 函数名（main.py 里的 def main）
   │          │    └─ 模块名（main.py 文件）
   │          └─ 子包（cli/ 目录，有 __init__.py）
   └─ 顶层包（clawcodex_ext/ 目录）
```

等价于 Python 代码：

```python
from clawcodex_ext.cli.main import main   # ← 入口点展开后就是这句
```

### 0.4 谁把声明变成可执行文件？（安装时）

光声明没用，还要**安装**才生成可执行文件。安装脚本 `install.sh` 做三件事：

**① 安装依赖 + 注册入口点**（`install.sh:633`）：

```bash
uv pip install -e ".[all]"
```

`-e` 是 editable（可编辑）安装。pip 读取 `pyproject.toml` 的 `[project.scripts]`，**自动生成一个可执行文件**（叫 "console script"），放进虚拟环境的 bin 目录：

```text
Linux/macOS:  <venv>/bin/clawcodex-dev          ← 可执行 Python 脚本
Windows:      <venv>/Scripts/clawcodex-dev.exe  ← 可执行文件
```

这个生成的文件内容（简化）就是一个"启动器"：

```python
#!/usr/bin/env python3
# 自动生成的 console script（pip 根据 [project.scripts] 生成）
import sys
from clawcodex_ext.cli.main import main    # ← 导入入口函数
sys.exit(main())                            # ← 调用它，返回值作为退出码
```

**② 生成 wrapper 脚本**（`install.sh:720-744`）——`~/.local/bin/clawcodex-dev`：

```bash
cat > ~/.local/bin/clawcodex-dev <<EOF
#!/usr/bin/env bash
export CLAWCODEX_CONFIG_DIR="\${CLAWCODEX_CONFIG_DIR:-$CONFIG_DIR}"
exec "<venv>/bin/clawcodex-dev" "\$@"      # ← 把参数转发给真正的入口
EOF
chmod +x ~/.local/bin/clawcodex-dev
```

**为什么还要套一层 wrapper？** 注释说了：wrapper 脚本比符号链接更可移植（Windows/Git Bash 友好），且 venv 重建后依然有效。它还顺便注入 `CLAWCODEX_CONFIG_DIR` 环境变量。

**③ 把 `~/.local/bin` 加进 PATH**（`install.sh:750-758`）：

```bash
# 写进 ~/.bashrc / ~/.zshrc
export PATH="$HOME/.local:$HOME/.local/bin:$PATH"
```

这样下次开终端，敲 `clawcodex-dev` 就能被找到。

### 0.5 完整链条（一次看全）

```text
安装时（只做一次）:
  pyproject.toml [project.scripts]          声明 clawcodex-dev → main:main
       │  uv pip install -e ".[all]"
       ▼
  <venv>/bin/clawcodex-dev                   pip 生成的 console script
       │  install.sh register_commands()
       ▼
  ~/.local/bin/clawcodex-dev                 wrapper 脚本（转发参数）
       │  install.sh update_shell_rc()
       ▼
  PATH 加入 ~/.local/bin                     以后敲命令能找到

运行时（每次执行）:
  你敲 "clawcodex-dev -p \"你好\"" + 回车
       │
       ▼
  shell 按 PATH 找到 ~/.local/bin/clawcodex-dev
       │  exec 执行
       ▼
  wrapper 脚本: export CLAWCODEX_CONFIG_DIR; exec <venv>/bin/clawcodex-dev "$@"
       │
       ▼
  console script: from clawcodex_ext.cli.main import main; sys.exit(main())
       │
       ▼
  main()  ←←← 这里就是第 1 章的开始！
       │
       ▼
  run_cli() → ... → query()（后面 6 章）
```

### 0.6 具体参数示例

```bash
# 你在终端敲（字符逐个显示，回车才执行）:
$ clawcodex-dev -p "你好" --model claude-sonnet-4
```

```text
# shell 内部实际发生:
# 拆词:    ["clawcodex-dev", "-p", "你好", "--model", "claude-sonnet-4"]
# 找命令:  PATH 里找到 ~/.local/bin/clawcodex-dev
# exec:    执行 wrapper → 转发给 venv/bin/clawcodex-dev
# sys.argv 到达 main():
sys.argv = ["clawcodex-dev", "-p", "你好", "--model", "claude-sonnet-4"]
```

```python
# 等价于你在 Python 里手动调用:
from clawcodex_ext.cli.main import main
main()   # 内部读 sys.argv，进入 run_cli()
```

### 0.7 反问题（自测）

**Q1.** 为什么敲字符时程序没启动，按回车才启动？

<details><summary>答案</summary>
终端输入是"缓冲"的——字符先存在 shell 的输入缓存里（readline 功能，支持退格/历史），只有回车才把整行交给 shell 处理。shell 拿到完整命令后才会 fork 子进程执行。
</details>

**Q2.** `clawcodex-dev` 是程序吗？它到底是什么？

<details><summary>答案</summary>
不是"一个程序"，而是一个**入口点（entry point）**。`pyproject.toml` 的 `[project.scripts]` 把它映射到 `clawcodex_ext.cli.main:main`——它是指向 Python 函数的"命令别名"。
</details>

**Q3.** wrapper 脚本和 console script 有什么区别？

<details><summary>答案</summary>
console script 是 pip 根据 `[project.scripts]` 自动生成的（导入 main 并调用，跨平台）；wrapper 是 install.sh 手写的 bash 脚本（注入环境变量 + 转发参数），套在 console script 外面，解决可移植性和 venv 重建问题。
</details>

**Q4.** `sys.argv[0]` 是什么？为什么第 2 章要用 `argv[1:]`？

<details><summary>答案</summary>
`sys.argv[0]` 是**程序名**（"clawcodex-dev"），`sys.argv[1:]` 才是真正的参数（["-p", "你好", ...]）。所以前端分发时要 `argv[1:]` 去掉程序名。
</details>

---

## 全景总览：主 agent 调用链一图流

你敲下 `clawcodex-dev -p "你好"`，程序按**文件调用顺序**依次经过这 6 层：

```
┌─────────────────────────────────────────────────────────────┐
│ ① cli/main.py          main()          "入口，很薄"           │
│       └─ 调 run_cli()                                       │
├─────────────────────────────────────────────────────────────┤
│ ② cli/dispatch.py      run_cli()       "解析参数+选前端"      │
│       └─ 解析 args → RuntimeOptions → RuntimeContext         │
│       └─ get_frontend("headless").run(ctx, argv)            │
├─────────────────────────────────────────────────────────────┤
│ ③ frontend/headless.py HeadlessFrontend.run() "前端插件"     │
│       └─ 把 ctx 提取成 HeadlessOptions → run_headless()      │
├─────────────────────────────────────────────────────────────┤
│ ④ entrypoints/headless.py run_headless()  "装配"            │
│       └─ 建 Session / tool_registry / tool_context          │
│       └─ 拼 system_prompt（build_effective_system_prompt）   │
│       └─ _run_one_agent_loop()                              │
├─────────────────────────────────────────────────────────────┤
│ ⑤ query/agent_loop_compat.py run_query_as_agent_loop() "桥" │
│       └─ 把 AgentLoop 参数翻译成 QueryParams                 │
│       └─ while True: query(params)                          │
├─────────────────────────────────────────────────────────────┤
│ ⑥ query/query.py       query() → _query_impl()  "主循环本体" │
│       └─ 推理 → 执行工具 → 观察 → 回填 → 直到 Terminal       │
└─────────────────────────────────────────────────────────────┘
```

**记忆口诀**：入口（main）→ 调度（dispatch）→ 前端（frontend）→ 装配（headless）→ 桥接（compat）→ 引擎（query）。

下面每一章讲一层。**每章都按同一个套路**：
1. **这一层是干嘛的**（整体，一句话 + 伪代码）
2. **代码结构**（局部，真实代码 + 行号）
3. **Python 语法讲解**（初学者专用）
4. **具体参数示例**（真实值长什么样）

---

## 第 1 章：`cli/main.py` —— 入口（最薄的一层）

### 1.1 这一层是干嘛的（整体）

`main.py` 是整个程序的**入口文件**，但它**几乎什么都不做**——它只做两件事：
1. 注册一个"会话记录初始化"钩子（`ensure_nested_transcript_initialized()`）
2. 把控制权交给 `run_cli()`

**为什么这么薄？** 因为真正的参数解析和分发在 `dispatch.py`。入口层保持薄，是为了让测试好替换、逻辑好分层。

**伪代码**：

```python
# main.py 的伪代码
def main():
    初始化会话记录()          # 确保 transcript 路径解析器先注册
    return 调用 run_cli()     # 把控制权交出去，返回退出码
```

### 1.2 真实代码

`cli/main.py:30-48`：

```python
def main():
    """Delegate to the downstream CLI dispatch."""
    from clawcodex_ext import ensure_nested_transcript_initialized

    ensure_nested_transcript_initialized()
    from clawcodex_ext.cli.dispatch import run_cli

    return run_cli()


if __name__ == "__main__":
    sys.exit(main())
```

### 1.3 Python 语法讲解

| 语法 | 说明 |
|------|------|
| `def main():` | 定义函数。注意这里**没有参数**——因为命令行参数通过 `sys.argv` 全局读取 |
| `from ... import ...` 写在函数**里面** | **延迟导入**（lazy import）。只在函数被调用时才 import，加快启动速度（不需要的东西不加载） |
| `return run_cli()` | 函数 A 调用函数 B 并**直接返回** B 的结果——"委托模式"，A 不加工，原样传递 |
| `if __name__ == "__main__":` | 程序入口惯用法。只有直接运行这个文件时 `__name__` 才是 `"__main__"`；被 import 时不是，不会执行 |
| `sys.exit(main())` | 把 main() 的返回值作为**进程退出码**（0 = 成功，非 0 = 失败） |

### 1.4 具体参数示例

你运行 `clawcodex-dev -p "你好"`，实际发生的值：

```python
# 伪代码演示（真实值）
sys.argv = ["clawcodex-dev", "-p", "你好"]   # 命令行参数（不传给 main，全局可读）
main()                                        # → 调用 run_cli()
# run_cli() 内部读 sys.argv，解析出:
#   args.prompt = "你好"
#   args.print  = True    （因为传了 -p）
```

---

## 第 2 章：`cli/dispatch.py` —— 调度（参数解析 + 选前端）

### 2.1 这一层是干嘛的（整体）

`run_cli()` 是**整个 CLI 的大脑**。它做四件事：
1. **解析参数**：把 `sys.argv` 变成结构化的 `args` 对象
2. **快路径分支**：`login`/`config`/`mcp`/`daemon`/`doctor` 等子命令直接处理返回
3. **构建运行时**：把解析结果打包成 `RuntimeOptions` → `RuntimeContext`
4. **选前端**：headless（单次）/ tui / repl 三选一，调用它的 `run()`

**伪代码**：

```python
def run_cli(argv):
    解析参数(argv)                # → args
    如果是子命令(login/config/mcp...): return 处理它()
    构建 RuntimeOptions(args)     # 参数打包
    构建 RuntimeContext(options)  # 运行时上下文（provider/工具/会话）
    if args.print:               # -p 单次模式
        return get_frontend("headless").run(ctx, argv)
    if 应该用TUI():              # 交互模式（有终端）
        return get_frontend("tui").run(ctx, argv)
    return get_frontend("repl").run(ctx, argv)   # 默认交互
```

### 2.2 真实代码（关键段）

**参数 → RuntimeOptions**（`dispatch.py:665-696`）：

```python
runtime_opts = RuntimeOptions(
    provider_name=getattr(args, "provider", None),      # 如 "anthropic"
    model=getattr(args, "model", None),                 # 如 "claude-sonnet-4"
    fallback_model=getattr(args, "fallback_model", None),
    prompt=getattr(args, "prompt", None),               # 如 "你好"
    max_turns=getattr(args, "max_turns", 20),           # 默认 20 轮
    allowed_tools=tuple(split_csv(getattr(args, "allowed_tools", None))),
    disallowed_tools=tuple(split_csv(getattr(args, "disallowed_tools", None))),
    permission_mode=getattr(args, "_resolved_permission_mode", "default"),
    skip_permissions=getattr(args, "dangerously_skip_permissions", False),
    ...
)
```

**前端分发**（`dispatch.py:762-824`）：

```python
if args.print:
    frontend = get_frontend("headless")
    rc = _run_with_worktree_keep_note(lambda: frontend.run(ctx, argv[1:]), worktree_session)
    return rc

if should_use_tui(explicit_tui):
    frontend = get_frontend("tui")
    rc = _run_with_worktree_keep_note(lambda: frontend.run(ctx, argv[1:]), worktree_session)
    return rc

frontend = get_frontend("repl")     # 兜底
rc = _run_with_worktree_keep_note(lambda: frontend.run(ctx, argv[1:]), worktree_session)
return rc
```

### 2.3 Python 语法讲解

| 语法 | 说明 |
|------|------|
| `getattr(args, "provider", None)` | **安全取值**：如果 `args` 对象有 `provider` 属性就返回它，否则返回默认值 `None`。因为 argparse 生成的 args 不一定有所有属性，用 getattr 防 KeyError |
| `tuple(split_csv(...))` | 函数套函数：`split_csv` 把 `"Read,Glob,Bash"` 拆成 `["Read","Glob","Bash"]`，`tuple()` 再转成**不可变元组**（防止后面被误改） |
| `lambda: frontend.run(ctx, argv[1:])` | **匿名函数**。把"调用 run 这件事"包成一个函数，延迟到 `_run_with_worktree_keep_note` 内部执行（它可能要先处理 worktree 逻辑再调用） |
| `frontend.run(ctx, argv[1:])` | `argv[1:]` 是**切片**：去掉第一个元素（程序名），只留后面的参数 |
| 三选一 if/elif/默认 | 注意 headless 用 `if args.print`，TUI 用 `should_use_tui()`，最后 `repl` 兜底——**顺序决定优先级** |

### 2.4 具体参数示例

```python
# 你输入: clawcodex-dev -p "你好" --model claude-sonnet-4 --max-turns 5
# 实际构造的 RuntimeOptions（关键字段值）:
runtime_opts = RuntimeOptions(
    provider_name=None,           # 没指定，后面用默认 provider
    model="claude-sonnet-4",      # 命令行 --model
    prompt="你好",                 # -p 的值
    max_turns=5,                  # --max-turns 5
    max_turns_explicit=True,      # 因为命令行出现了 --max-turns
    permission_mode="default",
    skip_permissions=False,
)
# frontend 选择: args.print=True → get_frontend("headless")
```

---

## 第 3 章：`frontend/headless.py` —— 前端插件（转译参数）

### 3.1 这一层是干嘛的（整体）

**Frontend 是一个"插件机制"**：headless / tui / repl 各自是一个插件类，都实现同一个 `run(ctx, argv)` 接口。这样 `dispatch.py` 不用关心具体前端是谁——**"我只要调 `frontend.run(ctx, argv)`，你负责用你的方式处理"**。

`HeadlessFrontend.run()` 做的事：把 `ctx`（RuntimeContext）里的一大堆信息，**提取成 `HeadlessOptions`**，然后调用 `run_headless()`。

**伪代码**：

```python
class HeadlessFrontend:                    # 前端插件
    name = "headless"

    def run(self, ctx, argv):
        options = HeadlessOptions(          # 从 ctx 提取参数
            prompt=ctx.options.prompt,
            provider_name=ctx.provider_name,
            model=ctx.options.model,
            max_turns=ctx.options.max_turns,
            ...
        )
        return run_headless(options)        # 交给装配层
```

### 3.2 真实代码

`frontend/headless.py:31-92`（节选）：

```python
@register_frontend
class HeadlessFrontend(FrontendPlugin):
    name = "headless"
    display_name = "Headless / Print Mode"

    def run(self, ctx, argv: list[str]) -> int:
        from src.entrypoints.headless import HeadlessOptions, run_headless

        options = HeadlessOptions(
            prompt=getattr(ctx.options, "prompt", None),
            output_format=getattr(ctx.options, "output_format", "text"),
            provider_name=ctx.provider_name,
            provider_instance=ctx.provider,
            model=ctx.options.model,
            fallback_model=ctx.options.fallback_model,
            max_turns=ctx.options.max_turns,
            permission_mode=ctx.options.permission_mode,
            ...
            external_session=getattr(ctx, "session", None),
        )
        try:
            return run_headless(options)
        finally:
            close = getattr(ctx, "close", None)
            if callable(close):
                close()
```

### 3.3 Python 语法讲解

| 语法 | 说明 |
|------|------|
| `@register_frontend` | **装饰器**：把 `HeadlessFrontend` 这个类注册到全局前端注册表。之后 `get_frontend("headless")` 能按名字找到它 |
| `class HeadlessFrontend(FrontendPlugin)` | **继承**：`HeadlessFrontend` 是 `FrontendPlugin` 的子类，必须实现 `run` 方法（父类定义的接口） |
| `getattr(ctx, "close", None)` | 防御性取值：ctx 可能有也可能没有 `close` 方法 |
| `if callable(close):` | **检查是否可调用**：`close` 可能是 None、可能是方法。`callable()` 判断"能不能当函数调" |
| `try: ... finally: close()` | **finally 保证执行**：无论 run_headless 成功还是抛异常，最后都会调用 `close()`（释放资源） |

### 3.4 具体参数示例

```python
# 上一层 RuntimeOptions 传进来 → HeadlessOptions 的实际值:
options = HeadlessOptions(
    prompt="你好",
    output_format="text",
    provider_name=None,          # 没指定 → run_headless 内部用默认
    provider_instance=<BaseProvider 实例>,   # 已建好的 provider（复用，不重建）
    model="claude-sonnet-4",
    max_turns=5,
    external_session=<Session 实例>,   # RuntimeContext 已建好会话，直接复用
)
# 注意: provider_instance 是关键——它把"已建好的 provider"传下去，避免重复初始化
```

---

## 第 4 章：`entrypoints/headless.py` —— 装配（攒齐所有零件）

### 4.1 这一层是干嘛的（整体）

`run_headless()` 是**装配车间**——它把"跑一次对话"需要的所有零件攒齐：
1. **校验参数**（output_format / input_format 合法性）
2. **建 Session**（新会话 / 恢复会话 / fork）
3. **建 tool_registry**（工具注册表，可能按 allowed/disallowed 过滤）
4. **建 tool_context**（工具上下文：权限、工作目录、abort 控制器）
5. **拼 system_prompt**（`build_effective_system_prompt` + 追加主 agent 提示词）
6. **调 `_run_one_agent_loop()`** → 进入 query 循环

**伪代码**：

```python
def run_headless(options):
    校验参数(options)
    session = 建会话(options)              # 新/恢复/fork
    tool_registry = 建工具注册表()          # 可按白名单/黑名单过滤
    tool_context = ToolContext(权限, 目录, abort)   # 工具上下文
    system_prompt = build_effective_system_prompt(style, tool_context)
    if options.append_system_prompt:
        system_prompt += 追加内容          # 主 agent 的身份提示词在这
    for prompt in 输入列表:
        result = _run_one_agent_loop(session, provider, tool_registry,
                                     tool_context, system_prompt, ...)
```

### 4.2 真实代码（关键段）

**Session 装配**（`headless.py:327-343`）——三种来源按优先级：

```python
if options.external_session is not None:      # 1. RuntimeContext 已建好 → 复用
    session = options.external_session
elif options.resume_session_id:               # 2. 指定 --resume → 恢复
    session = Session.resume(options.resume_session_id)
else:                                         # 3. 都没有 → 新建
    session = Session.create(provider_name, getattr(provider, "model", model or ""))
```

**tool_context 创建**（`headless.py:508-516`）：

```python
tool_context = ToolContext(
    workspace_root=workspace_root,            # 工作目录
    permission_context=_perm_setup.context,   # 权限上下文（允许/拒绝规则）
    abort_controller=abort_controller,        # Ctrl+C 中止信号
    env=options.env,
    startup_agent=options.startup_agent,      # 主 agent 身份！
    agent_type=getattr(options.startup_agent, "agent_type", None),
    bundle_context=getattr(options, "bundle_context", None),
)
```

**system_prompt 拼接**（`headless.py:2087-2104`）：

```python
effective_system_prompt = build_effective_system_prompt(_style_prompt, tool_context, provider=provider)
...
if options.append_system_prompt:              # ← 主 agent 的提示词追加在这
    effective_system_prompt = f"{effective_system_prompt}\n\n{options.append_system_prompt}"
```

### 4.3 Python 语法讲解

| 语法 | 说明 |
|------|------|
| `Session.resume(...)` / `Session.create(...)` | **类方法**（classmethod）——不通过实例调用，通过类直接调用，返回一个新实例 |
| `getattr(provider, "model", model or "")` | 嵌套：`model or ""` 先算（model 为空就用空串），再 getattr 兜底 |
| `_perm_setup.context` | `_perm_setup` 是 `setup_permissions()` 的返回值（一个对象），`.context` 是它的属性 |
| `f"{a}\n\n{b}"` | **f-string 字符串拼接**：把两段提示词用两个换行连起来 |

### 4.4 具体参数示例

```python
# build_effective_system_prompt 返回的实际结构（简化）:
effective_system_prompt = [
    {"type": "text", "text": "# 系统提示词基础部分（身份/规则/工具使用指南...）"},
    {"type": "text", "text": "# 输出样式提示词"},
    # ↓ append_system_prompt 追加的内容（主 agent 身份）:
    {"type": "text", "text": "You are a general-purpose agent. Given the user's message, you should use the tools available to complete the task..."}
]

# 输入处理（headless.py:670-676）:
# 你输入 "你好" → 
prompt_text = "你好"
inputs = [UserInputMessage(text="你好", raw={"prompt": "你好"})]
```

---

## 第 5 章：`query/agent_loop_compat.py` —— 桥接层（翻译参数）

### 5.1 这一层是干嘛的（整体）

**为什么需要"桥"？** 因为上游（`src/`，安装时拉取的 Claude Code 上游代码）的"AgentLoop"语义，和下游（`clawcodex_ext/`）的 `query()` 参数形状**不一样**。桥接层负责**翻译**：

- 上游说：`initial_messages`（一个消息列表）+ 各种参数
- 下游 `query()` 要：一个打包好的 `QueryParams` 对象

`run_query_as_agent_loop()` 就是这个翻译官——它接收上游风格参数，**组装出 `QueryParams`**，然后 `while True` 驱动 `query()`。

**伪代码**：

```python
def run_query_as_agent_loop(initial_messages, provider, tool_registry,
                            tool_context, system_prompt, max_turns, ...):
    messages = 复制(initial_messages)
    while True:
        params = QueryParams(           # 翻译：把参数打包
            messages=messages,
            system_prompt=system_prompt,
            tools=effective_tools,
            tool_registry=tool_registry,
            tool_use_context=tool_context,
            provider=provider,
            max_turns=max_turns,
            ...
        )
        async for msg in query(params, terminal_holder=holder):   # ← 真正调 query()
            处理消息(msg)
        if 循环结束: break
    return AgentLoopRunResult(messages, terminal)
```

### 5.2 真实代码（关键段）

**QueryParams 组装**（`agent_loop_compat.py:695-731`）：

```python
params = QueryParams(
    messages=messages_for_query,
    system_prompt=system_prompt,
    tools=effective_tools,
    tool_registry=tool_registry,
    tool_use_context=tool_context,
    provider=provider,
    abort_controller=abort_controller,
    max_turns=max_turns,
    fallback_model=fallback_model,
    pipeline_config=pipeline_config,
    query_source=query_source,
    on_text_chunk=on_text_chunk,
    on_thinking_chunk=on_thinking_chunk,
    on_attachment=on_attachment,
)
```

**循环驱动**（`agent_loop_compat.py:742-755`）：

```python
while True:
    holder = TerminalHolder()
    params = _make_params(next_messages)

    async def _drain_turn() -> None:
        async for msg in query(params, terminal_holder=holder):   # ← query() 调用点
            if isinstance(msg, StreamEvent):
                continue
            conversation_messages.append(msg)
            ...
```

### 5.3 Python 语法讲解

| 语法 | 说明 |
|------|------|
| `while True:` | 无限循环——靠内部 `break` / `return` 退出 |
| `async def` / `async for` | **异步**语法：`query()` 是异步生成器（async generator），必须用 `async for` 消费；`_drain_turn` 是异步函数（协程） |
| `async def _drain_turn() -> None:`（函数内再定义函数） | **嵌套函数**：在循环内定义，捕获外层的 `params`、`holder`、`conversation_messages` 等变量（闭包） |
| `isinstance(msg, StreamEvent)` | **类型判断**：过滤掉"流事件"（如 stream_request_start），只处理真正的消息 |
| `nonlocal main_transcript, num_turns` | **nonlocal 声明**：告诉 Python"我要修改外层函数定义的变量"（默认嵌套函数只能读不能改外层变量） |

### 5.4 具体参数示例

```python
# run_query_as_agent_loop 收到的真实参数（从 headless 传来）:
initial_messages = [UserMessage(content="你好")]        # 用户消息
system_prompt = [{"type": "text", "text": "You are..."}] # 上章拼好的
max_turns = 5                                            # --max-turns 5
query_source = "sdk"                                     # 来源标签

# 翻译后的 QueryParams（关键字段值）:
params = QueryParams(
    messages=[UserMessage(content="你好")],
    system_prompt=[{"type": "text", "text": "You are..."}],
    tools=<工具集（含 Read/Glob/Bash/Agent...）>,
    provider=<BaseProvider 实例>,
    max_turns=5,
    query_source="sdk",
)
```

---

## 第 6 章：`query/query.py` —— 引擎（主循环本体）

### 6.1 这一层是干嘛的（整体）

`query()` 和 `_query_impl()` 是**整个程序的心脏**。前面 5 层都是"送快递的"，到这里才是"干活的地方"。

- `query()`（query.py:3428）：**薄包装**——进入/退出技能作用域，把活转给 `_query_impl`
- `_query_impl()`（query.py:2078）：**主循环**——`while True` 里反复"推理 → 判终止 → 执行工具 → 回填"

**伪代码**（第 1 章的核心循环在这里落地）：

```python
async def query(params, terminal_holder=None):
    进入技能作用域()
    async for item in _query_impl(params, terminal_holder):
        yield item                      # 逐条转发给调用方
    退出技能作用域()

async def _query_impl(params, terminal_holder=None):
    state = QueryState(messages=params.messages, ...)   # 状态对象
    while True:
        messages = state.messages
        调模型 → assistant_messages, tool_use_blocks
        if 没有 tool_use:                      # 模型说完了
            set_terminal(holder, ...completed)
            return
        执行工具(tool_use_blocks) → tool_results
        重建 state = QueryState(原始历史 + 助手回复 + 工具结果)
        if 轮数超限:
            set_terminal(holder, ...max_turns)
            return
```

### 6.2 真实代码

**query() 薄包装**（`query.py:3428-3439`）：

```python
async def query(
    params: QueryParams,
    *,
    terminal_holder: TerminalHolder | None = None,
) -> AsyncGenerator[Message | StreamEvent, None]:
    context = params.tool_use_context
    _begin_skill_runtime_scope(context)          # 进入技能作用域
    try:
        async for item in _query_impl(params, terminal_holder=terminal_holder):
            yield item                          # 逐条转发
    finally:
        _restore_skill_runtime_scope(context)   # 退出技能作用域（保证执行）
```

**_query_impl 初始化**（`query.py:2104-2118`）：

```python
holder = terminal_holder or TerminalHolder()    # 终止结果收纳盒
natural_termination: list[bool] = [False]       # "自然终止"标记（list 包着用于闭包修改）
state = QueryState(
    messages=list(params.messages),              # 复制消息（不污染原列表）
    tool_use_context=params.tool_use_context,
    max_output_tokens_override=params.max_output_tokens_override,
)
config = build_query_config()
```

**_query_impl 主循环骨架**（`query.py:2285-2309` + 3350-3356）：

```python
while True:
    messages = state.messages
    for cb in state.on_turn_start_callbacks:    # 每轮开始回调
        cb(state)
    ...
    yield StreamEvent(type="stream_request_start")   # 通知 UI：开始请求
    ...
    # 调模型（_call_model_sync）→ assistant_messages, tool_use_blocks
    ...
    # 没有 tool_use → completed（见 query.py:2830 附近）
    set_terminal(holder, natural_termination, Terminal(reason="completed"))
    return
    ...
    # 执行工具 → tool_results
    ...
    next_turn_count = turn_count + 1
    exceeded_max_turns = _exceeded_max_turns(next_turn_count)   # 轮数检查
    if exceeded_max_turns is not None:
        yield _create_max_turns_attachment(exceeded_max_turns, next_turn_count)
        _finish_at_max_turns(next_turn_count)                   # max_turns 终止
        return
```

### 6.3 Python 语法讲解

| 语法 | 说明 |
|------|------|
| `async def ... -> AsyncGenerator[...]` | 返回类型标注：这是一个**异步生成器**（用 `yield` 的不是 `return` 值） |
| `yield item` | **生成器产出**：把 item 交给 `async for` 的调用方，**暂停**在这里，等调用方要下一个再继续 |
| `try: ... finally: ...` | 无论成功失败都执行 finally——保证技能作用域一定能退出 |
| `list(params.messages)` | **浅拷贝**：新建一个列表（元素还是同一批对象），避免外面改影响内部状态 |
| `terminal_holder or TerminalHolder()` | `or` 的妙用：如果传了就用传入的，没传就新建一个（空盒子） |
| `while True:` + `return` | 无限循环靠显式 return 退出，每个出口先 `set_terminal` 记原因 |

### 6.4 具体参数示例

```python
# 到达 query() 时的完整参数（把前面所有层的"快递"汇总）:
params = QueryParams(
    messages=[UserMessage(content="你好")],
    system_prompt=[{"type": "text", "text": "You are a general-purpose agent..."}],
    tools=<工具集>,
    tool_registry=<注册表>,
    tool_use_context=<ToolContext 实例>,   # 权限/目录/abort
    provider=<BaseProvider 实例>,
    abort_controller=<AbortController 实例>,
    max_turns=5,
    query_source="sdk",
)

# 一次典型的主循环回合（第一轮）:
# 1. 调模型 → 模型回复: "你好！我可以帮你..."（没有 tool_use）
# 2. 没有 tool_use → Terminal(reason="completed")
# 3. return → 调用方拿到回复 "你好！我可以帮你..."
# （如果模型要工具: tool_use(Read) → 执行 → 回填 → 再调模型 → 直到没有 tool_use）

# 轮数上限触发示例（max_turns=5 时）:
# 第 6 轮开始时 _exceeded_max_turns(6) 返回 5（因为 6 > 5）
# → Terminal(reason="max_turns", turn_count=6)
```

---

## 第 7 章：`_query_impl` 专项详解 —— 主循环内部到底怎么转

第 6 章只给了 `_query_impl` 的骨架。这一章把它**完整拆开**，讲清楚一个 `while True` 循环里每一段在干什么、按什么顺序执行、每个出口怎么终止。

### 7.1 总体结构（整体 → 局部）

`_query_impl`（`query.py:2078`）是一个约 **1300 行**的函数，可以分成 **4 大块**：

```
_query_impl(params, terminal_holder)
  │
  ├─【块 A】初始化（2078-2132）: 造状态对象、注册表、guard
  ├─【块 B】定义嵌套函数（2134-2284）: _goal_start_turn / _exceeded_max_turns / _finish_at_max_turns
  ├─【块 C】while True 主循环（2285-3399）: 核心
  │    └─ 每轮: 压缩 → 调模型 → 判终止 → 执行工具 → 回填 → 检查 → 下一轮
  └─【块 D】函数结束（3399 后）: 循环自然退出（async generator 结束）
```

**主循环每轮的 7 个阶段**（这是本章的核心）：

```
第 N 轮开始
  │
  ├─ Phase 0: 压缩（上下文太长先瘦身）
  ├─ Phase 1: 调模型（含重试/降级）→ assistant_messages + tool_use_blocks
  ├─ Phase 2: 判终止①（模型说完了？出错？中止？）→ 可能 return
  ├─ Phase 3: hooks + 续写判断（要继续吗？）
  ├─ Phase 4: 执行工具（_run_tools_partitioned）→ tool_results
  ├─ Phase 5: 判终止②（中止？hook_stopped？工具失败？max_turns？）→ 可能 return
  └─ Phase 6: 重建 state（原始历史+助手回复+工具结果+注入消息）→ 回到 while True
```

### 7.2 【块 A】初始化：攒齐"这一趟"的装备

**真实代码**（`query.py:2104-2132`）：

```python
holder = terminal_holder or TerminalHolder()          # 终止结果收纳盒
natural_termination: list[bool] = [False]              # 自然终止标记
state = QueryState(
    messages=list(params.messages),                    # 复制消息，不污染原列表
    tool_use_context=params.tool_use_context,
    max_output_tokens_override=params.max_output_tokens_override,
)
config = build_query_config()                          # 全局配置
tool_failure_guard_state = create_tool_failure_loop_guard_state()  # 工具失败防循环
budget_tracker = create_budget_tracker()               # token 预算追踪
params.tool_use_context.options.tools = list(params.tools)  # 工具列表挂到上下文
```

**语法讲解**：

| 语法 | 说明 |
|------|------|
| `terminal_holder or TerminalHolder()` | 传入的没有就新建一个——"有就用，没有就造" |
| `list[bool] = [False]` | 用**单元素列表**包一个 bool——因为嵌套函数要改它，list 是可变对象，闭包能改 |
| `QueryState(messages=list(...))` | 浅拷贝消息列表——**不复制消息对象本身**，只复制"装消息的列表" |
| `create_tool_failure_loop_guard_state()` | 工厂函数：返回一个全新的 guard 状态对象（记录"哪个工具失败了几次"） |

### 7.3 【块 B】嵌套函数：循环的"小工具"

**`_exceeded_max_turns`**（`query.py:2265-2269`）——问"轮数超了吗"：

```python
def _exceeded_max_turns(next_turn_count: int) -> int | None:
    max_turns = int(params.max_turns or 0)          # 读外层 params
    if max_turns and next_turn_count > max_turns:   # 设了上限且下一轮超出
        return max_turns                            # 返回上限值（非 None = 超了）
    return None                                     # None = 没超
```

**`_finish_at_max_turns`**（`query.py:2271-2277`）——"轮数超了，收工"：

```python
def _finish_at_max_turns(next_turn_count: int) -> None:
    error = RuntimeError("max turns reached")
    set_terminal(
        holder,                        # ← 写进收纳盒（外层变量）
        natural_termination,
        Terminal(reason="max_turns", turn_count=next_turn_count),
    )
```

**语法讲解**：

| 语法 | 说明 |
|------|------|
| 嵌套函数读外层变量 | 直接读 `params`、`holder`——**闭包**自动捕获，不用传参 |
| 返回 `int \| None` | 用 `None` 表示"没有"（类似别的语言 null），调用方 `if exceeded is not None` 判断 |
| `int(params.max_turns or 0)` | `or 0`：max_turns 是 None 就用 0，`int()` 转成整数 |

### 7.4 【块 C】主循环 Phase 0-1：压缩 + 调模型

**Phase 0 压缩**（`query.py:2313-2334`）——上下文太长先瘦身：

```python
if params.pipeline_config is not None:
    est_input_tokens = rough_token_count_estimation_for_messages(messages)  # 估算 token
    pipeline_result = await run_compression_pipeline(
        messages,
        input_token_count=est_input_tokens,
        config=params.pipeline_config,
    )
    if pipeline_result.tokens_saved > 0:
        messages = pipeline_result.messages           # 用压缩后的消息
```

**语法讲解**：`await` = 等待异步结果（压缩可能耗时）；`tokens_saved > 0` = 只有真的省了 token 才替换消息。

**Phase 1 调模型**（`query.py:2496-2511`）——真正的 LLM 调用（含重试）：

```python
while True:                                  # 内层重试循环
    _streamed_any[0] = False
    try:
        returned_assistants, returned_tool_blocks = await _call_model_sync(
            provider=params.provider,
            messages=messages,
            system_prompt=current_system_prompt,
            tools=effective_tools,
            model=tool_use_context.skill_model_override or params.model,
            abort_signal=params.abort_controller.signal,   # 中止信号传进去
            on_text_chunk=_marking_chunk_cb,               # 流式回调
            extended_thinking=params.extended_thinking,
            thinking_effort=params.thinking_effort,
        )
        break                                # 成功 → 跳出重试循环
    except AbortError:
        raise                                # 用户中止 → 直接抛
    except Exception as retry_exc:
        # 判断是否可重试 → 退避 sleep → 重试（见下）
        ...
```

**重试逻辑**（`query.py:2523-2610` 精简）：

```python
is_529 = _is_overloaded_error(retry_exc)       # 服务过载？
if not is_529 and not 可重试(分类): raise        # 不可重试直接抛
_consecutive_529s += 1 if is_529 else 重置
if 529次数 >= MAX_529_RETRIES and params.fallback_model:
    params.provider.model = params.fallback_model   # 切换备用模型
    yield SystemMessage(content="Switched to ...", level="warning")  # 通知用户
    continue                                  # 用新模型重试
delay = 计算退避延迟() + 随机抖动()              # 指数退避
yield SystemMessage(content=f"retrying in {delay:.1f}s ...")
await 分段sleep(delay)                        # 可被中止的等待
continue                                      # 重试
```

**语法讲解**：

| 语法 | 说明 |
|------|------|
| `_streamed_any = [False]` | 单元素列表标记"是否已流式输出过内容"——若已输出过就不再重试（避免重复文本） |
| `_marking_chunk_cb` | 包装回调：内部把 `_streamed_any[0] = True`（记录已输出）再转发给外层回调 |
| `or params.model` | skill 有模型覆盖就用 skill 的，否则用 params.model |
| `f"retrying in {delay:.1f}s"` | f-string 格式化：`:.1f` = 保留 1 位小数 |
| `while _remaining > 0:` 分段 sleep | 每 0.25 秒醒来检查一次中止信号——**不让 ESC 卡在长 sleep 里** |

### 7.5 【块 C】Phase 2-3：判终止① + hooks

**调完模型后**（`query.py:2611-2632`）：

```python
assistant_messages = returned_assistants
tool_use_blocks = returned_tool_blocks
needs_follow_up = len(tool_use_blocks) > 0      # ← 核心判断：有没有工具要执行

# post_llm hook：外部策略可修改回复
hook_result = _call_hooks_if_enabled("post_llm", assistant_messages, tool_use_blocks, ...)
assistant_messages = hook_result[0]
tool_use_blocks = hook_result[1]
```

**`needs_follow_up` 是分水岭**：

```text
needs_follow_up = len(tool_use_blocks) > 0
        │
        ├─ False（没有工具请求）→ 走"完成"分支（Phase 2 终止）
        └─ True（有工具请求）→ 走"执行工具"分支（Phase 4）
```

**完成分支**（`query.py:2819-2832` 精简）：

```python
if last_message and getattr(last_message, "isApiErrorMessage", False):
    # 最后一条是 API 错误 → 处理错误
    if 是 goal 模式: set_terminal(...Terminal(reason="model_error", error=...))
    else: set_terminal(...Terminal(reason="completed"))    # 无 goal 时也按完成处理
    return
# 正常完成
set_terminal(holder, natural_termination, Terminal(reason="completed"))
return
```

**语法讲解**：`getattr(last_message, "isApiErrorMessage", False)`——消息可能没有这个属性，默认 False；`isMeta`/`isApiErrorMessage` 是消息的标记位。

### 7.6 【块 C】Phase 4：执行工具

**工具执行**（`query.py:3237-3242`）：

```python
tool_results = await _run_tools_partitioned(
    tool_use_blocks,        # 模型要执行的工具列表
    params.tool_registry,   # 工具注册表（按名字找实现）
    tool_use_context,       # 工具上下文（权限/目录）
    effective_tools,        # 可用工具集
)
```

**执行前**，`query.py:3178-3182` 会先发"进度通知"：

```python
for block in tool_use_blocks:
    yield SystemMessage(
        content=f"Running tool: {block.name}",     # 如 "Running tool: Read"
        subtype="tool_use_progress",
    )
```

**`_run_tools_partitioned` 内部逻辑**（它把工具按"能否并行"分批）：

```python
# _partition_tool_calls 把工具分组成批次:
#   [并行批]: 多个并发安全工具一起跑
#   [独占批]: 并发不安全的工具单独跑
_batches = _partition_tool_calls(tool_use_blocks, effective_tools)
for batch in _batches:
    if batch.is_concurrent_safe:
        await asyncio.gather(*[执行(t) for t in batch.blocks])   # 并行
    else:
        for t in batch.blocks:
            await 执行(t)                                        # 串行
```

**语法讲解**：`asyncio.gather(*[...])` = 并发执行多个异步任务；`*` 解包列表成多个位置参数。

### 7.7 【块 C】Phase 5-6：判终止② + 重建 state

**执行完工具后的检查顺序**（`query.py:3281-3356`，**优先级从高到低**）：

```python
# ① 用户中止（最高优先）
if params.abort_controller.signal.aborted:
    set_terminal(holder, natural_termination, Terminal(reason="aborted_tools"))
    return

# ② hook_stopped（stop hook 说停）
if any(_is_hook_stopped_continuation(msg) for msg in tool_results):
    set_terminal(holder, natural_termination, Terminal(reason="hook_stopped"))
    return

# ③ 工具失败 guard（同一工具反复失败）
guard_decision = update_tool_failure_loop_guard(...)
if guard_decision.tripped:
    set_terminal(holder, natural_termination, Terminal(reason="tool_failure_loop"))
    return

# ④ max_turns（轮数上限）
next_turn_count = turn_count + 1
exceeded_max_turns = _exceeded_max_turns(next_turn_count)
if exceeded_max_turns is not None:
    yield _create_max_turns_attachment(exceeded_max_turns, next_turn_count)
    _finish_at_max_turns(next_turn_count)
    return
```

**最后，重建 state**（`query.py:3379-3399`）——这是"观察回填"的实现：

```python
state = QueryState(
    messages=[
        *messages,              # 原始历史
        *assistant_messages,    # 助手回复（含 tool_use）
        *tool_results,          # 工具结果（回填！）
        *advisory_messages,     # 工具失败建议
        *injected_messages,     # goal 提示 + 用户注入消息
    ],
    tool_use_context=tool_use_context,
    turn_count=next_turn_count,          # 轮数 +1
    transition=Transition(reason="next_turn"),
)
# 回到 while True → 下一轮：模型看到完整对话（含工具结果）
```

**语法讲解**：

| 语法 | 说明 |
|------|------|
| `[*a, *b, *c]` | **列表解包**：把多个列表平铺成一个新列表——"拼接"的简洁写法 |
| `Transition(reason="next_turn")` | 记录"为什么进入这个状态"——调试/追溯用 |
| `turn_count=next_turn_count` | 每轮 +1，供 max_turns 判断 |

### 7.8 完整流程图（一图看懂）

```
┌───────────── while True（第 N 轮）─────────────┐
│                                                │
│  Phase 0: 压缩流水线（token 超限时）             │
│      ↓                                         │
│  Phase 1: _call_model_sync() 调模型             │
│      │  内层重试循环: 529/限流 → 退避 → 重试      │
│      │  连续 529 → 切 fallback_model            │
│      ↓                                         │
│  Phase 2: needs_follow_up = 有 tool_use 吗？    │
│      ├─ 没有 → completed / model_error → return │
│      └─ 有 → 继续                              │
│      ↓                                         │
│  Phase 3: post_llm hook + 续写判断              │
│      ↓                                         │
│  Phase 4: _run_tools_partitioned() 执行工具     │
│      │  并行批 / 独占批 → tool_results          │
│      ↓                                         │
│  Phase 5: 终止检查（优先级递减）                 │
│      ├─ abort?        → aborted_tools → return │
│      ├─ hook_stopped? → hook_stopped → return  │
│      ├─ 工具失败循环?   → tool_failure_loop → return │
│      └─ max_turns?    → max_turns → return     │
│      ↓                                         │
│  Phase 6: 重建 state（历史+回复+结果+注入）       │
│      └─ 回到 while True（第 N+1 轮）            │
└────────────────────────────────────────────────┘
```

### 7.9 具体参数示例（走一遍真实数据）

```python
# 场景: 用户说 "帮我读一下 config.yaml"
# 假设第一轮模型回复:
assistant_messages = [AssistantMessage(content="好的，我来读配置文件")]
tool_use_blocks = [ToolUseBlock(name="Read", input={"path": "config.yaml"})]
needs_follow_up = True          # 有工具请求

# Phase 4 执行工具:
tool_results = [UserMessage(content=[ToolResultBlock(tool_use_id="toolu_01", content="server:\n  port: 8080")])]

# Phase 6 重建 state（第二轮模型看到）:
state.messages = [
    UserMessage("帮我读一下 config.yaml"),          # 原始
    AssistantMessage("好的，我来读配置文件"),        # 助手回复
    UserMessage(content="server:\n  port: 8080"),  # 工具结果回填
]

# 第二轮: 模型看到文件内容 → 回复 "config.yaml 配置了端口 8080"（没有 tool_use）
needs_follow_up = False
# Phase 2: → set_terminal(Terminal(reason="completed")) → return
```

```python
# 场景: 连续 529 过载 → 模型降级
# 第 4 次 529 后（MAX_529_RETRIES=3）:
_consecutive_529s = 4
if is_529 and _consecutive_529s >= 3 and params.fallback_model:
    params.provider.model = "deepseek-chat"        # 从 claude-sonnet 切到备用
    yield SystemMessage(content="Switched to deepseek-chat due to high demand for claude-sonnet-4")
    _consecutive_529s = 0
    continue                                       # 用新模型重试
```

```python
# 场景: max_turns=5，第 5 轮结束工具执行后:
next_turn_count = 5 + 1 = 6
_exceeded_max_turns(6) → max_turns=5, 6 > 5 → 返回 5（非 None）
yield _create_max_turns_attachment(5, 6)           # 通知"达到上限"
_finish_at_max_turns(6) → set_terminal(Terminal(reason="max_turns", turn_count=6))
return
```

### 7.10 反问题（自测）

**Q1.** `needs_follow_up` 是怎么算的？它决定了什么？

<details><summary>答案</summary>
`needs_follow_up = len(tool_use_blocks) > 0`——模型这轮回复里有没有工具请求。没有 → 走完成分支（completed）；有 → 继续执行工具。它是"这轮到底结束没结束"的分水岭。
</details>

**Q2.** 为什么重试逻辑要用 `_streamed_any = [False]` 这个单元素列表？

<details><summary>答案</summary>
两层原因：① 嵌套函数要修改它，list 是可变对象，闭包才能改（普通 bool 变量改了不生效）；② 语义上"只要已经流式输出过内容就不再重试"——否则会重复给用户显示已输出的文本。
</details>

**Q3.** 工具执行时为什么要分批（并行/独占）？

<details><summary>答案</summary>
并发安全的工具（如两个 Read）可以同时跑省时间；并发不安全的工具（如两个 Write 到同一文件）必须串行防冲突。`_partition_tool_calls` 按 `is_concurrent_safe` 标记分组，`asyncio.gather` 并行执行安全批。
</details>

**Q4.** Phase 5 的终止检查顺序为什么 abort 优先于 max_turns？

<details><summary>答案</summary>
用户中止是"人的意志"，轮数上限是"系统兜底"。如果两个同时发生，应该尊重用户（aborted_tools），而不是报告系统限制（max_turns）——注释明确"user-driven abort wins"。
</details>

**Q5.** `state = QueryState(messages=[*messages, *assistant_messages, *tool_results, ...])` 为什么每轮都重建，而不是原地 append？

<details><summary>答案</summary>
**不可变快照**设计。每轮生成全新的 QueryState，避免上轮状态被意外修改污染下轮；同时记录 `turn_count`、`transition` 等元信息，方便调试和追溯"这一轮为什么这样"。这也呼应了文档里"消息列表永远是模型视角的完整对话"的不变式。
</details>

---

## 总结：一条命令的完整旅程（带真实参数）

```
clawcodex-dev -p "你好" --model claude-sonnet-4 --max-turns 5
  │
  ▼
① main()                          cli/main.py:30
  │  sys.argv = ["clawcodex-dev", "-p", "你好", "--model", "claude-sonnet-4", "--max-turns", "5"]
  │
  ▼
② run_cli(argv)                   cli/dispatch.py:197
  │  args.prompt = "你好" / args.print = True / args.model = "claude-sonnet-4"
  │  RuntimeOptions(provider_name=None, model="claude-sonnet-4", prompt="你好",
  │                 max_turns=5, max_turns_explicit=True, ...)
  │  RuntimeContext.build(opts) → ctx
  │  get_frontend("headless").run(ctx, argv)
  │
  ▼
③ HeadlessFrontend.run(ctx, argv) frontend/headless.py:36
  │  HeadlessOptions(prompt="你好", provider_instance=<provider>, model="claude-sonnet-4",
  │                 max_turns=5, external_session=<session>, ...)
  │
  ▼
④ run_headless(options)           entrypoints/headless.py:212
  │  session = Session.create("anthropic", "claude-sonnet-4")
  │  tool_registry = build_default_registry(provider)
  │  tool_context = ToolContext(workspace_root=..., permission_context=..., abort_controller=...)
  │  effective_system_prompt = build_effective_system_prompt(style, tool_context)
  │  effective_system_prompt += append_system_prompt   # 主 agent 身份
  │  _run_one_agent_loop(session, provider, tool_registry, tool_context, ...)
  │
  ▼
⑤ run_query_as_agent_loop(...)    agent_loop_compat.py:561
  │  QueryParams(messages=[UserMessage("你好")], system_prompt=[...],
  │             tools=..., provider=..., max_turns=5, query_source="sdk")
  │  while True: async for msg in query(params, terminal_holder=holder)
  │
  ▼
⑥ query(params) → _query_impl()   query/query.py:3428 / 2078
  │  第 1 轮: 调模型 → 无 tool_use → Terminal("completed") → return
  │  结果: "你好！我可以帮你……"
  │
  ▼
返回退出码 0，进程结束
```

---

## 反问题（自测）

**Q1.** `main()` 为什么要在函数**内部** import `run_cli`，而不是文件顶部？

<details><summary>答案</summary>
延迟导入。`run_cli` 依赖一大串模块（argparse、frontend 等），如果文件顶部 import，程序一启动就要全部加载；函数内 import 只有真正调用时才加载，加快冷启动。而且避免循环导入（dispatch 可能反向依赖 main 的模块）。
</details>

**Q2.** `RuntimeOptions` 和 `HeadlessOptions` 字段几乎一样，为什么不直接用同一个？

<details><summary>答案</summary>
**分层解耦**。RuntimeOptions 是"CLI 层"的参数（面向命令行语义），HeadlessOptions 是"headless 层"的参数（面向执行语义）。两层可以独立演进——比如 headless 新增一个 `record` 参数，不必污染 CLI 层。这正是"每层一个自己的参数对象"的架构。
</details>

**Q3.** `query()` 为什么是薄包装，把循环放在 `_query_impl()`？

<details><summary>答案</summary>
**关注点分离**。`query()` 负责"进入/退出技能作用域"（资源管理，用 try/finally 保证清理），`_query_impl()` 负责"循环逻辑"（业务）。这样调用方只面对干净的 `query()`，而 `_query_impl` 的复杂循环被隔离。类似"门卫"和"车间"的分工。
</details>

**Q4.** 5 层调用下来，最终给模型的是什么？

<details><summary>答案</summary>
三个东西：① `system_prompt`（列表形式，含基础指令 + 输出样式 + 主 agent 身份）；② `messages`（用户消息历史）；③ `tools`（工具 schema 列表）。`_query_impl` 每次调模型把这三样传给 provider，provider 转成 API 请求发给 LLM。
</details>

**Q5.** 如果 `max_turns=5`，模型 3 轮就完成了，会发生什么？

<details><summary>答案</summary>
不会触发 max_turns。第 3 轮模型没有 tool_use → `Terminal("completed")` 提前 return。`_exceeded_max_turns` 只在"轮数真的超过 5"时才触发——max_turns 是**上限兜底**，不是强制跑满。
</details>

---

*下一站：第 2 章主题 2（Context）—— 拼出模型看到的完整上下文。*
