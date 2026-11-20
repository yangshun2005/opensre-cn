# 适合新手的 Issue

刚接触 OpenSRE？本页是你的起点。

## 什么是"适合新手的 Issue"？

适合新手的 Issue 是指满足以下条件的任务：

- **自包含** — 无需理解整个代码库即可解决
- **范围明确** — 预期产出有清晰定义
- **低风险** — 出错不会破坏关键路径

这类 Issue 的设计目标是：让你在熟悉项目的同时做出真实贡献。

## 查找开放的 Issue

浏览带有 `good first issue` 标签的 Issue：

[在 GitHub 上查看适合新手的 Issue](https://github.com/Tracer-Cloud/opensre/issues?q=is%3Aopen+label%3A%22good+first+issue%22)

## 如何挑选并着手处理

1. **浏览列表** — 认领前先阅读 Issue 描述和评论
2. **评论认领** — 发表评论如 `"I'd like to work on this"`，以便维护者将其分配给你
3. **阅读环境搭建指南** — 先让环境跑起来：[SETUP.md](../../SETUP.md)
4. **Fork 并创建分支** — `git checkout -b issue/123-short-description`
5. **进行修改** — 保持范围紧凑；一个 Issue 对应一个 PR
6. **提交 PR 前运行本地检查**：
```bash
   make lint && make format-check && make typecheck && make test-cov
```
7. **发起 Pull Request** — 在 PR 描述中用 `Fixes #123` 关联该 Issue

完整贡献流程见 [CONTRIBUTING.md](../../CONTRIBUTING.md)。
发起 PR 后，请遵循 [PR 审查流程](../guides/pr-review-flow.mdx)
（Greptile · 人工审查 · CI）。

## 寻求帮助

遇到困难？不要猜测 — 尽早提问。

- **Discord：** [#contribute](https://discord.gg/opensre)
- **GitHub：** 直接在 Issue 下评论

## 几点提示

- 开始前先通读 `CONTRIBUTING.md` — 它能提前解答大部分问题
- 每个 PR 只解决一个问题；不要捆绑无关的修复
- 如果 Issue 描述不清，先请求澄清再写代码
- 可以使用 AI 辅助写代码，但你必须理解并能解释每一行（见 [AI 辅助 PR](../../CONTRIBUTING.md#ai-assisted-prs)）