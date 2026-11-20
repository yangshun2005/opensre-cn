# 产品分析契约

指标清单与计算定义见 [METRICS.md](METRICS.md)。

OpenSRE 通过一个尽力而为、非阻塞的队列发送产品事件。该队列为每个请求向以下地址发送一个带版本的事件：

```text
POST {OpenSRE app origin}/api/analytics/events
```

应用源按以下顺序解析：

1. 托管组织 silo 使用 `OPENSRE_WEBAPP_URL`。
2. 个人 CLI 登录时保存的应用 URL。
3. `OPENSRE_APP_URL`，然后是 `https://app.opensre.com`。

以下情况会为该进程禁用远程投递，而不是回退到其他源或身份：显式 URL 无效、已保存账户不可读、silo URL 缺少 `AGENT_USAGE_SECRET`，或本地凭据存储启用时环境账户令牌与已保存登录冲突。仅环境账户投递需要 `OPENSRE_APP_URL`，并使用该显式源，而不是独立持久化的账户元数据。

请求携带已登录 CLI 的个人 `osre_pat_...` bearer，或 silo 的 `AGENT_USAGE_SECRET`。登录前没有授权头。Webapp 从个人 bearer 推导个人账户和组织身份。silo bearer 认证运行时；其配置的组织是 bearer 认证的断言，不解析 Clerk 用户。匿名请求无法建立任一身份，`properties.user_id` 从不被视为 OpenSRE 账户 ID。

已认证请求使用请求 bearer 对时间戳和精确 JSON 字节做 HMAC-SHA256 签名。bearer 仅存于所有者专属凭据存储，PostHog 项目密钥仅存于 webapp；两者都不嵌入事件体或源码分发。生产源要求 HTTPS。

## 摄取载荷

```json
{
  "schema_version": 1,
  "event_id": "00ed1192-3433-4e46-856a-9a0992e8b212",
  "occurred_at": "2026-09-08T12:34:56.789+00:00",
  "source": "opensre_runtime",
  "anonymous_id": "4d892bf3-7204-4410-9f03-f84190f8a936",
  "event": "cli_command_opensre_integrations_verify",
  "properties": {
    "entrypoint": "opensre",
    "command_family": "integrations"
  }
}
```

- `event_id` 是重复事件的 UUID。`install_detected` 使用稳定的 `install_detected:{anonymous_id}` 键，以便服务器去重安装。恢复未验证的 `installed` 标记使用 `install_detected:{anonymous_id}:delivery-v1`，并设置 `install_detection_reason=unverified_marker`。这是当前检测，不是对原始安装的重建。独立键保留服务器已存储的任何更早事件；`METRICS.md`（Installations）定义消费者如何将两者合并为一次安装。
- `occurred_at` 在事件进入本地队列时赋值，而非网络请求完成时。
- `anonymous_id` 是存储在 `~/.opensre/anonymous_id` 的随机安装 ID。在已认证事件上保留它，以将匿名获客活动与后续账户活动关联。
- `account_authenticated` 在浏览器登录后发出。Webapp 从 bearer 解析 Clerk 用户，将安装/用户关联存入 ClickHouse，并在 PostHog 中将同一 `anonymous_id` 合并到该用户。下游关联接受任何个人 bearer 事件，因此丢失的关联事件不会永久丢失关联。
- 成功的摄取应返回 `202 Accepted`。服务器应按 `(source, event_id)` 去重，并拒绝未知 schema 版本或事件名。只有该确认才会在 `install-deliveries-v1/` 下创建回执，键为安装 ID 和端点 URL 的哈希。较旧的 `installed` 标记为旧客户端保留，但不能建立第一方投递。

接受的事件名是 `events.py` 中的 `Event` 枚举、动态 `cli_command_opensre` 系列，以及内部身份控制 `$identify` 和 `$groupidentify`。Webapp 可将这些控制转换为分析存储，而不是作为产品活动存储。

## 通用属性

每个产品事件都包含：

| 属性 | 含义 |
| --- | --- |
| `cli_version`、`python_version` | 客户端兼容性和版本采用。 |
| `os_family`、`os_version` | 粗略平台支持。 |
| `analytics_properties_version` | 属性证据契约版本；当前为 `2`，独立于信封 schema `1`。 |
| `execution_environment` | `local`、`ci`、`container`、`ci_container` 或 `unknown`；检测器分类。 |
| `is_ci`、`is_container`、`container_runtime` | 已识别的运行时信号，非人工验证。未知测量值省略。 |
| `ci_detection_status`、`container_detection_status` | `detected`、`not_detected` 或 `unknown`。 |
| `automation_status`、`execution_origin` | 报告的自动化或未知来源；摄取将已认证 runner 证据升级为 `confirmed`。 |
| `distribution` | `source_checkout`、`editable_package`、`installed_package`、`frozen_binary` 或 `unknown`，基于此进程加载的代码。 |
| `is_test` | 显式 `OPENSRE_IS_TEST=1`（也接受 `true`/`yes`）、检测到的测试运行器或 CI。独立于 distribution。 |
| `composite_fingerprint` | 仅在无账户身份时使用的单向本地指纹。 |
| `identity_persistence` | 匿名 ID 是否已持久化到磁盘。 |
| `install_marker_state_before_install` | 最近一次记录的安装器运行开始时为 `present`、`absent` 或 `unknown`。 |
| `surface` | 已知时为 `cli`、`slack`、`telegram`、`discord` 或 `buzz`。 |
| `session_id` | 已知时为 OpenSRE 会话关联 ID。 |
| `organization_id` | 个人请求由服务器解析；silo 为 bearer 认证的运行时断言；匿名请求不可信。 |
| `user_id` | 网关轮次的聊天平台参与者 ID，不是 OpenSRE 账户 ID。 |

`$groups`、`$process_person_profile`、`$lib` 和 `distinct_id` 为下游 PostHog 兼容性保留。第一方账户用户 ID 应放在由 bearer 令牌解析的服务器自有列中。

对于 `cli_invoked`，`interactive_option` 是配置，其来源在 `interactive_option_source`。`stdin_is_tty` 和 `stdout_is_tty` 测量终端状态。观察到的 shell 启动使用 `interactive_shell_rendered`。

提示事件在已知时包含 `turn_outcome`、`response_source` 和 `llm_attempted`。仅事件名不能确立 AI 成功：静态终端分发和合成回退文本也会记录。缺失的令牌用量省略，并由 `token_usage_status` 描述。集成快照使用 `integration_snapshot_status`；不可用的清单不发出空列表或零计数。无已执行操作的动作率省略。

Shell 和 PowerShell 安装器以及 `make install` 在安装工作开始前快照 `installed`，解析其目录的方式与运行时的 `get_store_path()` 相同：设置时用 `OPENSRE_WIZARD_STORE_PATH` 的父目录，否则用 `OPENSRE_HOME`，否则用 `~/.opensre`。安装成功后，仅记录路径将该快照保存在标记旁的 `install_marker_state` 中。后续产品事件携带它，包括现有标记抑制 `install_detected` 时；重新安装不会制造另一个首次安装事件或 CLI 使用事件。

`present` 是先前安装状态的证据。`absent` 仅表示未找到标记：删除本地状态可产生与新安装相同的结果。`unknown` 表示检查无法确定存在或不存在。未记录安装器快照时，该属性省略。该值描述最近一次记录的安装器运行，不一定是首次安装或当前调用。不要将 `absent` 映射为“首次安装”。

## CLI 调用名与 distribution

命令名由已注册命令令牌生成，连字符规范化为下划线：`opensre health --rate 5` 发出 `cli_command_opensre_health`；`opensre integrations verify slack` 发出 `cli_command_opensre_integrations_verify`。裸 `opensre` 发出 `cli_command_opensre`。这些事件记录调用，而非完成或成功。参数和选项值从不进入名称。

替代 Python 入口点使用等效的 OpenSRE 命令名；`entrypoint` 属性保留其启动方式。每次调用发出一个命令事件，取代 `cli_invoked`。读取方必须同时接受历史 `cli_invoked` 记录和新系列。在分发发出这些名称的客户端之前，先部署 webapp 的系列验证。

`source_checkout` 和 `editable_package` 标识本地开发代码。`installed_package` 和 `frozen_binary` 标识打包代码，包括本地构建的包；它们不确立发布者签名或官方来源。打包二进制仍可有 `is_test=true`。`execution_environment=local` 描述计算机，而非构建来源。缺失的历史 distribution 或测试证据不得视为发布或非测试使用的证明。

## 事件清单

安装器标签 `-lp`、`-dc` 和 `-gh` 将 `install_origin` 设为 `landing_page`、`documentation` 和 `github`。使用 `bash -s -- -lp` 传递 Bash 参数；原生 PowerShell 接受相同标签。未标记命令省略来源，除非流水线显式设置 `OPENSRE_CICD=1`，此时记录 `cicd`。该标记还在运行时事件上设置 `is_ci=true` 和 `cicd_marker=true`，独立于供应商环境检测。显式命令标签对来源仍优先。该标记是报告的分类，非验证的 runner 身份。`install_source` 仍标识安装器机制，`install_channel` 仍标识请求的 `main`/`release` 构建轨道。

首个净化后的安装事件在投递前保存到 `install-events-v1`，并在确认后保留。重试复用该完整事件；后续带标签的重新安装不能替换其来源，包括未知来源。现有安装标记继续抑制先前记录的安装的捕获。

| 领域 | 事件 | 重要属性 / 回答的问题 |
| --- | --- | --- |
| 获客 | `install_detected`、`account_authenticated`、`cli_command_opensre…` | 安装来源/来源/构建通道/distribution、登录转化、入口点、命令名和布尔标志；从不包含原始参数值。官方安装器在安装后立即调用隐藏的仅记录路径。历史 `cli_invoked` 记录仍有效。 |
| 登录门 | `sign_in_prompted`、`sign_in_selected`、`stay_signed_out_selected` | 交互式 shell 的强制登录屏幕：每次未登录启动一次曝光，然后每个菜单轮次一个事件，含 `choice_label` 和 `method`（选择选项为 `menu`，未选择关闭菜单为 `dismissed`——Esc、`q`、Ctrl-C、Ctrl-D 或 EOF 不区分）。`sign_in_selected` 在浏览器流程开始前记录，仅为意图；`account_authenticated` 报告结果。已登录、非交互和测试运行不发出这些事件。 |
| GitHub 门 | `github_gate_prompted`、`github_gate_rechecked`、`github_gate_exited` | 登录后交互式 shell 的强制 Connect GitHub 屏幕：每次 GitHub 检查失败的启动一次曝光，含 `reason_code` 说明原因（`no_github_connection`、`github_connection_required`、`github_connection_unavailable`、`selection_failed`、`token_missing` 或 `check_unavailable`）；每次“I've connected GitHub, continue”一个 `github_gate_rechecked`，含 `connected`、从 1 开始的 `recheck_count`，仍未连接时含相同 `reason_code`；成员未连接离开时 `github_gate_exited` 含 `choice_label`、`method`（`menu` 或 `dismissed`）和 `recheck_count`。`github_gate_rechecked` 且 `connected=true` 是唯一通过已绘制屏幕的方式。打开仪表板为 `browser_open_requested` 且 `target=integration_setup`。已连接、非交互和测试运行不发出这些事件。在分发发出这些事件的客户端之前，先部署 web 应用摄取支持。 |
| CLI 浏览器认证 | `cli_auth_started` | 每次浏览器登录尝试前一个事件，包括手动 `--no-browser` 链接。`cli_auth_attempt_id` 是独立于 OAuth state 和 PKCE 的新 UUID，也携带在登录 URL 和自动 `browser_open_requested` 事件中。Web 应用将其附加到此尝试的浏览器页面/屏幕事件，最多十分钟，不合并浏览器和安装身份。此事件确立意图；匹配的浏览器视图确立接收。遥测退出时省略事件和 URL 标识符。在分发发出此事件的客户端之前，先部署 web 应用摄取支持。 |
| 运行时健康 | `user_id_load_failed`、`sentry_init_skipped` | 身份持久化和遥测设置失败。 |
| 入门 | `onboard_started`、`onboard_completed`、`onboard_failed` | 漏斗转化、向导模式、目标、提供商和模型。 |
| 集成 | `integration_setup_started`、`integration_setup_completed`、`integration_verified`、`integration_removed`、`integrations_listed` | 按服务的集成采用和设置/验证转化。 |
| 交互操作 | `terminal_actions_planned`、`terminal_actions_executed`、`terminal_turn_summarized` | 计划/执行/成功计数、LLM 回退和会话成功/回退桶。 |
| Agent 循环 | `react_turn_completed` | 阶段、迭代、上限命中、停止原因、工具调用计数、延迟、提供商和模型。`stop_reason` 将所有硬停止报告为 `iteration_cap`；`loop_stop_reason` 保留循环自身原因（`goal_unverified`、`stagnation_limit`、`iteration_cap`、`completed` 等）。抛出异常的运行添加 `error_type` 和经脱敏、限长的 `error_message`。计划 tick 内的运行添加 `scheduled_task_id`。 |
| LLM 额度 | `llm_credit_limit_reached` | 每次因额度耗尽被阻止的失败 LLM 运行一次，即使提示日志禁用。`reason_code` 为 `opensre_credits_exhausted` 或 `provider_credits_exhausted`；`credit_source` 为 `opensre` 或 `provider`。携带阶段、提供商、模型、会话、轮次类型和绑定的提示轮次 ID。运行摘要和提示生成也携带 `ai_error_reason`；提示的 `ai_error_kind` 保持 `quota`。瞬时限流不发出此事件。 |
| Agent 工具调用 | `agent_tool_call_completed` | 工具/来源/角色、是否执行、结果、延迟、错误状态和终止；从不包含工具参数或结果。失败调用添加工具经脱敏、限长的 `error_message`，以及已知时的 `blocked_by`（`duplicate_action`、`plan_required`、`menu_pending`、`approval_declined`、`approval_pending`、`hook_exception`）、`skipped_by`（`turn_terminated`、`host_cancel`）、`exception_type`、工具的 `error_class`，以及不可用集成的 `error_source` 和 `setup_command`。`unavailable` 始终设置；`prompt_turn_id`、`iteration` 和 `tool_call_index` 将调用置于其轮次中。 |
| Ask User | `ask_user_prompt_rendered`、`ask_user_prompt_answered`、`ask_user_prompt_dismissed` | 关联的提示曝光、限长且凭据脱敏的问题/选项文本、所选选项索引、限长的自定义答案，以及关闭选择器的键类（`dismiss_key`，如 `esc`、`ctrl_c` 或 `eof`）。列出的答案仅发送索引。渲染的提示在其作者声明时携带 `reason_code`（模型打开的菜单为 `choice`，宿主打开的 skill 入口菜单为 `entry_menu`）；从不从问题文本推断。 |
| Shell 和浏览器 | `interactive_shell_rendered`、`browser_open_requested` | 首个交互式 shell 界面，包括登录屏幕。`--resume`、`opensre onboard` 后自动启动或 CLI 子命令不记录。`browser_open_requested` 是按安全目标标签的应用程序请求浏览器打开结果。终端不暴露手动渲染的链接是否被点击。 |
| Agent 工作流 | `skill_executed`、`skills_release_activated`、`skill_prerequisite_missing`、`opensre_commit_created` | 成功的 skill 入口，含 skills 发布、来源（捆绑、远程或本地覆盖）、卡片版本和 12 字符内容摘要；进程切换到不同 skills 发布；因宿主拥有的先决条件未满足而在入口处暂停的 skill（`skill`、`check`、`reason_code`，如 `credential_missing`）；受支持的 OpenSRE 修复工作流产生的提交。 |
| AI 轮次 | `$ai_generation` | 轮次/会话 ID、轮次类型、模型/提供商、延迟、令牌、集成快照、结果和错误类别。还包含 `$ai_input` 和 `$ai_output_choices` 中经脱敏的提示和响应文本。操作轮次添加循环的 `stop_reason`、目标审查的最后拒绝（`goal_review_reason`）、`last