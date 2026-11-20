# gateway/core/ — 进程与叶子基础设施

每个界面都会用到的 Gateway 核心机制。**不得**导入 `gateway.transports.*` 或 `gateway.web`。只有 `lifecycle/controller.py` 会启动这些界面（通过 `gateway.startup`）。由 `gateway/tests/test_package_borders.py` 固定约束。

| 包 | 职责 |
|---------|------|
| `lifecycle/` | 组合根（`controller`）、凭据水合、Gateway 错误 |
| `process/` | 守护进程、轮询线程、就绪状态 |
| `middleware/` | 每个传输层都会运行的逐轮步骤（入站决策、身份策略、审批、关注、锁） |
| `storage/` | 会话绑定与调查、事件与反馈存储 |
| `billing/` | 额度客户端 |
| `attachments/` | 附件辅助函数 |
| `session/` | Gateway 聊天上下文辅助函数 |
| `config/` | 日志 / Gateway 配置辅助函数 |
| `infrastructure.turn_host` | 轮次处理器、会话代理池、可绑定输出、取消控制台、容量 |

传输层和 `web/` 可以导入上述包。聊天传输层代码不属于 `gateway/core/`。

## 谁可以驱动代理

只有 `infrastructure.turn_host` 会构建端口、绑定轮次、运行或刷新会话，以及格式化目标进度。其他 `gateway/` 代码可以导入 harness **类型**（`SessionCore`、`OutputSink`、`SlashPortsFactory`、`SessionGoal`）用于签名。自行执行轮次的传输层就是第二个处理器。

由 `gateway/tests/test_harness_behaviour_border.py` 固定约束，其允许列表只能缩小。`web/` 目前在该列表中，因为 `POST /investigate` 直接嵌入代理，因此无法获得任何 turn-host 保证（代理复用、审批 Hook、取消控制台、能力策略）。

## 获取容量槽位

此进程中的每个轮次——聊天、`POST /investigate`、调查 worker、定时运行——都从同一个 `process_turn_gate()` 获取一个许可。从 `infrastructure.process.turn_capacity` 中选择策略；不要手动配对 `acquire`/`release`：

- `turn_slot(gate)` — 满时**丢弃**。适用于持有连接或对话的调用方：它返回 `False`，由调用方应答（聊天最终返回 `AT_CAPACITY_MESSAGE`，web 将其作为 503 响应体返回）。
- `queued_turn_slot(gate)` — **等待**槽位。适用于已从队列中领取、无法被告知重试的工作。

缺少 `finally` 会泄漏许可，而泄漏的许可意味着进程永远回答“at capacity”。

## 进程启动与生命周期

共享进程设置（env → Sentry → harness 适配器 → 能力警告 → LLM 预加载）是 :func:`gateway.core.process.boot.configure_process`，配合 ``GATEWAY_PROFILE``。在日志和凭据之后，`GatewayController.start_gateway` 配置进程，构建**一个** `TurnRunner(gate=…)`，然后 `start_surfaces()` 和 `start_scheduler()`。不要包装 turn runner。不要在 controller 中重复进程启动。托管是一个薄调用：`scheduler_runners().gated(turn_gate).install()` 然后 :func:`infrastructure.scheduling.scheduler.runner.start_background_scheduler`。重载是 :func:`infrastructure.scheduling.scheduler.reload_signal.request_scheduler_reload`（shell/CLI 写入方）；controller 只轮询并重新同步。

进程启动只有一个入口点：:func:`gateway.core.process.boot.configure_process`，配合 ``GATEWAY_PROFILE``——即 :mod:`infrastructure.process.boot` 中的共享顺序加上此主机的 ``CAPABILITY_REGISTRARS``。不要在其周围添加另一个包装器。