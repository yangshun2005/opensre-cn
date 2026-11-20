# Runner 遥测

由 GitHub Actions 启动的 OpenSRE 工作负载，请使用 runner 启动器。该任务需要 `permissions: { contents: read, id-token: write }`。

```bash
uv run python -m infrastructure.analytics.runner_launcher \
  --manifest evidence/run.json -- uv run opensre ask "Your task"

uv run python -m infrastructure.analytics.runner_launcher \
  --docker your-opensre-image --manifest evidence/container.json \
  -- opensre ask "Your task"
```

每次调用都会获得全新的安装配置和可续期的 runner 凭据。请将 manifest 保存为工作流构建产物；它包含运行时安装 ID 和 job/run 标识符，不含凭据。合成检查用 `--test`；当命令记录了安装、且必须由摄取确认否则任务应失败时，用 `--require-delivery`。

Docker 路径挂载 runner 上下文，与 CI 环境变量无关。请让受信任的 job 通过此包装器启动每个容器。子进程不会获得自行签发 runner 凭据的权限。不要只把上下文 JSON 复制进镜像：凭据会过期，目录挂载必须保持存活以便续期。原生子进程必须保留提供的 `OPENSRE_HOME`、`OPENSRE_WIZARD_STORE_PATH` 和 `OPENSRE_EXECUTION_CONTEXT_PATH`。不要覆盖配置，也不要在多个 job 间共享其安装 ID。

摄取会验证 GitHub 的签名、仓库和所有者 ID、令牌有效期，以及安装专属的 audience。Runner 验证不授予任何账户或组织访问权限。当该验证不可用或无效时，事件保留 `reported` 或 `unknown` 来源；仅凭 CI 检测不能证明 runner 归属。`Runner provenance` 工作流会发出带标记的生产金丝雀，并保存其精确 ID 以供数据库核对。