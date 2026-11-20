---
name: operating-github-cli
description: >
  操作代理的默认 GitHub 技能。任何 GitHub 请求都用 github_cli——创建/列出/查看/关闭 Issue 和 PR、指派、标签、仓库、发布版本、检查、github.com/owner/repo URL 或 gh api。优先于 shell_run/!gh。在当前代理轮次中用 github_cli 执行。
tools:
  - github_cli
---

# operating-github-cli

OpenSRE 的已认证 `gh`。可读可写——无审批关卡。优先于 `shell_run` / `!gh`。在 `gh` 之后传 `args`；可选 `repo` 为 `owner/name` → `-R`。禁止的顶层命令：`auth`、`extension`、`workflow`、`run`、`secret`、`codespace`、`ssh-key`、`gpg-key`、`config`。

## 能力

| 意图 | 示例 `args` |
| --- | --- |
| 创建 Issue | `["issue", "create", "--title", "…", "--body", "…"]` |
| 列出 / 查看 Issue | `["issue", "list"]` / `["issue", "view", "42"]` |
| 关闭 / 评论 / 编辑 | `["issue", "close", "42"]` / `["issue", "comment", "42", "--body", "…"]` |
| 列出 / 查看 / 关闭 / 合并 PR | `["pr", "list"]` / `["pr", "view", "45"]` / `["pr", "close", "45"]` / `["pr", "merge", "45"]` |
| 仓库 / 搜索 | `["repo", "list"]` / `["search", "issues", "crash"]` |
| 任意 API | `["api", "repos/OWNER/REPO/issues"]` |

## github_cli 返回后

有 `summary` 就用它。回复简短、像聊天。

- **简单操作**（创建/关闭/评论/合并、URL/`#n`）：纯文本，一句话。
- **结构化读取**：轻量聊天 markdown——简短引导 + 要点（`* #42 — title`）；不用表格/标题。
- **变更附加信息：** 最多 2–4 个要点。不输出 GraphQL/JSON。不说“I found:”。
- **失败：** 用 `error` / `error_type` 写一句话——说明运行失败。

## 明确适用时优先用专用工具

Slack 提议/执行；工作流摘要；调查代码/提交搜索。
PR CI 修复并推送 → fix_github_pr_ci。安全/质量告警 → fix_github_security_alert。

## 限制

`gh` 在 PATH 中；OpenSRE token 认证。仅限操作——不在 gather/investigation 中。