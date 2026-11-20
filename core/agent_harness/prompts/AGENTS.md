# prompts/ — 单智能体提示词组装

## SKILL.md 文件与系统提示词：只可建议，绝不可编辑

智能体**绝不**允许修改 `SKILL.md` 文件——正文不行，frontmatter 不行，`version` / `last_changed_by` / `last_changed_at` 不行，重命名或移动不行，新增卡片不行，删除也不行。本目录下的系统提示词 `opensre_system_prompt.md` 同样如此：不得修改其文本，不得重命名、移动或删除。下文的格式与元数据规则描述的是**人类**作者必须做的事；对智能体而言，这些只是评审标准。当你认为某个 `SKILL.md` 或系统提示词需要修改时，请在回复或 PR 描述中写出提案（引用当前文本，给出建议文本），并保持文件原样。**在任何情况下**，智能体都不得以其他方式修改这些文件。唯一例外是字面意义上的 Ctrl-H 替换：用户给出确切的当前词或句子以及确切的替换内容，智能体逐字符替换，文件其他部分不做任何改动（参见根目录 `AGENTS.md`）。

## 技能以文字为主，脚本为辅

技能主要是自然语言：模型阅读、推理并用常规工具目录执行的编号步骤。辅助脚本是例外——至多几个小助手，每个只处理一项机械性杂务（收集、重塑、计数），其结果由模型解读。技能不得变成伪装的确定性工具：不得有决定下一步的脚本，不得有端到端运行工作流的脚本链，不得有本应放在 `integrations/` 或 `tools/` 下的厂商逻辑。当提议的脚本会取代模型的判断时，应改为提议一个工具或一个文字步骤。

## 目录结构

| 包 | 职责 |
|---------|------|
| `kernel/` | `PromptEnvelope` / 层级 / `SurfaceProfile` — 不感知智能体路径 |
| `grounding/` | `DefaultPromptContextProvider`，指明会话各轮次运行在哪个 surface 上；组装器读取轮次快照，而非该 provider |
| `action/` | 工具调用型智能体的提示词组装与策略（`goal_kernel.py` 是宿主拥有的目标规则；技能不能覆盖它） |
| `memory/` | 文本历史回退渲染（结构化历史关闭时的 RECENT CONVERSATION、PRIOR ACTION FACTS）以及肯定式追问扩展 |
| `runtime_facts/` | 提示词中的运行时元数据事实行 |
| `skills/` | 渐进式技能索引 + markdown 正文（`catalog/` + `content/` + 工作流 Markdown） |
| `loop_templates/` | 定时循环指令（`cron add --template`）：frontmatter 加至多十句话，每次触发都重新运行 |
| `rules.py` | 共享规则片段（叶子） |
| `system_prompt.py` + `opensre_system_prompt.md` | 共享系统基座的加载器与相邻 Markdown |

根目录 `__init__.py` 是常用导入的薄门面。

## 依赖规则（无环）

```
kernel  ←  memory, runtime_facts, skills, rules, grounding, system_prompt
        ↑
      action
```

- 叶子可导入 `kernel`（仅当存在明确归属时，叶子之间才可互相导入）。
- action 包可导入叶子 + `kernel`。

## 来源标注

`PromptBlock.provenance` 应指明本目录树下的归属模块（例如 `core.agent_harness.prompts.opensre_system_prompt.md`）。

## 技能正文格式

使用普通 Markdown，参照
[`skills/onboarding-github-ci/SKILL.md`](skills/onboarding-github-ci/SKILL.md)：

- 以 `#` 标题和简短的目的说明开头。
- 使用描述性的 `##` 章节，在顺序重要处使用 `###` 工作流步骤。
- 用短段落或列表写直接指令；避免装饰性横幅、全大写章节标签和重复的工具清单。
- 命令和精确输出示例放在行内代码或围栏代码块中。
- 修改呈现方式时，保留 frontmatter、工具契约、精确选项标签、授权边界和必需的输出格式。
- 技能不声明入口 Hook。唯一的宿主打开的入口菜单（`onboarding-github-ci` 演示选择器）是由 `skills/catalog/demo_menu.py` 中子项的演示元数据构建的目录数据；宿主在任何模型步骤之前打开它。不要把它重述为模型必须复现的文字。
- 在编号工作流中描述流程中途的菜单；模型在完成前一步后调用 `ask_user_choice`。
- 同级技能共享的规则应放在 `includes:` 下列出的 markdown 文件中（在技能树内解析），而不是复制到每个正文中。

## 技能元数据归属

每个 `SKILL.md` frontmatter 的 `metadata` 块记录两个人：

- `owner` — 创建该技能的人。使用其姓名，绝不用 `Tracer Team` 这类团队标签。创建时设定一次，之后他人编辑技能时不得更改。
- `last_changed_by` — 最近修改该技能的人的姓名。
- `last_changed_at` — 该修改的 ISO 日期（`YYYY-MM-DD`）。
- `version` — 带引号的 `MAJOR.MINOR` 字符串；每次编辑将小数点后的数字加一（`"2.1"` → `"2.2"` → … → `"2.15"`）。主版本号仅在卡片契约发生破坏性变更时才变动。

`last_changed_by`、`last_changed_at` 和 `version` 一起变动：任何编辑技能（正文或 frontmatter）的人都必须在同一次变更中更新这三行；遗漏任何一行的技能编辑都算不完整。

```yaml
metadata:
  owner: Vincent
  last_changed_by: Jan
  last_changed_at: 2026-09-09
  version: "2.2"
```

完整规则：[`skills/AGENTS.md`](skills/AGENTS.md)。

当你接触的技能其 `owner` 仍是团队标签时，如果你知道原始作者姓名就替换为该作者姓名；否则保持原样并在 PR 描述中注明，不要猜测。