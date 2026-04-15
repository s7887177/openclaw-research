# Gateway 子章節 05：Session 與 Chat 管理

> **引用範圍**：`src/gateway/server-chat.ts`, `session-utils.ts`, `session-utils.types.ts`, `session-lifecycle-state.ts`, `session-history-state.ts`

## 1. Session 資料模型

### 1.1 GatewaySessionRow

每個 Session 在 Gateway 中以 `GatewaySessionRow` 結構表示：

```typescript
// source-repo/src/gateway/session-utils.types.ts:18-70 (摘要)
export type GatewaySessionRow = {
  // 身份與血緣
  key: string;                     // Session 唯一 key
  spawnedBy?: string;              // 父 Session key
  forkedFromParent?: string;       // Fork 來源
  subagentRole?: string;           // 子 Agent 角色

  // 分類
  kind: "direct" | "group" | "global" | "unknown";

  // 顯示
  label?: string;
  displayName?: string;
  derivedTitle?: string;
  lastMessagePreview?: string;

  // 路由
  channel?: string;                // Channel ID
  subject?: string;                // 主題
  origin?: string;                 // 來源

  // 時間
  updatedAt?: number;

  // 運行資訊
  sessionId?: string;
  systemSent?: boolean;

  // Agent 行為設定
  thinkingLevel?: string;
  fastMode?: boolean;
  verboseLevel?: string;
  traceLevel?: string;
  reasoningLevel?: string;

  // 權限
  sendPolicy?: "allow" | "deny";

  // 用量統計
  inputTokens?: number;
  outputTokens?: number;
  totalTokens?: number;
  estimatedCostUsd?: number;

  // 生命週期
  status?: "running" | "done" | "failed" | "killed" | "timeout";
  startedAt?: number;
  endedAt?: number;
  runtimeMs?: number;

  // 關聯
  parentSessionKey?: string;
  childSessions?: string[];

  // 模型
  modelProvider?: string;
  model?: string;
  contextTokens?: number;
  deliveryContext?: string;
};
```

> **引用來源**：`source-repo/src/gateway/session-utils.types.ts:18-70`

### 1.2 Session Kind 分類

| Kind | 說明 | 來源 |
|------|------|------|
| `direct` | 一對一對話 | CLI、DM |
| `group` | 群組對話 | Discord/Slack 群組 |
| `global` | 全域 Session | 系統級 |
| `unknown` | 未知類型 | 遺留或異常 |

---

## 2. Chat 執行佇列

`server-chat.ts` 實作了一個 Chat 執行佇列，管理併發的聊天請求：

### 2.1 ChatRunEntry

```typescript
// source-repo/src/gateway/server-chat.ts:132-135
export type ChatRunEntry = {
  sessionKey: string;    // Session 識別
  clientRunId: string;   // 客戶端執行 ID
};
```

### 2.2 ChatRunRegistry

```typescript
// source-repo/src/gateway/server-chat.ts:137-143 (概要)
export type ChatRunRegistry = {
  add(entry: ChatRunEntry): void;     // 加入佇列
  peek(): ChatRunEntry | undefined;   // 查看佇列頭
  shift(): ChatRunEntry | undefined;  // 取出佇列頭
  remove(sessionKey: string): void;   // 移除指定 Session
  clear(): void;                      // 清空佇列
};
```

透過 `createChatRunRegistry()` 工廠函式建立。

> **引用來源**：`source-repo/src/gateway/server-chat.ts:132-145+`

### 2.3 心跳處理

聊天系統支援心跳訊息（Heartbeat），用於維持長時間運行的 Agent 活動狀態：

```typescript
// source-repo/src/gateway/server-chat.ts:35-46
export function resolveHeartbeatContext(params: {
  text: string;
  // ...
}): { isHeartbeat: boolean; heartbeatText?: string }
```

```typescript
// source-repo/src/gateway/server-chat.ts:68-88
export function normalizeHeartbeatChatFinalText(params: {
  text: string;
  isHeartbeat: boolean;
  // ...
}): string
```

心跳訊息在最終回應中被**靜默抑制**，不會顯示給使用者。

> **引用來源**：`source-repo/src/gateway/server-chat.ts:35-88`

### 2.4 文字合併

```typescript
// source-repo/src/gateway/server-chat.ts:109-130
export function resolveMergedAssistantText(params: {
  existingText: string;
  newText: string;
  // ...
}): string
```

處理串流回應時的增量文字合併。

> **引用來源**：`source-repo/src/gateway/server-chat.ts:109-130`

---

## 3. Session 生命週期狀態

`session-lifecycle-state.ts` 管理 Session 的狀態轉換：

```typescript
// source-repo/src/gateway/session-lifecycle-state.ts:92-130
export function deriveGatewaySessionLifecycleSnapshot(
  event: AgentLifecycleEvent
): GatewaySessionLifecycleSnapshot
```

從 Agent 生命週期事件衍生 Session 狀態快照。

```typescript
// source-repo/src/gateway/session-lifecycle-state.ts:132-144
export function derivePersistedSessionLifecyclePatch(
  snapshot: GatewaySessionLifecycleSnapshot
): SessionLifecyclePatch
```

將快照轉換為可持久化的差異（Patch）。

```typescript
// source-repo/src/gateway/session-lifecycle-state.ts:146-169
export function persistGatewaySessionLifecycleEvent(params: {
  event: AgentLifecycleEvent;
  storePath: string;
  sessionKey: string;
}): Promise<void>
```

將生命週期事件持久化到磁碟。

**Session 狀態轉換圖**：

```
created → running → done
                  → failed
                  → killed
                  → timeout
```

> **引用來源**：`source-repo/src/gateway/session-lifecycle-state.ts:92-169`

---

## 4. Session 歷史管理

### 4.1 分頁歷史

```typescript
// source-repo/src/gateway/session-history-state.ts:15-20
export type PaginatedSessionHistory = {
  items: unknown[];           // 歷史項目
  messages: unknown[];        // 訊息陣列
  nextCursor?: string;        // 分頁游標
  hasMore: boolean;           // 是否有更多
};
```

### 4.2 歷史快照

```typescript
// source-repo/src/gateway/session-history-state.ts:22-25
export type SessionHistorySnapshot = {
  history: PaginatedSessionHistory;
  rawTranscriptSeq: number;   // 原始轉錄序號
};
```

### 4.3 SSE 歷史狀態

```typescript
// source-repo/src/gateway/session-history-state.ts:119-219 (概要)
export class SessionHistorySseState {
  snapshot(): SessionHistorySnapshot;           // 取得當前快照
  appendInlineMessage(msg: unknown): void;      // 附加即時訊息
  refresh(): Promise<void>;                     // 刷新歷史
}
```

`SessionHistorySseState` 管理透過 SSE（Server-Sent Events）串流推送的即時歷史狀態，支援：
- 增量更新（新訊息附加）
- 完整刷新（從儲存載入）
- 分頁查詢（游標式分頁）

> **引用來源**：`source-repo/src/gateway/session-history-state.ts:15-219`

---

## 5. Session 相關 WS 方法

| 方法 | 說明 | 資料流向 |
|------|------|---------|
| `sessions.list` | 列出所有 Session | Client ← Server |
| `sessions.create` | 建立新 Session | Client → Server |
| `sessions.send` | 在 Session 中發送訊息 | Client → Server |
| `sessions.abort` | 中止運行中的 Session | Client → Server |
| `sessions.patch` | 更新 Session 屬性 | Client → Server |
| `sessions.reset` | 重設 Session | Client → Server |
| `sessions.delete` | 刪除 Session | Client → Server |
| `sessions.compact` | 壓縮 Session 歷史 | Client → Server |
| `sessions.subscribe` | 訂閱 Session 變更 | Client → Server (然後推送) |
| `sessions.messages.subscribe` | 訂閱特定 Session 訊息 | Client → Server (然後推送) |

相關事件推送：

| 事件 | 觸發時機 |
|------|---------|
| `sessions.changed` | Session 列表變更 |
| `session.message` | Session 中有新訊息 |
| `session.tool` | Session 中有工具調用 |

> **引用來源**：`source-repo/src/gateway/server-methods-list.ts:73-89, 143-146`

---

## 6. Session Key 結構

Session Key 是系統中的唯一識別碼，格式為：

```
{agentId}:{channelId}:{target}:{threadId?}
```

例如：`default:discord:123456789:thread-001`

Session Key 由 `resolveMainSessionKey()` 和相關工具函式從配置中解析。

> **引用來源**：`source-repo/src/config/sessions.js`（透過 `server.impl.ts:18` 匯入）

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| GatewaySessionRow 型別 | `source-repo/src/gateway/session-utils.types.ts` | 18-70 |
| ChatRunEntry 型別 | `source-repo/src/gateway/server-chat.ts` | 132-135 |
| ChatRunRegistry 型別 | `source-repo/src/gateway/server-chat.ts` | 137-143 |
| 心跳解析 | `source-repo/src/gateway/server-chat.ts` | 35-46 |
| 心跳文字正規化 | `source-repo/src/gateway/server-chat.ts` | 68-88 |
| 文字合併 | `source-repo/src/gateway/server-chat.ts` | 109-130 |
| 生命週期快照 | `source-repo/src/gateway/session-lifecycle-state.ts` | 92-130 |
| 生命週期持久化 | `source-repo/src/gateway/session-lifecycle-state.ts` | 146-169 |
| PaginatedSessionHistory | `source-repo/src/gateway/session-history-state.ts` | 15-20 |
| SessionHistorySseState | `source-repo/src/gateway/session-history-state.ts` | 119-219 |
| Session WS 方法 | `source-repo/src/gateway/server-methods-list.ts` | 73-89 |
| Session 事件 | `source-repo/src/gateway/server-methods-list.ts` | 143-146 |
