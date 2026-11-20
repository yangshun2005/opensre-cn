# gateway/web/ — Web 接口层

用于健康探针和告警接收的 FastAPI 应用。
不是聊天传输层——不绑定轮次运行器或轮次输出。

| 可导入 | 禁止导入 |
|------------|----------|
| `gateway.core.*` | `gateway.transports.*`、`gateway.startup` |

入口：`webapp.py`（`app`）——当 `MODE=web` 时使用 `uvicorn gateway.web.webapp:app`，
或通过 `web_server.serve_webapp_in_thread` 以守护进程 / shell 方式启动。

由 `gateway/tests/test_package_borders.py` 固定约束。

## 进程启动（`WEB_PROFILE`）

导入 `gateway.web.webapp`（通过 uvicorn 或进程内 `serve_webapp_in_thread`）会执行 `configure_process(WEB_PROFILE)`——包括环境变量、Sentry（`webapp`）和 harness 适配器。这是有意为之，使 `MODE=web` 和嵌入式 Web 启动方式一致，无需 CLI/管理器启动。**不要**在 `web/` 中添加第二个 harness 注册点；注册器仅保留在 `gateway/core/process/boot.py` 中。在完整的 gateway 进程中，`GATEWAY_PROFILE` 已执行；`WEB_PROFILE` 仍可能执行（使用独立的幂等键）——各步骤必须可安全重入，不得引入分叉的注册表。