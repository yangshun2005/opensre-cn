---
name: scheduling-github-ci-repairs
description: >-
  为一个仓库的开放 Pull Request 建立持续的本地监控，自动编辑、测试并推送修复，解决失败的 GitHub Actions 检查。提供一次性的私有仓库演示。用于周期性 PR 修复或本地 CI 上手演示。
getting_started: 在本机持续运行（推荐）
demo_order: 2
metadata:
  owner: Vincent
  last_changed_by: Vincent
  last_changed_at: 2026-10-04
  usecases:
    - 用于配置对单个仓库中失败 Pull Request 的持续修复。
    - 用于在一次性私有仓库中演示定时修复。
  requires:
    - 对被监控仓库的 GitHub 写权限，以及已认证的编码 agent
    - 调度主机上已安装 Git；修复用的检出会自动创建
    - 演示需要能创建私有仓库和示例 PR 的 GitHub token
  version: "0.82"
script_tools: references/script-tools.md
---

# 定时 CI 修复上手

每 **30 秒**监控一个仓库，每个周期自动编辑、测试并推送修复到一个失败的 PR 分支。PR 变绿后不停止监控。

私有演示使用相同的修复策略和相同的节奏。

## 目标

让用户跑起一个定时循环，修复一个失败的 PR，并在远少于五分钟内尽快展示一次真实修复。

## 计划

用 `update_plan` 按下面的工作流标题创建实时计划。在某一步的工具调用所在的同一响应中，把该步标记为 `in_progress` 或 `completed`。只调用 `update_plan` 的响应不算进展。私有演示改为计划三步：用一次 `run_ci_repair_demo` 调用运行演示（`verifies: true`），用结果报告响应（`deliverable: true`），然后用 `ask_user_choice` 提供后续选项：修复成功后的交接，或运行受阻或失败后的阻塞问题。

- [ ] 检查前置条件：GitHub 身份与权限范围，然后检查调度器。
- [ ] 用 ask_user_choice 选择仓库或私有演示。
- [ ] 选择失败的 PR，或确认已授权的演示范围。
- [ ] 用一次 `run_ci_repair_demo` 调用运行演示（仅演示）。
- [ ] 用 schedule_ci_repair_loop 调度有界修复，并记录其任务 id。
- [ ] 用一次 `get_ci_repair_loop` 调用等待修复进入终态，并读取其报告。
- [ ] 用一次 `pr view` 调用验证修复。
- [ ] 保存证据，移除演示循环，并用一次 `finish_ci_repair_demo` 调用验证。
- [ ] 以 Markdown 形式响应结果报告。
- [ ] 报告展示后，用 `ask_user_choice` 提供后续选项：成功后的交接，否则为阻塞问题。

## 工作流

### 第 1 步：检查前置条件

私有演示跳过此步：`run_ci_repair_demo` 会自行检查 token 和调度器。否则，两次调用，每个响应一次：

**确认认证并打印 token：**

- `github_cli` `["api", "user", "--include"]` — 确认认证并
  打印 token 的 `X-Oauth-Scopes` 头。

**检查调度器健康：**

- `slash_invoke` `{"command": "/cron", "args": ["list"]}` — 确认
  调度器有响应。

**满足以下条件时完成此步：**

- GitHub 身份、token 权限范围和调度器均已确认。

### 第 2 步：选择仓库

使用用户已指定的仓库，跳过此步其余部分。
开场回答 `Create <owner>/<repo>` 或 `Create <repo>` 即指定该演示仓库。跳过扫描和 `ask_user_choice`。不带 owner 的名称会让 `owner` 为空；演示工具使用 token 的登录名。`Don't create a demo repository` 则保留下面的选择器。
否则，两次调用，每个响应一次：

**找出当前失败的项：**

- `scan_github_ci_health()` — 用户账户和所有组织下的每个仓库，
  仅默认分支和开放 PR。读取 `failing_prs`；

**询问一次：**

- `ask_user_choice`，标题为 "CI Repair Target"：第一项为 "Private disposable demo
  repository"（推荐），然后每个仍有至少一个失败 PR 的仓库各一项，标签为 `owner/repo — N failing PRs`，
  失败最多的在前，最多六项。没有失败 PR 的仓库不提供；在那里跑循环只会空转。如果扫描返回
  `available: false`，照旧提供演示和已配置的仓库，并说明实时扫描不可用。

选择演示即授权创建私有仓库、其分支、PR 和循环。保留扫描结果：第 3 步从中选择 PR，无需再次读取 GitHub。

**满足以下条件时完成此步：**

- 仓库或演示范围已确定。

### 第 3 步：选择失败场景

**现有仓库：**

- `summarize_github_pr_status(owner, repo, state="open", include_checks=true)`
- 选择用户的 PR，或第一个有失败检查的 PR
- 不要使用 fork 仓库的 PR，它们无法工作。
- 如果没有失败的，循环仍在第 6 步启动，第 7 步跳过。

**演示仓库创建（"Demo"）**
范围已在第 1 步授权；无需获取。

**满足以下条件时完成此步：**

- 已从现有仓库选择 PR，或已授权演示在演示仓库中创建 PR。

### 第 4 步：创建演示失败（仅演示）

调用一次 `run_ci_repair_demo(owner="<owner>", repo="<repo>")`。它会播种下面的演示，用演示的快速检查调度有界修复，等待其进入终态，读取一次 PR，保存证据并移除循环，因此返回时第 5–8 步已完成。记录其 id、证据和 `loop_removed`，然后转到第 9 步。一次运行失败即为阻塞。不要在本计划中再次调用它。报告工具的错误文本。

该工具将 `GET /repos/{owner}/{repo}` 的 404 视为不存在，仅在这种情况下创建私有仓库。它提交一个通过的 `main`（`calculator.py` 做加法，`test_calculator.py` 断言 `add(2, 3) == 5`，以及 `.github/workflows/test.yml` 名为 `Demo calculator CI`，在 pull_request 上运行 `python -m unittest -v`），然后在 `demo/failing-ci` 上提交一次，仅修改 `calculator.py` 使 `add` 做减法，将该 pull request 开向 `main`，正文说明这是演示、不要合并，并在 pull-request Actions 运行失败后返回。记录 `pr_url`、`pr_number`、`head_sha` 和 `failed_run_id`。已有的演示仓库和 pull request 会被复用。如果该仓库不是 OpenSRE CI 修复演示，工具保持其不变，并在同一 owner 下播种 `opensre-ci-repair-demo-<4 个小写字母或数字>`。从结果中记录 `owner`、`repo`、`pr_url`、`pr_number`、`head_sha` 和 `failed_run_id`，并在后续步骤中使用该 owner 和 repo。留在本计划中。不要调用 `ask_user_choice`，不要结束回合，不要删除或覆盖被拒绝的仓库。

此步只使用那一个工具。`github_cli`、`list_github_actions_workflow_runs`、组织仓库列表、代码搜索和普通 `git` 都不属于此步。

**满足以下条件时完成此步：**

- 已记录 `pr_url` 和 `failed_run_id`。

### 第 5 步：调度有界修复

对第 3 步选择或第 4 步创建的 PR 调用一次：

`schedule_ci_repair_loop(owner="<owner>", repo="<repo>", pr_number=<n>)`。对于演示，`<owner>` 和 `<repo>` 是播种结果返回的。

该工具自行启动并检查本地后台调度器，注册一个真实的 30 秒 cron 任务，其 tick 直接调用 CI 修复器，并在 PR 变绿或十分钟过去后自行停止该任务。

它请求一次批准

从结果中记录 `task_id`、`pr_url` 和 `next_run`。`task_id` 是调度器的任务 id：`/cron list` 和 `/cron logs <task_id>` 读取它。

如果结果说 `reused: true`，同一 PR 的先前运行仍活跃；保留其 id 和截止时间，不要再次调度。

**满足以下条件时完成此步：**

- 已记录任务 id。

### 第 6 步：观察修复

不要运行 `/cron run <id>`。调度器会在 `next_run`（最多 30 秒后）接手任务，并从那时起负责修复：尝试、CI 验证和截止时间。在托管网关上，斜杠命令在 90 秒后被停止，被停止的 `/cron run` 会连带终止修复。

调用一次 `get_ci_repair_loop(task_id="<id>", wait_until_terminal=true)`。该工具等待直到 `terminal` 为 true 或修复截止时间已过，并从这一次调用返回该结果。其 `response_text` 是检测和修复证据。

第 3 步未发现失败 PR 时跳过此步。

将工作结果与交付分开阅读：已交付的报告可能描述受阻或失败的修复。

不要再次调度或请求另一次尝试；任务自行最多尝试三次。如果结果的截止时间已过且运行仍未进入终态，停止等待并转到第 7 步：PR 会显示发生了什么。

**满足以下条件时完成此步：**

- 结果有 `terminal: true` 且其结局已记录，或
- 截止时间已过且运行被记录为未完成。

### 第 7 步：验证修复

一次调用：`github_cli ["pr", "view", "<n>", "--json", "headRefOid,commits,statusCheckRollup"]`

head 必须是修复产生的新提交，每个 rollup 条目为 `SUCCESS`，且测试文件在修复提交的文件列表中未被改动。

不要在本地运行测试，不要通过第二个工具获取相同状态，不要再次克隆仓库。

如果检查仍在运行，等待 20 秒一次并重复同一调用。

**满足以下条件时完成此步：**

- head 提交的检查通过；否则重试。

### 第 8 步：清理（仅演示）

一次调用：`finish_ci_repair_demo(repo="<owner>/<repo>", pr_number=<n>, loop_id="<id>", outcome="<success|failed|blocked>", failed_run_id=<id>, fix_commit="<sha>", passing_run_id=<id>)`。

该工具在 `~/.opensre/demo-results/` 下为已批准的仓库名写入证据，移除该调度任务，并检查 `/cron list` 不再显示它。对于失败或受阻的演示，省略不可用的 ID。记录返回的证据路径。演示仓库永不删除；报告该仓库仍然存在。

此步只使用那一个工具。`write_demo_evidence`、`slash_invoke`、`github_cli` 和 shell heredoc 都不属于此步。

对于现有仓库，循环保留；只记录其 id。

循环消失且剩余资源已记录时完成。

### 第 9 步：报告

以 Markdown 形式响应报告，内联链接 PR：PR、失败运行 id、循环 id、修复提交、最终检查结果、清理状态、证据路径。

仅当检测、定时修复、检查通过以及（演示的）循环移除均有证据时才声称成功。

不要在此工作流中调用 `memory_recall` 或 `memory_remember`。

**满足以下条件时完成此步：**
报告已作为 Markdown 文本展示给用户时完成。

### 第 10 步：提供后续问题

运行受阻或失败后，改为询问阻塞问题而非下面的菜单：调用 `ask_user_choice`，指明受阻的运行步骤及其阻塞，选项为此处工具能执行的选项和一个保持阻塞的选项。在托管网关上，成功报告后跳过下面的菜单并将此项标记为完成；委派工作的 shell 负责后续。