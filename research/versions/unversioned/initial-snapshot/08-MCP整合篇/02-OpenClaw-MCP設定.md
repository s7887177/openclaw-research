# OpenClaw MCP 設定指南

## 目錄

- [1. 概述](#1-概述)
  - [1.1 OpenClaw 的 MCP 雙重角色](#11-openclaw-的-mcp-雙重角色)
  - [1.2 為什麼要整合 MCP](#12-為什麼要整合-mcp)
- [2. 安裝 MCP 擴展](#2-安裝-mcp-擴展)
  - [2.1 內建 MCP 支援](#21-內建-mcp-支援)
  - [2.2 安裝 MCP Server](#22-安裝-mcp-server)
  - [2.3 從 ClawHub 安裝](#23-從-clawhub-安裝)
  - [2.4 手動安裝](#24-手動安裝)
- [3. openclaw.json MCP 配置](#3-openclawjson-mcp-配置)
  - [3.1 基礎配置結構](#31-基礎配置結構)
  - [3.2 stdio 傳輸配置](#32-stdio-傳輸配置)
  - [3.3 HTTP 傳輸配置](#33-http-傳輸配置)
  - [3.4 環境變數注入](#34-環境變數注入)
  - [3.5 完整配置範例](#35-完整配置範例)
- [4. 多 Server 管理](#4-多-server-管理)
  - [4.1 同時連接多個 Server](#41-同時連接多個-server)
  - [4.2 工具衝突處理](#42-工具衝突處理)
  - [4.3 Server 優先級](#43-server-優先級)
  - [4.4 條件啟用](#44-條件啟用)
- [5. 權限與安全](#5-權限與安全)
  - [5.1 controlPolicy 設定](#51-controlpolicy-設定)
  - [5.2 工具級別權限](#52-工具級別權限)
  - [5.3 使用者確認機制](#53-使用者確認機制)
- [6. OpenClaw 作為 MCP Server](#6-openclaw-作為-mcp-server)
  - [6.1 啟動 MCP Server 模式](#61-啟動-mcp-server-模式)
  - [6.2 暴露技能為 MCP Tools](#62-暴露技能為-mcp-tools)
  - [6.3 暴露記憶為 MCP Resources](#63-暴露記憶為-mcp-resources)
  - [6.4 安全控制](#64-安全控制)
- [7. 常用 MCP Server 配置](#7-常用-mcp-server-配置)
  - [7.1 GitHub MCP Server](#71-github-mcp-server)
  - [7.2 Filesystem MCP Server](#72-filesystem-mcp-server)
  - [7.3 Brave Search MCP Server](#73-brave-search-mcp-server)
  - [7.4 PostgreSQL MCP Server](#74-postgresql-mcp-server)
- [8. 除錯與維護](#8-除錯與維護)
  - [8.1 連線除錯](#81-連線除錯)
  - [8.2 工具測試](#82-工具測試)
  - [8.3 日誌與監控](#83-日誌與監控)
- [9. 效能調校](#9-效能調校)
- [10. 常見問題](#10-常見問題)

---

## 1. 概述

### 1.1 OpenClaw 的 MCP 雙重角色

OpenClaw 在 MCP 生態系統中扮演雙重角色：

```
角色 1: MCP Client
  ┌──────────┐        ┌──────────────┐
  │ OpenClaw │───────▶│ MCP Server   │
  │ (Client) │        │ (GitHub 等)  │
  └──────────┘        └──────────────┘
  → 使用外部工具擴展能力

角色 2: MCP Server
  ┌──────────────┐        ┌──────────┐
  │ Copilot CLI  │───────▶│ OpenClaw │
  │ (Client)     │        │ (Server) │
  └──────────────┘        └──────────┘
  → 向其他 AI 工具提供 OpenClaw 的能力
```

### 1.2 為什麼要整合 MCP

- **能力擴展**：不需要寫新技能，直接使用現有的 MCP Server
- **生態共享**：社群開發的 MCP Server 可以直接使用
- **標準化**：統一的介面，降低整合成本
- **互操作性**：OpenClaw 的技能可以被其他 MCP Client 使用

---

## 2. 安裝 MCP 擴展

### 2.1 內建 MCP 支援

OpenClaw 內建 MCP Client 支援，不需要額外安裝：

```bash
# 確認 MCP 支援已啟用
openclaw mcp status

# 輸出：
# MCP Support: ✅ Enabled
# Protocol Version: 2025-03-26
# Connected Servers: 0
```

### 2.2 安裝 MCP Server

大多數 MCP Server 以 npm 或 pip 套件形式發布：

```bash
# 安裝 GitHub MCP Server
npm install -g @modelcontextprotocol/server-github

# 安裝 Filesystem MCP Server
npm install -g @modelcontextprotocol/server-filesystem

# 安裝 Brave Search MCP Server
npm install -g @anthropic/mcp-server-brave-search

# 安裝 PostgreSQL MCP Server
pip install mcp-server-postgres
```

### 2.3 從 ClawHub 安裝

OpenClaw 的 ClawHub 也提供 MCP Server 的安裝：

```bash
# 從 ClawHub 搜尋 MCP Server
openclaw hub search --type mcp-server

# 安裝
openclaw hub install mcp-server-github
```

### 2.4 手動安裝

對於自定義的 MCP Server：

```bash
# 克隆並建置
git clone https://github.com/example/my-mcp-server.git
cd my-mcp-server
npm install && npm run build

# 在 openclaw.json 中配置路徑
```

---

## 3. openclaw.json MCP 配置

### 3.1 基礎配置結構

MCP 配置位於 `openclaw.json` 的 `mcpServers` 區塊：

```json
{
  "mcpServers": {
    "server-name": {
      "command": "命令",
      "args": ["參數"],
      "env": { "環境變數": "值" }
    }
  }
}
```

### 3.2 stdio 傳輸配置

stdio 是最常見的傳輸方式，MCP Server 作為子進程運行：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

配置說明：

| 欄位 | 類型 | 說明 |
|------|------|------|
| `command` | string | 啟動 Server 的命令 |
| `args` | string[] | 命令參數 |
| `env` | object | 傳遞給 Server 的環境變數 |

### 3.3 HTTP 傳輸配置

對於遠端 MCP Server：

```json
{
  "mcpServers": {
    "remote-server": {
      "url": "https://mcp.example.com/sse",
      "transport": "sse",
      "headers": {
        "Authorization": "Bearer ${MCP_AUTH_TOKEN}"
      }
    }
  }
}
```

### 3.4 環境變數注入

OpenClaw 支援在配置中使用 `${VAR_NAME}` 語法引用環境變數：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

環境變數來源優先級：

1. `.env` 文件（`~/.openclaw/.env`）
2. 系統環境變數
3. 配置文件中的預設值

### 3.5 完整配置範例

```json
{
  "persona": {
    "name": "EasonClaw",
    "role": "你是一個全能的 AI 助手"
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
          "delete_repository": "deny",
          "create_issue": "confirm"
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
  
  "channels": {
    "discord": { "enabled": true },
    "cli": { "enabled": true }
  }
}
```

---

## 4. 多 Server 管理

### 4.1 同時連接多個 Server

OpenClaw 可以同時連接多個 MCP Server：

```json
{
  "mcpServers": {
    "github": { "command": "..." },
    "filesystem": { "command": "..." },
    "database": { "command": "..." },
    "search": { "command": "..." }
  }
}
```

所有 Server 的工具會被統一呈現給 LLM，LLM 自動選擇最適合的工具。

### 4.2 工具衝突處理

如果多個 Server 提供了相同名稱的工具，OpenClaw 會使用前綴來區分：

```
原始：
  github → search（搜尋 GitHub）
  search → search（搜尋網頁）

解決衝突後：
  github.search（搜尋 GitHub）
  search.search（搜尋網頁）
```

```json
{
  "mcpServers": {
    "github": {
      "toolPrefix": "gh",
      "comment": "工具名會變成 gh_search, gh_create_issue 等"
    }
  }
}
```

### 4.3 Server 優先級

```json
{
  "mcpServers": {
    "github": {
      "priority": 1,
      "comment": "最高優先級"
    },
    "backup-github": {
      "priority": 2,
      "fallbackFor": "github",
      "comment": "當 github server 不可用時使用"
    }
  }
}
```

### 4.4 條件啟用

根據通道或使用者條件啟用特定 Server：

```json
{
  "mcpServers": {
    "code-tools": {
      "command": "...",
      "enabledWhen": {
        "channels": ["cli", "discord"],
        "users": ["discord:123456789"]
      }
    }
  }
}
```

---

## 5. 權限與安全

### 5.1 controlPolicy 設定

OpenClaw 的 `controlPolicy` 讓你精細控制每個 MCP 工具的使用權限：

```json
{
  "mcpServers": {
    "filesystem": {
      "controlPolicy": {
        "default": "confirm",
        "overrides": {
          "read_file": "allow",
          "list_directory": "allow",
          "write_file": "confirm",
          "delete_file": "deny",
          "move_file": "confirm"
        }
      }
    }
  }
}
```

三種策略：

| 策略 | 說明 |
|------|------|
| `allow` | 自動允許，無需確認 |
| `confirm` | 需要使用者確認後才執行 |
| `deny` | 完全禁止使用 |

### 5.2 工具級別權限

更細粒度的權限控制：

```json
{
  "mcpServers": {
    "github": {
      "controlPolicy": {
        "default": "allow",
        "overrides": {
          "create_issue": {
            "policy": "confirm",
            "message": "確定要在 {repo} 建立 Issue: {title}?"
          },
          "delete_repository": {
            "policy": "deny",
            "reason": "禁止通過 AI 刪除倉庫"
          }
        }
      }
    }
  }
}
```

### 5.3 使用者確認機制

當工具策略為 `confirm` 時，OpenClaw 會在對應通道中向使用者確認：

```
Discord 中的確認：
  EasonClaw: 我需要在 eason/my-project 建立一個 Issue：
  標題：修復登入 bug
  
  ⚠️ 確認執行？
  ✅ 確認 | ❌ 取消

CLI 中的確認：
  [CONFIRM] 執行 create_issue on github:
  repo: eason/my-project
  title: 修復登入 bug
  
  Proceed? [y/N]: _
```

---

## 6. OpenClaw 作為 MCP Server

### 6.1 啟動 MCP Server 模式

OpenClaw 可以作為 MCP Server 運行，讓其他 MCP Client 使用 OpenClaw 的能力：

```bash
# 啟動 OpenClaw MCP Server（stdio 模式）
openclaw mcp serve

# 啟動 OpenClaw MCP Server（HTTP 模式）
openclaw mcp serve --transport http --port 3100
```

### 6.2 暴露技能為 MCP Tools

OpenClaw 的技能會自動轉換為 MCP Tools：

```json
{
  "mcpServer": {
    "enabled": true,
    "exposedSkills": [
      "weather-query",
      "translate",
      "summarize"
    ],
    "transport": "stdio"
  }
}
```

轉換範例：

```
SKILL.md（OpenClaw 技能）:
  name: weather-query
  description: 查詢天氣
  parameters:
    city: 城市名稱

自動轉換為 MCP Tool:
{
  "name": "weather-query",
  "description": "查詢天氣",
  "inputSchema": {
    "type": "object",
    "properties": {
      "city": { "type": "string", "description": "城市名稱" }
    }
  }
}
```

### 6.3 暴露記憶為 MCP Resources

OpenClaw 的對話記憶可以作為 MCP Resources 暴露：

```json
{
  "mcpServer": {
    "resources": {
      "memory": {
        "enabled": true,
        "scope": "current-session"
      },
      "persona": {
        "enabled": true
      }
    }
  }
}
```

### 6.4 安全控制

作為 MCP Server 時的安全設定：

```json
{
  "mcpServer": {
    "security": {
      "allowedClients": ["copilot-cli", "claude-desktop"],
      "maxRequestsPerMinute": 60,
      "requireAuthentication": false,
      "readOnlyMode": false
    }
  }
}
```

---

## 7. 常用 MCP Server 配置

### 7.1 GitHub MCP Server

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

提供的工具：`search_repositories`, `create_issue`, `get_file_contents`, `push_files`, `create_pull_request` 等。

### 7.2 Filesystem MCP Server

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y", "@modelcontextprotocol/server-filesystem",
        "/home/eason/projects",
        "/home/eason/documents"
      ]
    }
  }
}
```

提供的工具：`read_file`, `write_file`, `list_directory`, `create_directory`, `move_file`, `search_files` 等。

### 7.3 Brave Search MCP Server

```json
{
  "mcpServers": {
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

提供的工具：`brave_web_search`, `brave_local_search`。

### 7.4 PostgreSQL MCP Server

```json
{
  "mcpServers": {
    "postgres": {
      "command": "python",
      "args": ["-m", "mcp_server_postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

提供的工具：`query`, `list_tables`, `describe_table` 等。

---

## 8. 除錯與維護

### 8.1 連線除錯

```bash
# 檢查所有 MCP Server 的連線狀態
openclaw mcp status

# 輸出：
# ┌─────────────┬──────────┬───────┬──────────┐
# │ Server      │ Status   │ Tools │ Latency  │
# ├─────────────┼──────────┼───────┼──────────┤
# │ github      │ ✅ Ready │ 12    │ 45ms     │
# │ filesystem  │ ✅ Ready │ 8     │ 12ms     │
# │ brave-search│ ❌ Error │ 0     │ —        │
# └─────────────┴──────────┴───────┴──────────┘

# 查看特定 Server 的詳細資訊
openclaw mcp inspect github

# 查看 Server 提供的工具列表
openclaw mcp tools github
```

### 8.2 工具測試

```bash
# 直接測試 MCP 工具
openclaw mcp call github search_repositories '{"query": "openclaw"}'

# 互動式測試
openclaw mcp test github
```

### 8.3 日誌與監控

```bash
# 查看 MCP 通訊日誌
openclaw logs --filter mcp

# 啟用詳細日誌
openclaw mcp --debug

# 查看 JSON-RPC 通訊內容
openclaw mcp --trace
```

---

## 9. 效能調校

```json
{
  "mcp": {
    "performance": {
      "connectionTimeout": 10000,
      "requestTimeout": 30000,
      "maxConcurrentRequests": 5,
      "keepAlive": true,
      "lazyConnect": true,
      "preloadTools": true
    }
  }
}
```

| 參數 | 說明 | 建議值 |
|------|------|--------|
| `connectionTimeout` | 連線超時（ms） | 10000 |
| `requestTimeout` | 請求超時（ms） | 30000 |
| `maxConcurrentRequests` | 最大並行請求數 | 5 |
| `lazyConnect` | 延遲連線（首次使用時才連線） | true |
| `preloadTools` | 預先載入工具列表 | true |

---

## 10. 常見問題

**Q: MCP Server 啟動失敗怎麼辦？**

A:
1. 確認命令可以在終端中直接執行
2. 確認所需的環境變數已設定
3. 查看日誌：`openclaw logs --filter mcp`
4. 嘗試手動啟動 Server 確認是否正常

**Q: 工具呼叫超時怎麼辦？**

A:
1. 增加 `requestTimeout`
2. 檢查網路連線
3. 確認 MCP Server 運行正常

**Q: 如何知道有哪些 MCP Server 可用？**

A:
1. 官方列表：[modelcontextprotocol.io](https://modelcontextprotocol.io)
2. ClawHub：`openclaw hub search --type mcp-server`
3. GitHub：搜尋 "mcp-server" topic

**Q: OpenClaw 的技能和 MCP 工具有什麼區別？**

A:
- **技能（Skills）**：OpenClaw 原生的能力擴展，使用 SKILL.md 定義，更適合需要 LLM 推理的任務
- **MCP 工具**：標準化的外部工具，可被任何 MCP Client 使用，更適合確定性的操作（如 API 呼叫）
- 兩者可以共存，OpenClaw 會統一管理
