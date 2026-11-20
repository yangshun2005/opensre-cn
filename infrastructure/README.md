# 基础设施

`infrastructure/` 包含共享运行时服务，位于面向用户的应用包之外，也位于核心 agent harness 循环之外。

基础设施代码可以负责副作用，例如遥测上报、审计日志、运行时显示、追踪、认证校验、脱敏、沙箱执行和最小护栏。仅配置行为属于 `config/`；agent 编排、状态、工具规划和工具执行契约属于 `core/`。

按**功能**命名包。不要添加 `common/` / `shared/` / `util/` 这类杂物抽屉。优先使用叶子导入，而非再导出垫片。

## 进程与安装

- `process/` — 退出码、CLI 运行时标志、进程级轮次容量（`process/turn_capacity/`）、进程树终止，以及已安装版本与最新发布版本（`process/release_version.py`）。
- `setup_state.py` — 通过提示上下文暴露给 agent 的安装/设置事实。
- `alert_intake.py` — 最小 HTTP 入口（`POST /alerts`），写入进程级收件箱，由网关和交互式 shell 共享，两者互不导入。
- `asgi_server.py` — 通用 ASGI 传输：在后台线程中运行任意 ASGI 应用（`serve_asgi_in_thread`）。
- `request_body_limit.py` — 共享 ASGI 中间件，在独立告警监听器、网关 Web 应用或 Slack Events API 缓冲之前限制变更类请求体大小。
- `turn_host/` — 共享轮次宿主，其单一 `TurnRunner` 服务于交互式 shell 和所有网关传输。
- `harness_providers/` — 接入 harness 的集成提供方注册表。

## 契约

- `errors/` — `OpenSREError` 契约（任意层均可抛出/捕获）。
- `service_families/` — 工具可用性族键规范化。

## 提示规模结果

- `text/` — 截断 / 强制转换 / URL 校验辅助函数。
- `evidence/` — 日志与证据压缩；工具结果的指标摘要。

## 可观测性与 UX

- `observability/` — 日志、追踪、进度、调试输出和运行时显示端口。
- `logging/` — shell/第三方日志处理器。
- `analytics/` — 产品与运行时分析。
- `terminal/` — 终端主题与显示辅助函数，以及在用户浏览器中打开链接。

## 安全 — `safety/`

- `safety/auth/` — 运行时认证与身份校验。
- `safety/guardrails/` — 核心 agent 循环之外的最小运行时安全检查。
- `safety/masking/` — 可逆脱敏与标识符规范化。
- `safety/sandbox/` — 受限执行环境。

## 定时与后台工作 — `scheduling/`

- `scheduling/scheduler/` — cron 与 agentic 循环任务，由网关进程托管。不是网关子模块：所有界面都读取和修改任务存储，网关只提供进程和容量门控。
- `scheduling/task_types.py` / `scheduling/task_registry.py` — 进行中任务契约与持久化注册表。（不在 `tasks/` 目录下——根 `.gitignore` 在所有位置忽略 `tasks/`。）
- `scheduling/background_investigations/` — 后台调查存储与类型。

## 投递 — `delivery/`

- `delivery/notifications/` — 通知投递传输与渠道专用发送器。
- `delivery/reporting/` — 跨厂商报告投递注册表与界面无关端口。

## 持久化

- `database/` — 共享数据库连接与事务机制；领域包保留对其 Schema、迁移和运维查询的所有权。
- `filestorage/` — 可同步文件存储提供方与操作。

## 部署与打包 — `deployment/`

- `deployment/ec2/` — EC2 AWS 原语与 Telegram 网关 AMI/systemd 部署（`telegram_gateway/`）。Makefile：`make deploy-gateway`。
- `deployment/packaging/` — wheel 校验与发布清单辅助函数。
- `deployment/contracts/` — 共享部署模型。`SizeProfile` 在运行时由网关容量门控读取，因此这不是仅构建时代码。

`install.opensre.com` 的 Cloudflare Worker 不是 Python，位于 `deployment/cloudflare_install_proxy/`。

未来的迁移应逐步将现有模块移入此文件夹，并更新导入和测试。避免仅用于兼容的转发模块；每次迁移应只留下一个规范导入路径。