# 交互式 Shell 包

这些说明适用于 `interactive_shell/` 及其所有子目录。仓库根目录的 `AGENTS.md` 仍然适用。

## 用途

`interactive_shell/` 负责交互式 OpenSRE 终端界面：REPL 循环、斜杠命令界面、本地告警接入、Shell 执行，以及 Rich / prompt-toolkit UI。可复用的 agent 会话状态、提示历史与提示构建位于 `core.agent_harness` 下。

设计面向可能正处于事故处理中的终端用户：行为应当可预测、可中断、可解释，且默认安全。

## 包结构与职责归属

| 区域 | 负责 | 不应包含 |
| --- | --- | --- |
| `main.py` | 启动 REPL 的进程/引导边界 | 每轮调度/运行时逻辑 |
| `controller.py` | 顶层 REPL 接线、告警监听器生命周期、提示循环、后台 worker 与关闭流程 | 特定功能的业务逻辑或仅用于兼容的转发 |
| `runtime/core/turn_accounting.py` | Shell 轮次核算（`ShellTurnAccounting`），用于分析、记录器增强、历史与意图标记 | 轮次流转控制（由 `core.agent_harness` 负责）或工具调用轮次执行 |
| `command_registry/` | 斜杠命令定义、参数校验、命令分发 | 更适合放在 services/runtime 模块中的长时间运行实现细节 |
| `runtime/` | 后台任务 worker、生命周期/`ReplState`、运行时上下文组装、语义化 Shell 轮次执行，以及核心 harness 适配器 | 提示文本、可复用会话持久化或兼容垫片 |
| `tools/interactive_shell/shell/` | Shell 命令规范化、Shell 执行策略、子进程执行，以及 `run_shell_command` 运行器（与 `tools/interactive_shell/actions/shell.py` 中的 `shell_run` 工具相邻） | 斜杠命令执行 |
| `config/` | 交互式 Shell 配置加载与工具目录元数据 | 与 REPL 无关的全局应用配置 |
| `ui/` | Rich/prompt-toolkit 渲染、主题、菜单、流式输出，以及诸如 `incoming_alerts.py` 的领域视图（接收器/队列/监听器生命周期位于 `core.domain.alerts.inbox`） | 业务逻辑或网络调用 |

当变更跨越这些边界时，优先在归属区域提取小型辅助函数，而不是在调用方添加更多逻辑。

## 横切规则

- 将所有外部输入视为不可信：用户提示文本、斜杠命令参数、告警载荷、读入提示的文件、历史、子进程输出、模型输出与集成元数据。
- 保持交互路径响应性：长时间运行的工作必须可取消、有超时限制、移出输入路径，或以清晰进度呈现。
- 保持导入时轻量——不要在模块导入时启动线程、调用 LLM、读取大文件或访问网络。
- 斜杠命令名称、标志、输出标签、提示、响应体与错误措辞属于面向用户的 API——应将其视为有意设计，而非附带产物。
- 避免新增模块级可变全局变量；若全局协调不可避免，提供确定性的重置/清理钩子以便测试隔离。

## 斜杠命令

- 在相关的 `command_registry/*` 模块中以 `SlashCommand` 条目添加命令。保持处理函数精简：解析参数、调用聚焦的辅助函数、渲染结果。
- **REPL + CLI 对等（必需）：** `SLASH_COMMANDS` 中的每个命令都必须在 `command_registry/slash_catalog.py` 中有匹配的 `MCP_BY_COMMAND` 条目。该目录为 LLM 规划器（`slash_invoke`）、规划器工具规格与紧凑帮助文本提供数据。缺少它会导致 CI 失败（`test_slash_catalog_covers_all_registered_commands`）。
  - **新增仅 REPL 的斜杠命令：** 在归属的 `command_registry/*` 模块中添加 `SlashCommand`，**并**在 `slash_catalog.py` 中添加 `_mcp(...)`（保持 `MCP_BY_COMMAND` 中的键按字母顺序排序）。
  - **新增具有 REPL 对等的 CLI：** 在 `surfaces/cli/commands/` 下添加 Click 命令，在 `command_registry/cli_parity.py` 中注册 `SlashCommand`（子进程调用 `opensre …`），**并**在 `slash_catalog.py` 中添加 `MCP_BY_COMMAND`，其 `llm_description`、`use_cases` 与 `anti_examples` 需与命令的 `usage` 元组对齐。
  - **推送前验证：**
    `uv run python -m pytest tests/interactive_shell/command_registry/test_slash_catalog.py -q`
- 使用 `validate_args` 进行廉价的策略前校验，使错误参数不会触发确认或副作用。
- 通过中央分发与执行策略辅助函数发送命令执行。新命令不得绕过 `execution_policy.py`。
- **Alpha 全允许执行策略（当前行为）：** REPL 默认使用 **`/auto high`**——无工具确认。`execution_policy.py` 将每个操作解析为 `allow`；仅当 `/auto` 低于 High 时，`apply_auto_level` 才提升为 `ask`。**没有 Shell 命令安全策略**：只读/变更/受限分类与 `deny` 下限已被移除（`shell_policy.py` 已删除；解析/策略/执行位于 `tools/shell/` 下）。变更命令（`rm`/`mv`/`docker`）、`restricted` 命令（`sudo`、`systemctl`、`kill`、`dd` 等）、Shell 操作符（`| && ; > <`）与命令替换在批准后（或在 High 级别立即）运行。`!` 前缀被尊重但可选。唯一仍被拒绝的 Shell 输入是真正为空的输入（裸 `!` 或空白）。在 `docs/getting-started/interactive-shell-commands.mdx`（`/auto`）与 `docs/interactive-shell-action-policy.md` 中记录级别与 `/trust` 交互。在 alpha 期间**不要**重新添加 Shell 允许列表或拒绝下限——在 `execution_policy.py` 中门控更严格的策略（`ask` 判定、确认 UX、`trust_mode` 与 `/auto` 是钩子），而非通过规划器阶段的拒绝。
- 默认允许下的非 TTY 行为：操作不再在非交互式 stdin 上失败关闭（没有可确认的内容）。失败关闭路径仅在判定明确为 `ask` 时适用，而默认策略不会发出该判定。
- **CPR / 独占 stdin 注册（输出表格的命令必需）：** 在 `patch_stdout(raw=True)` 下，REPL 在下一个 `prompt_async()` 的同时并发运行分发。当命令输出 Rich 表格时，prompt_toolkit 会在中途重绘提示，发送 `ESC[6n` DSR 查询；终端的 CPR 响应（`ESC[<row>;<col>R`）会作为字面按键到达传入的提示缓冲区，导致出现类似 `^[[60;1R` 的垃圾字符。
  **任何调用 `print_repl_table` 的命令（直接调用或通过 `render_table` / `render_integrations_table` / `render_models_table` / 等）都必须添加到 `runtime/input_policy.py` 中的 `_EXCLUSIVE_STDIN_MENU_COMMANDS`。** 这使主循环调用 `await state.queue.join()`，阻塞下一个提示直到分发完成，且两个排空周期在下一个 `prompt_async()` 开始前清理陈旧的 CPR 字节。
  - **如何检查：** 添加命令后，在 REPL 中运行它，并在下一个提示中输入几个字符。如果没有出现 `^[[…R` 垃圾字符，则注册正确。
  - **Agent 选择的交互式命令：** `_EXCLUSIVE_STDIN_MENU_COMMANDS` 仅为 `_literal_slash_command_text` 识别的字面 `/slash` 命令文本保留 stdin。当自由文本如 "remove github" 被动作 agent 解析为内联选择器命令（`/integrations remove`、`/integrations setup`、`/mcp connect`、`/mcp disconnect`，或裸 `/integrations` / `/mcp` 菜单）时，循环尚未保留 stdin，因此 `tools/interactive_shell/actions/slash.py` 不得内联运行选择器。它通过 `set_auto_command(session, ...)`（`core.agent_harness.spi.session_state`）延迟，将命令作为字面命令文本重新提交，以便循环在 agent 路径运行前保留独占 stdin。工具结果在 `QUEUED_COMMAND_KEY` 下命名该命令，这会结束动作轮次，使模型无法在命令运行前重试。动作 agent 可发出的新原始 stdin 选择器/向导命令必须添加到 `tools/interactive_shell/actions/slash.py` 中的 `_INTERACTIVE_PICKER_MENUS` / `_INTERACTIVE_PICKER_SUBCOMMANDS`。
  - **Agent 选择的列表：** 通过 REPL 适配器使用 `runtime/exclusive_output.with_exclusive_output`，在渲染时暂停提示，保持阻塞读取远离其事件循环，并将 ID 返回到同一 agent 轮次。将列表作为选择器排队会结束该轮次并丢失后续操作。
  - **设置后恢复：** 技能内延迟的 `/integrations setup <service>` 会暂存该轮次的消息，技能的前置条件门控也是如此。`command_registry/setup_resume.py`（在向导结束时以及设置菜单的 "continue" 行调用）重新运行前置条件的检查——`run_cli_command` 对每次交互式运行都报告成功，因此永远不要信任它——并重放暂存的消息一次；它从不替换排队的自动提交。

## 动作选择与执行

- **硬边界：** 不要为自然语言添加正则/关键词/模糊意图路由，也不要添加从非 `/` 前缀散文到动作的任何确定性映射（例如 "show integrations" -> `/integrations`）。工程师曾因重新引入与动作 agent 竞争的意图启发式而被解雇。
  - **获准的例外**（显式命令绕过，而非意图推断；自由形式散文仍由 LLM 选择）：
    1. 用户键入的字面 `/slash`——`core/agent_harness/turns/action_driver.py` 中的静态 `slash_invoke`（在动作 agent LLM 宕机时保持 `/login` 等可用）。参见 `docs/interactive-shell-action-policy.md`（"Deterministic literal-`/slash` dispatch"）。
    2. 本会话中待处理提议武装后的结构化 Want-me-to *accept*——计划 yes 展开为 `/cron …`。这是字面斜杠文本，仅使用路径 1。
- **无规划阶段失败关闭保护（v0.1 决策）。** 第二阶段动作 agent 从不拒绝轮次。由于每个终端动作都是只读的，未匹配/模糊/闲聊的子句不是安全风险——agent 执行它能映射的子句，其余落入对话助手。我们移除了 `denied` 决策路径、`mark_unhandled` 规划器工具、`UNHANDLED:` 约定与 "I couldn't safely decide actions" 消息，因为它们导致频繁的误拒绝（例如嵌入引用式列表指令的对话问题），且没有安全收益。细节与理由见 `docs/interactive-shell-action-policy.md`。如果将来引入变更动作，请用执行阶段确认策略（`tools/interactive_shell/shared/execution_policy.py`）门控，而非规划器阶段拒绝。
- 将确定性命令检测保留在 `orchestration/` 中，仅用于终端 UI 策略；使用动作 agent 进行斜杠/工具动作选择。
- 将不确定性发送到安全表面：帮助/聊天或澄清，而非直接变更或 Shell 执行。
- LLM 生成的文本绝不能直接执行。将提议的动作转换为显式计划动作，在适当时向用户展示，然后通过 `orchestration/` 与策略门控执行。
- 添加新动作类型时，测试允许、拒绝与需要确认的路径。

## LLM 提示、接地与引用

- 对包含在模型上下文中的文档、源码块、历史、观察、告警文本与命令输出强制执行大小上限。
- 将程序性/帮助答案接地于维护的引用（`docs/`、CLI 帮助、AGENTS 文件、源码片段）。如果引用不支持答案，应说明而非编造步骤。
- 不要在提示中包含机密——编辑或省略令牌、认证头、环境值、本地凭据与原始集成配置。
- 在 `core.agent_harness.prompts` 中保持提示规则可复用，使聊天/帮助/动作表面使用一致的术语与格式。

## 终端 UI 与渲染

- 在将用户控制的内容传递给 Rich 标记前进行转义（`rich.markup.escape`）：告警、命令输出、文件路径、集成名称、模型/提供商标签、错误、文档片段，以及尚未有意渲染为 Markdown 的模型文本。
- 使用 `infrastructure/terminal/theme.py` 中的语义令牌。不要在 `infrastructure/terminal/theme.py` 之外引入原始十六进制颜色、Rich 命名颜色或原始 ANSI 转义，除非狭窄的 prompt-toolkit 兼容路径需要。
- 任何原始终端模式代码必须检查 TTY 支持并在 `finally` 中恢复终端状态。
- 混合 `prompt_toolkit.patch_stdout`、Rich 实时渲染与后台输出时要小心——参见上文斜杠命令下的 CPR 陷阱。优先使用仅追加、段落缓冲或节流渲染路径，以免损坏可编辑提示。

## Shell、子进程与本地系统影响

- Shell 执行变更属于 `tools/interactive_shell/shell/`。将命令语法传递给宿主 Shell，而非复制其解析器，并保留输入规范化、超时、编辑与策略行为。
- 将子进程输出视为不可信显示文本；在 Rich 标记前转义，并限制保留或发送到提示的内容。
- 对子进程使用显式超时与清晰的取消行为。避免可能无限期挂起 REPL 的等待。
- 保留的 CLI 子进程必须在交互式宿主退出后存活，而不阻塞宿主或丢失可写输出管道。使用独立排空，带上限的前台捕获与超时后丢弃；保留无头等待。验证子进程等待时宿主实际退出，然后写入超出管道容量并完成。

## 状态、历史、配置与后台工作

- 后台线程/任务/监听器必须有确定性关闭。测试应在 fixture 或 `finally` 块中停止句柄与 worker。
- 用锁或单一所有者纪律保护共享队列与可变会话数据。避免围绕队列、取消标志、当前任务与监听器句柄的检查后行动竞态。
- 历史应避免存储机密或过多载荷。一致地应用截断与隐私策略。

## 外部输入与本地监听器安全

- 类似网络的本地表面，如 `core.domain.alerts.inbox`（由 REPL 入口点启动），必须在阻塞读取或昂贵解析前验证廉价的请求元数据。
- 永远不要执行无界的请求体读取。特别是告警 POST，先验证 `Content-Length`，然后才读取有界请求体：
  - 非数字 `Content-Length` 值会使 `int(...)` 抛出 `ValueError`；捕获此错误并返回 `400`。
  - 负长度必须返回 `400`；`rfile.read(-1)` 会读取直到 EOF 而非零字节，这可能使单线程处理器停滞。
  - 过大的正长度必须返回 `413`，而不尝试读取声明的请求体。
- 通过在返回 `401` 前仅排空有界请求体，为真实 POST 请求体保留干净的未授权响应；这避免了某些平台上关闭时未读数据导致的重置，同时不允许过大的预认证读取。
- 保持非环回监听器绑定受令牌保护。使用恒定时间令牌比较，且永远不要记录 bearer 令牌、原始认证头或完整告警载荷。

## 测试期望

- 将测试放在 `tests/interactive_shell/` 下，在有用时镜像包区域（`orchestration/`、`ui/` 等）。永远不要在源码包下添加测试。
- 对于聚焦的变更，运行最接近的测试，例如：
  - `uv run python -m pytest tests/core/domain/alerts/test_inbox.py`
  - `uv run python -m pytest tests/interactive_shell/<area>/`
  - `uv run python -m pytest tests/interactive_shell/`
- 为易出事故的边缘添加回归测试：平台套接字行为、畸形输入、非 TTY 执行、取消、策略拒绝、提示大小上限、Rich 转义与后台清理。
- 优先使用确定性测试而非重度依赖 sleep 的测试。使用假分类器、假会话、假控制台、monkeypatch 的子进程与小 fixture。