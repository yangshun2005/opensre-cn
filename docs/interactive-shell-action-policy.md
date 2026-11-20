# 交互式 Shell 操作策略（ADR）

## 状态
已被取代 — 2026年6月18日。原始决策中描述的声明式规则包确定性映射器，以及基于正则的规划器后处理覆盖，均已移除。参见下文"决策（当前）：LLM 是唯一的工具选择器"。原始决策保留以供历史参考。

## 背景
交互式 Shell 操作策略曾在单个模块中通过层层启发式规则不断膨胀：一个正则/关键词确定性映射器从自由文本推断工具，规划器后处理则用更多正则重写模型选定的操作。这些启发式规则与 LLM 竞争，导致误分类（例如"investigate a sample test alert?"被当作信息性问题处理，而非运行示例告警），并且是优先级漂移的反复来源。

## 决策（当前）：Shell 操作代理是唯一的工具选择器
1. 没有正则/关键词意图推断。非命令轮次完全由 Shell 操作代理通过原生工具调用选择。
2. 工具选择由操作代理系统提示词（`core/agent_harness/prompts/action/assemble.py`）和工具目录（`tools/interactive_shell/*`）中各工具的描述驱动。保持两者精确——它们是唯一的选择信号。
3. 操作路径不会事后重写模型的工具调用。工具调用作为一等 `AgentTool` 通过共享的 `core` 工具调用循环执行；参数形态和可用性由 AgentTool 运行时契约和各工具门控强制执行。
4. 当操作代理提示词溢出上下文窗口时，该轮次回退为对话式回复，而非猜测操作。当操作代理 LLM 本身不可用时，REPL 渲染并持久化一个失败的助手轮次，以便 `/resume` 能显示该故障。
5. 用户逐字输入的 `/slash` 命令文本会被确定性分发，不经过操作代理 LLM（参见下文"确定性字面 `/slash` 分发"附录）。这是显式命令旁路，而非自然语言意图推断：自由文本仍完全由操作代理选择。运行时在 `runtime/input_policy._literal_slash_command_text` 中的字面 `/slash` 检测仍属终端 UI 策略（抑制 spinner 和独占 stdin 门控）；执行侧的确定性分发位于 `core/agent_harness/turns/action_driver.py`。

## 这对变更意味着什么
- 要改变某种措辞映射到工具的方式，请编辑操作代理系统提示词和/或相关工具描述——绝不要添加正则。
- 要添加新工具，请将其加入工具目录，并提供清晰、自描述的 `description` 和 `input_schema`；操作代理从该文本中选择它，并将其作为 AgentTool 接收。
- `tests/core/agent/scenarios/` 下的实时轮次场景是操作代理行为的回归测试面。确定性场景（`intent_class: deterministic`）仅断言字面命令分发。

## 原始决策（历史，已被取代）
1. 确定性映射被拆分为声明式规则包，并配有一张显式优先级表。
2. 规则匹配窗口被命名为类型化策略，而非内联数字切片。
3. 规划器后处理作为对类型化 `PlannerState` 的纯变换运行。
4. 失败关闭的策略变换和规范化变换分别注册，并在一个有序列表中执行。
5. 旧版规划器结果元组兼容性被收敛到单个适配器之后。
6. 规划器契约包含策略追踪产物，以检测静默优先级漂移。

## 集成感知与 LLM 驱动的只读发现

附录 — 2026年6月18日。

关于实时状态的事实性问题（例如"is sentry installed?"）无需添加关键词/正则规则即可回答。两种互补机制：

1. 上下文接地（非操作规划）。在 REPL 启动时，`run_repl_async`（`surfaces/interactive_shell/main.py`）从 `integrations/catalog.py` 中共享的 `configured_integration_services()` 辅助函数填充 `session.configured_integrations`（与欢迎横幅使用同一来源，因此二者永不偏离）。代理提示词将已配置集合列为事实，使模型在状态已知时可直接回答。
2. LLM 驱动的发现。代理系统提示词（`core/agent_harness/prompts/action/assemble.py`）允许模型自行决定发出只读发现操作（例如 `slash_invoke("/integrations", ["list"])` 或 `["verify"]`）来发现答案，而非回避。此过程没有关键词映射——由 LLM 决定。在 alpha 全允许策略下，每个发现操作都无需确认即可运行（`execution_policy.allow_tool("slash")` 返回 `allow`）；原先的 `ExecutionTier`/`resolve_slash_execution_tier` 分类已被移除，因为它没有门控任何东西。不涉及失败关闭的正则规则；由代理决定是否发出发现操作。

### 观察→回答摘要循环

附录 — 2026年6月18日。

当代理运行只读发现命令时，工具结果会追加到同一 ReAct 对话中。随后同一模型撰写面向用户的回答。没有第二次助手处理，也没有确定性的代理前分发。

发现命令也不再向 REPL 倾倒验证器堆栈跟踪：验证期间的供应商/配置失败（例如 GitHub MCP `401`）会记录为一行警告，而非完整回溯，因为 `report_validation_failure` 现在默认 `include_traceback=False`，同时仍将异常捕获到 Sentry。

### 自动启动交互式设置（"can you configure X?"）

附录 — 2026年6月18日。

当用户要求配置、连接、设置或添加集成（"can you configure sentry?"、"connect datadog"）时，操作代理不会仅仅移交给对话助手——它会为用户启动设置向导。操作代理发出 `slash_invoke` 工具调用，指向 `/integrations setup <service>` 或 `/mcp connect <server>`。由模型选择服务；没有按供应商硬编码。

设置向导是需要独占 stdin 的子进程，因此无法在轮次中途内联运行（实时提示词正在竞争 stdin）。取而代之，`tools/interactive_shell/actions/slash.py` 通过 `session.queue_auto_command(...)` 将命令排队，该函数预填下一个提示词并标记为自动提交。随后提示词刷新钩子（`surfaces/interactive_shell/ui/input_prompt/refresh.py` 中的 `wire_prompt_refresh`）提交它，使命令流经 REPL 正常的独占 stdin 轮次路径（`turn_needs_exclusive_stdin` 识别 `/integrations setup`）——这是交互式子进程获得干净 stdin 的唯一位置。在非 TTY/脚本化上下文中（没有可提交的提示词），slash 命令路径退化为正常的非交互式 slash 行为。

### 移除规划阶段的失败关闭保护（v0.1）

附录 — 2026年6月18日。

操作代理不会拒绝轮次。此前，旧规划器无法映射到可执行工具的任何子句——通过 `mark_unhandled` 工具、`UNHANDLED:` 文本标记或不可用的工具调用标记——都会将整个轮次折叠为硬拒绝，打印*"I couldn't safely decide actions for that request."*。实践中，这会在合法输入上触发（最常见的是对话式问题中嵌入了带引号的列表式指令，例如*figure out why X is crashing by querying (a) sentry, (b) github, (c) posthog*），产生死胡同且无安全收益。

v0.1 中的每个终端操作都是**只读**的，因此未匹配、模糊或闲聊式的子句不构成安全风险。操作代理现在：

- 运行它*能*映射到可执行操作的每个子句，并且
- 让其他一切回退到对话助手（或直接丢弃复合请求中的闲聊式子句）。

作为此变更的一部分被移除：`ActionPlanningDecision` 上的 `denied` 字段、`enforce_plan_fail_closed_policy`、`normalize_terminal_plan`、`render_plan_denied`、`mark_unhandled` 规划器工具，以及 `UNHANDLED:` 约定。`fail_closed`、`has_unhandled_clause` 和 `turn.expected_signals` 字段也从轮次场景夹具中移除，因为 oracle 从未对它们进行断言；夹具的 `policy` 块现在携带单个 `executes_terminal_action` `boolean`（仅当预期运行 Shell 操作 AgentTool 时为 true）。

如果以后引入写入/变更操作，请用执行阶段确认策略（`tools/interactive_shell/shared/execution_policy.py`）来门控它们，**而非**操作选择拒绝。

### alpha 期间的 Shell 命令

附录 — 2026年6月27日。

**行为：** 当 OpenSRE 处于 **alpha** 阶段时，交互式 REPL 接受每个非空 Shell 命令。没有命令允许列表或硬拒绝底线。这是有意的权衡：alpha 优先考虑开发者速度而非命令沙箱化，命令以开发者权限在其机器上运行。

- 只读和变更命令、Shell 操作符（`| && ; > <`）、命令替换（`` ` ``/`$(...)`）、重定向和 heredoc 均受支持。紧凑形式如 `cat README.md;echo done` 在操作符周围无空格也能工作。可选的 `!` 前缀仍被接受。
- **唯一**剩余的非执行结果是真正为空的输入（裸 `!` 或空白），这作为输入验证被拒绝，而非作为护栏。

每个命令在 OpenSRE 的工作目录中启动一个新 Shell。因此目录变更仅在同一命令内生效；需要其他目录的每个命令请加上 `cd path && command` 前缀。在 POSIX 上，命令使用非交互式 `/bin/sh` 语法，而非加载配置的交互式 Shell、其别名或启动文件。

现有确认流程仍可用于 **`/auto`**（Off/Low/Med）和 `/trust`。在那些更严格的 `/auto` 级别下，每个 Shell 命令都会请求批准，包括看似只读的命令。仅规划模式在运行 Shell 命令前也会询问；其确认区分仅允许该命令与授权计划其余部分。Shell 在批准决策之后解释展开，因此 OpenSRE 不会基于部分解析器豁免 Shell 文本。

### `/auto` 自主性（工具类型确认）

附录 — 2026年8月。

**决策：** REPL 暴露 `/auto off|low|med|high` 作为会话级工具批准自主性。默认为 **high**（alpha：无确认）。较低级别基于 `tool_type`（`config/constants/repl_autonomy.py`）将默认 `allow` 的策略结果提升为 `ask`，而非 Shell 命令允许列表。

| 级别 | 在以下操作前询问 |
| --- | --- |
| `high` | 无 |
| `med` | 变更型代理工具（`shell`、`code_agent`、`slash`、`cli_command`、`opensre_cli`、`switch_llm_provider`、`synthetic_test`、`sentry_issue_fix`、……） |
| `low` | 同 `med` |
| `off` | 所有工具类型 |

**与 `/trust` 的交互：** `trust_mode` 仍会将 `ask` 短路为允许（跳过提示词）。非 TTY 的 `ask` 仍为失败关闭。

**仍然成立：** alpha 下没有 Shell argv 允许列表/拒绝底线。`/auto` 仅门控工具启动前是否运行现有确认 UX。

如果 alpha 后重新引入 Shell 命令允许列表，请在此执行阶段门控——绝不要在规划器中用操作选择拒绝。

### 确定性字面 `/slash` 分发（无 LLM）

附录 — 2026年6月28日。

**决策：** 用户作为字面 `/slash` 命令输入的输入会被确定性分发，**不咨询操作代理 LLM**。这取代了早先"字面 `/slash` 检测绝不能成为操作执行捷径"的措辞。

**原因：** 此前所有 REPL 轮次都经由操作代理 LLM 路由，因此当该 LLM 不可用时（无额度的供应商、认证失败、故障），每个 slash 命令都会失败——包括恢复所需的精确命令（`/login`、`/auth`、`/onboard`、`/model`）。这是死锁：你无法登录，因为登录需要你正试图修复的 LLM。键入的命令不应依赖有额度的 LLM。

**范围——保持原始关切完整的界线。** 原始 ADR 移除正则/关键词启发式规则，是因为它们*从自然语言推断意图*并与 LLM 竞争。此旁路恰恰相反：它**仅**在消息文本本身是用户逐字键入的字面 `/command` 时触发。没有推断。自由形式的自然语言（"log me in"、"show my integrations"）仍完全由操作代理选择。界线是"显式键入命令"与"自然语言"，与终端 UI 策略（`_literal_slash_command_text`）已划定的界线相同。

**工作原理。** `core/agent_harness/turns/action_driver.ActionTurnRunner` 识别字面 `/slash` 输入，并通过与显式 `!cmd` Shell 转义相同的静态 LLM 路径（`_StaticToolCallLLM`）发出确定性的 `slash_invoke` 工具调用。随后执行流经正常的 `slash_invoke` AgentTool → `dispatch_slash`，因此记录、执行策略、独占 stdin 门控、选择器和退出行为与 LLM 选择的 slash 调用完全相同——唯一区别是工具*选择*是确定性的而非 LLM 驱动的。当 `slash_invoke` 在该轮次不是可用工具时，旁路无操作（回退到 LLM 路径）。

**后果。**

- 字面 slash 命令更快、免费，且即使没有 LLM 额度也可靠。
- *以*字面 slash 开头的复合请求被分发为该单个命令（单个 `slash_invoke`）；不以 slash 开头的复合措辞（例如 `run /health and then investigate …`）不受影响，仍由 LLM 路由。没有轮次场景使用以字面 `/` 开头的提示词。
- 字面发现命令（例如 `/integrations list`）直接显示其原始输出；可选的 LLM 摘要处理仅在操作代理 LLM 可用时运行，不可用时干净降级（无摘要）。

**仍然禁止：** 对自然语言进行正则/关键词/模糊意图路由、事后重写 LLM 选择的工具调用，以及任何从非 `/` 前缀文本到操作的确定性映射。这些与 LLM 竞争，已被有充分理由地移除。

### 计划守卫（宿主强制执行）

实时计划（`update_plan`）由宿主检查，而非信任模型：

- 仅当非记账工具在其为 `in_progress` 期间返回时，步骤才为 `completed`；计划不能一诞生就完成，也不能批量勾选完成（`core/agent_harness/task_plan/completion.py`）。
- 标记为 `verifies: true` 的步骤是唯一标记 `(verify)` 的步骤。它从不豁免于该规则，且纯文本收尾步骤仅在此类步骤完成后才能免费关闭。若没有此类步骤，收尾步骤会被重置，工具结果会说明；模型添加检查或将步骤标记为 `blocked`，结果报告为未验证。
- 新标记为 `blocked` 的步骤不会结束轮次：在模型询问用户如何解决之前，结论会被拒绝（`core/agent_harness/task_plan/conclusion.py`），且在该问题排队期间不会打印"Plan ended"分解。
- 引导菜单的答案轮次若仅加载所选演示技能而不做其他事，会被拒绝一次，并提示编写计划并运行第一步；第一个演示曾这样实时停滞。
- 当没有存储含开放工作的计划时，轮次的第二个工作工具会被拒绝（`core/agent_harness/task_plan/required.py`）。拒绝会指明修复方法：在一个响应中重新发送该工具，并将计划写入列在其前。

这些是执行路径上的工具前/后钩子，与重复调用守卫是同一接缝；它们从不路由意图或重写工具调用。