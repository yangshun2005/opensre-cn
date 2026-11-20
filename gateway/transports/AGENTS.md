# gateway/transports/ — 聊天对等包

每个入站聊天平台一个包。对等关系：**互不导入**。

用 API 框架的术语来说，传输层是入口适配器：平台入站负责鉴权、解析会话、构建轮次输出，并调用轮次运行器。它们实现的轮次契约是 `infrastructure.turn_host.turn_callback` / `infrastructure.turn_host.turn_output`；注册的注册表是 `gateway.transports.startup`；组合的共享每轮步骤是 `gateway.core.middleware`；启动它们的门面是 `gateway/startup.py`。

| 包 | 启动函数 |
|---------|--------|
| `slack/` | `startup.start_slack_worker` |
| `discord/` | `startup.start_discord_worker` |
| `telegram/` | `startup.start_telegram_worker` |
| `buzz/` | `startup.start_buzz_worker` |

注册表和工作线程启停循环位于本包自己的 `startup.py` —— 这是此处唯一允许导入对等包的模块（仅限其 `startup` 子模块）。Web + 聊天的组合保留在 `gateway/startup.py`。`GatewayController` 只持有不透明的 `StartedGateway`。传输层特有的工作（设置加载、Discord 就绪等待）保留在各包的 `startup.py` 中。

每个包各自拥有设置、入站工作线程、安全、轮次输出和 `startup.py`。两个传输层都需要的任何内容应放在 `gateway.core`（每轮步骤在 `gateway.core.middleware`；轮次输出/回调在 `infrastructure.turn_host`）。各传输层的关注点完整性由 `gateway/tests/test_transport_contract.py` 固定（含其已知缺口清单）。

对等包不得导入 `gateway.startup` 或 `gateway.web`。

对等导入 DAG 由 `gateway/tests/test_package_borders.py` 固定。
Discord↔Slack 隔离附加测试：
`gateway/tests/discord/test_transport_borders.py`。