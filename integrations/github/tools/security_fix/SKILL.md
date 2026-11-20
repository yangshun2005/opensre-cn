---
name: operating-github-security-fixer
description: 当用户要求 OpenSRE 修复 GitHub 安全与质量问题、Security and quality 页面发现项、Code Quality 标准发现项、Dependabot 告警、代码扫描或 CodeQL 告警、存在漏洞的依赖、仓库安全问题，或要求修复发现项并可选地开启 Pull Request 时使用。
tools:
  - fix_github_security_alert
---

# operating-github-security-fixer

GitHub 安全修复请求使用 `fix_github_security_alert`，不要用 `github_cli` 或 `shell_run`。

规则：

- 支持 Dependabot、代码扫描/CodeQL 告警和 Code Quality 标准发现项。
- 密钥扫描告警会被工具拒绝；告知用户先轮换或吊销密钥。
- 针对整个仓库的宽泛请求，省略 `alert_number` 并使用 `alert_type="auto"`，工具会按严重程度选择一个开放的受支持安全或质量发现项。
- 如果用户说"hey fix the security issues"但未指定仓库，省略 `owner` 和 `repo`；工具使用当前检出目录的 GitHub 源。
- 针对 GitHub `/security/code-scanning` 页面或 CodeQL/代码扫描积压请求，传入 `alert_type="code_scanning"`。
- 针对 GitHub `/security/quality` 页面或质量积压请求，传入 `alert_type="code_quality"`。
- 工具自行修复发现项：先使用内置修复器，再使用自动检测到的编码代理 CLI。除工具 `error` 文本已说明的内容外，不要添加编码代理建议、CLI 名称或安装命令。
- 如果工具返回 `response_text`，原样输出该文本并停止。
- 如果未生成自动补丁，回复仅限 `error` 中的一行简短说明；不要说"后续步骤"，不要添加编号选项、列出示例命令或提出宽泛的追问。
- 仅当用户要求开启、发起、创建或提交 Pull Request 时才设置 `open_pr=true`。
- 使用 `open_pr=true` 且指定了仓库时，省略 `workspace` 以使用隔离检出。仅本地修复使用已配置的工作区；显式指定的检出必须与目标仓库匹配。
- 工具每次调用处理一个告警。除非用户在首个结果后明确要求继续，否则不要循环处理多个告警。
- 根据 `summary`、`changed_files`、`branch_name` 和 `pr_url` 报告结果。