---
name: operating-github-ci-fixer
description: 当用户要求 OpenSRE 修复失败的 GitHub CI、GitHub Actions 检查、失败的 Pull Request 检查、损坏的 PR 分支，或指定分支（如 main）上的 CI 时使用。
tools:
  - fix_github_pr_ci
---

# operating-github-ci-fixer

GitHub CI 修复请求使用 `fix_github_pr_ci`，不要用 `github_cli` 或 `shell_run`。

规则：

- 用户提供 GitHub Pull Request URL 时，传 `pr_url`。
- 用户指定仓库和 PR 编号时，传 `owner`、`repo` 和 `pr_number`。
- 用户要求修复某个分支自身的失败 CI 时，传 `branch`（如 `branch="main"`）；绝不能与 PR 选择器同时使用。已合并或已关闭的 PR 会被拒绝——基础分支上已有的失败请用 `branch`。
- 指定仓库或 PR URL 时省略 `workspace`；工具会创建隔离的检出。仅在需要编辑该特定匹配检出时传路径。
- 未指定仓库时，省略 `owner` 和 `repo`；工具使用当前检出的 GitHub origin。
- 工具会检查失败的 GitHub Actions 检查，修复本地检出或临时关联的 git worktree，提交，推送 PR 分支或新的修复分支，并等待该推送触发的检查。它不会创建新 PR。
- Fork 的 PR 分支会被工具拒绝，因为 OpenSRE 只推送到同一仓库内的分支。
- 分支目标（如 `main`）绝不直接推送；OpenSRE 会推送新的 `opensre/ci-fix-*` 修复分支。
- 当 PR 分支落后于基础分支，或 GitHub 报告其存在冲突（检查从未启动）时，工具会先将基础分支合并进 PR 分支，解决冲突，重新生成 lockfile，然后推送。若推送的修复本身又产生冲突，它会再次合并基础分支并重新验证。无法安全解决的冲突会报告具体文件和需人工决定的事项。不要绕过它执行 `git merge`。
- 工具负责 CI 日志检查、修复执行、分支检出、提交、推送以及推送后的检查验证。不要绕过它运行原始 `gh` 工作流。
- 若工具返回 `response_text`，原样输出该文本并停止。
- 若未产生修复，回复仅保留 `error` 中的一行简短说明；不要说"后续步骤"，不要加编号选项，不要列示例命令，也不要提出宽泛的追问。