# deploy-tencent-project

把本地已完成的项目通过 Git 推送、SSH、Docker Compose、健康检查、SSH 隧道和浏览器/API 测试部署到远程云主机。每个项目完全隔离——独立的部署目录、Compose 项目名、端口、容器、网络和卷，互不干扰。

## 为什么需要

往一台服务器上部署多个项目，不做严格隔离很容易出事。这个 skill 强制执行按项目隔离、清洁构建门禁、溯源验证（本地 SHA = 裸仓 ref = 检出 SHA = 运行镜像）和三层测试策略（本地深度 → 隧道中等 → 公网轻量），确保部署可重复、可验证。

## 安装

### 方式 A — 交给 agent 安装

把仓库地址给你的 agent，让它安装：

```
https://github.com/sabrina-fan/deploy-tencent-project
```

### 方式 B — 手动安装

把 `deploy-tencent-project/` 目录复制到你的 agent skill 目录下。

## 配置

- **SSH 别名**：在 `~/.ssh/config` 里配置一个 SSH 别名（如 `tencent-dev`）指向你的服务器，永远不硬编码 IP。
- **SSH 用户主目录**：裸仓存放在 `~/git/<project>.git`，部署检出到 `~/projects/<project>`，均在 SSH 用户主目录下。
- **目标平台**：默认 `linux/amd64`。
- **项目配置**：在项目的 `AGENTS.md` 里添加部署配置段，写明项目专属端口、compose 文件、健康路径等，模板见 [references/project-config.md](references/project-config.md)。
- **密钥不入 Git**：`.env` 文件、token、密码和数据库快照默认放 Git 外面（权限 `0600`），除非明确授权做自包含私有部署。

## 使用方法

需要在服务器上部署、重新部署、检查或测试项目时触发。skill 会：

1. **预检** — 检查 Git 状态、SSH 连通性、远端资源、端口冲突。
2. **解析配置** — 从 `package.json` 和 `AGENTS.md` 检测项目名、分支、compose 文件、端口。
3. **初始化或更新** — 首次部署创建裸仓和检出；后续部署用 fast-forward 更新。
4. **构建并启动** — 在服务器端用隔离的项目名和端口做 Docker Compose 构建。
5. **验证版本** — 核对 SHA 链：本地 HEAD → 裸仓 ref → 检出 → 运行镜像。
6. **测试** — 三层验证：本地深度、隧道中等、公网轻量。
7. **报告** — 部署 SHA、数据库策略、端口映射、测试结果、隧道命令。

## 兼容性

- **macOS** — 主要开发平台（使用 SSH、Docker、本地浏览器测试）。
- **Linux** — 作为开发和部署平台都完全支持。
- **Windows** — 通过 WSL2 或 Docker Desktop 可用；SSH 隧道命令可能需适配。

## 安全与边界

这个 skill 只部署和测试；不写源码。绝不打印密钥、token、密码或 `.env` 内容。数据库迁移需要明确授权。破坏性操作（删卷、删库、Git force-push）需要单独的明确授权。SSH 凭据绝不在对话中请求或处理。
