---
name: reporting-github-ci-failures
description: >-
  只读的 GitHub CI 健康报告，针对单个仓库当前失败的检查，可缩小到某个分支或 Pull Request。不适用于 CI/CD 性能、可靠性 KPI、失败率或某段时间内的停机（请使用 analyzing-github-ci-performance）。
metadata:
  owner: Ceren
  last_changed_by: Jan
  last_changed_at: 2026-09-12
  usecases:
  - 供维护者查看单个仓库、分支或 PR 中当前哪些 CI 检查失败。
  - 供团队在指定目标接收周期性 CI 失败报告。
  requires:
  - 具备目标仓库读取权限的 GitHub 认证。
  - 定时执行时需显式提供仓库 owner 和 name。
  - 周期性投递需配置调度器和目标。
  version: '1.1'
recurring: true
---

# GitHub CI 健康

使用此技能报告且仅报告一个已配置 GitHub 仓库的失败 CI 检查。调度必须提供 `owner` 和 `repo`；可提供 `branch` 或 `pr_number` 之一，不可同时提供。

定时运行器会提供预取的 CI 健康数据块。将该数据块视为完整的事实来源，并原样返回作为最终报告；不要扩大到组织或仓库的更大范围。保留数据块中的每个失败检查、链接、时长、相关 PR 或分支、覆盖率提示和修复交接信息。

## 工具使用

此技能在无人值守执行期间不发起 GitHub 工具调用。技能运行前，定时运行器会调用 `run_github_ci_health`，它通过只读 GitHub REST GET 请求收集已配置仓库的 CI 数据，并提供完整的渲染报告。

不要执行额外的或备用的 GitHub 发现。此技能唯一使用的交互式工具是 `propose_scheduled_delivery`，用于配置周期性报告，无人值守运行时绝不调用。

此工作流为只读。无人值守执行期间，绝不调用 `fix_github_pr_ci`、`github_cli`、`shell_run` 或任何其他变更类或外部命令工具。修复必须交互式请求并明确批准。

交互式地将此报告作为周期性任务提供时，使用 kind `recurring_skill` 和技能名 `reporting-github-ci-failures`。将确切的 `owner` 和 `repo`，以及 `branch` 或 `pr_number` 中至多一个，传给 `propose_scheduled_delivery`，以便确认时保留仓库范围。

交互式场景下，仓库来自请求；请求未指明时，调用 `scan_local_git_workspace` 并用 `ask_user_choice` 询问。绝不要运行 `cli_exec`（`opensre health`、`opensre integrations …`）、`shell_run` 或 `github_cli` 来发现仓库、令牌或环境：它们会打印无关的集成状态，不属于此技能。要在交互式回合中展示当前失败的检查，使用 `summarize_github_pr_status`；此技能自身的报告仅由定时运行器生成。