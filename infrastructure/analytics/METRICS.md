# 产品分析指标

跨商店安装键是 `~/.opensre/anonymous_id` 中的随机 `analytics_id`。个人 bearer 会附加服务器解析的 Clerk 用户和组织。silo bearer 会附加经过认证的组织断言，但不含 Clerk 用户；聊天参与者 ID 仍作为事件属性，不得计为人数。

当前 ClickHouse 视图仅实现安装漏斗和账户/会话头条指标。具体而言，`analytics_core_metrics.daily_active_users` 是已认证的 webapp 活动，而非下文定义的个人产品 DAU。其余指标需要查询 `analytics_product_events`。

## 获取与激活

合格的安装观测排除合成/测试身份，以及保留运行时历史中已确认或已报告自动化的所有身份。剩余身份来源未知：这些是观测到的安装，而非实测的人类获取分母。其转化率须与已验证账户指标分开报告，并同时展示自动化计数。

| 指标 | 事件与计算 |
| --- | --- |
| 安装数 | 具有 `install_detected` 的去重合格 `analytics_id` 值。每个安装只计一次；当某安装同时有原始事件和恢复事件（`install_detection_reason=unverified_marker`，事件 ID 后缀 `:delivery-v1`）时，取最早的 `occurred_at` 作为安装日期，并从该最早事件取 `install_source`/`install_channel`/`install_version`。若恢复事件是某安装唯一的 `install_detected`，则该安装有效，其安装日期仅为下界，来源维度可能为运行时回退值。 |
| 安装到注册转化 | 在 `install_detected` 与首次认证关联之间创建 Clerk 注册的安装数，除以安装数。 |
| 已认证安装 | 具有后续个人 bearer 事件的去重合格安装观测。`account_authenticated` 是正常的首次关联，但该指标不依赖此单一事件是否送达。 |
| 登录门转化 | 具有 `sign_in_selected` 的去重合格安装观测，以及具有 `stay_signed_out_selected` 的去重安装，分别除以具有 `sign_in_prompted` 的去重安装。按 `method` 切分 `stay_signed_out_selected`，以区分显式退出选项（`menu`）与关闭菜单（`dismissed`）。无后续 `account_authenticated` 的 `sign_in_selected` 表示放弃或失败的浏览器登录。 |
| GitHub 门转化 | 具有 `github_gate_rechecked` 且 `connected=true` 的去重合格安装观测，以及具有 `github_gate_exited` 的去重安装，分别除以具有 `github_gate_prompted` 的去重安装。按 `reason_code` 切分曝光和失败复查：成员在应用中连接 GitHub 后仍持续失败的复查，指向 CLI 对连接的视图，而非成员本身。曝光后两个事件均无，表示终端已关闭或捕获丢失，而非决策。 |
| 引导转化 | 完成引导的去重合格安装观测，以及引导失败的去重安装，分别除以已开始的去重安装。 |
| 个人激活 | 服务器解析的用户，其关联安装完成引导，且后续记录到已完成、已捕获的 AI 响应，并具有观测到的 LLM 尝试且无错误。旧版事件要求真实模型/提供商且输出非合成。 |
| 网关激活 | 具有已认证 `gateway_turn_completed` 且 `answered=true` 的组织。须与个人激活分开，因为网关参与者不是 Clerk 用户。 |

## 使用与留存

| 指标 | 事件与计算 |
| --- | --- |
| 个人 DAU / WAU / MAU | 在请求窗口内具有个人 bearer `cli_command_opensre…`（历史上为 `cli_invoked`）或 `$ai_generation` 的去重服务器解析用户。 |
| 组织 DAU / WAU / MAU | 在请求窗口内具有网关活动的去重已认证组织；与个人用户分开报告。 |
| D1 / D7 / D30 留存 | 在目标日或窗口内有另一合格个人事件的个人激活用户。组织留存是独立的网关指标。 |
| 功能采用 | 按 CLI/AI 功能统计个人用户，按网关界面统计组织；切勿混合两种身份粒度。 |
| 集成采用 | 完成或验证设置的去重已认证组织，按服务分组。个人事件携带服务器解析的组织；silo 事件携带 bearer 认证的运行时断言。 |

## 可靠性与质量

| 指标 | 事件与计算 |
| --- | --- |
| 网关回答率 | `gateway_turn_completed` 且 `answered=true`，除以所有已完成的网关轮次。 |
| 终端操作成功率 | `executed_success_count` 之和，除以 `executed_count` 之和。 |
| LLM 回退率 | `terminal_turn_summarized` 且 `fallback_to_llm=true`，除以所有已摘要轮次。 |
| Agent 可靠性 | 错误、取消和迭代上限的 ReAct 轮次，除以所有 `react_turn_completed` 事件。按 `loop_stop_reason` 切分迭代上限轮次，以区分预算耗尽与 `goal_unverified` 和 `stagnation_limit`。 |
| 工具调用成功率 | 已执行的 `agent_tool_call_completed` 事件中 `outcome=ok` 的数量，除以所有已执行工具调用。执行前拒绝结果单独切分，被阻止的调用按 `blocked_by` 切分。 |
| Ask User 响应 | `ask_user_prompt_answered` 除以选择器模式的 `ask_user_prompt_rendered`；`ask_user_prompt_dismissed` 和自定义答案占比单独报告。 |
| 计划任务可靠性 | 按任务类型和提供商统计已完成与失败的定时任务。 |
| 延迟 | 按界面、模型和提供商统计网关、ReAct 和 AI 生成时长的 p50/p95。 |

将已确认/已报告的自动化与来源未知的观测分开。非 CI 指标要求记录布尔值 `is_ci=false`；缺失标志保持未知。检测器否定结果不等于人工验证。匿名安装和引导计数仅具方向性，因为公共开源客户端无法对机器所有者保密签名密钥。可信用户指标使用个人 bearer 关联，切勿用网关参与者 ID 替代 Clerk 用户 ID。