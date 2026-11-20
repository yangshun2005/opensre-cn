# Agent 状态

`core/state/` 负责 OpenSRE 在 agent 轮次之间携带的、与提供方无关的对话状态。

跨轮次对话记录及其压缩辅助函数放在此包。这是**状态**——不是 REPL 会话状态、CLI 提示词接地，也不是通用 agent 运行时请求组装。

## 属于此处

- 可变的按会话对话存储（`MutableAgentState`）。
- 每个已完成轮次的结构化记录（`TurnEvidence`）：后续轮次重放的工具调用和有界结果。
- 对话记录窗口压缩辅助函数，以及历史开关和预算（`history_settings`）。

## 不属于此处

- 上下文裁剪、排序和预算逻辑；保留在 `core/context_budget.py`。
- LLM/工具调用循环和运行时请求契约；保留在 `core/` 下的同级运行时模块。
- 终端 UI、REPL 会话状态、提示词历史和斜杠命令；保留在 `surfaces/interactive_shell/`。
- 外部客户端、配置规范化和验证；保留在 `integrations/`。
- Agent 可调用的工具实现；保留在 `tools/`。
- 基础设施服务，如护栏、脱敏、认证、遥测、通知和沙箱；保留在 `infrastructure/`。

## 同时导出（临时）

- ``agent_state.py`` 中的 ``MutableAgentState``。现已精简为跨轮次对话记录（``messages``）、其轮次证据（``turn_evidence``）和 ``clear()``——按轮次的工具/提示词机制已移除。它仍留在此处是为了兼容历史导入路径；剩余迁移至 ``core/agent_harness/session/`` 的工作尚待完成（[#3685](https://github.com/Tracer-Cloud/opensre/issues/3685)）。

## 命名规则

此处的新名称应让状态边界显而易见。优先使用 `state`、`slice` 和 `snapshot` 等术语。避免在此添加通用的 `prompt`、`session`、`runtime` 或 `grounding` 模块；这些属于其所属的界面或运行时包。