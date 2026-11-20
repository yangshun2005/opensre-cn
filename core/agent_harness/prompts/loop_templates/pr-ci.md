---
name: PR CI
description: 修复开放 Pull Request 上失败的 CI 检查。
cron: "28 * * * *"
mode: agent
---

修复开放 Pull Request 上失败的 CI 检查。

1. 对本仓库调用 summarize_github_pr_status；若没有可修复的 Pull Request 存在失败检查或合并冲突，它会报告无需修复。
2. 否则选择一个此类 Pull Request，并用其 pr_number 调用 fix_github_pr_ci。
3. 绝不强制推送或合并，回复 Pull Request 链接及所做更改。