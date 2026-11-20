# `core/` 命名规范

一套小而可执行的词汇表，让文件名和类型名说明其自身含义。目标是读者仅凭名称就能区分数据类型与流程、可变状态与冻结视图、包的用途。

## 术语表（每个术语一个含义）

| 术语 | 含义 | 示例 |
| --- | --- | --- |
| **State** | 运行期间演进的可变 agent/session 事实 | `AgentState`、harness 端口 `SessionState` |
| **Storage** / **Repo** | 持久化后端（非内存中的 turn 端口） | `SessionStore`、`SessionRepo`、`JsonlSessionStore` |
| **Snapshot** | 在边界处（turn 开始、run 开始）捕获的冻结视图 | `TurnSnapshot` |
| **RunInput** / **RunResult** | 一个 `Agent.run()` 边界的输入与输出 | `AgentRunInput`、`AgentRunResult` |
| **Resources** | 传入工具执行器、供单次调用使用的句柄 | `ToolCallResources` |
| **Budget** | LLM token/窗口策略——不是应用状态 | `enforce_token_budget` |
| **Host** | 算法驱动的回调契约（一个 `Protocol`） | `LoopHost` |

## 模块命名：`{domain}_{role}.py`

按文件所承载的概念命名，不用泛化的桶式词汇。

```
core/agent/
  agent.py          # Agent 门面
  react_loop.py     # ReactLoop + run_react_loop（算法）
  loop_host.py      # LoopHost（回调契约）
  run_io.py         # AgentRunInput、AgentRunResult（run 边界的 I/O）
  mixins.py         # 可复用的 *Mixin 行为
  provider_hooks.py # ProviderHookDelegate
```

## 类型命名

- **Mixin** 带 `Mixin` 后缀——它们不能独立存在（依赖宿主提供的字段/方法）。`EventEmitterMixin`、`ToolFilterMixin`、`SteeringMixin`。
- **Protocol** 按角色命名，不加 `Protocol` 后缀——与标准库（`Iterable`、`SupportsRead`）和 `agent_harness/ports.py`（`OutputSink`、`SessionState`）一致。是 `LoopHost`，不是 `LoopHostProtocol`。
- **不要用类型自身所在的包名作前缀。** 在 `core/agent/` 内，类名是 `EventEmitterMixin`，不是 `AgentEventEmitter`——命名空间已经表明"agent"。

## 反模式（新代码中不要添加）

- 在 `core/` 或 `core/agent/` 根目录放 `context.py`——"context"在仓库中含义过载。按概念命名（`run_io.py`、`turn_snapshot.py`）。
- 文件只承载 run I/O 时用 `models.py`——太模糊。说明模型是什么（`run_io.py`）。
- 已存在另一个 `*Context` 时，新增不带域前缀的 `*Context`。
- 包的唯一子项是单个子包——折叠掉这层包装。
- 把可变的 harness session 端口叫作 `SessionStore`——该词指持久化。端口是 `SessionState`（无头场景为 `InMemorySessionState`）；JSONL 持久化仍叫 `SessionStore` / `SessionRepo`。
- 用模块级可变全局变量表示"当前 session"——显式传入 `SessionState`（见本地笔记：无全局变量设计）。

## 导入

代码中使用完全限定路径；文档中保留简短的心智标签。

| 心智标签 | 导入 |
| --- | --- |
| ReAct run I/O | `from core.agent.run_io import AgentRunInput, AgentRunResult` |
| ReAct 循环 | `from core.agent.react_loop import run_react_loop` |
| 循环回调契约 | `from core.agent.loop_host import LoopHost` |
| agent 原语 | `from core.agent import Agent` |
| Harness turn 快照 | `from core.agent_harness.turns.turn_snapshot import TurnSnapshot` |

仅在包的 `__init__.py` 中重新导出其唯一的规范符号（`Agent`），不要导出全部——避免 `from core.agent import *` 式的歧义。