---
name: opensre-cli-setup
description: 安装并维护 OpenSRE CLI（安装、配置、验证、首次运行）
user-invocable: true
allowed-tools: Bash(command:*), Bash(curl:*), Bash(opensre:*)
metadata:
  author: opensre
---

# OpenSRE CLI 配置

以最小阻力配置或维护 `opensre`。如果你正在运行此提示，请按以下步骤操作，为用户配置 OpenSRE，使其能在交互式 shell 中查询基础设施。

除非用户要求贡献代码，否则不要克隆 OpenSRE 仓库。已发布的 CLI 已足够。

## 第 1 步：安装或升级 CLI

检查 CLI 是否存在：

```bash
command -v opensre
opensre --version
```

如果 `opensre` 不存在，请安装。在 macOS 或 Linux 上：

```bash
curl -fsSL https://install.opensre.com | bash -s -- -dc
```

在 Windows 上，使用 PowerShell：

```powershell
& ([scriptblock]::Create((irm https://install.opensre.com/install.ps1))) -dc
```

Windows 详情：[Windows](https://www.opensre.com/docs/environments/windows-local)。

macOS/Linux 通常无需 sudo。如果安装程序无法使用 `PATH` 中已有的可写目录，它会将二进制文件放入 `~/.local/bin`，并打印添加该目录的命令。执行该 PATH 更新（或打开新终端），然后重新检查 `command -v opensre`。

如果 `opensre` 已存在，请升级：

```bash
opensre update
```

确认其可运行：

```bash
opensre --help
```

## 第 2 步：开始使用

首次启动是交互式的，需要 TTY。不要试图伪造登录环节。运行 `opensre`，并在其请求输入时提示用户：

```bash
opensre
```

浏览器会打开 OpenSRE 注册页面。用户可创建账户，或使用邮箱或其他已启用的提供商登录。这会激活托管模型；首次运行期间不要向用户索要单独的 LLM 密钥。

登录完成后，会打开交互式 shell。之后当智能体需要查询某些工具时，用 `opensre integrations setup <service>` 添加它们。

之后连接单个工具：

```bash
opensre integrations setup <service>
```

将 `<service>` 替换为 slug，例如 `datadog`、`grafana` 或 `slack`。

如果登录无法验证 webapp 账户，请运行 `opensre account status`。当会话已过期、被撤销、不完整或无法访问时，交互式 shell 会刻意保持关闭。

## 第 3 步：验证

```bash
opensre integrations verify
```

仅验证一个工具：

```bash
opensre integrations verify datadog
```

如果验证失败，请检查打印的错误。常见原因：凭据缺失或过期、URL 错误，或跳过了某个配置步骤。为该工具重新运行 `opensre integrations setup <service>`。

## 第 4 步：建议首次运行

第 2 步之后，用户已处于交互式 shell 中。请用户用自然语言描述一个事件或提出问题，或使用 `/help`。之后要再次启动：

```bash
opensre
```

## 常见问题

- **`opensre: command not found`** — 打开新终端，或添加安装程序打印的 bin 目录（macOS/Linux 上通常是 `~/.local/bin`）。
- **登录阻塞或看起来卡住** — 它在等待 webapp 浏览器认证。向用户显示 URL 和提示；不要终止它。
- **shell 反复返回登录** — 运行 `opensre account status`；账户必须处于活动状态，shell 或托管模型才能启动。
- **只会查询已连接的工具** — 如果从未配置 Datadog，智能体无法拉取 Datadog 数据。生产运行前请先运行 `opensre integrations verify`。

人类文档：https://opensre.com/docs/install 和 https://opensre.com/docs/quickstart