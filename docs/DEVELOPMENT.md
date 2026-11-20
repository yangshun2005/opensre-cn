# 开发指南

面向贡献者的工作流：本地环境搭建细节见仓库根目录的 [SETUP.md](https://github.com/Tracer-Cloud/opensre/blob/main/SETUP.md)（Windows、故障排查、MCP）。

## 克隆与安装

```bash
git clone https://github.com/Tracer-Cloud/opensre.git
cd opensre
make install
```

[`make install`](https://github.com/Tracer-Cloud/opensre/blob/main/Makefile) 会执行 `uv sync --frozen --extra dev` 和分析安装辅助脚本。请在仓库根目录使用 **`uv run opensre …`**，确保始终使用本检出的 `.venv`，而非 `PATH` 上的其他 `opensre`。

```bash
opensre onboard
uv run opensre   # 打开交互式 shell
```

## 本地检查与 PR 验证

本地验证、必需的 PR 检查和评审跟进流程见 [CI.md](https://github.com/Tracer-Cloud/opensre/blob/main/CI.md)。

## 交互式 shell 动作策略

动作规划器行为、后处理转换、兼容性接缝和规则扩展清单见 [`docs/interactive-shell-action-policy.md`](https://github.com/Tracer-Cloud/opensre/blob/main/docs/interactive-shell-action-policy.md)。

## 包架构

八个一方包、五层分层（哪个包可导入哪个包）、目录图、各层职责和跨层流程见 [`docs/ARCHITECTURE.md`](ARCHITECTURE.md)。

## 工具注册表——按界面限定、懒加载

启动时加载所有厂商工具很慢。静态索引（`tools/registry_index.py`）通过扫描源码读取工具元数据，不导入执行器，因此每轮只加载所需工具。

- `get_registered_tools(surface)` 只导入该界面的工具模块。
- `get_tool_descriptors(surface)` 返回元数据，不导入执行器。
- `load_tool(descriptor)` 仅在工具运行时导入执行器。

新增厂商工具只需一个 `@tool`/`BaseTool` 模块；索引会自动发现，且不导入其他厂商。`tests/tools/test_registry_index.py` 校验索引与实际导入的注册表完全一致，防止二者漂移。

## VS Code 开发容器

开发容器定义在 [`.devcontainer/`](https://github.com/Tracer-Cloud/opensre/tree/main/.devcontainer)。它基于 [`.devcontainer/Dockerfile`](https://github.com/Tracer-Cloud/opensre/blob/main/.devcontainer/Dockerfile)（Python **3.13**）构建，随后 **`postCreateCommand`** 创建 `.venv-devcontainer` 并运行 **`pip install -e '.[dev]'`**（不使用 `uv`）。宿主机需具备 Docker Desktop、OrbStack、Colima 或其他兼容运行时。

## 部署

完整部署说明、前置条件和环境变量参考：
**[DEPLOYMENT.md](https://github.com/Tracer-Cloud/opensre/blob/main/DEPLOYMENT.md)**

快速参考：

| 路径 | 命令 |
| ---- | -------- |
| 网关（AMI + systemd——仅网关） | `make build-gateway-image` → `make deploy-gateway` / `make destroy-gateway` |
| 托管（Railway / ECS / Vercel） | 使用仓库 `Dockerfile` 部署；设置 `LLM_PROVIDER` + API key |

### 托管运行时（Railway / ECS / Vercel）

1. 部署仓库 `Dockerfile`，用 `MODE`（`web`、`gateway` 或 `scheduler`）选择运行时。容器启动时安装当前 main 通道构建。重启容器，或在容器内运行 `opensre update -y`，以获取更新的构建。工具链（Git、GitHub CLI、Node、Codex）变化时需重新构建镜像。
2. 设置 `LLM_PROVIDER` 及对应的 API key（例如 `ANTHROPIC_API_KEY`、`OPENAI_API_KEY`——见 [`.env.example`](https://github.com/Tracer-Cloud/opensre/blob/main/.env.example)）。
3. 当网关记录需通过 Postgres 共享时设置 `DATABASE_URL`
   （Slack Events API 必需，除非启用单副本本地去重逃生开关）。
4. 多个进程需共享同一调度任务时，使用共享的 `OPENSRE_HOME` 挂载。若设置了 `OPENSRE_CONTEXT_ROOT`，该根目录也需持久化并共享，供组织绑定会话使用。
5. 添加部署所需的集成和存储环境变量。

最小 LLM 环境变量：

```bash
export LLM_PROVIDER=anthropic
export ANTHROPIC_API_KEY=...
```

Railway：添加 Postgres，仅在网关需共享记录时设置 `DATABASE_URL`；Redis 不属于网关持久化路径。设置 `OPENSRE_DEPLOYMENT_METHOD=railway` 用于遥测标记。

## 遥测与隐私

`opensre` 内置两套遥测，均可关闭：

- **产品分析**——通过 `app.opensre.com` 发送生命周期和使用事件（所用命令、成功/失败、大致运行时长、CLI/Python/OS/架构，以及有限的命令元数据）。
- **Sentry**——崩溃和错误（堆栈跟踪、环境、发布版本）。

事件带有 `entrypoint`、`opensre.runtime` 和 `deployment_method` 标签。敏感请求头、路径和形似密钥的键在发送前会被清除。

产品事件还携带 `execution_environment`（`local`、`ci`、`container`
或 `ci_container`）、`is_ci`、`is_container` 和 `container_runtime`。用这些
一方字段将自动化环境从产品漏斗中排除。

随机安装 ID 存于 `~/.opensre/anonymous_id`。在 GitHub Actions 和 pytest 中遥测关闭。

### 关闭开关矩阵

| 环境变量                        | 产品分析 | Sentry     |
| ------------------------------ | ----------------- | ---------- |
| `OPENSRE_NO_TELEMETRY=1`       | 禁用   | 禁用   |
| `DO_NOT_TRACK=1`               | 禁用   | 禁用   |
| `OPENSRE_ANALYTICS_DISABLED=1` | 禁用   | 不受影响 |
| `OPENSRE_SENTRY_DISABLED=1`    | 不受影响 | 禁用   |
| `OPENSRE_SENTRY_LOGGING_DISABLED=1` | 不受影响 | 禁用 `logger.error`/`logger.exception` 转发至 Sentry；`capture_exception` 不受影响 |

完全关闭：

```bash
export OPENSRE_NO_TELEMETRY=1
```

### Sentry DSN

自托管用户可将 `SENTRY_DSN` 设为自有项目；未设置则使用内置默认值。`SENTRY_DSN=`（空值）会在 `before_send` 中丢弃事件。

### 部署标记

将 `OPENSRE_DEPLOYMENT_METHOD` 设为 `railway`、`ec2`、`vercel` 或 `local`（默认 `local`），用于标记 Sentry 事件。

### 本地产品事件日志

默认情况下，出站产品事件属性还会追加到
`~/.opensre/analytics_events.txt`（1000 行轮转）。禁用方式：

```bash
export OPENSRE_ANALYTICS_LOG_EVENTS=0
```

元数据事件不收集告警内容、文件内容、主机名、凭据或原始 CLI 参数。另行记录的 `$ai_generation` 事件包含脱敏后的提示词和响应文本。