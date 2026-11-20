---
name: Merge conflicts
description: 解决打开的 Pull Request 上的合并冲突。
cron: "28 * * * *"
mode: agent
---

解决打开的 Pull Request 上的合并冲突。

1. 列出本仓库分支中带有合并冲突的打开的非草稿 Pull Request。
2. 跳过最近 10 分钟内有推送的 Pull Request。
3. 选择一个，将默认分支合并到其分支，并在解决每个冲突时保留作者的意图。
4. 在本地运行相关测试并推送合并提交。
5. 如果某个冲突需要产品决策，在 Pull Request 上评论说明并继续处理下一个。
6. 绝不强制推送或合并，回复 Pull Request 链接及你所解决的内容。