# Gateway 子章節 03：WebSocket 協議

> **引用範圍**：`src/gateway/protocol/`, `server-methods-list.ts`, `server-methods.ts`, `server-ws-runtime.ts`, `ws-log.ts`

## 1. 協議總覽

Gateway 的 WebSocket 通訊基於**JSON-RPC 風格的請求/回應協議**。每個連線建立後，客戶端可以：
- **發送方法呼叫**（request → response）
- **接收伺服器事件**（server → client push）

協議的 schema 定義使用 **JSON Schema + AJV** 驗證，集中管理在 `src/gateway/protocol/` 目錄。

---

## 2. Protocol Schema 目錄結構

```
src/gateway/protocol/
├── index.ts                      # 中央協議 schema 註冊
├── schema.ts                     # Schema 彙總匯出
├── client-info.ts                # 客戶端中繼資料
├── connect-error-details.ts      # 連線錯誤細節
├── schema/                       # 分領域 Schema 定義
│   ├── agent.ts                  # Agent 訊息類型
│   ├── agents-models-skills.ts   # Agent/Model/Skill 管理
│   ├── channels.ts               # Channel 操作
│   ├── commands.ts               # 指令列表
│   ├── config.ts                 # 配置操作
│   ├── cron.ts                   # 排程任務
│   ├── devices.ts                # 裝置配對
│   ├── error-codes.ts            # 錯誤碼定義
│   ├── exec-approvals.ts         # 執行核准
│   ├── frames.ts                 # Gateway/Event 框架
│   ├── logs-chat.ts              # 日誌與聊天
│   ├── nodes.ts                  # 節點互動
│   ├── plugin-approvals.ts       # Plugin 核准
│   ├── primitives.ts             # 基本類型
│   ├── protocol-schemas.ts       # 協議 schema 工具
│   ├── push.ts                   # 推送通知
│   ├── secrets.ts                # 密鑰管理
│   ├── sessions.ts               # Session 操作
│   ├── snapshot.ts               # 快照
│   ├── types.ts                  # 共用類型
│   └── wizard.ts                 # Setup Wizard
```

> **引用來源**：`source-repo/src/gateway/protocol/` 目錄結構

### 2.1 Protocol Index（protocol/index.ts）

`protocol/index.ts` 是協議的中央註冊點，彙整所有 schema 定義並用 AJV 進行驗證：

```typescript
// source-repo/src/gateway/protocol/index.ts:1-4 (摘要)
import AjvPkg, { type ErrorObject } from "ajv";
import { AgentEventSchema, AgentParamsSchema, ... } from "./schema/agent.js";
import { SessionsPatchResult } from "../session-utils.types.js";
// ... 匯入所有 schema
```

每個 schema 都使用 JSON Schema 格式定義，並透過 AJV 編譯為高效的驗證函式。

> **引用來源**：`source-repo/src/gateway/protocol/index.ts:1-60`

### 2.2 客戶端資訊（client-info.ts）

定義支援的客戶端識別碼和模式：

```
GATEWAY_CLIENT_IDS    — 支援的客戶端標識符
GATEWAY_CLIENT_MODES  — 客戶端模式常數
PROTOCOL_VERSION      — 當前協議版本號
```

> **引用來源**：`source-repo/src/gateway/protocol/client-info.ts`

---

## 3. WebSocket 方法清單（134 個方法）

所有 WS 方法定義在 `server-methods-list.ts` 的 `BASE_METHODS` 陣列中：

```typescript
// source-repo/src/gateway/server-methods-list.ts:4-134
const BASE_METHODS = [
  "health",
  // ... 134 個方法
];
```

以下按功能領域分類：

### 3.1 系統與健康

| 方法 | 說明 |
|------|------|
| `health` | 健康狀態查詢 |
| `status` | 系統狀態 |
| `system-presence` | 系統在線狀態 |
| `system-event` | 系統事件 |

### 3.2 配置管理

| 方法 | 說明 |
|------|------|
| `config.get` | 讀取配置 |
| `config.set` | 設定配置值 |
| `config.apply` | 套用完整配置 |
| `config.patch` | 部分更新配置 |
| `config.schema` | 配置 schema 查詢 |
| `config.schema.lookup` | Schema 欄位查詢 |

### 3.3 Session 管理

| 方法 | 說明 |
|------|------|
| `sessions.list` | 列出 Session |
| `sessions.create` | 建立 Session |
| `sessions.send` | 發送訊息 |
| `sessions.abort` | 中止 Session |
| `sessions.patch` | 更新 Session |
| `sessions.reset` | 重設 Session |
| `sessions.delete` | 刪除 Session |
| `sessions.compact` | 壓縮 Session |
| `sessions.subscribe` | 訂閱 Session 變更 |
| `sessions.unsubscribe` | 取消訂閱 |
| `sessions.messages.subscribe` | 訂閱訊息 |
| `sessions.messages.unsubscribe` | 取消訊息訂閱 |
| `sessions.preview` | 預覽 Session |
| `sessions.compaction.*` | 壓縮管理（list/get/branch/restore） |

### 3.4 Agent 管理

| 方法 | 說明 |
|------|------|
| `agents.list` | 列出 Agent |
| `agents.create` | 建立 Agent |
| `agents.update` | 更新 Agent |
| `agents.delete` | 刪除 Agent |
| `agents.files.list` | Agent 檔案列表 |
| `agents.files.get` | 讀取 Agent 檔案 |
| `agents.files.set` | 寫入 Agent 檔案 |
| `agent` | Agent 訊息 |
| `agent.identity.get` | Agent 身份 |
| `agent.wait` | 等待 Agent 完成 |

### 3.5 聊天（WebChat）

| 方法 | 說明 |
|------|------|
| `chat.send` | 發送聊天訊息 |
| `chat.abort` | 中止聊天 |
| `chat.history` | 聊天歷史 |
| `send` | 通用發送 |

### 3.6 節點（Node）操作

| 方法 | 說明 |
|------|------|
| `node.list` | 列出節點 |
| `node.describe` | 節點描述 |
| `node.rename` | 重新命名 |
| `node.invoke` | 調用節點 |
| `node.event` | 節點事件 |
| `node.pending.drain` | 排空待處理 |
| `node.pending.enqueue` | 加入佇列 |
| `node.pending.pull` | 拉取待處理 |
| `node.pending.ack` | 確認待處理 |
| `node.invoke.result` | 調用結果 |
| `node.canvas.capability.refresh` | Canvas 能力刷新 |

### 3.7 配對（Pairing）

| 方法 | 說明 |
|------|------|
| `node.pair.request` | 請求節點配對 |
| `node.pair.list` | 列出配對請求 |
| `node.pair.approve` | 核准配對 |
| `node.pair.reject` | 拒絕配對 |
| `node.pair.verify` | 驗證配對 |
| `device.pair.list` | 列出裝置配對 |
| `device.pair.approve` | 核准裝置配對 |
| `device.pair.reject` | 拒絕裝置配對 |
| `device.pair.remove` | 移除裝置 |
| `device.token.rotate` | 輪換裝置 Token |
| `device.token.revoke` | 撤銷裝置 Token |

### 3.8 核准（Approval）

| 方法 | 說明 |
|------|------|
| `exec.approvals.get/set` | 全域核准設定 |
| `exec.approval.get/list/request/waitDecision/resolve` | 執行核准工作流 |
| `plugin.approval.list/request/waitDecision/resolve` | Plugin 核准工作流 |

### 3.9 其他功能

| 方法 | 說明 |
|------|------|
| `models.list` | 模型列表 |
| `tools.catalog` / `tools.effective` | 工具目錄 |
| `skills.*` | 技能管理（status/search/detail/bins/install/update） |
| `tts.*` | TTS 管理（status/providers/enable/disable/convert/setProvider） |
| `talk.*` | Talk 模式（config/speak/mode） |
| `cron.*` | 排程管理（list/status/add/update/remove/run/runs） |
| `wizard.*` | Setup Wizard（start/next/cancel/status） |
| `secrets.*` | 密鑰管理（reload/resolve） |
| `update.run` | 執行更新 |
| `commands.list` | 指令列表 |
| `doctor.memory.*` | 記憶體診斷 |
| `logs.tail` | 日誌追蹤 |
| `usage.*` | 用量統計 |
| `message.action` | 訊息動作 |

**動態方法擴充**：Channel Plugin 可以註冊額外的 WS 方法：

```typescript
// source-repo/src/gateway/server-methods-list.ts:136-139
export function listGatewayMethods(): string[] {
  const channelMethods = listChannelPlugins().flatMap((plugin) => plugin.gatewayMethods ?? []);
  return Array.from(new Set([...BASE_METHODS, ...channelMethods]));
}
```

> **引用來源**：`source-repo/src/gateway/server-methods-list.ts:4-139`

---

## 4. Gateway 事件（25 個事件）

伺服器推送給客戶端的事件類型：

```typescript
// source-repo/src/gateway/server-methods-list.ts:141-166
export const GATEWAY_EVENTS = [
  "connect.challenge",          // 連線挑戰
  "agent",                      // Agent 事件
  "chat",                       // 聊天事件
  "session.message",            // Session 訊息
  "session.tool",               // Session 工具調用
  "sessions.changed",           // Session 變更
  "presence",                   // 在線狀態
  "tick",                       // 心跳
  "talk.mode",                  // Talk 模式變更
  "shutdown",                   // 關閉通知
  "health",                     // 健康狀態
  "heartbeat",                  // 心跳
  "cron",                       // 排程事件
  "node.pair.requested",        // 節點配對請求
  "node.pair.resolved",         // 節點配對解決
  "node.invoke.request",        // 節點調用請求
  "device.pair.requested",      // 裝置配對請求
  "device.pair.resolved",       // 裝置配對解決
  "voicewake.changed",          // 語音喚醒變更
  "exec.approval.requested",    // 執行核准請求
  "exec.approval.resolved",     // 執行核准解決
  "plugin.approval.requested",  // Plugin 核准請求
  "plugin.approval.resolved",   // Plugin 核准解決
  "update.available",           // 更新可用
];
```

> **引用來源**：`source-repo/src/gateway/server-methods-list.ts:141-166`, `source-repo/src/gateway/events.ts:3`

---

## 5. 方法授權

每個 WS 方法呼叫都經過授權檢查：

```typescript
// source-repo/src/gateway/server-methods.ts:40-67 (概要)
export function authorizeGatewayMethod(params: {
  method: string;
  role: "operator" | "node" | "guest";
  operatorScopes?: string[];
}): { authorized: boolean; reason?: string }
```

**角色層級**：
- **operator**：完整存取權限
- **node**：節點操作（node.*, device.*）
- **guest**：有限讀取

**寫入方法限制**：

```typescript
// source-repo/src/gateway/server-methods.ts:39 (概要)
const CONTROL_PLANE_WRITE_METHODS = ["config.apply", "config.patch", "update.run"];
```

這三個方法受控制平面速率限制保護（每分鐘最多 3 次）。

> **引用來源**：`source-repo/src/gateway/server-methods.ts:39-67`

---

## 6. Handler 聚合

所有方法處理器聚合在 `coreGatewayHandlers` 物件中：

```typescript
// source-repo/src/gateway/server-methods.ts:69-100 (概要)
export const coreGatewayHandlers = {
  // 連線處理
  ...connectHandlers,
  // 日誌
  ...logsHandlers,
  // 語音喚醒
  ...voicewakeHandlers,
  // 健康
  ...healthHandlers,
  // Channel 操作
  ...channelsHandlers,
  // 聊天
  ...chatHandlers,
  // 指令
  ...commandsHandlers,
  // 排程
  ...cronHandlers,
  // 裝置
  ...deviceHandlers,
  // 診斷
  ...doctorHandlers,
  // 執行核准
  ...execApprovalHandlers,
  // 模型
  ...modelsHandlers,
  // 配置
  ...configHandlers,
  // Wizard
  ...wizardHandlers,
  // Talk
  ...talkHandlers,
  // 工具
  ...toolsCatalogHandlers,
  // TTS
  ...ttsHandlers,
  // 技能
  ...skillsHandlers,
  // Session
  ...sessionsHandlers,
  // 系統
  ...systemHandlers,
  // 更新
  ...updateHandlers,
  // 節點
  ...nodesHandlers,
  // 推送
  ...pushHandlers,
  // 發送
  ...sendHandlers,
  // 用量
  ...usageHandlers,
  // Agent
  ...agentsHandlers,
  // Web
  ...webHandlers,
};
```

> **引用來源**：`source-repo/src/gateway/server-methods.ts:69-100`

---

## 7. WebSocket 連線管理

`attachGatewayWsHandlers()` 是 WS 連線的入口：

```typescript
// source-repo/src/gateway/server-ws-runtime.ts:24-46
export function attachGatewayWsHandlers(params: {
  wss: WebSocketServer;
  clients: Set<GatewayClient>;
  preauthConnectionBudget: PreauthConnectionBudget;
  port: number;
  gatewayHost: string;
  canvasHostEnabled: boolean;
  resolvedAuth: ResolvedGatewayAuth;
  getResolvedAuth: () => ResolvedGatewayAuth;
  rateLimiter: AuthRateLimiter;
  browserRateLimiter: AuthRateLimiter;
  gatewayMethods: Record<string, GatewayMethodHandler>;
  events: EventEmitter;
  // ... 日誌、廣播等參數
}): void
```

此函式將連線處理委託給 `server/ws-connection.js` 中更底層的處理器。

> **引用來源**：`source-repo/src/gateway/server-ws-runtime.ts:24-46`

---

## 8. WebSocket 日誌系統

`ws-log.ts` 提供多層級的 WS 通訊日誌：

```typescript
// source-repo/src/gateway/ws-log.ts:11-12
const LOG_VALUE_LIMIT = 240;  // 值的最大顯示字元數
const UUID_RE = /.../ ;       // UUID 正規表示式（用於縮短 ID）
```

核心日誌函式：

| 函式 | 用途 | 行號 |
|------|------|------|
| `shouldLogWs()` | 檢查是否啟用 WS 日誌 | 89-91 |
| `shortId()` | UUID 縮短（前 8 碼） | 93-102 |
| `formatForLog()` | 格式化日誌值（截斷至 240 字元） | 104-158 |
| `logWs()` | 主日誌函式 | 260-318 |
| `summarizeAgentEventForWsLog()` | Agent 事件摘要 | 168-258 |

`logWs()` 支援三種方向和三種類型：

| 方向 | 類型 | 說明 |
|------|------|------|
| `in` | `req` | 收到的請求 |
| `out` | `res` | 發出的回應 |
| `out` | `event` | 推送的事件 |

> **引用來源**：`source-repo/src/gateway/ws-log.ts:11-318`

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| Protocol schema 目錄 | `source-repo/src/gateway/protocol/` | — |
| AJV schema 驗證入口 | `source-repo/src/gateway/protocol/index.ts` | 1-60 |
| 客戶端資訊 | `source-repo/src/gateway/protocol/client-info.ts` | — |
| BASE_METHODS (134 個) | `source-repo/src/gateway/server-methods-list.ts` | 4-134 |
| listGatewayMethods() | `source-repo/src/gateway/server-methods-list.ts` | 136-139 |
| GATEWAY_EVENTS (25 個) | `source-repo/src/gateway/server-methods-list.ts` | 141-166 |
| 方法授權 | `source-repo/src/gateway/server-methods.ts` | 39-67 |
| Handler 聚合 | `source-repo/src/gateway/server-methods.ts` | 69-100 |
| WS 連線管理 | `source-repo/src/gateway/server-ws-runtime.ts` | 24-46 |
| WS 日誌常數 | `source-repo/src/gateway/ws-log.ts` | 11-12 |
| WS 日誌函式 | `source-repo/src/gateway/ws-log.ts` | 89-318 |
| GATEWAY_EVENT_UPDATE_AVAILABLE | `source-repo/src/gateway/events.ts` | 3 |
