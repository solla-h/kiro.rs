# Admin 管理系统架构

## 1. Identity

- **What it is:** kiro-rs 的可选管理子系统，提供凭据管理 REST API 和内嵌 SPA 管理界面。
- **Purpose:** 让运维人员通过 Web UI 或 API 对凭据进行增删改查、余额查询、负载均衡配置等操作。

## 2. Core Components

- `src/admin/router.rs` (`create_admin_router`): 注册全部 10 个 REST 端点，挂载认证中间件，挂载路径 `/api/admin`。
- `src/admin/handlers.rs` (`get_all_credentials`, `add_credential`, `delete_credential`, `set_credential_disabled`, `set_credential_priority`, `reset_failure_count`, `force_refresh_token`, `get_credential_balance`, `get_load_balancing_mode`, `set_load_balancing_mode`): 各端点 handler，从 `AdminState` 提取 service 并委托调用。
- `src/admin/service.rs` (`AdminService`, `CachedBalance`): 业务逻辑层，持有 `Arc<MultiTokenManager>` 和余额缓存（`Mutex<HashMap<u64, CachedBalance>>`），TTL 300 秒。
- `src/admin/error.rs` (`AdminServiceError`): 四种错误变体 `NotFound`(404)、`UpstreamError`(502)、`InternalError`(500)、`InvalidCredential`(400)。
- `src/admin/middleware.rs` (`AdminState`, `admin_auth_middleware`): 共享状态与认证中间件，调用 `constant_time_eq` 校验 API Key。
- `src/admin/types.rs` (`CredentialsStatusResponse`, `AddCredentialRequest`, `BalanceResponse`, `AdminErrorResponse`): 请求/响应序列化类型。`AddCredentialRequest` 包含可选 `name` 字段用于设置凭据自定义名称；`CredentialStatusItem` 包含可选 `name` 字段用于返回凭据名称。
- `src/common/auth.rs` (`extract_api_key`, `constant_time_eq`): 提取 API Key（`x-api-key` 优先，其次 `Authorization: Bearer`），使用 `subtle::ct_eq` 常量时间比较。
- `src/admin_ui/router.rs` (`Asset`, `create_admin_ui_router`): `rust-embed` 嵌入 `admin-ui/dist`，SPA fallback，挂载路径 `/admin`。
- `src/model/config.rs` (`admin_api_key`): 配置字段，`Option<String>`，非空时启用 Admin 功能。
- `admin-ui/src/api/credentials.ts`: axios 实例，`baseURL: /api/admin`，拦截器自动注入 `x-api-key`。
- `admin-ui/src/hooks/use-credentials.ts`: TanStack Query hooks，凭据列表 30 秒轮询，mutation 成功后 invalidate。
- `admin-ui/src/components/dashboard.tsx` (`Dashboard`): 主面板，统计卡片 + 凭据列表 + 批量操作。

## 3. Execution Flow (LLM Retrieval Map)

### 3.1 启用判断

- **1.** `src/main.rs:111-136` 检查 `config.admin_api_key` 是否非空，非空则创建 `AdminService` → `AdminState`，将 Admin API nest 到 `/api/admin`，Admin UI nest 到 `/admin`。

### 3.2 API 请求处理

- **1. 认证:** 请求进入 `admin_auth_middleware`（`src/admin/middleware.rs:36-50`），调用 `extract_api_key`（`src/common/auth.rs:14-31`）提取 key，`constant_time_eq`（`src/common/auth.rs:39-41`）比较，失败返回 401。
- **2. 路由分发:** `create_admin_router`（`src/admin/router.rs:35-56`）将请求分发到对应 handler。
- **3. Handler 委托:** handler（`src/admin/handlers.rs`）从 `AdminState` 取出 `Arc<AdminService>`，调用 service 方法。
- **4. 业务逻辑:** `AdminService`（`src/admin/service.rs`）执行逻辑，所有凭据操作委托 `MultiTokenManager`。
- **5. 错误映射:** service 方法通过 `classify_*` 方法将 `anyhow::Error` 转为 `AdminServiceError`（`src/admin/error.rs`），handler 调用 `status_code()` + `into_response()` 生成 HTTP 响应。

### 3.3 余额缓存机制

- **1.** `get_balance`（`src/admin/service.rs:126-156`）先查内存缓存（`Mutex<HashMap>`），命中且未过期（300s TTL）直接返回。
- **2.** 缓存未命中则调用 `fetch_balance` → `token_manager.get_usage_limits_for(id)`。
- **3.** 结果写入内存缓存，同时调用 `save_balance_cache`（`src/admin/service.rs:321-340`）持久化到 `<cache_dir>/kiro_balance_cache.json`。
- **4.** 启动时 `load_balance_cache_from`（`src/admin/service.rs:287-319`）加载磁盘缓存并过滤过期条目。

### 3.4 Admin UI 静态文件服务

- **1.** `create_admin_ui_router`（`src/admin_ui/router.rs:18-22`）注册 `/` 和 `/{*file}` 路由。
- **2.** `static_handler`（`src/admin_ui/router.rs:30-68`）从 `rust-embed` 的 `Asset` 中查找文件，找到则返回并设置缓存头（`assets/` 目录 1 年不可变缓存）。
- **3.** 文件不存在且非资源路径时 SPA fallback 到 `index.html`。

### 3.5 前端技术栈

- React 18 + TypeScript + Vite 5 + Tailwind CSS + Radix UI + TanStack Query
- API 层：`admin-ui/src/api/credentials.ts`（axios，拦截器注入 `x-api-key`）
- 状态管理：`admin-ui/src/hooks/use-credentials.ts`（TanStack Query hooks，30s 轮询）
- 登录态：`admin-ui/src/lib/storage.ts`（`localStorage` 存储 `adminApiKey`）

## 4. Design Rationale

- 认证使用静态 API Key + 常量时间比较，无 JWT/Session，适合内部管理场景。
- 余额双层缓存（内存 + 磁盘 JSON）减少上游 API 调用，重启后缓存可恢复。
- 错误分类通过字符串匹配 `anyhow::Error` 消息实现，因底层 `MultiTokenManager` 使用非类型化错误传播。
- 前端通过 `rust-embed` 编译进二进制，部署为单一可执行文件，无需独立前端服务。
- 删除凭据前必须先禁用（API 层 + UI 层双重校验），防止误删活跃凭据。

## 5. API 端点速查

| 方法 | 路径 | Handler | 说明 |
|------|------|---------|------|
| GET | `/credentials` | `get_all_credentials` | 获取所有凭据状态 |
| POST | `/credentials` | `add_credential` | 添加新凭据 |
| DELETE | `/credentials/{id}` | `delete_credential` | 删除凭据（需先禁用） |
| POST | `/credentials/{id}/disabled` | `set_credential_disabled` | 设置禁用状态 |
| POST | `/credentials/{id}/priority` | `set_credential_priority` | 设置优先级 |
| POST | `/credentials/{id}/reset` | `reset_failure_count` | 重置失败计数 |
| POST | `/credentials/{id}/refresh` | `force_refresh_token` | 强制刷新 Token |
| GET | `/credentials/{id}/balance` | `get_credential_balance` | 查询余额（5min 缓存） |
| GET | `/config/load-balancing` | `get_load_balancing_mode` | 获取负载均衡模式 |
| PUT | `/config/load-balancing` | `set_load_balancing_mode` | 设置负载均衡模式 |

所有端点前缀为 `/api/admin`，需携带 `x-api-key` 或 `Authorization: Bearer` 认证头。
