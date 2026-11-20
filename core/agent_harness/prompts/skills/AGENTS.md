# 技能发布与编写契约

## Agent 绝不编辑 SKILL.md（先读这一节）

本目录树中的每个 `SKILL.md` 都归人类所有。Agent **绝不**允许创建、编辑、重命名、移动或删除其中任何一个——不是工作流步骤，不是 `## Plan` 行，不是 `metadata` 字段，也不是一个字符的拼写修正。"绝不"包括：用户要求这样做、同目录的 `test_*.py` 或 CI 检查因此失败、卡片违反了本文件中的规则，或者改动看起来微不足道。下文描述卡片必须如何编写的所有内容都适用于人类作者；Agent 只用它来**审查**和**建议**——在回复或 PR 描述中引用当前文本和建议文本，然后保持文件不变。Agent 仍可按常规规则编辑本目录树中的非 `SKILL.md` 文件（目录代码、测试、不属于卡片改动的 `references/*.md`）。

## 技能是自然语言；脚本只是少量配角

技能的实质是它的文字：激活条件、编号的工作流步骤、完成条件、报告形态。模型阅读它，并借助共享工具目录推理完成。技能本地脚本（`scripts/<script>.py`，在 `references/script-tools.md` 中声明并由 `script_tools:` 指向）只是为了替模型承担一项机械杂务——获取载荷、重塑表格、计算计数——并把结果交回给模型解读。

限制：

- **少量脚本，而非大量。** 需要超过少量辅助脚本的技能描述的是一个工具，而不是技能；把该行为移到 `integrations/<vendor>/tools/` 或 `tools/system/`，让卡片调用它。
- **脚本中不做决策。** 辅助脚本不选择下一步、不按业务规则分支、不判断步骤是否完成。这些判断留在工作流文字和模型中。
- **不做端到端自动化。** 把脚本串起来让工作流在步骤之间无需模型推理即可运行，会把技能变成确定性流水线；那不是技能。
- **仅标准库、单一用途、JSON 进 / JSON 出**（见下文 `script_tools` 契约）。脚本一旦增加依赖或第二个用途，就说明它属于工具层。

审查或提议技能时，问一问每个脚本是否都可以移除，而该步骤仍能用模型可遵循的一句话描述。如果技能只因脚本才能工作，那它就是一个披着卡片外衣的工具。

## 运行时模块

保持技能根目录作为公共导入门面和工流资产的家。目录发现、Schema 校验、名称和菜单属于 `catalog/`；Markdown 解析、正文/引用加载和索引渲染属于 `content/`；周期性修订固定属于 `scheduling/`。本包之外的消费者导入其公共门面或 `scheduling` 门面。

移动资源读取器时，保留用于发现的技能根目录并包含包含性检查。测试替换内置资源时，通过 `clear_skills_caches()` 清除活动目录。

`snapshot/` 拥有进程实际提供的目录。`SkillCatalogSnapshot` 从单一根目录构建一次（已校验卡片、已渲染正文、引用、索引、每技能摘要），之后不再重读文件。`active_skill_catalog()` 按以下顺序选择根目录：

1. `OPENSRE_SKILLS_DIR`（独占；文件变化时重读）。
2. 最新的已存储发布版本，需通过校验、声明受支持的 `SKILLS_API_VERSION`、包含每个内置技能名称，并以零诊断构建。发布版本整体接受或整体拒绝。
3. 内置目录树。

`run_turn` 每轮绑定一个快照，因此新拉取的发布版本或编写编辑在下一轮生效，绝不在轮中生效。每次操作只读一次目录（`active_skill_catalog().current()`），而不是调用多个可能跨越切换的查找。

## 实时发布

技能无需二进制发布即可发布：
- `opensre skills push`（OpenSRE 员工，或合并时的 `.github/workflows/skills-sync.yml`）把每个 `metadata.version` 比线上版本新的技能包发送到 OpenSRE 应用，由后者签名一个新的不可变发布版本。
- 每台主机在后台拉取（`infrastructure/skills_registry/`，每五分钟）到 `~/.opensre/skills/`。
- `opensre skills rollback` 把较早的发布版本重新发布为最新版本。
- 客户端用 `config/constants/skills.py` 中的 `SKILLS_RELEASE_PUBLIC_KEYS` 验证 ECDSA P-256 签名。签名消息由 `test_signing_message_matches_the_server_contract` 和 webapp 的签名器测试固定；两者一起改，或都不改。

由于版本门控决定发布什么，不提升 `metadata.version` 的编辑不会被发布。发布版本必须保留主机代码依赖的每个技能名称；二进制会拒绝丢弃内置技能的发布版本。

## 设计参考

Claude 技能设计：
- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices


NVIDIA 发布检查清单：
- https://docs.nvidia.com/skills/release-checklist


NVIDIA 技能卡片指南：
- https://docs.nvidia.com/skills/skill-cards

## 主工作流 frontmatter

`catalog/schema.py` 定义运行时 Schema。CI 通过 `read_skill_catalog()` 读取每张原始卡片，并在任何诊断上失败；它不能只校验过滤后的运行时目录。运行时发现会记录无效卡片并排除它们，因此一张坏卡片不会阻止启动。贡献者文件和报告模板不是技能条目。

| 字段 | 契约 |
| --- | --- |
| `name` | 必填的稳定小写 kebab-case 标识；在卡片间唯一。 |
| `description` | 必填的非空发现文本；说明行为和激活条件。 |
| `metadata` | 必填块，如下所述；未知字段会被拒绝。 |
| `getting_started`、`demo_order` | 可选对：非空演示标签和正整数顺序。标签和顺序必须唯一。 |
| `includes` | 可选的本地 Markdown 文件，作为指令追加，每个解析文件一次。 |
| `recurring` | 可选布尔值，默认 `false`；启用周期性技能运行器。实际 cron 和时区属于计划任务。 |
| `script_tools` | 可选指针，指向 `references/script-tools.md`，声明技能本地辅助工具。 |

每个 `metadata` 块要求 `owner`、`last_changed_by`、`last_changed_at`、`version`、`usecases` 和 `requires`。`usecases` 和 `requires` 是非空字符串的非空列表。前置条件缺失、为空或格式错误时 CI 失败。包含冒号加空格的条目要加引号，以免 YAML 把它们变成映射。`version` 写成带引号的 `MAJOR.MINOR` 字符串（`"2.1"`；递增规则见下文），`last_changed_at` 写成不带引号的 ISO 日期，不晚于今天。

- `usecases` 指明目标用户和具体工作流，每个不同场景一条。
- `requires` 指明必要的工具、凭据、访问级别、本地状态和执行环境。明确说明条件性要求。对于不需要外部前置条件的技能，用一条说明。绝不要把秘密值放进卡片。元数据记录要求；工具执行要求。
- 所有权和变更跟踪遵循下文规则。这些字段支持人工审查，尽管它们不驱动执行。这是 OpenSRE 对 NVIDIA 技能卡片用例和需求内容的 YAML 表示；它不声称替代完整发布记录。

`metadata.type`、`metadata.dependencies`、`metadata.prerequisite_for`、工作流 `tools`、`pre_execute`、`after_tool` 和顶层 `references` 不受支持。卡片不声明入口 Hook；见下文入门菜单段落。工具使用卡片保留其独立 Schema 和必需的 `tools` 字段。

把脚本工具定义放在 `references/script-tools.md`。在 `SKILL.md` frontmatter 中只保留一行指针 `script_tools: references/script-tools.md`。运行时从该文件加载并校验定义。

该引用的 YAML frontmatter 包含一个 `script_tools` 列表。每个条目有 `name`、`script`（同级 `scripts/<script>.py` 文件名）、`description`，以及一个对象 `input_schema`，含标量参数和 `additionalProperties: false`。脚本辅助程序接收一个 JSON 对象作为第一个命令行参数，并输出一个包含 `ok` 的 JSON 对象；非零退出即失败。保持辅助程序仅用标准库、捕获子进程输出，并在结果中返回诊断。主机负责超时、取消和可见的失败报告。辅助程序是变更性、顺序性工具；它们不能遮蔽常规工具目录。加载技能会改变下一个请求的工具列表。切换技能、结束其计划或开始新请求会移除其辅助程序。执行时会再次检查活动会话，包括来自较旧工具快照的调用。

`includes` 相对于技能目录、其父目录或技能根目录解析文件。解析后的文件必须留在技能树内。目录、URL、自包含和另一个 `SKILL.md` 无效。支持性引用保持为 Markdown 链接。可选的 `references/<slug>.md` 文件仍通过 `skill_view(name=..., reference=...)` 按需加载；它们不是自动包含。

入门主技能的入口菜单不在其卡片中声明。加载器（`catalog/demo_menu.py`）在代码中构建它：标题是 `config.constants.skills.ONBOARDING_MENU_TITLE`，自由文本被禁用。当四个入门子技能都存在时，第一个菜单是 `OUTCOME_MENU_OPTIONS` 中的结果行（分析仓库、保持 CI/CD 健康、打开 shell）。自动化行不是技能；shell 打开 `AUTOMATION_MENU_OPTIONS` 并只提交该叶子标签。交接把这些叶子标签映射到技能名称。任何其他子集仍按 `demo_order` 使用每个子技能的 `getting_started` 标签，后跟 shell 的 Skip 选项。子技能数量必须为一到七，否则主技能被排除并给出诊断。子技能不能复用保留的 Skip 标签。主机在技能入口（`tools/interactive_shell/actions/skill_entry.py`）通过真实的 `ask_user_choice` 执行器打开菜单，并在 `skill_view` 结果的 `entry_menu` 键下报告 `queued`、`suppressed` 或 `unavailable`。

在此之前，尚未激活的技能要通过其主机拥有的前置条件（`config/constants/skill_prerequisites.py` 中的 `SKILL_PREREQUISITES`，由 `infrastructure/harness_providers/skill_prerequisites.py` 按 id 注册的检查）。未满足的前置条件会扣留正文、排队设置菜单（`skill_prerequisite_gate.py`），并在 `prerequisite` 键下报告；设置完成后 shell 重新提交被阻止的消息。在入门菜单中选择的演示会在菜单自身的后续（演示仓库问题）之前检查，因此那里也是设置优先。每个入门技能都有一行，即使是空行。

## 窄用途

技能有窄而具体的目的。
SKILL.md 描述技能应在何时激活。
工具、shell、网络、文件、环境和 MCP 能力在使用时声明。

## 每步一个动作

每个工作流步骤只做一件事，并有一个可观察的完成条件。把不同动作拆成单独的编号步骤。

顺序执行步骤及其工具调用。完成当前动作，包括交付任何面向用户的输出，再开始下一个。不要用"同时"、"在同一时间"或"在同一响应中"等措辞把不同步骤耦合起来。

运行时（`core.tool.execution`）按模型列出的顺序逐个运行一个响应中的工具调用——绝不并发。一个响应可以批量包含多个独立调用；依赖较早调用结果的调用属于下一个响应。每个工具在其契约上声明角色（`ToolRole`，取代旧的 `parallel_safe` 标志）：

- `ACTION` 和 `BOOKKEEPING`（`update_plan`、`memory_remember`、`session_goal_complete`）调用可以共享一个响应。卡片应说明这一点——"在与其工具调用相同的响应中把步骤标记为 `in_progress`"——而不是让模型为每次计划写入花费一个单独轮次。`scheduling-github-ci-repairs` 的一次实际运行曾仅计划写入就花费九个单独的 `update_plan` 轮次（约 90 秒）。
- `TURN_ENDING`（`ask_user_choice`）把轮次交给用户，是其响应中的**唯一**动作。簿记可以随它一起运行并先执行，因此标记菜单步骤的 `update_plan` 属于菜单的响应，而不是之前的单独轮次（主机会拒绝单独的推进计划）。把动作与菜单批量放在同一响应中不会执行任何内容，并为每个调用返回相同错误。

一旦某个调用的结果结束轮次（排队的菜单、待处理的批准、主机取消），批次中它之后的调用会被跳过，并返回错误结果说明它们未运行。

因此，一个步骤内的独立只读检查（例如身份加调度器）可以共享一个响应，或一个同时运行两者的 shell 命令。

报告交付和询问下一步是分开的动作：先用 Markdown 文本回复报告；只有在其显示之后，下一步才可以打开菜单。说"报告已就绪"或更新计划并不交付报告。在进一步计划步骤之前的报告步骤必须在 `update_plan` 中带有 `deliverable: true`（告诉卡片的读者设置它）；只有当当前或下一步被标记时，主机才会在计划结束前显示纯文本回复，并把任何其他情况视为过早停止。

检查结果的步骤带有 `verifies: true`（告诉卡片的读者设置它）。它是 shell 唯一标记为 `(verify)` 的步骤，只有在其自身工具返回后才完成，而纯文本的最后一步只有在它之后才免费关闭。因此，最后一步是文字（解释、报告）的卡片要指明哪个较早步骤进行验证，否则计划无法标记为完成。

不要把这一教导应用于入门演示 A（`onboarding-github-ci/a-analyzing-github-ci-performance/`）。保持卡片、引用和工作流脚本与 `main` 上一致。当主机或技能加载器变化时仍要运行 `test_workflow.py`；如果失败，修复产品或 harness，而不是卡片。演示 B 和 C 未冻结。

## 默认推荐计划

每个工作流 `SKILL.md` 在其目的陈述之后、`## Workflow` 之前直接带有 `## Plan` 部分。它是 Agent 一读到技能就通过 `update_plan` 加载到实时计划中的默认计划，因此用户一开始就能看到整个流程，并按步骤跟踪进度。

该部分有两部分：

1. 一段简短文字，告诉 Agent 调用 `update_plan`，用下面的步骤创建或修订指定的实时计划，把报告和后续作为单独的计划项，把已满足的步骤标记为 `completed`，并在每个步骤的完成条件满足时更新状态。
2. 一个检查清单，每个编号的 `## Workflow` 标题对应一行 `- [ ] Step N. …`，顺序相同、数量相同。每行用一句话说明该步骤的结果及其使用的工具（`scan_local_git_workspace`、`ask_user_choice`、……）。

一个含八个独立步骤的工作流示例形态：

```markdown
## Plan

After reading this skill, use `update_plan` to create or revise the live
CI/CD Reliability Progress plan using the eight numbered workflow headings
below as its steps. Keep reporting and follow-up as separate plan items.
Mark already-satisfied steps `completed` and update statuses when each
step's completion condition is met.

- [ ] Step 1. Scan local repositories with scan_local_git_workspace.
- [ ] Step 2. Select a repository using ask_user_choice.
- [ ] Step 3. Read the metric and benchmark references with skill_view.
- [ ] Step 4. Collect complete 30-day workflow, rerun, and merged-PR history.
- [ ] Step 5. Calculate the required metrics from the collected history.
- [ ] Step 6. Validate the calculations against the coverage and consistency requirements.
- [ ] Step 7. Respond with the report as a Markdown table and its coverage notes.
- [ ] Step 8. After the report is shown, use ask_user_choice to offer the next step.
```

规则：

- 计划和 `## Workflow` 标题是同一列表的两处。添加、删除或重排工作流步骤时，要在同一次编辑中改变计划；计划与标题不一致的技能是不完整的。
- 每个工作流步骤说明其完成条件（"Complete when …"），以便 Agent 能把匹配的计划项移到 `completed` 而无需猜测。
- 流程可能合理跳过的步骤（例如请求中已命名的仓库）要在步骤正文中说明，并告诉 Agent 把跳过的计划项标记为已满足，而不是删除它们。
- 工具使用卡片描述一次调用，而不是流程，不带计划。

## 成功标准

每个技能，无论工作流还是工具使用，都在 `config/constants/skill_success.py` 中有一个或多个可验证的成功标准。标准用反引号指明一个可观察结果——工具字段、返回值或回复中的事实。当卡片尚未包含 `## Success criteria` 部分时，主机会追加一个。添加没有标准的技能会使 `test_every_skill_has_verifiable_success_criteria` 失败。卡片可以自己带有同一部分；主机不会追加第二份。

## 同目录工作流测试

编排多步骤工作流的主要技能必须在其 `SKILL.md` 旁保留端到端 `test_*.py`。用已发布的技能演练真实 Agent 循环，检查工具调用顺序、参数和用户选择暂停。为离线 CI 编写脚本化的模型响应和外部工具 I/O；保持技能加载和主机 Hook 真实。把技能目录纳入默认 pytest 发现和 CI 分片，并在更改工作流时运行其测试。

这些文件位于 `core/` 内，因此层契约适用于它们：绝不导入 `tools`、`integrations` 或 `surfaces`。通过 `core.agent_harness.tools.action_tools.get_action_tool` 解析真实动作工具（`ask_user_choice`、`skill_view`）；其背后的注册表由 `tests/harness_providers_plugin.py`（从 `pytest.ini` 加载）在每个测试周围安装，而不是由 `tests/conftest.py` 安装，后者无法到达本目录树。


## 技能元数据所有权、变更日期和版本

每个