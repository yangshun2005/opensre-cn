# 基准测试

本基准测试运行一组固定的合成场景：
- 001-replication-lag
- 002-connection-exhaustion
- 003-storage-full

报告的指标：
- 耗时
- Token 用量
- 预估 LLM 成本

不报告：
- 准确率
- 误报
- 漏报

## 运行基准测试

在仓库根目录执行：

```shell
make benchmark
```

该命令运行基准测试套件，**并**用汇总表更新 `README.md` 中的 `## Benchmark` 部分。完整报告写入 `docs/benchmarks/results.md`。

仅根据已生成的报告更新 README（不调用 LLM）：

```shell
make benchmark-update-readme
```

在基准测试运行期间跳过 README 更新：

```shell
python -m tests.benchmarks.toolcall_model_benchmark.benchmark_generator --no-update-readme
```

## README 自动更新机制

主 `README.md` 包含两个 HTML 注释标记：

```html
<!-- BENCHMARK-START -->
...summary content...
<!-- BENCHMARK-END -->
```

每次基准测试运行后，这两个标记之间的内容会被替换为最新的汇总表。替换是幂等的——多次运行基准测试会替换之前的结果，而不是追加重复内容。

这与其他 `README.md` 部分（例如贡献者区块）使用的标记分隔替换模式相同。

当 `main` 分支上的 `docs/benchmarks/results.md` 发生变化时，GitHub Actions 工作流（`.github/workflows/benchmark-readme.yml`）也会自动运行，无需人工干预即可保持 README 同步。

## 输出文件

- `docs/benchmarks/results.md` — 包含详细指标的完整逐用例报告
- `README.md`（基准测试部分）— 精简汇总表

## 自定义 README 路径

将汇总写入其他 README 文件：

```shell
python -m tests.benchmarks.toolcall_model_benchmark.benchmark_generator --readme-path /path/to/README.md
```

## 运行选定场景

```shell
python -m tests.benchmarks.toolcall_model_benchmark.benchmark_generator \
    --scenario 001-replication-lag \
    --scenario 002-connection-exhaustion
```