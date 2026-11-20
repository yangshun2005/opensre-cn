# 解决合并冲突

与基础分支冲突的 Pull Request 不会触发任何检查：GitHub 会提示“This branch has conflicts that must be resolved”，CI 根本不会启动。此时把基础分支合入 PR 分支即可解决。

## 用哪个工具

- 在 CI 修复流程中，`fix_github_pr_ci` 会自行合并基础分支，并在做任何修改前解决冲突；不要额外加一步。
- 当用户在检出目录中要求解决、修复或完成合并，或要求提交并推送已解决的合并时，调用 `resolve_merge_conflicts`。它处理当前目录中正在进行的合并，也可以传 `ref`（例如 `origin/main`）先合并该分支。
- 把用户的决定放进 `instructions`（“config.py 保留我们的版本”）；这些内容会原样传给编码 agent。

不要为此通过 `shell_run` 执行 `git merge`、`git add`、`git commit` 或 `git push`；合并进行中 shell 会拒绝这些命令。不要用 `code_implement` 解决冲突标记。

## 工具做什么

1. 在 shell 中一次性并排显示每个冲突块：我们的版本、他们的版本，以及“(to decide)”。发生此情况时 `rendered_in_shell` 为 true，因此不要重复文件内容。
2. 让编码 agent 像 Claude Code 或 Cursor 那样直接解决每个文件，无需先询问。只有当 agent 无法解决某个文件时（保留我们的、采用他们的、合并、或自由文本），或用户要求逐文件决定时（传 `decisions={"*": "each"}`），才通过 shell 菜单询问。结果中显示 `menu: queued` 时，结束本轮；答案会作为下一条用户消息到达，下一次不带参数的调用会自行读取。仅当用户已用文字明确说明选择时，才传逐文件的 `decisions`。绝不要自己就如何解决某个文件提问：由工具来问。
3. 用 git 应用“保留我们的”和“采用他们的”，只把需要合并的文件交给编码 agent。锁文件从解决后的清单重新生成，绝不手工合并。然后验证没有残留冲突标记或未合并路径，并显示结果表。
4. 请求 shell 的审批策略提交合并，并把分支推送到其跟踪的分支。在 `/auto high`（默认）下直接执行；更低级别会先询问用户；无人值守运行直接执行。远程分支绝不会是受保护的基础分支。
5. 等待以该分支为 head 的开放 Pull Request 的检查，并报告 `checks_state`（`passed`、`failed`、`timed_out`、`superseded`、`conflicted` 或 `not_watched`）。仅当用户要求不等待时才传 `wait_for_checks=false`。

## 如何回复

- 第一行重复 `outcome`。它说明合并是否已提交并推送，优先级高于 `coding_agent_summary`（后者由 agent 在提交前撰写）。
- 列出 `resolutions`（每个文件一行：保留我们的、采用他们的、合并两者），让用户看到决定了什么。
- 以 `next_step` 结尾。
- 当 `error_kind` 为 `conflicts_remain` 时，用 `ask_user_choice` 逐个询问 `questions` 中的条目，`question` 和 `options` 原样使用（它们显示每一侧的内容），然后把答案放进 `instructions` 再次调用工具，注明文件名和所选的一侧。绝不要编造不同的问题或自行回答。合并仍在进行中；不要中止它。
- 当 `error_kind` 为 `cancelled` 时，用户按了 ESC：说明做了什么、没做什么，然后等待下一条指令。
- 当 `error_kind` 为 `confirmation_denied` 时，说明已解决的文件在工作区中未提交，并询问需要改什么。
- 当 `error_kind` 为 `push_failed` 时，合并已在本地提交；给出 `next_step` 中的推送命令。
- 当 `error_kind` 以 `checks_` 开头时，合并已推送；指出 `failing_checks`，并提出修复该 Pull Request 的 CI。
- 当 `checks_state` 为 `not_watched` 时，合并已完成：说明为何未观察检查（来自 `checks_detail`）并结束。不要把计划步骤标记为阻塞，也不要向用户询问验证事宜。