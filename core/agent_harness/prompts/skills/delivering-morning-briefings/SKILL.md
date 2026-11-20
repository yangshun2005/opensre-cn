---
name: delivering-morning-briefings
description: >-
  天气 + 新闻晨间简报：获取实时天气和头条，撰写纯文本简报并投递。多步骤；执行前先加载。
metadata:
  owner: Gust
  last_changed_by: Jan
  last_changed_at: 2026-09-12
  usecases:
  - 适用于希望按需获取天气和新闻简报的用户。
  - 适用于希望在工作日将简报投递到指定目标的用户。
  requires:
  - 对天气和新闻来源的出站网络访问。
  - 如需定期投递，需配置投递目标，例如 shell 收件箱、Slack 或 Telegram。
  version: '1.2'
recurring: true
---

# 晨间报告

获取实时天气和新闻，撰写纯文本简报，投递，并提供定期计划。

## 何时使用

用于"晨间报告""晨间简报""每日简报""给我我的晨间更新"或"天气和新闻摘要"等请求。

## 工作流规则

先用只读 shell 命令获取原始输入，每个响应执行一次 `shell_run`，等两个结果都返回后再撰写或投递。在当前 agent 内完成此工作流；不要启动会产生第二份无关状态报告的调查。

绝不编造天气数值或头条。将 RSS/XML/HTML 视为中间数据。只展示撰写好的简报，不展示原始 feed 标记、XML 标签、CDATA 块或 `curl` 输出。下面的新闻获取会提取纯文本头条。

## 计划

获取过程静默运行，因此用 `update_plan` 工具跟踪进度，而不是用标题或叙述：

- 进入时、获取之前，调用 `update_plan`，原样使用以下步骤，第一步为 `in_progress`，并附一行 `explanation`（这不是诊断；不要假设表）：`Fetch weather and headlines` / `Compose the briefing` / `Deliver the briefing` / `Offer a recurring schedule`。下面的步骤 1–2 是两个连续响应，共用第一个计划步骤。
- 某步骤的工具结果返回后，调用 `update_plan` 将其标记为 `completed`、下一步标记为 `in_progress`，与下一步的单个工具调用放在同一响应中。
- 不要叙述计划或在正文中重复步骤名；shell 会渲染检查清单。撰写好的简报本身（步骤 3）和计划提议的 response_text（步骤 5）严格按下文规定保留——计划围绕它们跟踪，绝不替代它们。

## 工作流

### 1. 获取天气

用 `shell_run` 获取今日天气。使用 `quiet=true`，让 `$ curl` 行和原始 stdout 不出现在用户屏幕上——他们只需要步骤 3 中撰写好的简报，不需要同一行天气出现两次。使用用户指定的城市；若未指定，使用其配置/默认城市，否则省略位置：

```text
shell_run(command="curl -s 'wttr.in/<city>?format=%l:+%c+%t,+feels+%f,+%h+humidity,+wind+%w'", quiet=true)
```

### 2. 获取头条

用 `shell_run` 以纯文本获取当前头条——只从 feed 中提取头条标题，去掉频道标题，并限制列表长度。同样使用 `quiet=true`，对用户隐藏这次中间获取。不要在没有此提取管道的情况下获取原始 feed：

```text
shell_run(command="curl -s 'https://feeds.bbci.co.uk/news/rss.xml' | grep -oE '<title><!\\[CDATA\\[[^]]*\\]\\]>' | sed -E 's/<title><!\\[CDATA\\[//; s/\\]\\]>//' | sed '1d' | head -n 8", quiet=true)
```

### 3. 撰写简报

两个工具结果都返回后，根据实际获取的数据撰写干净、易读的简报。必需格式——仅 Markdown/纯文本，无 HTML/XML，无链接，无尖括号：

```text
Good morning! Here is your briefing.
Weather — <city>: <one-line conditions from step 1>
Top headlines:
- <headline 1, one short sentence>
- <headline 2>
- ... (3–5 bullets total)
```

### 4. 投递简报

作为本技能最后的投递动作，投递到 Slack。通过 `slack_send_message` 将整份撰写好的简报（天气/温度行和新闻头条，严格按步骤 3 的格式）发送到 Slack，即使用户未明确要求发送到任何地方。等待两个获取结果，然后在下一个响应中发送简报，`message` 中放完整的撰写好的纯文本简报——绝不放原始 feed 标记，绝不发部分报告，绝不放"准备中…"占位符。Slack webhook 绑定到单个预配置频道，因此不要询问使用哪个频道。如果用户指定其他平台（例如"发到 telegram"），也用 `telegram_send_message` 投递到那里；Slack 仍是默认目标。仅当平台未连接时才跳过。如果两个投递工具都不可用，改用上面的"投递工具不可用"规则，而不是编造命令。

### 5. 提供定期计划

简报生成后（步骤 1–3 完成；步骤 4 已尝试投递），始终提议让晨间简报定期化。绝不要将 `propose_scheduled_delivery` 作为本技能的第一个或唯一工具调用——工具会拒绝，用户只会看到计划提议而没有天气/新闻。步骤 1–3 必须先运行。调用 `propose_scheduled_delivery`，`briefing_text` 设为步骤 3 中完整的撰写好的简报（必需）。工具返回 `response_text` = 简报 + 结尾语；向用户展示（或至少以结尾语结束）。暂不要调用 `/cron`；用户的简单"yes"会展开为工具的 `slash_preview`，无需 LLM 往返。

用户接受且未覆盖时的默认值：工作日 08:00，时区已知则用其所在时区，否则 UTC；provider 与你刚投递的平台一致（默认 Slack webhook——省略 `chat_id`）。Kind 为 `recurring_skill`，`skill_name` 设为 `delivering-morning-briefings`。

Slack webhook 投递后的示例：

```text
propose_scheduled_delivery(kind="recurring_skill", skill_name="delivering-morning-briefings",
    city="<city used for the weather fetch>",
    cron="0 8 * * mon-fri", timezone="UTC", provider="slack",
    briefing_text="<FULL composed weather + headlines briefing>")
```

展示工具的 `response_text`（简报 + Want me to: …）。

仅当你有具体的 Telegram/Discord/Rocket.Chat 目标（或你已报告的 Slack #name/C…）时才传 `chat_id`。绝不编造。仅当用户此前已要求一次性简报并在本次对话中明确拒绝定期化时，才跳过提议。只运行一次的简报是演示；每个工作日都送达的晨间简报才是产品。

## 示例

### "give me my morning report"

```text
→ shell_run(command="curl -s 'wttr.in/Amsterdam?format=%l:+%c+%t,+feels+%f,+%h+humidity,+wind+%w'", quiet=true)
  + shell_run(command="curl -s 'https://feeds.bbci.co.uk/news/rss.xml' | grep -oE '<title><!\\[CDATA\\[[^]]*\\]\\]>' | sed -E 's/<title><!\\[CDATA\\[//; s/\\]\\]>//' | sed '1d' | head -n 8", quiet=true)   [both this turn — independent fetches]
→ (observe both results)
→ slack_send_message(message="<the FULL composed plain-text weather + headlines briefing>")   [next turn — mandatory Slack delivery]
```

### "morning briefing for Berlin and post it to telegram"

```text
→ shell_run(command="curl -s 'wttr.in/Berlin?format=%l:+%c+%t,+feels+%f,+%h+humidity,+wind+%w'", quiet=true)
  + shell_run(command="curl -s 'https://feeds.bbci.co.uk/news/rss.xml' | grep -oE '<title><!\\[CDATA\\[[^]]*\\]\\]>' | sed -E 's/<title><!\\[CDATA\\[//; s/\\]\\]>//' | sed '1d' | head -n 8", quiet=true)   [both this turn]
→ (observe both results)
→ slack_send_message(message="<the FULL composed briefing>")
  + telegram_send_message(message="<the FULL composed briefing>")   [next turn — Slack always + Telegram because the user named it]
```