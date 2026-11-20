# 贡献指南

欢迎来到 OpenSRE

## 快速链接

- **GitHub：** [https://github.com/Tracer-Cloud/opensre](https://github.com/Tracer-Cloud/opensre)
- **Discord：** [https://discord.gg/opensre](https://discord.gg/opensre)
- **X/Twitter：** [@open_sre](https://x.com/open_sre)

## 当前优先级

公开路线图聚焦三个方向：

- **定时任务** — 周期性技能、调度器可靠性与投递
- **CI/CD 技能与改进** — 发布门禁、打包检查、安装器验证
- **并行化** — 并发定时运行、投递扇出、CI/测试并行

按 `scheduled-task`、`ci`、`skills`、`parallelization` 标签浏览待办。调查流水线、证据映射器以及仅清理类重构任务已不在路线图中；这些领域的新 Issue 将被关闭。

## 如何贡献

想找安全的首次贡献？参见 [Good First Issues](docs/good-first-issues/README.md)。

按贡献类型选择对应路径：

1. **Bug 与小修复** -> 直接开 PR。如需先提 Issue，使用 [Bug 报告模板](https://github.com/Tracer-Cloud/opensre/issues/new?template=bug_report.yml)。
2. **新功能或行为变更** -> 先提 [功能请求](https://github.com/Tracer-Cloud/opensre/issues/new?template=feature_request.yml)，或在 Discord 中询问后再写代码。多数功能想法更适合通过插件 SDK 以第三方插件形式发布。
3. **与具体工作挂钩的改进** -> 针对定时任务、CI/CD、性能、可靠性或并行执行提出可度量的改进时，使用 [改进模板](https://github.com/Tracer-Cloud/opensre/issues/new?template=improvement.yml)。
4. **仅重构的 PR** -> 除非维护者明确要求作为实际修复的一部分，否则不要开。
5. **针对已知 `main` 失败的仅测试/CI PR** -> 除非变更是验证维护者要求的实际修复所必需，否则不要开。
6. **问题咨询** -> 查阅文档、发邮件至 [support@opensre.com](mailto:support@opensre.com)，或在 Discord [#contribute](http://discord.gg/opensre) 中提问。GitHub Issues 仅用于可执行的工作。
7. **安全问题** -> 遵循 `SECURITY.md`；不要开公开 Issue。

### 环境搭建

详细搭建说明（含 Windows 专属指引）见 **[SETUP.md](SETUP.md)**。基准测试、部署细节与遥测参考见 **[docs/DEVELOPMENT.md](docs/DEVELOPMENT.md)**。

**快速开始：**

1. 安装 [uv](https://docs.astral.sh/uv/getting-started/installation/) 并克隆仓库（Windows 及替代方案见 [SETUP.md](SETUP.md)）
2. 安装依赖：`make install`
3. 遵循 [CI.md](CI.md) 中的验证流程。
    - 从本地检出调用 CLI 时，优先使用 **`uv run opensre …`**（若其他 `opensre` 遮蔽了 `.venv`，见 `SETUP.md` 故障排查）。
4. 需要时构建发布产物：`make build`

如使用 VS Code，请使用 [.devcontainer/devcontainer.json](.devcontainer/devcontainer.json) 中的 devcontainer。详情：[docs/DEVELOPMENT.md](docs/DEVELOPMENT.md#vs-code-dev-container)。

---

**贡献流程：**

1. **找到或创建 Issue** — 选择已有 Issue（路径 A）或新提一个（路径 B）
2. **申请指派** — 在 Issue 下评论，让维护者知道你正在处理
3. **讨论（如需要）** — 功能/变更类先在 Issue 中讨论方案再写代码
4. **Fork 并建分支** — 为工作创建分支：`git checkout -b issue/123-description`
5. **编码与测试** — 修改代码、添加测试、确保所有检查通过
6. **提交 PR** — 开 Pull Request（或草稿 PR）并关联 Issue；使用 PR 模板
7. **评审与迭代** — 回应反馈，按需修改
8. **合并** — 维护者批准后合并

**详细步骤：** 见下方"开发工作流"。

---

## 开发工作流

### 1. 创建分支

```bash
git checkout -b issue/123-short-description
```

修复 Bug 用 `issue/` 或 `fix/`，新功能开发用 `/feat`。分支名用小写加连字符。

### 2. 修改代码

- 提交保持聚焦、逻辑清晰
- 写清提交信息：`"Fix: CLI returns error on incomplete commands"`
- 尽量一次提交只做一件事

### 2.1 添加工具（快速路径：单文件）

简单工具无需类或 `ClassVar` 元数据。在 `tools/` 下新增一个文件，用 `@tool` 注册函数即可。

示例（`tools/example_status_tool.py`）：

```python
from core.tool_framework.tool_decorator import tool


@tool(source="knowledge")
def get_example_status(run_id: str, include_history: bool = False) -> dict[str, object]:
    """Return a lightweight status summary for a run."""
    return {
        "run_id": run_id,
        "include_history": include_history,
    }
```

说明：

- 函数式工具必须提供 `source`。
- `name`、`description`、`input_schema` 默认自动推断。
- `surfaces` 默认为 `("chat",)`。传 `surfaces=("chat", "action")` 可让工具同时出现在聊天与操作上下文中。
- 工具若含复杂辅助逻辑、多个导出或大量集成专属代码，请沿用现有包/类风格。

以上示例是最小形态。工具**放在哪**（单厂商放 `integrations/<vendor>/tools/`，还是 `tools/system/` / `tools/cross_vendor/`）见 [docs/tool-placement-policy.md](docs/tool-placement-policy.md)；完整完成标准（元数据、实时载荷解析、文档、测试、验证）见 [docs/adding-tools-and-integrations.md](docs/adding-tools-and-integrations.md)。

### 3. 添加或更新测试

- **测试位置：** 新测试放在 `tests/` 目录，必要时镜像源码包结构（如 `surfaces/cli/` 的测试放 `tests/cli/`）。
- **禁止内联测试：** 不要在源码包内直接添加 `*_test.py` 文件。现有内联测试正在逐步移除，以保持核心逻辑整洁。
- Bug 修复应附带能捕获该 Bug 的测试
- 新功能应有对应测试
- 目标代码覆盖率 >80%；`make test-cov` 可生成本地报告（可选）。

### 4. 验证变更

本地验证、PR 必需检查与评审跟进见 [CI.md](CI.md)。

### 运行单个聚焦测试

将占位符替换为实际文件或测试名：

```bash
pytest tests/cli/test_.py                                       # single file
pytest tests/cli/test_.py::test_                                # single function
pytest tests/tools/ -k "test_registry"                          # tools example
```

### 5. 提交 Pull Request

遵循 PR 模板（见下文）。关联相关 Issue，说明改了什么、为什么改。

## Pull Request 指南

### 如何写好 PR 描述

使用 **[PR 模板](.github/PULL_REQUEST_TEMPLATE.md)**（开 PR 时自动提供）。关键部分：

- **Issue 链接：** `Fixes #123`（合并时自动关闭 Issue）
- **变更类型：** 选择 Bug 修复、功能、破坏性变更或文档（便于分类）
- **描述：** 改了什么、为什么改
- **测试：** 如何测试，附具体步骤与证据
- **影响分析：** 是否向后兼容？有无破坏性变更？性能影响？

### 提交前的 PR 检查清单

- 已关联相关 Issue
- 已遵循 [CI.md](CI.md) 并在 PR 中报告验证结果
- Bug 修复或新功能已添加测试
- 行为变更已更新文档
- 代码符合项目风格（见下方 **代码质量标准**）
- 已先自查代码
- 已考虑边界情况

### Greptile 代码评审

评审触发条件与合并要求见 [CI.md §8](CI.md#8-post-pr-follow-through)。自动化与人工评审概览见 [Pull request review flow](docs/guides/pr-review-flow.mdx) 指南。

### PR 含截图或日志时

以下情况需提供**修改前**与**修改后**示例：

- 变更 CLI 输出或错误信息
- 更新 Agent 行为
- 修复有可见影响的 Bug

### AI 辅助 PR

若使用 AI 工具（Claude、ChatGPT、Copilot 等）生成代码，**[PR 模板](.github/PULL_REQUEST_TEMPLATE.md)** 要求你确认：

- 我已审阅 AI 生成代码的**每一行**（不只是粗略浏览）
- 我理解其逻辑，能用自己的话解释
- 我测试了边界情况（哪些地方可能出错？）
- 我调整了输出以符合项目规范（[代码质量标准](#code-quality-standards)）
- 已验证 AI 生成代码的测试通过

这确保你理解代码，而非仅复制。评审者会格外关注 AI 辅助代码。

## 代码质量标准

仓库级规范见 [AGENTS.md](AGENTS.md)；测试替身、LLM 请求处理与安全的 CodeQL 修复见 [贡献者说明](CONTRIBUTOR_NOTES.md)。

- **清晰优于取巧：** 代码应易于理解和维护
- **DRY 原则：** 不要重复自己；抽取通用模式
- **强类型：** 所有函数参数与返回值使用类型提示
- **单一职责：** 每个函数/类只做好一件事
- **注释解释"为什么"：** 说明非显而易见的逻辑；代码本身已体现"做什么"
- **破坏性变更：** 在 PR 描述与文档中明确标注

### 风格与格式

我们使用：

- **ruff** 做 lint 与导入排序
- **mypy** 做严格类型检查
- **Black 兼容**格式（4 空格缩进）
- **pytest** 做测试并跟踪覆盖率

验证包可发布，运行：

```bash
make build
```

## 报告 Bug

创建 Issue 时使用 **[Bug 报告模板](https://github.com/Tracer-Cloud/opensre/issues/new?template=bug_report.yml)**。模板会引导你填写：

- **摘要：** 一行描述 Bug（具体，不笼统）
- **预期行为：** 应该发生什么
- **实际行为：** 实际发生了什么（附错误信息）
- **复现步骤：** 清晰、最小、能稳定触发 Bug 的步骤
- **能否稳定复现？** 每次 / 间歇性 / 偶尔
- **环境：** 操作系统、Python 版本、Agent 版本、安装方式、相关配置
- **错误输出：** 完整错误信息与日志（隐去 API Key 等机密）
- **临时方案：** 如找到绕过方法
- **背景：** 你想做什么？是否阻塞了工作？

**示例：**

```
### Expected Behavior
`opensre ask "why is checkout-api slow?"` should print an answer

### Actual Behavior
Command exits silently with no output
Error: exit code 0

### Steps to Reproduce
1. Run `opensre ask "why is checkout-api slow?"`
2. Observe output

### Environment
- OS: macOS 14.2
- Python: 3.11.5
- opensre version: v0.2.1
```

## 请求功能

使用 **[功能请求模板](https://github.com/Tracer-Cloud/opensre/issues/new?template=feature_request.yml)** 提出新功能。模板会引导你说明：

- **问题陈述：** 为什么需要？（聚焦问题，而非方案）
- **建议方案：** 应如何工作？（具体、可操作，附示例）
- **验收标准：** 满足什么条件才算"完成"？
- **替代方案：** 考虑过的其他方案及为何偏好此方案
- **是否向后兼容？** 是 / 否 / 破坏性变更（描述变更内容）
- **影响：** 涉及哪些模块？是否新增依赖？

## 建议改进

使用 **[改进模板](https://github.com/Tracer-Cloud/opensre/issues/new?template=improvement.yml)** 针对定时任务、CI/CD、性能、可靠性或并行执行提出可度量的改进。仅清理类重构不在范围内。需填写：

- **当前状态：** 现在如何工作？（附代码引用）
- **期望状态：** 应如何工作？
- **为何重要：** 性能？可维护性？可靠性？
- **范围：** 每个 Issue 只聚焦一个问题（不捆绑多项工作）
- **验收标准：** 如何衡量成功？
- **指标：** 修改前后数值（如 "15ms → <1ms"）

## 需要帮助？

- **搭建问题？** 先查本指南，再带详细信息开 Issue
- **如何……？** 查阅项目文档或在 Discussion 中提问
- **发现 Bug？** 用模板开 Bug 报告 Issue
- **有想法？** 先发起 Discussion 试探兴趣，再开 Issue

## 许可

参与贡献即表示你同意贡献按项目许可证授权（见 `LICENSE`）。