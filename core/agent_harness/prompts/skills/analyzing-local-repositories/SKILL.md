---
name: analyzing-local-repositories
description: >-
  展示用户本地仓库反映出的工作方式，从本机的 git 和 CI 文件读取，无需 GitHub token 或网络：后续修复、CI 试错、AI 协作、工作流规范、测试与遗留问题。适用于以下情况：GitHub 未连接或入门引导期间无法读取、用户仓库没有 GitHub Actions、或用户询问其提交或本地仓库说明了什么。如需 CI 运行历史和失败率，请使用 analyzing-github-ci-performance。
metadata:
  owner: Vincent
  last_changed_by: Vincent
  last_changed_at: 2026-10-04
  usecases:
  - 适用于 GitHub 连接缺失或失败的首次用户，使入门引导仍能展示来自其自身仓库的洞察。
  - 适用于仓库没有 GitHub Actions 或使用其他 CI 服务的开发者。
  - 适用于询问其本地提交和 CI 文件反映了哪些习惯的用户。
  requires:
  - analyze_local_repositories 工具，无需凭据或网络即可读取本地 git 检出。
  - 至少一个包含近期提交的本地 git 检出；否则该技能会提议查看其他文件夹。
  - 后续菜单需要交互式终端；在其他环境中，选项以文本形式列出。
  version: '1.0'
---

# 本地仓库洞察

向用户展示其自身仓库反映出的工作方式，从本机 git 读取，无需 GitHub token 和网络，然后提供下一步。这通常在用户的首次会话中运行，此时 GitHub 无法使用，因此结尾必须涉及用户自身的工作，绝不能只给出一个阻塞项。

## 计划

阅读本技能后，使用 `update_plan` 根据下方四个编号的工作流标题创建实时的 Local Insights 计划。在该步骤的工具调用所在的同一响应中，将步骤标记为 `in_progress` 或 `completed`。在每次 `update_plan` 调用中，将步骤 1 标记为 `verifies: true`，步骤 2 标记为 `deliverable: true`：步骤 2 的工作就是回复本身。

- [ ] 步骤 1. 使用 analyze_local_repositories 读取本地仓库。
- [ ] 步骤 2. 以纯文本回复展示本地洞察报告。
- [ ] 步骤 3. 使用 ask_user_choice 询问要关注什么以及下一步做什么。
- [ ] 步骤 4. 使用 memory_remember 保存要关注的内容，并启动所选的下一步。

## 工作流

### 1. 读取本地仓库

调用 `analyze_local_repositories`，参数如下：

- `reason`：
  - 当用户在 GitHub 设置菜单中选择其本地仓库，或 GitHub 没有 token 时，为 `github_not_connected`；
  - 当 `analyze_github_ci_reliability` 无法读取 GitHub 时，为 `github_failed`；若其指定了 `error_kind`，则将其作为 `github_error` 传入；
  - 当用户选择了标记为 `(local insights)` 的仓库时，为 `no_github_actions`；
  - 其他情况为 `requested`。
- `repository`：用户在本会话中早先选择的仓库，格式为 `owner/repo` 或其本地路径（如有）。
- `paths`：若运行了扫描，则为本次会话 `scan_local_git_workspace` 结果中每个仓库的 `path`，所选仓库自身的路径在前，然后是活动最频繁的仓库。否则省略；工具会自行扫描。

当 `analyze_local_repositories` 在本轮返回时完成。若返回 `cancelled`，停止。若返回 `success: false` 并带有 `error`，用一行说明无法读取本地仓库，然后停止。若此构建没有 `analyze_local_repositories` 工具，用一行说明 `opensre update` 会添加此报告，然后停止。

### 2. 展示报告

以 Markdown 文本回复下方报告，不要包含其他内容；该消息中不调用任何工具。宿主会展示它，然后计划继续执行步骤 3。

每个数字都来自工具结果：绝不估算、四舍五入或添加。工具不返回提交消息或代码，因此绝不引用任何内容。

1. 当 `reason` 为 `github_not_connected` 或 `github_failed` 时，以一行开头，用平实的语言说明阻塞项，并说明此报告不需要 GitHub，例如："无法从本机读取 GitHub，因此以下是你的本地历史所展示的内容。" 不再多说阻塞项。
2. 一行：`<repositories> 个仓库 · 你在过去 <days> 天内的 <own_commits> 次提交。从本机 git 读取；没有代码或提交消息离开本机。`
3. 一个 `What stands out` 标题，然后按给定顺序列出前三个标签不为 `Rhythm` 的 `insights`。每条为一个项目符号：`label` 加粗并带冒号，`fact` 保持不变，然后从下表添加一个简短从句。
4. 当存在 `Rhythm` 洞察时，在项目符号后加一行：其 `fact`。
5. 当 `reason` 不为 `requested` 时，加一行：`Connect GitHub to add:` 以及用 ` · ` 连接的 `github_only_metrics`。
6. 每个 `coverage_notices` 条目单独一行。

| 标签 | 要添加的从句 |
|---|---|
| Retired actions | 迁移到当前版本即可修复；GitHub 连接后 OpenSRE 可以打开该 Pull Request。 |
| Follow-up fixes | 快速重复修复通常意味着问题在提交后才暴露，往往在 CI 中；OpenSRE 会修复你 Pull Request 上失败的检查。 |
| CI trial and error | 每次尝试都消耗一次 CI 运行；OpenSRE 为你复现并修复 CI 失败。 |
| AI pairing | 智能体写代码的速度超过 CI 的跟进速度；OpenSRE 以该速度保持 CI 绿色。 |
| Workflow hygiene | 标签可以被移动到新代码上，如 2025 年 tj-actions 事件；超时和并发组可限制浪费的运行器时间。 |
| No CI | 这些提交在合入前没有任何检查。 |
| Tests with code | 没有测试的代码变更正是回归容易溜过的地方。 |
| Local checks | 运行框架的安装命令可在 CI 之前捕获这些问题。 |
| Reverts | 每次回滚都是一次到达分支后不得不撤回的变更。 |
| Loose ends | 过期的分支和暂存隐藏了未完成的工作。 |
| Large commits | 大型变更审查更慢，回滚也更难。 |
| Other CI | OpenSRE 的 CI 分析目前读取 GitHub Actions。 |

当 `repositories` 为 0 时，将第 2 至 6 项替换为一行，说明在该时间窗口内未找到有提交的本地仓库，并且当 `skipped_protected` 列出文件夹时，说明未搜索这些文件夹。

当包含报告的回复已展示时完成。

### 3. 询问要关注什么以及下一步做什么

调用一次 `ask_user_choice`，包含两个 `questions`：

1. `label` 为 `Watch`，`title` 为 `Which of these should OpenSRE keep an eye on?`，`multi_select` 为 true，选项：步骤 2 中 `What stands out` 下展示的洞察标签，然后是 `None of these`。
2. `label` 为 `Next`，`title` 为 `What next?`，选项来自第一个适用的行，然后是 `Not now`：

| 条件 | 选项 |
|---|---|
| `repositories` 为 0 且 `skipped_protected` 列出文件夹 | 每个列出的文件夹对应 `Look in <folder>` |
| `reason` 为 `github_not_connected` | `Connect GitHub and get the CI report` |
| `github_error` 为 `tls_untrusted` | `Show me how to update OpenSRE` |
| `github_error` 为 `unauthorized` 或 `not_found` | `Reconnect GitHub`、`Pick another repository` |
| `reason` 为 `github_failed` | `Try the CI report again` |
| `reason` 为 `no_github_actions` | `Watch OpenSRE fix a failing check in a demo repo (under 5 min)` |
| 其他情况 | `Get the CI report for one of these repositories` |

当步骤 2 在 `What stands out` 下未展示任何内容时（例如 `repositories` 为 0），只询问 `Next` 问题，作为 `title` 和 `options`。调用后结束本轮。若结果表示菜单不可用，则以文本形式列出问题及编号选项并等待。

当 `ask_user_choice` 调用在本轮返回时完成。答案作为下一条用户消息到达。

### 4. 记住并启动下一步

当 `Watch` 答案指定了洞察时，调用 `memory_remember`，`name` 为 `local-insights-watch`，`type` 为 `preference`，一行 `description`，`content` 列出所选标签和今天的日期，与下方的下一操作在同一响应中。

然后根据 `Next` 答案执行操作：

- `Connect GitHub and get the CI report`、`Try the CI report again`、`Pick another repository` 或 `Get the CI report for one of these repositories`：调用 `skill_view(name="analyzing-github-ci-performance")` 并遵循它。
- `Reconnect GitHub`：调用 `slash_invoke`，命令为 `/integrations`，参数为 `["setup", "github"]`，然后结束本轮。
- `Show me how to update OpenSRE`：用一行回复说明 `opensre update` 会安装最新构建，之后可再次询问。
- `Watch OpenSRE fix a failing check in a demo repo (under 5 min)`：调用 `skill_view(name="scheduling-github-ci-repairs")` 并遵循它。
- `Look in <folder>`：再次调用 `analyze_local_repositories`，`root` 设为该文件夹，`reason` 相同，然后重复步骤 2 和 3。
- `Not now`：用一行确认并结束。

当所选的下一步已启动或用户拒绝时完成。