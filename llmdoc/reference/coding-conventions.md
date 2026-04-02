# 编码规范

本项目的核心编码约定。所有新增代码必须遵循以下规则。

## 1. 命名规范

| 类别 | 风格 | 示例 |
|------|------|------|
| 结构体 / 枚举 | `PascalCase` | `KiroProvider`, `ParseError`, `TlsBackend` |
| 函数 / 方法 / 变量 | `snake_case` | `build_client`, `extract_api_key` |
| 常量 | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES_PER_CREDENTIAL`, `MAX_BODY_SIZE` |
| 模块文件 | `snake_case` | `token_manager.rs`, `http_client.rs` |
| JSON 字段序列化 | `camelCase` | 通过 `#[serde(rename_all = "camelCase")]` |

特殊情况：`TlsBackend` 枚举使用 `rename_all = "kebab-case"`，因其值映射配置文件字段。

## 2. 模块组织

- 每个功能域一个目录：`anthropic/`, `kiro/`, `admin/`, `common/`, `model/`。
- 每个目录包含 `mod.rs`，负责模块声明和 `pub use` 重导出。
- 模块顶部使用 `//!` 文档注释描述职责。
- 数据模型与业务逻辑分离：类型定义放 `model/` 或 `types.rs`，逻辑放 `handlers.rs`、`service.rs`。
- 公共工具集中在 `src/common/` 模块。

参考：`src/anthropic/mod.rs`, `src/kiro/mod.rs`, `src/admin/mod.rs`。

## 3. 错误处理（三层模式）

| 层级 | 方式 | 示例位置 |
|------|------|----------|
| 底层/库 | 自定义错误枚举 + `impl Display` + `impl Error` + `From<T>` | `src/admin/error.rs` (`AdminServiceError`), `src/kiro/parser/error.rs` (`ParseError`) |
| 应用层 | `anyhow::Result<T>` + `anyhow::bail!` | `src/kiro/token_manager.rs`, `src/model/config.rs` |
| HTTP 层 | 错误映射为 HTTP 状态码 | `src/anthropic/handlers.rs` (`map_provider_error`) |

常用模式：
- 类型别名：`type ParseResult<T> = Result<T, ParseError>`
- 自定义枚举提供 `status_code()` 和 `into_response()` 方法

## 4. 依赖注入与共享状态

- 共享状态通过 `AppState` 结构体传递，使用 Builder 模式构建。
- `KiroProvider` 通过 `Arc` 包装实现跨线程共享。
- `MultiTokenManager` 通过 `Arc` 注入到 `KiroProvider` 和 `AdminService`。
- axum handler 通过 `State(state): State<AppState>` 提取器获取状态。

参考：`src/anthropic/middleware.rs:20-48` (`AppState`)。

## 5. 序列化约定

- 所有 API 类型统一使用 `#[serde(rename_all = "camelCase")]`。
- 可选字段使用 `#[serde(skip_serializing_if = "Option::is_none")]`。
- 配置枚举根据场景选择 `camelCase` 或 `kebab-case`。

## 6. 并发原语选择

| 场景 | 原语 |
|------|------|
| 同步互斥（client 缓存、统计） | `parking_lot::Mutex` |
| 异步互斥（token 刷新） | `tokio::sync::Mutex` |
| 标志位 | `std::sync::atomic::AtomicBool` |
| 跨线程共享所有权 | `Arc<T>` |
| 全局一次性初始化 | `OnceLock<T>` |

## 7. 其他约定

- 注释和提交信息语言：简体中文。
- Git 提交格式：`type(scope): 中文描述`（Conventional Commits）。
- Builder 模式：`with_xxx` 方法链式构建复杂对象。
- Rust edition：2024。
- 无显式 rustfmt/clippy 配置，依赖 edition 默认行为。
