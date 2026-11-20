---
name: measuring-github-star-velocity
description: 使用 stargazers API 计算 GitHub 仓库在滚动时间窗口内的 Star 增长速度。
tools:
  - execute_python_code
---

# GitHub Star 增长速度

仅在专用工具 `get_github_star_history` 不可用时使用本技能。该工具可用时，优先使用它来获取 GitHub Star 增长速度、近期窗口内新增 Star 数、逐日 Star 数、Star 增长率或仓库的类似指标。

## 前提条件

1. 在 `execute_python_code` 上设置 `allow_network: true`。
2. 从子进程环境读取 `GITHUB_TOKEN`（配置 GitHub 凭据时注入）。切勿打印该令牌。
3. 从仓库 URL 或用户输入中解析 `owner/repo`。

## API 方法

1. 通过 `GET /repos/{owner}/{repo}` 获取当前 `stargazers_count`。
2. 通过 `GET /repos/{owner}/{repo}/stargazers?per_page=100` 获取带时间戳的 stargazers，请求头使用 `Accept: application/vnd.github.v3.star+json`。
3. 每个条目包含 `starred_at`。统计 `starred_at` 落在请求窗口内的 Star。

## 关键分页陷阱

Stargazers 按**最旧优先**返回，而非最新优先。当某页最新的条目已落在窗口之外时，**不要**在第一页后就停止——这会产生错误的零值。

为保证响应速度，根据 `stargazers_count` 和 `per_page=100` 计算最后一页，按 `page` 编号降序显式获取最新页，一旦某页最旧的 `starred_at` 早于窗口起点就停止。仅在用户明确要求全量历史或审计级导出时，才执行完整的历史扫描。

## 报告前的合理性检查

以下结果应视为可疑，在回答前用完整分页扫描重跑：

- 拥有数千 Star 且近期活跃的仓库，过去 24 小时新增 `0` 个 Star
- 增长速度暗示 Star 减少，但并无明确的取消 Star 请求
- 有近期基线时，计数与 `stargazers_count` 的增量严重不符

结果看起来有误时，扩大扫描范围（所有页），打印观察到的最新 `starred_at`，然后再返回增长速度。

## 输出结构

返回：

- 当前 Star 总数
- 窗口内新增 Star 数
- 增长速度（每小时 Star 数和每天 Star 数）
- 窗口起始时间戳（UTC）
- 窗口内最新的 `starred_at`
- 可选：过去 24 小时的逐小时明细