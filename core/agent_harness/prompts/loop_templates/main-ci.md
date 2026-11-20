---
name: Main CI
description: 修复默认分支上失败的 CI。
cron: "58 * * * *"
mode: agent
---

修复默认分支上失败的 CI。

1. 如果已有打开的 Pull Request 修复了默认分支，用该 Pull Request 的 pr_number 调用 fix_github_pr_ci；其检查通过后，该修复正等待人工合并。
2. 否则将 branch 设为默认分支调用 fix_github_pr_ci；若最新检查已通过，它会报告无需修复。
3. 如果它推送了新的修复分支，从该分支向默认分支发起 Pull Request。
4. 绝不合并，并回复 Pull Request 链接、根本原因，以及是否等待合并。