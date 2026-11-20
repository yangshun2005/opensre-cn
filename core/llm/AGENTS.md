# 托管 LLM 运行时

本包负责托管 LLM 提供商客户端，以及 agent 循环使用的运行时辅助工具。基于子进程的 LLM CLI 位于 `integrations/llm_cli/` 下。

## 提供商接线所在位置

| 文件 | 职责 |
| --- | --- |
| `config/llm_models.py` | 各提供商规范的模型层级默认值、基础 URL 和 `*_LLM_CONFIG` 结构。 |
| `config/llm_settings.py` | 声明 `LLMProvider`、`LLMSettings` 校验以及基于环境变量的解析。 |
| `config/llm_auth/provider_catalog.py` | 向导、认证和运行时检查共享的规范 `ProviderSpec` 元数据。 |
| `core/llm/factory.py` | 唯一路由入口：`resolve_llm_route()`、`get_llm(role)`、`reset_llm_clients()`。 |
| `core/llm/readiness.py` | `llm_ready()` 预检：路由（账户，否则为已配置的提供商）是否具备其设置和密钥，不构建客户端。 |
| `core/llm/client_builders.py` | 为已解析的路由构建客户端：`build_agent_client()`、`build_reasoning_client()`。 |
| `core/llm/providers/provider_registry.py` | 构建器读取的 `FIRST_PARTY_PROVIDERS` 表（模型、max_tokens、LiteLLM 前缀、api-key 环境变量）。 |
| `core/llm/transport_mode.py` | `OPENSRE_LLM_TRANSPORT`（`sdk` 与 `litellm`）和 `use_litellm_for_provider()`。 |
| `core/llm/internal/client_cache_key.py` | 单例缓存失效键 `(transport, runtime_provider)`。 |
| `core/llm/providers/openai_compat_providers.py` | OpenAI 兼容提供商目录及模型/基础 URL 解析。 |
| `core/llm/providers/azure_openai.py` | Azure OpenAI 辅助工具：端点规范化、部署选择、LiteLLM kwargs。 |
| `core/llm/providers/vertex_ai.py` | Vertex AI 辅助工具：项目/位置解析、LiteLLM kwargs（环境 ADC 认证，无 API 密钥）。 |
| `core/llm/transports/litellm/routing.py` | 各提供商的 LiteLLM 客户端构建（模型前缀、`api_base`、`api_version`）。 |
| `core/llm/transports/litellm/clients.py` | 围绕 `litellm.completion` 的 `LiteLLMAgentClient` / `LiteLLMLLMClient` 封装。 |
| `core/llm/transports/sdk/agent_clients.py` | 原生 SDK 工具调用客户端（Anthropic、OpenAI、Bedrock、CLI 支持）。 |
| `core/llm/transports/sdk/llm_clients.py` | 原生 SDK 非 agent 客户端。 |
| `core/llm/shared/tool_schema_normalize.py` | 严格工具调用适配器共享的 JSON Schema 规范化。 |
| `surfaces/shared/llm_setup/catalog.py` | 引导元数据（`SUPPORTED_PROVIDERS`）和模型选项。 |
| `surfaces/shared/llm_setup/env_sync.py` | 提供商/模型选择变更时的 `.env` 同步。 |

面向用户的设置和环境变量表：[`docs/platform/llm-providers.mdx`](../../docs/platform/llm-providers.mdx)。

## 传输：原生 SDK 与 LiteLLM

默认路径是**原生厂商 SDK**（`OPENSRE_LLM_TRANSPORT` 未设置或为 `sdk`）。

**LiteLLM 路径**（`OPENSRE_LLM_TRANSPORT=litellm`）：通过 `core/llm/transports/litellm/routing.py` 而非 `core/llm/transports/sdk/*` 路由托管 API 提供商。

**Azure OpenAI**（`LLM_PROVIDER=azure-openai`）**始终**使用 LiteLLM——即使 `OPENSRE_LLM_TRANSPORT` 未设置。引导流程会向 `.env` 写入 `OPENSRE_LLM_TRANSPORT=litellm`；从 Azure 切换走时会移除该键，使其他提供商恢复 SDK 路由。

分发入口——所有路由逻辑都在**一处**，即 `core/llm/factory.py`；构建逻辑在 `core/llm/client_builders.py`：

```text
get_llm(role)  # role ∈ {AGENT, REASONING, CLASSIFICATION, TOOLCALL}   # factory.py
  → resolve_llm_route()               # the single provider/transport decision  # factory.py
  → client_builders.build_agent_client(route) / build_reasoning_client(route, model_type)
      cli_provider_registration?  → CLI-backed subprocess client
      use_litellm_for_provider? → build_litellm_*_client(settings, provider)   # transports/litellm/routing.py
      else      → native SDK client in transports/sdk/agent_clients.py or transports/sdk/llm_clients.py
```

修改路由时，只改 `factory.py` 中的 `resolve_llm_route`；修改提供商客户端的构建方式时，改 `client_builders.py` 中的构建器——没有第二份副本需要同步。

`factory.py` 中有一个缓存，键为 `(role, transport, runtime_provider)`，在 `(transport, runtime_provider)` 变化时一起失效（而非仅 transport）。REPL `/model` 和向导环境同步会直接调用 `reset_llm_clients()`。

## 添加托管 API 提供商

1. 将提供商字面量添加到 `config/llm_settings.py` 中的 `LLMProvider` 及规范化/校验路径。
2. 在 `config/llm_models.py` 中添加 `ProviderModelDefaults` 行（模型层级默认值 / 基础 URL）。
3. 在 `config/llm_auth/provider_catalog.py` 中添加 `ProviderSpec`，并在 `surfaces/shared/llm_setup/catalog.py` 中添加匹配的 `ProviderOption`（模型环境变量、默认值、如需要则加 `endpoint_env`）。
3. 添加运行时构建（路由本身仍在 `core/llm/factory.py`；客户端在 `core/llm/client_builders.py` 中构建）：
   - **一方提供商**（自有 SDK 模型 + LiteLLM 前缀）：在 `core/llm/providers/provider_registry.py` 的 `FIRST_PARTY_PROVIDERS` 中添加**一行**——SDK 和 LiteLLM 构建器读取该表，因此除非客户端类是新的，否则无需按提供商分支。
   - **SDK 路径：** 在 `core/llm/transports/sdk/llm_clients.py` 和/或 `core/llm/transports/sdk/agent_clients.py` 中添加客户端类；`client_builders.py` 中的构建器会选择它。
   - **LiteLLM 路径（可选或必需）：** 由注册表行覆盖；仅在非标准情况（如 Azure）下才在 `core/llm/transports/litellm/routing.py` 中添加分支。
   - **OpenAI 兼容：** 在 `providers/openai_compat_providers.py`（SDK 兼容路径）和/或 `transports/litellm/routing.py`（LiteLLM 路径）中注册。
4. 如果引入新的非密钥环境变量键，更新 `surfaces/shared/llm_setup/env_sync.py`；当提供商需要持久化 URL/版本设置时，将端点键保留在 `active_non_secret` 中。
5. 在 `tests/core/runtime/llm/` 下添加或更新测试；如果引导流程有变更，也更新向导测试。

### Azure OpenAI（`azure-openai`）

Azure 使用**部署名称**（而非公开的 OpenAI 模型 ID）和资源**基础 URL**：

- `AZURE_OPENAI_BASE_URL`、`AZURE_OPENAI_API_KEY`、`AZURE_OPENAI_API_VERSION`（未设置时应用默认值）
- `AZURE_OPENAI_*_MODEL` 环境变量保存用户 Azure 资源中的部署名称
- LiteLLM 模型字符串：通过 `azure_openai_litellm_model()` 得到 `azure/<deployment>`

不要添加单独的 Azure 客户端类——扩展 `transports/litellm/routing.py` 和 `providers/azure_openai.py` 中的辅助工具。

### Google Vertex AI（`vertex-ai`）

Vertex AI 通过 Google 应用默认凭据（ADC）认证——无 API 密钥：

- `VERTEX_AI_PROJECT`、`VERTEX_AI_LOCATION`（默认为 `us-central1`）
- `VERTEX_AI_*_MODEL` 环境变量保存通过 Vertex 提供的 Gemini 模型 ID（如 `gemini-2.5-pro`）
- LiteLLM 模型字符串：通过 `resolve_vertex_ai_request_kwargs()` 得到 `vertex_ai/<model>`

不要添加单独的 Vertex 客户端类——扩展 `transports/litellm/routing.py` 和 `providers/vertex_ai.py` 中的辅助工具。与 Bedrock 一样，`credential_kind="ambient"`：OpenSRE 从不存储 Vertex 密钥，也不强制必填字段前置条件——缺失项目会在请求时以 LiteLLM/google-auth 错误的形式暴露。