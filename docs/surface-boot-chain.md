# 各界面如何启动

OpenSRE 有多种访问方式——`opensre` CLI、交互式 shell、网关守护进程（聊天频道）、网关的 Web 应用、定时 `cron` 命令，甚至是在普通 Python 脚本中嵌入 agent。它们启动时都运行同一套共享设置——`configure_process(<profile>)`，按顺序执行 [`infrastructure/process/boot.py`](https://github.com/Tracer-Cloud/opensre/blob/main/infrastructure/process/boot.py) 中的步骤——而不是各自拼装启动流程。界面自身的代码只负责其通道和用户体验（输出到终端、在聊天中回复、提供 HTTP 路由、解析 CLI 参数）以及该界面特有的前置条件；部分界面会在共享步骤前后做一些自己的设置（见下文"各界面在何处调用"）。

## 共享设置做什么

启动是一份简短的设置清单，每次都必须按相同顺序执行：

1. **加载环境** —— 读取配置和密钥。
2. **启动错误上报** —— 这样当所选 profile 负责 Sentry 时，启动后期的失败仍能被捕获。
3. **注册适配器** —— 将 agent 连接到可用的集成和工具。
4. **注册定时投递适配器** —— 为定时任务绑定出站投递。Agent 运行器由调度器宿主在启动时构造，而非全局存储。
5. **记录能力警告** —— 标记沙箱在当前环境中无法完成的事项。
6. **预加载 LLM 客户端模块** —— 避免长时间运行的进程在后续代码变更后混用这些模块的新旧版本。
7. **拉取技能** —— 在后台刷新技能目录。拉取的发布版本在后续轮次生效，绝不在轮次中途生效。

并非每个界面都需要每一步。Profile 只是"某界面所需清单子集"的名称——CLI 需要的远少于网关守护进程。用某个 profile 运行共享设置，就会按上述顺序精确执行这些步骤；对已启动的界面再次运行则是无害的空操作。

## 六个 profile

| Profile | 设置内容 | 使用者 |
| --- | --- | --- |
| CLI | 环境、技能拉取。 | `opensre` 命令 |
| Gateway | 环境、错误上报、适配器、能力警告、LLM 预加载、技能拉取。 | 网关守护进程（聊天频道） |
| Web | 环境、错误上报、适配器。 | 网关的独立 Web 应用 |
| Scheduler worker | 环境、错误上报、适配器、定时投递适配器、技能拉取。 | `opensre cron start` 守护进程 |
| Scheduled command | 环境、适配器、定时投递适配器。 | 创建、运行或分发定时任务的一次性 CLI 命令 |
| Embedded | 环境、适配器。 | 从其他 Python 程序驱动 agent |

## 各界面在何处调用

- **CLI** —— [`surfaces/cli/startup.py`](https://github.com/Tracer-Cloud/opensre/blob/main/surfaces/cli/startup.py) 先运行 CLI profile，再处理 CLI 专属设置：自己的错误上报（在 `opensre update` 期间容忍依赖缺失）、终端输出样式和键盘中断处理。[`main.py`](https://github.com/Tracer-Cloud/opensre/blob/main/main.py) 是模块级入口（`python main.py`），直接委托到同一条 CLI 路径——它不使用 embedded profile。`python -m surfaces` 是同一入口。（embedded profile 只出现在 `main.py` 的文档字符串中，作为给在自己脚本中嵌入 agent 的人的示例。）交互式 shell 运行在同一个已启动的 CLI 进程内，因此不会再次启动。
- **交互式 shell 的已保存循环** —— 当已保存的提示循环需要自己的后台调度器时，[`surfaces/interactive_shell/runtime/loop_scheduler.py`](https://github.com/Tracer-Cloud/opensre/blob/main/surfaces/interactive_shell/runtime/loop_scheduler.py) 运行 scheduled-command profile。[`loops_cmds.py`](https://github.com/Tracer-Cloud/opensre/blob/main/surfaces/interactive_shell/command_registry/loops_cmds.py) 中的斜杠命令通过该模块重新加载；它们自身不会启动第二个 profile。
- **网关守护进程** —— [`gateway/core/lifecycle/controller.py`](https://github.com/Tracer-Cloud/opensre/blob/main/gateway/core/lifecycle/controller.py) 先设置自己的日志、就绪状态和凭据，再在连接聊天频道前运行 gateway profile。当它同时托管调度器时，控制器会在该后续生命周期阶段安装定时投递适配器并构造受门控的调度器运行器。
- **网关 Web 应用** —— [`gateway/web/webapp.py`](https://github.com/Tracer-Cloud/opensre/blob/main/gateway/web/webapp.py) 在模块加载时立即运行 web profile，使进程内网关和独立 Web 服务器在处理请求前都具备所需的一切。
- **定时/cron CLI 命令** —— [`surfaces/cli/commands/cron.py`](https://github.com/Tracer-Cloud/opensre/blob/main/surfaces/cli/commands/cron.py) 中的 `cron run` 等一次性命令以及 [`sentry_digest.py`](https://github.com/Tracer-Cloud/opensre/blob/main/surfaces/cli/commands/sentry_digest.py) 运行 scheduled-command profile。[`posthog_report.py`](https://github.com/Tracer-Cloud/opensre/blob/main/surfaces/cli/commands/posthog_report.py) 先检查 PostHog 集成是否已配置，再运行同一 profile。长时间运行的 `cron start` 守护进程改用 scheduler-worker profile，因为它需要自己的错误上报，且只需注册一次任务运行器，而非每次运行都注册。
- **嵌入式 Python 用法** —— [`surfaces/embedded.py`](https://github.com/Tracer-Cloud/opensre/blob/main/surfaces/embedded.py) 的 `start_embedded_session()` 运行 embedded profile，然后调用 `AgentSession.start`。从其他 Python 程序调用时应使用它。它等同于 `configure_process(EMBEDDED_PROFILE)` 后接 `start`；`start` 本身不注册适配器，因为 `core` 可能无法导入其上的宿主。

## 为什么这样设计

保持一份共享启动清单，意味着无论以何种方式启动，每个界面都获得相同的保证——不跳过任何步骤，不出现顺序错乱。这也意味着新增或修改启动步骤只需在一处进行。关于它如何融入 OpenSRE 的整体包布局，见 [`docs/ARCHITECTURE.md`](ARCHITECTURE.md)。