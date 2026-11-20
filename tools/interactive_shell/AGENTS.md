# tools/interactive_shell/ 包规则

这些说明适用于 `tools/interactive_shell/` 及其所有子目录。仓库根目录的 `AGENTS.md` 仍然适用。

## 用途

本包承载 agent harness 在交互式 shell 回合中调用的**动作工具实现**——即 `core/agent_harness/tools/action_tools.py` 中列出的动作工具背后的具体 `run` 函数体：

- `actions/` — 动作工具本身（`shell_run`、`cli_exec`、`slash_invoke`、`code_implement`、`session_goal_set`、`session_goal_complete`、`llm_set_provider`、`task_cancel` 等；完整集合见 `action_names.py`）。
- `shell/` — shell 命令规范化、执行策略，以及 `actions/shell.py` 背后的 `run_shell_command` 运行器。
- `implementation/` — `/implement`（Claude Code）启动器。
- `contracts` — 在早期导入装配阶段由 `command_registry.slash_catalog` 导入；工具子模块为何必须显式导入而不能在此处急切导入，见 `__init__.py` 的文档字符串（避免循环导入）。

按照仓库根目录的 `AGENTS.md`，`tools/` 拥有所有 `@tool(...)` 函数以及 `RegisteredTool`/`BaseTool` 类；本包是这一所有权中交互式 shell 的部分。

## 子进程运行器解耦（T-03）

子进程运行器必须保持分为两层：

1. **纯执行**，位于 `tools/interactive_shell/` 下——启动、通信、规划、结构化结果。不使用 Rich，不使用 `surfaces.interactive_shell.ui`，不使用 `execution_confirm`。
2. **REPL 展示**，位于 `surfaces/interactive_shell/runtime/subprocess_runner/` 下——`ReplSubprocessPresenter` 实现 `SubprocessPresenter`，并通过 `action_turn.py` 中的 `ActionToolScope.subprocess_presenter` 注入。

仅依赖标准库的共享辅助函数位于 `tools/interactive_shell/subprocess.py`。Rich 流中继保留在 `surfaces/.../subprocess_runner/task_streaming.py`。

**不要重新引入** `surfaces.interactive_shell.ui` 或 `surfaces.interactive_shell.runtime.subprocess_runner` 的导入到以下文件：

- `shell/runner.py`
- `implementation/claude_code_executor.py`
- `actions/cli_command.py`

由 `tests/tools/interactive_shell/test_import_boundaries.py` 强制执行。

## 依赖方向

```text
surfaces/interactive_shell (ui, dispatch, ReplSubprocessPresenter)
  -> tools/interactive_shell (action-tool implementations)
    -> core/agent_harness (session types, ports)
```

`tools/interactive_shell/` **可以**依赖：

- `core.agent_harness.session` 运行时类型（`Session`、`TaskKind`、`TaskRecord`、`TaskStatus`），用于会话/任务记录。
- `integrations/`，当动作工具封装集成客户端时（例如 `implementation/claude_code_executor.py` 中的 Claude Code）。

它**不得**：

- 被 `core/agent_harness/` 导入（由导入边界测试强制执行）。
- 在子进程运行器中导入 `surfaces.interactive_shell.*`（见上方 T-03 列表）。其他动作工具（`slash` 等）仍可暂时访问 surface——那是单独的 T-4 债务。
- 在 `__init__.py` 中增加急切子模块导入（保持那里记录的显式导入纪律；多个工具模块会反向导入 `command_registry`，因此此处的急切导入会重新引入循环导入）。

## 剩余的 surface 耦合（T-4 债务，不在 T-03 范围内）

若干非子进程动作工具仍直接导入 `surfaces.interactive_shell.ui` 或 `command_registry`（`slash.py` 等）。没有端口之前，不要在那里新增 surface 导入；子进程运行器是今后的参考模式。