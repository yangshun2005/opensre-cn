---
name: fixing-github-merge-conflicts
description: >-
  将基础分支合并到与其冲突的开放 Pull Request 中，每次一个，使用 fix_github_pr_ci，并报告需要人工决定的合并。当被要求修复仓库 Pull Request 上的合并冲突时使用，也用于定时冲突循环。
metadata:
  owner: Vincent
  last_changed_by: Vincent
  last_changed_at: 2026-10-04
  usecases:
  - 用于定时循环，使仓库的开放 Pull Request 保持无合并冲突。
  - 用于维护者要求 OpenSRE 修复仓库开放 Pull Request 上的合并冲突。
  requires:
  - 具有 Pull Request 分支写权限的 GitHub 认证。
  - 已安装并认证的编码代理，用于处理 git 无法自行合并的冲突。
  - summarize_github_pr_status 和 fix_github_pr_ci 工具。
  version: "1.1"
---

# 修复 GitHub 合并冲突

将基础分支合并到每个与其冲突的、同仓库的、非草稿的开放 Pull Request 中。

## 计划

仅当步骤 1 找到目标时，在首次调用 `fix_github_pr_ci` 前列出 `update_plan`：步骤 1 `completed`，步骤 2 `in_progress` 并带 `verifies: true`，步骤 3 待处理。

- [ ] 步骤 1. 用 summarize_github_pr_status 找到冲突的 Pull Request。
- [ ] 步骤 2. 用 fix_github_pr_ci 将基础分支合并到每一个中。
- [ ] 步骤 3. 每个新结果回复一行。

## 工作流

### 1. 找到冲突的 Pull Request

调用 `summarize_github_pr_status(owner, repo, conflicts_only=true)`；目标具有 `repairable: true`，且 `mergeable: false` 或 `mergeable_state: "dirty"`。扫描返回即完成。

### 2. 将基础分支合并到每个目标

对每个目标调用一次 `fix_github_pr_ci(owner, repo, pr_number)`，一次一个；它会合并、推送、等待检查，并在需要人工决定时在 Pull Request 上评论。切勿在其周围运行 `git` 或 `gh`，也切勿合并 Pull Request。每个目标都有结果即完成。

### 3. 回复

为每个结果为新（`already_reported: true` 的不算新）的目标写一行：其链接，然后是返回的 `checks_state` 以及任何 `error_kind` 和 `error`；若没有剩余行，仅回复说明行。回复发送即完成。