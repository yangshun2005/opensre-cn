---
name: repair-github-ci
description: >-
  通过 fix_github_pr_ci 修复失败的 GitHub CI / Actions 检查并推送到现有 PR 头分支，或通过关联的修复工作树修复某个分支的失败 CI
metadata:
  owner: Vaibhav
  last_changed_by: Jan
  last_changed_at: 2026-09-14
  usecases:
  - 供仓库贡献者修复 Pull Request 上失败的 CI。
  - 供维护者通过关联的本地工作树修复指定分支。
  requires:
  - 具备目标仓库写权限的 GitHub 认证。
  - 本地检出，其 origin 与目标仓库一致。
  - 已安装并完成认证的编码代理。
  - fix_github_pr_ci 工具及其支持的本地执行环境。
  version: '2.0'
---

# GitHub CI 修复

使用 `fix_github_pr_ci` 修复 Pull Request 或指定分支上失败的检查。

## 何时使用

- 用户要求修复失败的 CI、损坏的 GitHub Actions 检查、失败的 PR 检查、变红的 Pull Request 分支，或 `main` 等指定分支上的 CI。
- 用户说"修复这个 PR 的 CI"、"修复 PR 123 的 CI 并推送"、"修复 owner/repo#123 上失败的检查"，或提供 GitHub Pull Request URL 并要求修复 CI/检查。
- 用户要求修复分支本身的失败 CI——"修复 main 上的 CI"、"main 变红了，修一下"、"修复默认分支的 CI 并推送"——不涉及 PR。

## 相关工作流

- 普通 PR 读取、评论、关闭、合并、标签或 Issue 操作。使用 `github_cli`。
- 安全告警修复。使用 `fix_github_security_alert`。

## 工作流规则

- 对于 GitHub PR URL，调用：
  `fix_github_pr_ci(pr_url="<url>")`
- 对于 `owner/repo#123` 或"owner/repo 中的 PR 123"，调用：
  `fix_github_pr_ci(owner="owner", repo="repo", pr_number=123)`
- 对于"修复 main 上的 CI"（或任何无 PR 的指定分支），调用：
  `fix_github_pr_ci(branch="main")`
  切勿将 `branch` 与 PR 选择器一起传入，也切勿编造用户未指定的分支。
- 如果未指定 owner/repo，则两者都省略，让工具使用当前检出的 GitHub origin。
- 此工作流中切勿使用 `github_cli` 或 `shell_run` 运行原始 `gh pr checks`、`gh run view`、检出、提交或推送。CI 修复器负责 PR 元数据、失败检查日志检查、修复执行、分支安全、提交和推送。
- 工具在获得批准后推送到现有 PR 头分支。不要询问用户是否要开新 PR。
- 落后于基础分支或与之冲突的 PR（GitHub 显示为不可合并且不启动检查）由工具处理：它将基础分支合并到 PR 分支、解决冲突、重新生成 lockfile、推送并等待检查——即使冲突仅在其自身推送后才出现也是如此。切勿在其周围运行 `git merge`、`git rebase` 或编辑冲突。如果它报告被阻塞的文件，转述那一行并停止。
- 对于 `main` 等分支目标，工具会创建一个单独的关联 git 工作树，在新的 `opensre/ci-fix-*` 分支上提交，并推送该分支。它绝不会直接推送到受保护分支。
- 如果工具返回 `response_text`，原样输出该文本并停止。
- 如果设置了 `error_kind`，用 `error` 中的一句话简短回复。不要说"后续步骤"，不要添加编号选项，不要列出示例命令，也不要提出宽泛的追问。

## 示例

- "修复 https://github.com/Tracer-Cloud/opensre/pull/4597 的 CI 并推送"

  ```text
  fix_github_pr_ci(pr_url="https://github.com/Tracer-Cloud/opensre/pull/4597")
  ```

- "修复 Tracer-Cloud/opensre#4597 上失败的检查"

  ```text
  fix_github_pr_ci(owner="Tracer-Cloud", repo="opensre", pr_number=4597)
  ```

- "修复 Tracer-Cloud/opensre 中 main 上的 CI 并推送"

  ```text
  fix_github_pr_ci(owner="Tracer-Cloud", repo="opensre", branch="main")
  ```

- "当前 PR 的 CI 失败了，修复并推送"

  ```text
  fix_github_pr_ci()
  ```

- "修复 main 上的 CI"

  ```text
  fix_github_pr_ci(branch="main")
  ```