# kiro-rs 文档索引

**kiro-rs** — 一个用 Rust 编写的 Anthropic Claude API 兼容代理服务，支持多凭据管理、流式响应和可选的 Admin 管理界面。

---

## Overview（项目概览）

- [project-overview.md](overview/project-overview.md) — 项目定位、核心能力与整体架构概述

## Architecture（系统架构）

- [kiro-client-architecture.md](architecture/kiro-client-architecture.md) — Kiro API 客户端核心：Provider 请求层与 MultiTokenManager 多凭据管理
- [anthropic-api-architecture.md](architecture/anthropic-api-architecture.md) — Anthropic Messages API 协议到 Kiro 内部协议的兼容转换层
- [event-stream-parser-architecture.md](architecture/event-stream-parser-architecture.md) — AWS Event Stream 二进制帧协议的流式解析器
- [admin-system-architecture.md](architecture/admin-system-architecture.md) — Admin 管理子系统：凭据管理 REST API 与内嵌 SPA 界面

## Guides（操作指南）

- [how-to-add-api-endpoint.md](guides/how-to-add-api-endpoint.md) — 如何添加新的 API 端点和修改协议转换
- [how-to-manage-credentials.md](guides/how-to-manage-credentials.md) — 如何配置和管理凭据、认证方式与 Token 刷新
- [how-to-use-admin.md](guides/how-to-use-admin.md) — 如何启用和使用 Admin 管理系统
- [how-to-deploy-service.md](guides/how-to-deploy-service.md) — 如何在 Linux 服务器上编译、配置并以 systemd 服务方式部署 kiro-rs

## Reference（参考资料）

- [anthropic-api-types.md](reference/anthropic-api-types.md) — Anthropic API 兼容层的请求/响应/错误类型定义
- [kiro-data-models.md](reference/kiro-data-models.md) — Kiro 内部数据模型：请求模型、事件模型、凭据模型
- [event-stream-protocol.md](reference/event-stream-protocol.md) — AWS Event Stream 二进制帧协议规范
- [coding-conventions.md](reference/coding-conventions.md) — 项目编码规范与命名约定
- [git-conventions.md](reference/git-conventions.md) — Git 提交规范、版本号管理与 CI/CD 流程
