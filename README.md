# 学习计划：从零理解 clawcodex-ascend

> 目标读者：了解一些 agent 基础知识、想系统学习本项目的新人。
> 原则：由外到内、先跑起来再读代码、每阶段都有明确目标和一个小练习。
> 已有配套文档：`docs/architecture.md`（模块总览）、`docs/runtime_flow.md`（一条命令怎么跑完）、`docs/memory_architecture.md`（记忆能力）。

---

## 0. 先弄清楚这个项目是什么（30 秒版）

**clawcodex-ascend 是一个命令行 AI 助手**（"Claude Code"的 Python 重实现）。你输入一句话，它调大模型，必要时自动调用工具（读文件、执行命令、改代码），再继续调模型，直到任务完成。

代码分三层，**先记住这个结构**：

```
src/             上游代码（安装时才拉取，平时目录是空的，不用管）
clawcodex_ext/   本项目主体（绝大部分代码在这，重点学这里）
extensions/      可选插件（团队记忆、lkb 等，先不深究）
```

它的核心动作只有一个循环（后面第 3 阶段细讲）：

```
用户输入 → 拼上下文 → 调模型 → 看结果
              ↑                    │
              └── 有工具要执行 ─────┘（执行工具，结果带回，再调模型）
                  直到模型不再调用工具 → 输出回答
```

---

## 1. 阶段一：把项目跑起来（半天）

**目标**：亲眼看到 agent 工作，建立直觉。

**步骤**：

```bash
cd clawcodex-ascend
uv sync                                    # 安装依赖（会生成 clawcodex-dev 命令）
uv run clawcodex-dev -p "你好"             # 单次模式：问一句，得到回答
uv run clawcodex-dev -p "列出当前目录文件"  # 这次它会调用工具（Bash/Read）
echo "你好" | uv run clawcodex-dev -p      # 从管道读输入
uv run clawcodex-dev                       # 交互模式（体验完 /exit 退出）
```

**观察**：第二次命令和第一次的差别——它"会动手了"。这就是 agent 和普通聊天机器人的区别。

**练习**：跑 `-p "查看当前目录结构"`，观察它调了什么工具、分几步完成。

---

## 2. 阶段二：理解入口链——一条命令怎么变成一次对话（1 天）

**目标**：从敲命令到出结果，把每一跳对应到具体文件。

**必读**：`docs/runtime_flow.md`（已写好，直接看，里面有完整例子和文件路径）。

**再读这几个文件**（每个只读关键函数）：

| 文件 | 看什么 |
|------|--------|
| `pyproject.toml:160` | `[project.scripts]`——`clawcodex-dev` 命令指向哪个函数 |
| `clawcodex_ext/cli/main.py` | `main()`——入口函数，转给 run_cli |
| `clawcodex_ext/cli/dispatch.py:197` | `run_cli()`——解析参数，决定走哪种界面 |
| `clawcodex_ext/entrypoints/headless.py:212` | `run_headless()`——单次模式的准备工作 |

**关键概念（一句话解释）**：
- **entry point**：打包时声明"命令名 → 函数"的对应关系，装完就有了 `clawcodex-dev` 命令。
- **headless / repl / tui**：三种界面。headless 是"给参数出结果就退出"（脚本用）；repl 和 tui 是"交互式，一直等你输入"（人用）。

**练习**：打开 `runtime_flow.md`，用手指着"一句话图"，对照源码把每行走一遍。

---

## 3. 阶段三：理解 agent 的核心循环 query()（2~3 天，最重要）

**目标**：看懂 agent 的本质——"调模型 → 执行工具 → 再调模型"这个循环。

**必读**：`clawcodex_ext/query/query.py`，**只读三个函数**（这个文件很大，别从头读到尾）：

| 函数 | 行号 | 干什么 |
|------|------|--------|
| `query()` | 3428 | 主循环入口（每轮重复：压缩→调模型→执行工具→判断是否继续） |
| `_call_model_sync()` | 843 | 真正调用大模型（把消息转成 API 格式、组装工具清单、发请求） |
| `_run_tools_partitioned()` | 1823 | 执行工具（安全的并行，不安全的串行） |

**关键概念**：
- **turns（轮次）**：一次"调模型→执行工具"叫一轮。普通问答 1 轮，用工具的任务可能好几轮。
- **工具（tools）**：agent 的"手"。模型不会自己读文件，它返回"我要调用 Bash，参数是 xxx"，代码去执行，把结果还给模型。
- **循环怎么结束**：模型这次不调用任何工具 → 说明回答完了 → 循环结束。

**练习**：跑 `CLAWCODEX_DEBUG=1 uv run clawcodex-dev -p "查看当前目录"`，观察日志里的 turn 数和工具调用。

---

## 4. 阶段四：理解"agent 看到什么"——上下文组装（1~2 天）

**目标**：搞懂系统提示词（system prompt）是怎么拼出来的，记忆在其中扮演什么角色。

**必读**：
- `clawcodex_ext/context_system/builder.py:42`（`build_context_prompt`，看它拼了哪几段）
- `clawcodex_ext/context_system/section_registry.py:164`（`register_section`，看"往提示词里加一段"的机制）
- `docs/memory_architecture.md` 的第 2~3 章（记忆是怎么注入提示词的）

**关键概念**：
- **system prompt**：给模型看的"说明书"，说明身份、规则、记忆、项目背景。
- **section registry**：一个"提示词插件系统"——谁想往提示词里加内容，就注册一个 section（比如记忆注册了 `memory` section）。这样加功能不用改主代码。
- **记忆的本质**：就是一堆 Markdown 文件 + 每次对话时挑几条塞进提示词。不是数据库，没有魔法。

**练习**：打开 `memory_architecture.md` 第 0 章"记忆相关包一览"，对照文件清单数一数有哪些包。

---

## 5. 阶段五：理解扩展机制——不改上游怎么加功能（1~2 天）

**目标**：理解这个项目最核心的设计思想：**在不动上游代码的前提下扩展功能**。

**必读**：
- `clawcodex_ext/__init__.py:97`（`ensure_eager_extensions_installed()`，看启动时注册了哪几个扩展）
- `clawcodex_ext/memdir/__init__.py`（惰性门面模式：`__getattr__` + 符号表）
- `clawcodex_ext/init.py` 的 docstring（讲了启动管线）

**关键概念**：
- **decoupling mandate（解耦原则）**：不许直接改 `src/` 和 `extensions/` 的源码。要改行为，用三种方式之一：往 registry 注册、挂 hook、monkey-patch。
- **惰性导入（facade）**：import 包时不真正加载全部内容，用到哪个符号才加载哪个。好处是启动快。
- **feature gate**：功能开关。`init()` 注册一堆开关，代码里查"这个功能开没开"再决定行为。

**练习**：在 `clawcodex_ext/memdir/__init__.py` 里找一个符号（比如 `sanitize_path`），顺着 `_SYMBOLS_BY_MODULE` 表找到它实际在哪个文件。

---

## 6. 阶段六：选一个子系统深入（2~3 天）

**推荐选记忆系统**（因为已有完整文档，学起来最顺）：

- 必读：`docs/memory_architecture.md` 全部
- 然后读实现：`clawcodex_ext/memdir/`（核心）、`clawcodex_ext/dreaming/`（后台巩固）、`clawcodex_ext/away_summary/`（会话摘要）

想换个口味也可以：会话压缩（`clawcodex_ext/compact_service/`）、命令系统（`clawcodex_ext/command_system/`）、团队记忆（`extensions/agents/`）。

**练习**：给记忆文件加一条自己的记忆，下次对话看它有没有被召回（`/remember` 可以查看记忆层）。

---

## 7. 阶段七：动手改代码（3~5 天）

从易到难，选做：

1. **改提示词**：找一个 section 的提示词文案，改一句，跑起来看效果。
2. **注册一个命令**：仿照 `clawcodex_ext/skills/bundled/remember.py`，写一个 `/hello` 命令。
3. **写一个简单工具**：仿照 tool_system 里现有工具，加一个"返回当前时间"的工具。
4. **写测试**：仿照 `tests/` 里同类测试，给上面的改动补一个测试。

跑测试的方法（AGENTS.md 里写的）：

```bash
cd clawcodex-ascend
uv run pytest -m "not integration"          # 全部单元测试
uv run pytest tests/xxx.py -v               # 单个文件
```

---

## 8. 学习顺序总表

| 阶段 | 内容 | 时间 | 产出 |
|------|------|------|------|
| 1 | 跑起来 | 半天 | 亲眼看到 agent 调用工具 |
| 2 | 入口链 | 1 天 | 能说出命令到输出的每跳文件 |
| 3 | query() 主循环 | 2~3 天 | 能解释"调模型→执行工具→再调模型" |
| 4 | 上下文组装 | 1~2 天 | 知道 system prompt 怎么拼、记忆怎么注入 |
| 5 | 扩展机制 | 1~2 天 | 知道"不改上游加功能"的三种方式 |
| 6 | 一个子系统深入 | 2~3 天 | 精读记忆系统全部代码 |
| 7 | 动手改代码 | 3~5 天 | 至少完成改提示词 + 注册一个命令 |

总用时：约 2~3 周（每天 2~3 小时）。

---

## 9. 常见问题（先知道，少踩坑）

| 问题 | 答案 |
|------|------|
| `src/` 目录为什么是空的/找不到 | 上游代码安装时才拉取（`install.sh`），本地代码都在 `clawcodex_ext/` |
| 这个文件怎么几千行 | 只读关键函数（文档里给了行号），别从头读 |
| 启动为什么慢 | 第一次导入要加载很多模块，正常 |
| `CLAWCODEX_DEBUG=1` 是什么 | 打开诊断日志，能看到每轮调用和工具执行细节，调试神器 |
| 改代码后怎么验证 | `uv run pytest -m "not integration"` |
| 哪些可以先不学 | TUI 界面细节、cron 调度、multimodel、遥测、团队记忆细枝末节——用到再看 |

## 10. 术语小词典（遇到就查）

| 术语 | 一句话解释 |
|------|------------|
| entry point | 打包时声明的"命令 → 函数"映射，装完就有命令可用 |
| headless | 无界面模式：给参数、出结果、退出 |
| system prompt | 给模型看的系统说明（身份、规则、记忆、项目背景） |
| turn | 一轮"调模型→执行工具"的循环 |
| tool / 工具 | agent 的"手"：读文件、跑命令、改代码等能力 |
| section registry | 提示词插件系统：注册一段内容，它就会进提示词 |
| facade / 惰性导入 | 用到才加载，加快启动 |
| feature gate | 功能开关，控制扩展是否生效 |
| hook | 在流程特定时机插入的扩展点（如调模型前、执行工具后） |
| decoupling mandate | 项目规矩：不直接改上游源码，用注册/hook/补丁扩展 |
| transcript | 会话记录，每次对话存成文件，支持 `--resume` 恢复 |
