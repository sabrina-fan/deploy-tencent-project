# deploy-tencent-project

把当前项目部署/测试到腾讯云主机的 AI agent skill。

用 Git 推代码、SSH、Docker Compose、健康检查、SSH 隧道、浏览器/API 测试。每项目按仓库隔离——独立部署目录、Compose 项目名、端口、容器、网络、卷，互不干扰。用 SSH alias 管理主机，不硬编码 IP。

适用于任何需要部署到远程主机（Docker Compose 架构）的项目，不限于腾讯云。

## 安装

把 `SKILL.md`（及 `agents/`、`references/` 子目录如有）复制到你的 agent skill 目录下即可。

## License

MIT
