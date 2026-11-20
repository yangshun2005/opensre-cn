# 订阅适配器

`config/llm_auth/subscriptions.py` 负责提供商身份、模型默认值、认证方式和设置提示。加入此目录即选择订阅计费；`auth_method` 描述用户如何授权（`oauth`、`api_key` 或提供商支持的替代方式）。OAuth 登录本身不建立订阅权益，订阅计费也不要求浏览器。

`registry.py` 配对 `types.py` 中定义的两个契约：

- `SubscriptionAdapter`：登录、请求时凭证解析/续期、模型发现、验证和登出。登录在激活账户前选择模型。解析过程绝不打开浏览器，也不读取普通 API 计费密钥。
- `SubscriptionClientFactory`：Agent 和推理客户端的构建。提供商拥有自己的端点、请求格式、工具序列化和订阅限制。使用相同的 API 协议不建立订阅资格。

设置按提供商身份分发。运行时构建、就绪检查和缓存键使用现有提供商目录；仅目录条目而无适配器会失败。客户端在每次请求时解析活动账户。令牌绝不进入缓存键。

## 添加提供商

1. 确认支持的开户/客户端注册、符合条件的计划/工作负载、订阅端点、作用域、发现、续期/撤销和配额语义。
2. 将元数据添加到现有订阅目录，并将提供商字面量添加到 `config/llm_settings.py`。保持按用量计费的 API 连接独立。
3. 在提供商自有模块中实现两个契约，并添加一次注册。复用 `subscription_lock`、`read_store` 和 `write_store` 进行受保护的原子存储。隔离提供商/账户凭证，串行化轮换的刷新令牌，清除终态无效授权，在瞬时故障时保留凭证，并在登出期间保留注册。
4. 实现提供商特定的推理。使用 Responses 的提供商可复用 `ResponsesAgentClient`/`ResponsesLLMClient`，提供自己的请求序列化器。ChatGPT 的 `additional_tools` 扩展不得发送给其他提供商。
5. 覆盖账户/提供商隔离、显式计费、请求时续期、模型选择、终态/瞬时错误、工具/结果和推理重放、实时文本、中断流、取消/关闭、用量核算和配额失败。失败绝不能切换到 API 计费凭证或路由。
6. 记录设置，并使用符合条件的账户执行实时登录、推理/工具重放、续期、重连、配额、计费和登出验收。

## Grok 状态

Grok 适配器使用 RFC 8628 设备授权和 LiteLLM 原生 xAI Responses 路由。浏览器可在另一台设备上运行；没有回环回调。默认使用 OpenCode、Kilo 和 Hermes 共享的公共 Grok CLI 客户端注册，并带有 `referrer=opensre` 归属。参见 [Kilo 的实现](https://github.com/Kilo-Org/kilocode/blob/main/packages/opencode/src/plugin/xai.ts)。`OPENSRE_GROK_OAUTH_CLIENT_ID` 可选择为新账户覆盖它。凭证在续期和重连期间保留原始客户端 ID；不复用其他应用的凭证存储。

xAI 正式宣布了 [OpenCode](https://x.ai/news/grok-opencode) 和 [Kilo Code](https://x.ai/news/grok-kilocode) 的订阅访问。这确立了这些集成；OpenSRE 中的当前计划资格和实时订阅计费仍未确认。认证/模型列表成功不能证明推理权益。适配器已通过合成 OAuth 和真实 LiteLLM HTTP/SSE 测试覆盖；在声称生产支持前仍需本地订阅验收。