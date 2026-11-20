# 可观测性

负责 OpenSRE 的运行时可观测性服务。

职责：

- 进度上报端口。
- 调试输出 Hook。
- 运行时显示适配器。
- 结构化日志辅助工具。
- 追踪与 Span 辅助工具。

当前实现候选位于 `infrastructure/observability/` 下，应作为导入迁移的一部分移至此处。