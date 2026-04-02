# Kiro 数据模型参考

## 1. Core Summary

Kiro API 客户端的数据模型分为三层：请求模型（`KiroRequest` → `ConversationState` → 消息/工具）、事件模型（AWS Event Stream 二进制帧，四种事件类型 + Error/Exception）、凭据模型（`KiroCredentials` 支持 Social 和 IdC 两种认证）。所有 JSON 序列化使用 camelCase，可选字段为 None 时跳过序列化。

## 2. Source of Truth

### 请求模型层级

- **顶层请求:** `src/kiro/model/requests/kiro.rs` (`KiroRequest`) - 包含 `conversationState` 和可选 `profileArn`。
- **对话状态:** `src/kiro/model/requests/conversation.rs` (`ConversationState`) - 包含 `conversationId`、`currentMessage`、`history[]`，可选 `agentTaskType`（如 `"vibe"`）、`chatTriggerType`（`"MANUAL"` / `"AUTO"`）。
- **当前消息:** `src/kiro/model/requests/conversation.rs` (`CurrentMessage` → `UserInputMessage`) - 包含 `content`、`modelId`、`images[]`、`origin`（默认 `"AI_EDITOR"`）、`userInputMessageContext`（tools + toolResults）。
- **历史消息:** `src/kiro/model/requests/conversation.rs` (`Message`) - `#[serde(untagged)]` 枚举，`User(HistoryUserMessage)` 序列化为 `{ "userInputMessage": ... }`，`Assistant(HistoryAssistantMessage)` 序列化为 `{ "assistantResponseMessage": ... }`。
- **工具定义:** `src/kiro/model/requests/tool.rs` (`Tool` → `ToolSpecification` → `InputSchema`) - 工具名称、描述、JSON Schema 输入定义。
- **工具结果:** `src/kiro/model/requests/tool.rs` (`ToolResult`) - 包含 `toolUseId`、`content`、`status`（`"success"` / `"error"`）。
- **工具调用记录:** `src/kiro/model/requests/tool.rs` (`ToolUseEntry`) - 记录在 `assistantResponseMessage.toolUses[]` 中。

### 事件模型（四种事件类型）

- **事件基础:** `src/kiro/model/events/base.rs` (`Event`, `EventType`, `EventPayload`) - 统一事件枚举，按 `:message-type` 和 `:event-type` 帧头分发。
- **助手响应:** `src/kiro/model/events/assistant.rs` (`AssistantResponseEvent`) - 流式文本增量，`content: String`，`#[serde(flatten)]` 吸收未知字段。
- **工具使用:** `src/kiro/model/events/tool_use.rs` (`ToolUseEvent`) - 流式工具调用，`name`、`tool_use_id`、`input`（JSON 字符串，可能是部分数据）、`stop: bool`（终止信号）。
- **上下文使用率:** `src/kiro/model/events/context_usage.rs` (`ContextUsageEvent`) - `context_usage_percentage: f64`（0-100）。
- **计费事件:** `Metering(())` - payload 被忽略。
- **错误/异常:** `Event::Error { error_code, error_message }` 和 `Event::Exception { exception_type, message }` - 从帧头和 payload 文本解析。

### 凭据模型

- **凭据结构:** `src/kiro/model/credentials.rs` (`KiroCredentials`) - 核心字段：`accessToken`、`refreshToken`、`profileArn`、`expiresAt`（RFC3339）、`authMethod`（`"social"` / `"idc"`）、`clientId`/`clientSecret`（IdC 需要）、`name`（可选，凭据自定义名称，用于 Admin UI 显示辨识）、`priority`（默认 0，越小越优先）、`region`/`authRegion`/`apiRegion`、`machineId`、`email`、`subscriptionTitle`、`proxyUrl`/`proxyUsername`/`proxyPassword`、`disabled`。
- **凭据配置:** `src/kiro/model/credentials.rs` (`CredentialsConfig`) - `#[serde(untagged)]` 枚举，自动识别单对象或数组格式。
- **Token 刷新模型:** `src/kiro/model/token_refresh.rs` - Social: `RefreshRequest`/`RefreshResponse`；IdC: `IdcRefreshRequest`/`IdcRefreshResponse`。IdC 响应无 `profileArn`。
- **使用额度模型:** `src/kiro/model/usage_limits.rs` (`UsageLimitsResponse`) - 包含 `subscriptionInfo`（`subscriptionTitle`）和 `usageBreakdownList`，总额度累加基础 + 激活的 free trial + 激活的 bonus。

### 传输协议

- **帧格式:** `src/kiro/parser/frame.rs` (`Frame`) - AWS Event Stream 二进制帧：4 字节总长度 + 4 字节头部长度 + 4 字节 prelude CRC32 + 变长 headers + 变长 payload + 4 字节 message CRC32。最大 16 MB。
- **帧头:** `src/kiro/parser/header.rs` (`Headers`) - 类型化访问器：`message_type()`（`:message-type`）、`event_type()`（`:event-type`）、`exception_type()`、`error_code()`。

### 相关架构文档

- **客户端架构:** `/llmdoc/architecture/kiro-client-architecture.md` - 重试/故障转移/Token 刷新/负载均衡的详细执行流程。
