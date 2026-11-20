# 添加工具与集成 — 完成定义

每当你添加或实质性变更以下内容时，请使用此检查清单：

- 一个工具 — 单厂商工具放在 `integrations/<vendor>/tools/` 下，跨领域工具放在 `tools/system/` / `tools/cross_vendor/` 下（参见 [tool-placement-policy.md](tool-placement-policy.md)）
- `integrations/<name>/` 下的一个集成 — 包括其配置、客户端、验证器和工具

这是详细的完成定义；请配合 [AGENTS.md](https://github.com/Tracer-Cloud/opensre/blob/main/AGENTS.md) 和 [CI.md](https://github.com/Tracer-Cloud/opensre/blob/main/CI.md) 使用。

## 1. 工具检查清单

### 通常涉及的文件

- `integrations/<vendor>/tools/<tool_name>_tool/__init__.py` — 工具包（最常见路径：工具属于某个厂商集成）
- `tools/system/<tool_name>/` 或 `tools/cross_vendor/<tool_name>/` — 仅当工具不属于特定厂商时（例如 `tools/system/sre_guidance_tool/`）
- `integrations/<name>/client.py` — 复用专用的集成 API 客户端，而不是内联请求
- `core/tool_framework/utils/` — 跨厂商复用的共享辅助代码
- `docs/<tool_name>.mdx` — 面向用户的使用说明、参数、示例
- `tests/tools/test_<tool_name>.py` — 行为与回归覆盖

工具从 **`tools/` 和 `integrations/<vendor>/tools/` 两处** 通过注册表发现，因此放置位置关乎归属，而非发现 — 参见 [tool-placement-policy.md](tool-placement-policy.md)。无论工具位于何处，它都调用集成本地的客户端/辅助函数，而不是内联传输逻辑，且绝不放在顶层 `vendors/` 或 `services/` 包中。

工具包必须是实质性的生产模块 — 不允许空的或仅用于发现的 `__init__.py`，不允许仅满足注册表导入的薄包装。任何具有验证、凭据/参数解析、传输/客户端调用、输出规范化或错误处理的工具，都应将这些关注点拆分到专注的同级文件中（`tool.py`、`models.py`、`validation.py`、`delivery.py`/`client.py`、`results.py`），让 `__init__.py` 保持为一个小型注册表入口点，仅导入公共工具对象。

### 契约与实现

- [ ] 选择最合适的简单形态（轻量工具用 `@tool(...)`，仅在需要时使用更丰富的类）
- [ ] `__init__.py` 是小型注册表入口点；非平凡工具使用同级模块承载实现关注点
- [ ] 元数据完整且准确：`name`、`description`、`source`、`surfaces`、`requires`，以及任何 `use_cases` / `outputs` / `retrieval_controls`
- [ ] `input_schema` 与实际运行时参数和必填字段一致
- [ ] `is_available` 仅在工具能真正运行时返回 `True`
- [ ] `extract_params` 正确地将已解析的集成状态映射到工具参数
- [ ] 验证、凭据/参数解析、传输/客户端调用和结果格式化相互分离，以便各自独立测试
- [ ] 可复用的传输或集成特定解析逻辑放在 `integrations/<name>/` 或 `core/tool_framework/utils/` 中，不复制到工具主体
- [ ] 失败响应具有稳定、对 agent 友好的形态；预期的外部失败（缺少配置、认证、速率限制、上游 4xx/5xx）返回结构化错误而非抛出异常 — 意外异常有意使用全局 `BaseTool` 包装器，或在具备遥测覆盖的情况下迁移
- [ ] 输出经过充分规范化，以便 agent/LLM 可靠消费
- [ ] 密钥绝不通过 `extract_params`、返回值、日志或可追踪的工具调用 kwargs 泄露；密钥/PII 输出在返回前经过 `infrastructure/safety/masking/` 处理
- [ ] 外部副作用在适当处声明 `side_effect_level`、`requires_approval` 和 `approval_reason`
- [ ] 纯读取可设置 `concurrency_safe=True`，以便与同一模型响应中的其他读取并行运行：它必须声明 `side_effect_level` 为 `NONE` 或 `READ_ONLY`，保持默认操作角色，无需审批或运行时上下文，且调用之间不共享未加锁的客户端或缓存
- [ ] 要同时出现在聊天和操作回合中，设置 `surfaces=(ToolSurface.CHAT, ToolSurface.ACTION)`

### 实时载荷解析

如果工具解析 API、MCP、日志或 webhook 载荷：

- [ ] 针对真实或有文档的上游响应形态进行验证，而不仅是理想化的 mock
- [ ] 处理实时载荷中使用的替代字段名
- [ ] 处理缺失或部分字段，不返回不可用的输出
- [ ] 在截断、尾部读取、分页或展平数据时保留重要上下文
- [ ] 上游 429 / 5xx 响应返回清晰、对 agent 友好的错误而非抛出异常
- [ ] 至少添加一个使用真实 fixture 载荷的回归测试

需要考虑的常见失败模式：分组 + 未分组的日志内容；嵌套/文件夹化的资源；分页响应；`hasMore` / 游标不匹配；内容与指针形态（`logs_content` 与 `logs_url` 风格的载荷）。

### 技能指导（可选）

工具可以通过附带 `SKILL.md` 来携带模型每次调用都会读取的工作流指导。该指导会 **追加到工具的 `description`** 中，位于 `Workflow guidance:` 标题下 — 它是永久的 schema 文本，不是旁路通道。针对一个工具的所有指导会合并并在 **2400 字符** 处截断（`tools/registry_skill_guidance.py`），因此要像描述文本一样预算：指导越长，每次请求消耗的越多。

技能指导与 harness 剧本是 **独立且可组合的** — 一个工具可以同时拥有两者。技能指导用调用规则（参数、拒绝、工具职责）重写单个工具的描述；`core/agent_harness/prompts/skills/` 下的 harness 剧本描述其周围的多步工作流。GitHub 工具（`github_cli`、`ci_fix`、`security_fix`）各自附带 `SKILL.md`；`ci_fix` 和 `security_fix` 还有工作流卡片。给两者不同的 `name`（`operating-github-ci-fixer` 与 `repair-github-ci`），并将工具级事实仅保留在工具卡片中。放置规则：`core/agent_harness/prompts/skills/AGENTS.md`。

**何时添加（两个独立维度 — 都要评估）：**

| 维度 | 添加时机 | 跳过时机 |
| --- | --- | --- |
| **工具 `SKILL.md`**（本节） | 满足以下 ≥2 项：误用代价高；从工具 `description` 不明显；跨多回合复用 | 单一清晰工具；薄/罕见的厂商；适合放入 `description` 的提示 |
| **Harness 剧本**（`core/agent_harness/prompts/skills/*/SKILL.md` + `skill_view`） | 多步 WHEN / DO NOT、同级排除、同回合 vs 下一回合，或报告模板 | 无跨回合流程的单工具提示 |

对于约 70 个 `tools/` 包中的大多数，**两者都不是**默认项。缺少 `SKILL.md` 通常是正确的。**绝不**添加每厂商一个的存根，或在失败模式共享时每个可观测性厂商一个技能（查询卫生是一个类别，不是 Datadog + Grafana + CloudWatch 的副本）。

Harness 编写模板：`core/agent_harness/prompts/skills/_template/SKILL_TEMPLATE.md`。

**文件。** 一个带 YAML frontmatter 和 markdown 正文的 `SKILL.md`：

```yaml
---
name: operating-github-cli          # required — gerund-first kebab-case, ≤ 64 chars
description: >            # required — ≤ 1024 chars; the model reads this to decide relevance
  One or two sentences: when to reach for these tools.
tools:                    # required — the registered tool name(s) this guidance applies to
  - github_cli
disable-model-invocation: false   # optional — set true to suppress attachment entirely
---

# Body — markdown workflow guidance shown after the tool's own description.
```

**注册它，否则会被静默忽略。** 加载器只读取被告知的文件：

- [ ] **显式：** 将文件路径添加到 `tools/registry_skill_guidance.py` 中的 `_skill_guidance_files()`。磁盘上存在但不在该元组中的 `SKILL.md` **永远不会被加载** — 未列出和缺失的文件会被跳过且无诊断，与忘记 `docs.json` 条目是同一个陷阱。
- [ ] 或将其放在 `tools/system/python_execution_tool/skills/*/SKILL.md` 下，该位置会被自动发现。

**检查注册表加载日志** — 加载器会警告并跳过，而不是让构建失败：

- `unknown_tool` — `tools:` 下的名称不匹配任何已注册工具；该目标被丢弃。
- `invalid_metadata` — 缺少 `name`/`description`/`tools`，`name`/`description` 过长，或 `name` 不是小写 kebab-case；整个技能被跳过。
- `parse_failed` — frontmatter YAML 格式错误。

## 2. 集成检查清单

### 通常涉及的文件

- `integrations/<name>/__init__.py` — 包门面：一个 docstring，加上调用方需要时对公共 API 的重新导出（参见 [AGENTS.md](https://github.com/Tracer-Cloud/opensre/blob/main/AGENTS.md) 中的 `__init__.py` 规则）
- `integrations/<name>/config.py` — 配置模型、`classify()`、验证器、选择器、规范化辅助函数
- `integrations/<name>/client.py` — 专用 API 客户端，当集成进行直接远程调用时
- `integrations/<name>/verifier.py` — 本地验证逻辑
- `integrations/<name>/tools/<tool_name>_tool/` — 该厂商的 agent 可调用工具（参见 §1）
- `integrations/catalog.py` — 将集成解析到共享运行时配置
- `integrations/verify.py` — 接入本地验证路径
- `docs/<name>.mdx` — 面向用户的设置、使用、验证
- `tests/integrations/test_<name>.py`，以及工具或场景涉及时的 `tests/tools/` 或 `tests/e2e/`

`integrations/<name>/` 拥有某个厂商的一切 — 配置、解析、客户端、验证器、辅助函数 **及其工具**。只有无厂商（`tools/system/`）和跨厂商（`tools/cross_vendor/`）工具位于顶层 `tools/` 下。

### 仓库中的示例

- Datadog：`integrations/datadog/`（含 `integrations/datadog/tools/`）、`integrations/catalog.py`，测试位于 `tests/integrations/datadog/` 和 `tests/tools/test_datadog_*.py`。
- Grafana：`integrations/grafana/`（含 `integrations/grafana/tools/`）、`integrations/catalog.py`、`surfaces/cli/wizard/local_grafana_stack/`，测试位于 `tests/integrations/grafana/` 和 `tests/tools/test_grafana_*.py`。
- Bitbucket：`integrations/bitbucket/` 展示了门面布局 — 一行 `__init__.py` 与 `config.py`、`client.py`、`verifier.py` 和 `tools/` 并列。

### 核心完整性

- [ ] 配置、规范化、验证器和 `classify()` 位于 `integrations/<name>/config.py` 下，让 `__init__.py` 保持为门面
- [ ] Catalog 解析 / 环境变量加载已正确接入
- [ ] 验证路径已在 `integrations/verify.py` 及所需的适配器/注册表中接入
- [ ] 集成本地客户端已添加到 `integrations/<name>/client.py` 下（仅当它进行直接远程调用时）
- [ ] 工具层已接入且稳定
- [ ] 如果集成可在本地由用户配置，CLI 设置流程已更新
- [ ] 已添加 `opensre integrations setup <name>` 对等支持，或有意记录为超出范围
- [ ] 新的必需环境变量 / 凭据已添加到 `.env.example`（绝不添加到 `.env`）
- [ ] 敏感凭据遵循下方的 [凭据解析](#credential-resolution) 契约
- [ ] `make verify-integrations` 通过

### 凭据解析

密钥环境变量名和非密钥配置遵循不同的写入/读取路径。
添加或变更集成时保持此契约。

| 表面 | 写入（向导 / 设置） | 读取（运行时） |
| --- | --- | --- |
| 集成存储（`~/.opensre/integrations.json`） | 设置时始终开启 | 优先（首选） |
| 本地凭据文件（`~/.opensre/credentials.json`） | 通过 `sync_env_secret` 写入密钥 | 环境变量未设置时通过 `resolve_env_credential` 读取 |
| `.env` / 进程环境 | 公共配置和密钥（`sync_env_values` / `sync_env_secret`） | 该层级用普通 `os.getenv`；`resolve_env_credential` 先读环境变量 |

**新代码的硬性规则**

- 绝不对密钥环境变量名（`*_TOKEN`、`*_KEY`、`*_PASSWORD`、`*_SECRET`、连接字符串及类似项）使用裸 `os.getenv`。使用 `config.llm_credentials` 中的 `resolve_env_credential`（先环境变量，再凭据文件）。
- Webhook / `*_URL` 值 **绝不** 写入凭据文件（向导将其路由到存储/`.env`，而非 `sync_env_secret`）。仅用存储 → 普通 `os.getenv` 读取。Webhook URL 通常 **嵌入** 密钥令牌 — 在日志/掩码中将其视为密码。
- 保持 `load_env_integration_services` 仅使用普通环境变量（启动安全；启动时不读取凭据文件）。
- 在 `resolve_effective` / 合并中存储仍然优先 — 环境变量/凭据文件仅是回退层级。
- 工具通过 `extract_params`（已解析的集成状态）接收凭据，绝不自行读取环境变量。执行时，工具 `injected_params` 中列出的键会覆盖模型提供的值，因此即使模型传入令牌，已验证的来源仍然胜出。GitHub 工具需要经认证的应用授权，并保留显式空令牌而不回退到环境变量。
- 设置 `OPENSRE_DISABLE_KEYRING=1` 以跳过本地文件读写（环境变量和存储仍然有效）。

规范辅助函数：`resolve_env_credential`（环境变量 → 凭据文件）、`sync_env_secret` / `save_credential`（密钥写入凭据文件和 `.env`）、`sync_env_values`（`.env` 键，包括密钥）。

## 3. 发现与边界情况

对于列出、搜索或检查资源的工具：

- [ ] 在上游支持的地方考虑文件夹/嵌套资源布局
- [ ] 大型结果集有意设置上限或分页
- [ ] 部分获取被清晰呈现（`truncated`、`fetch_error` 等）
- [ ] 时间/顺序敏感的结果在重要处保留因果顺序

## 4. 文档与测试

### 文档

- [ ] 在同一 PR 中发布或更新 `docs/` 页面/章节（新工具、CLI 命令、agent 行为或集成；以及每当工具的 API/schema 或集成的设置变更时）
- [ ] 任何新的 `docs/` 页面都在 `docs/docs.json` 中注册（不带 `.mdx` 后缀），以便 Mintlify 导航显示它

### 测试

- [ ] 配置/规范化的单元测试
- [ ] 工具契约测试，或等效的 schema/元数据覆盖
- [ ] 注册表/发现测试证明工具在预期表面上可见
- [ ] 成功和失败路径的运行时行为测试
- [ ] 涉及外部载荷时，至少一个用于实时载荷解析的真实 fixture
- [ ] 测试证明 agent 能通过正常运行时路径发现或调用该工具
- [ ] 集成接线变更时更新 `tests/integrations/`

如果测试只覆盖理想化的 mock，绿色测试是不够的。

### 最终关卡（新集成）

以上全部完成，**并且**：

- [ ] 展示集成端到端工作的截图或演示 GIF
- [ ] 已添加 E2E 测试
- [ ] CI 检查通过（参见 [CI.md](https://github.com/Tracer-Cloud/opensre/blob/main/CI.md)）

## 5. 审查者关注点

在打开或批准 PR 之前，确认最常遗漏的项目被 **明确** 处理：工具放置（§1）、实时载荷健壮性（§1）、入门/设置/文档对等（§2 和 §4）、分页/截断/部分响应行为（§3），以及覆盖真实载荷和对 agent 有用性的测试 — 而不仅是快乐路径 mock（§4）。

强制预推送命令请遵循 [CI.md](https://github.com/Tracer-Cloud/opensre/blob/main/CI.md)。