# Presence、Retry 機制、Session 持久化與 ACP 映射

## 本章摘要

上一章介紹了 session 的基礎概念與路由引擎。本章關注 session 的**運行時行為**：使用者的線上狀態追蹤（presence）、失敗重試機制、session 資料的持久化策略、ACP（Agent Communication Protocol）的 session 映射、以及規則引擎驅動的發送政策（send policy）。

---

## 1. Presence：線上狀態追蹤

### 1.1 Discord Presence 設定

Discord 是 OpenClaw 支援 presence 最完整的通道（`source-repo/src/config/types.discord.ts`）：

**基礎 Presence 設定**（Lines 110-119）：
```typescript
// DiscordActionConfig
presence?: boolean;  // 啟用 bot presence 變更（預設 false）

// DiscordIntentsConfig
presence?: boolean;  // Guild Presences 特權 intent（預設 false）
```

**自動 Presence 設定**（Lines 206-219）：
```typescript
type DiscordAutoPresenceConfig = {
  enabled?: boolean;              // 啟用自動更新（預設 false）
  intervalMs?: number;            // 輪詢間隔（預設 30000ms）
  minUpdateIntervalMs?: number;   // 最小更新間距（預設 15000ms）
  healthyText?: string;           // 健康狀態文字
  degradedText?: string;          // 降級狀態文字
  exhaustedText?: string;         // 耗盡狀態文字
};
```

**帳號級 Presence 設定**（Lines 321-328）：
```typescript
// DiscordAccountConfig 中的 presence 欄位
activity?: string;                // 自訂狀態文字
status?: "online" | "dnd" | "idle" | "invisible";
autoPresence?: DiscordAutoPresenceConfig;
activityType?: 0 | 1 | 2 | 3 | 4 | 5; // 預設 4=Custom
activityUrl?: string;             // 用於 streaming type 1
```

### 1.2 Presence 的廣播

Presence 狀態變更透過 Gateway 的廣播系統推送：

```typescript
// source-repo/src/gateway/server-broadcast-types.ts:1-5
type GatewayBroadcastStateVersion = {
  presence?: number;  // Presence 版本號
  health?: number;    // 健康狀態版本號
};
```

每次 presence 變更都遞增 `presence` 版本號，客戶端可以據此判斷是否需要更新。

---

## 2. Session 生命週期狀態

### 2.1 狀態模型

```typescript
// source-repo/src/gateway/session-lifecycle-state.ts:1-26
// LifecyclePhase: "start" | "end" | "error"
```

狀態映射（`source-repo/src/gateway/session-lifecycle-state.ts:30-37`）：

| 生命週期階段 | 終止原因 | 最終狀態 |
|-------------|---------|---------|
| `phase="start"` | — | `"running"` |
| `phase="error"` | — | `"failed"` |
| `phase="end"` | `stopReason="aborted"` | `"killed"` |
| `phase="end"` | 超時 | `"timeout"` |
| `phase="end"` | 其他 | `"done"` |

### 2.2 時間戳解析

```typescript
// source-repo/src/gateway/session-lifecycle-state.ts:53-90
// 時間戳解析：startedAt, endedAt
// 運行時間計算：runtimeMs = endedAt - startedAt
```

### 2.3 生命週期快照

```typescript
// source-repo/src/gateway/session-lifecycle-state.ts:92-130
// deriveGatewaySessionLifecycleSnapshot()
// 主要的生命週期推導函式
```

### 2.4 生命週期持久化

```typescript
// source-repo/src/gateway/session-lifecycle-state.ts:146-169
// persistGatewaySessionLifecycleEvent()
// 透過 updateSessionStoreEntry() 持久化到 session store
```

---

## 3. Retry 機制

### 3.1 協定級重試

Gateway 協定的錯誤結構內建重試指引：

```typescript
// source-repo/src/gateway/protocol/schema/frames.ts:128-137
// ErrorShapeSchema 欄位：
// retryable?: boolean      - 是否可重試
// retryAfterMs?: number    - 建議等待時間（毫秒）
```

建議的重試步驟：
- `"retry_with_device_token"` — 使用裝置 token 重試
- `"wait_then_retry"` — 等待後重試

### 3.2 壓縮重試

壓縮操作有自己的重試語義（`source-repo/src/gateway/session-compaction-checkpoints.ts:40-54`）：

```typescript
// resolveSessionCompactionCheckpointReason() 映射：
// trigger="manual"    → "manual"
// timedOut=true       → "timeout-retry"
// trigger="overflow"  → "overflow-retry"
// default             → "auto-threshold"
```

`"timeout-retry"` 和 `"overflow-retry"` 表示這些壓縮是因為先前的嘗試失敗而重試的。

### 3.3 設定重載重試

```typescript
// source-repo/src/gateway/config-reload.ts:133-148
// 設定檔缺失時最多重試 2 次
```

---

## 4. Session 持久化

### 4.1 Session Store Entry

每個持久化的 session 包含以下結構：

```
SessionEntry:
  sessionId: UUID
  sessionKey: 正規化路由鍵
  status: "running" | "done" | "failed" | "killed" | "timeout"
  startedAt, endedAt, runtimeMs, abortedLastRun
  sessionFile: 轉錄檔路徑
  compactionCheckpoints: SessionCompactionCheckpoint[]
```

### 4.2 Session 歸檔

```typescript
// source-repo/src/gateway/session-archive.fs.ts:22-31
// classifySessionTranscriptCandidate()
// 分類：
//   "current"  - 當前活躍
//   "stale"    - 過期
//   "custom"   - 使用者自訂路徑

// source-repo/src/gateway/session-archive.fs.ts:33-55
// extractGeneratedTranscriptSessionId()
// 從檔名提取 UUID 模式
```

歸檔時，轉錄檔被重新命名：
```
{path}.{reason}.{timestamp}
```
其中 reason 可以是 `"reset"` 或 `"deleted"`。

### 4.3 Session History 狀態

```typescript
// source-repo/src/gateway/session-history-state.ts:11-20
// SessionHistoryMessage, PaginatedSessionHistory 型別

// source-repo/src/gateway/session-history-state.ts:62-65
// resolveMessageSeq() - 提取 message.__openclaw?.seq

// source-repo/src/gateway/session-history-state.ts:67-94
// paginateSessionMessages()
// 游標格式: "seq:N"，向後分頁

// source-repo/src/gateway/session-history-state.ts:96-117
// buildSessionHistorySnapshot()
// 清洗、分頁、建構快照
```

### 4.4 SSE 歷史串流狀態

```typescript
// source-repo/src/gateway/session-history-state.ts:119-219
// SessionHistorySseState 類別
// Line 143-163: 建構子，可選的初始訊息
// Line 169-198: appendInlineMessage() - 串流中追加訊息
// Line 200-210: refresh() - 從轉錄重新載入歷史
```

---

## 5. ACP Session 映射

### 5.1 ACP 概述

ACP（Agent Communication Protocol）是 IDE 與 OpenClaw 之間的標準化通訊協定（`source-repo/docs.acp.md`）。

```
IDE ↔ ACP ↔ Gateway
```

### 5.2 相容性矩陣

ACP 支援的操作（`source-repo/docs.acp.md:28-40`）：
- ✅ `initialize`、`newSession`、`prompt`、`cancel`
- 🟡 部分支援：`loadSession`、session modes、tool streaming

### 5.3 Session 映射策略

ACP 的 session 映射（`source-repo/docs.acp.md:155-189`）：

**預設策略**：每個 ACP 客戶端使用 `acp:<uuid>` 格式的 session key。

**覆蓋選項**：
- `sessionKey` — 明確指定 session key
- `sessionLabel` — 設定 session 標籤
- `resetSession` — 重置 session
- `requireExisting` — 要求 session 已存在

### 5.4 ACP Session Store

```typescript
// source-repo/src/acp/session.ts:4-12
// AcpSessionStore interface

// source-repo/src/acp/session.ts:21-26
// 常數：
// MAX_SESSIONS = 5000
// IDLE_TTL_MS = 24 小時

// source-repo/src/acp/session.ts:31-59
// 收割邏輯：移除閒置 session，溢位時淘汰最舊的

// source-repo/src/acp/session.ts:80-107
// createSession() - 建立或重用（by sessionId）

// source-repo/src/acp/session.ts:119-129
// getSessionByRunId() - runId → sessionId → session

// source-repo/src/acp/session.ts:131-140
// setActiveRun() - 追蹤活躍 run（含 AbortController）

// source-repo/src/acp/session.ts:155-168
// cancelActiveRun() - 中止並清除追蹤
```

ACP Session 結構：
```typescript
AcpSession = {
  sessionId: string;
  sessionKey: string;
  cwd: string;
  createdAt: number;
  lastTouchedAt: number;
  activeRunId?: string;
  abortController?: AbortController;
}
```

### 5.5 Prompt 翻譯

ACP 的 prompt 翻譯（`source-repo/docs.acp.md:197-204`）：
- `text/resource` → prompt
- `images` → attachments

### 5.6 終止狀態映射

ACP 的終止狀態映射（`source-repo/docs.acp.md:207-211`）：
- `complete` → `stop`
- `aborted` → `cancel`
- `error` → `error`

---

## 6. Send Policy：發送政策引擎

### 6.1 政策正規化

```typescript
// source-repo/src/sessions/send-policy.ts:9-20
// normalizeSendPolicy()
// 回傳: "allow" | "deny" | undefined
```

### 6.2 Chat Type 推導

```typescript
// source-repo/src/sessions/send-policy.ts:51-79
// deriveChatTypeFromKey()
// 從 session key 模式推斷 chat type
```

### 6.3 核心政策引擎

```typescript
// source-repo/src/sessions/send-policy.ts:81-151
// resolveSendPolicy() - 主要政策引擎
// Line 88: entry override 最優先
// Line 103-109: 推導 channel, chatType
// Line 112-143: 評估規則匹配
//   匹配條件: channel, chatType, keyPrefix, rawKeyPrefix
// Line 145-150: 回傳決策或回退（"allow"）
```

Send policy 是一個規則引擎，按順序評估規則。每個規則可以匹配：
- 通道（channel）
- 對話型別（chatType）
- Session key 前綴
- 原始 session key 前綴

第一個匹配的規則決定發送決策（`"allow"` 或 `"deny"`）。若無匹配，預設為 `"allow"`。

---

## 7. 壓縮檢查點

```typescript
// source-repo/src/gateway/session-compaction-checkpoints.ts:17
// MAX_COMPACTION_CHECKPOINTS_PER_SESSION = 25

// source-repo/src/gateway/session-compaction-checkpoints.ts:56-105
// captureCompactionCheckpointSnapshot()
// - 複製轉錄檔
// - 擷取 leafId
// - 回傳 { sessionId, sessionFile, leafId }

// source-repo/src/gateway/session-compaction-checkpoints.ts:120-188
// persistSessionCompactionCheckpoint()
// 儲存: checkpointId, reason, token counts, summary
// 裁剪到最後 25 個

// source-repo/src/gateway/session-compaction-checkpoints.ts:190-207
// 查詢函式
```

壓縮檢查點（SessionCompactionCheckpoint）結構：
```
checkpointId: string
sessionKey: string
reason: "manual" | "timeout-retry" | "overflow-retry" | "auto-threshold"
createdAt: number
preCompaction: { sessionId, sessionFile, leafId }
postCompaction: { sessionId, sessionFile, leafId, entryId }
```

---

## 8. 跨元件互動流程

```
入站訊息
  │
  ├── Channel Plugin 提取 metadata
  │   └── peer, guild, team, account
  │
  ├── Send Policy 評估
  │   └── 允許 or 拒絕發送
  │
  ├── Routing Engine 決策
  │   ├── Binding 匹配（8 層）
  │   ├── Session Key 建構
  │   └── DM Scope 應用
  │
  ├── Session 載入/建立
  │   ├── 轉錄檔載入
  │   ├── 生命週期狀態更新 → "running"
  │   └── Presence 更新（Discord）
  │
  ├── Agent 執行
  │   ├── Context Engine (assemble)
  │   ├── LLM 推理
  │   └── Context Engine (ingest)
  │
  ├── 回應發送
  │   ├── Send Policy 再次評估
  │   └── Channel Plugin 送出
  │
  └── Session 結束
      ├── 生命週期狀態更新 → "done"/"failed"/"killed"
      ├── 轉錄檔歸檔
      └── Presence 更新
```

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/config/types.discord.ts:110-328` | Discord Presence 設定 |
| `source-repo/src/gateway/server-broadcast-types.ts:1-5` | 廣播狀態版本 |
| `source-repo/src/gateway/session-lifecycle-state.ts:1-169` | Session 生命週期完整 |
| `source-repo/src/gateway/protocol/schema/frames.ts:128-137` | 錯誤結構重試欄位 |
| `source-repo/src/gateway/session-compaction-checkpoints.ts:40-207` | 壓縮檢查點與重試 |
| `source-repo/src/gateway/config-reload.ts:133-148` | 設定重載重試 |
| `source-repo/src/gateway/session-archive.fs.ts:22-55` | Session 歸檔分類 |
| `source-repo/src/gateway/session-history-state.ts:11-219` | Session 歷史狀態 |
| `source-repo/docs.acp.md:1-211` | ACP 協定文件 |
| `source-repo/src/acp/session.ts:4-168` | ACP Session Store |
| `source-repo/src/sessions/send-policy.ts:9-151` | Send Policy 引擎 |
