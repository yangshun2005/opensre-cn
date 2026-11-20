# 贡献者须知

修改测试替身、LLM 请求或修复 CodeQL 发现项时，请阅读相关章节。本须知补充 [AGENTS.md](AGENTS.md)；验证与评审的后续流程仍由 [CI.md](CI.md) 规定。

## 测试替身与类型标注

不要在 `monkeypatch.setattr` / `patch` 中内联构造临时 `type(...)` 对象的 lambda，或嵌套另一个 lambda。应提取为具名辅助函数：

```python
def _build_harness() -> Any:
    return type("H", (), {"resolve_env_variables": lambda _self: None})()


monkeypatch.setattr(startup, "_build_harness", _build_harness)
```

`lambda **_kw: None` 或 `lambda: sentinel` 这类简单 lambda 可以内联。

`make typecheck` 只覆盖产品包，不覆盖 `tests/`。它不能证明新测试类型干净；必要时请直接对测试路径运行 mypy。

## LLM 请求与工具 Schema

- 工具 Schema 必须在提供方适配器中规范化，并由注册表契约测试覆盖。宽松的 JSON Schema 校验可能放过在 LLM API 处失败的 Schema，因为所有可用工具会一起发送。
- LLM 客户端按角色缓存，并服务并发的网关轮次。构建请求时捕获请求局部事实，重试时依据这些事实判断；另一个进行中的请求可能已改变共享标志。
- 例如，提示缓存回退必须检查失败请求是否携带缓存标记，而不是检查客户端是否仍启用标记。可通过让假依赖在抛错前修改共享状态来确定性复现该竞态；无需使用线程。

## CodeQL 与代码质量

处理告警时保持行为不变。不要用空操作替代有用的工作，也不要压制发现项而不修复其根因。

| 发现项 | 安全修复方式 |
| --- | --- |
| `py/cyclic-import` | 将共享符号移到叶子模块。函数内导入和 `TYPE_CHECKING` 导入仍算作循环边。 |
| `py/uninitialized-local-variable` / 不可达代码 | CodeQL 不建模 `pytest.skip`、`pytest.fail`、`sys.exit` 或 `typer.Exit` 等 `NoReturn` 辅助函数。在每条可见路径上绑定值；优先使用 `next(iterable, None)` 加显式守卫，而非 `StopIteration` 控制流。针对裸 `raise` 的可达性告警，提取 `_raise()` 辅助函数。 |
| Protocol 上的 `py/ineffectual-statement` | 遵循根目录的仅文档字符串规则。干净的扫描或 mypy 结果不能证明合规：`pass` 和 `raise NotImplementedError` 函数体不会被捕获。 |
| `await` 上的 `py/ineffectual-statement` | 保留 await：回收已取消的任务可能正是关闭其客户端的方式。在小辅助函数中绑定被 await 的结果，而不是删除 await。 |
| `py/implicit-string-concatenation-in-list` | 将长文本提取为常量，并在列表/元组中引用。不要用显式 `+` 消除告警；括号赋值中的隐式拼接没问题。共享常量仍遵循根目录归属规则。 |
| `py/import-and-import-from` | 遵循文件现有导入风格，包括函数内部；不要对同一模块混用模块导入和 `from` 导入。 |
| 捕获 `BaseException` | 普通失败捕获 `Exception`；不要吞掉 `KeyboardInterrupt` / `SystemExit`，也不要保留 `noqa: BLE001` 来掩盖问题。 |
| 未使用的全局变量 | 跨模块导入可能不被计为使用。将相关副本保留在真正使用的结构中，或将私有常量与其唯一读取者放在一起。不要添加空操作自引用或 `# noqa`；共享常量仍应放在 `config/constants/` 下。 |
| `py/stack-trace-exposure` | 在接收端/响应边界应用根目录的外部错误规则。在服务端记录详情；绝不向 HTTP 或聊天网关用户发送异常文本、参数、回溯或提供方/模型/字段内部信息。 |

## 人工负责的提示词变更

人工应通过 Web 编辑器每次提交只编辑一张技能卡或一个提示词章节，并自行审查生成的 diff。Agent 可以提出确切的当前/替换文本，但必须保持受保护文件不变，[AGENTS.md](AGENTS.md#human-owned-files-mandatory) 中唯一的字面替换例外除外。