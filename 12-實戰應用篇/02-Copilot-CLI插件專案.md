# 實戰專案：Copilot CLI 插件整合

> **摘要**：本文探討如何將 OpenClaw 與 GitHub Copilot CLI 整合，透過 MCP（Model Context Protocol）協議讓 Copilot CLI 能調用 OpenClaw 的能力。涵蓋三種整合方式、完整實作指南、以及進階使用場景。

---

## 目錄

1. [整合概覽](#1-整合概覽)
2. [MCP 協議基礎](#2-mcp-協議基礎)
3. [方式一：OpenClaw 作為 MCP Server](#3-方式一openclaw-作為-mcp-server)
4. [方式二：Copilot CLI 作為 OpenClaw 通道](#4-方式二copilot-cli-作為-openclaw-通道)
5. [方式三：雙向橋接](#5-方式三雙向橋接)
6. [MCP Server 實作](#6-mcp-server-實作)
7. [工具定義與註冊](#7-工具定義與註冊)
8. [記憶整合](#8-記憶整合)
9. [安全考量](#9-安全考量)
10. [進階場景](#10-進階場景)
11. [配置範例](#11-配置範例)
12. [測試與除錯](#12-測試與除錯)
13. [部署指南](#13-部署指南)

---

## 1. 整合概覽

### 1.1 為什麼整合？

```
OpenClaw 的優勢：
├── 持久記憶（跨 Session）
├── 個性化人格（SOUL.md）
├── 豐富的自訂技能
├── 多通道支援
└── 完整的安全模型

Copilot CLI 的優勢：
├── 終端原生體驗
├── 程式碼上下文感知
├── Git 整合
├── 強大的程式碼生成
└── 無需額外基礎設施

整合目標：在 Copilot CLI 中使用 OpenClaw 的記憶、人格和技能
```

### 1.2 三種整合方式

```
方式一：OpenClaw 作為 MCP Server
  Copilot CLI → MCP → OpenClaw Gateway
  ↳ Copilot 透過 MCP 調用 OpenClaw 的記憶和技能

方式二：Copilot CLI 作為 OpenClaw 通道
  OpenClaw Gateway → CLI Channel Adapter → Terminal
  ↳ 在終端中使用 OpenClaw（類似 Discord Channel）

方式三：雙向橋接
  Copilot CLI ←→ MCP ←→ OpenClaw
  ↳ 雙方互相調用，能力互補
```

---

## 2. MCP 協議基礎

### 2.1 什麼是 MCP

```
MCP（Model Context Protocol）是一個讓 AI 助手與外部工具溝通的標準協議。

架構：
  ┌──────────────┐     MCP Protocol     ┌──────────────┐
  │  MCP Client  │ ←────────────────→  │  MCP Server  │
  │  (Copilot    │    JSON-RPC 2.0     │  (OpenClaw)  │
  │   CLI)       │    over stdio/SSE   │              │
  └──────────────┘                     └──────────────┘

MCP 支援三種能力：
  1. Tools（工具）     → 讓 Client 調用 Server 的功能
  2. Resources（資源） → 讓 Client 讀取 Server 的資料
  3. Prompts（提示）   → 讓 Server 提供提示模板
```

### 2.2 MCP 傳輸方式

```
1. stdio（標準輸入/輸出）
   → 適合本地 MCP Server
   → Copilot CLI 預設使用
   → 啟動 Server 作為子進程

2. SSE（Server-Sent Events）
   → 適合遠端 MCP Server
   → 基於 HTTP，可穿越防火牆
   → 支援雲端部署

3. Streamable HTTP
   → 最新的傳輸方式
   → 單一 HTTP 端點
   → 支援雙向通訊
```

---

## 3. 方式一：OpenClaw 作為 MCP Server

### 3.1 架構

```
┌──────────────────────────────────┐
│  Terminal                        │
│                                  │
│  Copilot CLI                     │
│  ┌────────────────────────────┐  │
│  │  MCP Client                │  │
│  │  ├── discover tools        │  │
│  │  ├── call tools            │  │
│  │  └── read resources        │  │
│  └────────┬───────────────────┘  │
│           │ stdio / SSE          │
│           ▼                      │
│  ┌────────────────────────────┐  │
│  │  OpenClaw MCP Server       │  │
│  │  ├── memory_search         │  │
│  │  ├── memory_write          │  │
│  │  ├── persona_chat          │  │
│  │  ├── skill_execute         │  │
│  │  └── context_get           │  │
│  └────────┬───────────────────┘  │
│           │                      │
│           ▼                      │
│  ┌────────────────────────────┐  │
│  │  OpenClaw Gateway          │  │
│  │  ├── Memory Store          │  │
│  │  ├── Skills Engine         │  │
│  │  └── Inference Engine      │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
```

### 3.2 Copilot CLI MCP 配置

```json5
// ~/.copilot/mcp-config.json（Copilot CLI 的 MCP 配置）
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw",
      "args": ["mcp", "serve", "--stdio"],
      "env": {
        "OPENCLAW_CONFIG": "/path/to/openclaw.json5"
      }
    }
  }
}
```

```json5
// 或者使用 SSE 連接到遠端 Gateway
{
  "mcpServers": {
    "openclaw-remote": {
      "url": "http://localhost:3000/mcp/sse",
      "headers": {
        "Authorization": "Bearer ${OPENCLAW_API_KEY}"
      }
    }
  }
}
```

---

## 4. 方式二：Copilot CLI 作為 OpenClaw 通道

### 4.1 架構

```
使用者在終端輸入
    │
    ▼
┌──────────────────────────────────┐
│  CLI Channel Adapter             │
│  ├── 解析終端輸入                │
│  ├── 支援 Rich Text 輸出        │
│  ├── 處理 Tab 補全               │
│  └── 歷史記錄管理                │
│           │                      │
│           ▼                      │
│  ┌────────────────────────────┐  │
│  │  OpenClaw Gateway          │  │
│  │  ├── 使用完整的人格         │  │
│  │  ├── 存取所有記憶           │  │
│  │  ├── 調用所有技能           │  │
│  │  └── 九層權限保護           │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
```

### 4.2 CLI 通道實作

```typescript
// src/channels/cli-adapter.ts
import * as readline from "readline";

export class CLIChannelAdapter {
  private rl: readline.Interface;
  private gateway: OpenClawGateway;
  private sessionId: string;

  constructor(gateway: OpenClawGateway) {
    this.gateway = gateway;
    this.sessionId = `cli-${Date.now()}`;
    this.rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout,
      prompt: "🐾 > "
    });
  }

  async start(): Promise<void> {
    console.log("🐾 OpenClaw CLI Mode");
    console.log("Type your message or 'exit' to quit.\n");
    this.rl.prompt();

    this.rl.on("line", async (line) => {
      const input = line.trim();
      if (input === "exit") {
        console.log("Goodbye! 👋");
        process.exit(0);
      }

      if (!input) {
        this.rl.prompt();
        return;
      }

      // 送到 OpenClaw 推理
      const response = await this.gateway.chat({
        sessionId: this.sessionId,
        userId: "cli-user",
        message: input,
        channel: "cli"
      });

      // 輸出回應
      console.log(`\n${response.text}\n`);
      this.rl.prompt();
    });
  }
}
```

---

## 5. 方式三：雙向橋接

### 5.1 架構

```
┌────────────────────────────────────────────────────┐
│  雙向橋接架構                                       │
│                                                     │
│  Copilot CLI                 OpenClaw Gateway       │
│  ┌─────────────┐            ┌─────────────────┐    │
│  │ MCP Client  │───tools──→│ MCP Server      │    │
│  │             │            │ (記憶/技能)      │    │
│  │ MCP Server  │←──tools───│ MCP Client      │    │
│  │ (程式碼能力) │            │ (需要程式碼時)   │    │
│  └─────────────┘            └─────────────────┘    │
│                                                     │
│  Copilot 可以：               OpenClaw 可以：       │
│  ├── 搜尋記憶                ├── 分析程式碼          │
│  ├── 寫入記憶                ├── 搜尋檔案            │
│  ├── 使用人格對話            ├── 執行指令            │
│  └── 調用自訂技能            └── 產生程式碼          │
└────────────────────────────────────────────────────┘
```

---

## 6. MCP Server 實作

### 6.1 入口點

```typescript
// src/mcp/server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "openclaw",
  version: "1.0.0",
  description: "OpenClaw AI Agent Framework - Memory, Skills, and Persona"
});

// === Tools ===

// 記憶搜尋
server.tool(
  "memory_search",
  "Search through OpenClaw's long-term memory for relevant information",
  {
    query: z.string().describe("Search query"),
    limit: z.number().optional().default(10).describe("Max results"),
    userId: z.string().optional().describe("Filter by user ID")
  },
  async ({ query, limit, userId }) => {
    const results = await gateway.memoryStore.search({
      query,
      limit,
      userId
    });

    return {
      content: [{
        type: "text",
        text: JSON.stringify(results, null, 2)
      }]
    };
  }
);

// 記憶寫入
server.tool(
  "memory_write",
  "Save information to OpenClaw's long-term memory",
  {
    content: z.string().describe("Content to remember"),
    category: z.enum(["fact", "preference", "conversation", "task"])
      .describe("Memory category"),
    importance: z.number().min(1).max(10).optional().default(5)
  },
  async ({ content, category, importance }) => {
    const id = await gateway.memoryStore.write({
      content,
      category,
      importance,
      source: "copilot-cli",
      timestamp: Date.now()
    });

    return {
      content: [{
        type: "text",
        text: `Memory saved (ID: ${id})`
      }]
    };
  }
);

// 人格對話
server.tool(
  "persona_chat",
  "Chat with OpenClaw using its configured personality (SOUL.md)",
  {
    message: z.string().describe("Message to send"),
    agentId: z.string().optional().default("default")
  },
  async ({ message, agentId }) => {
    const response = await gateway.chat({
      sessionId: `mcp-${Date.now()}`,
      userId: "copilot-cli",
      message,
      agentId
    });

    return {
      content: [{
        type: "text",
        text: response.text
      }]
    };
  }
);

// 技能執行
server.tool(
  "skill_execute",
  "Execute an OpenClaw skill (weather, reminder, translate, etc.)",
  {
    skill: z.string().describe("Skill name"),
    args: z.record(z.any()).describe("Skill arguments")
  },
  async ({ skill, args }) => {
    const result = await gateway.skillEngine.execute(skill, args);

    return {
      content: [{
        type: "text",
        text: JSON.stringify(result, null, 2)
      }]
    };
  }
);

// === Resources ===

// SOUL.md（人格定義）
server.resource(
  "persona",
  "openclaw://persona/{agentId}",
  async (uri) => {
    const agentId = uri.pathname.split("/").pop() || "default";
    const soul = await gateway.loadPersona(agentId);

    return {
      contents: [{
        uri: uri.href,
        mimeType: "text/markdown",
        text: soul
      }]
    };
  }
);

// 記憶摘要
server.resource(
  "memory-summary",
  "openclaw://memory/summary",
  async () => {
    const summary = await gateway.memoryStore.getSummary();

    return {
      contents: [{
        uri: "openclaw://memory/summary",
        mimeType: "application/json",
        text: JSON.stringify(summary, null, 2)
      }]
    };
  }
);

// === Prompts ===

server.prompt(
  "with-memory",
  "Chat with context from OpenClaw's memory about a topic",
  {
    topic: z.string().describe("Topic to recall memories about")
  },
  async ({ topic }) => {
    const memories = await gateway.memoryStore.search({
      query: topic,
      limit: 5
    });

    const memoryContext = memories.map(m => `- ${m.content}`).join("\n");

    return {
      messages: [{
        role: "user",
        content: {
          type: "text",
          text: `Based on what I know:\n${memoryContext}\n\nPlease help me with: ${topic}`
        }
      }]
    };
  }
);

// === 啟動 ===
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("OpenClaw MCP Server running on stdio");
}

main().catch(console.error);
```

---

## 7. 工具定義與註冊

### 7.1 完整工具清單

```
OpenClaw MCP Server 提供的工具：

記憶類：
├── memory_search      搜尋記憶
├── memory_write       寫入記憶
├── memory_delete      刪除記憶
└── memory_summary     記憶摘要

對話類：
├── persona_chat       人格對話（使用 SOUL.md）
└── context_get        獲取當前上下文

技能類：
├── skill_execute      執行技能（通用）
├── weather_query      天氣查詢
├── translate          翻譯
├── remind_set         設定提醒
└── calendar_read      讀取行事曆

管理類：
├── agent_list         列出 Agent
├── agent_info         Agent 資訊
└── session_list       列出 Session
```

---

## 8. 記憶整合

### 8.1 跨平台記憶共享

```
OpenClaw 的記憶是跨通道共享的：

  Discord 寫入的記憶
       ↓
  ┌──────────────────┐
  │  Memory Store    │ ← 統一儲存
  └──────────────────┘
       ↑
  Copilot CLI 可以讀取

這意味著：
  1. 在 Discord 語音中提到「我喜歡 TypeScript」
  2. 之後在 Copilot CLI 中，Agent 會記得這個偏好
  3. 在 Copilot CLI 中學到的新資訊，Discord Bot 也能存取
```

### 8.2 記憶同步策略

```json5
{
  memory: {
    sync: {
      // 所有通道共享同一個記憶儲存
      shared_store: true,

      // 但某些記憶可以標記為通道專屬
      channel_scoped: {
        enabled: true,
        // 程式碼相關記憶只在 CLI 通道可見
        scopes: {
          "copilot-cli": ["code", "project"],
          "discord": ["social", "preference"],
          "all": ["fact", "personal"]  // 所有通道可見
        }
      }
    }
  }
}
```

---

## 9. 安全考量

### 9.1 MCP Server 安全

```
MCP Server 安全清單：

1. 認證
   ├── stdio 模式：依賴作業系統層級的進程安全
   └── SSE 模式：必須使用 API Key 或 JWT

2. 授權
   ├── MCP Client 只能存取授權的工具
   └── 不同的 MCP Client 可以有不同的權限

3. 輸入驗證
   ├── 所有工具參數使用 Zod schema 驗證
   └── 防止注入攻擊

4. 速率限制
   ├── 每分鐘最大請求數
   └── 每小時最大 token 數

5. 日誌
   └── 記錄所有 MCP 工具調用
```

### 9.2 權限矩陣

```
MCP Client 權限矩陣：

┌────────────────┬──────────┬──────────┬──────────┐
│ 工具           │ Copilot  │ 自訂CLI  │ 外部     │
│                │ CLI      │ Client   │ Client   │
├────────────────┼──────────┼──────────┼──────────┤
│ memory_search  │ ✅       │ ✅       │ ❌       │
│ memory_write   │ ✅       │ ✅       │ ❌       │
│ memory_delete  │ ❌       │ ✅       │ ❌       │
│ persona_chat   │ ✅       │ ✅       │ ✅       │
│ skill_execute  │ ✅       │ ✅       │ ⚠️ 受限  │
│ agent_list     │ ✅       │ ✅       │ ❌       │
│ session_list   │ ❌       │ ✅       │ ❌       │
└────────────────┴──────────┴──────────┴──────────┘
```

---

## 10. 進階場景

### 10.1 場景：程式碼審查結合記憶

```
使用者在 Copilot CLI 中請求程式碼審查：

1. Copilot CLI 分析程式碼
2. 調用 memory_search 查詢：
   → 「這個專案之前有什麼已知問題？」
   → 「使用者偏好的程式碼風格是什麼？」
3. 結合記憶和程式碼分析給出建議
4. 調用 memory_write 記住本次審查的發現
```

### 10.2 場景：開發日記自動化

```
每次 git commit 後自動記錄：

1. Git hook 觸發
2. 調用 OpenClaw MCP：
   → memory_write: 記錄今天的開發內容
   → persona_chat: 請 Agent 生成開發日記
3. 下次開發時：
   → memory_search: 回顧之前的進度
   → persona_chat: Agent 提供上下文和建議
```

### 10.3 場景：跨通道任務管理

```
在 Discord 語音中說：「小夜，提醒我明天修改 API 的錯誤處理」

1. Discord Bot 記憶寫入：
   → 記住這個任務

2. 隔天在 Copilot CLI 中：
   → Copilot 調用 memory_search
   → 發現待辦事項
   → 提示：「你昨天說要修改 API 的錯誤處理，要現在開始嗎？」
```

---

## 11. 配置範例

### 11.1 Copilot CLI MCP 配置

```json5
// ~/.copilot/mcp-config.json
{
  "mcpServers": {
    "openclaw": {
      "command": "node",
      "args": ["/path/to/openclaw-mcp-server/dist/index.js"],
      "env": {
        "OPENCLAW_CONFIG": "/path/to/openclaw.json5",
        "OPENCLAW_MCP_LOG_LEVEL": "info"
      }
    }
  }
}
```

### 11.2 OpenClaw MCP 模組配置

```json5
// openclaw.json5 中的 MCP 設定
{
  mcp: {
    server: {
      enabled: true,
      transport: "stdio",  // stdio | sse

      // SSE 設定（如果 transport = "sse"）
      sse: {
        port: 3001,
        path: "/mcp/sse",
        auth: {
          type: "bearer",
          token: "${OPENCLAW_MCP_TOKEN}"
        }
      },

      // 公開哪些工具
      exposed_tools: [
        "memory_search",
        "memory_write",
        "persona_chat",
        "skill_execute",
        "weather_query",
        "translate"
      ],

      // 公開哪些資源
      exposed_resources: [
        "persona",
        "memory-summary"
      ]
    }
  }
}
```

---

## 12. 測試與除錯

### 12.1 測試 MCP Server

```bash
# 使用 MCP Inspector 測試
npx @modelcontextprotocol/inspector openclaw mcp serve --stdio

# 或直接用 stdio 測試
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | openclaw mcp serve --stdio

# 測試特定工具
echo '{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "memory_search",
    "arguments": {"query": "TypeScript", "limit": 5}
  }
}' | openclaw mcp serve --stdio
```

### 12.2 除錯日誌

```bash
# 啟用 MCP 除錯日誌
OPENCLAW_MCP_LOG_LEVEL=debug openclaw mcp serve --stdio 2>mcp-debug.log

# 查看日誌
tail -f mcp-debug.log
```

---

## 13. 部署指南

### 13.1 本地部署

```bash
# 1. 安裝 OpenClaw MCP Server
npm install -g @openclaw/mcp-server

# 2. 配置 Copilot CLI
cat > ~/.copilot/mcp-config.json << 'EOF'
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw-mcp-server",
      "env": {
        "OPENCLAW_CONFIG": "~/openclaw.json5"
      }
    }
  }
}
EOF

# 3. 驗證
# 在 Copilot CLI 中，OpenClaw 的工具應該自動可用
```

### 13.2 遠端部署

```bash
# 在伺服器上啟動 OpenClaw Gateway + MCP SSE
openclaw gateway start --mcp-sse --mcp-port 3001

# 在本地 Copilot CLI 配置
cat > ~/.copilot/mcp-config.json << 'EOF'
{
  "mcpServers": {
    "openclaw-remote": {
      "url": "https://my-server.com:3001/mcp/sse",
      "headers": {
        "Authorization": "Bearer my-secret-token"
      }
    }
  }
}
EOF
```

---

> **下一章**：[個人 AI 助手配置範例](./03-個人AI助手配置範例.md)
