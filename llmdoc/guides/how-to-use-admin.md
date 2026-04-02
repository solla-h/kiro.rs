# 如何使用 Admin 管理系统

## 前置条件

Admin 功能默认关闭，需在配置中启用。配置文件位置参考 `src/model/config.rs` (`admin_api_key` 字段)。

## 1. 启用 Admin

在 `config.json` 中设置 `adminApiKey` 字段为一个非空字符串：

```json
{
  "adminApiKey": "your-secret-admin-key"
}
```

启动后日志会输出 `Admin API 已启用` 和 `Admin UI 已启用: /admin`。若 key 为空或未设置，Admin 功能不会加载（`src/main.rs:111-136`）。

## 2. 访问 Admin UI

浏览器打开 `http://<host>:8990/admin`，输入配置的 `adminApiKey` 登录。登录态保存在浏览器 `localStorage`（key: `adminApiKey`），刷新页面无需重新登录。

## 3. 凭据管理操作

通过 Dashboard 页面（`admin-ui/src/components/dashboard.tsx`）可执行以下操作：

- **查看凭据列表:** 自动加载并每 30 秒轮询刷新，展示 email/ID、禁用状态、优先级、失败次数、订阅等级、剩余用量等。
- **添加凭据:** 点击"添加凭据"按钮，填写 `refresh_token`（必填），可选 `name`（自定义名称，便于辨识）、auth_method（`social`/`idc`）、priority、region、email、proxy 等。idc 方式需额外填写 `clientId`/`clientSecret`。添加后自动获取订阅等级。凭据卡片标题优先显示 `name`，未设置时回退显示 email 或 ID。
- **删除凭据:** 必须先禁用凭据，然后才能删除（UI 和 API 双重校验）。
- **禁用/启用:** 通过凭据卡片上的开关切换。禁用当前活跃凭据时自动切换到下一个可用凭据。
- **调整优先级:** 在凭据卡片上内联编辑优先级数值（数字越小优先级越高），或使用快捷按钮提高/降低。
- **重置失败计数:** 点击"重置"按钮，清零失败计数并重新启用凭据。
- **强制刷新 Token:** 点击"刷新 Token"按钮，强制重新获取 access token。

## 4. 批量导入

点击"批量导入"按钮（`admin-ui/src/components/batch-import-dialog.tsx`），输入 JSON 数组格式的凭据数据。导入流程：

1. 逐个添加凭据并调用余额接口验活。
2. 通过 SHA-256 哈希比对 `refreshTokenHash` 自动检测重复，跳过已存在的凭据。
3. 验活失败的凭据自动回滚（先禁用再删除）。

Dashboard 还支持批量验活（每次间隔 2 秒防封号）、批量刷新 Token、批量恢复异常、批量删除、清除已禁用等操作。

## 5. 余额查询

在凭据卡片上点击"查看余额"，展示订阅等级、当前用量、用量上限、剩余额度、使用百分比、下次重置时间。余额数据有 5 分钟缓存（内存 + 磁盘双层），缓存期内重复查询直接返回缓存结果。

## 6. 负载均衡模式

Dashboard 顶部可切换负载均衡模式：
- `priority`: 按优先级顺序使用凭据（默认）。
- `balanced`: 均衡分配请求到各凭据。

## 7. API 直接调用

所有操作也可通过 REST API 完成，端点前缀 `/api/admin`，认证方式二选一：
- Header: `x-api-key: <your-admin-key>`
- Header: `Authorization: Bearer <your-admin-key>`

完整端点列表参考 `/llmdoc/architecture/admin-system-architecture.md` 的 API 端点速查表。
