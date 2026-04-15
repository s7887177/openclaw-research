# Copilot CLI 整合藍圖

> **本章摘要**：本藍圖基於 OpenClaw v2026.4.12 的完整原始碼分析，設計將 OpenClaw 接上 GitHub Copilot CLI 的可行技術方案。我們識別出五條整合路徑——MCP Server、ACP Agent、copilot-proxy 擴展、Gateway 橋接器、Shell CLI 橋接——並針對每條路徑從原始碼層級評估可行性。最終推薦「MCP Server + Shell CLI 雙軌策略」，提供從 MVP 到完整整合的三階段實作路線圖。

---

## 本章定位

這是整個研究專案中最核心的應用文獻之一。Eason 的首要目標是**讓 OpenClaw 接上 GitHub Copilot CLI**，使兩個系統的能力互補：

- **OpenClaw** 提供：多通道訊息能力、記憶系統、人格引擎、Skills 生態、多 Agent 協作
- **Copilot CLI** 提供：程式碼智能、檔案系統操作、Git 整合、終端自動化

整合後的效果：在 Copilot CLI 中可以直接與 OpenClaw 的各通道對話、查詢記憶、使用 Skills；同時 OpenClaw 的 Agent 也能呼叫 Copilot 的程式碼能力。

---

## 章節索引

| 檔案 | 內容 | 字數估計 |
|------|------|----------|
| [01-目標定義與技術路徑分析.md](01-目標定義與技術路徑分析.md) | 需求分析 + 五條技術路徑的深度評估（含事件佇列、Gateway 協議、CLI 命令映射） | ~11,000 |
| [02-推薦方案與實作路線圖.md](02-推薦方案與實作路線圖.md) | 最佳路徑推薦、三階段實作、風險評估、FAQ、Discord 機器人關聯 | ~9,000 |

---

## 核心結論預覽

### 路徑比較總表

| 路徑 | 方案 | 可行性 | 複雜度 | 功能完整度 | 維護成本 | 推薦度 |
|------|------|--------|--------|------------|----------|--------|
| A | OpenClaw MCP Server | ★★★★★ | 低 | 高 | 低 | **首選** |
| B | ACP Agent | ★★★☆☆ | 中 | 中高 | 中 | 備選 |
| C | copilot-proxy 擴展 | ★★☆☆☆ | 高 | 低 | 高 | 不推薦 |
| D | Gateway 橋接器 | ★★★★☆ | 中高 | 最高 | 中高 | 進階 |
| E | Shell CLI 橋接 | ★★★★★ | 最低 | 中 | 最低 | **MVP 首選** |

### 推薦策略：雙軌漸進

1. **Phase 1（MVP, 1-2 天）**：路徑 E — `openclaw` CLI 命令直接作為 Copilot CLI 的 shell 工具
2. **Phase 2（增強, 1 週）**：路徑 A — `openclaw mcp serve` 作為 MCP Server 接入 Copilot CLI
3. **Phase 3（完整, 2-4 週）**：路徑 D — 自訂 MCP Server 包裝 Gateway WebSocket API

---

## 關鍵技術發現

1. **`openclaw mcp serve` 已經是完整的 MCP Server**（`source-repo/src/mcp/channel-server.ts`），使用 `@modelcontextprotocol/sdk` 的 stdio transport，暴露 8 個 MCP tools
2. **`plugin-tools-serve.ts` 可暴露所有 plugin 工具為 MCP**（如 memory_recall、memory_store），這是額外的整合點
3. **copilot-proxy extension 方向相反**——它讓 OpenClaw 使用 Copilot 的模型，不是讓 Copilot 使用 OpenClaw
4. **Gateway API 有完整的 method 體系**——60+ 個 WebSocket methods，涵蓋 sessions、chat、agents、skills、memory 等全部能力
5. **Copilot CLI 原生支援 MCP**——可透過 `~/.copilot/mcp-config.json` 或 `.mcp.json` 設定 MCP servers

---

## 引用來源

| 來源 | 路徑 | 用途 |
|------|------|------|
| MCP Channel Server | `source-repo/src/mcp/channel-server.ts` | MCP Server 核心實作 |
| MCP Channel Bridge | `source-repo/src/mcp/channel-bridge.ts` | Gateway 橋接邏輯 |
| MCP Channel Tools | `source-repo/src/mcp/channel-tools.ts` | 8 個 MCP tool 定義 |
| Plugin Tools Serve | `source-repo/src/mcp/plugin-tools-serve.ts` | Plugin 工具 MCP 暴露 |
| Copilot Proxy Extension | `source-repo/extensions/copilot-proxy/` | 反向整合（Copilot → OpenClaw） |
| ACP Bridge | `source-repo/docs.acp.md` | ACP 協議橋接文件 |
| ACP Server | `source-repo/src/acp/server.ts` | ACP 實作 |
| Gateway Methods | `source-repo/src/gateway/server-methods.ts` | 全部 Gateway API handlers |
| Gateway Scopes | `source-repo/src/gateway/method-scopes.ts` | API 權限模型 |
| MCP CLI | `source-repo/src/cli/mcp-cli.ts` | CLI 入口點 |
| MCP HTTP Loopback | `source-repo/src/gateway/mcp-http.ts` | HTTP MCP 內部 loopback |
| VISION.md | `source-repo/VISION.md` | 專案願景與 MCP 立場 |
| mcporter Skill | `source-repo/skills/mcporter/SKILL.md` | MCP 工具管理 |
