# Topic 2: 部署策略與遠端存取（Deployment & Remote Access）

本章涵蓋 OpenClaw 的多種部署模式與遠端存取設定。

## 章節索引

| 檔案 | 內容 |
|------|------|
| [01-部署模式概覽.md](./01-部署模式概覽.md) | 本地模式、Daemon 模式、Docker/Podman 模式、雲端部署 |
| [02-遠端存取與安全.md](./02-遠端存取與安全.md) | Gateway 認證、Tailscale、裝置配對、多裝置共用 |

## 摘要

OpenClaw Gateway 支援多種部署拓撲：本地直接執行、Daemon 背景服務、Docker/Podman 容器、以及 Fly.io/Render 等雲端平台。遠端存取可透過 Tailscale tunnel、SSH tunnel 或直接暴露 Gateway 埠。安全性由 Token/Password 認證、Device Pairing 與角色控制共同保障。
