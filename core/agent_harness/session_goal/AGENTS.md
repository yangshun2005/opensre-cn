# session_goal/ — `/goal` / SessionGoal 组件

跨多个 `chat` 轮次的主机级完成判定。**不是** ReAct 的 `Goal` / `turns/goal_review.py`。

## 叶子模块（按需导入）

| 模块 | 职责 |
|--------|------|
| `goal.py` | `SessionGoal`、状态/原因、附加/清除、原因推导 |
| `judge.py` | 廉价模型对对话记录的判定（已达成 / 尚未达成 / 不可能） |
| `validate.py` | 廉价模型校验新勾选的清单项 |
| `review_input.py` | 合格的工具观察结果，以及完整、限长的评审输入 |
| `evaluate.py` | 主机完成判定：``GOAL_REACHED`` 需有证据；``NOT_REACHED`` 继续 |
| `progress.py` | `SESSION_GOAL_PROGRESS_MARK` + 进度 / 状态行格式化 |
| `continuation.py` | 首轮与续接提示词（未运行工具时要求调用工具） |
| `persist.py` | 刷新 / 恢复载荷 |
| `run_until.py` | `run_until_session_goal` |

**不要**从 `goal.py` 导入 progress/continuation/persist/evaluate/judge/validate/run_until（避免 `py/cyclic-import`）。调用方应导入叶子模块，或从 `session_goal` 包的 `__init__` 导入精选名称。

## 边界

- 用户词汇仅 `/goal`（见 progress 常量）。
- 通过 `/goal set`、显式 `goal=` 或结构化交接附加——绝不按用户文本关键词路由。
- Shell 多步骤 ≠ 本组件（`prompts/action/multi_step_policy.py`）。
- L0/L1 证据仅在收尾时耦合（Want-me-to / CTA 抑制）。

设计 SoT（本地笔记）：`opensre-notes/session-goal-opensre-aug2026.html`。