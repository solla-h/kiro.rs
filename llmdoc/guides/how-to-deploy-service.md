# 如何在 Linux 服务器上部署 kiro-rs

在 Linux 服务器（以 Amazon Linux 2 为例）上从源码编译、配置并以 systemd 服务方式运行 kiro-rs。

1. **安装编译依赖**
   - 安装 Rust 工具链（`rustup`）、Node.js、`pnpm`。
   - 安装 OpenSSL 开发库。Amazon Linux 2 上为 `openssl11-devel`：
     ```bash
     sudo yum install -y openssl11-devel
     ```
   - 若编译时报 OpenSSL 找不到，设置环境变量：`export OPENSSL_DIR=/usr/local/ssl`。

2. **构建前端 Admin UI**
   ```bash
   cd admin-ui && pnpm install && pnpm build
   ```
   构建产物 `admin-ui/dist/` 会在编译时通过 `rust-embed` 嵌入二进制。

3. **编译 Rust 二进制**
   ```bash
   OPENSSL_DIR=/usr/local/ssl cargo build --release
   ```
   产物路径：`target/release/kiro-rs`。

4. **准备配置文件**
   - 在工作目录下创建 `config.json` 和 `credentials.json`。
   - 配置格式详见 `/llmdoc/guides/how-to-manage-credentials.md` 和 `README.md`。
   - `config.json` 必填字段：`host`、`port`、`apiKey`、`region`。
   - `credentials.json` 支持单对象或数组格式，详见 `README.md` (#credentials.json)。

5. **配置 systemd 服务**
   - 项目根目录已提供模板：`kiro-rs.service`。
   - 关键配置项：`WorkingDirectory` 指向项目目录，`User=ec2-user`，`Restart=always`，`RestartSec=3`，`Environment=RUST_LOG=info`。
   - 安装并启动：
     ```bash
     sudo cp kiro-rs.service /etc/systemd/system/
     sudo systemctl daemon-reload
     sudo systemctl enable kiro-rs
     sudo systemctl start kiro-rs
     ```

6. **验证服务运行**
   - 查看状态：`sudo systemctl status kiro-rs`
   - 查看实时日志：`sudo journalctl -u kiro-rs -f`
   - 发送测试请求验证 API 可用性（参考 `README.md` 中的 curl 示例）。

7. **常用运维命令**
   - 重启服务：`sudo systemctl restart kiro-rs`
   - 停止服务：`sudo systemctl stop kiro-rs`
   - 重新编译后需执行 `sudo systemctl restart kiro-rs` 加载新二进制。

## 常见问题

| 症状 | 原因 | 解决方法 |
|------|------|----------|
| 启动失败，端口被占用 | 旧进程未停止 | `sudo lsof -i :<端口号>` 找到 PID 后 `kill` |
| 编译报 OpenSSL 错误 | 缺少开发库或路径未设置 | 安装 `openssl11-devel` 并设置 `OPENSSL_DIR` |
| exit code 101 | 端口冲突或配置文件格式错误 | 检查端口占用，验证 JSON 格式 |
| TLS/代理请求失败 | rustls 兼容性问题 | 在 `config.json` 中设置 `"tlsBackend": "native-tls"` |

## 源文件参考

- `kiro-rs.service` — systemd 服务模板文件
- `config.example.json` — 配置文件示例
- `README.md` (#开始) — 完整的编译与配置说明
- `/llmdoc/guides/how-to-manage-credentials.md` — 凭据配置详解
