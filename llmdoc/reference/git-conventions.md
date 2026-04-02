# Git 规范

## 1. 核心摘要

本项目遵循 Conventional Commits 规范，提交消息使用 `type(scope): 中文描述` 格式。版本号采用日期格式 `YYYY.M.D`，通过 `bump: vYYYY.M.D` 提交管理。CI/CD 基于 GitHub Actions，包含多平台二进制构建和多架构 Docker 镜像两条发布路径。

## 2. 提交消息格式

格式：`type(scope): 中文描述`

常用 type：

| type    | 用途           | 频率   |
| ------- | -------------- | ------ |
| `fix`   | 缺陷修复       | ~66%   |
| `feat`  | 新功能         | ~10%   |
| `chore` | 杂项维护       | ~14%   |
| `bump`  | 版本号提升     | 偶发   |
| `docs`  | 文档更新       | 偶发   |

scope 示例：`token_manager`, `provider`, `anthropic`, `admin-ui`, `websearch`, `handlers`, `converter`, `docker`, `workflow`, `model`, `config`, `proxy`。

特殊约定：
- 描述语言：简体中文
- PR 合并提交附带 `(#NNN)` 编号后缀
- 版本提升：`bump: vYYYY.M.D`
- 跳过 CI：在消息末尾添加 `[skip ci]`

## 3. 版本号格式

采用日期版本：`YYYY.M.D`（如 `2026.2.7`）。

- `Cargo.toml` 中的 `version` 字段为权威来源
- Git tag 格式：`v2026.2.7`（带 `v` 前缀）
- 版本提升通过 `bump: vYYYY.M.D` 提交完成

## 4. 分支策略

- `master`：主分支，push 触发 beta 构建（版本号 `beta-{short_sha}`）
- `v*` tag：release 构建，tag 名即版本号
- 功能分支：从 master 分出，通过 PR 合并回 master
- `pre-check` job 确保同一 commit 不会同时触发 beta 和 release 构建

## 5. CI/CD 概述

两个 GitHub Actions workflow，触发条件相同：push 到 master、`v*` tag、或手动触发。

### build.yaml — 多平台二进制构建

构建矩阵（5 平台）：macOS-arm64, macOS-x64, Windows-x64, Linux-x64, Linux-arm64。

流程：构建 admin-ui (Node 20 + pnpm 9) → `cargo build --release` → 上传 artifact。

### docker-build.yaml — Docker 镜像构建

构建矩阵：linux/amd64, linux/arm64。推送到 `ghcr.io/{owner}/kiro-rs:{version}-{arch}`。

`manifest` job 合并为多架构镜像，别名 tag：release → `latest`，master → `beta`。

### 版本决定逻辑

| 触发方式           | 版本号                |
| ------------------ | --------------------- |
| `workflow_dispatch` | 手动输入值            |
| `v*` tag push      | tag 名称              |
| master push        | `beta-{short_sha}`   |

## 6. 信息来源

- **提交规范**：`git log` — 从历史提交中归纳的实际规范。
- **版本号**：`Cargo.toml:3` — `version` 字段为权威来源。
- **CI 构建**：`.github/workflows/build.yaml` — 多平台二进制构建配置。
- **Docker 构建**：`.github/workflows/docker-build.yaml` — 多架构 Docker 镜像构建配置。
- **忽略规则**：`.gitignore` — 版本控制排除规则。
