# 如何添加 API 端点和修改协议转换

## 添加新的 API 端点

1. **定义请求/响应类型：** 在 `src/anthropic/types.rs` 中添加新的请求和响应结构体，使用 `#[derive(Debug, Deserialize)]` 和 `#[derive(Debug, Serialize)]`。

2. **实现 Handler 函数：** 在 `src/anthropic/handlers.rs` 中添加 `pub async fn` handler。签名参考现有 handler：用 `State(state): State<AppState>` 获取共享状态，用 `JsonExtractor(payload)` 解析请求体。

3. **注册路由：** 在 `src/anthropic/router.rs:50-76` 的 `create_router_with_provider` 中，将新端点添加到 `v1_routes` 或 `cc_v1_routes`（或两者）。使用 `.route("/your_path", post(your_handler))` 或 `get()`。

4. **添加认证（如需要）：** 新路由如果嵌套在 `v1_routes` 或 `cc_v1_routes` 下，会自动继承 `auth_middleware`。独立路由需手动添加 `.layer(middleware::from_fn_with_state(state.clone(), auth_middleware))`。

5. **验证：** 运行 `cargo test` 确认编译通过和现有测试不受影响。在 `handlers.rs` 中为新 handler 添加单元测试。

## 修改协议转换逻辑

1. **修改模型映射：** 编辑 `src/anthropic/converter.rs:83-103` 的 `map_model` 函数。同步更新 `get_context_window_size`（`converter.rs:109-114`）和 `get_models` 中的硬编码模型列表（`handlers.rs:76-167`）。

2. **修改消息转换：** 核心转换在 `convert_request`（`converter.rs:200-303`）。历史消息构建在 `build_history`（`converter.rs:593-687`）。连续消息合并在 `merge_assistant_messages` 和 `merge_user_messages`。

3. **修改工具转换：** 工具 schema 规范化在 `normalize_json_schema`（`converter.rs:21-60`）。工具描述后缀在 `convert_tools`（`converter.rs:519-555`）。

4. **修改流式响应：** SSE 事件生成在 `StreamContext::process_kiro_event`（`stream.rs:576-619`）。Thinking 标签解析在 `process_content_with_thinking`（`stream.rs:641-784`）。

5. **运行测试：** `converter.rs` 和 `stream.rs` 底部有大量单元测试。运行 `cargo test -- anthropic` 验证修改。
