# 基础设施包

这些说明适用于 `infrastructure/` 及其所有子目录。
仓库根目录的 `AGENTS.md` 仍然适用。

## 用途

`infrastructure/` 负责共享运行时服务，这些服务位于**用户可见的应用层之外**，也位于**核心 agent harness 循环之外**。
它可以负责副作用：遥测、审计日志、链路追踪、认证、脱敏、沙箱执行和最小限度的护栏。

仅涉及配置的行为属于 `config/`。
Agent 编排、状态、工具规划和执行契约属于 `core/`。

按**功能**命名包。不要创建 `common/`、`shared/` 或 `util/` 这类杂物抽屉。优先使用叶子导入，而非再导出中间层。

各子包的归属关系见 [`README.md`](./README.md)。添加模块前先阅读它。

`alert_intake.py` 和 `asgi_server.py` 保留在包根目录：网关和交互式 shell 共享它们，且互不导入。不要将它们移入子包。

## 导入边界

依据 [`docs/ARCHITECTURE.md`](../docs/ARCHITECTURE.md) 中的 **Tier 3**：

`infrastructure/` 可以导入：`config`（并可交叉导入其 Tier-3 同级 `core`）。

`infrastructure/` 不得导入：`surfaces`、`gateway`、`tools`、`integrations`。

## 横切规则

- 保持导入时的工作轻量——模块导入时不得启动线程、发起网络请求或执行繁重 I/O。
- 未来迁入此包的代码必须只保留一条规范导入路径；不要留下仅用于兼容的转发模块。