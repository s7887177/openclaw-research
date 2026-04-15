# Level 2：元件機制層（Component Mechanics）

> **版本快照**：OpenClaw v2026.4.12  
> **層級定位**：深入研究 OpenClaw 的 15 個核心元件，逐一拆解其內部架構與實作邏輯。  
> **15 Topics · 46 份文件 · ~858,000 字元**

---

## 本層概述

元件機制層是整份研究中最大、最詳細的一層。OpenClaw 的功能由 15 個核心元件組成，每個元件都有自己的目錄結構、配置格式、API 介面與運行邏輯。在這一層，我們對每個元件進行「解剖式」的分析，記錄其內部如何運作。

這一層的研究風格是「由內而外」——先看程式碼結構，再看配置 Schema，然後追蹤執行流程。

---

## Topic 索引

### 核心執行引擎

| # | Topic | 路徑 | 核心問題 | 檔案數 |
|---|-------|------|---------|--------|
| 1 | [Agents 系統](./agents/) | `agents/` | Agent 如何定義、配置、協調多個 Agent？ | 5 |
| 2 | [Auto-Reply 引擎](./auto-reply/) | `auto-reply/` | 訊息如何被派發並生成回覆？ | 2 |
| 3 | [Gateway 閘道](./gateway/) | `gateway/` | HTTP/WebSocket 閘道如何管理 Session 與工具審批？ | 7 |

### 擴充與整合

| # | Topic | 路徑 | 核心問題 | 檔案數 |
|---|-------|------|---------|--------|
| 4 | [Channels 通道系統](./channels/) | `channels/` | 25+ 通訊平台如何統一接入？ | 4 |
| 5 | [Extensions 擴充系統](./extensions/) | `extensions/` | 擴充如何架構、發現、載入？ | 3 |
| 6 | [Skills 技能系統](./skills/) | `skills/` | Skill 如何定義、載入、執行？ | 3 |
| 7 | [Plugin 系統](./plugins/) | `plugins/` | Plugin 架構與生命週期 | 1 |
| 8 | [MCP / Hooks / Flows](./mcp-hooks-flows/) | `mcp-hooks-flows/` | MCP 整合、事件驅動、UI 貢獻 | 3 |

### 基礎設施

| # | Topic | 路徑 | 核心問題 | 檔案數 |
|---|-------|------|---------|--------|
| 9 | [配置系統](./config/) | `config/` | 配置如何載入、驗證、合併？ | 1 |
| 10 | [Memory 記憶系統](./memory/) | `memory/` | 記憶如何存儲與檢索？ | 1 |
| 11 | [Security 安全系統](./security/) | `security/` | 安全模型與存取控制如何運作？ | 1 |
| 12 | [Infra & Daemon](./infra-daemon/) | `infra-daemon/` | 基礎設施工具庫與跨平台服務管理 | 2 |

### 使用者介面

| # | Topic | 路徑 | 核心問題 | 檔案數 |
|---|-------|------|---------|--------|
| 13 | [CLI 命令系統](./cli/) | `cli/` | CLI 指令如何架構與實作？ | 1 |
| 14 | [UI & Apps](./ui-apps/) | `ui-apps/` | 控制面板 UI 與原生 App 如何運作？ | 2 |
| 15 | [Voice & Media](./voice-media/) | `voice-media/` | 語音與媒體處理系統 | 1 |

---

## 閱讀順序建議

1. **Agents** → OpenClaw 的核心概念，理解 Agent 是什麼
2. **Auto-Reply** → 理解訊息處理的主要流程
3. **Gateway** → 了解外部存取如何進入系統
4. **Channels** → 了解平台整合機制
5. 其餘元件按興趣或需求選讀

---

## 與其他層級的關係

- **↓ Level 1（倉庫證據層）**：Level 1 的模組邊界定義了本層的元件拆分
- **↑ Level 3（橫切機制層）**：本層的元件如何協作，由 Level 3 的橫切模式描述
- **↑ Level 4（系統行為層）**：本層的元件組合在一起，形成 Level 4 描述的系統行為
