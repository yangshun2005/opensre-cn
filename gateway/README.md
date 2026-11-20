# OpenSRE 消息网关

面向聊天平台的独立入站消息网关：Telegram 私聊文本通过长轮询，Slack 提及/私信通过 **Socket Mode**（默认）或 **Events API HTTP**，Discord 通过 Gateway WebSocket，Buzz 提及通过中继轮询。

网关是独立的接入面。传输层接收注入的轮次运行器；agent 启动和集成加载通过共享 harness 运行，而非传输层专属代码。

## 入口点

| 目标 | 文件 / 符号 | 启动方式 |
|---------------|---------------|-------------------|
| **生产入口** | CLI 组合根（`gateway/` 之外） | `opensre gateway start` / `--foreground`（注入 slash 端口） |
| **包 main** | `gateway/__main__.py` → `main()` | 失败关闭——无 slash 端口胶水 |
| **进程组合根** | `gateway/core/lifecycle/controller.py` → `GatewayController` | 由 CLI 注入 `slash_ports_factory`；裸 `controller.main` 失败关闭 |
| **守护进程（pidfile + spawn）** | `gateway/core/process/supervision.py` | 由 CLI `gateway start/stop/status` 使用（pidfile + `components.json`）；调用方传入 argv |
| **Web 接入面（仅 Web 任务）** | `gateway/web/webapp.py` → `app` | `uvicorn gateway.web.webapp:app`（Docker 中 `MODE=web`） |
| **接入面启动** | `gateway/startup.py` → `start_gateway` / `StartedGateway` | 由 `GatewayController.start_surfaces` 调用 |
| **聊天传输注册表** | `gateway/transports/startup.py` → `TRANSPORTS` / `start_transports` | 由 `start_gateway` 使用 |
| **Telegram 传输** | `gateway/transports/telegram/startup.py` → `start_telegram_worker` | 通过启动注册表 |
| **Slack 传输** | `gateway/transports/slack/startup.py` → `start_slack_worker` | 通过启动注册表 |
| **Discord 传输** | `gateway/transports/discord/startup.py` → `start_discord_worker` | 通过启动注册表（含就绪等待） |
| **单消息轮次** | `infrastructure/turn_host/turn_runner.py` → `TurnRunner` | 作为 agent 回调注入聊天传输 |

```text
opensre gateway start
        │
        ▼
gateway.core.process.supervision.start_gateway_daemon
        │  spawns surface-owned argv (see surfaces.shared.gateway_entrypoint):
        │    venv:   python -m surfaces.gateway_entry
        │    frozen: opensre gateway start --foreground
        ▼
surfaces/gateway_entry.py  (or Click foreground → same composition root)
        │  injects headless slash ports
        ▼
gateway.core.lifecycle.controller.GatewayController.start_gateway
        ├── start_surfaces()  →  gateway.startup.start_gateway
        │     ├── web/web_server  →  web/webapp:app
        │     └── transports/startup.start_transports
        │           (telegram / slack / discord / buzz startup)
        └── start_scheduler()   # hosts infrastructure.scheduling.scheduler; not a gateway surface
```

目录结构：`core/`（运行时、存储等）、`startup.py`（接入面组合器）、`transports/`（slack、discord、telegram 同级）、`web/`（Web 接入面）。见 `AGENTS.md`。

## 各部分如何配合（接入面、网关、集成）

三样容易混淆的东西。

接入面是人跟 agent 对话的方式（消息进，回答出）。目前有三个：交互式 shell（`surfaces/interactive_shell`）、CLI 一次性调用（`surfaces/cli`）和网关（`gateway/`，聊天应用）。

网关是常驻进程，把聊天应用连接到 agent。它支持 Telegram（长轮询）、Slack（Socket Mode 或 Events API HTTP）、Discord（Gateway WebSocket）和 Buzz（中继轮询）。Slack 的两种入站模式共享同一轮次栈，仅载荷到达方式不同（`gateway/transports/slack/transport/`）。

集成和工具是出站/队友侧：agent 在平台上读取和发帖。Slack 的共享客户端是 `integrations/slack/web_client.py`。常用 Slack 工具：`slack_send_message`（webhook）、`slack_reply_message`（bot token，任意频道）、`slack_read_messages`（历史/线程）、`slack_list_team_members`（成员名册），以及 `integrations/slack/tools/` 下的搜索/加入/表情回应辅助工具。OAuth 权限范围见 `docs/integrations/messaging/slack.mdx`。Telegram 有 `telegram_send_message`；Discord 目前是网关聊天加投递（见 `docs/integrations/messaging/discord.mdx`）。

每个平台的入站和出站相互独立：

| | 入站（人 → agent） | 出站 / 队友工具 |
|---|---|---|
| **Telegram** | 是 — `gateway/transports/telegram/` | 是 — 集成 + `telegram_send_message` |
| **Slack** | 是 — `gateway/transports/slack/`（默认 Socket Mode；可选 Events API HTTP；每个线程是一个会话） | 是 — webhook + bot token 工具 |
| **Discord** | 是 — `gateway/transports/discord/`（私信、提及、线程） | 投递 + slash 注册 |
| **Buzz** | 是 — `gateway/transports/buzz/`（提及和频道后续消息） | 是 — 集成 + `buzz_send_message` |

**所有接入面共用一个核心。** Shell、CLI 和网关传输都把消息交给同一处：会话作用域的 `HeadlessAgent`（通过 `TurnRunner` 调用 `agent.handle(...)`）。区别仅在于*如何接收输入和发送输出*——agent 的思考方式从不改变。

## 快速开始

```bash
# Allow your Telegram user id (from @userinfobot)
uv run opensre messaging allow -p telegram -u 123456789

# Allow your Slack member id (profile → Copy member ID; see below)
uv run opensre messaging allow -p slack -u U0123ABCD

# Start the gateway daemon (web + chat). The process also hosts infrastructure.scheduling.scheduler.
uv run opensre gateway start
```

**查找你的 Slack 用户 ID（成员 ID）：**

1. 在 Slack 应用中打开你的个人资料（头像/姓名）。
2. 点击 **View as** / 资料操作旁的 **⋯**（更多）。
3. 选择 **Copy member ID**——该值以 `U…` 开头，正是 `SLACK_ALLOWED_USERS` / `messaging allow -p slack -u …` 所需的。
4. **不要**使用 `@显示名`（如 `@Yauhen`）；只有成员 ID 是稳定的。

两种传输加载配置的方式相同：token 优先从环境变量读取，集成存储作为回退；允许用户优先从集成存储读取（由 `opensre messaging allow` 写入），`*_ALLOWED_USERS` 环境变量作为回退。

在 Telegram 私信你的 bot，在 Slack 中提及/私信它，在 Discord 中聊天，或在 Buzz 中提及 agent（应用设置见 `docs/integrations/messaging/`）。Slack Socket Mode 无需公网 URL；Events API HTTP 需要可达 URL 和 `SLACK_SIGNING_SECRET`。

## 环境变量

| 变量 | 用途 |
|----------|---------|
| `TELEGRAM_BOT_TOKEN` | Telegram bot token |
| `TELEGRAM_ALLOWED_USERS` | 逗号分隔的 Telegram 用户 ID |
| `TELEGRAM_GATEWAY_MAX_CONCURRENT` | 跨聊天的并行轮次（默认取 `OPENSRE_SIZE_PROFILE` 限制：SMALL=1、MEDIUM=2、LARGE=4） |
| `SLACK_BOT_TOKEN` | Slack bot token（`xoxb-…`） |
| `SLACK_APP_TOKEN` | Socket Mode 的 Slack 应用级 token（`xapp-…`） |
| `SLACK_SIGNING_SECRET` | Events API HTTP 请求验证的 Slack 签名密钥 |
| `SLACK_GATEWAY_INBOUND_TRANSPORT` | `socket_mode`（默认）或 `events_api_http` |
| `SLACK_GATEWAY_HTTP_PORT` | Events API HTTP 监听端口（默认 `3000`） |
| `SLACK_GATEWAY_ALLOW_LOCAL_DEDUP` | `1` 允许 Events API 的进程内事件去重（仅单副本） |
| `SLACK_ALLOWED_USERS` | 逗号分隔的 Slack 用户 ID（除非开放工作区，否则必填） |
| `SLACK_ALLOW_OPEN_WORKSPACE` | `1` 允许任意工作区成员（仅内部试用） |
| `DISCORD_BOT_TOKEN` | Discord bot token |
| `DISCORD_ALLOWED_USERS` | 逗号分隔的 Discord 用户 snowflake |
| `DISCORD_ALLOW_OPEN_GUILD` | `1` 允许任意公会成员（仅内部试用） |
| `BUZZ_PRIVATE_KEY` | agent 身份的 Nostr 私钥 |
| `BUZZ_RELAY_URL` | Buzz 中继 URL（默认 `http://localhost:3000`） |
| `BUZZ_ALLOWED_PUBKEYS` | 逗号分隔的 64 字符十六进制公钥，允许与 agent 对话 |
| `BUZZ_GATEWAY_MAX_CONCURRENT` | Buzz 并行轮次（默认 4） |

通过 `opensre messaging pair` 配对使用与网关相同的集成存储策略。

## 添加聊天平台

消息处理器**与传输无关**——它接收 `(text, session, output, logger)`，对任何平台一无所知。添加平台**不需要**改动 agent、提示词或工具。你只需添加一个包，包含 `gateway/transports/telegram/` 和 `gateway/transports/slack/` 都有的五个部分：

1. **设置**（`settings.py`）：环境变量驱动的配置，缺失时抛出 `GatewayConfigurationError`（来自 `gateway/core/lifecycle/errors.py`）。
2. **监听器**（`startup.py` + 传输 worker）：接收入站消息，用 `(text, session, output, logger)` 调用共享处理器。
3. **入站安全**：对每条消息授权并审计日志（`integrations/messaging_security`）。
4. **轮次输出**（实现 `infrastructure/turn_host/turn_output.py` 中的 `TurnOutput`）：流式输出状态并投递答案。
5. **会话绑定**：通过 `gateway/core/storage/session/resolver.py` 使用新的 `platform` 值，将平台会话键映射到 `Session`。

然后在组合根（`gateway/core/lifecycle/controller.py` 中的 `GatewayController`）中与现有传输一起注册它。直接复用 `TurnRunner(...)` 的处理器。

**永远不要改动：** `TurnRunner`、harness 提示词/工具或会话 agent 池。保持处理器与传输无关，正是让新平台成为小而自包含的添加的关键。