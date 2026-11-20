# LLM CLI 提供方（子进程）

当你要新增一个**非交互式** LLM、通过 shell 调用厂商 CLI（如 OpenAI Codex）而非 HTTP API 时，使用本包。

## 目录结构


| 文件                 | 作用                                                                                        |
| -------------------- | ------------------------------------------------------------------------------------------- |
| `base.py`            | `LLMCLIAdapter` 协议、`CLIProbe`、`CLIInvocation`。                                      |
| `constants.py`       | 共享的 runner/adapter 常量（探测缓存 TTL、临时失败重试参数、通用超时默认值）。 |
| `registry.py`        | `CLI_PROVIDER_REGISTRY`：将 `LLM_PROVIDER` 映射到 `adapter_factory` + 可选的 `model_env_key`。`opensre doctor` 使用 `get_cli_provider_registration()`，因此该注册表中的任何键都被视为 CLI 后端——不要在 `doctor.py` 中硬编码提供方 ID。 |
| `subprocess_env.py` | 传给 CLI 子进程的过滤后 `env`（`build_cli_subprocess_env`）。当某个 CLI 需要厂商特定的环境变量前缀时，在此扩展 `_SAFE_SUBPROCESS_ENV_PREFIXES`。 |
| `env_overrides.py` | 当 `build_cli_subprocess_env` 会丢弃某些显式 HTTP/API 密钥时，将其合并进 `CLIInvocation.env`（共享元组 + `nonempty_env_values`）。 |
| `timeout_utils.py`   | 共享的超时环境变量解析（`resolve_timeout_from_env`，用于默认值 + 钳制行为）。 |
| `probe_utils.py`     | 共享的子进程探测辅助函数（目前是 `run_version_probe`，用于 `<binary> --version`）。 |
| `semver_utils.py`    | 共享的 semver 辅助函数（`parse_semver_three_part`、`semver_to_tuple`）。 |
| `binary_resolver.py` | 共享的可执行文件解析辅助函数（`env -> PATH -> fallback paths`）。                     |
| `runner.py`          | `CLIBackedLLMClient`：护栏、`detect()`、`subprocess.run`、ANSI 剥离、`LLMResponse`。  |
| `agent_exec.py`      | 编码代理后端所用 *agentic*（“hands”）角色的共享机制：注入防护的任务提示、带管道排空的截止时间轮询子进程、结果分类（限制标记）。叶子模块——不从 `integrations/pi` 或 `integrations/coding_agent` 导入任何内容。 |
| `text.py`            | `flatten_messages_to_prompt`，用于从聊天式载荷生成 stdin。                            |
| `codex.py`           | 参考适配器：二进制解析、`codex exec`、`--version`，以及仅按需启用的 `login status` 探测。 |
| `opencode.py`        | 多提供方 CLI：`--version`，然后 `opencode auth list`（见 `_parse_opencode_auth_list_output`）。 |
| `kimi.py`            | `kimi --print` 路径：`--version`、`kimi login status`，然后回退到 env/config.toml（`KIMI_API_KEY`）。 |
| `copilot.py`         | 管道提示路径：`--version`，然后环境变量令牌，然后 `gh auth status`（当 `COPILOT_GH_HOST` / `GH_HOST` 指向非默认主机时还加 `--hostname`）；否则 `logged_in=None`。不检查明文 `$COPILOT_HOME/config.json`。不探测操作系统钥匙串。 |
| `pi_cli.py`          | Pi CLI（pi.dev，BYOK 多提供方）`pi -p` 打印模式：`--version`，然后基于状态的认证（环境变量中的提供方 API 密钥，否则读取 `/login` 写入的 `~/.pi/agent/auth.json`）——Pi 没有非交互式认证状态命令。提供方密钥通过 `CLIInvocation.env`（`PI_PROVIDER_ENV_KEYS`）转发，绝不使用通配前缀。 |
| `grok_cli.py`        | xAI Grok Build CLI（`grok -p --output-format plain`）：`--version`，然后 `grok models`（约 0.5 秒，不调用 LLM）以判定认证状态——在输出中查找 "You are logged in" / "logged in with"。即使探测结果不明确，`XAI_API_KEY` 环境变量也会作为无头/CI 回退将其提升为已认证。`XAI_API_KEY` 仅通过 `CLIInvocation.env`（`XAI_CLI_ENV_KEYS`）转发，绝不使用通配前缀。绝不传 `--always-approve`。 |


## 接入新提供方

**合并前**，请阅读下方的 **[子进程环境变量允许列表](#subprocess-environment-allowlist)**：如果你的 CLI 读取厂商特定的环境变量，你必须在 `subprocess_env.py` 中扩展 `_SAFE_SUBPROCESS_ENV_PREFIXES`，否则子进程看不到这些变量（认证和配置会静默失败）。

实现 `LLMCLIAdapter`（`detect()` 不得抛异常；`build()` 返回 argv + 可选 stdin；`parse` / `explain_failure` 处理成功和非零退出——runner 不会基于单独的“投递模式”分支，`CLIInvocation` 携带 `build()` 产生的内容），然后按本文件底部的 **[提供方检查清单](#provider-checklist-copypaste)** 接入注册表、配置、向导和测试。

## 二进制解析（推荐模式）

使用 `binary_resolver.resolve_cli_binary(...)`，让所有适配器共享相同行为。

解析顺序：

1. 显式二进制环境变量（`<PROVIDER>_BIN`，如 Codex 的 `CODEX_BIN`）**仅当其指向可运行文件时**。
2. 对平台特定二进制名执行 `shutil.which(...)` 查找。
3. 来自 `default_cli_fallback_paths(...)` 的回退安装位置。

说明：

- 二进制环境变量默认可选。
- 空白/无效的显式路径会被忽略；PATH/回退解析仍会运行。
- 对 Codex，保持此行为：用户可以在没有 `CODEX_BIN` 的情况下运行。

## 约定

- **无 TTY**：调用必须适合在没有交互式会话的情况下通过 `subprocess.run` 执行。
- **探测 vs 运行**：`detect()` 开销低；`CLIBackedLLMClient.invoke` 在执行前会再次探测，使缺失认证快速失败并给出清晰错误。
- **结构化输出**：`CLIBackedLLMClient.with_structured_output` 委托给 `StructuredOutputClient`（JSON-in-prompt），与 API 客户端模式相同。
- **可选模型环境变量**：使用 `<PROVIDER>_MODEL`（见 [各提供方环境变量](#per-provider-env-vars-required-for-every-new-cli)）；始终可选——未设置时依赖厂商 CLI 默认值。

## 各提供方环境变量（每个新 CLI 都必须提供）

**Codex 是参考。** 每个子进程 LLM 都必须暴露相同*形态*的配置项：

| 环境变量 | 作用 |
| ------- | ---- |
| `<PROVIDER>_BIN` | 可选的厂商可执行文件显式路径。将相同名称作为 `explicit_env_key` 传给 `resolve_cli_binary(...)`。缺失、空白或无效路径会被忽略；PATH + 回退仍会运行。 |
| `<PROVIDER>_MODEL` | 可选的模型覆盖。在 `registry.py` 的 `CLIProviderRegistration` 上注册为 `model_env_key`。为空或未设置 → runner 省略该标志，CLI 使用其默认值。 |

**命名：** 从注册表 / `LLM_PROVIDER` 字符串派生 `<PROVIDER>`：**大写**，然后加 `_BIN` / `_MODEL`。示例：`codex` → `CODEX_BIN`、`CODEX_MODEL`；未来的 `gemini` → `GEMINI_BIN`、`GEMINI_MODEL`。

在适配器模块 docstring 或 `binary_env_key` / 注册条目附近的一行注释中记录这两个变量，以便用户和向导文案保持一致。

## 认证探测模式

`detect()` 必须返回带有 `logged_in: bool | None` 的 `CLIProbe`——三种状态：

| 值 | 含义 | 向导行为 |
| ----- | ------- | ---------------- |
| `True` | 找到二进制**且**认证已确认。 | 立即继续。 |
| `False` | 找到二进制但明确**未**认证。 | 提示用户运行登录命令（`auth_hint`）。 |
| `None` | 找到二进制但认证**状态不明确**（网络错误、意外输出等）。 | 要求用户重试或重新选择提供方。 |

对具有安全的非交互式认证状态命令的 CLI，推荐的探测顺序：

1. 运行 `<binary> --version`——如果失败，立即返回 `installed=False`。
2. 运行 `<binary> <auth-status-command>`——解析 stdout/stderr 以判定 `logged_in`。
3. 编写 `_classify_<name>_auth(returncode, stdout, stderr) -> tuple[bool | None, str]`
   辅助函数。**先检查否定短语**（例如 `"not logged in"` 先于 `"logged in"`），
   以避免子串误报。
4. 将网络/超时错误映射为 `None`，而非 `False`——用户可能处于不稳定的
   连接，不应被迫重新认证。

完整参考实现见 `codex.py` 中的 `_classify_codex_auth`。

**Codex 例外：** 默认不要运行 `codex login status`。某些 Codex CLI
版本在检查会话时可能打开浏览器 OAuth，因此 Codex 适配器
仅在显式设置 `OPENSRE_CODEX_AUTH_STATUS_PROBE=1` 时才运行该命令。
正常的提示安全状态路径报告 `logged_in=None`，并让
`codex exec` 在请求时暴露认证失败。

**OpenCode** 是多提供方：用户可能依赖 `auth.json`、环境 API 密钥，或两者兼有。
在 `--version` 之后运行 `opencode auth list`，并解析报告的凭据/环境
计数，使检测与 CLI 一致（不要仅从 JSON 文件推断认证）。

**Kimi** 在 `--version` 之后使用 `kimi login status`，当 CLI 未明确确认认证时
（`logged_in` 不为 `True`）使用 `_check_kimi_auth_fallback()`：检查 `KIMI_API_KEY`，然后检查
`~/.kimi/config.toml`（或 `KIMI_SHARE_DIR`）中的 API 密钥。仅使用 API 密钥的安装可能在
`login status` 中省略“logged in”措辞，因此回退逻辑反映真实用法。当 `login status` 超时
或无法启动（`logged_in=None`）时也运行相同回退，因此已配置的 API 密钥仍算作已认证。

## 子进程环境变量允许列表

`CLIBackedLLMClient` 仅通过 `subprocess_env.py` 中的
`build_cli_subprocess_env`（`_SAFE_SUBPROCESS_ENV_KEYS` +
`_SAFE_SUBPROCESS_ENV_PREFIXES`）向子进程传递安全子集的环境变量。

共享的 HTTP/API 覆盖位于 `env_overrides.py`：将 `nonempty_env_values(...)` 与
`OPENAI_PLATFORM_ENV_KEYS`（Codex）、`HTTP_LLM_PROVIDER_ENV_KEYS`（OpenCode）、
`ANTHROPIC_CLI_ENV_KEYS`（Claude Code）、`CURSOR_CLI_ENV_KEYS`（Cursor Agent 无头 API 密钥）
或 `COPILOT_CLI_ENV_KEYS`（Copilot CLI 凭据环境变量：`COPILOT_GITHUB_TOKEN`、`GH_TOKEN`、`GITHUB_TOKEN`）以及 `COPILOT_CLI_CONFIG_ENV_KEYS`（`COPILOT_HOME`、`COPILOT_MODEL`、`COPILOT_GH_HOST`、`GH_HOST`）一起使用。
当你向 `LLMSettings` 添加匹配的 API 密钥环境变量时，扩展这些元组。

**Kimi** 目前不使用这些元组：OAuth/API 材料通过 `_SAFE_SUBPROCESS_ENV_PREFIXES` 转发任何 `KIMI_*` 键来覆盖；`KimiAdapter.build()` 使用 `CLIInvocation(env=None)` 并依赖该允许列表。

当前前缀允许列表包括 `CODEX_`、`CURSOR_`、`CLAUDE_`、`OPENCODE_`、`KIMI_` 和区域设置键（`LC_`）。`COPILOT_` 故意不是前缀条目——`COPILOT_GITHUB_TOKEN` 是 GitHub PAT，通配前缀会将其泄漏到所有其他 CLI 子进程；Copilot 适配器改为通过自己的 `CLIInvocation.env` 转发每个 Copilot 作用域的环境变量（见 `env_overrides.py` 中的 `COPILOT_CLI_ENV_KEYS` / `COPILOT_CLI_CONFIG_ENV_KEYS`）。

**如果你的 CLI 读取自定义环境变量**（例如 `GEMINI_*`），你必须在
`subprocess_env.py` 的 `_SAFE_SUBPROCESS_ENV_PREFIXES` 中添加相关前缀，否则
子进程将收不到这些变量，认证或配置会静默
失败。添加一个断言所需键已转发的测试。

## Codex 二进制解析（参考）

`CodexAdapter._resolve_binary` 中的顺序（现已委托给共享解析器辅助函数）：

1. 如果设置了 `CODEX_BIN` 且路径可运行（显式覆盖）。
2. `shutil.which("codex")`（以及 Windows 的 `codex.cmd` / `codex.ps1`）。
3. `_fallback_codex_paths()`——常规安装位置；无效或空白的 `CODEX_BIN` 会被忽略，因此 PATH/回退仍适用。

## Codex 环境变量快速参考（上述约定的实例）

全部可选：

```bash
CODEX_MODEL=
CODEX_BIN=
```

- 如果未设置 `CODEX_MODEL`，`codex exec` 使用其默认模型行为。
- 如果未设置 `CODEX_BIN`，适配器解析回退到 PATH + 已知安装位置。

## 提供方检查清单（复制/粘贴）

- 在 `integrations/llm_cli/` 中添加适配器。
- 按 [各提供方环境变量](#per-provider-env-vars-required-for-every-new-cli) 定义 `<PROVIDER>_BIN` + `<PROVIDER>_MODEL`；为 `_resolve_binary` 复用 `resolve_cli_binary(..., explicit_env_key=...)`。
- 实现 `detect()`，包含 `--version` + 认证状态检查；遵循上述三态 `logged_in` 模式。
- 编写 `_classify_<name>_auth`——合并前针对真实已登录**和**已登出会话进行测试。
- 如果 CLI 读取自定义环境变量（例如 `GEMINI_*`），在 `subprocess_env.py` 的 `_SAFE_SUBPROCESS_ENV_PREFIXES` 中添加前缀。
- 在 `registry.py` 中注册提供方，并在 `config/llm_settings.py` 中添加相同的 `LLM_PROVIDER` 值。
- （可选）在 `surfaces/shared/llm_setup/catalog.py` 中添加向导引导选项。
- 在 `tests/integrations/llm_cli/` 下为 detect/build/失败路径添加测试，包括环境变量转发。

## 测试

- `tests/integrations/llm_cli/`——适配器和 runner 单元测试；按需 mock `subprocess` / `shutil.which`。
- 平台特定断言必须 patch `integrations.llm_cli.binary_resolver.sys.platform`（而非 `codex.sys.platform`），因为解析逻辑位于 `binary_resolver.py`。
- `npm_prefix_bin_dirs` 带有 `@lru_cache`；改变环境变量或平台的测试应在每个用例前调用 `npm_prefix_bin_dirs.cache_clear()`（或使用共享的 autouse fixture），以避免测试间缓存陈旧。