---
name: analyzing-github-ci-performance
description: >-
  根据单个仓库最近 30 天的原始 GitHub Actions 记录计算 CI/CD 指标表，
  包括失败率和开发者等待时间。用于历史 CI 性能问题或
  首次体验的仓库演示。对于当前失败的检查，请使用 reporting-github-ci-failures。
getting_started: 分析并改进仓库（推荐）
demo_order: 1
metadata:
  owner: Vincent
  last_changed_by: Vincent
  last_changed_at: 2026-10-04
  usecases:
  - 供仓库维护者分析过去 30 天的 CI 可靠性。
  - 供工程团队评估预估的开发者等待时间和失败模式。
  - 供新用户选择本地或示例仓库进行 CI 性能演示。
  requires:
  - 具有仓库 Actions 历史读取权限的 GitHub 认证。
  - analyze_github_ci_reliability 和 scan_local_git_workspace 工具。
  - 本地发现需要本地 Git 检出；示例仓库不需要。
  version: '1.24'
---

# CI/CD 分析

根据原始 GitHub Actions 记录，为一个仓库生成 CI/CD 可靠性报告，包括对已合并 Pull Request 上 CI 等待时间的估算。

除非请求指定其他时间段，否则使用 30 天窗口。

## 计划

阅读本技能后，使用 `update_plan` 创建或修订实时的
CI/CD 可靠性进度计划，以以下五个编号工作流标题
作为其步骤。在与该步骤的工具调用相同的响应中，将该步骤标记为 `in_progress` 或 `completed`。

将展示表格和提供下一步作为单独的计划项持续展示；在每个步骤的完成条件满足时更新状态。

在每次 `update_plan` 调用中，将步骤 4 标记为 `deliverable: true`：其工作
就是回复本身，如果没有该标记，宿主会将计划确定前的纯文本回复
视为过早停止，并且不会显示它。

- [ ] 步骤 1. 使用 scan_local_git_workspace 扫描本地仓库。
- [ ] 步骤 2. 使用 ask_user_choice 选择仓库。
- [ ] 步骤 3. 使用 analyze_github_ci_reliability 收集 30 天指标，并根据其结果准备指标表。
- [ ] 步骤 4. 以纯文本回复展示指标表。
- [ ] 步骤 5. 使用 ask_user_choice 提供调度、Slack 设置或完成选项。

## 工作流

### 1. 扫描本机

无参数调用 `scan_local_git_workspace()`。

当 `scan_local_git_workspace` 在本轮返回时完成，即使
结果为空；示例仓库在步骤 2 中仍然可用。

### 2. 选择仓库

调用 `ask_user_choice`，标题为 `Which repository should I analyze?`。
提供最多 5 个已扫描且具有 GitHub Actions 工作流的仓库
作为 `<owner/repo>`，然后提供 `Tracer-Cloud/opensre` 作为示例选项。
即使只找到一个仓库也要提供选择器；单个扫描
结果不是选择。当没有已扫描仓库具有 GitHub Actions
工作流时，在示例选项之前提供最多 3 个最活跃的仓库作为 `<name> (local insights)`。

调用 `ask_user_choice` 后结束本轮；答案将作为
下一条用户消息到达。

当用户对 `ask_user_choice` 的答案作为消息到达时完成。使用该仓库在步骤 3 继续。`(local insights)` 答案
不是 GitHub 仓库：调用 `skill_view(name="analyzing-local-repositories")`
并针对它遵循该技能，而不是步骤 3 到 5。

### 3. 收集并计算指标

调用 `analyze_github_ci_reliability(owner="<owner>", repo="<repo>", days=30)`。
它读取整个窗口的 Actions 历史（默认分支运行、PR 运行、
重跑尝试、已合并 PR），计算报告中的每个指标，并仅返回
数字：`headline`、`key_results`、`comparison_figures`（比较表中该仓库的
列）、`benchmarks`（对等列）、
`coverage_notices` 以及原始计数。它不渲染任何内容；下面的报告
由你编写。不要分页 REST API 或自行运行 `execute_python_code`。

如果工具报告 GitHub 连接缺失或被拒绝，告诉用户使用宿主或工具提供的设置 URL 在 OpenSRE 应用中连接或重新连接 GitHub。结束本轮，不重试该工具，也不通过 `slash_invoke` 调用 GitHub 设置。应用连接刷新后，恢复同一仓库分析。

如果 GitHub 因任何其他原因无法读取，导致分析未返回 `key_results`，不要展示空表：用一行说明其阻塞原因，然后调用 `skill_view(name="analyzing-local-repositories")` 并针对同一仓库遵循该技能，而不是步骤 4 和 5。

指标定义位于 [Metrics](references/metrics.md)
（`skill_view(name="analyzing-github-ci-performance", reference="metrics")`）；仅在
用户询问某个数字如何定义时阅读它。

#### 将指标表准备为 Markdown 文本

从分析结果的 `benchmarks` 中获取对等列，这些列在
`benchmarks_measured_on` 上按 30 天测量，并用该日期
和窗口为表格添加标题：它们是历史背景，不是排名。不要为它们调用
`skill_view`。对照分析结果检查每个表格单元格。

如果某个单元格在分析结果中没有来源，请返回
上述调用：重新阅读 `coverage_notices` 以查找指定的缺口，如果分析
未返回成功，再运行一次。仍然没有来源的单元格为
`n/a`，并在表格下方说明缺口；绝不要估算它。

准备下面的报告，以便在步骤 4 中交付。

#### 报告格式

标识仓库、默认分支、UTC 窗口和覆盖率。将此表格渲染为文本，将每个占位符替换为计算值或
参考中的基准值：

```
Developer impact:
- xx developer-hours spent waiting on CI across xx developers.
- Most affected developer: up to xx h/week waiting on CI.
- xx% of PR runs failed, creating substantial retry and investigation overhead.

Compared with langchain-ai/langchain and anomalyco/opencode:

| Metric | <owner/repo> | langchain-ai/langchain | anomalyco/opencode |
|---|---:|---:|---:|
| Red time on main | <hours and % of window> | <benchmark> | <benchmark> |
| Mean time to green | <hours> | <benchmark> | <benchmark> |
| CI-caused failure rate | <% of PR workflow runs> | <benchmark> | <benchmark> |
| Slowest normal run | <minutes and workflow> | <benchmark> | <benchmark> |
| PR failure rate | <% of PR workflow runs> | <benchmark> | <benchmark> |

What insights stand out:
- CI-caused failures account for x.x% of all PR runs, roughly x.x-x.x× higher than the comparison repositories.
```

当 `analyze_github_ci_reliability` 在本轮返回时完成，
无论是带有 `key_results` 还是带有指定的阻塞原因，并且每个表格单元格
在其结果中都有来源，或者是 `n/a` 并说明了其缺口。

### 4. 展示指标表

仅以 Markdown 文本回复步骤 3 的报告，不包含其他内容；在该消息中不调用任何
工具。宿主会绘制该回复，然后继续
计划中的步骤 5——之后不要重复报告。

当包含表格的助手回复已显示时完成。

### 5. 提供下一步

调用 `ask_user_choice`，标题为 `See a failing check get fixed?`，`allow_custom` 为 false，并附此说明：`A private demo repository, so the one in the report stays untouched. You watch one check go from red to green.`

选项：
- Watch it fix a failing PR in a demo repo (under 5 min)
- Run the same fix in OpenSRE managed service and close your laptop
- Not now

当此菜单的 `ask_user_choice` 调用在本轮返回时完成。用户的答案将在下一轮到达。除 `Stop at the report` 外的每个分支都由同级技能负责：使用 `skill_view` 加载它并遵循其计划；不要在此处重新实现其步骤。

- **Watch it fix a failing PR in a demo repo (under 5 min):** 调用 `skill_view(name="scheduling-github-ci-repairs")`
  并遵循该技能。仓库已在本会话中选择并分析，
  因此其计划省略扫描和仓库选择步骤，
  其分析步骤复用今天保存的报告。
- **Run the same fix in OpenSRE managed service and close your laptop:** 调用 `skill_view(name="delegating-github-ci-repairs")`
  并遵循该技能。
- **Not now:** 用一行确认并结束。