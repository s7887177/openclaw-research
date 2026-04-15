# Session 概念、Session Key 與路由引擎

## 本章摘要

在 OpenClaw 中，每一則對話訊息的處理都始於**路由**——系統必須決定這則訊息應該交給哪個 agent、在哪個 session 上下文中處理。本章深入分析 session 的基礎概念、session key 的建構規則、以及 OpenClaw 獨特的 8 層匹配路由引擎。

---

## 1. Session 的基礎概念

### 1.1 什麼是 Session

Session 是 OpenClaw 中的**對話上下文容器**。一個 session 代表一段持續的對話，包含：
- 訊息歷史（transcript）
- Agent 的狀態
- 記憶上下文
- 壓縮檢查點

Session 的核心識別碼有兩個維度：
- **Session ID**：UUID 格式，用於磁碟上的檔案識別
- **Session Key**：正規化的路由鍵，用於邏輯識別

### 1.2 Session ID

```typescript
// source-repo/src/sessions/session-id.ts:1-5
// SESSION_ID_RE 正則驗證 UUID 格式
// /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i
// looksLikeSessionId(value) 函式
```

Session ID 使用標準的 UUID v4 格式，純粹用於唯一識別。

### 1.3 Session Label

```typescript
// source-repo/src/sessions/session-label.ts:1-20
// SESSION_LABEL_MAX_LENGTH = 512 字元
// parseSessionLabel() - 驗證字串、非空、不超過最大長度
```

Session label 是使用者可見的 session 名稱，最長 512 字元。

### 1.4 Session Chat Type

```typescript
// source-repo/src/sessions/session-chat-type.ts:12-24
// deriveSessionChatType()
// 使用 channel plugin 的回調來推導 session 的 chat type
```

Chat type 表示對話的性質——直接訊息（direct）、群組（group）、頻道（channel）等。

### 1.5 Session 生命週期事件

```typescript
// source-repo/src/sessions/session-lifecycle-events.ts:1-28
// SessionLifecycleEvent 型別：
// - sessionKey, reason, parentSessionKey, label, displayName
// 事件監聽系統（觀察者模式，使用 Set<SessionLifecycleListener>）
```

---

## 2. Session Key：路由的基石

### 2.1 Session Key 格式

Session Key 是 OpenClaw 路由系統的核心概念，格式為：

```
agent:<agentId>:<rest>
```

例如：
- `agent:assistant:main` — assistant agent 的主 session
- `agent:helper:discord:user123` — helper agent 在 Discord 上和 user123 的 session
- `agent:coder:acp:550e8400-e29b-41d4-a716-446655440000` — ACP 的 session

### 2.2 Session Key 解析

```typescript
// source-repo/src/sessions/session-key-utils.ts:7-10
type ParsedAgentSessionKey = {
  agentId: string;
  rest: string;
};

// source-repo/src/sessions/session-key-utils.ts:28-48
// parseAgentSessionKey()
// 解析 "agent:agentId:rest" 格式（小寫正規化）
```

### 2.3 特殊 Session Key 模式

OpenClaw 辨識多種特殊的 session key 模式：

**Cron Run Session**：
```typescript
// source-repo/src/sessions/session-key-utils.ts:50-56
// isCronRunSessionKey()
// 匹配模式: /^cron:[^:]+:run:[^:]+$/
```

**Subagent Session**：
```typescript
// source-repo/src/sessions/session-key-utils.ts:66-76
// isSubagentSessionKey()
// 偵測 "subagent:" 前綴

// source-repo/src/sessions/session-key-utils.ts:78-84
// getSubagentDepth()
// 計算 ":subagent:" 出現次數（巢狀深度）
```

**ACP Session**：
```typescript
// source-repo/src/sessions/session-key-utils.ts:86-97
// isAcpSessionKey()
// 偵測 "acp:" 前綴或 agent session rest 中的 acp
```

**Thread Session**：
```typescript
// source-repo/src/sessions/session-key-utils.ts:99-118
// parseThreadSessionSuffix()
// 提取 ":thread:threadId" 後綴與基底 key
```

---

## 3. 路由引擎

### 3.1 路由入口

路由的核心函式是 `resolveAgentRoute()`，位於 `src/routing/resolve-route.ts`（836 行）：

```typescript
// source-repo/src/routing/resolve-route.ts:27-38
export type ResolveAgentRouteInput = {
  cfg: OpenClawConfig;
  channel: string;
  accountId?: string | null;
  peer?: RoutePeer | null;
  parentPeer?: RoutePeer | null;  // 用於 thread 的父 peer
  guildId?: string | null;
  teamId?: string | null;
  memberRoleIds?: string[];       // Discord 成員角色 ID
};
```

### 3.2 路由結果

```typescript
// source-repo/src/routing/resolve-route.ts:40-61
export type ResolvedAgentRoute = {
  agentId: string;
  channel: string;
  accountId: string;
  sessionKey: string;
  mainSessionKey: string;
  lastRoutePolicy: "main" | "session";
  matchedBy:
    | "binding.peer"
    | "binding.peer.parent"
    | "binding.peer.wildcard"
    | "binding.guild+roles"
    | "binding.guild"
    | "binding.team"
    | "binding.account"
    | "binding.channel"
    | "default";
};
```

`matchedBy` 欄位記錄了路由是如何匹配的——對於除錯和日誌非常有價值。

### 3.3 八層匹配層級

路由引擎使用**八層匹配層級**，按優先順序從高到低評估（`source-repo/src/routing/resolve-route.ts:748-813`）：

| 層級 | matchedBy | 說明 | 行號 |
|------|-----------|------|------|
| 1 | `binding.peer` | 精確的 peer 匹配 | 755-761 |
| 2 | `binding.peer.parent` | Thread 父 peer 回退 | 762-768 |
| 3 | `binding.peer.wildcard` | Peer 種類萬用字元 | 769-775 |
| 4 | `binding.guild+roles` | Guild + 成員角色交集 | 776-783 |
| 5 | `binding.guild` | 僅 Guild | 784-791 |
| 6 | `binding.team` | Team ID | 792-798 |
| 7 | `binding.account` | 帳號範圍 | 799-805 |
| 8 | `binding.channel` | 通道回退 | 806-812 |

**第一個匹配勝出**（first match wins）。若無任何匹配，回傳預設 agent（`source-repo/src/routing/resolve-route.ts:815-832`）。

### 3.4 Binding 索引

為了高效匹配，路由引擎預先建構索引：

```typescript
// source-repo/src/routing/resolve-route.ts:373-421
// buildEvaluatedBindingsIndex()
// 建立查找 Map：
// - byPeer: 按 peer ID 索引
// - byGuildWithRoles: 按 guild ID + 角色索引
// - byGuild: 按 guild ID 索引
// - byTeam: 按 team ID 索引
// - byAccount: 按 account ID 索引
// - byChannel: 按 channel 索引
```

```typescript
// source-repo/src/routing/resolve-route.ts:239-280
// buildEvaluatedBindingsByChannel()
// 按 channel 然後按 account 建立索引
```

### 3.5 Binding 範圍匹配

```typescript
// source-repo/src/routing/resolve-route.ts:597-630
// matchesBindingScope()
// 評估 binding 是否與請求範圍匹配
```

---

## 4. Session Key 建構

### 4.1 主 Session Key

```typescript
// source-repo/src/routing/session-key.ts:120-127
// buildAgentMainSessionKey() → "agent:agentId:main"
```

每個 agent 都有一個「主 session」，用於不特定到任何 peer 的對話。

### 4.2 Peer Session Key

```typescript
// source-repo/src/routing/session-key.ts:129-176
// buildAgentPeerSessionKey()
// 依據 dmScope 建構不同粒度的 session key
```

**DM Scope** 控制 session 的隔離粒度（`source-repo/src/routing/session-key.ts:138`）：

| dmScope | Session Key 格式 | 說明 |
|---------|------------------|------|
| `"main"` | `agent:id:main` | 所有 peer 共用主 session |
| `"per-peer"` | `agent:id:peer:peerId` | 每個 peer 獨立 session |
| `"per-channel-peer"` | `agent:id:channel:peer:peerId` | 每個通道+peer 獨立 |
| `"per-account-channel-peer"` | `agent:id:account:channel:peer:peerId` | 最細粒度隔離 |

DM Scope 還支援**身份連結解析**（identity link resolution）（`source-repo/src/routing/session-key.ts:142-171`），用於跨通道的 peer 映射。例如，同一個使用者在 Discord 和 Telegram 上可以映射到同一個 session。

### 4.3 Thread Session Key

```typescript
// source-repo/src/routing/session-key.ts:236-254
// resolveThreadSessionKeys()
// 處理 ":thread:threadId" 後綴
```

Thread session 是在現有 session 之上附加 thread 上下文。

### 4.4 Session Key 分類

```typescript
// source-repo/src/routing/session-key.ts:78-89
// classifySessionKeyShape()
// 回傳：
//   "missing"              - 缺失
//   "agent"                - 標準 agent 格式
//   "legacy_or_alias"      - Legacy 或別名格式
//   "malformed_agent"      - 格式錯誤的 agent key
```

---

## 5. Account ID 正規化

### 5.1 正規化規則

```typescript
// source-repo/src/routing/account-id.ts:4
// DEFAULT_ACCOUNT_ID = "default"

// source-repo/src/routing/account-id.ts:15-25
// canonicalizeAccountId()
// 將 /[^a-z0-9_-]+/g 替換為 "-"，最長 64 字元

// source-repo/src/routing/account-id.ts:35-47
// normalizeAccountId() - 含快取
// 空值或無效值回傳 "default"
```

### 5.2 快取

```typescript
// source-repo/src/routing/account-id.ts:10-13
// 快取：FIFO 淘汰，最多 512 條目
```

Account ID 正規化是高頻操作，快取避免重複計算。

### 5.3 Account 查找

```typescript
// source-repo/src/routing/account-lookup.ts:3-34
// resolveAccountEntry() - 不區分大小寫查找
// resolveNormalizedAccountEntry() - 含正規化函式的查找
```

---

## 6. Bindings 系統

### 6.1 Binding 概念

Binding 是將「誰從哪個通道來」映射到「哪個 agent 處理」的規則。它定義了：
- 通道（channel）
- 帳號（account）
- Peer / Guild / Team / Role 等範圍條件
- 目標 agent ID

### 6.2 Binding 查找

```typescript
// source-repo/src/routing/bindings.ts:49-63
// listBoundAccountIds()
// 回傳排序後的已綁定 account ID 列表

// source-repo/src/routing/bindings.ts:88-104
// buildChannelAccountBindings()
// 建構巢狀 Map：channel → agent → accountIds[]

// source-repo/src/routing/bindings.ts:106-115
// resolvePreferredAccountId()
// 回傳第一個已綁定的或預設的 account ID
```

### 6.3 Agent ID 選擇

```typescript
// source-repo/src/routing/resolve-route.ts:153-168
// pickFirstExistingAgentId()
// 使用 WeakMap 快取（以 config 物件為 key）
// 選擇設定中第一個存在的 agent ID
```

---

## 7. 多 Agent 路由

### 7.1 子代理（Subagent）路由

子代理的 session key 包含 `subagent:` 標記：

```
agent:main:session123:subagent:helper:task456
```

巢狀深度可以透過 `getSubagentDepth()` 計算（`source-repo/src/sessions/session-key-utils.ts:78-84`）。

### 7.2 子代理重新啟動

```typescript
// source-repo/src/gateway/session-subagent-reactivation.ts:1-26
// reactivateCompletedSubagentSession()
// 透過替換 runId 來重新啟動已完成的子代理 session
```

### 7.3 Last Route Policy

```typescript
// source-repo/src/routing/resolve-route.ts:65-77
// deriveLastRoutePolicy()
// 決定入站的 last-route 更新應該寫入哪個 session
// "main" - 寫入主 session（sessionKey == mainSessionKey 時）
// "session" - 寫入當前 session
```

---

## 8. Input Provenance

```typescript
// source-repo/src/sessions/input-provenance.ts
// 追蹤訊息的來源（哪個通道、哪個帳號、哪個 peer）
```

Input provenance 讓系統知道每則訊息的確切來源，這對於審計和安全非常重要。

---

## 9. 路由的橫切模式

路由是 OpenClaw 中最典型的橫切機制之一，它觸及幾乎所有其他元件：

```
入站訊息
  │
  ├── Channel Plugin → 提取 peer, guild, team 資訊
  │
  ├── Account Lookup → 正規化 account ID
  │
  ├── Binding Evaluation → 八層匹配
  │   ├── Peer binding → 精確匹配
  │   ├── Guild binding → 伺服器匹配
  │   └── Channel binding → 通道回退
  │
  ├── Session Key Construction → 依 dmScope 建構
  │   ├── Identity Link Resolution → 跨通道映射
  │   └── Thread Suffix → ":thread:threadId"
  │
  ├── Context Engine → 記憶載入
  │
  └── Agent Runtime → 執行
```

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/sessions/session-id.ts:1-5` | Session ID 正則 |
| `source-repo/src/sessions/session-label.ts:1-20` | Session Label |
| `source-repo/src/sessions/session-chat-type.ts:12-24` | Chat Type 推導 |
| `source-repo/src/sessions/session-lifecycle-events.ts:1-28` | 生命週期事件 |
| `source-repo/src/sessions/session-key-utils.ts:7-118` | Session Key 解析工具 |
| `source-repo/src/routing/resolve-route.ts:27-61` | 路由輸入/輸出型別 |
| `source-repo/src/routing/resolve-route.ts:153-168` | Agent ID 選擇 |
| `source-repo/src/routing/resolve-route.ts:239-421` | Binding 索引建構 |
| `source-repo/src/routing/resolve-route.ts:597-630` | Binding 範圍匹配 |
| `source-repo/src/routing/resolve-route.ts:632-836` | resolveAgentRoute() 核心 |
| `source-repo/src/routing/resolve-route.ts:748-813` | 八層匹配邏輯 |
| `source-repo/src/routing/session-key.ts:78-176` | Session Key 建構 |
| `source-repo/src/routing/session-key.ts:236-254` | Thread Session Key |
| `source-repo/src/routing/account-id.ts:4-47` | Account ID 正規化 |
| `source-repo/src/routing/account-lookup.ts:3-34` | Account 查找 |
| `source-repo/src/routing/bindings.ts:49-115` | Binding 查找 |
| `source-repo/src/gateway/session-subagent-reactivation.ts:1-26` | 子代理重啟 |
