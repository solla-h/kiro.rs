# Anthropic API 类型参考

## 1. Core Summary

Anthropic API 兼容层的类型定义集中在 `src/anthropic/types.rs`，涵盖请求、响应、错误、模型列表和 token 计数等结构体。WebSearch 相关类型定义在 `src/anthropic/websearch.rs`。所有类型使用 serde 进行序列化/反序列化，部分字段有自定义反序列化逻辑。

## 2. Source of Truth

- **核心类型定义:** `src/anthropic/types.rs` - `MessagesRequest`, `Thinking`, `Tool`, `ContentBlock`, `ErrorResponse`, `Model` 等全部 Anthropic API 类型。
- **WebSearch 类型:** `src/anthropic/websearch.rs` - `McpRequest`, `McpResponse`, `WebSearchResults`, `WebSearchResult` 等 MCP 和搜索结果类型。
- **Kiro 请求类型:** `src/kiro/model/requests/` - 转换目标类型 `ConversationState`, `KiroRequest`, `Tool` 等。
- **Kiro 事件类型:** `src/kiro/model/events.rs` - 上游事件 `Event` 枚举（`AssistantResponse`, `ToolUse`, `ContextUsage`, `Exception`）。
- **架构文档:** `/llmdoc/architecture/anthropic-api-architecture.md` - 完整的架构和执行流程。

## 3. 关键类型说明

### MessagesRequest (`types.rs:115-130`)

主请求体。`system` 字段通过自定义 `deserialize_system` 同时支持字符串和数组格式。`metadata.user_id` 用于提取 session UUID。

### Thinking (`types.rs:67-83`)

`thinking_type` 取值 `"enabled"` 或 `"adaptive"`。`budget_tokens` 反序列化时上限截断为 `MAX_BUDGET_TOKENS = 24576`，默认值 20000。`is_enabled()` 对两种类型均返回 true。

### Tool (`types.rs:210-236`)

统一结构体支持普通工具和 WebSearch 工具。WebSearch 工具通过 `tool_type` 字段（如 `"web_search_20250305"`）和 `name == "web_search"` 识别。`is_web_search()` 检查 `tool_type` 是否以 `"web_search"` 开头。

### ContentBlock (`types.rs:239-261`)

通用内容块，`block_type` 区分 `text`、`thinking`、`tool_use`、`tool_result`、`image` 等类型。所有可选字段使用 `skip_serializing_if = "Option::is_none"`。

### WebSearch 特殊处理 (`websearch.rs`)

- 触发条件：`tools` 有且仅有一个 `name == "web_search"` 的工具（`has_web_search_tool`，`websearch.rs:104-108`）
- 绕过常规转换流程，直接构建 JSON-RPC 2.0 `tools/call` MCP 请求
- 请求 ID 格式：`web_search_tooluse_{22位随机}_{毫秒时间戳}_{8位随机}`
- Tool use ID 格式：`srvtoolu_{32位UUID无连字符}`
- 始终返回 SSE 流，事件序列固定为：text(决策) → `server_tool_use` → `web_search_tool_result` → text(摘要) → `message_delta` → `message_stop`
- 查询提取时自动去除 `"Perform a web search for the query: "` 前缀
