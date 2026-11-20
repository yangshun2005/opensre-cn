---
name: <verb-ing>-<object>
description: >-
  <说明此工作流做什么、面向谁、以及 agent 应在何时加载它。>
metadata:
  owner: <原作者>
  last_changed_by: <本次编辑者>
  last_changed_at: YYYY-MM-DD
  version: "1.0"
  usecases:
    - <目标用户与具体工作流。>
  requires:
    - <所需工具、凭据/访问级别或执行环境。>
# 可选布尔资格；定时任务提供 cron 和时区。
# recurring: true
# 可选演示归属；主菜单从这些字段派生选项。
# getting_started: <菜单标签>
# demo_order: 1
# 可选的本地指令包含，区别于按需加载的 references/ 文件。
# includes:
#   - common/shared-rule.md
---

# <工作流标题>

<说明具体产出。>

## 计划

使用 `update_plan` 跟踪下方编号的工作流步骤。保留已满足的步骤并标记为已完成。每个步骤的完成条件满足时更新其状态。将检查产出的步骤标记为 `verifies: true`；纯文本的最后一步只有在其运行后才算结束。

- [ ] 步骤 1. <使用所属工具解析所需输入。>
- [ ] 步骤 2. <使用指定工具执行工作。>
- [ ] 步骤 3. <使用指定工具检查产出；此步骤带有 `verifies: true`。>
- [ ] 步骤 4. <交付产出。>

## 工作流

### 1. <解析输入>

<使用已知范围，仅询问缺失的决策，并指明检查前置条件的工具。列出此步骤已满足的条件。>

完成条件：<可观察的输入与前置条件>。

### 2. <执行工作>

<指明执行工具及所需参数。将分支相关的参考资料放在显式链接后，并说明何时阅读。>

完成条件：<可观察的工具结果>。

### 3. <交付产出>

<说明输出格式及支撑结果所需的证据。>

完成条件：<用户已收到结果>。

## 成功标准

- [ ] `<tool>` 返回 `<observable field>`，或回复中说明 `<the stopping fact>`。

在 `config/constants/skill_success.py` 中此技能的 `name` 下注册相同的行。

<!-- 编写说明：复制到 skills/<name>/SKILL.md，替换占位符，
     设置实际的未加引号的变更日期，并删除这些说明。遵循
     ../AGENTS.md。单次调用工具用法卡片使用其独立 Schema。
     卡片不声明入口 Hook；在编号工作流中将每个问题（包括第一个）
     描述为 ask_user_choice 步骤。
     发布前运行原始卡片校验测试及相关工作流测试。 -->