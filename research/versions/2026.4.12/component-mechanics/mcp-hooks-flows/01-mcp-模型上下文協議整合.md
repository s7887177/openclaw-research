# MCP 模型上下文協議整合（Model Context Protocol Integration）

> **引用範圍**：`src/mcp/` 全部 7 個檔案
> **字數目標**：6,000–10,000 字

OpenClaw 對 MCP（Model Context Protocol，模型上下文協議）的整合採用「**橋接優先**」哲學：核心不內建完整的 MCP 執行環境，而是透過外部工具 mcporter 代理 MCP 伺服器的生命週期。同時，為了讓 Claude Desktop 等 MCP 原生客戶端能直接與 OpenClaw 閘道互動，`src/mcp/` 提供了一組輕量的 **Channel MCP Server**，將閘道功能以 MCP 工具的形式暴露出去。本章將從設計決策出發，逐一拆解這 7 個檔案的實作細節。

---

## 1. MCP 在 OpenClaw 中的定位

### 1.1 橋接模型（Bridge Model）而非第一類執行環境

根據 OpenClaw 的 `VISION.md`（第 72–83 行），MCP 整合的核心決策是：

> 使用 [mcporter](https://github.com/steipete/mcporter) 作為 MCP 整合橋接器。

這代表 OpenClaw **不會**在自身核心中嵌入一個完整的 MCP 伺服器執行引擎（runtime）。取而代之的是一個「橋接模型」，由 mcporter 負責：

1. **啟動與管理 MCP 伺服器**的生命週期
2. **轉發工具呼叫**（tool call）到對應的 MCP 伺服器
3. **回傳結果**至 OpenClaw 閘道

選擇橋接模型的理由有三：

| 理由 | 說明 |
|------|------|
| **不需重啟閘道** | 新增或修改 MCP 伺服器時，只需調整 mcporter 配置，閘道本身不受影響 |
| **保持核心精簡** | OpenClaw 核心不需追蹤 MCP SDK 的版本變動，降低依賴鏈複雜度 |
| **減少 MCP 變動衝擊** | MCP 協議仍在快速演進中，將整合邏輯外置可降低升級風險 |

`VISION.md` 第 105 行甚至明確將「在核心中建立第一類 MCP 執行環境」列入了**不會合併**（will not merge）清單：

> "First-class MCP runtime in core when mcporter already provides the integration path"

### 1.2 內建的 Channel MCP Server

儘管如此，OpenClaw 並非完全不碰 MCP 協議。`src/mcp/` 包含一個**內建的 MCP 頻道伺服器**（Channel MCP Server），目的是讓 **Claude Desktop** 這類 MCP 原生客戶端能透過標準 MCP 協議與 OpenClaw 的閘道（gateway）通訊。

這形成了一個有趣的雙軌架構：

```
┌─────────────────────────────────────────────────────┐
│                   OpenClaw Gateway                   │
│                                                     │
│  ┌──────────────┐          ┌──────────────────────┐ │
│  │   mcporter   │──bridge──│  MCP 伺服器們        │ │
│  │  (外部橋接)   │          │  (filesystem, git…)  │ │
│  └──────────────┘          └──────────────────────┘ │
│                                                     │
│  ┌──────────────────────────────────────┐           │
│  │  Channel MCP Server (內建)           │           │
│  │  ← Claude Desktop 等客戶端連入       │           │
│  └──────────────────────────────────────┘           │
└─────────────────────────────────────────────────────┘
```

- **mcporter** 方向：OpenClaw → 外部 MCP 伺服器（OpenClaw 是 MCP 客戶端）
- **Channel MCP Server** 方向：外部 AI 客戶端 → OpenClaw（OpenClaw 是 MCP 伺服器）

---

## 2. MCP 伺服器建立（channel-server.ts，103 行）

`channel-server.ts` 是 Channel MCP Server 的入口檔案，負責建立與啟動整個 MCP 伺服端。

### 2.1 配置選項

```typescript
// source-repo/src/mcp/channel-server.ts:11-18
interface OpenClawMcpServeOptions {
  gatewayUrl: string;       // 閘道 URL
  gatewayToken: string;     // 閘道認證 token
  gatewayPassword: string;  // 閘道密碼
  config: OpenClawConfig;   // 完整 OpenClaw 配置
  claudeChannelMode: ClaudeChannelMode; // "off" | "on" | "auto"
  verbose: boolean;         // 除錯輸出
}
```

`claudeChannelMode` 控制 Claude Desktop 整合的行為模式：
- `"off"`：不啟用 Claude 頻道功能
- `"on"`：強制啟用
- `"auto"`：依據客戶端能力自動判斷

### 2.2 createOpenClawChannelMcpServer()

```typescript
// source-repo/src/mcp/channel-server.ts:20-63（概要）
export function createOpenClawChannelMcpServer(opts: OpenClawMcpServeOptions) {
  // 1. 建立 McpServer 實例
  const server = new McpServer({
    name: "openclaw",
    version: VERSION  // 取自全域版本常數
  });

  // 2. 建立閘道橋接器
  const bridge = new OpenClawChannelBridge(opts.config, {
    gatewayUrl: opts.gatewayUrl,
    gatewayToken: opts.gatewayToken,
    gatewayPassword: opts.gatewayPassword,
    claudeChannelMode: opts.claudeChannelMode,
    verbose: opts.verbose,
  });

  // 3. 註冊 Claude 權限請求通知處理
  server.server.setNotificationHandler(
    ClaudePermissionRequestSchema,
    (notification) => bridge.handleClaudePermissionRequest(notification)
  );

  // 4. 註冊頻道工具
  registerChannelMcpTools(server, bridge, opts);

  // 5. 回傳控制物件
  return { server, bridge, start, close };
}
```

這個函式使用 `@modelcontextprotocol/sdk`（版本 v1.29.0）的 `McpServer` 類別。值得注意的是第 3 步：它註冊了一個**通知處理器**（notification handler），用來接收 Claude Desktop 發來的權限請求（permission request）。這是 Claude 特有的擴充功能，不在標準 MCP 規範中。

### 2.3 serveOpenClawChannelMcp()

```typescript
// source-repo/src/mcp/channel-server.ts:65-102（概要）
export async function serveOpenClawChannelMcp(opts: OpenClawMcpServeOptions) {
  const { server, bridge, close } = createOpenClawChannelMcpServer(opts);

  // 建立 Stdio 傳輸層
  const transport = new StdioServerTransport();

  // 優雅關閉
  const shutdown = async () => { await close(); process.exit(0); };
  process.on("SIGINT", shutdown);
  process.on("SIGTERM", shutdown);
  process.stdin.on("end", shutdown);
  process.stdin.on("close", shutdown);

  // 連接傳輸層並啟動橋接器
  await server.connect(transport);
  await bridge.start();

  // 等待關閉
  await closed;
}
```

此函式建立 `StdioServerTransport`——MCP 最常見的傳輸方式，透過 stdin/stdout 進行 JSON-RPC 通訊。這讓 Claude Desktop 可以將 OpenClaw 當作一個本地子程序（subprocess）來啟動。

優雅關閉（graceful shutdown）處理了四種情境：
1. 收到 `SIGINT`（如使用者按 Ctrl+C）
2. 收到 `SIGTERM`（程序管理器要求停止）
3. stdin 結束（父程序關閉了管道）
4. stdin 關閉（連線斷開）

---

## 3. Gateway 橋接器（channel-bridge.ts，516 行）

`OpenClawChannelBridge` 是整個 MCP 子系統中最複雜的元件，它在 MCP 協議與 OpenClaw 閘道之間建立雙向通訊橋樑。

### 3.1 建構與啟動

```typescript
// source-repo/src/mcp/channel-bridge.ts:62-78（概要）
class OpenClawChannelBridge {
  constructor(config: OpenClawConfig, params: {
    gatewayUrl: string;
    gatewayToken: string;
    gatewayPassword: string;
    claudeChannelMode: ClaudeChannelMode;
    verbose: boolean;
  }) {
    // 儲存配置，初始化內部狀態
  }
}
```

```typescript
// source-repo/src/mcp/channel-bridge.ts:84-130（概要）
async start() {
  // 1. 解析閘道客戶端啟動配置
  const bootstrap = await resolveGatewayClientBootstrap();

  // 2. 建立 GatewayClient，宣告所需權限範圍
  this.client = new GatewayClient({
    ...bootstrap,
    scopes: [READ_SCOPE, WRITE_SCOPE, APPROVALS_SCOPE],
    clientName: "OpenClaw MCP",
  });

  // 3. 訂閱閘道事件
  this.client.on("event", (event) => this.handleGatewayEvent(event));

  // 4. 連接閘道
  await this.client.connect();
}
```

橋接器要求三個權限範圍（scope）：
- `READ_SCOPE`：讀取對話與訊息
- `WRITE_SCOPE`：發送訊息
- `APPROVALS_SCOPE`：處理執行與 plugin 的核准請求

### 3.2 事件佇列系統

橋接器內建一個有界事件佇列（bounded event queue），用來緩衝從閘道收到的事件，供 MCP 工具輪詢（poll）或等待（wait）：

```typescript
// source-repo/src/mcp/channel-bridge.ts:42
const QUEUE_LIMIT = 1_000;
```

```typescript
// source-repo/src/mcp/channel-bridge.ts:353-367（概要）
enqueue(event: QueueEvent) {
  this.queue.push(event);
  // 超過上限時移除最舊的事件
  if (this.queue.length > QUEUE_LIMIT) {
    this.queue.shift();
  }
  // 喚醒所有等待中的 Promise
  for (const waiter of this.pendingWaiters) {
    waiter.resolve();
  }
  this.pendingWaiters.clear();
}
```

佇列上限為 **1,000 個事件**。當佇列滿時，最舊的事件會被丟棄（FIFO 淘汰策略）。每次有新事件入列時，所有透過 `waitForEvent()` 等待的消費者都會被喚醒。

兩個消費端口：

```typescript
// source-repo/src/mcp/channel-bridge.ts:244-248
pollEvents(filter: WaitFilter): { events: QueueEvent[]; nextCursor: string } {
  // 根據 filter.afterCursor 和 filter.sessionKey 過濾
  // 回傳符合條件的事件 + 下一個游標
}
```

```typescript
// source-repo/src/mcp/channel-bridge.ts:250-271
async waitForEvent(filter: WaitFilter, timeoutMs = 30_000): Promise<QueueEvent | null> {
  // 先 poll 一次，如果有事件直接回傳
  // 否則建立 Promise，加入 pendingWaiters
  // 等待 timeout 或被 enqueue() 喚醒
}
```

`pollEvents()` 是非阻塞的，立即回傳；`waitForEvent()` 是阻塞式的（Promise-based），預設超時 30 秒。這對應了 MCP 工具中的 `events_poll` 與 `events_wait`。

### 3.3 閘道事件處理

```typescript
// source-repo/src/mcp/channel-bridge.ts:393-437（概要）
handleGatewayEvent(event: GatewayEvent) {
  switch (event.type) {
    case "session.message":
      this.handleSessionMessageEvent(event);
      break;
    case "exec.approval.requested":
      this.trackApproval(event);
      this.enqueue(/* exec_approval_requested */);
      break;
    case "exec.approval.resolved":
      this.resolveTrackedApproval(event);
      this.enqueue(/* exec_approval_resolved */);
      break;
    case "plugin.approval.requested":
      this.trackApproval(event);
      this.enqueue(/* plugin_approval_requested */);
      break;
    case "plugin.approval.resolved":
      this.resolveTrackedApproval(event);
      this.enqueue(/* plugin_approval_resolved */);
      break;
  }
}
```

事件分為兩大類：

1. **訊息事件**（`session.message`）→ 委派給 `handleSessionMessageEvent()` 做進一步處理
2. **核准事件**（`exec.approval.*` / `plugin.approval.*`）→ 追蹤核准狀態 + 入列

### 3.4 對話管理

橋接器封裝了四個對話相關操作，全部透過閘道客戶端轉發：

```typescript
// source-repo/src/mcp/channel-bridge.ts:154-219（概要）

// 列出對話
async listConversations(channel?: string): Promise<ConversationDescriptor[]> {
  const sessions = await this.client.call("sessions.list");
  // 過濾 channel，轉換為 ConversationDescriptor
}

// 取得單一對話
async getConversation(sessionKey: string): Promise<ConversationDescriptor | null> {
  const list = await this.listConversations();
  return list.find(c => c.sessionKey === sessionKey) ?? null;
}

// 讀取訊息
async readMessages(sessionKey: string, limit = 20): Promise<Message[]> {
  return this.client.call("chat.history", { sessionKey, limit });
}

// 發送訊息（帶冪等性金鑰）
async sendMessage(sessionKey: string, content: string): Promise<void> {
  await this.client.call("send", {
    sessionKey,
    content,
    idempotencyKey: crypto.randomUUID(),
  });
}
```

`sendMessage()` 使用 `randomUUID()` 作為**冪等性金鑰**（idempotency key），確保即使網路重試也不會產生重複訊息。

### 3.5 核准管理

```typescript
// source-repo/src/mcp/channel-bridge.ts:221-242（概要）

// 列出待處理核准
listPendingApprovals(): Approval[] {
  return [...this.pendingApprovals.values()]
    .sort((a, b) => a.createdAtMs - b.createdAtMs);
}

// 回應核准請求
async respondToApproval(id: string, kind: "exec" | "plugin", decision: string) {
  const method = kind === "exec"
    ? "exec.approval.resolve"
    : "plugin.approval.resolve";
  await this.client.call(method, { id, decision });
}
```

### 3.6 Claude 權限處理

這是最複雜的部分，專門處理 Claude Desktop 的權限請求流程：

```typescript
// source-repo/src/mcp/channel-bridge.ts:273-295（概要）
handleClaudePermissionRequest(notification: ClaudePermissionRequest) {
  // 儲存到 pendingClaudePermissions Map
  this.pendingClaudePermissions.set(notification.id, notification);
  // 將權限請求作為事件入列
  this.enqueue({ type: "claude_permission_request", ... });
}
```

```typescript
// source-repo/src/mcp/channel-bridge.ts:440-502（概要）
// 用於匹配使用者的權限回覆訊息
const CLAUDE_PERMISSION_REPLY_RE = /^(yes|no)\s+([a-km-z]{5})$/i;

handleSessionMessageEvent(event: SessionMessageEvent) {
  const match = event.content.match(CLAUDE_PERMISSION_REPLY_RE);

  if (match) {
    // 使用者回覆了權限請求（如 "yes abcde" 或 "no fghij"）
    const [, decision, code] = match;
    // 發送 MCP 通知回 Claude Desktop
    this.server.sendNotification("notifications/claude/channel/permission", {
      allow: decision.toLowerCase() === "yes",
      code,
    });
  } else {
    // 一般訊息
    this.enqueue({ type: "message", ... });
    // 如果是使用者角色的訊息，額外發送 Claude 頻道通知
    if (event.role === "user") {
      this.server.sendNotification("notifications/claude/channel", { ... });
    }
  }
}
```

權限流程的運作方式：

1. Claude Desktop 需要執行某個操作，透過 MCP 通知發送權限請求
2. 橋接器收到請求，儲存在 `pendingClaudePermissions` Map 中，同時入列事件
3. MCP 工具將權限請求呈現給使用者（例如在聊天介面中顯示）
4. 使用者回覆 `yes xxxxx` 或 `no xxxxx`（xxxxx 是五位辨識碼）
5. 橋接器偵測到這個格式，發送 MCP 通知回 Claude Desktop
6. Claude Desktop 根據結果執行或取消操作

正規表達式 `/^(yes|no)\s+([a-km-z]{5})$/i` 的設計值得注意：辨識碼使用了 `a-km-z`（排除了 `l`），這是為了避免 `l` 與 `1` 的視覺混淆。

---

## 4. MCP 工具註冊（channel-tools.ts，188 行）

`registerChannelMcpTools()` 將閘道功能包裝為 8 個標準 MCP 工具加上 1 個內部工具。

### 4.1 實驗性能力宣告

```typescript
// source-repo/src/mcp/channel-tools.ts:11-21（概要）
if (claudeChannelMode !== "off") {
  server.server.setCapabilities({
    experimental: {
      "claude/channel": {},
      "claude/channel/permission": {},
    },
  });
}
```

當 `claudeChannelMode` 不是 `"off"` 時，伺服器會宣告兩個實驗性能力（experimental capabilities），讓 Claude Desktop 知道這個 MCP 伺服器支援頻道互動與權限管理。

### 4.2 工具清單

以下是全部 9 個工具的詳細說明：

#### 對話類工具

| 工具名稱 | 行號 | 說明 |
|----------|------|------|
| `conversations_list` | 24–41 | 列出對話清單，支援 `channel` 與 `search` 過濾條件 |
| `conversation_get` | 43–60 | 根據 `session_key` 取得單一對話的詳細資訊 |
| `messages_read` | 62–76 | 讀取指定對話的近期訊息，預設 20 則，上限 200 則 |
| `attachments_fetch` | 78–101 | 列出某則訊息中的非文字附件（圖片、檔案等） |
| `messages_send` | 143–157 | 發送訊息至指定對話 |

#### 事件類工具

| 工具名稱 | 行號 | 說明 |
|----------|------|------|
| `events_poll` | 103–121 | 非阻塞式輪詢：自指定 cursor 後取得新事件 |
| `events_wait` | 123–141 | 阻塞式等待：等待下一個事件，預設超時 30 秒，上限 300 秒 |

#### 權限類工具

| 工具名稱 | 行號 | 說明 |
|----------|------|------|
| `permissions_list_open` | 159–170 | 列出所有待處理的核准請求 |
| `permissions_respond` | 172–187 | 回應核准請求 |

`permissions_respond` 的參數設計：

```typescript
// source-repo/src/mcp/channel-tools.ts:172-187（概要）
{
  kind: "exec" | "plugin",           // 核准類型
  decision: "allow-once"             // 僅此次允許
           | "allow-always"          // 永久允許
           | "deny",                 // 拒絕
  id: string,                        // 核准請求 ID
}
```

三種決策選項對應了不同的安全等級：`allow-once` 最保守，`allow-always` 最寬鬆，`deny` 直接拒絕。

### 4.3 輪詢與等待的設計哲學

`events_poll` 和 `events_wait` 形成了一個經典的**長輪詢**（long polling）模式：

1. 客戶端先呼叫 `events_poll` 取得目前所有事件和一個 cursor
2. 之後呼叫 `events_wait` 帶入 cursor，等待新事件
3. 收到事件後回到步驟 1

這比 WebSocket 更適合 MCP 的 stdio 傳輸模式，因為 MCP 工具呼叫（tool call）是請求-回應式的，不支援伺服器主動推送。

---

## 5. 共享型別（channel-shared.ts，210 行）

`channel-shared.ts` 定義了 MCP 子系統中共用的所有型別、schema 和工具函式。

### 5.1 核心型別

```typescript
// source-repo/src/mcp/channel-shared.ts:5
type ClaudeChannelMode = "off" | "on" | "auto";
```

```typescript
// source-repo/src/mcp/channel-shared.ts:7-18
interface ConversationDescriptor {
  sessionKey: string;       // 對話唯一識別
  channel: string;          // 頻道名稱（如 "whatsapp", "discord"）
  to: string;               // 目標聯絡人
  accountId: string;        // 帳號 ID
  threadId: string;         // 執行緒 ID
  label: string;            // 顯示標籤
  displayName: string;      // 顯示名稱
  derivedTitle: string;     // 衍生標題
  lastMessagePreview: string; // 最後訊息預覽
  updatedAt: number;        // 最後更新時間戳
}
```

### 5.2 佇列事件（QueueEvent）

`QueueEvent` 使用**區別聯合型別**（discriminated union），以 `type` 欄位區分不同事件：

```typescript
// source-repo/src/mcp/channel-shared.ts:76-105（概要）
type QueueEvent =
  | { type: "message"; sessionKey: string; content: string; role: string; ... }
  | { type: "claude_permission_request"; id: string; code: string; ... }
  | { type: "exec_approval_requested"; id: string; ... }
  | { type: "exec_approval_resolved"; id: string; decision: string; ... }
  | { type: "plugin_approval_requested"; id: string; ... }
  | { type: "plugin_approval_resolved"; id: string; decision: string; ... };
```

六種事件型別涵蓋了 MCP 子系統中所有的異步通訊場景。

### 5.3 等待過濾器

```typescript
// source-repo/src/mcp/channel-shared.ts:113-116
interface WaitFilter {
  afterCursor: string;       // 只回傳此 cursor 之後的事件
  sessionKey?: string;       // 可選：僅過濾特定對話的事件
}
```

### 5.4 Claude 權限請求 Schema

```typescript
// source-repo/src/mcp/channel-shared.ts:118-126
const ClaudePermissionRequestSchema = z.object({
  method: z.literal("notifications/claude/channel/permission_request"),
  params: z.object({
    id: z.string(),
    code: z.string(),
    description: z.string(),
  }),
});
```

使用 Zod 進行執行期驗證（runtime validation），確保 Claude Desktop 發來的通知格式正確。

### 5.5 工具函式

`channel-shared.ts` 還提供了多個工具函式：

| 函式 | 說明 |
|------|------|
| `resolveMessageId()` | 從訊息物件中解析出穩定的 ID |
| `summarizeResult()` | 將閘道回傳的結果摘要化（用於 MCP 工具回應） |
| `toConversation()` | 將閘道的 session 物件轉換為 `ConversationDescriptor` |
| `matchEventFilter()` | 判斷事件是否符合 `WaitFilter` 條件 |
| `extractAttachmentsFromMessage()` | 從訊息中提取非文字附件 |
| `normalizeApprovalId()` | 正規化核准 ID 格式 |

---

## 6. Plugin 工具伺服（plugin-tools-serve.ts，140 行）

除了 Channel MCP Server，`src/mcp/` 還包含一個**獨立的 Plugin 工具 MCP 伺服器**。

### 6.1 用途

Plugin 可以註冊自己的工具（tool），這些工具需要透過 MCP 協議暴露給外部客戶端。`plugin-tools-serve.ts` 負責：

1. 透過 `resolvePluginTools()` 載入所有啟用中的 plugin 工具
2. 使用 `wrapToolWithBeforeToolCallHook()` 包裝每個工具，確保執行邊界（execution boundary）的安全性
3. 建立獨立的 MCP 伺服器提供這些工具

### 6.2 日誌路由

```
stdout ─── MCP 協議（JSON-RPC）
stderr ─── 日誌輸出
```

MCP 規範要求 stdout 專門用於協議通訊，因此所有的日誌（log）訊息必須路由到 stderr。這個限制在 `plugin-tools-serve.ts` 中有明確的處理。

### 6.3 安全包裝

`wrapToolWithBeforeToolCallHook()` 在每次工具呼叫前觸發一個 hook，讓安全子系統可以：

- 檢查工具是否被允許執行
- 記錄工具呼叫審計日誌
- 套用沙箱限制

---

## 7. 引用來源

| 引用代碼 | 檔案路徑 | 行號範圍 | 說明 |
|----------|----------|----------|------|
| MCP-SRV-1 | `source-repo/src/mcp/channel-server.ts` | 11–18 | `OpenClawMcpServeOptions` 介面定義 |
| MCP-SRV-2 | `source-repo/src/mcp/channel-server.ts` | 20–63 | `createOpenClawChannelMcpServer()` 函式 |
| MCP-SRV-3 | `source-repo/src/mcp/channel-server.ts` | 65–102 | `serveOpenClawChannelMcp()` 函式 |
| MCP-BRG-1 | `source-repo/src/mcp/channel-bridge.ts` | 42 | `QUEUE_LIMIT` 常數 |
| MCP-BRG-2 | `source-repo/src/mcp/channel-bridge.ts` | 62–78 | `OpenClawChannelBridge` 建構式 |
| MCP-BRG-3 | `source-repo/src/mcp/channel-bridge.ts` | 84–130 | `start()` 方法 |
| MCP-BRG-4 | `source-repo/src/mcp/channel-bridge.ts` | 154–177 | `listConversations()` 方法 |
| MCP-BRG-5 | `source-repo/src/mcp/channel-bridge.ts` | 179–188 | `getConversation()` 方法 |
| MCP-BRG-6 | `source-repo/src/mcp/channel-bridge.ts` | 190–200 | `readMessages()` 方法 |
| MCP-BRG-7 | `source-repo/src/mcp/channel-bridge.ts` | 202–219 | `sendMessage()` 方法 |
| MCP-BRG-8 | `source-repo/src/mcp/channel-bridge.ts` | 221–225 | `listPendingApprovals()` 方法 |
| MCP-BRG-9 | `source-repo/src/mcp/channel-bridge.ts` | 227–242 | `respondToApproval()` 方法 |
| MCP-BRG-10 | `source-repo/src/mcp/channel-bridge.ts` | 244–248 | `pollEvents()` 方法 |
| MCP-BRG-11 | `source-repo/src/mcp/channel-bridge.ts` | 250–271 | `waitForEvent()` 方法 |
| MCP-BRG-12 | `source-repo/src/mcp/channel-bridge.ts` | 273–295 | `handleClaudePermissionRequest()` 方法 |
| MCP-BRG-13 | `source-repo/src/mcp/channel-bridge.ts` | 353–367 | `enqueue()` 方法 |
| MCP-BRG-14 | `source-repo/src/mcp/channel-bridge.ts` | 393–437 | `handleGatewayEvent()` 方法 |
| MCP-BRG-15 | `source-repo/src/mcp/channel-bridge.ts` | 440–502 | Claude 權限回覆處理 |
| MCP-TLS-1 | `source-repo/src/mcp/channel-tools.ts` | 11–21 | 實驗性能力宣告 |
| MCP-TLS-2 | `source-repo/src/mcp/channel-tools.ts` | 24–41 | `conversations_list` 工具 |
| MCP-TLS-3 | `source-repo/src/mcp/channel-tools.ts` | 43–60 | `conversation_get` 工具 |
| MCP-TLS-4 | `source-repo/src/mcp/channel-tools.ts` | 62–76 | `messages_read` 工具 |
| MCP-TLS-5 | `source-repo/src/mcp/channel-tools.ts` | 78–101 | `attachments_fetch` 工具 |
| MCP-TLS-6 | `source-repo/src/mcp/channel-tools.ts` | 103–121 | `events_poll` 工具 |
| MCP-TLS-7 | `source-repo/src/mcp/channel-tools.ts` | 123–141 | `events_wait` 工具 |
| MCP-TLS-8 | `source-repo/src/mcp/channel-tools.ts` | 143–157 | `messages_send` 工具 |
| MCP-TLS-9 | `source-repo/src/mcp/channel-tools.ts` | 159–170 | `permissions_list_open` 工具 |
| MCP-TLS-10 | `source-repo/src/mcp/channel-tools.ts` | 172–187 | `permissions_respond` 工具 |
| MCP-SHR-1 | `source-repo/src/mcp/channel-shared.ts` | 5 | `ClaudeChannelMode` 型別 |
| MCP-SHR-2 | `source-repo/src/mcp/channel-shared.ts` | 7–18 | `ConversationDescriptor` 介面 |
| MCP-SHR-3 | `source-repo/src/mcp/channel-shared.ts` | 76–105 | `QueueEvent` 區別聯合型別 |
| MCP-SHR-4 | `source-repo/src/mcp/channel-shared.ts` | 113–116 | `WaitFilter` 介面 |
| MCP-SHR-5 | `source-repo/src/mcp/channel-shared.ts` | 118–126 | `ClaudePermissionRequestSchema` |
| MCP-PLG-1 | `source-repo/src/mcp/plugin-tools-serve.ts` | 1–140 | Plugin 工具 MCP 伺服器 |
| VISION-1 | `source-repo/VISION.md` | 72–83 | MCP 橋接策略 |
| VISION-2 | `source-repo/VISION.md` | 105 | 不合併清單 |
