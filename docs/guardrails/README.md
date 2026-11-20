# Guardrails：敏感信息保护

Guardrails 在每次 LLM 调用前拦截内容，按可配置规则检测、脱敏、阻断或审计敏感信息。

## 快速开始

```shell
# 生成含常用模式的初始配置
opensre guardrails init

# 用示例文本测试
opensre guardrails test "my key is AKIAIOSFODNN7EXAMPLE"

# 查看已配置的规则
opensre guardrails rules
```

## 工作原理

1. 首次 LLM 调用时从 `~/.opensre/guardrails.yml` 加载规则
2. 每次 LLM API 请求前，所有消息内容都会按规则扫描
3. 根据规则动作：
   - **redact**：匹配文本替换为 `[REDACTED:<rule_name>]`
   - **block**：请求被拒绝，抛出 `GuardrailBlockedError`
   - **audit**：记录匹配，文本原样通过
4. 所有匹配写入 `~/.opensre/guardrail_audit.jsonl`

若 `guardrails.yml` 不存在，所有内容原样通过，零开销。

## 配置

配置文件位于 `~/.opensre/guardrails.yml`。每条规则可使用正则模式、关键词列表，或两者兼用。

```yaml
rules:
  - name: aws_access_key
    description: "AWS access key IDs"
    action: redact
    patterns:
      - "(?:AKIA|ASIA)[A-Z0-9]{16}"

  - name: credit_card
    description: "Credit card numbers"
    action: block
    patterns:
      - "\\b\\d{4}[- ]?\\d{4}[- ]?\\d{4}[- ]?\\d{4}\\b"

  - name: internal_domains
    description: "Internal hostnames that should not leak"
    action: audit
    keywords:
      - "prod-db.internal.corp"
      - "staging.internal.corp"

  - name: pii_fields
    description: "Common PII field names"
    action: redact
    keywords:
      - "social_security"
      - "date_of_birth"
    replacement: "[PII_REDACTED]"
```

### 规则字段

| 字段 | 必填 | 说明 |
|-------|----------|-------------|
| `name` | 是 | 规则唯一标识 |
| `action` | 否 | `redact`、`block` 或 `audit`（默认：`audit`） |
| `patterns` | 否* | 正则模式列表（不区分大小写） |
| `keywords` | 否* | 字面关键词列表（不区分大小写） |
| `description` | 否 | 人类可读描述 |
| `replacement` | 否 | 自定义替换文本（默认：`[REDACTED:<name>]`） |
| `enabled` | 否 | 设为 `false` 可禁用而不删除（默认：`true`） |

*`patterns` 与 `keywords` 至少需提供一项。

## CLI 命令

### `opensre guardrails init`

创建初始 `~/.opensre/guardrails.yml`，含 AWS 密钥、信用卡、私钥和 API 令牌的常用模式。不覆盖已有配置。

### `opensre guardrails test "text"`

试运行：用所有规则扫描给定文本，显示哪些内容会被匹配、脱敏或阻断。

```
$ opensre guardrails test "key=AKIAIOSFODNN7EXAMPLE"
  [REDACT] aws_access_key: matched 'AKIAIOSFODNN7EXAMPLE'

  Redacted output: key=[REDACTED:aws_access_key]
```

### `opensre guardrails rules`

列出所有已配置规则及其动作和状态。

### `opensre guardrails audit`

显示 `~/.opensre/guardrail_audit.jsonl` 审计日志的最近条目。

## 健康检查

`opensre health` 显示当前 guardrails 状态：

```
CLI
  environment: development
  integration store: ~/.opensre/integrations.json
  guardrails: 5 rules active (~/.opensre/guardrails.yml)
```

## 覆盖范围

Guardrails 保护所有 LLM 调用路径：

- 自定义 Anthropic 客户端（`LLMClient.invoke`）
- OpenAI 兼容客户端（`OpenAILLMClient.invoke`）
- 结构化输出调用（委托给基础客户端）
- 交互式 shell 和网关聊天调用
- 告警提取提示
- 动作规划提示

## 常用模式

以下模式可添加到配置中：

```yaml
# Email addresses
- name: email
  action: redact
  patterns:
    - "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"

# IPv4 addresses
- name: ipv4
  action: audit
  patterns:
    - "\\b\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\b"

# GitHub personal access tokens
- name: github_pat
  action: redact
  patterns:
    - "ghp_[a-zA-Z0-9]{36}"
    - "github_pat_[a-zA-Z0-9]{22}_[a-zA-Z0-9]{59}"

# Slack webhook URLs
- name: slack_webhook
  action: redact
  patterns:
    - "https://hooks\\.slack\\.com/services/T[A-Z0-9]+/B[A-Z0-9]+/[a-zA-Z0-9]+"

# JWT tokens
- name: jwt
  action: redact
  patterns:
    - "eyJ[A-Za-z0-9_-]+\\.eyJ[A-Za-z0-9_-]+\\.[A-Za-z0-9_-]+"
```