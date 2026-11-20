## 部署

OpenSRE 提供 OpenSRE Cloud 托管网关、AWS EC2 路径，以及适用于 ASGI 兼容平台的通用托管运行时选项：

- **OpenSRE Cloud 托管网关** — 组织管理员在 Web 应用中开通。已登录的 shell 用户可查看、启动或停止其组织的网关；启动和停止需要审批及管理员权限。
- **Slack** — 独立部署和运维，不通过本仓库。下方 EC2 路径不会携带 `SLACK_*` 变量（Socket Mode 为单消费者模式——第二个消费者会拆分事件）。
- **Telegram** — 使用下方 EC2 网关部署。

---

## 网关部署 — AMI + systemd（Telegram）

在 EC2 上以 systemd 服务直接运行 Telegram 网关。网关一次性烘焙进自定义 AMI；后续部署从该 AMI 启动，约需 2–3 分钟。

**前提条件：** 具备 EC2 / IAM / SSM 权限的 AWS 凭证。无需 Docker。

复制 [`.env.deploy.example`](.env.deploy.example) 并导出所需变量：

| 变量 | 必需 | 使用方 |
| -------- | -------- | ------- |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | 是（或使用角色） | 资源开通 |
| `TELEGRAM_BOT_TOKEN` | 是 | 网关服务 |
| `TELEGRAM_ALLOWED_USERS` | 推荐 | 网关配对门禁 |
| `LLM_PROVIDER` + API key | 是 | 网关服务 |

EC2 部署会忽略 `SLACK_*` 变量（校验时会警告）——Slack 独立部署和运维，不通过本仓库。

```bash
# Step 1 — bake a gateway AMI (run once per code change, takes ~5-10 minutes):
make build-gateway-image

# Step 2 — launch EC2 instance from the saved AMI (fast):
make deploy-gateway

# Tear down (keeps AMI by default):
make destroy-gateway

# Full teardown including AMI deregistration:
OPENSRE_GATEWAY_DESTROY_PURGE_AMI=1 make destroy-gateway
```

回滚到之前烘焙的 AMI：

```bash
OPENSRE_GATEWAY_AMI_ID=ami-<previous-id> make deploy-gateway
```

通过 SSM 检查运行中的网关：

```bash
aws ssm start-session --target <InstanceId>
# inside:
sudo systemctl status opensre-gateway
sudo journalctl -u opensre-gateway -f
```

输出写入 `~/.opensre/deployments/opensre-gateway.json`。

部署后，Web API 可公开访问：

```bash
curl http://<PublicIpAddress>:8000/health
```

用 `OPENSRE_WEB_API_INGRESS_CIDR` 限制允许的源 CIDR（默认 `0.0.0.0/0`）。

### 直接部署（无预烘焙 AMI）

通过 SSM 在全新 EC2 实例上内联安装 OpenSRE——较慢，但无需烘焙步骤：

```bash
make install-gateway-on-new-server
make destroy-gateway-on-new-server
```

---

## 运行时环境（托管 / 通用）

使用仓库 `Dockerfile`、Railway、ECS、Vercel 或其他支持 ASGI 的托管平台，将 OpenSRE 作为标准 Python/FastAPI 应用部署。

1. 使用托管平台的常规工作流构建并部署。
2. 设置 `LLM_PROVIDER` 及对应的提供商 API key：
    - `LLM_PROVIDER=anthropic` 时设置 `ANTHROPIC_API_KEY`
    - `LLM_PROVIDER=openai` 时设置 `OPENAI_API_KEY`
    - `LLM_PROVIDER=openrouter` 时设置 `OPENROUTER_API_KEY`
    - `LLM_PROVIDER=trustedrouter` 时设置 `TRUSTEDROUTER_API_KEY`
    - `LLM_PROVIDER=gemini` 时设置 `GEMINI_API_KEY`
3. 网关需要共享的 Postgres 状态时设置 `DATABASE_URL`。Slack Events API 需要它，除非你刻意使用单副本本地去重的逃生通道。
4. 运行多个需要相同调度任务的进程前，将 `OPENSRE_HOME` 放在共享挂载上。若设置了 `OPENSRE_CONTEXT_ROOT`，也要让该根目录持久且共享，供组织绑定会话使用。
5. 添加集成所需的任何其他环境变量。

最小环境：

```bash
LLM_PROVIDER=anthropic
ANTHROPIC_API_KEY=...
```

支持的提供商 key 全集及可选模型覆盖记录在 [`.env.example`](.env.example) 中。

### Railway

使用 Slack Events API 或其他需要在副本间共享网关记录的布局时，添加 Railway Postgres 并设置 `DATABASE_URL`。Redis 不属于网关持久化路径。调度文件仍需共享的 `OPENSRE_HOME` 挂载。组织绑定会话在设置时使用 `OPENSRE_CONTEXT_ROOT`，否则回退到 `OPENSRE_HOME`；两者不同时都要持久化。

遥测标记方面，在 Railway 服务上设置 `OPENSRE_DEPLOYMENT_METHOD=railway`。

---