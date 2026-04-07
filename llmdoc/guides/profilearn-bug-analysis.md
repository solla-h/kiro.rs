# profileArn 缺失导致 400 错误的排查与分析

## 1. 问题描述

- **现象：** kiro-claw 的 JudgeDirect 调用 kiro-rs 时持续报 502 错误
- **错误信息：** `上游 API 调用失败: 非流式 API 请求失败: 400 Bad Request {"message":"profileArn is required for this request.","reason":null}`
- **影响范围：** 所有通过 kiro-rs 的普通 API 请求（流式和非流式）都受影响，MCP 请求不受影响
- **发现时间：** 2026-04-04 起开始出现，推测上游 Kiro API 在此时间点开始强制要求 profileArn

## 2. 排查过程

### 2.1 日志分析

- **kiro-claw 侧：** JudgeDirect 持续报 `ClaudeCode HTTP 502`，每条消息的安全审查都失败后 fallback 到 SAFE
- **kiro-rs 侧：** `handlers.rs` 报 `Kiro API 调用失败: 400 Bad Request {"message":"profileArn is required for this request."}`
- **MCP 请求：** websearch 也报同样错误，但走的是不同的代码路径

### 2.2 凭据检查

- 检查 `credentials.json`：所有 10 个凭据的 profileArn 字段均为空（MISSING）
- profileArn 不是手动配置的，它来自 token 刷新响应，刷新成功后由 `persist_credentials` 回写到 `credentials.json`

### 2.3 代码追踪

profileArn 在系统中存在两条独立的传递路径：

**路径 A — JSON body 中的 profileArn（用于 generateAssistantResponse）— 有 bug：**

```
main.rs:108  first_credentials.profile_arn.clone()  → None（凭据中没有）
  → router.rs:46  state.with_profile_arn(arn)       → 因为是 None，不调用
    → AppState.profile_arn = None                    → 全局固定为 None
      → handlers.rs:247/730  KiroRequest { profile_arn: None }
        → serde skip_serializing_if = "Option::is_none"  → JSON 中没有 profileArn
          → 上游 Kiro API: "profileArn is required" → 400
```

**路径 B — HTTP header 中的 profileArn（用于 MCP 请求）— 正确实现：**

```
provider.rs:232  ctx.credentials.profile_arn
  → header "x-amzn-kiro-profile-arn"
  → 每次请求动态从当前凭据读取
```

## 3. 根因

两个问题叠加：

1. **设计缺陷：** `AppState.profile_arn` 在服务启动时从第一个凭据一次性取值（`src/main.rs:108`），之后永远不变。即使 token 刷新后凭据文件里有了 profileArn，AppState 也不会更新。在 balanced 模式下切换凭据时，request body 中的 profileArn 始终是启动时第一个凭据的值，不会跟随当前凭据切换。

2. **上游策略变更：** 上游 Kiro API 在 2026-04-04 前后开始强制要求 profileArn 字段，之前是可选的。

**对比：** MCP 请求在 `src/kiro/provider.rs:232` 正确地从 `ctx.credentials.profile_arn` 动态读取并通过 HTTP header 传递，不受此设计缺陷影响。但由于所有凭据的 profileArn 都为空，MCP 请求实际上也会失败。

## 4. 影响分析

- kiro-claw 的 JudgeDirect（安全审查）每次都失败，fallback 到 SAFE 继续执行，增加了每条消息的处理延迟
- 直接通过 kiro-rs 的非流式 API 请求全部失败（返回 502）
- 流式 API 请求也受影响（同样缺少 profileArn）
- 间接加剧了 kiro-claw 的消息排队问题

## 5. GitHub 社区情况

- **Issue #125**（2026-04-04，kissman911）— "请求缺少profileArn引起的报错"，有社区贡献者提交了 PR 修复
- **Issue #129**（2026-04-05，siyuan-123）— "非流式 API 请求失败: 此请求需要 profileArn"
- 作者 hank9999 截至 2026-04-05 未回复这两个 issue，无官方修复计划
- 社区提供的临时方案：从 Kiro 客户端的 profile 文件中手动获取 profileArn
  - Linux: `~/.config/Kiro/User/globalStorage/kiro.kiroagent/profile.json`
  - Windows: `$env:APPDATA\Kiro\User\globalStorage\kiro.kiroagent\profile.json`

## 6. 版本历史

- **2026.3.1**（2026-03-30）— 添加了 MCP 请求的 profile ARN header 支持（`src/kiro/provider.rs:232`），但遗漏了普通 API 请求路径
- 2026-04-04 起上游开始强制要求 profileArn，暴露了这个遗漏

## 7. 修复方案（待作者发布）

正确的修复方向：将 profileArn 的注入从 handlers 层下沉到 provider 层，让每次请求都从当前凭据动态获取 profileArn。

涉及修改的文件：

- `src/main.rs:108` — 不再传 profile_arn 给 router
- `src/anthropic/router.rs:40-47` — 移除 profile_arn 参数
- `src/anthropic/middleware.rs:27` — 从 AppState 移除 profile_arn 字段
- `src/anthropic/handlers.rs:247,730` — KiroRequest 不再从 state 取 profile_arn
- `src/kiro/provider.rs` — 在 `call_api_with_retry` 中像 MCP 请求一样动态注入 `x-amzn-kiro-profile-arn` header

临时止血方案：

1. 手动从 Kiro 客户端获取 profileArn 写入 `credentials.json`，然后重启 kiro-rs
2. 或者自行修改代码，在 `provider.rs` 的 `call_api_with_retry` 中添加 header 注入逻辑

## 8. 当前决策

等待作者 hank9999 发布官方修复版本。如果长时间未修复，考虑自行 fork 修改。
