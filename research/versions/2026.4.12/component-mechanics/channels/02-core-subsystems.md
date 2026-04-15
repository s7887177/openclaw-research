# Channels 子章節 02：核心子系統

> **引用範圍**：`src/channels/status-reactions.ts`, `mention-gating.ts`, `typing.ts`, `run-state-machine.ts`, `draft-stream-controls.ts`

## 1. 狀態反應系統

### 1.1 概觀

狀態反應（Status Reactions）透過訊息的 Emoji 反應來顯示 Agent 的目前狀態。這是一個**通道無關**的抽象層，各平台透過 `StatusReactionAdapter` 接入。

### 1.2 StatusReactionAdapter

```typescript
// source-repo/src/channels/status-reactions.ts:12-17
export type StatusReactionAdapter = {
  setReaction: (emoji: string) => Promise<void>;
  removeReaction?: (emoji: string) => Promise<void>;  // Discord 等需要
};
```

`removeReaction` 是可選的，因為有些平台（如 Telegram）的反應是**替換式**（設新反應自動清除舊反應），而有些（如 Discord）需要明確移除。

> **引用來源**：`source-repo/src/channels/status-reactions.ts:12-17`

### 1.3 預設 Emoji

```typescript
// source-repo/src/channels/status-reactions.ts:57-68
export const DEFAULT_EMOJIS: Required<StatusReactionEmojis> = {
  queued:     "👀",   // 排隊等待
  thinking:   "🤔",   // LLM 思考中
  tool:       "🔥",   // 工具呼叫中
  coding:     "👨‍💻",   // 寫程式
  web:        "⚡",   // 瀏覽網頁
  done:       "👍",   // 完成
  error:      "😱",   // 錯誤
  stallSoft:  "🥱",   // 輕度卡頓（10秒）
  stallHard:  "😨",   // 嚴重卡頓（30秒）
  compacting: "✍",    // Context 壓縮中
};
```

> **引用來源**：`source-repo/src/channels/status-reactions.ts:57-68`

### 1.4 時序常數

```typescript
// source-repo/src/channels/status-reactions.ts:70-76
export const DEFAULT_TIMING: Required<StatusReactionTiming> = {
  debounceMs:  700,      // 狀態切換防抖
  stallSoftMs: 10_000,   // 10 秒：輕度卡頓
  stallHardMs: 30_000,   // 30 秒：嚴重卡頓
  doneHoldMs:  1_500,    // 完成後保持 1.5 秒
  errorHoldMs: 2_500,    // 錯誤後保持 2.5 秒
};
```

**設計解讀**：
- `debounceMs=700` 防止快速狀態切換導致 Emoji 閃爍
- `stallSoftMs=10s` 和 `stallHardMs=30s` 形成**兩級告警**，讓使用者知道 Agent 可能卡住
- `doneHoldMs` 和 `errorHoldMs` 確保使用者有時間看到最終狀態

> **引用來源**：`source-repo/src/channels/status-reactions.ts:70-76`

### 1.5 StatusReactionController

```typescript
// source-repo/src/channels/status-reactions.ts:40-51
export type StatusReactionController = {
  setQueued: () => Promise<void> | void;
  setThinking: () => Promise<void> | void;
  setTool: (toolName?: string) => Promise<void> | void;
  setCompacting: () => Promise<void> | void;
  cancelPending: () => void;       // 取消待處理的防抖
  setDone: () => Promise<void>;
  setError: () => Promise<void>;
  clear: () => Promise<void>;
  restoreInitial: () => Promise<void>;
};
```

Controller 是狀態機的使用者介面。注意 `setTool()` 接受可選的 `toolName`，用於根據工具類型選擇不同的 Emoji（如 `coding` vs `web`）。

### 1.6 工具類型偵測

```typescript
// source-repo/src/channels/status-reactions.ts:78-80
export const CODING_TOOL_TOKENS: string[] = [
  "exec", "process", ...
];
```

系統維護一個「coding 工具」的 token 列表，當工具名稱包含這些 token 時，使用 👨‍💻 代替 🔥。類似地，web 相關工具使用 ⚡。

> **引用來源**：`source-repo/src/channels/status-reactions.ts:78-80`

---

## 2. Mention 門控

### 2.1 概觀

Mention 門控決定 Agent **是否應該回應**某條訊息。在群聊中，Agent 預設只回應被 @提及 的訊息。

### 2.2 新版 API：resolveInboundMentionDecision()

```typescript
// source-repo/src/channels/mention-gating.ts:59-66
export type ResolveInboundMentionDecisionNestedParams = {
  facts: InboundMentionFacts;
  policy: InboundMentionPolicy;
};
```

新版 API 將**事實**（facts）與**策略**（policy）分離：

**事實（Facts）**：
```typescript
// source-repo/src/channels/mention-gating.ts:40-45
export type InboundMentionFacts = {
  canDetectMention: boolean;          // 平台是否支援 mention 偵測
  wasMentioned: boolean;              // 是否被明確 @提及
  hasAnyMention?: boolean;            // 是否有任何 mention
  implicitMentionKinds?: readonly InboundImplicitMentionKind[];  // 隱式 mention 類型
};
```

**策略（Policy）**：
```typescript
// source-repo/src/channels/mention-gating.ts:47-54
export type InboundMentionPolicy = {
  isGroup: boolean;                   // 是否為群聊
  requireMention: boolean;            // 是否要求 mention
  allowedImplicitMentionKinds?: readonly InboundImplicitMentionKind[];
  allowTextCommands: boolean;         // 是否允許文字指令
  hasControlCommand: boolean;         // 是否包含控制指令
  commandAuthorized: boolean;         // 指令是否已授權
};
```

> **引用來源**：`source-repo/src/channels/mention-gating.ts:40-66`

### 2.3 隱式 Mention

```typescript
// source-repo/src/channels/mention-gating.ts:34-38
export type InboundImplicitMentionKind =
  | "reply_to_bot"            // 回覆 Bot 的訊息
  | "quoted_bot"              // 引用 Bot 的訊息
  | "bot_thread_participant"  // Bot 是 Thread 參與者
  | "native";                 // 平台原生隱式 mention
```

隱式 Mention 是 OpenClaw 的關鍵設計之一。即使使用者沒有 @Bot，以下情況也視為「提及」：

1. **reply_to_bot**：使用者回覆了 Bot 之前的訊息
2. **quoted_bot**：使用者引用了 Bot 的訊息
3. **bot_thread_participant**：Bot 已經在該 Thread 中參與對話
4. **native**：平台原生的隱式提及機制

```typescript
// source-repo/src/channels/mention-gating.ts:74-79
export function implicitMentionKindWhen(
  kind: InboundImplicitMentionKind,
  enabled: boolean,
): InboundImplicitMentionKind[] {
  return enabled ? [kind] : [];
}
```

> **引用來源**：`source-repo/src/channels/mention-gating.ts:34-38, 74-79`

### 2.4 決策輸出

```typescript
// source-repo/src/channels/mention-gating.ts:68-72
export type InboundMentionDecision = MentionGateResult & {
  implicitMention: boolean;
  matchedImplicitMentionKinds: InboundImplicitMentionKind[];
  shouldBypassMention: boolean;
};
```

---

## 3. 打字指示系統

### 3.1 概觀

打字指示（Typing Indicator）在 Agent 處理訊息時顯示「正在輸入...」，讓使用者知道 Bot 正在工作。

### 3.2 TypingCallbacks

```typescript
// source-repo/src/channels/typing.ts:4-9
export type TypingCallbacks = {
  onReplyStart: () => Promise<void>;    // 開始打字
  onIdle?: () => void;                  // 閒置
  onCleanup?: () => void;              // 清理（NO_REPLY 時）
};
```

### 3.3 建立參數

```typescript
// source-repo/src/channels/typing.ts:11-21
export type CreateTypingCallbacksParams = {
  start: () => Promise<void>;           // 平台 API：開始打字
  stop?: () => Promise<void>;           // 平台 API：停止打字
  onStartError: (err: unknown) => void;
  onStopError?: (err: unknown) => void;
  keepaliveIntervalMs?: number;         // 預設 3,000ms
  maxConsecutiveFailures?: number;      // 預設 2
  maxDurationMs?: number;               // 預設 60,000ms (1分鐘)
};
```

> **引用來源**：`source-repo/src/channels/typing.ts:4-21`

### 3.4 內部機制

打字指示由三個元件組成：

```
createTypingCallbacks()
├── createTypingStartGuard()     // 啟動防護（失敗計數器）
├── createTypingKeepaliveLoop()  // 保活迴圈（每 3 秒重發）
└── TTL 計時器                    // 最長 60 秒自動停止
```

**啟動防護**：連續失敗 2 次後停止嘗試，避免無意義的 API 呼叫。

**保活迴圈**：多數平台的「正在輸入」狀態有 TTL（通常 5-10 秒），需要定期重發。

**安全 TTL**：即使 Agent 長時間運行，打字指示最多持續 60 秒，避免「永遠在輸入」的異常狀態。

> **引用來源**：`source-repo/src/channels/typing.ts:23-60`

---

## 4. 入站防抖策略

```
src/channels/inbound-debounce-policy.ts
```

入站防抖用於處理使用者**快速連續發送多條訊息**的情況。系統會等待短暫的靜默期，將多條訊息合併後再交給 Agent 處理。

這避免了 Agent 逐條回覆造成的碎片化體驗，特別是在行動裝置上使用者習慣分段輸入的場景。

---

## 5. 草稿串流控制

```
src/channels/draft-stream-controls.ts
src/channels/draft-stream-loop.ts
```

草稿串流是 OpenClaw 在支援的平台上**即時顯示 Agent 回覆**的機制。Agent 的回覆一邊生成一邊更新訊息，類似 ChatGPT 的串流效果。

控制參數包括：
- 合併閾值（最少多少字元才更新）
- 閒置超時（多久沒新內容就發送當前內容）
- 區塊串流模式（逐區塊更新 vs 全文替換）

---

## 6. 執行狀態機

```
src/channels/run-state-machine.ts
```

每個通道的訊息處理會經過一個狀態機：

```
IDLE → QUEUED → PROCESSING → STREAMING → DONE
                    │                       │
                    └── ERROR ──────────────┘
```

狀態機驅動：
- 狀態反應 Emoji 切換
- 打字指示的啟停
- 草稿串流的開始/結束
- 逾時偵測

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| StatusReactionAdapter | `source-repo/src/channels/status-reactions.ts` | 12-17 |
| StatusReactionEmojis | `source-repo/src/channels/status-reactions.ts` | 19-30 |
| DEFAULT_EMOJIS | `source-repo/src/channels/status-reactions.ts` | 57-68 |
| DEFAULT_TIMING | `source-repo/src/channels/status-reactions.ts` | 70-76 |
| StatusReactionController | `source-repo/src/channels/status-reactions.ts` | 40-51 |
| CODING_TOOL_TOKENS | `source-repo/src/channels/status-reactions.ts` | 78-80 |
| InboundImplicitMentionKind | `source-repo/src/channels/mention-gating.ts` | 34-38 |
| InboundMentionFacts | `source-repo/src/channels/mention-gating.ts` | 40-45 |
| InboundMentionPolicy | `source-repo/src/channels/mention-gating.ts` | 47-54 |
| InboundMentionDecision | `source-repo/src/channels/mention-gating.ts` | 68-72 |
| implicitMentionKindWhen() | `source-repo/src/channels/mention-gating.ts` | 74-79 |
| TypingCallbacks | `source-repo/src/channels/typing.ts` | 4-9 |
| CreateTypingCallbacksParams | `source-repo/src/channels/typing.ts` | 11-21 |
| 打字內部機制 | `source-repo/src/channels/typing.ts` | 23-60 |
