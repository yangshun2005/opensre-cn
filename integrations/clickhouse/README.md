# ClickHouse 调查读取器：开发者参考

源码检出还提供四个有界的 ClickHouse 证据操作。这些是用于构建调查器的读取器 API；上文的 watch 仍使用其原有的一次性综合。此 API 不启用额外的模型工具，也不发起模型调用或应用更改。

| 操作 | 返回的证据 |
| --- | --- |
| `compare_service_windows()` | 事故及明确允许的基线的请求/错误测量值；仅当两者均通过流量/新鲜度检查时才返回错误率差值。 |
| `summarize_failure_patterns()` | 基于已存储错误 span 的精确操作/消息分组、已存储行数/不同 trace ID 的计数，以及一个代表性 trace ID。 |
| `get_trace(trace_id)` | 在允许的服务/环境/窗口内为单个 trace 选定的 span，包括父链接、时序及选定的依赖/版本属性。 |
| `get_trace_logs(trace_id, span_id=...)` | 关联日志，可选过滤到单个 span。 |

从事故记录中一次性固定允许的窗口。以下是针对运行中演示的近期示例：

```python
import time
from dataclasses import asdict

from config.constants.telemetry import TELEMETRY_CLICKHOUSE_PASSWORD_ENV
from config.llm_credentials import resolve_env_credential
from core.telemetry.evidence_queries import InvestigationScope
from core.telemetry.models import Scope, WatchRule
from integrations.clickhouse.investigation_queries import ClickHouseInvestigationReader
from integrations.clickhouse.telemetry import ClickHouseEvidenceConfig

rule = WatchRule(service="payment", environment="demo")
incident = rule.scope(time.time())
baseline = Scope("payment", "demo", incident.start - 30, incident.start)
reader = ClickHouseInvestigationReader(
    ClickHouseEvidenceConfig(),
    resolve_env_credential(TELEMETRY_CLICKHOUSE_PASSWORD_ENV),
    InvestigationScope(incident, baseline, minimum_requests=5),
    rule.metric,
)
try:
    print(asdict(reader.compare_service_windows()))
    patterns = reader.summarize_failure_patterns(limit=5)
    if patterns.data:
        trace_id = patterns.data[0].example_trace_id
        print(asdict(reader.get_trace(trace_id, limit=10)))
        print(asdict(reader.get_trace_logs(trace_id, limit=10)))
finally:
    reader.close()
```

使用与 watch 相同的凭据设置。在首次证据操作前，会检查账户是否具有数据库强制只读访问权限。读取器是顺序的、短生命周期对象；完成后请关闭读取器。

必需输入为源配置、读取器凭据和事故范围。基线可选；未提供基线却请求基线证据会在访问源之前抛出 `ValueError`。若提供基线，它必须是同一服务/环境的更早、不重叠、等长的窗口。每个窗口最长五分钟；事故结束到基线开始最长一小时。基线是已授权的历史数据，并不表示其健康。

Trace/日志操作和模式摘要接受 `window="incident"`（默认）或 `window="baseline"`。`limit` 默认为十，接受 1–10。Trace ID 必须是非零的 32 字符十六进制 ID；可选 span ID 必须是非零的 16 字符十六进制 ID。服务、环境、时间戳、凭据和 SQL 不是操作参数，因此证据读取保持在固定范围内。窗口端点必须是整 Unix 秒；小数端点在访问任何源之前即被拒绝，因此证据查询和继承的测量查询均使用精确的允许边界。窗口使用 `(start, end]` 时间戳边界。Span 按开始时间选择。

每个类型化结果标识其来源、查询的窗口、观测时间、执行状态、截断和限制。单个模式/span/日志具有证据引用。`empty` 表示查询成功但无匹配证据；`unavailable` 表示访问/查询/解码失败，并包含异常类型，绝不包含异常消息或凭据。无效的范围、ID 和限制会在查询前抛出 `ValueError`。

所有读取均使用现有 SDK 的字节、行、执行时间和扫描限制。多取一行可检测行截断。超大的选定文本字段会整体替换为 `[OMITTED:oversize]`，并暴露截断；不返回可能包含部分凭据的前缀。其他选定文本在返回证据前会经过已知的凭据/PII 清理器；这仍是有限脱敏，而非全面 DLP。

模式计数描述已存储的 span 和 trace ID，包括可能的重复导出；它们不是权威的失败请求计数。Trace 结果始终是范围内的子集，即使 `truncated` 为 false。其他服务、窗口外开始的 span、采样/丢弃的遥测和延迟到达的数据可能缺失。无 trace 关联的日志无法通过这些操作恢复。缺失的依赖/版本属性保持为空。指标新鲜度检查无法排除窗口内的间隙；缺失/不足的测量会产生不合格的观测且无比率差值。无效测量产生不可用证据。恢复决策仍由 watch 负责。

在更改 SQL/Schema 契约之前，请针对一次性本地栈运行[实时查询验收测试](../../docs/guides/otel-payment-triage.mdx#live-evidence-query-acceptance-test)。