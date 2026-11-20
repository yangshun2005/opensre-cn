---
name: summarizing-posthog-analytics
description: 将 PostHog 产品分析汇总为按指标划分的团队脉搏。适用于 PostHog 使用情况概览、按指标汇总、产品分析摘要，或“本周发生了什么”类报告。
tools:
  - list_posthog_tools
  - call_posthog_tool
---

# PostHog 摘要

面向聊天交付的 PostHog 产品分析报告。生成某时间窗口内按指标划分的脉搏——哪些指标有变化、为何重要——而非原始仪表盘导出。单一数据源（PostHog MCP）；在此完成，之后可选建议多数据源的后续跟进。

需要 **PostHog MCP** 集成（`posthog_mcp`）。仅支持 REST 的 `posthog` 集成无法支撑本技能。

## 1. 发现

先调用一次 `list_posthog_tools`，参数 `name_filter: "insights dashboard sql events"`，找到查询接口。然后在任何聚合查询之前调用 `read-data-schema`（或等效的 Schema 工具），只引用它确认存在的事件和属性。未在 Schema 中见过某属性（例如 `properties.$mcp_error`）之前不要引用它——HogQL 会拒绝针对未知属性的查询。要拉取指标数据，使用 `call_posthog_tool`，`tool_name: "execute-sql"`，**SQL 作为顶层字符串**：

```json
{
  "tool_name": "execute-sql",
  "arguments": { "query": "SELECT count() FROM events WHERE event = '$pageview'" }
}
```

切勿嵌套 SQL（`{"query": {"query": "SELECT …"}}`）——PostHog 会以 `parameter "query" must be of type string` 拒绝。当用户指定某个仪表盘时，使用 insight/仪表盘工具。

## 2. 获取指标

将用户措辞映射到时间窗口：

- **24h** — “今天”、“隔夜”、早间报告
- **7d** — “本周”、默认周报
- **30d** — “本月”、月度汇总

**每个指标运行一条小型 HogQL 查询**——不要将多个指标合并到单条聚合语句，也绝不要将事件与合成的“窗口”行交叉连接。这种合并形态是 PostHog 报 `unknown error running this query` 最常见的原因。只返回**聚合数字**（两列：当前 / 上一期）——绝不返回事件级行。用单个时间过滤上的显式条件聚合计算两个窗口，例如：

```sql
SELECT
  countIf(timestamp >= now() - INTERVAL 7 DAY) AS current,
  countIf(timestamp >= now() - INTERVAL 14 DAY
          AND timestamp < now() - INTERVAL 7 DAY) AS previous
FROM events
WHERE event = '$pageview'
```

按指标替换 `event = '...'`（活跃用户则用 `count(DISTINCT person_id)`）。若查询仍失败，用更简单的 `SELECT count() ... WHERE timestamp >= now() - INTERVAL 7 DAY` 对该单个指标重试一次，然后将该指标报告为失败，而不是中止整份报告。

用户未指定指标时的默认指标集（限制为 Schema 确认存在的事件）：

- 活跃用户（`count(DISTINCT person_id)`）
- 关键事件（`$pageview`、会话，或项目的热门自定义事件）
- 新注册 / 新用户——**仅当** Schema 确认存在注册事件（`user_signed_up`、`signed_up`、`signup` 等）。绝不要用 `user_signed_in` / 登录 / `$identify` 替代注册或“已注册”留存队列——若注册未经验证，跳过该指标并说明。
- Schema 中出现的值得注意的自定义事件

## 3. 计算各指标变化

对每个指标：当前值、上一期值、绝对变化、百分比变化、方向（上升 / 下降 / 持平）。当百分比变化任一方向超过约 15%，或某值意外归零时，将该指标标记为值得注意。

## 4. 汇总（团队脉搏）

以 **I found:** 开头：一行范围摘要（窗口 + 项目 + 指标数）。然后是按指标表格：

指标 | 当前 | 上一期 | 变化 | % | 趋势

- 趋势：使用上升 / 下降 / 持平文字，绝不只依赖颜色。
- 表格之后，指出 1-3 个最值得注意的变化项，附一句简短的“为何重要”，并在数据支持时给出一个明确的下一步行动。
- 当某窗口无数据时，对该指标明确说明——绝不悄悄扩大窗口或编造数字。

## 陷阱

- HogQL 时间范围是相对的；在报告中说明绝对窗口。
- 区分“指标返回零”（真实）与“查询失败”（报告失败，不要将其呈现为零）。
- `execute-sql` 参数必须是 `{"query": "<sql string>"}`。嵌套的 `{"query": {"query": "…"}}` 会在读取任何数据前失败——解包并重试一次，若仍报错则将该指标标记为失败。
- MCP 服务器暴露 240+ 个工具；调用前用 `name_filter` 缩小范围——绝不要输出完整列表。
- 每个指标组一条有界查询；事件级扫描开销大。
- `unknown error running this query` 几乎总意味着语句过于复杂（多个指标合并、交叉连接，或未经验证的属性）。通过拆分为一条简单的按指标查询来恢复，而不是放弃。