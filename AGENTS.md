# OpenSRE 智能体说明

本文件只保留仓库级规则和任务相关的阅读指引。
编辑前先阅读适用的包级 `AGENTS.md`；仅在相关工作范围内加载所链接的参考文档。解释和事故历史放入参考文档，不放在这里。

## 人工维护的文件（强制）

智能体不得创建、编辑、重命名、移动或删除本仓库中任何 `SKILL.md`，也不得改动 `core/agent_harness/prompts/opensre_system_prompt.md`，包括其 frontmatter。直接请求、检查失败或微小改动都不能覆盖此规则。否则只能在聊天或 PR 中提出修改建议（引用当前文本和拟议文本），保持文件不变，并在交付其余工作的同时报告受阻的工作。

**唯一例外——字面 Ctrl-H 替换。** 当用户逐字提供当前文本和替换文本时，智能体可以执行精确的查找替换。替换是绝对的：将给定字符串逐字符替换为给定字符串，不触碰其他任何内容——不重写措辞、不重排格式、不做"顺手"修复、不提升 frontmatter，除非用户以同样方式明确写出。如果旧文本有歧义、缺失，或用户只是描述改动而非口述，则例外不适用；回退到建议模式。

技能引导模型推理，而非确定性流水线。辅助脚本只能处理少数狭窄的机械性杂务，并在 `references/script-tools.md` 中声明；决策、分支和端到端工作流不得移入这些脚本。需要时建议使用真正的工具。

## 命令与交付

- 安装环境和推送钩子：`make install`。
- 运行此检出：在仓库根目录执行 `uv run opensre …`。
- 运行 Python：`uv run python …`。
- 遵循 [CI.md](CI.md) 进行限定范围的本地验证、每次推送/PR 以及 CI/审查跟进。不要添加更宽泛的本地验证要求。
- 使用 [PR 模板](.github/PULL_REQUEST_TEMPLATE.md)，包括 AI 披露。

## 仓库级约定

- 使用严格类型，每个模块只承担一个明确职责，编写简洁、聚焦契约的 docstring。保留非显而易见的不变量和理由，而非缺陷历史。
- `__init__.py` 文件只保留公共导入和 `__all__`；实现和副作用放在专注的模块中。
- 新增或修改的 Protocol 方法以 docstring 作为**唯一**方法体：不得使用 `...`、`pass` 或 `raise NotImplementedError`。不要批量转换无关的现有桩，也不要依赖干净的 typecheck/CodeQL 扫描来强制执行此规则。
- 在源码和测试中使用 `http.HTTPStatus` 常量，绝不使用数字 HTTP 状态码字面量。
- 环境变量名和共享静态常量放在 `config/constants/` 下的领域叶子模块中，由其 `__init__.py` 重新导出。仅在 `config/llm_settings.py` 内部使用的名称可以放在那里；不要造成认证导入循环。
- 将行为放在其所属的领域/提供商/厂商模块中。先检查同类模式；保持编排和共享分发层薄，注册表仅限元数据/默认值，共享 UI 保持通用。两个或更多相关的功能特定函数应放在专用模块中。不要为服务多个提供商的文件添加提供商特定分支；改为委托给专用模块。
- 遵循架构边界：`core/` 和 `integrations/` 绝不从 `surfaces/` 导入。用共享叶子模块打破导入循环，而非惰性导入。
- 重构后，在同一次变更中迁移导入/测试并移除仅用于兼容的转发模块；保持一条规范的导入路径。
- 绝不用恒真条件开关隐藏失败行为，或用跳过测试来掩盖失败。
- 自然语言意图属于动作智能体：不得使用正则/关键字/模糊路由或确定性绕过。交互式 shell 规划器从不拒绝轮次；不要恢复 `mark_unhandled` 或 `UNHANDLED:`。显式命令例外和交接标签遵循适用的 harness/shell 说明。
- 在外部 HTTP/聊天响应边界脱敏异常详情，而非在共享调用点。在服务端用 `logger` 和 `capture_exception` 记录完整详情；返回通用消息或异常类型名。本地终端输出可以保留详情，但外部响应不得暴露提供商/模型内部信息。
- 缓存的 LLM 客户端服务并发轮次。重试决策必须使用从当前请求捕获的事实，而非可变的共享客户端标志。

## 测试、UI、文档与性能

- 每个失败类别优先一个高信号测试：安全/隔离、并发、崩溃/关闭、包/传输边界和真实回归。精简冗余的 mock 调用、词汇、成功和仅控制流测试，除非它们是某契约的唯一覆盖；在相关处执行真实循环/接线。
- 行为保持型重构前，添加特征测试并确认其在原始代码上通过；在整个变更过程中保持绿色。之后优化，并在 PR 中包含任何基准测试。
- 在热路径上，对重复的成员判断/查找使用 set/map，复用已解析的值，避免重复的 `deepcopy`/序列化/排序，并验证缓存对象未被修改。冷代码保持简单。
- CLI/UI 输入必须显示必填/可选字段和默认值，保存前一致地校验表单和直接命令，并在出错时保留值。
- `docs/` 面向用户：包含操作、必填/可选值、命令、环境变量和不可见的坑——而非内部 API 或缺陷历史。将新的 `.mdx` 页面添加到 `docs/docs.json`，使其出现在导航中。

## 相关时阅读

| 工作 | 必读参考 |
| --- | --- |
| 包结构或归属 | [架构与仓库地图](docs/ARCHITECTURE.md)；`core/` 变更参阅 [core 命名](docs/NAMING.md) |
| 工具或集成 | 添加或批准前参阅[放置策略](docs/tool-placement-policy.md)和[贡献者指南 / 完成定义](docs/adding-tools-and-integrations.md)；通过 `core.tool` 导入契约，通过 `core.tool_framework` 使用编写辅助 |
| 持久化或迁移 | [PERSISTENCE.md](PERSISTENCE.md) |
| CLI、REPL 或共享终端呈现 | [共享终端 UI 规则与视觉验证](surfaces/interactive_shell/ui/AGENTS.md) |
| 智能体执行或动作策略 | [Harness 规则](core/agent_harness/AGENTS.md)、[shell 规则](surfaces/interactive_shell/AGENTS.md)和[动作策略](docs/interactive-shell-action-policy.md) |
| LLM 提供商、重试或工具 Schema | [托管 LLM 规则](core/llm/AGENTS.md)和[请求/Schema 说明](CONTRIBUTOR_NOTES.md#llm-requests-and-tool-schemas) |
| 测试替身或测试类型 | [测试说明](CONTRIBUTOR_NOTES.md#test-fakes-and-typing) |
| CodeQL/代码质量发现 | [安全修复说明](CONTRIBUTOR_NOTES.md#codeql-and-code-quality) |
| 人工技能/提示词提案 | [提示词规则](core/agent_harness/prompts/AGENTS.md)和[技能契约](core/agent_harness/prompts/skills/AGENTS.md)；这些不授权编辑受保护文件 |