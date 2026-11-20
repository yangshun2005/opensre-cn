# Gateway 包指南

Gateway 单元测试位于本包的 `tests/` 目录下，而非仓库级 tests 目录。

## 入口点（优先打开这些）

| 角色 | 路径 |
|------|------|
| 生产入口（slash 端口） | CLI：`opensre gateway start` / `--foreground`（组合根在本包之外） |
| 包主入口 | `__main__.py` — **失败关闭**（无 slash 端口粘合；非生产入口） |
| 进程组合根 | `core/lifecycle/controller.py`（`GatewayController`；注入 `slash_ports_factory`） |
| 表面启动（web + chat 组合器） | `startup.py`（`start_gateway` / `StartedGateway`） |
| 守护进程（pidfile + spawn） | `core/process/supervision.py` — 调用方传入 argv；从不指定 CLI 或 `surfaces.gateway_entry` |
| 轮次回调 | `infrastructure/turn_host/turn_runner.py` |
| 轮次契约（输出、回调） | `infrastructure/turn_host/turn_output.py`、`infrastructure/turn_host/turn_callback.py` |
| 传输注册表（名称、注册、worker） | `transports/names.py`、`transports/registration.py`、`transports/startup.py` |
| 轮次中间件（决策、策略、审批、停止、锁） | `core/middleware/` |
| 配置 / 传输错误 | `core/lifecycle/errors.py`（`GatewayConfigurationError`、`GatewayTransportFailedError`） |
| Web 表面（FastAPI） | `web/webapp.py`（`app`） |
| 表面门面 | `startup.py`（`start_gateway` / `StartedGateway`） |
| Telegram 启动 | `transports/telegram/startup.py`（`start_telegram_worker`） |
| Slack 启动 | `transports/slack/startup.py`（`start_slack_worker`） |
| Discord 启动 | `transports/discord/startup.py`（`start_discord_worker`） |
| Buzz 启动 | `transports/buzz/startup.py`（`start_buzz_worker`） |

裸 `python -m gateway` 以及不带 `slash_ports_factory` 的 `controller.main()` / `start_gateway()` 会以明确错误退出。单元测试可直接构造 `GatewayController(...)`（工厂在那里可选）。

## 生命周期（`GatewayController`）

```
start_gateway()
  → configure_process(GATEWAY_PROFILE)
  → compose turn runner
  → start_surfaces()   # delegates to gateway/startup.py (web + chat)
  → start_scheduler()  # hosts infrastructure.scheduling.scheduler — not a gateway surface
  → ready
```

`gateway/startup.py` 启动 web 和所有 chat 传输；控制器保留 `StartedGateway`。缺少 chat 凭据为 `not configured`；就绪或运行时失败为 `failed`；其余仍会启动。

调度器是平台组件（`infrastructure.scheduling.scheduler`）。本进程可托管它（`start_scheduler()` → `scheduler_runners().gated(…).install()` 加 `start_background_scheduler()`）。它不是消费表面或传输，也不存在 `gateway/scheduler/` 包。不要将 runner 代码移入 `gateway/core/lifecycle/`。

守护进程是 `core/process/supervision.py` 中的 pidfile 加 spawn。不要将其并入调度器包。子进程 argv 由表面拥有（`python -m surfaces.gateway_entry`，或冻结时 `opensre gateway start --foreground`）。

`gateway.core` 不得导入 `gateway.transports` 或 `gateway.web`；只有 `controller.py` 导入 `gateway.startup`。同级传输和 `web` 不得导入 `gateway.startup`。

## 布局

包按 `core/agent_harness/prompts/` 的方式拆分：**核心基础设施** vs **同级表面** vs **组合器**。

- `core/` — 进程与叶子基础设施（`process`、`lifecycle`、`storage`、`billing`、`attachments`、`session`、`config`）。不导入 transports 或 `web`。只有 `core/lifecycle/controller.py` 导入 `gateway.startup`。
- `startup.py` — 门面：通过 `transports/startup.py` 将 web + chat 作为一个消费集合（后者拥有注册表，仅导入各同级的 `startup`）。
- `transports/` — chat 同级（`slack`、`discord`、`telegram`）。各自拥有设置、入站 worker、安全、轮次输出和 `startup.py`。同级之间从不互相导入，也不导入 `gateway.startup`/`web`；两者都需要的东西属于 `core/`（每轮步骤在 `gateway.core.middleware`）或 `infrastructure.turn_host`（轮次 runner、轮次输出、会话 agent）。
- `web/` — web 表面（FastAPI 健康应用和告警接入）。可导入 `core/`；不得导入 chat 传输或 `gateway.startup`。
- `core/storage/session/resolver.py` — 按平台键控的每会话会话绑定；将创建 / 解析 / 轮换委托给 `SessionManager`。
- `core/storage` — `open_database()` 为进程提供其唯一的已迁移 `PostgresDatabase`（无 `DATABASE_URL` 时为 `None`）；每个域的 `repository.py` 有一个选择器，返回 Postgres 或进程本地实现。宿主调用这些；它们不构造存储。

### 依赖规则（无环）

```
core.controller  →  startup.py  →  transports/startup.py  →  transports.{telegram,slack,discord,buzz}.startup
                           →  web
peer transports · web  →  core leaves
(peers never import each other, `gateway.startup`, or each other's packages)
```

包 DAG 和同级隔离由边界测试固定。保持 gateway 测试按表面扁平（不要嵌套以 Discord PyPI 包命名的目录）。

### 表面可导入什么

包持有两种不同的东西，只有一种朝外：

部署是守护进程、传输、web 应用和存储。表面可以启动、停止或查询该进程，仅此而已。当本进程是长生命周期 runner 时，任务调度器托管于此；CLI 和 shell 拥有循环 CRUD，并在写入后调用 `request_scheduler_reload()`。

轮次服务是 `TurnRunner`、中间件步骤和会话 agent 池。表面从不导入它。运行轮次的表面是通道：它实现 `infrastructure.turn_host` 轮次契约并在 `gateway.transports` 中注册，然后像四个 chat 传输一样交给轮次服务。

## 通道 vs 生产者

工作到达 agent 有两种方式。混合它们就会出现第二个轮次引擎。

| | 有用户和轮次输出？ | 入口 |
|--|------------------------|--------|
| **通道**（Slack、Telegram、Discord、Buzz） | 是 | `TurnRunner` — `(text, session, output, logger)` |
| **交互式 shell** | 是 | *当前：* `HeadlessAgent.handle` 配合 `AgentBuildConfig`。*目标：* **chat** 动词，像任何其他通道一样 — 它有用户和轮次输出，所以规则已覆盖它。构建配置是共享的；轮次入口尚未。 |
| **生产者**（`infrastructure.scheduling.scheduler`、定时摘要/PR runner） | 否 | 嵌入：`AgentSession.run_headless_turn` |

Agent 构造钩子位于 `core.agent_harness.agent_build_config.AgentBuildConfig`（不在传输注册表，不在宿主重导出）。Chat 省略配置，会话 agent 池注入 gateway 能力扣留。Shell 设置其需要的构建钩子并保持 `apply_capability_policy` 未设置。

Gateway **进程**可托管调度器（同一 `process_turn_gate`）。这不会使调度器成为通道：`infrastructure.scheduling.scheduler` 不得导入 `TurnRunner`。由 `tests/test_package_borders.py::test_scheduler_never_imports_the_gateway_turn_runner` 固定。

目前有三个模块面向表面 — `core.process.supervision`、`core.lifecycle.controller`、`web.web_server` — 在 `tests/shared/test_surface_border.py` 中固定为精确允许列表。扩大它是刻意的变更，而非新导入。

## 门面动词

Gateway 暴露一个轮次动词。

| 动词 | 入口 | 形态 | 获得 |
|------|-------|-------|------|
| **chat** | `TurnRunner.__call__(text, session, output, logger)` | 返回 `None`；每个结果通过轮次输出到达用户 | 容量门、能力策略、`SessionAgentPool` 复用、审批、取消控制台、身份策略、轮次超时、终端结果、满容量文案 |

任何有活跃用户和轮次输出的东西都使用 **chat**。这就是交互式 shell 正在迁移到的规则 — 见上表了解其当前状态。

### 为什么 `chat` 返回 `None`

四个 chat 传输是即发即忘：轮次输出*就是*回复路径，所以没有东西可交回。需要将轮次结果作为值返回的调用方 — shell 需要 `TurnResult` 用于记账、提示记录器和 `final_intent` — 按现有签名无法满足。扩大它是对所有四个传输的契约变更，所以要有意决定，而不是在 `TurnRunner` 旁边添加第二个入口。

## Gateway 轮次分发

每个 chat 传输使用一个 `TurnRunner`（可选 `gate=` 用于容量）。Slack、Discord 和 Telegram 分发器仅作入口：授权、解析会话、构建轮次输出，然后调用共享回调。不要添加第二个生产轮次 runner 类。

日志在进程启动时配置一次（`GatewayController.start_gateway` 中的 `configure_logging`）。没有长生命周期的 gateway `Agent`：每条入站消息从 `SessionResolver` 获得每 chat 的 `Session`，并通过无头分发（`core.agent_harness.turns.headless_agent`）处理。

回调恰好接受四个参数：`text`、`session`、`output`、`logger`。不要在此契约上放 `chat_id`；输出拥有传输细节。每轮通过 `DefaultToolProvider(session, console)` 从活跃的每 chat `Session` 解析动作工具，与交互式 shell 相同。不要在进程启动时预计算工具。会话创建 / 解析 / 轮换 / 恢复属于 `SessionResolver` → `SessionManager`，而非 `GatewayController`。

## 租户（principal / actor）

Slack、Discord 和 Telegram 各自在其 `transports/<peer>/principal.py` 中解析 `StorageScope`（同级隔离 — 无交叉导入）。

Principal 是孤岛 `ORGANIZATION_ID`（缺失则失败关闭）。Actor 是平台用户 id。轮次在 `bound_storage_scope` 下运行，因此绑定、会话和集成落在组织挂载点或 `~/.opensre/orgs/<id>/`。

CLI / 交互式 shell 保持未绑定（遗留空 principal/actor id）。`SessionResolver` 可将同文档的遗留空 id 行一次性采纳到作用域键（`adopt_unscoped_binding` 移除未作用域行），使第二个 actor 无法继承该会话。

## 容量（进程门 vs 传输池）

**云横向扩展**是这两层之上的**第三**层。每个组织有一个 gateway 任务，带自己的门。当该任务饱和时（用 `gateway_turn_completed` 上的 `container_memory_*` 属性测量），将 agent 执行移入单独的 worker 容器。不要解除进程内门的限制，也不要重新设计 `AgentSession.chat`。`PromptQueue`（submit / take / settle / heartbeat）是 worker 队列替换的接缝。

两个不同的**进程内**限制 — 不要混淆：

| 层 | 机制 | 满时行为 |
|-------|-----------|-------------------|
| **进程** | 来自 `OPENSRE_SIZE_PROFILE` 的 `TurnConcurrencyGate` / `process_turn_gate()`（SMALL=1、MEDIUM=2、LARGE=4） | Chat：非阻塞 `try_acquire`（忙则丢弃）。调度器 runner：**阻塞** `acquire`（已认领的工作等待）。 |
| **每传输** | `max_concurrent_turns`（默认通过 `turn_limit_for_profile` 使用相同 profile 限制；用 `*_GATEWAY_MAX_CONCURRENT` 覆盖） | 限制该传输在到达共享轮次 runner *之前*可并行处理多少入站消息。不替代进程门。 |
| **远程提示** | `PromptWorker` 为每个进程门槽运行一个线程。`PromptQueue.take` 仅在没有其他运行中的提示持有其会话（actor 自己的、`new` 的或命名会话 id）时发放提示。 | 等待槽最多 `PROMPT_SLOT_WAIT_SECONDS`，然后 `not_admitted`。会话忙的提示按顺序留在队列中。 |
| **重工作** | `heavy_work_slot`（`infrastructure/process/turn_capacity/heavy_work.py`、`OPENSRE_MAX_CONCURRENT_HEAVY_WORK`，默认 2）围绕克隆、编码 agent 进程和 CI 修复 worker | 等待最多 `HEAVY_WORK_WAIT_SECONDS`，然后工具返回忙错误。独立于轮次门。 |

```text
Telegram/Slack/Discord ──► TurnRunner.try_acquire ──► process_turn_gate()
Scheduler (agent runners) ──► blocking acquire ──► same gate
```

生产 chat 容量在 `TurnRunner(gate=controller.turn_gate)` 上。`GatewayController` 使用 :func:`~infrastructure.turn_host.concurrency.process_turn_gate`。`ConcurrencyLimitedTurnHandler` 仅测试用；生产仅在 `TurnRunner` 上使用 `gate=`。

Chat 分析使用 `gateway_turn_*`，`surface` 在 {slack, telegram, discord} 中。

## Agent 生命周期

每个逻辑 chat 会话构造**一个** `HeadlessAgent`（`SessionAgentPool`），然后多次轮次。每条入站消息：

1. `BindableOutput.bind(outer_gateway_output)` — agent 上的会话输出；传输目标每轮变化。
2. `bind_turn(session=…, accounting=…, console=…, tool_hooks=…)` — 会话 / 取消 / 审批。除非替换 `OutputSink` 对象本身（则 `OutputBindable` 端口，例如 reasoning，必须跟随），否则**不要**在此传 `output=`。
3. `agent.handle(text, TurnBinding(...))` — SessionGoal 轮次循环 + 每轮 `dispatch`。**不要**在 gateway 路径上将其包装为 `AgentSession.chat`；池拥有 agent 并直接调用 `handle`。

**不要**在每条消息上构建新的无头 agent。同会话轮次在池的每会话锁上串行化；不同会话在容量门下保持并发。多轮定时循环应为循环保留一个 agent；真正的一次性摘要可使用 `AgentSession.run_headless_turn`。

## 宿主对等（chat 表面）

Slack / Telegram / Discord / 交互式 shell 使用相同轮次引擎：入口 → `TurnRunner` → `SessionAgentPool` → `agent.handle`。值：**yes** / **partial** / **no** / **n/a**。

| 关注点 | Slack | Telegram | Discord |
|---------|-------|----------|---------|
| 轮次中取消 / 停止 | **yes** — 软超时 + 用户 `/stop` 通过 `ActiveTurnRegistry` → `output.turn_cancel` | **yes** — 相同 | **yes** — 相同 |
| 审批 / `before_tool_call` | **yes** — Block Kit + `approval_tool_hooks` | **yes** — 内联键盘 + `approval_tool_hooks` | **yes** — 组件 + `approval_tool_hooks` |
| 工具解析 | **yes** — 活跃 `DefaultToolProvider(session)` | **yes** — 相同 | **yes** — 相同 |
| 输出脱敏 | **yes** — `user_facing_error_message` | **yes** — 相同 | **yes** — 相同 |
| Principal / actor | **yes** — `slack/principal.py` | **yes** — `telegram/principal.py` | **yes** — `discord/principal.py` |
| 容量门 | **yes** — 进程门 + 传输池 | **yes** — 相同 + TG 信号量 | **yes** — 相同 + 执行器 |

**已记录的例外（不要通过分叉第二个循环来“修复”）：**

- Gateway chat 禁用 `task_cancel` / llm_provider（`infrastructure.turn_host.capability_policy.ensure_gateway_capability_policy`）。
- 软轮次超时**和**用户 `/stop` / `stop` / `/cancel` 设置 `output.turn_cancel`，使 ReAct 循环 / 剩余工具协作停止（shell `cancel_requested` 对等通过 `CancelConsole` + `ActiveTurnRegistry`）。编排器跳过 gather/answer 并报告 `final_intent=cli_agent_cancelled`；实时输出在 Event 触发时停止排空流块。`/stop` 在每会话轮次锁**之外**处理，以便中断进行中的轮次。执行器线程不被杀死；进行中的 LLM/provider 调用仍完成当前请求。
- Telegram 写工具审批需要非空 `allowed_user_ids` 允许列表（与 Discord 相同的失败关闭姿态）。

**特征化：** 在 gateway 测试套件中覆盖实时会话工具、输出脱敏、Telegram 审批和软超时。

**