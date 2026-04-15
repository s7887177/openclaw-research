# MCP 協議概述

## 目錄

- [1. MCP 是什麼](#1-mcp-是什麼)
  - [1.1 MCP 的誕生背景](#11-mcp-的誕生背景)
  - [1.2 MCP 的核心理念](#12-mcp-的核心理念)
  - [1.3 MCP 解決了什麼問題](#13-mcp-解決了什麼問題)
- [2. MCP 協議架構](#2-mcp-協議架構)
  - [2.1 Client-Server 模型](#21-client-server-模型)
  - [2.2 JSON-RPC 2.0 基礎](#22-json-rpc-20-基礎)
  - [2.3 協議生命週期](#23-協議生命週期)
- [3. MCP 核心能力](#3-mcp-核心能力)
  - [3.1 Tools（工具）](#31-tools工具)
  - [3.2 Resources（資源）](#32-resources資源)
  - [3.3 Prompts（提示模板）](#33-prompts提示模板)
  - [3.4 Sampling（取樣）](#34-sampling取樣)
- [4. Tool Discovery 工具發現](#4-tool-discovery-工具發現)
  - [4.1 工具列表查詢](#41-工具列表查詢)
  - [4.2 工具描述格式](#42-工具描述格式)
  - [4.3 動態工具發現](#43-動態工具發現)
- [5. Transport 傳輸層](#5-transport-傳輸層)
  - [5.1 stdio 傳輸](#51-stdio-傳輸)
  - [5.2 SSE 傳輸（HTTP）](#52-sse-傳輸http)
  - [5.3 Streamable HTTP 傳輸](#53-streamable-http-傳輸)
  - [5.4 傳輸層比較](#54-傳輸層比較)
- [6. MCP 訊息格式](#6-mcp-訊息格式)
  - [6.1 初始化握手](#61-初始化握手)
  - [6.2 工具呼叫](#62-工具呼叫)
  - [6.3 資源存取](#63-資源存取)
  - [6.4 錯誤處理](#64-錯誤處理)
- [7. MCP 生態系統](#7-mcp-生態系統)
  - [7.1 主要 MCP Client](#71-主要-mcp-client)
  - [7.2 MCP Server 範例](#72-mcp-server-範例)
  - [7.3 OpenClaw 在 MCP 生態中的定位](#73-openclaw-在-mcp-生態中的定位)
- [8. MCP 與其他方案的比較](#8-mcp-與其他方案的比較)
  - [8.1 MCP vs Function Calling](#81-mcp-vs-function-calling)
  - [8.2 MCP vs LangChain Tools](#82-mcp-vs-langchain-tools)
  - [8.3 MCP vs OpenAI Plugins](#83-mcp-vs-openai-plugins)
- [9. 安全考量](#9-安全考量)
  - [9.1 權限控制](#91-權限控制)
  - [9.2 資料隔離](#92-資料隔離)
  - [9.3 信任邊界](#93-信任邊界)
- [10. 未來展望](#10-未來展望)

---

## 1. MCP 是什麼

### 1.1 MCP 的誕生背景

MCP（Model Context Protocol）是由 Anthropic 在 2024 年底發布的開放協議，旨在標準化 AI 模型與外部工具和資料來源之間的互動方式。在 MCP 之前，每個 AI 應用都需要自行實作與外部系統的整合，導致大量重複工作和不一致的介面。

### 1.2 MCP 的核心理念

MCP 的設計理念可以用一個比喻來理解：

```
USB 之於硬體 = MCP 之於 AI 工具

在 USB 出現之前：
  每個外設都有自己的接口 → 鍵盤用 PS/2、印表機用 LPT、搖桿用 Game Port

USB 出現之後：
  統一接口 → 所有外設都用 USB → 即插即用

在 MCP 出現之前：
  每個 AI 工具都有自己的整合方式 → Claude 用 Tool Use、GPT 用 Function Calling

MCP 出現之後：
  統一協議 → 任何 MCP Server 可被任何 MCP Client 使用 → 即插即用
```

### 1.3 MCP 解決了什麼問題

**問題 1：碎片化的工具整合**

```
沒有 MCP：
  Claude Desktop → 自己寫 Tool Use 整合 GitHub
  Cursor → 自己寫 Function Call 整合 GitHub
  OpenClaw → 自己寫 Skill 整合 GitHub
  每個 Client 都要重複寫一遍

有了 MCP：
  GitHub MCP Server（寫一次）
    ├── Claude Desktop（直接使用）
    ├── Cursor（直接使用）
    └── OpenClaw（直接使用）
```

**問題 2：缺乏標準的工具描述**

MCP 提供了統一的工具描述格式，讓 AI 能自動理解工具的功能和用法。

**問題 3：安全性和權限控制**

MCP 在協議層面內建了權限控制和人機互動確認機制。

---

## 2. MCP 協議架構

### 2.1 Client-Server 模型

MCP 使用經典的 Client-Server 架構：

```
┌──────────────┐         ┌──────────────┐
│  MCP Client  │◄───────▶│  MCP Server  │
│              │  JSON-   │              │
│  AI 應用     │  RPC 2.0 │  工具/資源    │
│              │         │  提供者       │
│  例如：      │         │              │
│  Claude      │         │  例如：      │
│  Cursor      │         │  GitHub      │
│  OpenClaw    │         │  Filesystem  │
│              │         │  Database    │
└──────────────┘         └──────────────┘
      Host                   Server
```

- **Host**：宿主應用（如 Claude Desktop、Cursor）
- **Client**：MCP 客戶端，由 Host 管理
- **Server**：MCP 伺服器，提供工具和資源

一個 Host 可以連接多個 Server：

```
┌──────────────┐
│   OpenClaw   │
│   (Host)     │
│              │
│  ┌────────┐  │     ┌──────────────┐
│  │Client 1│──┼────▶│ GitHub Server │
│  └────────┘  │     └──────────────┘
│  ┌────────┐  │     ┌──────────────┐
│  │Client 2│──┼────▶│ FS Server    │
│  └────────┘  │     └──────────────┘
│  ┌────────┐  │     ┌──────────────┐
│  │Client 3│──┼────▶│ DB Server    │
│  └────────┘  │     └──────────────┘
└──────────────┘
```

### 2.2 JSON-RPC 2.0 基礎

MCP 基於 JSON-RPC 2.0 協議通訊。所有訊息都是 JSON 格式：

```json
// 請求（Request）
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "Taipei"
    }
  }
}

// 回應（Response）
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "台北市目前溫度 28°C，多雲時晴。"
      }
    ]
  }
}

// 通知（Notification，無需回應）
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

### 2.3 協議生命週期

```
1. 初始化（Initialize）
   Client → Server: initialize（交換能力）
   Server → Client: initialize response
   Client → Server: initialized（確認）

2. 正常運作（Operating）
   Client → Server: tools/list（查詢可用工具）
   Client → Server: tools/call（呼叫工具）
   Server → Client: 結果
   ...

3. 關閉（Shutdown）
   Client or Server: 關閉連接
```

---

## 3. MCP 核心能力

### 3.1 Tools（工具）

Tools 是 MCP 最核心的能力 — 讓 AI 可以「做事」：

```json
{
  "name": "create_issue",
  "description": "在 GitHub 上建立一個新的 Issue",
  "inputSchema": {
    "type": "object",
    "properties": {
      "repo": {
        "type": "string",
        "description": "倉庫名稱（owner/repo）"
      },
      "title": {
        "type": "string",
        "description": "Issue 標題"
      },
      "body": {
        "type": "string",
        "description": "Issue 內容"
      }
    },
    "required": ["repo", "title"]
  }
}
```

### 3.2 Resources（資源）

Resources 讓 AI 可以「讀取」資料：

```json
{
  "uri": "file:///home/user/project/README.md",
  "name": "Project README",
  "mimeType": "text/markdown",
  "description": "專案的 README 文件"
}
```

Resources 是唯讀的，與 Tools 的區別在於：
- **Tools**：執行操作（可能有副作用）
- **Resources**：讀取資料（唯讀，無副作用）

### 3.3 Prompts（提示模板）

Prompts 讓 Server 提供預定義的提示模板：

```json
{
  "name": "code_review",
  "description": "生成程式碼審查的提示",
  "arguments": [
    {
      "name": "code",
      "description": "要審查的程式碼",
      "required": true
    },
    {
      "name": "language",
      "description": "程式語言",
      "required": false
    }
  ]
}
```

### 3.4 Sampling（取樣）

Sampling 允許 Server 反向請求 Client 的 LLM 能力：

```
正常流程：Client 使用 Server 的 Tools
Sampling：Server 請求 Client 的 LLM 幫忙

例如：
  Database MCP Server 收到複雜查詢
  → 不知道如何生成 SQL
  → 請求 Client 的 LLM 幫忙生成 SQL
  → 拿到 SQL 後執行查詢
  → 回傳結果
```

---

## 4. Tool Discovery 工具發現

### 4.1 工具列表查詢

```json
// Client 查詢可用工具
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}

// Server 回傳工具列表
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "description": "查詢指定城市的天氣",
        "inputSchema": { ... }
      },
      {
        "name": "search_web",
        "description": "搜尋網頁",
        "inputSchema": { ... }
      }
    ]
  }
}
```

### 4.2 工具描述格式

好的工具描述是 AI 正確使用工具的關鍵：

```json
{
  "name": "search_files",
  "description": "在專案目錄中搜尋文件。支援 glob 模式和正則表達式。回傳匹配的文件路徑列表。",
  "inputSchema": {
    "type": "object",
    "properties": {
      "pattern": {
        "type": "string",
        "description": "搜尋模式。支援 glob（如 '*.ts'）或正則表達式（如 '/test_.*/i'）"
      },
      "path": {
        "type": "string",
        "description": "搜尋的起始目錄。預設為專案根目錄。"
      },
      "maxResults": {
        "type": "number",
        "description": "最大回傳結果數。預設 100。",
        "default": 100
      }
    },
    "required": ["pattern"]
  }
}
```

### 4.3 動態工具發現

MCP 支援工具列表的動態變化。Server 可以在運行時新增或移除工具，並通知 Client：

```json
// Server 通知 Client 工具列表已變更
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
// Client 收到通知後重新查詢 tools/list
```

---

## 5. Transport 傳輸層

### 5.1 stdio 傳輸

最簡單的傳輸方式，通過標準輸入/輸出通訊：

```
MCP Client                     MCP Server（子進程）
    │                               │
    ├── stdin ──────────────────────▶│
    │                               │
    │◀────────────────────── stdout ─┤
    │                               │
```

```bash
# 啟動一個 stdio MCP Server
node my-mcp-server.js
# Server 從 stdin 讀取 JSON-RPC 請求
# Server 寫入 JSON-RPC 回應到 stdout
```

### 5.2 SSE 傳輸（HTTP）

使用 HTTP Server-Sent Events 進行通訊：

```
MCP Client                     MCP Server（HTTP）
    │                               │
    ├── POST /message ─────────────▶│
    │                               │
    │◀────────── SSE /events ───────┤
    │                               │
```

### 5.3 Streamable HTTP 傳輸

MCP 最新的傳輸方式，基於 HTTP 但支援串流：

```
MCP Client                     MCP Server（HTTP）
    │                               │
    ├── POST /mcp ─────────────────▶│
    │                               │
    │◀── SSE stream / JSON response ┤
    │                               │
```

### 5.4 傳輸層比較

| 特性 | stdio | SSE | Streamable HTTP |
|------|-------|-----|-----------------|
| 部署方式 | 本地子進程 | HTTP 伺服器 | HTTP 伺服器 |
| 適合場景 | 本地工具 | 遠端服務 | 遠端服務 |
| 串流支援 | ✅ | ✅ | ✅ |
| 防火牆友好 | N/A | ✅ | ✅ |
| 多 Client | ❌ | ✅ | ✅ |

---

## 6. MCP 訊息格式

### 6.1 初始化握手

```json
// Client → Server: 初始化請求
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-03-26",
    "capabilities": {
      "sampling": {}
    },
    "clientInfo": {
      "name": "OpenClaw",
      "version": "1.0.0"
    }
  }
}

// Server → Client: 初始化回應
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-03-26",
    "capabilities": {
      "tools": {},
      "resources": {},
      "prompts": {}
    },
    "serverInfo": {
      "name": "github-mcp-server",
      "version": "0.1.0"
    }
  }
}

// Client → Server: 初始化完成通知
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

### 6.2 工具呼叫

```json
// Client → Server: 呼叫工具
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "Taipei"
    }
  }
}

// Server → Client: 工具結果
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "台北市目前 28°C，多雲時晴，濕度 75%。"
      }
    ],
    "isError": false
  }
}
```

### 6.3 資源存取

```json
// Client → Server: 讀取資源
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "resources/read",
  "params": {
    "uri": "file:///project/config.json"
  }
}

// Server → Client: 資源內容
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "contents": [
      {
        "uri": "file:///project/config.json",
        "mimeType": "application/json",
        "text": "{ \"name\": \"my-project\" }"
      }
    ]
  }
}
```

### 6.4 錯誤處理

```json
// 錯誤回應
{
  "jsonrpc": "2.0",
  "id": 4,
  "error": {
    "code": -32602,
    "message": "Invalid params: 'city' is required"
  }
}
```

MCP 的標準錯誤碼：

| 錯誤碼 | 說明 |
|--------|------|
| -32700 | 解析錯誤（JSON 格式錯誤） |
| -32600 | 無效請求 |
| -32601 | 方法不存在 |
| -32602 | 參數無效 |
| -32603 | 內部錯誤 |

---

## 7. MCP 生態系統

### 7.1 主要 MCP Client

| Client | 說明 | 特點 |
|--------|------|------|
| Claude Desktop | Anthropic 官方桌面版 | 最早支援 MCP |
| Cursor | AI 程式碼編輯器 | 開發者最愛 |
| GitHub Copilot CLI | GitHub 官方 CLI 工具 | 終端整合 |
| Windsurf | Codeium 的 AI IDE | 程式碼編輯 |
| OpenClaw | AI 助手框架 | 多通道支援 |

### 7.2 MCP Server 範例

| Server | 功能 | 傳輸方式 |
|--------|------|----------|
| GitHub MCP Server | GitHub 操作（Issue、PR 等） | stdio |
| Filesystem MCP Server | 本地文件操作 | stdio |
| PostgreSQL MCP Server | 資料庫操作 | stdio |
| Brave Search MCP Server | 網頁搜尋 | stdio |
| Puppeteer MCP Server | 瀏覽器自動化 | stdio |

### 7.3 OpenClaw 在 MCP 生態中的定位

OpenClaw 在 MCP 生態中扮演雙重角色：

```
角色 1: MCP Client（使用 MCP Server 的工具）
  OpenClaw → GitHub MCP Server → 操作 GitHub
  OpenClaw → FS MCP Server → 讀寫文件
  OpenClaw → 任何 MCP Server → 擴展能力

角色 2: MCP Server（向其他 Client 提供工具）
  Copilot CLI → OpenClaw MCP Server → 使用 OpenClaw 的技能
  Claude → OpenClaw MCP Server → 使用 OpenClaw 的記憶
```

---

## 8. MCP 與其他方案的比較

### 8.1 MCP vs Function Calling

| 特性 | MCP | Function Calling |
|------|-----|-----------------|
| 協議標準 | 開放協議 | 各 LLM 自定義 |
| 跨平台 | ✅ | ❌ |
| 動態工具 | ✅ | 部分 |
| 傳輸層 | 多種 | HTTP |
| 生態系統 | 開放 | 封閉 |
| 安全控制 | 內建 | 需自行實作 |

### 8.2 MCP vs LangChain Tools

| 特性 | MCP | LangChain Tools |
|------|-----|-----------------|
| 語言無關 | ✅（JSON-RPC） | ❌（Python/JS） |
| 進程隔離 | ✅ | ❌ |
| 標準化 | ✅ | 框架特定 |
| 學習曲線 | 中 | 中 |

### 8.3 MCP vs OpenAI Plugins

| 特性 | MCP | OpenAI Plugins |
|------|-----|----------------|
| 狀態 | 活躍發展 | 已停用 |
| 開放性 | 完全開放 | OpenAI 控制 |
| 傳輸方式 | 多種 | HTTP |

---

## 9. 安全考量

### 9.1 權限控制

MCP 支援在協議層面控制工具的使用權限：

```json
{
  "permissions": {
    "tools": {
      "get_weather": "allow",
      "delete_file": "confirm",
      "execute_command": "deny"
    }
  }
}
```

### 9.2 資料隔離

每個 MCP Server 運行在獨立的進程中，天然實現了資料隔離。

### 9.3 信任邊界

```
高信任：本地 stdio Server（你安裝和控制的）
中信任：可信來源的遠端 Server
低信任：第三方未知 Server

建議：
- 只安裝來自可信來源的 MCP Server
- 對危險操作（如文件刪除、命令執行）設定 "confirm" 策略
- 定期審計 MCP Server 的行為日誌
```

---

## 10. 未來展望

MCP 仍在快速發展中。值得關注的趨勢：

- **更多供應商支援**：越來越多 AI 產品支援 MCP
- **MCP Registry**：官方的 MCP Server 注冊表和發現機制
- **進階安全機制**：OAuth 整合、細粒度權限控制
- **效能優化**：二進制協議支援、更低延遲
- **標準化進程**：可能成為正式的行業標準

OpenClaw 將持續跟進 MCP 的發展，確保與最新版本的相容性。
