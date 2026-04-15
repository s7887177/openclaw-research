# Level 5：操作者工作流層（Operator Workflows）

> **版本快照**：OpenClaw v2026.4.12  
> **層級定位**：從操作者（Operator）的角度，整理安裝、部署、測試、監控、自動化等實務工作流。  
> **7 Topics · 21 份文件 · ~151,000 字元**

---

## 本層概述

操作者工作流層面向實際操作。如果 Level 1-4 回答的是「OpenClaw 是什麼、如何運作」，本層回答的是「我要怎麼用它」：

- 如何安裝與初始化？
- 如何部署到生產環境？
- 如何開發自訂 Skill 和 Provider？
- 如何設定自動化與排程？
- 如何監控與除錯？
- 如何建立測試與 CI 流程？

---

## Topic 索引

| # | Topic | 路徑 | 核心問題 | 檔案數 |
|---|-------|------|---------|--------|
| 1 | [安裝與設定](./installation-setup/) | `installation-setup/` | 系統需求、安裝流程、Onboarding 與 Doctor | 2 |
| 2 | [部署與遠端存取](./deployment-remote/) | `deployment-remote/` | 部署模式概覽、遠端存取與安全設定 | 2 |
| 3 | [自訂 Skill 開發](./custom-skill-dev/) | `custom-skill-dev/` | Skill 架構與格式、開發與發佈流程 | 2 |
| 4 | [自訂 Provider 開發](./custom-provider-dev/) | `custom-provider-dev/` | Provider 架構與介面、實作與參考 | 2 |
| 5 | [自動化與 Webhooks](./automation-webhooks/) | `automation-webhooks/` | Hooks 事件系統、Cron 排程系統 | 2 |
| 6 | [監控與維運](./monitoring-ops/) | `monitoring-ops/` | 健康檢查與日誌、診斷與可觀測性 | 2 |
| 7 | [測試與驗證](./testing-verification/) | `testing-verification/` | 測試框架與工具、QA 與 CI 流程 | 2 |

---

## 閱讀順序建議

### 快速上手路線

1. **安裝與設定** → 把 OpenClaw 跑起來
2. **部署與遠端存取** → 讓它可以從外部存取

### 開發路線

3. **自訂 Skill 開發** → 擴充 Agent 能力
4. **自訂 Provider 開發** → 接入新的 LLM 提供者
5. **自動化與 Webhooks** → 設定觸發器與排程

### 維運路線

6. **監控與維運** → 日誌、健康檢查、診斷
7. **測試與驗證** → 確保系統穩定

---

## 與其他層級的關係

- **↓ Level 2（元件機制層）**：本層的操作流程建立在 Level 2 的元件知識之上
- **↓ Level 4（系統行為層）**：本層的操作實務以 Level 4 的架構理解為前提
- **↑ Level 6（概念綜合層）**：本層的操作經驗為 Level 6 的應用藍圖提供素材
