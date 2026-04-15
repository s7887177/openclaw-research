# ReAct 推理迴圈（二）：Hook 系統、串流回應、狀態管理與子 Agent

> **摘要**：本章接續前章的 ReAct 迴圈核心，深入四個進階主題：28 種 Plugin Hook 
> 如何串連整個推理流程、Agent 串流回應如何即時傳遞給使用者、多輪對話的狀態如何管理、
> 以及子 Agent（Sub-agent）如何被調度與回收。

---

## 1. Hook 系統：推理迴圈中的擴展點

### 1.1 Hook 在推理迴圈中的位置

28 種 Plugin Hook 分布在推理迴圈的每一個階段。以下按照執行順序排列：

**一、Session 開始**
```
session_start → before_model_resolve → before_prompt_build
```

**二、Agent 啟動**
```
before_agent_start → [建構 prompt] → before_agent_reply
```

**三、LLM 通訊**
```
llm_input → [呼叫 LLM] → llm_output
```

**四、工具調用（每個工具）**
```
before_tool_call → [執行工具] → after_tool_call → tool_result_persist
```

**五、訊息處理**
```
inbound_claim → message_received → before_message_write
→ message_sending → message_sent
```

**六、子 Agent**
```
subagent_spawning → subagent_delivery_target → subagent_spawned → subagent_ended
```

**七、壓縮**
```
before_compaction → [壓縮] → after_compaction
```

**八、Session 結束**
```
agent_end → session_end → before_reset
```

**九、系統級**
```
gateway_start → gateway_stop → before_dispatch → reply_dispatch → before_install
```

### 1.2 Hook Context

每個 Hook 都接收一個上下文物件，攜帶當前的執行狀態：

```typescript
// source-repo/src/plugins/hook-types.ts:140-151
export type PluginHookAgentContext = {
  runId?: string;
  agentId?: string;
  sessionKey?: string;
  sessionId?: string;
  workspaceDir?: string;
  modelProviderId?: string;
  modelId?: string;
  messageProvider?: string;
  trigger?: string;
  channelId?: string;
};
```

### 1.3 重要 Hook 詳解

**before_agent_start**：在 Agent 開始推理之前觸發。Plugin 可以修改系統 prompt、
注入額外上下文、或直接接管回覆。

**llm_input / llm_output**：記錄 LLM 的輸入和輸出，用於監控、除錯和分析。

```typescript
// source-repo/src/plugins/hook-types.ts:163-188
export type PluginHookLlmInputEvent = {
  runId: string;
  sessionId: string;
  provider: string;
  model: string;
  systemPrompt?: string;
  prompt: string;
  historyMessages: unknown[];
  imagesCount: number;
};

export type PluginHookLlmOutputEvent = {
  runId: string;
  sessionId: string;
  provider: string;
  model: string;
  assistantTexts: string[];
  lastAssistant?: unknown;
  usage?: {
    input?: number;
    output?: number;
    cacheRead?: number;
    cacheWrite?: number;
    total?: number;
  };
};
```

**before_compaction / after_compaction**：壓縮前後的擴展點。

```typescript
// source-repo/src/plugins/hook-types.ts:197-200+
export type PluginHookBeforeCompactionEvent = {
  messageCount: number;
  compactingCount?: number;
  tokenCount?: number;
};
```

**agent_end**：Agent 完成時觸發，包含成功/失敗資訊和執行時間。

```typescript
// source-repo/src/plugins/hook-types.ts:190-195
export type PluginHookAgentEndEvent = {
  messages: unknown[];
  success: boolean;
  error?: string;
  durationMs?: number;
};
```

### 1.4 Hook 的執行保證

- **修改式 Hook**：依序執行，結果會被合併。支援短路（Short-circuit）。
- **通知式 Hook**：fire-and-forget，錯誤不會影響主流程。
- **安全 Hook**：`before_prompt_build` 和 `before_agent_start` 被標記為
  Prompt Injection Hook（`source-repo/src/plugins/hook-types.ts:128-131`），
  結果會受到額外安全檢查。

---

## 2. 串流回應：即時傳遞

### 2.1 Draft Stream Loop

當 Agent 正在思考和生成回覆時，串流回應讓使用者能即時看到部分結果。
`DraftStreamLoop` 管理這個流程：

```typescript
// source-repo/src/channels/draft-stream-loop.ts:1-104
export type DraftStreamLoop = {
  update: (text: string) => void;           // 更新文字
  flush: () => Promise<void>;               // 刷新待發送內容
  stop: () => void;                         // 停止串流
  resetPending: () => void;                 // 重置待發送緩衝
  resetThrottleWindow: () => void;          // 重置節流窗口
  waitForInFlight: () => Promise<void>;     // 等待所有進行中的發送完成
};

export function createDraftStreamLoop(params: {
  throttleMs: number;                                          // 節流間隔
  isStopped: () => boolean;                                    // 是否已停止
  sendOrEditStreamMessage: (text: string) => Promise<void | boolean>; // 發送函式
}): DraftStreamLoop
```

**關鍵設計**：
- **節流（Throttle）**：不是每收到一個 token 就發送更新，而是等待 `throttleMs`
 （例如 300ms）後才發送一次。這避免了對平台 API 的過度呼叫。
- **進行中追蹤（In-flight Tracking）**：追蹤正在發送的 Promise，
  確保不會出現競態條件（Race Condition）。
- **優雅刷新（Graceful Flush）**：在 Agent 完成回覆時，刷新所有剩餘的文字。
- **待發送緩衝（Pending Buffer）**：累積收到的文字，在下次節流窗口一次發送。

### 2.2 Draft Stream Controls

```typescript
// source-repo/src/channels/draft-stream-controls.ts（概要）
// 控制串流的開始、更新和停止
// 與 Channel 的 streaming adapter 整合
```

### 2.3 Channel Streaming 配置

每個 Channel 可以自訂串流行為：

```typescript
// source-repo/src/plugin-sdk/channel-streaming.ts:83-97
// resolveChannelStreamingChunkMode() — 分塊模式
// resolveChannelStreamingBlockEnabled() — 是否啟用區塊串流
// resolveChannelStreamingBlockCoalesce() — 合併設定
```

---

## 3. 多輪對話狀態管理

### 3.1 Session 與 Session Key

OpenClaw 使用兩層標識符管理對話狀態：

- **`sessionId`**：短暫的 UUID，每次 `/reset` 或 `/new` 命令時更新。
  代表一個「對話回合」（conversation round）。
- **`sessionKey`**：持久的對話識別碼。代表一個「對話頻道」
 （conversation channel）——例如「使用者 A 在 Discord 伺服器 B 的 #general 頻道」。

### 3.2 Session 生命週期事件

```
// source-repo/src/sessions/session-lifecycle-events.ts
// 管理 session 的建立、恢復、結束事件
```

### 3.3 Transcript 管理

```
// source-repo/src/sessions/transcript-events.ts
// 管理對話記錄：訊息的新增、讀取、壓縮
```

### 3.4 Context Engine

Context Engine 負責決定在有限的上下文窗口中放入什麼內容：

```
// source-repo/src/context-engine/
// - index.ts      — 主入口
// - types.ts      — 型別定義
// - registry.ts   — Context Engine registry
// - delegate.ts   — 委派邏輯
```

在主迴圈中的使用：

```typescript
// source-repo/src/agents/pi-embedded-runner/run.ts:522
const contextEngine = await resolveContextEngine(params.config);
```

Context Engine 的職責：
1. **Session 歷史管理**：決定保留多少對話歷史
2. **Token 預算分配**：將上下文窗口分配給不同內容（系統 prompt、歷史、工具定義等）
3. **壓縮決策**：決定何時觸發壓縮
4. **上下文注入**：將外部知識注入到 prompt 中

---

## 4. 子 Agent 調度

### 4.1 子 Agent 的概念

OpenClaw 支援「子 Agent」（Sub-agent）機制——一個 Agent 可以生成另一個 Agent 
來處理子任務。這在複雜的多步驟任務中特別有用。

### 4.2 子 Agent 生命週期 Hook

子 Agent 有四個專用 Hook：

```typescript
// source-repo/src/plugins/hook-types.ts:55-84（相關 Hook）
| "subagent_spawning"          // 子 Agent 準備生成
| "subagent_delivery_target"   // 決定子 Agent 的遞送目標
| "subagent_spawned"           // 子 Agent 已生成
| "subagent_ended"             // 子 Agent 已結束
```

### 4.3 子 Agent Session Key

子 Agent 使用特殊前綴的 Session Key，讓路由系統能區分主 Agent 和子 Agent：

```
// source-repo/src/routing/session-key.ts
// isSubagentSessionKey() — 檢測是否為子 Agent session
```

---

## 5. Run State Machine

### 5.1 Agent 執行狀態機

`RunStateMachine` 追蹤 Agent 的執行狀態：

```typescript
// source-repo/src/channels/run-state-machine.ts:1-99
export function createRunStateMachine(params: RunStateMachineParams) {
  return {
    isActive(): boolean { return lifecycleActive; },
    onRunStart(): void {
      activeRuns += 1;
      publish();           // 發佈「忙碌」狀態
      ensureHeartbeat();   // 啟動心跳
    },
    onRunEnd(): void {
      activeRuns = Math.max(0, activeRuns - 1);
      if (activeRuns <= 0) clearHeartbeat();
      publish();
    },
    deactivate(): void {
      lifecycleActive = false;
      clearHeartbeat();
    },
  };
}
```

**心跳機制**：當 Agent 正在執行時，每 60 秒發送一次心跳。
這讓 Channel 知道 Agent 仍在處理（例如顯示「正在輸入...」狀態）。

### 5.2 多 Run 並行

`activeRuns` 計數器支援多個 Agent run 同時進行。只有當所有 run 都結束時，
心跳才會停止。

---

## 6. 路由：訊息如何到達 Agent

### 6.1 Route Resolution

```
// source-repo/src/routing/resolve-route.ts
// 根據訊息的來源、目標和上下文，解析出正確的 Agent route
```

### 6.2 Session Key Generation

```
// source-repo/src/routing/session-key.ts
// 從 Channel、Account、Target 等資訊生成唯一的 session key
```

### 6.3 Account Routing

```
// source-repo/src/routing/account-id.ts
// source-repo/src/routing/account-lookup.ts
// 基於帳號的訊息路由
```

---

## 7. Agent Harness：Agent 的運行框架

### 7.1 Harness 概念

Agent Harness 是 Agent 的「運行框架」——它提供了啟動、停止、排隊訊息等能力：

```typescript
// source-repo/src/plugin-sdk/agent-harness.ts:4-57
export type AgentHarness = { ... };
export type AgentHarnessAttemptParams = { ... };
export type AgentHarnessAttemptResult = { ... };

export function resolveEmbeddedAgentRuntime(): ...
export function abortEmbeddedPiRun(): ...
export function queueAgentHarnessMessage(): ...
```

### 7.2 內建 PI Harness

OpenClaw 的內建 Agent 使用 PI（Personal Intelligence）框架：

```
// source-repo/src/agents/harness/builtin-pi.ts
// 內建的 PI embedded agent harness
```

---

## 8. 完整推理流程：端到端追蹤

以下是一則使用者訊息的完整處理流程：

```
使用者在 Discord 發送 "幫我查天氣"
    │
    ▼
1. Discord Channel 收到 Webhook
   └─ webhook-ingress 驗證、限流
    │
    ▼
2. Inbound Envelope Builder 正規化訊息
   └─ { channel: "discord", from: "user123", body: "幫我查天氣" }
    │
    ▼
3. 路由系統解析 Route 和 Session Key
   └─ session_key: "discord:guild456:channel789"
    │
    ▼
4. Hook: session_start
    │
    ▼
5. Hook: before_model_resolve → before_prompt_build
    │
    ▼
6. Hook: before_agent_start
   └─ Plugin 可以注入額外 prompt 或直接接管
    │
    ▼
7. === ReAct 迴圈開始 ===
   │
   │  Hook: llm_input → 發送到 LLM
   │
   │  LLM 回應："我需要調用天氣工具"
   │
   │  Hook: llm_output
   │
   │  LLM 包含 tool_call: { name: "weather", params: { city: "台北" } }
   │
   │  Hook: before_tool_call
   │  ├─ 執行策略檢查（exec-policy）
   │  ├─ 不需要核准（weather 是安全工具）
   │  └─ 通過
   │
   │  執行 weather 工具 → { temperature: 28, condition: "晴天" }
   │
   │  Hook: after_tool_call (fire-and-forget)
   │  Hook: tool_result_persist
   │
   │  結果加入對話歷史 → 繼續迴圈
   │
   │  Hook: llm_input → 發送到 LLM（含工具結果）
   │
   │  LLM 回應："台北目前 28°C，晴天。" (stopReason: "end_turn")
   │
   │  Hook: llm_output
   │
   │=== ReAct 迴圈結束 ===
    │
    ▼
8. Hook: agent_end
    │
    ▼
9. Hook: before_agent_reply
    │
    ▼
10. Reply Pipeline 組裝回覆
    └─ Channel Streaming → Draft Stream Loop
    │
    ▼
11. Hook: message_sending → 發送到 Discord → Hook: message_sent
    │
    ▼
12. Hook: session_end

使用者在 Discord 看到 "台北目前 28°C，晴天。"
```

---

## 9. 跨元件協作總結

ReAct 推理迴圈是 OpenClaw 所有橫切機制的交匯點：

- **Plugin 系統**提供了 Hook 擴展點和 Tool 註冊
- **Channel Adapter** 負責入站正規化和出站遞送
- **Provider Abstraction** 負責 LLM 通訊和串流處理
- **ReAct 迴圈本身** 編排整個推理過程

這四個機制的協作構成了 OpenClaw 的「智慧」——它不只是一個 LLM 包裝器，
而是一個完整的 Agent 運行時（Agent Runtime），支援工具調用、上下文管理、
多輪對話、子 Agent、和跨平台遞送。

---

## 引用來源

| 來源檔案 | 行號 | 內容 |
|---------|------|------|
| `source-repo/src/channels/draft-stream-loop.ts` | 1-104 | Draft Stream Loop |
| `source-repo/src/channels/draft-stream-controls.ts` | — | Stream 控制 |
| `source-repo/src/channels/run-state-machine.ts` | 1-99 | Run State Machine |
| `source-repo/src/plugin-sdk/channel-streaming.ts` | 83-97 | Streaming 配置 |
| `source-repo/src/plugin-sdk/agent-harness.ts` | 4-57 | Agent Harness |
| `source-repo/src/plugin-sdk/agent-runtime.ts` | — | Agent Runtime |
| `source-repo/src/plugin-sdk/agent-config-primitives.ts` | — | Agent Config |
| `source-repo/src/plugins/hook-types.ts` | 140-200+ | Hook Context 與事件型別 |
| `source-repo/src/plugins/hook-types.ts` | 128-131 | Prompt Injection Hook |
| `source-repo/src/sessions/session-lifecycle-events.ts` | — | Session 生命週期 |
| `source-repo/src/sessions/transcript-events.ts` | — | Transcript 管理 |
| `source-repo/src/context-engine/` | — | Context Engine |
| `source-repo/src/routing/resolve-route.ts` | — | Route 解析 |
| `source-repo/src/routing/session-key.ts` | — | Session Key 生成 |
| `source-repo/src/agents/pi-embedded-runner/run.ts` | 522 | Context Engine 使用 |
| `source-repo/src/agents/harness/builtin-pi.ts` | — | 內建 PI Harness |
