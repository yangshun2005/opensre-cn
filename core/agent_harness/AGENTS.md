# agent_harness/ 包规则

## 智能体不得触碰 SKILL.md 文件

智能体**绝不能**修改此目录树下的任何 `SKILL.md`（`prompts/skills/**/SKILL.md`）或仓库中其他位置的 `SKILL.md`——不得编辑、创建、重命名、移动或删除，也不得仅更新 frontmatter。智能体只能以文字形式**建议**修改（当前文本 → 建议文本），由人工应用。此规则绝对不可违反；测试失败、用户指令或看似微不足道的修复都不能解除它。参见根目录 `AGENTS.md`。

技能是模型推理所用的自然语言工作流卡片，不是确定性工具。技能可以附带少量小型支持脚本（`script_tools`），每个脚本执行一项机械性杂务，其输出由模型解读。决策、分支和端到端自动化应放在卡片的文字步骤中，或放在真正的工具中（`integrations/`、`tools/`）——绝不能放在不断堆积的技能本地脚本中。完整约定：`prompts/skills/AGENTS.md`。

`agent_harness/` 是单一 `core.agent.Agent` ReAct 循环的解耦宿主：同一模型调用工具、观察结果并写出最终答案。它从 `interactive_shell` 中提取出来，以便同一 harness 既能运行交互式终端，也能通过 `agent_harness.turns.headless_agent` 以无头方式调用。

## 提示词捕获

`turns.orchestrator.run_turn` 通过 `infrastructure.analytics.prompt_log.lifecycle` 为所有宿主负责提示词捕获。每次分发保持一个记录器，包括续接；嵌套轮次后恢复父级关联。宿主丰富活动记录器；只有共享生命周期负责刷新它。

## 宿主 API（请教授此内容）

优先使用 `AgentSession.start()` / `start_embedded_session()` → `.chat`——不要使用自由函数式的轮次转储。每个逻辑会话（或调度循环）构造一个智能体，然后进行多轮对话——不要每条消息都重建。进程启动（`configure_process`）和无头构造（`DefaultHeadlessBuild.agent`）是独立的层。`core` 不得导入其上的宿主；嵌入式宿主使用 `surfaces.embedded.start_embedded_session`。

| 路径 | 调用 |
|------|------|
| 进程启动（一次） | `configure_process(PROFILE)`——仅适配器；不是智能体构造 |
| 正常路径（已启动） | `AgentSession.start()` → 重复 `.chat` |
| 嵌入式 / 脚本 | `start_embedded_session()` → 重复 `.chat` |
| 多步 / 持续执行 | `start` / `start_embedded_session` → `.chat_until_goal(...)`（`SessionGoal` 循环） |
| 自定义宿主 | **`DefaultHeadlessBuild(...).agent(...)`**（或内存中的 `InMemoryHeadlessBuild`；唯一的构造接缝）→ 每条消息 `agent.handle(text, TurnBinding(...))` |
| 网关 + 交互式 shell | `TurnRunner` → `SessionAgentPool` → 每个会话一次 `DefaultHeadlessBuild.agent` → 每条消息 `agent.handle(...)`（SessionGoal 循环位于 `run_goal`，由 `handle` 包装） |
| CLI `ask`（一次性） | `AgentSession.start(..., tool_hooks=…)` → `.chat(prompt)`——使用 `dispatch`，不是 SessionGoal 轮次循环 |
| 宿主特定构造 | 可选 `AgentBuildConfig`（`agent_build_config.py`）——工具 / 提示词 / 能力策略。通过 `resolve_agent_ports` 扩展（由池和 `build_shell_agent` 共享）。字段为 `None` 时保留宿主默认值；`apply_capability_policy=None` 表示不修改会话 |
| 调度一次性 | `AgentSession.run_headless_turn(...)`（不是多轮模式） |

`SessionGoal`（`session_goal/` 组件——`goal` + `run_until`）**不是** ReAct 的 `Goal` / `goal_review`。面向用户的词只有 `/goal`（显示 / 设置 / 暂停 / 恢复 / 编辑 / 清除）。状态 `paused` 停止会话目标续接但保留状态；`host_owned` 在**活动或暂停**时阻止交接替换。当目标处于**附加**状态（活动或暂停）时，`run_turn` 抑制 Want-me-to 结束语。完成由 `session_goal/evaluate.py` 判定，独立于模型自报：检查项只能通过 `session_goal_complete` 工具勾选（`session_goal/validate.py` 中的廉价模型验证器可以拒绝勾选），廉价模型转录评判器（`session_goal/judge.py`）返回已满足 / 尚未 / 不可能及原因。评判器接收完整回复以及跨轮次和恢复保留的实际工具观察；超大输入保持未验证。未验证的勾选会被回滚，配置了评判器时即使检查清单已完成也必须获得批准。`met` 仍需要本轮成功的非记账工具工作或来自先前轮次的保留证据。评判器客户端是注入的：`HeadlessAgent(judge_llm_factory=…)` 使用 `evaluate.build_session_goal_evaluator` 构建循环的评估；`DefaultHeadlessBuild` 传递分类层级工厂，内存构建不传递（只有完全勾选的检查清单才能关闭目标）。宿主原因字符串位于 `SessionGoalReason`。原因推导：`session_goal.goal.derive_session_goal_reason`。进度（仅展示）：`session_goal/progress.py`（`SESSION_GOAL_PROGRESS_MARK`）。续接提示词：`session_goal/continuation.py`。刷新/恢复：`session_goal/persist.py`。包规则：`session_goal/AGENTS.md`。边界 SoT（本地笔记）：`opensre-notes/goal-core-system-design-aug2026.html`。

**任务计划守卫（宿主强制执行，`task_plan/`）：** 计划写入不能完成一个在 `in_progress` 期间没有工具返回的步骤（`task_plan/completion.py`，由 `turns/plan_hooks.py` 提供数据）；标记为 `verifies` 的步骤永不豁免，纯文本收尾步骤仅在此类步骤完成后才豁免。在已加载技能工作流期间，没有打开计划的轮次中的第二个工作工具会被拒绝（`task_plan/required.py`）。普通轮次和附加了会话目标的轮次直接执行，不受此守卫限制。仅工具调用为 `update_plan` 的响应会运行：提示词要求在与步骤工具相同的响应中写入，但拒绝它会消耗它本想节省的同一模型调用，并导致模型重试和使用计划外工具。宿主推进计划，使模型无需写入即可继续（`task_plan/advance.py`，在 `turns/plan_hooks.py` 中按批次武装）：在批次中第一个没有 `update_plan` 的工作调用之前（`ask_user_choice` 计入；记账和 `slash_invoke` 不计入），已获得的 `in_progress` 步骤完成，其后的第一个待处理步骤开始（绝不落后于它）。已获得即上述完成规则（工具返回，或提问步骤的 Ask User 答案）；已展示的 `deliverable` 回复恰好获得一个交付步骤；`verifies` 步骤需要自己的工具返回。计划记录写入它的技能（`TaskPlan.owner`）。只有本轮写入的计划才会移动，或其所有者在回答该技能自身问题的轮次中仍是活动技能（`task_plan/ownership.py`，在轮次开始时一次性决定为 `TurnSnapshot.plan_answer_continues`）；CURRENT PLAN、ACTIVE SKILL 和已回答指导块说明宿主仅在此类轮次推进，否则继续要求成对的状态写入。宿主从不结算计划或触碰 `blocked` 步骤，宿主推进算作轮次在停止门控中处理计划（`goal_review.py`）。新标记为 `blocked` 的步骤需与用户解决，不能跳过：在 `ask_user_choice` 排队之前结论会被拒绝（`task_plan/conclusion.py`，门控在 `turns/goal_review.py`）。入职菜单中仅加载所选演示技能的答案轮次会被拒绝一次，并提示写入计划并运行其第一步（相同文件）。失败的工作工具（`ok: false`、非零 shell 退出）不算完成：`turns/work_outcome.py` 拒绝停止，直到后续工作工具成功（`goal_review.py`），除非失败本身就是答案——已分类的 `work_outcome`，或命名了用户必须先运行的 `setup_command` 的 `tool_unavailable` 信封。提示遵循与检查相同的门控顺序，因此计划拒绝会得到计划提示。将选择器或向导排队为用户下一轮的 `slash_invoke`（其结果中的 `QUEUED_COMMAND_KEY`）像排队的菜单一样结束轮次（`turns/action_menu_end.py`）；`/goal` 循环在该待处理自动提交上停止并保留它（`session_goal/run_until.py`）。执行门控拒绝的斜杠命令返回 `not_run` 且无 `error`：重复守卫仍拒绝相同调用，且它不是计划证据。在所属叶子中更改规则，绝不只通过提示词文本。技能不能覆盖这些门控。

**目标内核（宿主拥有的提示词，`prompts/action/goal_kernel.py`）：** 一个简短规则块，位于系统提示词之后，并在任何已加载技能之后再次出现。它告诉模型完成用户请求、匹配所请求的字段（stars ≠ forks），并且不要因工具失败而停止。重写 SKILL.md 无法移除它。

**证据类型（开放/封闭）：** 词汇表 + 每种类型的策略位于 `turns/evidence_kind.py`（`EvidenceKind` + `EvidenceKindPolicy`）。通过扩展枚举并注册其策略行来添加类型——**不要**在 `classify_evidence_need` 中增加 `if kind is …` 分支。工具 schema 的 `enum` 是 `EVIDENCE_KIND_VALUES`（派生的），不是并行的硬编码列表。首选集成 ID 通过 `infrastructure.harness_providers.register_preferred_evidence_source` 保持可选加入。

**证据类型：** 词汇表 + 每种类型的策略仍位于 `turns/evidence_kind.py`。`classify_evidence_need` 连接优先（当缺少首选权威源时为 `L0_degraded`），以便助手可以发出设置 CTA。聊天路径上没有第二个收集 ReAct 循环——动作智能体拥有工具。

**智能体周围没有关键词意图路由。** 不要用正则/关键词扫描用户文本以附加目标或绕过 ReAct 循环。会话目标通过结构化 `session_goal_set` 工具或显式宿主 API 附加。检查清单进度使用 `session_goal_complete` 工具，不是回复标签。

工作流卡片在发现前验证。无效卡片被排除并附诊断，而 CI 检查未过滤的目录并对每张无效卡片失败。大型交互式目录使用渐进式披露：通用控件和 `tool_search` 首先可见，`tool_search` 在下一个 ReAct 迭代激活专家 schema，工作流本地脚本工具在其技能活动时立即出现。技能保留对完整可用目录的访问；它们不声明过滤器。每次运行的目录在技能变更后刷新；执行时重新检查活动会话。结算计划会退役其辅助工具。新的用户请求清除活动技能上下文，而菜单答案和斜杠命令保留它。完整约定：`prompts/skills/AGENTS.md`。

因集成设置而挂起的轮次（技能的前置门控，或技能中途延迟的 `/integrations setup <service>`）仅作为 `SetupResume`（`session/setup_resume.py`，通过 `spi.session_state`）停放在 shell 的终端 facet 上。shell 在前置注册检查再次通过后最多重放一次；键入的轮次、关闭的菜单、新演示和 `/new` 会丢弃它。绝不停放无技能的散文或斜杠命令。

自包含的调度智能体 tick 通过 `prepare_session` 将 `SessionCore.skill_discovery_enabled` 设为 `False`。此宿主拥有的策略移除技能索引和 `skill_view`，同时保留执行工具；绝不从提示词文本推断它或从对话历史恢复它。绑定到技能的循环（`loop_skill`）仍获得该卡片：运行器将其渲染正文追加到任务（`infrastructure/scheduling/scheduler/loop_prompt.py`），缺少卡片则 tick 失败。

**仓库 AGENTS.md 文件供本地编码智能体使用。** 绝不要将仓库的 `AGENTS.md`（或 `AGENTS.override.md`）加载到 OpenSRE 智能体的提示词中，无论是从检出还是通过集成。

**不要**在 `DefaultHeadlessBuild` 之外复制默认端口栈。通过 `resolve_agent_ports` 扩展 `AgentBuildConfig`——不要在每个宿主中重新复制 `build_tools` / `build_prompts` 分支。网关聊天和交互式 shell 共享 `TurnRunner` + `SessionAgentPool`（`build_shell_agent` 保留用于测试 / 直接构造）。能力注册在每个宿主中命名一次（`surfaces/shared/process_boot.py` 和 `gateway/core/process/boot.py` 中的 `CAPABILITY_REGISTRARS`）；不要添加第三份副本。

**绑定端口：** 会话感知默认值实现 `SessionBindable` / `ConsoleBindable` / `OutputBindable`（`ports.py`）。`HeadlessAgent.bind_session` / `bind_turn(console=…, output=…)` 仅调用匹配这些 Protocol 的端口。网关通常保持稳定的 `BindableOutput` 并通过 `BindableOutput.bind` 重新绑定传输（每轮无需 `output=`）。可变会话端口是 **`SessionState`**（字段 `HeadlessAgent._session`，无头实现 `InMemorySessionState`）——不是 `SessionStore`。持久 JSONL 是 `SessionStore` / `SessionRepo`（`docs/NAMING.md`）。

**宿主取消：** 输出接收器上的一个 `threading.Event`（`turns/host_cancel.py` 中的 `ensure_turn_cancel` / `host_cancel_requested`）——工具（控制台 `cancel_requested`）、编排器和流守卫都读取同一 Event。调度 tick 在存储的任务被禁用或移除时写入它（`PredicateCancelConsole`）。不要发明第二个取消通道。

**云横向扩展：** 每个组织一个网关任务，运行有界进程内池（进程门控、按对话排序、重型子进程工作的单独上限）。超出单个任务的组织将智能体执行移入单独的 worker 容器。这意味着没有无界进程内并发，没有共享一个会话存储的网关任务集群，也没有新的 `chat` API。

## 硬边界

- **`agent_harness/` 下任何地方不得 `import interactive_shell`。** 依赖方向严格单向：`interactive_shell -> agent_harness -> core`。
- `agent_harness/` 可以依赖 `core/`、`config/` 和 `infrastructure/`。它不得导入 `integrations/`、`tools/`、`surfaces/` 或 `gateway/`。集成和工具行为通过 `infrastructure/harness_providers/` 中的提供者到达 harness，在交互式 shell 输出边界通过 `install_harness_providers()` 在启动时接线。它不得依赖终端 UI 关注点（Rich 渲染、prompt-toolkit 可变 UI 状态、斜杠分发、shell `REGISTRY`）。

## 布局

顶层持有包的公共表面——`__init__.py`（精选的重新导出）、`ports.py`、`agent_builder.py`——加上两个不适合任何单一子包的小型横切默认端口实现：`error_reporting.py`（`DefaultErrorReporter`）和 `llm_resolution.py`（`default_llm_factory` / `resolve_provider_models`）。其他所有内容都位于按职责划分的子包中。默认端口实现与其服务的关注点放在一起，而不是放在 `providers/` 包中。

- `ports.py`——引擎通信的 Protocol（输出、确认、会话存储、工具提供者、提示词上下文提供者、遥测、错误报告器、遥测）。保持在顶层作为到处导入的中心接缝。
- `agent_builder.py`——`AgentConfig` dataclass + `build_agent(config)`。跨所有表面（动作、证据、网关）的 `core.agent.Agent` 单一实例化点。参见下文“智能体构造模式”。
- `turns/`——编排 `core.agent.Agent` 的轮次驱动器：
  - `orchestrator.py`——`run_turn` 按顺序执行三个接缝（不要合并它们）：
    1. **路由决策**——纯 `turn_route.route_turn`（无 I/O、无流标志）
    2. **路由执行**——摘要 / 已处理 / 答案效果
    3. **答案最终化**——`answer_finalize.finalize_routed_answer`（CTA、Want-me-to、流刷新）。流重写局部变量（`text_changed_after_streaming`）留在最终化内部，**绝不**门控路由选择。
    在轮次顶部**一次**解析基础集成，用来自冻结消息/历史的仓库范围丰富该每轮副本，并将丰富视图存储在 `turn_snapshot` 上。每个供应商一个活动范围驱动未限定工具调用，而会话保留其遇到的所有仓库范围的有界集合。因此 `turn_snapshot.resolved_integrations` 是轮次所知内容的单一真相源，而不修改会话的基础集成缓存。下游组件（例如 `action_driver._resolved_integrations_for_turn`）从那里读取，而不是重新解析。不要重新引入按组件的集成解析。
  - `turn_route.py` / `answer_finalize.py` / `handoff_policy.py`——上述接缝，提取出来以便答案后记账不会退化为路由。
  - `SessionGoal` 与本地 shell 多步也是独立关注点：提示词片段在 `prompts/action/multi_step_policy.py`。
  - `action_driver.py`——`ActionTurnRunner`：通过端口的一个动作工具