# Kiro API 客户端架构

## 1. Identity

- **What it is:** Kiro API 客户端的核心请求层，包含 Provider（请求发送/重试/故障转移）和 MultiTokenManager（多凭据管理/Token 刷新/负载均衡）。
- **Purpose:** 提供可靠的 Kiro API 调用能力，通过多凭据故障转移和自动 Token 刷新实现高可用。

## 2. Core Components

- `src/kiro/provider.rs` (`KiroProvider`, `call_api_with_retry`, `call_mcp_with_retry`, `retry_delay`, `client_for`): 请求发送层，负责 HTTP 请求构建、状态码分类处理、指数退避重试、Client 按代理配置缓存。
- `src/kiro/token_manager.rs` (`MultiTokenManager`, `acquire_context`, `try_ensure_token`, `select_next_credential`, `persist_credentials`): 多凭据 Token 管理器，负责凭据选择、Token 刷新（双重检查锁定）、凭据禁用/自愈、统计持久化。
- `src/kiro/token_manager.rs` (`refresh_token`, `refresh_social_token`, `refresh_idc_token`): Token 刷新路由，根据 `auth_method` 分发到 Social 或 IdC 刷新路径。
- `src/kiro/machine_id.rs` (`generate_from_credentials`, `normalize_machine_id`): 设备指纹生成，三级优先级，输出固定 64 字符十六进制。
- `src/kiro/model/credentials.rs` (`KiroCredentials`, `CredentialsConfig`, `effective_auth_region`, `effective_api_region`, `effective_proxy`): 凭据数据模型，Region/代理回退链。

## 3. Execution Flow (LLM Retrieval Map)

### 3.1 API 请求流程（重试与故障转移）

- **1. 入口:** `call_api` / `call_api_stream` 委托给 `call_api_with_retry`。`src/kiro/provider.rs:157-176`
- **2. 重试上限计算:** `max_retries = min(凭据数 × 3, 9)`。`src/kiro/provider.rs:344`
- **3. 获取调用上下文:** 调用 `token_manager.acquire_context(model)` 获取绑定 id/credentials/token 的 `CallContext`。`src/kiro/provider.rs:353`
- **4. 构建请求:** 通过 `client_for` 获取按代理配置缓存的 Client，构建 POST 请求到 `q.{api_region}.amazonaws.com/generateAssistantResponse`。`src/kiro/provider.rs:370-391`
- **5. 状态码分类处理:**
  - `200` → `report_success`，重置失败计数。`src/kiro/provider.rs:416-419`
  - `400` / 其他 4xx → 直接 bail，不重试不切换。`src/kiro/provider.rs:454-456`, `src/kiro/provider.rs:510-512`
  - `401/403` → `report_failure`，累计 3 次后禁用并切换凭据。`src/kiro/provider.rs:459-485`
  - `402` + `MONTHLY_REQUEST_COUNT` → `report_quota_exhausted`，立即禁用并切换。`src/kiro/provider.rs:425-451`
  - `408/429/5xx` / 网络错误 → 指数退避重试，不切换凭据。`src/kiro/provider.rs:489-507`
- **6. 指数退避:** 基础 200ms，上限 2000ms，加 25% 随机抖动。`src/kiro/provider.rs:543-552`

### 3.2 Token 刷新流程（双重检查锁定）

- **1. 快速检查（无锁）:** 判断 Token 是否过期（提前 5 分钟）或即将过期（10 分钟内）。`src/kiro/token_manager.rs:879`
- **2. 获取刷新锁:** `refresh_lock: TokioMutex`，确保单一刷新者。`src/kiro/token_manager.rs:883`
- **3. 二次检查:** 重新读取凭据，若已被其他请求刷新则跳过。`src/kiro/token_manager.rs:886-895`
- **4. 刷新路由:** `refresh_token` 根据 `auth_method` 分发：`idc`/`builder-id`/`iam` → `refresh_idc_token`；其他 → `refresh_social_token`。`src/kiro/token_manager.rs:138-163`
- **5. 回写:** 刷新成功后更新 entries 并调用 `persist_credentials` 回写文件（仅多凭据数组格式）。`src/kiro/token_manager.rs:906-915`

### 3.3 负载均衡模式

- **priority（默认）:** 选择 `priority` 值最小的可用凭据，固定使用 `current_id`。`src/kiro/token_manager.rs:713-716`
- **balanced:** Least-Used 策略，选择 `success_count` 最少的凭据，平局按 priority 排序。每次请求重新选择。`src/kiro/token_manager.rs:703-708`
- opus 模型过滤：`subscription_title` 含 "FREE" 的凭据被排除。`src/kiro/token_manager.rs:688-689`

### 3.4 凭据禁用与自愈

- **禁用原因（`DisabledReason`）:** `Manual`（手动）、`TooManyFailures`（API 失败 ×3）、`TooManyRefreshFailures`（刷新失败 ×3）、`QuotaExceeded`（402 额度用尽）。`src/kiro/token_manager.rs:414-423`
- **自愈:** 仅 `TooManyFailures` 导致全部禁用时触发，重置失败计数并重新启用。`QuotaExceeded` 和 `TooManyRefreshFailures` 不自愈。`src/kiro/token_manager.rs:767-784`

### 3.5 设备指纹生成

- **三级优先级:** 凭据级 `machine_id` > 全局 `config.machine_id` > `SHA-256("KotlinNativeAPI/" + refreshToken)`。`src/kiro/machine_id.rs:38-62`
- **格式标准化:** 64 字符十六进制直接使用；UUID 格式去连字符后重复拼接为 64 字符。`src/kiro/machine_id.rs:14-33`

### 3.6 Region 配置回退链

- **Auth Region（Token 刷新）:** `凭据.auth_region` > `凭据.region` > `config.auth_region` > `config.region`。`src/kiro/model/credentials.rs:204-209`
- **API Region（API 请求）:** `凭据.api_region` > `config.api_region` > `config.region`（注意：`凭据.region` 不参与 API Region 回退链）。`src/kiro/model/credentials.rs:213-217`

## 4. Design Rationale

- 瞬态错误（429/5xx/网络）不切换凭据，避免网络抖动误禁所有凭据导致需要重启。
- Token 刷新使用 Tokio Mutex 而非 parking_lot Mutex，因为刷新是异步 HTTP 操作。
- 统计数据（`kiro_stats.json`）与凭据文件分离，debounce 30 秒，`Drop` 时强制落盘。
- Auth Region 和 API Region 回退链独立设计，因为 Token 刷新端点和 API 端点可能在不同 Region。
