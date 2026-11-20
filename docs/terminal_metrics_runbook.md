# 终端指标运行手册

本运行手册说明如何解读 CLI 发出的交互式终端分析数据。

## 事件分组

- 交互式终端行为：`terminal_actions_planned`、`terminal_actions_executed`、`terminal_turn_summarized`
- Agent 循环可靠性：`react_turn_completed`、`$ai_generation`
- 网关可靠性：`gateway_turn_started`、`gateway_turn_completed`、`gateway_turn_failed`

## 核心 KPI

- `terminal_action_execution_success_rate`：确定性操作执行成功率
- `terminal_fallback_rate`：需要 LLM 回退的轮次占比

## 运维指引

- `terminal_fallback_rate` 高而 `planned_count` 低，说明确定性操作覆盖不足；应先改进操作识别器，再调整 LLM 提示词。
- `planned_count` 高但执行成功率低，说明命令执行可靠性有问题（shell 失败、依赖缺失、超时阈值）。

## 数据契约的权威来源

- 事件枚举：`infrastructure/analytics/events.py`
- 采集辅助函数与 KPI 查询规范：`infrastructure/analytics/capture.py`
- 事件属性构建器：`infrastructure/analytics/event_properties.py`
- 提供方类型约束与类型转换：`infrastructure/analytics/provider.py`