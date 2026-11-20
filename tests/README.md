# 测试

## 快速开始命令

| 目标 | 命令 | 使用场景 |
|---|---|---|
| 运行默认单元测试套件并生成覆盖率 | `make test-cov` | 本地首先运行；无需实时基础设施。 |
| 验证所有集成配置和客户端 | `make verify-integrations` | 添加或修改集成后。 |
| 运行默认 pytest 收集（pytest 配置排除 `tests/e2e`） | `make test-full` | 广泛的本地或 CI 回归运行。 |

## 目录结构

测试应放在领域目录下——不要在 `tests/` 根目录放散落文件。

| 路径 | 覆盖内容 |
|---|---|
| `tests/<domain>/` | 产品模块的单元测试和集成测试（`cli/`、`tools/`、`integrations/`、`core/`、`infrastructure/` 等）。 |
| `tests/e2e/` | 针对实时服务和基础设施的真实端到端场景。场景设计原则见 [e2e/AGENTS.md](e2e/AGENTS.md)。 |
| `tests/github_ci/` | 仓库卫生守卫（命名、导入边界、架构引用）。 |
| `tests/conftest.py` | 整个测试树的共享 pytest fixture。 |

## E2E 命名规则

- 目录格式：`tests/e2e/<scenario_name>/`，其中 `<scenario_name>` 描述系统和工作负载（例如：`install`、`quickstart`）。
- 环境特定的测试文件使用明确的文件名：
  - `test_local.py` 用于本地环境。
  - `test_<cloud>.py` 用于云环境。

## 遥测命名规则

- `OTEL_RESOURCE_ATTRIBUTES` 值必须使用语义目录名称，不得使用旧版 `test_case_*` 值。
- e2e 场景使用 `test_case=e2e_<scenario_name>`。

## 旧版名称

`tests/` 下旧版 `test_case_*` 路径命名已弃用。仅使用 `tests/e2e/*`。