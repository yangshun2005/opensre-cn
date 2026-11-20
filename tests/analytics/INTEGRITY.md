# 安装与账号关联完整性探测

运行 `uv run python -m pytest tests/analytics/test_integrity_environments.py`。
设置 `INTEGRITY_EVIDENCE_DIR` 可保留捕获的请求体和结果。

专用的 Telemetry 完整性工作流在原生 Linux、macOS、Windows 和 Linux Docker 容器上运行此测试套件。每个用例都使用真实的匿名 ID 存储、安装标记、属性构建器、遥测队列、目标解析器、已保存的账号凭据和签名，启动全新的 Python 进程。仅替换出站 HTTP 传输。接收端只接受合成的 fixture 凭据，并校验请求签名。子进程环境会排除继承的分析凭据和目标地址。

这些用例区分以下粒度：

- 保留存储重启进程，只产生一个安装身份。
- 替换存储后，每个运行时都会产生一个新的安装身份，尽管每个运行时都报告 `identity_persistence=disk`。这些身份不能作为独立用户或独立下载的证据。
- 投递被拒后，下一个进程会用同一个去重键重试。
- 已保存的个人凭据（按登录流程在令牌交换后存储的方式持久化）会保留匿名安装 ID，用个人 bearer 签名 `account_authenticated`，并在后续进程中为每个请求签名。不运行交互式登录和令牌交换。
- silo 的 bearer 与个人账号 bearer 不同，不会仅因运行时启动就发出 `account_authenticated`。
- 真实的主机/容器检测可区分原生、CI、容器和 CI 容器流量；非 CI 用例会移除继承的 CI 环境信号。

此套件测试的是 emitter 行为。它不下载或执行发布版本安装程序，不重建物理主机，不运行浏览器登录或令牌交换，不创建 Clerk 账号，不断言服务端用户解析，也不验证投递到生产 ClickHouse/PostHog。保留存储与替换存储下的进程重启，独立于容器调度器来建模身份持久化边界。捕获的 JSON 产物只包含生成的 fixture 身份和请求。