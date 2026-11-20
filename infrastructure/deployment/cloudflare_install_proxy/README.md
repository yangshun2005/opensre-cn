# Cloudflare 安装代理

用于安装域名 `install.opensre.com` 的 Cloudflare Worker。

在 `infrastructure/deployment/cloudflare_install_proxy` 目录下部署：

1. 登录 Cloudflare：
   `npx wrangler login`
2. 部署 Worker：
   `npx wrangler deploy`

该代理从仓库提供安装脚本：

- `https://install.opensre.com` -> 根据请求自动识别 shell
- `https://install.opensre.com/install.sh` -> Unix shell 安装脚本
- `https://install.opensre.com/install.ps1` -> PowerShell 安装脚本

根端点可选查询参数覆盖：

- `?shell=sh`
- `?shell=powershell`