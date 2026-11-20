# 集成

`integrations/<vendor>/` 负责单个供应商的全部内容：配置规范化、验证器、客户端、辅助函数、目录/存储接线，以及该供应商的工具（`integrations/<vendor>/tools/`）。新增集成前，请先阅读 [docs/adding-tools-and-integrations.md](../docs/adding-tools-and-integrations.md) 了解完整的完成标准。

## 工具使用 SKILL.md 卡片由人工维护

`integrations/<vendor>/tools/**/SKILL.md` 卡片**绝不**允许由 agent 创建、编辑、重命名、移动或删除——frontmatter 更新、测试修复、直接请求均无例外。agent 只能在回复或 PR 描述中**建议**修改（当前文本 → 建议文本），并保持文件不变，由人工应用。规则同根目录 `AGENTS.md`。

Skill 是自然语言卡片，不是确定性工具。确定性的供应商行为——API 调用、解析、重试、决策逻辑——应放在 `integrations/<vendor>/tools/` 下的真实工具中，由 skill 描述如何使用。不要把这些逻辑塞进 skill 的辅助脚本；skill 只能携带少量小型辅助函数，且不得替代模型的推理。完整约定见：`core/agent_harness/prompts/skills/AGENTS.md`。

## 配置流程——供应商如何变为"已配置"

配置在三个界面运行——`opensre onboard`（向导）、`opensre integrations setup <vendor>`（CLI）以及交互式 shell 操作工具。它们仅在如何**收集**值上不同；后续处理完全一致，集中在一处：[`setup_flow.py`](setup_flow.py)。

- **声明 `IntegrationSetupSpec`**：在 `integrations/<vendor>/setup.py` 中（字段 + 验证器），然后将收集到的值交给 `apply_setup()`。它会验证、可选地解析，并持久化到**所有层级**——集成存储、keyring（密钥）以及项目 `.env`。
- **绝不在界面处理器中手写 `upsert_integration` + 环境变量写入。** 仅写存储 vs. 三层写入的差异正是本模块要防止的 bug：运行时优先解析存储并隐藏问题，但部署预检读取 `.env`，会看到半配置的集成。
- **`SetupField`** 携带 `name`/`label`/`prompt`/`env_var`/`default`/`required`/`secret`/`constant`。层级由 `env_var` 自动推导（`*_TOKEN`/`*_KEY`/`*_PASSWORD` → keyring，否则 `.env`）；字段不能自行选择层级。

## 跨字段规则放在验证器中，而非配置 Hook

跨多个字段的规则——二选一、XOR、全有或全无——应放在 `integrations/<vendor>/verifier.py` 中，**而非**仅配置时的检查。spec 上刻意**没有 `validate` 钩子**。

原因：`opensre integrations verify <vendor>`（健康检查）只调用验证器。仅配置时的检查会导致配置与健康检查对"已配置"的定义不一致。在验证器中执行规则可提供唯一定义，两者共用。

- 先例：`verify_rocketchat` 拒绝不完整的"webhook **或** server-url/token/user-id 三件套"。
- UI 选择器**不是**验证保证——agent 或手动编辑的存储会绕过它，因此验证器仍须拒绝错误组合。

## 模式选择器（`SetupMode`）

当供应商有互斥的配置路径（某种认证方式；webhook vs. Socket Mode）时，在 spec 上声明 `SetupMode` + `mode_prompt`，而不是平铺提示每个字段。选择器限定收集哪些字段；所选模式之外的字段会被**清除，而非提示**，因此选择一种模式会关闭其他模式。验证器仍执行真实组合校验（见上文）。

## 指引

| 文件 | 内容 |
| --- | --- |
| [`setup_flow.py`](setup_flow.py) | 约定：`apply_setup`、`SetupField`、`SetupMode` 以及 `resolve`/`finalize` 钩子 |
| `<vendor>/setup.py` | 各供应商的 spec（如 `rocketchat`、`telegram`） |
| `<vendor>/verifier.py` | 验证**及**跨字段规则 |
| [docs/adding-tools-and-integrations.md](../docs/adding-tools-and-integrations.md) | 完整清单 / 完成标准 |