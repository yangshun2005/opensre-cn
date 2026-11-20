# Grafana Cloud 验证

## validate_grafana_cloud.py

通过日志、指标和追踪检查 `prefect-etl-pipeline` 的 Grafana Cloud 数据摄取。

必需的环境变量：
- GCLOUD_HOSTED_METRICS_URL
- GCLOUD_HOSTED_METRICS_ID
- GCLOUD_HOSTED_LOGS_URL
- GCLOUD_HOSTED_LOGS_ID
- GCLOUD_HOSTED_TRACES_URL_TEMPO
- GCLOUD_HOSTED_TRACES_ID
- GCLOUD_RW_API_KEY

运行：
```
python3 tests/e2e/grafana_validation/validate_grafana_cloud.py
```

## grafana_otlp_env_check.py

确认 OTLP 环境变量已设置，并打印本地运行 Prefect 的后续步骤。

必需的环境变量：
- GCLOUD_OTLP_ENDPOINT
- GCLOUD_OTLP_AUTH_HEADER

运行：
```
python3 tests/e2e/grafana_validation/grafana_otlp_env_check.py
```

## Pytest 查询冒烟测试

这些测试验证 Grafana Cloud 查询端点能否正常响应，无需特定流水线的遥测数据。

前提条件：
- `.env` 或环境中存在 `GRAFANA_READ_TOKEN`
- 可选：`GRAFANA_INSTANCE_URL`（默认为 `https://tracerbio.grafana.net`）

注意：配置缺失时这些测试会跳过；配置的实时 Grafana 凭据被拒绝（如 `401` 或 `403` 认证错误）时也会跳过。

运行：
```
python3 -m pytest tests/e2e/grafana_validation/test_grafana_cloud_queries.py -v
```