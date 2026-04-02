# 如何管理 Kiro 凭据

配置和管理 Kiro API 客户端的凭据，包括认证方式选择、代理配置和 Token 自动刷新。

1. **配置凭据文件:** 在 `credentials.json` 中配置凭据。支持两种格式：
   - 单凭据（旧格式）：JSON 对象 `{ "refreshToken": "...", ... }`
   - 多凭据（新格式）：JSON 数组 `[{ "refreshToken": "...", "priority": 0 }, ...]`
   - 格式由 `CredentialsConfig`（`#[serde(untagged)]`）自动识别。参考 `src/kiro/model/credentials.rs:119-126`

2. **选择认证方式:** 通过 `authMethod` 字段指定：
   - `"social"`（默认）：Kiro OAuth，仅需 `refreshToken`。刷新端点：`prod.{auth_region}.auth.desktop.kiro.dev/refreshToken`
   - `"idc"` / `"builder-id"` / `"iam"`：AWS SSO OIDC，需额外提供 `clientId` 和 `clientSecret`。刷新端点：`oidc.{auth_region}.amazonaws.com/token`
   - 未指定时根据 `clientId`/`clientSecret` 是否存在自动判断。参考 `src/kiro/token_manager.rs:146-163`

3. **配置代理（可选）:** 支持全局和凭据级两层代理：
   - 全局代理：在 `config.json` 中配置，所有凭据默认使用
   - 凭据级代理：在凭据中设置 `proxyUrl`（支持 http/https/socks5），可选 `proxyUsername`/`proxyPassword`
   - 特殊值 `"direct"`：显式绕过全局代理（大小写不敏感）
   - 优先级：凭据代理 > 全局代理 > 无代理。参考 `src/kiro/model/credentials.rs:222-236`

4. **配置 Region（可选）:** Auth Region 和 API Region 独立配置：
   - Auth Region 回退链：`凭据.authRegion` > `凭据.region` > `config.authRegion` > `config.region`
   - API Region 回退链：`凭据.apiRegion` > `config.apiRegion` > `config.region`
   - 注意：`凭据.region` 不参与 API Region 回退链。参考 `src/kiro/model/credentials.rs:204-217`

5. **Token 自动刷新和回写:** 无需手动操作：
   - Token 过期（提前 5 分钟）或即将过期（10 分钟内）时自动刷新
   - 刷新后新的 `accessToken`、`refreshToken`、`expiresAt` 自动回写到凭据文件（仅多凭据数组格式）
   - 回写使用 `block_in_place` 避免阻塞 Tokio worker
   - 单凭据格式不回写。参考 `src/kiro/token_manager.rs:957-998`

6. **验证配置:** 启动时会自动验证：
   - `refreshToken` 非空且长度 >= 100，不含 `...`（防止 Kiro IDE 截断的 token）
   - 重复凭据 ID 检测（SHA-256 哈希去重）
   - 缺少 ID 的凭据自动分配递增 ID 并回写
   - 运行 `cargo test` 验证凭据模型的序列化/反序列化正确性
