# Copilot CLI 整合指南

## 目錄

- [1. 整合概述](#1-整合概述)
  - [1.1 為什麼整合 Copilot CLI](#11-為什麼整合-copilot-cli)
  - [1.2 整合價值](#12-整合價值)
  - [1.3 三種整合方式](#13-三種整合方式)
  - [1.4 架構總覽](#14-架構總覽)
- [2. 方式一：OpenClaw 作為 MCP Server 供 Copilot CLI 使用](#2-方式一openclaw-作為-mcp-server-供-copilot-cli-使用)
  - [2.1 概念說明](#21-概念說明)
  - [2.2 Step 1：配置 OpenClaw MCP Server 模式](#22-step-1配置-openclaw-mcp-server-模式)
  - [2.3 Step 2：在 Copilot CLI 中註冊 OpenClaw](#23-step-2在-copilot-cli-中註冊-openclaw)
  - [2.4 Step 3：驗證連線](#24-step-3驗證連線)
  - [2.5 使用範例](#25-使用範例)
  - [2.6 進階配置](#26-進階配置)
- [3. 方式二：Copilot CLI 的 MCP Server 供 OpenClaw 使用](#3-方式二copilot-cli-的-mcp-server-供-openclaw-使用)
  - [3.1 概念說明](#31-概念說明)
  - [3.2 可用的 MCP Server](#32-可用的-mcp-server)
  - [3.3 在 OpenClaw 中配置](#33-在-openclaw-中配置)
  - [3.4 使用範例](#34-使用範例)
- [4. 方式三：共用 MCP Server 生態](#4-方式三共用-mcp-server-生態)
  - [4.1 概念說明](#41-概念說明)
  - [4.2 共享配置](#42-共享配置)
  - [4.3 統一工具生態](#43-統一工具生態)
- [5. EasonClaw 完整整合藍圖](#5-easonclaw-完整整合藍圖)
  - [5.1 目標架構](#51-目標架構)
  - [5.2 Step-by-Step 建置指南](#52-step-by-step-建置指南)
  - [5.3 配置文件完整範例](#53-配置文件完整範例)
  - [5.4 工作流程範例](#54-工作流程範例)
- [6. MCP Config 統一管理](#6-mcp-config-統一管理)
  - [6.1 配置文件位置](#61-配置文件位置)
  - [6.2 共享配置策略](#62-共享配置策略)
  - [6.3 配置同步](#63-配置同步)
- [7. 開發者工作流整合](#7-開發者工作流整合)
  - [7.1 程式碼開發流程](#71-程式碼開發流程)
  - [7.2 日常任務自動化](#72-日常任務自動化)
  - [7.3 跨工具協作](#73-跨工具協作)
- [8. 安全與權限](#8-安全與權限)
  - [8.1 Token 管理](#81-token-管理)
  - [8.2 權限邊界](#82-權限邊界)
  - [8.3 審計日誌](#83-審計日誌)
- [9. 疑難排解](#9-疑難排解)
  - [9.1 常見連線問題](#91-常見連線問題)
  - [9.2 工具不可用](#92-工具不可用)
  - [9.3 效能問題](#93-效能問題)
- [10. 最佳實踐與建議](#10-最佳實踐與建議)

---

## 1. 整合概述

### 1.1 為什麼整合 Copilot CLI

GitHub Copilot CLI 是 GitHub 官方的終端 AI 助手，它能直接在終端中回答問題、執行任務、操作 Git 倉庫。將 OpenClaw 與 Copilot CLI 整合，可以讓兩個 AI 系統互補，創造出 1+1>2 的效果。

**OpenClaw 的優勢**：
- 自定義人格和記憶
- 多通道支援（Discord、WhatsApp、Telegram）
- 語音能力（Talk Mode）
- 自定義技能（SKILL.md）
- 開放且可控

**Copilot CLI 的優勢**：
- 深度 GitHub 整合
- 豐富的程式碼理解能力
- 強大的終端操作能力
- 原生 MCP 支援
- GitHub 生態系統

**整合後的效果**：

```
OpenClaw (多通道 AI 助手)
    +
Copilot CLI (開發者 AI 工具)
    =
一個能在 Discord 語音頻道中幫你寫程式、
在 WhatsApp 中幫你管理 GitHub Issue、
在終端中既有記憶又有程式碼能力的
超級 AI 助手 🚀
```

### 1.2 整合價值

| 場景 | 只用 OpenClaw | 只用 Copilot CLI | 整合後 |
|------|-------------|-----------------|--------|
| Discord 問程式問題 | ✅ 可以但程式碼理解有限 | ❌ 無法在 Discord | ✅ Discord 中有完整程式碼能力 |
| 終端管理 GitHub | ❌ 需要自寫技能 | ✅ 原生支援 | ✅ 兩者都可用 |
| 語音寫程式 | ⚠️ 有語音但程式碼能力有限 | ❌ 無語音 | ✅ 語音 + 程式碼 |
| 跨專案記憶 | ✅ 有記憶 | ❌ 無長期記憶 | ✅ 有記憶 |
| CI/CD 管理 | ❌ 需自寫 | ✅ 原生支援 | ✅ 透過 MCP 共享 |

### 1.3 三種整合方式

```
方式一：OpenClaw → MCP Server → Copilot CLI 使用
  OpenClaw 暴露技能和記憶給 Copilot CLI

方式二：Copilot CLI 生態的 MCP Server → OpenClaw 使用
  OpenClaw 使用 Copilot CLI 環境中的 MCP Server

方式三：共用 MCP Server 生態
  兩者共享同一套 MCP Server 配置
```

### 1.4 架構總覽

```
┌─────────────────────────────────────────────────────────┐
│                    MCP Server 生態                        │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │ GitHub   │  │ FileSystem│  │ Database │  │ Search │  │
│  │ Server   │  │ Server   │  │ Server   │  │ Server │  │
│  └────▲─────┘  └────▲─────┘  └────▲─────┘  └───▲────┘  │
│       │              │              │             │       │
└───────┼──────────────┼──────────────┼─────────────┼──────┘
        │              │              │             │
   ┌────┴────┐    ┌────┴────┐    ┌────┴────┐       │
   │ Copilot │    │OpenClaw │    │ Copilot │       │
   │   CLI   │    │(Client) │    │   CLI   │       │
   └─────────┘    └────┬────┘    └─────────┘       │
                       │                            │
                  ┌────┴────┐                       │
                  │OpenClaw │                       │
                  │(Server) │───────────────────────┘
                  └────┬────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
     ┌────┴───┐  ┌────┴───┐  ┌────┴───┐
     │Discord │  │WhatsApp│  │Telegram│
     └────────┘  └────────┘  └────────┘
```

---

## 2. 方式一：OpenClaw 作為 MCP Server 供 Copilot CLI 使用

### 2.1 概念說明

在這種模式下，OpenClaw 作為 MCP Server 運行，向 Copilot CLI 暴露 OpenClaw 的技能和記憶。Copilot CLI 可以直接使用 OpenClaw 的能力，例如查詢天氣、翻譯文字、存取對話記憶等。

```
┌──────────────┐         ┌──────────────┐
│ Copilot CLI  │────────▶│   OpenClaw   │
│ (MCP Client) │  stdio  │ (MCP Server) │
│              │         │              │
│  使用者在     │         │  暴露技能：    │
│  終端中問問題 │         │  - 天氣查詢    │
│              │         │  - 翻譯       │
│              │         │  - 記憶查詢    │
│              │         │  - 任務管理    │
└──────────────┘         └──────────────┘
```

### 2.2 Step 1：配置 OpenClaw MCP Server 模式

在 `openclaw.json` 中啟用 MCP Server：

```json
{
  "mcpServer": {
    "enabled": true,
    "transport": "stdio",
    
    "exposedSkills": [
      "weather-query",
      "translate",
      "summarize",
      "task-manager",
      "calendar"
    ],
    
    "exposedResources": {
      "memory": {
        "enabled": true,
        "scope": "all-sessions",
        "description": "OpenClaw 的對話記憶"
      },
      "persona": {
        "enabled": true,
        "description": "AI 助手的人格設定"
      },
      "recentConversations": {
        "enabled": true,
        "maxItems": 50,
        "description": "最近的對話紀錄"
      }
    },
    
    "security": {
      "allowedClients": ["copilot-cli"],
      "maxRequestsPerMinute": 120
    }
  }
}
```

### 2.3 Step 2：在 Copilot CLI 中註冊 OpenClaw

編輯 Copilot CLI 的 MCP 配置文件：

```bash
# Copilot CLI 的 MCP 配置位置
# ~/.copilot/mcp-config.json（全域）
# 或 .mcp.json（專案級別）
```

```json
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw",
      "args": ["mcp", "serve"],
      "env": {
        "OPENCLAW_HOME": "/home/eason/.openclaw"
      }
    }
  }
}
```

或者使用 `npx` 方式：

```json
{
  "mcpServers": {
    "openclaw": {
      "command": "npx",
      "args": ["-y", "openclaw", "mcp", "serve"]
    }
  }
}
```

### 2.4 Step 3：驗證連線

```bash
# 在 Copilot CLI 中驗證 OpenClaw 連線
# Copilot CLI 啟動時會自動連接已配置的 MCP Server
# 可以直接詢問 Copilot CLI 來驗證：

copilot "你能使用 OpenClaw 的天氣查詢功能嗎？"
# → Copilot CLI 應該能看到並使用 OpenClaw 的工具
```

### 2.5 使用範例

配置完成後，在 Copilot CLI 中可以這樣使用 OpenClaw 的能力：

```bash
# 在終端中使用 Copilot CLI，它可以呼叫 OpenClaw 的技能

# 查詢天氣（使用 OpenClaw 的天氣技能）
copilot "幫我查台北的天氣"
# → Copilot CLI 透過 MCP 呼叫 OpenClaw 的 weather-query 工具

# 查詢 OpenClaw 的記憶
copilot "我上次和 EasonClaw 聊了什麼？"
# → Copilot CLI 透過 MCP 讀取 OpenClaw 的記憶資源

# 使用 OpenClaw 的翻譯技能
copilot "用 EasonClaw 的翻譯功能翻譯 'Hello World'"
# → Copilot CLI 透過 MCP 呼叫 OpenClaw 的 translate 工具

# 管理任務
copilot "在 EasonClaw 中新增一個任務：完成 MCP 整合文件"
# → Copilot CLI 透過 MCP 呼叫 OpenClaw 的 task-manager 工具
```

### 2.6 進階配置

**選擇性暴露技能**：

不是所有技能都適合暴露給 Copilot CLI。建議只暴露工具性質的技能：

```json
{
  "mcpServer": {
    "exposedSkills": [
      "weather-query",
      "translate",
      "task-manager"
    ],
    "hiddenSkills": [
      "casual-chat",
      "emotional-support"
    ],
    "comment": "閒聊和情感支持不適合透過 MCP 暴露"
  }
}
```

**自定義工具描述**：

為 Copilot CLI 最佳化工具描述：

```json
{
  "mcpServer": {
    "toolDescriptionOverrides": {
      "weather-query": {
        "description": "Query current weather for a city. Returns temperature, humidity, and conditions. Best for quick weather checks. Supports cities worldwide with Chinese city names.",
        "comment": "為英文環境的 Copilot CLI 提供英文描述"
      }
    }
  }
}
```

---

## 3. 方式二：Copilot CLI 的 MCP Server 供 OpenClaw 使用

### 3.1 概念說明

在這種模式下，OpenClaw 作為 MCP Client，使用那些通常與 Copilot CLI 搭配使用的 MCP Server。這些 Server 提供了強大的開發者工具能力。

```
┌──────────────┐         ┌──────────────┐
│   OpenClaw   │────────▶│  GitHub MCP  │
│ (MCP Client) │  stdio  │   Server     │
│              │         │              │
│  Discord 中   │         │  提供：       │
│  使用者問問題 │         │  - 搜尋倉庫   │
│              │         │  - 建立 Issue │
│              │         │  - 管理 PR    │
│              │         │  - 讀取文件   │
└──────────────┘         └──────────────┘
```

### 3.2 可用的 MCP Server

Copilot CLI 生態中常用的 MCP Server：

| Server | 功能 | 安裝方式 |
|--------|------|----------|
| GitHub MCP Server | GitHub 完整操作 | `npm i -g @modelcontextprotocol/server-github` |
| Filesystem Server | 文件系統操作 | `npm i -g @modelcontextprotocol/server-filesystem` |
| Brave Search | 網頁搜尋 | `npm i -g @anthropic/mcp-server-brave-search` |
| Puppeteer | 瀏覽器自動化 | `npm i -g @anthropic/mcp-server-puppeteer` |
| SQLite | 資料庫操作 | `npm i -g @anthropic/mcp-server-sqlite` |
| Memory | 知識圖譜記憶 | `npm i -g @anthropic/mcp-server-memory` |

### 3.3 在 OpenClaw 中配置

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      },
      "controlPolicy": {
        "default": "allow",
        "overrides": {
          "create_pull_request": "confirm",
          "push_files": "confirm",
          "delete_repository": "deny"
        }
      }
    },
    
    "filesystem": {
      "command": "npx",
      "args": [
        "-y", "@modelcontextprotocol/server-filesystem",
        "/home/eason/projects"
      ],
      "controlPolicy": {
        "default": "confirm",
        "overrides": {
          "read_file": "allow",
          "list_directory": "allow"
        }
      }
    },
    
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "${BRAVE_API_KEY}"
      }
    }
  }
}
```

### 3.4 使用範例

配置後，使用者可以在任何 OpenClaw 通道中使用這些工具：

```
# Discord 中的對話：

使用者: @EasonClaw 幫我看一下 eason/my-project 的最近 Issue

EasonClaw: 好的，讓我查一下...
(透過 GitHub MCP Server 查詢)

📋 eason/my-project 最近的 Issue：
1. #42 - 修復登入頁面 CSS 問題 (open)
2. #41 - 新增暗色模式支援 (open)
3. #40 - API 回應時間過長 (closed)

需要我處理哪個 Issue 嗎？

---

使用者: @EasonClaw 幫我搜尋 "MCP protocol" 的最新資訊

EasonClaw: 🔍 搜尋中...
(透過 Brave Search MCP Server 搜尋)

以下是最新的 MCP 相關資訊：
1. Anthropic 發布 MCP 協議 2025-03-26 版本...
2. GitHub Copilot CLI 新增 MCP 支援...
...
```

---

## 4. 方式三：共用 MCP Server 生態

### 4.1 概念說明

最理想的整合方式是讓 OpenClaw 和 Copilot CLI 共享同一套 MCP Server 配置，確保工具生態的一致性。

```
共用 MCP 配置
    │
    ├── Copilot CLI 使用
    │   ├── GitHub Server
    │   ├── Filesystem Server
    │   └── OpenClaw Server（方式一）
    │
    └── OpenClaw 使用
        ├── GitHub Server
        ├── Filesystem Server
        └── Brave Search Server
```

### 4.2 共享配置

創建一個共享的 MCP 配置，兩個工具都引用：

```json
// ~/.config/mcp/shared-servers.json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y", "@modelcontextprotocol/server-filesystem",
        "/home/eason/projects"
      ]
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "${BRAVE_API_KEY}"
      }
    }
  }
}
```

OpenClaw 引用共享配置：

```json
// openclaw.json
{
  "mcp": {
    "includeFrom": "~/.config/mcp/shared-servers.json"
  },
  "mcpServers": {
    "comment": "額外的 OpenClaw 專用 Server 可以加在這裡"
  }
}
```

### 4.3 統一工具生態

共享配置的好處：

- **一致性**：兩個工具看到相同的工具集
- **維護簡單**：只需維護一份配置
- **避免衝突**：不會有重複的 Server 實例
- **統一 Token**：API Token 只需配置一次

---

## 5. EasonClaw 完整整合藍圖

### 5.1 目標架構

為 EasonClaw 專案設計的完整整合方案：

```
┌─────────────────── EasonClaw 系統 ──────────────────────┐
│                                                          │
│  ┌───────────── OpenClaw Core ─────────────┐             │
│  │                                          │             │
│  │  人格：EasonClaw 🐾                      │             │
│  │  記憶：長期記憶 + 對話記憶                │             │
│  │  技能：天氣、翻譯、任務、筆記 等          │             │
│  │                                          │             │
│  │  ┌──── MCP Client ────┐                 │             │
│  │  │ GitHub Server      │                 │             │
│  │  │ Filesystem Server  │                 │             │
│  │  │ Brave Search       │                 │             │
│  │  └────────────────────┘                 │             │
│  │                                          │             │
│  │  ┌──── MCP Server ────┐                 │             │
│  │  │ 暴露技能給外部      │◀──── Copilot CLI│             │
│  │  └────────────────────┘                 │             │
│  │                                          │             │
│  │  ┌──── 通道 ────────────┐               │             │
│  │  │ Discord（主要）       │               │             │
│  │  │ Telegram（備用）      │               │             │
│  │  │ CLI（開發）           │               │             │
│  │  └──────────────────────┘               │             │
│  └──────────────────────────────────────────┘             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 5.2 Step-by-Step 建置指南

```
Phase 1: 基礎設定（30 分鐘）
  ☐ 安裝 OpenClaw
  ☐ 配置基本人格和記憶
  ☐ 啟用 CLI 通道測試

Phase 2: MCP Client 設定（20 分鐘）
  ☐ 安裝 GitHub MCP Server
  ☐ 安裝 Filesystem MCP Server
  ☐ 安裝 Brave Search MCP Server
  ☐ 在 openclaw.json 中配置
  ☐ 測試各工具是否正常運作

Phase 3: MCP Server 設定（15 分鐘）
  ☐ 啟用 OpenClaw MCP Server 模式
  ☐ 選擇要暴露的技能
  ☐ 在 Copilot CLI 中註冊 OpenClaw
  ☐ 驗證 Copilot CLI 可以使用 OpenClaw 工具

Phase 4: 通道配置（20 分鐘）
  ☐ 配置 Discord 通道
  ☐ 配置 Telegram 通道
  ☐ 測試各通道的 MCP 工具存取

Phase 5: 權限與安全（10 分鐘）
  ☐ 設定 controlPolicy
  ☐ 配置 Token 安全存儲
  ☐ 測試確認機制
```

### 5.3 配置文件完整範例

```json
{
  "persona": {
    "name": "EasonClaw",
    "role": "你是 EasonClaw 🐾，Eason 的個人 AI 助手。你友善、有幫助、記得之前的對話。你擅長程式開發、日常任務管理、資訊查詢。",
    "language": "zh-TW"
  },
  
  "memory": {
    "enabled": true,
    "provider": "local",
    "longTerm": {
      "enabled": true,
      "maxEntries": 10000
    }
  },
  
  "llm": {
    "provider": "openai",
    "model": "gpt-4o",
    "temperature": 0.7,
    "maxTokens": 4096
  },
  
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      },
      "controlPolicy": {
        "default": "allow",
        "overrides": {
          "create_pull_request": "confirm",
          "push_files": "confirm",
          "delete_repository": "deny"
        }
      }
    },
    
    "filesystem": {
      "command": "npx",
      "args": [
        "-y", "@modelcontextprotocol/server-filesystem",
        "/home/eason/projects",
        "/home/eason/documents"
      ],
      "controlPolicy": {
        "default": "confirm",
        "overrides": {
          "read_file": "allow",
          "list_directory": "allow",
          "write_file": "confirm",
          "delete_file": "deny"
        }
      }
    },
    
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "${BRAVE_API_KEY}"
      },
      "controlPolicy": {
        "default": "allow"
      }
    }
  },
  
  "mcpServer": {
    "enabled": true,
    "transport": "stdio",
    "exposedSkills": [
      "weather-query",
      "translate",
      "task-manager",
      "note-taker",
      "calendar"
    ],
    "exposedResources": {
      "memory": { "enabled": true, "scope": "all-sessions" },
      "persona": { "enabled": true }
    }
  },
  
  "channels": {
    "discord": {
      "enabled": true,
      "botToken": "${DISCORD_BOT_TOKEN}",
      "allowFrom": {
        "users": ["discord:YOUR_DISCORD_ID"],
        "guilds": ["discord:YOUR_GUILD_ID"]
      },
      "voice": {
        "enabled": true,
        "talkMode": true
      }
    },
    
    "telegram": {
      "enabled": true,
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "allowFrom": {
        "users": ["telegram:YOUR_TELEGRAM_ID"]
      }
    },
    
    "cli": {
      "enabled": true
    }
  },
  
  "voice": {
    "stt": {
      "provider": "whisper",
      "language": "zh"
    },
    "tts": {
      "provider": "edge-tts",
      "voice": "zh-TW-HsiaoChenNeural"
    }
  }
}
```

Copilot CLI 端的配置：

```json
// ~/.copilot/mcp-config.json
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw",
      "args": ["mcp", "serve"],
      "env": {
        "OPENCLAW_HOME": "/home/eason/.openclaw"
      }
    }
  }
}
```

### 5.4 工作流程範例

**範例 1：在 Discord 中管理 GitHub Issue**

```
使用者（Discord）: @EasonClaw 幫我看一下 my-project 有什麼 open 的 issue

EasonClaw: 讓我查一下... 🔍
(呼叫 github MCP Server → list_issues)

📋 eason/my-project 目前有 3 個 open issue：

1. #42 - Fix login page CSS
   Labels: bug, frontend
   
2. #41 - Add dark mode support  
   Labels: enhancement, frontend

3. #40 - Optimize API response time
   Labels: performance, backend

要我幫你處理哪一個嗎？

---

使用者: 幫我把 #42 標記為 in-progress，然後建一個相關的 branch

EasonClaw: 好的！
(呼叫 github MCP Server → add_label + create_branch)

✅ 完成！
- Issue #42 已標記為 in-progress
- 已建立分支 fix/login-page-css

你可以開始開發了：
git checkout fix/login-page-css
```

**範例 2：在 Copilot CLI 中使用 OpenClaw 的記憶**

```bash
$ copilot "我之前和 EasonClaw 討論過什麼 API 設計方案？"

# Copilot CLI 透過 MCP 查詢 OpenClaw 的記憶：

根據 EasonClaw 的對話記憶，你們在 3 天前討論了以下 API 設計方案：

1. REST API 方案：使用標準 RESTful 設計
2. GraphQL 方案：更靈活的查詢
3. 最終決定：採用 REST + GraphQL hybrid 方案

關鍵決策：
- 公開 API 用 REST（簡單明確）
- 內部 API 用 GraphQL（靈活高效）
- 認證統一用 JWT
```

**範例 3：跨工具協作開發**

```bash
# 1. 在 Copilot CLI 中開始工作
$ copilot "幫我 review 最近的 PR"
# → 使用 GitHub MCP Server

# 2. 切換到 Discord 語音頻道
使用者: @EasonClaw /voice join
# → 用語音討論 code review 的發現

# 3. 回到終端
$ copilot "根據 EasonClaw 的建議，修改 auth.ts 中的 token 驗證邏輯"
# → Copilot CLI 可以參考 OpenClaw 記憶中的對話

# 4. 在 Telegram 上收到通知
EasonClaw (Telegram): PR #43 已經通過 CI 測試 ✅
要我幫你 merge 嗎？
```

---

## 6. MCP Config 統一管理

### 6.1 配置文件位置

```
Copilot CLI 的 MCP 配置：
  全域：~/.copilot/mcp-config.json
  專案：.mcp.json（專案根目錄）

OpenClaw 的 MCP 配置：
  主配置：~/.openclaw/openclaw.json 的 mcpServers 區塊
  
共享配置建議位置：
  ~/.config/mcp/shared-servers.json
```

### 6.2 共享配置策略

```bash
# 建立共享配置目錄
mkdir -p ~/.config/mcp

# 建立共享的 MCP Server 配置
cat > ~/.config/mcp/shared-servers.json << 'EOF'
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/eason/projects"]
    }
  }
}
EOF
```

### 6.3 配置同步

如果在多台設備上使用，可以通過 Git 同步配置：

```bash
# 將配置納入 dotfiles 管理
cd ~/dotfiles
ln -s ~/.config/mcp/shared-servers.json mcp-config.json
git add mcp-config.json
git commit -m "Add shared MCP config"
```

---

## 7. 開發者工作流整合

### 7.1 程式碼開發流程

```
1. 需求討論（Discord 語音 + OpenClaw）
   → Talk Mode 中討論技術方案
   → OpenClaw 記憶會保存討論結果

2. 程式碼編寫（Copilot CLI + IDE）
   → 使用 Copilot CLI 的程式碼能力
   → 透過 MCP 查詢 OpenClaw 記憶中的設計決策

3. Code Review（Discord + GitHub MCP）
   → 在 Discord 中 @EasonClaw review PR
   → OpenClaw 透過 GitHub MCP Server 讀取 PR diff

4. 部署與監控（Telegram 通知）
   → CI/CD 完成後 OpenClaw 透過 Telegram 通知
```

### 7.2 日常任務自動化

```json
{
  "automations": [
    {
      "trigger": "每天早上 9:00",
      "action": "透過 GitHub MCP 檢查 assigned issues，在 Discord 回報"
    },
    {
      "trigger": "PR 被 merge",
      "action": "透過 Telegram 發送通知"
    },
    {
      "trigger": "使用者說 '開始工作'",
      "action": "總結今天的待辦事項，包括 GitHub issues 和任務列表"
    }
  ]
}
```

### 7.3 跨工具協作

```
場景：處理一個複雜的 bug

1. Discord 語音：「EasonClaw，我發現 API 回傳 500 錯誤」
2. OpenClaw 記錄問題到記憶
3. OpenClaw 透過 GitHub MCP 建立 Issue
4. 使用者切換到終端
5. Copilot CLI：「幫我看 API 的 error log」
6. Copilot CLI 透過 OpenClaw MCP 查詢相關記憶
7. 找到問題，修復程式碼
8. Copilot CLI 建立 PR
9. OpenClaw 在 Discord 中通知：「PR #44 已建立，請 review」
```

---

## 8. 安全與權限

### 8.1 Token 管理

```bash
# 所有 Token 統一存放在 .env 文件
cat ~/.openclaw/.env

GITHUB_TOKEN=ghp_xxxxxxxxxxxx
DISCORD_BOT_TOKEN=MTIx...
TELEGRAM_BOT_TOKEN=7123...
BRAVE_API_KEY=BSA...
OPENAI_API_KEY=sk-...
ELEVENLABS_API_KEY=...

# 確保文件權限
chmod 600 ~/.openclaw/.env
```

### 8.2 權限邊界

```
OpenClaw 作為 MCP Client 時：
  → 使用 controlPolicy 控制工具權限
  → 危險操作需要使用者確認
  → 某些操作可以完全禁止

OpenClaw 作為 MCP Server 時：
  → 只暴露安全的技能
  → 限制 Client 的請求頻率
  → 記憶的暴露範圍可控

Copilot CLI 使用 OpenClaw 時：
  → OpenClaw 的安全策略仍然有效
  → Copilot CLI 無法繞過 OpenClaw 的權限控制
```

### 8.3 審計日誌

```json
{
  "security": {
    "auditLog": {
      "enabled": true,
      "logPath": "~/.openclaw/logs/audit.log",
      "logLevel": "all",
      "includeToolCalls": true,
      "includeResourceAccess": true,
      "retention": "90d"
    }
  }
}
```

---

## 9. 疑難排解

### 9.1 常見連線問題

**問題：OpenClaw MCP Server 啟動失敗**

```bash
# 手動測試 Server 啟動
openclaw mcp serve --debug

# 常見原因：
# 1. openclaw 命令不在 PATH 中
# 2. 配置文件格式錯誤
# 3. 依賴套件未安裝
```

**問題：Copilot CLI 看不到 OpenClaw 的工具**

```bash
# 確認 MCP 配置正確
cat ~/.copilot/mcp-config.json

# 確認 openclaw 命令可用
which openclaw

# 重啟 Copilot CLI
```

### 9.2 工具不可用

```bash
# 檢查 MCP Server 狀態
openclaw mcp status

# 查看特定 Server 的工具
openclaw mcp tools github

# 測試特定工具
openclaw mcp call github search_repositories '{"query": "test"}'
```

### 9.3 效能問題

```bash
# 查看 MCP 通訊延遲
openclaw mcp benchmark

# 常見效能問題：
# 1. npx 每次啟動都要下載 → 改用全局安裝
# 2. 太多 Server 同時連接 → 使用 lazyConnect
# 3. 網路延遲 → 優先使用本地 Server
```

---

## 10. 最佳實踐與建議

**1. 從簡單開始**

```
第一步：CLI + GitHub MCP Server
第二步：加入 Discord 通道
第三步：啟用 MCP Server 模式
第四步：加入更多 MCP Server
第五步：啟用語音和 Talk Mode
```

**2. 權限最小化原則**

```json
{
  "controlPolicy": {
    "default": "confirm",
    "comment": "預設需要確認，逐步開放 allow 的工具"
  }
}
```

**3. 合理分工**

```
OpenClaw 負責：
  - 多通道互動（Discord、WhatsApp、Telegram）
  - 人格和記憶管理
  - 語音對話
  - 日常任務管理

Copilot CLI 負責：
  - 終端操作
  - 程式碼理解和生成
  - Git/GitHub 操作

MCP Server 負責：
  - 具體的工具能力
  - 資料存取
  - 外部 API 整合
```

**4. 持續優化**

```bash
# 定期檢查使用情況
openclaw mcp analytics

# 根據使用頻率調整權限
# 高頻工具 → allow
# 中頻工具 → confirm（確認後記住選擇）
# 低頻危險工具 → deny
```

**5. EasonClaw 專案的建議配置**

對於你的 EasonClaw 專案，建議的優先級：

```
必裝 MCP Server：
  1. GitHub MCP Server（管理你的專案）
  2. Filesystem MCP Server（讀寫本地文件）

推薦 MCP Server：
  3. Brave Search（網頁搜尋）
  4. SQLite（本地資料庫）

可選 MCP Server：
  5. Puppeteer（網頁截圖/自動化）
  6. Memory（額外的知識圖譜記憶）
```

最終目標：一個能在任何地方（終端、Discord、Telegram）、任何方式（文字、語音）與你互動的 AI 助手，同時擁有完整的開發者工具鏈。
