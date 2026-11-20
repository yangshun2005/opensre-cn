---
name: connecting-slack
description: >-
  将 OpenSRE 连接到 Slack，并演示如何通过频道提及或私信分派 DevOps 事务。用 cli_exec 验证；
  若 Slack 未配置，通过 slash_invoke 排队 `/integrations setup slack`（向导需要完整终端）。
  用于启动演示选项"将 OpenSRE 连接到 Slack 并为团队分派 DevOps 事务"。此流程中绝不向 Slack 发帖、回复或发送消息。多步骤；执行前先加载。
getting_started: 连接 Slack
demo_order: 4
metadata:
  owner: Vincent
  last_changed_by: Jan
  last_changed_at: 2026-10-04
  usecases:
  - 适用于为团队连接 OpenSRE 的 Slack 工作区管理员。
  - 适用于验证现有 Slack 连接并学习频道或私信分派的用户。
  requires:
  - 有权将 OpenSRE 机器人添加到目标 Slack 工作区。
  - cli_exec 和 slash_invoke 工具。
  - 需要配置 Slack 时，需有交互式终端运行 /integrations setup slack。
  version: '1.4'
---

# Slack 分派

验证或配置 Slack，然后说明团队如何通过频道提及或私信分派 DevOps 事务。

## 工作流规则

- 绝不调用 Slack 发送/回复/表情回应工具。
- 绝不臆造 Slack 已连接；只信任 `cli_exec` 的验证结果。
- 绝不用 `cli_exec` 调用 `integrations setup slack`。该向导是交互式的，`cli_exec` 会拒绝执行。
- 配置完成（或验证成功）后，用两句话说明如何分派事务：在 OpenSRE 可见的频道中提及它，或给它发私信。

## 计划

用 `update_plan` 工具跟踪进度，不要用标题或正文：

- 进入时，在首次工作流工具调用（`cli_exec`）的同一响应中调用 `update_plan`，
  原样传入以下步骤，第一步设为 `in_progress`，并附一行
  `explanation`（这不是诊断；不要假设表）：
  `Check Slack` / `Set up if needed` / `Explain the hand-off`。每次调用都将
  `Check Slack` 标记为 `verifies: true`：纯文本的最后一步依赖此检查来收尾。
- 某步骤的工具结果返回后，在下一步骤单次工具调用的同一响应中调用 `update_plan`，
  将该步骤标记为 `completed`，下一步骤标记为 `in_progress`。若 Slack 已连接，
  直接将 `Set up if needed` 标记为 completed，无需新的工具调用，不要中途丢弃该步骤。
- 不要叙述计划或在正文中重复步骤名；shell 会渲染检查清单。

## 工作流

### 1. 检查 Slack

调用 `cli_exec`，载荷为 `integrations verify slack`。

### 2. 按需配置

若 Slack 未配置，调用 `slash_invoke`，命令为
`/integrations`，参数为 `["setup", "slack"]`，然后停止。shell 会在
下一个提示符排队该向导，使其获得独占 stdin。若 Slack 已连接，说明情况并跳过配置。

### 3. 说明分派方式

两句话：在频道中提及 OpenSRE 或给它发私信；本次演示不要发布任何内容。然后停止。