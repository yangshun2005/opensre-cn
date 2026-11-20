# 工具放置策略（T-20）

一个可供 agent 调用的能力放在哪里，取决于**其领域逻辑涉及多少个厂商集成**，而不是取决于方便程度或文件大小。这是 [ARCHITECTURE.md](ARCHITECTURE.md#tier-3--tools-and-integrations) 引用的决策规则。

## 决策规则

问：这个工具的*用途*是否依赖某个特定的外部厂商或 SaaS 后端（`integrations/<vendor>/` 包）？

1. **单一厂商** → `integrations/<vendor>/tools/`。
   该工具只在一个厂商的领域内才有意义（Datadog 查询、GitHub 变更、Sentry Issue 查询）。这是默认且最常见的情况——参见 `AGENTS.md` 中的“Adding a Tool”。
2. **完全不涉及厂商** → `tools/system/`。
   该工具的领域用途与外部厂商无关——本地进程内省、沙箱化代码执行、RAG 式指引检索。为附带关注点而*偶然*导入厂商客户端（例如解析多个可能凭据来源之一，或选择投递通知渠道）不会使工具变成厂商专属；判断标准是工具存在的理由，而不是文件中的每个导入。
3. **确实横跨 2 个以上厂商集成** → `tools/cross_vendor/`。
   工具逻辑本身跨多个 `integrations/<vendor>/` 包进行关联或编排——例如 `fix_sentry_issue` 从 `integrations.sentry` 读取，并把修复交给 `integrations.pi`。这是范围很窄的类别；不要仅仅因为工具恰好为第二个厂商格式化输出就归入此类（例如“Slack 就绪”的报告文本仍属单一厂商逻辑，而非跨厂商）。
4. **真正的表层（CLI + REPL）重复，完全不是工具逻辑** → `surfaces/shared/`（见 T-21）。这是完全不同的维度——它涉及两个*表层*都需要的展示代码，而不是 agent 可调用工具的业务逻辑放在哪里。

`tools/` 还存放并非单个工具的框架子系统——`tools/interactive_shell`（REPL 操作工具）、`tools/registry.py`（工具注册表本身）。这些保留在 `tools/` 顶层；`system` / `cross_vendor` 的划分只适用于单个工具包。

## 当前状态（截至 T-19）

应用于已有的顶层 `tools/` 包：

| 包 | 放置位置 | 原因 |
| --- | --- | --- |
| `tools/system/python_execution_tool/` | system | 通用沙箱化 Python 执行；GitHub token 导入是多个可选凭据来源之一，不是工具用途。 |
| `tools/system/sre_guidance_tool/` | system | 本地知识库检索；无厂商。 |
| `tools/cross_vendor/fix_sentry_issue/` | cross_vendor | 读取 Sentry Issue 并把修复交给 Pi 编码 agent——一个工具逻辑中涉及两个 `integrations/` 包。 |
| `tools/cross_vendor/resolve_merge_conflicts/` | cross_vendor | 读取本地 git 合并的冲突并交给编码 agent，然后提交合并——一个工具逻辑中涉及 `integrations/git` 和 `integrations/coding_agent`。 |

**已迁移到各自的厂商包**——每个单一厂商工具现在都位于 `integrations/<vendor>/tools/` 下，因此规则 1 已无例外：

- `integrations/github/tools/` — `architecture_issue_tool`、`community_followup_tool`、`git_deploy_timeline_tool`、`github_cli`、`work_status_report_tool`。
- `integrations/pi/tools/pi_coding_tool/` — 仅 Pi。
- `integrations/slack/tools/slack_send_message_tool/` — 仅 Slack。

一旦厂商的工具包的点分路径列入 `INTEGRATION_TOOL_PACKAGES`（`tools/registry_discovery.py`），该包就会被遍历；其下嵌套的工具包无需额外接线即可被发现。

## 注册表机制

`tools/system/` 和 `tools/cross_vendor/` 是普通包，由 `tools/registry.py` 对 `tools/` 的顶层遍历发现。每个包在其 `__init__.py` 中声明一个 `TOOL_MODULES` 元组（与 `integrations/slack/tools/slack_send_message_tool/__init__.py` 已为其自身 `tool` 子模块使用的清单机制相同），列出嵌套在其下一层的工具包。新增系统工具或跨厂商工具，只需把其包名加入相应的 `TOOL_MODULES` 元组——无需修改注册表代码。