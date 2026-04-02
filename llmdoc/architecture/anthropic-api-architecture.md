# Anthropic API 兼容层架构

## 1. Identity

- **What it is:** 将 Anthropic Messages API 协议转换为 Kiro 内部 API 协议的兼容层。
- **Purpose:** 使 Claude Code 等 Anthropic 客户端能够通过标准 Anthropic API 格式与 Kiro 后端通信。

## 2. Core Components

- `src/anthropic/router.rs` (`create_router_with_provider`): 路由注册入口，构建 `/v1` 和 `/cc/v1` 两组端点，应用认证中间件和 CORS。
- `src/anthropic/middleware.rs` (`AppState`, `auth_middleware`, `cors_layer`): 共享状态和认证中间件，API Key 通过常量时间比较验证。
- `src/anthropic/handlers.rs` (`post_messages`, `post_messages_cc`, `get_models`, `count_tokens`): 请求处理函数，协调转换、调用上游、流式响应。
- `src/anthropic/converter.rs` (`convert_request`, `map_model`, `build_history`): 协议转换核心，将 Anthropic 请求映射为 Kiro `ConversationState`。
- `src/anthropic/types.rs` (`MessagesRequest`, `Thinking`, `Tool`, `ContentBlock`): Anthropic API 类型定义。
- `src/anthropic/stream.rs` (`StreamContext`, `BufferedStreamContext`, `SseStateManager`): 流式响应处理，Kiro 事件流转换为 Anthropic SSE 事件。
- `src/anthropic/websearch.rs` (`handle_websearch_request`, `create_mcp_request`): WebSearch 特殊路径，通过 Kiro MCP 端点实现。

## 3. Execution Flow (LLM Retrieval Map)

### 3.1 路由结构

两组端点共享认证和 CORS，唯一行为差异在流式响应模式：

| 路径前缀 | 端点 | Handler | 流式模式 |
|---------|------|---------|---------|
| `/v1` | `GET /models` | `get_models` | - |
| `/v1` | `POST /messages` | `post_messages` | 即时流式 (`StreamContext`) |
| `/v1` | `POST /messages/count_tokens` | `count_tokens` | - |
| `/cc/v1` | `POST /messages` | `post_messages_cc` | 缓冲流式 (`BufferedStreamContext`) |
| `/cc/v1` | `POST /messages/count_tokens` | `count_tokens` | - |

`src/anthropic/router.rs:50-76`

### 3.2 请求处理流程

- **1. 认证:** `auth_middleware` 提取 `x-api-key` 或 `Authorization: Bearer` 头，常量时间比较。`src/anthropic/middleware.rs`
- **2. Thinking 覆写:** `override_thinking_from_model_name` 检测模型名 `-thinking` 后缀，Opus 4.6 设为 `adaptive`，其他设为 `enabled`，`budget_tokens=20000`。`src/anthropic/handlers.rs:598-629`
- **3. WebSearch 短路:** 若 tools 仅含一个 `web_search`，直接路由到 MCP 端点，跳过常规转换。`src/anthropic/websearch.rs:104-108`
- **4. 协议转换:** `convert_request` 执行完整转换。`src/anthropic/converter.rs:200-303`
- **5. 上游调用:** 构建 `KiroRequest`，调用 `provider.call_api_stream` 或 `provider.call_api`。`src/anthropic/handlers.rs:244-295`
- **6. 响应转换:** `StreamContext` / `BufferedStreamContext` 将 Kiro 事件流转为 Anthropic SSE。`src/anthropic/stream.rs:458-1065`

### 3.3 模型名映射

`src/anthropic/converter.rs:83-103` (`map_model`)

| Anthropic 模型名模式 | Kiro 模型 ID | 上下文窗口 |
|---------------------|-------------|-----------|
| 含 `sonnet` + `4-6`/`4.6` | `claude-sonnet-4.6` | 1,000,000 |
| 含 `sonnet`（其他） | `claude-sonnet-4.5` | 200,000 |
| 含 `opus` + `4-5`/`4.5` | `claude-opus-4.5` | 200,000 |
| 含 `opus`（其他） | `claude-opus-4.6` | 1,000,000 |
| 含 `haiku` | `claude-haiku-4.5` | 200,000 |

### 3.4 协议转换核心逻辑

`convert_request` 执行以下步骤（`src/anthropic/converter.rs:200-303`）：

1. `map_model` 映射模型名，未知模型返回 `UnsupportedModel` 错误
2. 验证消息非空
3. 剥离末尾 assistant prefill（Claude 4.x 已弃用）
4. 从 `metadata.user_id` 提取 session UUID 作为 `conversation_id`，否则生成新 UUID
5. 系统消息转为 user+assistant 历史对（assistant 回复 "I will follow these instructions."）
6. Thinking XML 标签注入系统消息头部（`<thinking_mode>` + `<max_thinking_length>` 或 `<thinking_effort>`）
7. 连续同角色消息合并，末尾孤立 user 消息自动配对 "OK" assistant 响应
8. 工具 schema 规范化（修复 `null` 的 `required`/`properties`/`type`）
9. Write/Edit 工具追加分块策略描述后缀
10. 验证 tool_use/tool_result 配对，移除孤立项
11. 为历史中引用但 tools 列表缺失的工具生成占位符定义

### 3.5 流式响应处理

**StreamContext**（`/v1`）：即时转发，`src/anthropic/stream.rs:458-1065`
- 收到 Kiro 事件立即转换为 SSE 事件发送
- `input_tokens` 在 `message_start` 中使用估算值，后续通过 `contextUsageEvent` 更新到 `message_delta`
- 每 25 秒发送 ping 保活

**BufferedStreamContext**（`/cc/v1`）：缓冲后发送，`src/anthropic/stream.rs:1067-1158`
- 缓冲所有事件直到流结束
- 用 `contextUsageEvent` 的准确值回填 `message_start.usage.input_tokens`
- 等待期间仅发送 ping 保活

**Thinking 模式处理**：`src/anthropic/stream.rs:641-784`
- Kiro 返回的 `<thinking>...</thinking>` XML 标签被解析提取为独立的 `thinking` content block
- 使用引号字符检测跳过被引用的标签（如 `` `</thinking>` ``）
- `find_real_thinking_end_tag` 要求标签后有 `\n\n` 才认定为真正结束
- `find_real_thinking_end_tag_at_buffer_end` 处理边界场景（tool_use 紧跟或流结束）
- 仅产生 thinking 块时，补发空格 text 块并设 `stop_reason=max_tokens`

**SseStateManager**：`src/anthropic/stream.rs:225-454`
- 强制 SSE 事件顺序：`message_start` → `content_block` 生命周期 → `message_delta` → `message_stop`
- tool_use 开始时自动关闭当前 text block
- 后续文本自动创建新 text block（自愈机制）

## 4. Design Rationale

- `/cc/v1` 缓冲模式的存在是因为 Claude Code 依赖 `message_start` 中准确的 `input_tokens` 来判断上下文窗口使用率，而 Kiro 的 `contextUsageEvent` 在流末尾才到达。
- 系统消息转为 user+assistant 对是因为 Kiro API 不支持独立的 system role，只能通过历史消息注入。
- Thinking XML 标签注入而非使用 API 参数，是因为 Kiro API 通过系统消息中的 XML 标签控制 thinking 行为。
- `SYSTEM_CHUNKED_POLICY` 常量追加到每条系统消息，确保模型遵守 Write/Edit 工具的分块大小限制。
