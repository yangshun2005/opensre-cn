---
name: delegating-github-ci-repairs
description: >-
  在交互式 Shell 中，准备托管网关并委派一次有边界的 GitHub PR 修复或远程演示。用于远程 CI 修复入门。
getting_started: 在 OpenSRE Cloud 中运行一次修复
demo_order: 3
metadata:
  owner: Vincent
  last_changed_by: Vincent
  last_changed_at: 2026-10-04
  usecases:
    - 适用于在托管网关上运行 GitHub CI 修复的交互式 Shell 用户。
  requires:
    - 一个可访问的托管网关，具备 GitHub 集成和已认证的编码代理。
    - 一个交互式 Shell，以及组织中已登录的 OpenSRE 账户，用于访问托管网关。
    - 对所选 PR 的 GitHub 写权限；演示模式还需要创建私有仓库的权限。
  version: "2.13"
---

# 委派远程 CI 修复

本流程在网关上完成一次有边界的修复，无需 Shell 参与。它不设置持续的仓库监控。

## 角色

**编排器（本地 Shell）：**

- 检查网关是否就绪，选择目标，委派任务，验证并汇报。

**执行器（网关）：**

- 调用一次 `run_ci_repair_demo`，并将结果回报给 Shell。

## 计划

使用 `update_plan` 根据以下工作流标题创建实时计划。在同一步骤的工具调用所在的同一响应中，将该步骤标记为 `in_progress` 或 `completed`。仅调用 `update_plan` 的响应不算进展。

**在交互式 Shell 中：**

- [ ] 使用 check_hosted_gateway 准备托管网关。
- [ ] 通过一次 ask_hosted_gateway 提示探测网关上的 GitHub 访问权限。

**在交互式 Shell 中：**

- [ ] 通过一次 ask_user_choice 问题确认目标和审批。
- [ ] 使用 ask_hosted_gateway 委派执行，并保留其提示 ID。


**远程网关代理：**

- [ ] 对已批准的 owner 和 repo 调用一次 `run_ci_repair_demo`，不加载技能。
- [ ] 以 Markdown 形式返回该工具的结果。GitHub 上不会删除任何内容；演示仓库会保留。

**在交互式 Shell 中：**

- [ ] 通过重新读取委派的提示 ID 验证结果（verifies: true）。
- [ ] 以 Markdown 形式展示远程结果和证据。
- [ ] 通过 ask_user_choice 提供下一步或阻塞解决方案。

## 成功标准

仅当以下条件全部满足时，工作流才算成功：

1. 托管网关健康，且 GitHub 权限处于活跃状态。
2. 已配置一个种子演示仓库，其中包含一个失败的 PR
3. CI 失败在远程被检测到。
4. 远程修复循环在无需本地 Shell 干预的情况下修复了失败。

## 工作流说明

### 准备托管网关

**仅限 Shell：**

- 运行 `check_hosted_gateway()`。如果未运行，调用一次 `start_hosted_gateway()`，并反复检查直到就绪。
- 不要发送测试提示。下面的探测是第一个提示，其回答即可证明网关能接收提示。

**完成条件：**

- `check_hosted_gateway()` 报告网关正在运行。

### 探测 GitHub 访问权限

发送一条 `ask_hosted_gateway` 提示：

报告此网关用于 CI 修复演示的 GitHub 访问权限。调用一次 `probe_github_repair_access`，不要创建任何内容。回答登录名、令牌类型和权限范围，以及 `owners` 中每个 owner（登录名加上每个组织）一行，说明 `can_create_repositories` 是否为 true。不要提议仓库名称。

**完成条件：**

- 网关给出了登录名、令牌类型，以及至少一个可创建仓库的 owner。否则记录一个阻塞项。

### 展示最终修复计划

如果开场消息已经用 `Create <owner>/<repo>` 或 `Create <repo>` 回答了 `Create a private demo repository?`，那就是目标。不要为此调用 `ask_user_choice`。将该 owner 和 repo 作为委派的 `facts` 传入。没有 owner 的名称使用探测得到的登录名。`Don't create a demo repository` 则走下面的现有 Pull Request 路径。

展示标题为 `Remote Repair Plan` 的最终修复计划。将探测结果（登录名、令牌类型、owner）放入概览，让用户看一次。

- `Create <owner>/<name> and run the demo there`
- `Use an existing pull request`

**完成条件：**

- 所选范围已知时。
- 如果并非所有必需参数都已知，继续用 `ask_user_choice` 询问。

### 委派修复

- 发送一条 `ask_hosted_gateway` 提示："This is a new request. Call `run_ci_repair_demo` once with owner <owner> and repo <repo>. Do not load a skill; this prompt is the whole task. Do not reuse task IDs or repositories from earlier in this conversation. Delete nothing on GitHub. Ask the user only about a blocked step. If the seed or schedule fails, return that failure and do not schedule another loop."
- 将目标作为 `facts` 传入（`demo`、`owner`、`repo`、`pr_number`）。保留提示 ID。


**完成条件：** 网关返回了任务 ID 和结果，或返回了阻塞项。

### 验证远程结果

### 验证远程结果

- 调用一次 `ask_hosted_gateway(prompt_id=<the delegated prompt ID>)`。不要发送新提示。
- 将记录与委派报告核对：相同的任务 ID、修复提交、通过的运行 ID，以及循环已移除。
- 此重新读取是唯一的验证方式。`github_cli` 和 `list_github_actions_workflow_runs` 不属于此步骤。只要记录中有任务 ID、修复提交和通过的运行 ID，即使记录简短也算验证通过。
- 如果显示网关未运行，保持其停止状态。将此步骤标记为阻塞，注明"gateway stopped after reporting; the delegated report is unverified"，并展示该报告。

**完成条件：**

- 重新读取的结果与报告一致，或该步骤因原因被阻塞。

### 在报告中展示结果

- 报告目标 PR、任务 ID、修复结果、可用的 CI 证据链接和保留的资源。清晰简洁地说明待处理、阻塞或失败的结果。

- 将每个 GitHub 链接写为完整 URL，例如 `Pull request: https://github.com/<owner>/<repo>/pull/<n>`，不要写成 Markdown 链接文本：终端会显示链接文本而不显示其 URL。给出委派记录中有的 Pull Request、失败提交、失败运行、修复提交和通过运行的 URL。

- 从委派记录中添加 `Root cause analysis` 部分：什么失败了、根本原因、修复（当记录显示 diff 时附上）、以及验证。只陈述记录所述内容；如果记录没有根本原因分析，说明网关未报告。

- 任务的网关归属确立了与 Shell 的独立性；只有在执行期间 Shell 确实断开连接时，才声称测试过断开连接。

- 只展示一次报告。不要重复网关流式输出的步骤。

- 在此工作流中不要调用 `memory_recall` 或 `memory_remember`。

**完成条件：**

- Markdown 报告已展示。保持网关运行，以便未完成的修复可以继续。

### 提供后续步骤

在成功修复报告之后，用一次 `ask_user_choice`，标题为
`Hand off the next CICD fix?`，`allow_custom` 为 false，选项如下：

- Fix a failing PR from Slack by tagging @OpenSRE
- Guard failing PRs on one of your repos
- Not now

完成条件：此菜单的 `ask_user_choice` 调用已在本轮返回。用户的回答在下一轮到达。

- **连接 Slack 以便代理在那里修复问题：** 调用
  `skill_view(name="connecting-slack")` 并遵循该技能。频道提及或私信会移交下一个失败的检查。不要向 Slack 发帖，也不要提供 Telegram。
- **在你使用的仓库上安排修复循环，以便下一个失败的 PR 得到修复：**
  调用 `skill_view(name="scheduling-github-ci-repairs")` 并遵循该技能。
  不要复用私有演示。选择用户已在使用的仓库，以便循环保留并为下一个失败的 Pull Request 推送修复。
  机器必须保持开机。
- **返回 Shell：** 一行确认并结束。

### 阻塞项

**GitHub 连接阻塞项：**

- 如果缺少凭据，引导用户前往 https://app.opensre.com/dashboard/github，并在修正后遵循工具的后续指引。

**完成条件：**

- 阻塞项已解决，或用户想自己写具体回答。
- 对于待处理任务，提供继续观察同一任务 ID 或让其继续运行的选项。
- 将这些恢复选项与上面的成功演示选项分开。