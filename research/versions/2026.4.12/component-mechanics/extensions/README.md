# Extensions（擴充套件）研究索引

> **層級**：Level 2 — Component Mechanics  
> **版本**：OpenClaw v2026.4.12  
> **範圍**：111 個擴充套件的完整架構分析

## 目錄

| 檔案 | 內容 | 字數估計 |
|------|------|----------|
| [01-extensions-architecture.md](./01-extensions-architecture.md) | 目錄結構、manifest 格式、Plugin SDK 邊界 | ~8,000 |
| [02-extensions-catalog.md](./02-extensions-catalog.md) | 111 個擴充套件分類總覽（含完整表格） | ~10,000 |
| [03-extensions-deep-dives.md](./03-extensions-deep-dives.md) | 重點擴充套件深入分析 | ~9,000 |

## 摘要

OpenClaw 的擴充套件系統（Extensions）是整個平台的核心可擴展機制。透過 `openclaw.plugin.json` 清單檔和 `openclaw/plugin-sdk/*` 模組介面，111 個擴充套件涵蓋了 LLM 供應商、聊天平台通道、語音/語音、媒體生成、網路搜尋、記憶系統、工具/公用程式、閘道代理等八大類別。每個擴充套件都是獨立的 npm 套件，透過 `definePluginEntry()` 或 `defineBundledChannelEntry()` 向核心註冊能力。
