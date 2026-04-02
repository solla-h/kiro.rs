# kiro-rs 项目概览

## 1. Identity

- **What it is:** 一个用 Rust 编写的 Anthropic Claude API 兼容代理服务。
- **Purpose:** 将标准 Anthropic API 请求转换为 Kiro API 请求，使任何兼容 Anthropic API 的客户端（如 Claude Code）能够通过 Kiro 后端获取 Claude 模型响应。

## 2. High-Level Description

kiro-rs 是一个协议转换代理，对外暴露 Anthropic Messages API（`/v1` 和 `/cc/v1`），对内调用 AWS Kiro API（`q.{region}.amazonaws.com`）。核心职责包括：请求格式转换（Anthropic → Kiro ConversationState）、OAuth Token 自动刷新与多凭据管理、SSE 流式响应转换（Kiro Event Stream → Anthropic SSE）、以及可选的 Web 管理界面。最终产物为单一静态链接二进制，前端 Admin UI 通过 `rust-embed` 编译时嵌入。

## 3. 核心功能

- Anthropic Messages API 完整兼容（流式/非流式、Thinking、Tool Use、WebSearch）
- 多凭据管理：优先级调度（priority）和均衡调度（balanced），自动故障转移
- OAuth Token 自动刷新：支持 Social 和 IdC (AWS SSO OIDC) 两种认证方式
- 凭据级代理配置：每个凭据可独立配置 HTTP/SOCKS5 代理
- Admin API + Web UI：凭据 CRUD、余额查询、负载均衡模式切换
- Claude Code 兼容端点（`/cc/v1`）：缓冲模式确保 `input_tokens` 准确

## 4. 技术栈

| 组件 | 技术 |
|------|------|
| Web 框架 | Axum 0.8 |
| 异步运行时 | Tokio (full features) |
| HTTP 客户端 | Reqwest 0.12 (rustls-tls) |
| 序列化 | Serde (JSON, camelCase) |
| CLI 解析 | Clap 4.5 |
| 日志 | tracing + tracing-subscriber |
| 静态文件嵌入 | rust-embed 8 |
| 常量时间比较 | subtle 2.6 |
| 并发原语 | parking_lot 0.12 + tokio::sync::Mutex |
| 前端 (Admin UI) | React 18 + Vite 5 + TypeScript + Tailwind CSS + Radix UI + TanStack Query |

## 5. 请求流转路径

```
客户端 (Anthropic API 格式)
  │
  ▼
[auth_middleware] ── API Key 验证 (x-api-key / Bearer)
  │
  ▼
[handlers.rs] ── post_messages / post_messages_cc
  │
  ├─ WebSearch? ──→ [websearch.rs] ──→ provider.call_mcp ──→ Kiro MCP 端点
  │
  ▼
[converter.rs] ── convert_request (Anthropic → Kiro ConversationState)
  │
  ▼
[provider.rs] ── call_api / call_api_stream (带重试 + 故障转移)
  │
  ├─ [token_manager.rs] ── acquire_context (Token 刷新 + 凭据选择)
  │
  ▼
Kiro API (q.{region}.amazonaws.com/generateAssistantResponse)
  │
  ▼
[stream.rs / handlers.rs] ── Kiro Event Stream → Anthropic SSE/JSON 响应
  │
  ▼
客户端
```

## 6. 模块组织

| 模块路径 | 职责 |
|----------|------|
| `src/main.rs` | 启动入口：CLI 解析 → 配置加载 → 组件组装 → Axum 服务启动 |
| `src/anthropic/` | Anthropic API 兼容层：路由、Handler、认证中间件、类型定义、协议转换、SSE 流处理、WebSearch |
| `src/kiro/provider.rs` | Kiro API 客户端：请求发送、重试策略（指数退避）、故障转移 |
| `src/kiro/token_manager.rs` | Token 管理：单/多凭据管理、OAuth 刷新、凭据选择（priority/balanced）、凭据持久化 |
| `src/kiro/machine_id.rs` | 设备指纹生成（SHA-256） |
| `src/kiro/model/` | Kiro 数据模型：凭据、请求/响应事件、Token 刷新、使用额度 |
| `src/kiro/parser/` | AWS Event Stream 二进制协议解析器（帧解析、CRC 校验） |
| `src/admin/` | Admin API：凭据 CRUD、余额查询（5 分钟缓存）、负载均衡配置 |
| `src/admin_ui/` | Admin UI 静态文件服务（rust-embed 嵌入） |
| `src/model/` | 应用级模型：Config、CLI Args |
| `src/common/` | 公共工具：API Key 提取、常量时间比较 |
| `admin-ui/` | 前端工程（React + Vite），构建产物嵌入二进制 |
