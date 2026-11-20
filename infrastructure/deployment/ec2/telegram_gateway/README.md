# `infrastructure/deployment/ec2/telegram_gateway/`

OpenSRE **Telegram** 网关（长轮询）的 AMI + systemd 部署路径。Slack 单独部署和运维，不在此仓库中。

此路径将网关进程作为 systemd 服务直接运行在 EC2 主机上，因此网关会话内可正常执行 shell 命令（如 `curl`、`systemctl`、`sudo`）。

## 目录内容

| 路径 | 用途 |
| ---- | ------- |
| `systemd/opensre-gateway.service` | 包含在服务器镜像中的 systemd 单元文件。从 `/etc/opensre/gateway.env` 读取环境变量。 |
| `stack.py` | `GatewayStack` 数据类 + 辅助函数，用于将 AMI id 和部署输出持久化到 `~/.opensre/deployments/`。 |
| `build_server_image.py` | `build_server_image()` — 启动临时构建器 EC2 实例，通过 SSM 执行内联安装命令，快照 AMI，然后终止构建器。 |
| `install_on_new_server.py` | `install_on_new_server()` / `destroy_installed_server()` — 启动普通 EC2 实例，从已发布的安装器安装网关，无需预先构建镜像。 |
| `provision.py` | `provision_gateway_via_ssm()` 和 `wait_for_gateway_ready()` — 写入 `/etc/opensre/gateway.env` 并通过 SSM 重启服务。 |
| `lifecycle.py` | CLI 入口：`build-server-image`、`deploy`、`destroy`、`install-on-new-server`、`destroy-installed-server` 子命令。 |

## 命令

从**仓库根目录**运行（先执行 `make install`）。

| 命令 | 作用 |
| ------- | ------------ |
| `make build-gateway-image` | 启动临时 EC2，安装当前 git HEAD 的 OpenSRE，快照 AMI，本地保存 AMI id |
| `make deploy-gateway` | 销毁之前的栈，从保存的 AMI 启动 EC2，写入环境变量，启动服务 |
| `make destroy-gateway` | 终止实例，删除 IAM 配置/角色；默认保留 AMI |
| `make install-gateway-on-new-server` | 启动普通 EC2 实例并在其上安装网关，无需镜像 |
| `make destroy-gateway-on-new-server` | 拆除上述命令创建的服务器 |

等效的 Python 入口：

```bash
uv run python -m infrastructure.deployment.ec2.telegram_gateway.lifecycle build-server-image
uv run python -m infrastructure.deployment.ec2.telegram_gateway.lifecycle deploy
uv run python -m infrastructure.deployment.ec2.telegram_gateway.lifecycle destroy
uv run python -m infrastructure.deployment.ec2.telegram_gateway.lifecycle install-on-new-server
uv run python -m infrastructure.deployment.ec2.telegram_gateway.lifecycle destroy-installed-server
```

### 前置条件

1. **AWS 凭证** — 静态密钥或通过默认 boto3 链获取的角色。
2. **权限** — 部署账户/区域需要 EC2、SSM、IAM 权限。无需 ECR。
3. **区域** — 默认为 `us-east-1`（与主部署相同）。

### 环境变量

复制 [`.env.deploy.example`](../../../../.env.deploy.example) 并设置 `TELEGRAM_BOT_TOKEN`，以及主部署使用的 `LLM_PROVIDER` 和 API 密钥。

| 变量 | 必需 | 使用方 |
| -------- | -------- | ------- |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | 是（或角色） | 资源创建 |
| `TELEGRAM_BOT_TOKEN` | 是 | 网关服务 |
| `TELEGRAM_ALLOWED_USERS` | 推荐 | 网关配对门控 |

`SLACK_*` 变量在此部署路径中被忽略（校验时会警告）— Slack 单独部署和运维，不在此仓库中。

| `LLM_PROVIDER` + API 密钥 | 是 | 网关服务 |
| `OPENSRE_GATEWAY_GIT_REF` | 否 | 安装到镜像中的 Git ref（默认：本地 HEAD SHA） |
| `OPENSRE_GATEWAY_AMI_ID` | 否 | 跳过镜像构建，使用已有镜像 id |
| `OPENSRE_GATEWAY_DESTROY_PURGE_AMI` | 否 | 设为 `1` 以在销毁时同时注销 AMI |
| `OPENSRE_STACK_SUFFIX` | 否 | 按开发者区分的资源名称后缀 |

### `make deploy-gateway` 创建的内容

一个名为 `opensre-gateway` 的栈：

- **EC2** `t3.micro`，位于账户默认 VPC
- **IAM** 实例配置 — SSM + Bedrock（无需 ECR）
- **systemd** `opensre-gateway.service`，以 `opensre` 系统用户运行

输出写入 `~/.opensre/deployments/opensre-gateway.json`。

### 一次构建，多次部署

```bash
# 每次代码变更构建一次（约 5-10 分钟）：
make build-gateway-image

# 使用保存的 AMI id 快速重新部署（约 2-3 分钟）：
make deploy-gateway
make destroy-gateway
make deploy-gateway
```

### 回滚

回滚到之前构建的镜像：

```bash
OPENSRE_GATEWAY_AMI_ID=ami-<previous-id> make deploy-gateway
```

### 检查网关

```bash
# SSH（如果设置了 EC2_KEY_NAME）或 SSM 会话：
aws ssm start-session --target <InstanceId>

# 在实例内：
sudo systemctl status opensre-gateway
sudo journalctl -u opensre-gateway -f
```

## 持久化

网关会话状态位于实例 EBS 根卷上的 `/var/lib/opensre-gateway/.opensre/`。它能在服务重启和系统重启后保留，但**不能**在完整的 `make deploy-gateway` 后保留（新实例 = 全新磁盘）。如果会话连续性很重要，请在重新部署前备份 `/var/lib/opensre-gateway/.opensre/gateway/state.db`。