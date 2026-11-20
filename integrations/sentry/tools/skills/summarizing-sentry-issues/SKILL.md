---
name: summarizing-sentry-issues
description: 按主题和影响汇总 Sentry Issue，可选附带 uptime 汇总。适用于概览、晨间摘要、可靠性问题。
tools:
  - search_sentry_issues
  - get_sentry_issue_details
  - list_sentry_issue_events
  - get_sentry_uptime_digest
---

# Sentry 摘要

仅限 Sentry（#3 按需、#10 晨间摘要）。多源场景：先在此完成 Sentry 部分，再建议进行多源调查。

## 1. 获取 Issue

`search_sentry_issues`，`query: "is:unresolved"`：
- **24h** — 晨间摘要、夜间（#10）；**7d** — “本周”、概览。

每页最多 100 个分组。用 `digest.page_saturated` 与 `page_complete` 区分。当 `completeness` 为 `empty` 时，报告为零——除非被要求，否则不要扩大范围。

## 2. 获取 uptime

`get_sentry_uptime_digest`。若 `uptime_watch_active` 为 false 则跳过。仍处于宕机状态的放在 Issue **之前**，已恢复的放在 Issue **之后**。

## 3. 分类

`digest.structural_clusters` → 业务主题，附 `sample_short_ids`。

## 4. 排序

用 `digest.priority_candidates` + `business_impact_score`，而非原始 `count`。优先 userCount、阻塞项、回归；引用 `impact_reasons`。

## 5. 补充信息

仅前 3–5 项：`get_sentry_issue_details` + `list_sentry_issue_events`（limit 10）。

## 6. 汇总

- 标题 + 摘要要点计数（Issue + Uptime）。
- `[DOWN]` 置于 Issue 之上；Issue 分组 + 优先级表格；`[RECOVERED]` 置于其后。

## 陷阱

- `count` = 事件数/分组；`issue_count` = 返回的分组数
- `page_saturated` = 第一页；`completeness: empty` = 零分组
- Uptime 需要运行中的 watch 计划——无 watch 则无数据