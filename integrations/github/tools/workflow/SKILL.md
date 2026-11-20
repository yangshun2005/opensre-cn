---
name: tracking-github-work-status
description: 使用 GitHub 工作流工具读取工作状态、起草报告、汇总后续事项，并仅执行已批准的 Issue 变更。
tools:
  - list_github_work_items
  - summarize_github_pr_status
  - list_github_security_alerts
  - generate_work_status_report
  - summarize_community_followups
  - propose_github_issue_mutation_from_slack
  - execute_github_issue_mutation
---

# GitHub 工作流

当用户询问 GitHub 工程状态、PR 就绪情况、社区后续事项，或将明确的 Slack 请求转为 GitHub Issue 时，使用此工作流。

1. 先读取再报告。优先使用只读 GitHub 工具；读取缺失或失败应视为状态不完整，而非证明没有阻塞项。
2. 基于已知数据报告或汇总。用 `generate_work_status_report` 生成可直接发到 Slack 的状态报告，用 `summarize_community_followups` 汇总未答复的贡献者问题或议程事项。
3. 先提议变更再执行。仅对明确来自 Slack 的创建、更新或关闭请求，使用 `propose_github_issue_mutation_from_slack`。
4. 仅在批准后执行。`execute_github_issue_mutation` 是唯一会修改 GitHub 的工作流工具。它绝不是调查操作，必须仅在聊天中、经批准、由提议驱动。
5. 对于临时 `gh` 操作（此 Slack 提议流程之外的灵活 Issue/PR/API 操作），使用 `github_cli`——绝不用 `shell_run` 直接跑 `gh`。