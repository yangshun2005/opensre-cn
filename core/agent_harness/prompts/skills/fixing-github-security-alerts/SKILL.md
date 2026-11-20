---
name: fixing-github-security-alerts
description: >-
  通过 fix_github_security_alert 修复 GitHub 安全 / Dependabot / CodeQL / 代码质量告警
metadata:
  owner: Vaibhav
  last_changed_by: Jan
  last_changed_at: 2026-09-12
  usecases:
  - 适用于修复 Dependabot、CodeQL 或代码质量告警的仓库维护者。
  - 适用于从具体安全告警或仓库安全页面请求修复的用户。
  requires:
  - 具有仓库写权限和安全事件读权限的 GitHub 认证。
  - 已安装并完成认证的编码代理。
  - fix_github_security_alert 工具及其支持的本地执行环境。
  version: '1.0'
---

# GitHub 安全与质量修复

使用 `fix_github_security_alert` 修复受支持的安全或质量问题，按需生成
本地差异或 Pull Request。

## 何时使用

- 用户要求修复/解决 GitHub 安全与质量问题、安全告警、安全与质量发现、
  代码质量发现、Dependabot 告警、CodeQL/代码扫描告警、存在漏洞的依赖，
  或仓库安全问题。
- 用户说"嘿，修复安全问题"、"修复 owner/repo 中的安全问题"、"修复安全
  与质量问题"，或"修复 opensre 仓库的安全问题并提一个 PR"。
- 提供了 GitHub 安全告警 URL、`/security/code-scanning` 页面 URL，或
  `/security/quality` URL。

## 范围

以下请求请使用其他工作流：

- 常规 GitHub Issue/PR 的创建、关闭、评论、指派、打标签、合并，或仓库
  读取。使用 `github_cli`。
- 密钥扫描修复。工具会拒绝，因为必须先在本仓库之外吊销/轮换密钥，
  再清理代码。

## 工作流规则

- 对于宽泛的仓库请求，调用：
  `fix_github_security_alert(owner?, repo?, alert_type="auto", open_pr=<用户是否要求 PR>)`
  由工具选择一个处于打开状态的受支持安全或质量发现。
- 未指定 owner/repo 时，两者都省略，由工具使用当前检出目录的 GitHub
  origin。
- 用户提供告警 URL、`/security/code-scanning` 页面 URL 或
  `/security/quality` URL 时，作为 `alert_url` 传入。
- 用户指明 Dependabot 告警、代码扫描告警、CodeQL 告警或代码质量发现
  编号时，传入 `alert_type` 和 `alert_number`。
- GitHub `/security/quality` 标准发现（如未使用的导入、空的 except、
  不可达代码、混合返回值）使用 `alert_type="code_quality"`。
- GitHub `/security/code-scanning` 页面或宽泛的 CodeQL/代码扫描请求使用
  `alert_type="code_scanning"`。
- 仅当用户要求打开/发起/创建 PR 或"交付"修复时，才设置
  `open_pr=true`；否则保持 false，生成本地差异。
- 绝不要用 `github_cli` 围绕此工具执行原始 `gh` 工作流。修复器负责告警
  上下文、修复执行、分支安全和 PR 创建。
- 工具每次调用只处理一个告警。除非用户明确要求在首个结果后继续，否则
  不要循环处理多个告警。
- 工具自行修复发现：先使用内置修复器，再使用自动检测的编码代理 CLI
  （无需配置）。除工具 `error` 文本已说明的内容外，绝不要添加编码代理
  建议、CLI 名称或安装命令，也绝不要让外部代理成为用户的下一步。
- 工具返回 `response_text` 时，原样输出该文本并停止。
- 工具报告无自动补丁时，用 `error` 中的一句话简短回复。不要说"下一步"，
  不要添加编号选项，不要列出示例命令，也不要提出宽泛的后续问题。
- 工具返回后，根据结果简短回复：发现类型/编号、变更文件，以及 PR URL
  （如有）。若设置了 `error_kind`，根据 `error` 说明所需的下一步。

## 示例

- "修复 Tracer-Cloud/opensre 中的安全问题并提一个 PR"

  ```text
  fix_github_security_alert(owner="Tracer-Cloud", repo="opensre", alert_type="auto", open_pr=true)
  ```

- "嘿，修复安全问题"

  ```text
  fix_github_security_alert(alert_type="auto")
  ```

- "修复 https://github.com/Tracer-Cloud/opensre/security/quality 上的代码质量发现"

  ```text
  fix_github_security_alert(alert_url="https://github.com/Tracer-Cloud/opensre/security/quality", alert_type="code_quality")
  ```

- "修复本仓库中的 Dependabot 告警 12"

  ```text
  fix_github_security_alert(alert_type="dependabot", alert_number=12)
  ```

- "修复 https://github.com/acme/app/security/code-scanning/7 并打开一个 PR"

  ```text
  fix_github_security_alert(alert_url="https://github.com/acme/app/security/code-scanning/7", open_pr=true)
  ```

- "修复 https://github.com/Tracer-Cloud/opensre/security/code-scanning 上的代码扫描错误并提一个 PR"

  ```text
  fix_github_security_alert(alert_url="https://github.com/Tracer-Cloud/opensre/security/code-scanning", alert_type="code_scanning", open_pr=true)
  ```