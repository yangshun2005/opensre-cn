---
name: 代码扫描
description: 修复未解决的代码扫描质量告警，绝不处理安全类告警。
cron: "58 * * * *"
mode: agent
---

修复未解决的代码扫描质量告警。

1. 调用 fix_github_security_alert，参数为 alert_type code_scanning、quality_only true 和 open_pr true。
2. 它会挑选一个尚无未关闭 Pull Request 修复的未解决告警；若没有剩余告警，则报告无需修复。
3. 绝不合并，并回复 Pull Request 链接及其修复的告警。