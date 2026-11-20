# 运行基准测试

## 语料库位置

- **Hugging Face（上游）：** [tracer-cloud/cloud-ops-bench-dataset](https://huggingface.co/datasets/tracer-cloud/cloud-ops-bench-dataset)
- **AWS S3 镜像：** `s3://cloud-ops-bench-dataset/<hf-revision-sha>/` — 同一语料库，按修订版本固定，Fargate 基准任务启动时使用（比 HF 更快，无速率限制，运行时无需 `HF_TOKEN`）。由 `make mirror-cloudopsbench-s3` 填充。

```bash
# 一次性设置
make install
make download-cloudopsbench-hf      # 拉取 452 个场景的语料库
echo "ANTHROPIC_API_KEY=sk-..." >> .env   # 按需添加 OPENAI_API_KEY、DEEPSEEK_API_KEY

# 在 5 个场景上做冒烟运行（dev 模式跳过完整性检查，但仍调用真实 LLM）
uv run python -m tests.benchmarks._framework.cli run \
    tests/benchmarks/cloudopsbench/configs/cloudopsbench_smoke.yml --dev

# 产物输出到 .bench-results/example/<run-id>/：
#   report.json        ← 机器可读
#   report.md          ← 人类可读摘要
#   report.html        ← 自包含，可在任意浏览器中打开
#   provenance.json    ← 代码 SHA、配置内容、环境、模型版本
#   cases/*.json       ← 每个用例的原始产物
```

## 其他命令

```bash
uv run python -m tests.benchmarks._framework.cli list        # 显示可用的适配器
uv run python -m tests.benchmarks._framework.cli validate <config>   # 仅校验配置，不运行
uv run python -m tests.benchmarks._framework.cli report <run_dir>    # 从 report.json 重新渲染 md + html
```

## 生产运行（真实数据，非 dev 模式）

去掉 `--dev`。除非在配置指定的路径提交了预注册文件，否则框架会拒绝启动。完整指南见
[../../docs/cloudopsbench.mdx](../../docs/cloudopsbench.mdx)。

## 从 GitHub CI 运行

从 **Actions → "Benchmark — run on Fargate" → Run workflow** 触发。填写配置路径和 dev_mode 开关。工作流会启动一个 ECS 任务，并在不到一分钟内退出——实际基准测试在 Fargate 上运行。用 `aws logs tail /ecs/opensre-bench --follow` 查看实时日志，或在 AWS 控制台的 ECS → Clusters → opensre-bench → Tasks 下查看。任务完成后，结果会存入基准结果 S3 桶的 `runs/<date>-<sha>/` 下。

首次 CI 运行前的一次性设置：添加仓库密钥 `ANTHROPIC_API_KEY`、`OPENAI_API_KEY`、`DEEPSEEK_API_KEY`（仅添加配置所需的）。工作流位于
[../../.github/workflows/benchmark-run.yml](../../.github/workflows/benchmark-run.yml)。