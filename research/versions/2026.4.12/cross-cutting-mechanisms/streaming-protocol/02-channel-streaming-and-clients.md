# 通道級串流差異、客戶端消費與錯誤處理

## 本章摘要

上一章介紹了 Gateway 層的協定框架與 SSE 串流機制。本章向下深入到**通道層**（channel layer）——不同的聊天平台（Discord、Telegram、Web UI 等）對串流有截然不同的支援程度。OpenClaw 透過一套抽象的串流控制機制來統一處理這些差異。同時，我們也探討客戶端如何消費串流、以及整個串流管線的錯誤處理與重連策略。

---

## 1. 通道級串流配置

### 1.1 串流型別定義

每個通道可以配置不同的串流模式（`source-repo/src/plugin-sdk/channel-streaming.ts:1-110`）：

| 模式 | 說明 |
|------|------|
| `"block"` | 區塊級串流——整段文字一次送出 |
| `"partial"` | 漸進式預覽串流——邊產生邊顯示 |
| `"off"` | 不串流——等待完整回應後一次性送出 |

此外還有 **chunk 模式**設定：
- `"length"` — 以固定長度分塊
- `"newline"` — 以換行符分塊

以及**合併（coalescing）配置**，用於將多個小更新合併為一次傳送。

### 1.2 為什麼需要通道級串流差異

不同平台的限制截然不同：

- **Discord**：支援訊息編輯，可以實現「打字中」效果。但有速率限制，不能每毫秒更新
- **Telegram**：支援訊息編輯，但 API 有嚴格的速率限制
- **Web UI**：透過 WebSocket 直接串流，延遲最低
- **CLI**：透過 stdio 串流，逐字元輸出
- **Slack**：透過 Block Kit 更新，有 API 呼叫限制
- **WhatsApp / Signal**：不支援訊息編輯，只能「一次性送出」模式

---

## 2. Draft Stream 控制

### 2.1 概念

**Draft Stream** 是 OpenClaw 處理通道級串流的核心抽象。「草稿」（draft）指的是一個正在被串流更新的訊息——它會隨著 LLM 的輸出逐步更新，直到最終完成。

### 2.2 控制結構

```typescript
// source-repo/src/channels/draft-stream-controls.ts:1-100
type FinalizableDraftStreamState = {
  stopped: boolean;
  final: boolean;
};

export function createFinalizableDraftStreamControls(params: {
  throttleMs: number;                                        // 節流間隔
  isStopped: () => boolean;                                  // 是否已停止
  isFinal: () => boolean;                                    // 是否已完成
  markStopped: () => void;                                   // 標記停止
  markFinal: () => void;                                     // 標記完成
  sendOrEditStreamMessage: (text: string) => Promise<boolean>; // 送出或編輯訊息
})
```

### 2.3 節流邏輯

Draft Stream 需要節流（throttle）來避免超過平台的 API 速率限制。例如 Discord 的 API 通常限制每秒 5 次更新。

### 2.4 串流迴圈

```typescript
// source-repo/src/channels/draft-stream-loop.ts:1-110
// 實作智慧批次處理（intelligent batching）
```

串流迴圈是一個狀態機（`source-repo/src/channels/draft-stream-loop.ts:14-62`）：

| 方法 | 說明 |
|------|------|
| `update()` | 累積文字，排程 flush |
| `flush()` | 當準備好時送出累積的文字 |
| `stop()` | 停止串流，清除待處理緩衝區 |
| `resetThrottleWindow()` | 重置計時，允許立即送出 |

關鍵設計：
- **節流視窗**（Throttle Window）：確保兩次傳送之間有最小延遲
- **飛行中追蹤**（In-Flight Tracking）：等待待處理的傳送完成後，才排隊下一次更新
- **累積與合併**：在節流視窗內的多次更新會被合併為一次傳送

這種設計讓串流在各平台上都能呈現流暢的效果，同時不會觸發速率限制。

---

## 3. Provider Stream 抽象

### 3.1 串流包裝器

LLM Provider 的串流回應需要經過一層抽象才能統一處理。這個抽象位於 `src/plugin-sdk/provider-stream-shared.ts`：

```typescript
// source-repo/src/plugin-sdk/provider-stream-shared.ts:1-120
export type ProviderStreamWrapperFactory =
  | ((streamFn: StreamFn | undefined) => StreamFn | undefined)
  | null | undefined | false;

export function composeProviderStreamWrappers(
  baseStreamFn: StreamFn | undefined,
  ...wrappers: ProviderStreamWrapperFactory[]
): StreamFn | undefined
```

### 3.2 組合模式

`composeProviderStreamWrappers` 使用函式組合模式（function composition），將多個串流轉換器串接起來：

```
原始 LLM 串流
  → Wrapper 1（例如：記錄日誌）
    → Wrapper 2（例如：轉換格式）
      → Wrapper 3（例如：速率限制）
        → 最終輸出
```

每個 wrapper 是一個工廠函式，接收上一層的 `StreamFn` 並回傳一個新的 `StreamFn`。如果 wrapper 回傳 `undefined`、`null` 或 `false`，則跳過該層。

---

## 4. 客戶端消費模式

### 4.1 Web UI 消費

Web UI 透過 WebSocket 直接與 Gateway 通訊。消費流程：

1. 建立 WebSocket 連線（攜帶 `ConnectParams`）
2. 完成挑戰-回應握手
3. 發送 `req` 框架（如 `chat.send`）
4. 接收 `event` 框架（包含串流文字差量）
5. 接收 `res` 框架（回合結束）

事件包含 `seq` 序列號，客戶端可以偵測事件是否遺漏。

### 4.2 CLI 消費

CLI 客戶端同樣透過 WebSocket 連線，但最終輸出是 stdio。CLI 將 WebSocket 的事件框架轉換為終端機的逐字元輸出。

### 4.3 HTTP API 消費

透過 `/v1/chat/completions`（OpenAI 相容）或 Open Responses API 的 SSE 串流：

1. 發送 HTTP POST 請求（`stream: true`）
2. 接收 SSE 事件（`data: {...}\n\n`）
3. 最後接收 `data: [DONE]\n\n` 標記結束

### 4.4 Session History SSE

用於即時同步 session 歷史的 SSE 端點：

1. 建立 SSE 連線
2. 接收 `history` 事件（初始快照）
3. 持續接收 `message` 事件（增量更新）
4. 每 15 秒接收心跳保活
5. 連線斷開後，客戶端在 1000ms 後自動重連

---

## 5. 錯誤處理

### 5.1 框架級錯誤

回應框架中的錯誤結構（`source-repo/src/gateway/protocol/schema/frames.ts:128-137`）提供了豐富的上下文：

```json
{
  "type": "res",
  "id": "req-123",
  "ok": false,
  "error": {
    "code": "rate_limited",
    "message": "Too many requests",
    "retryable": true,
    "retryAfterMs": 5000
  }
}
```

客戶端可以根據 `retryable` 和 `retryAfterMs` 做出智慧重試決策。

### 5.2 SSE 錯誤回退

在 SSE 串流中，如果在串流開始前發生錯誤，系統會回退到非串流的 JSON 錯誤回應（`source-repo/src/gateway/openresponses-http.ts:793-797`）。串流已開始後的錯誤則透過 SSE 事件傳遞。

### 5.3 連線級錯誤

WebSocket 連線的異常關閉會被詳細記錄：
- 關閉原因（close cause）
- 最後一個框架的型別、方法和 ID
- 連線持續時間
- 客戶端 IP 和 User Agent

---

## 6. 重連機制

### 6.1 SSE 重連

SSE 協定本身就支援自動重連。OpenClaw 透過 `retry` 指令設定重連間隔：

```
retry: 1000\n\n
```

客戶端（瀏覽器或 SDK）在連線斷開後會自動等待 1000ms 後重連。

### 6.2 WebSocket 重連

WebSocket 的重連由客戶端控制。Gateway 提供以下支援：

**重連閘門**（Reconnect Gating，`source-repo/src/gateway/reconnect-gating.ts`）：
控制客戶端的重連行為，防止雷暴效應（thundering herd）。

**關機通知**（Shutdown Event）：
Gateway 在計劃關機時發送 `shutdown` 事件，包含預期的重啟時間：

```json
{
  "type": "event",
  "event": "shutdown",
  "payload": {
    "reason": "config-reload",
    "restartExpectedMs": 5000
  }
}
```

客戶端可以據此在適當時機重連。

### 6.3 建議的重試步驟

錯誤碼的 schema 中定義了建議的重試步驟：
- `"retry_with_device_token"` — 使用裝置 token 重試
- `"wait_then_retry"` — 等待後重試

---

## 7. 完整串流管線

將所有層次串在一起：

```
LLM Provider
  │
  ▼
Provider Stream (src/plugin-sdk/provider-stream-shared.ts)
  │ composeProviderStreamWrappers()
  ▼
Agent Runtime
  │ onAgentEvent()
  ▼
Gateway Layer
  ├─ WebSocket: EventFrame { type:"event", event:"assistant", payload:{delta} }
  ├─ OpenAI SSE: data: {"choices":[{"delta":{"content":"..."}}]}
  └─ OpenResponses SSE: event: response.output_text.delta
  │
  ▼
Channel Layer (src/channels/draft-stream-controls.ts)
  │ createFinalizableDraftStreamControls()
  │ Draft Stream Loop (throttle, batch, flush)
  ▼
Platform API
  ├─ Discord: editMessage() (throttled)
  ├─ Telegram: editMessageText() (throttled)
  ├─ Slack: chat.update() (throttled)
  ├─ Web UI: WebSocket frame (direct)
  ├─ CLI: stdout.write() (direct)
  └─ WhatsApp: sendMessage() (no edit, final only)
```

---

## 8. 設計考量

### 8.1 為什麼不用 gRPC 或其他二進位協定？

OpenClaw 選擇 JSON-over-WebSocket 而非 gRPC 或 Protobuf，主要考量：
1. **Web 相容性**：瀏覽器原生支援 WebSocket 和 SSE
2. **可觀察性**：JSON 框架易於除錯和記錄
3. **TypeBox schema**：型別安全且可產出 JSON Schema
4. **多客戶端**：CLI、Web、Mobile 都能輕鬆實作

### 8.2 慢消費者的處理策略

`MAX_BUFFERED_BYTES` 機制是一個重要的背壓（backpressure）策略。在串流場景中，如果客戶端處理速度跟不上伺服器的推送速度，緩衝區會持續膨脹。OpenClaw 的策略是「果斷切斷」而非「無限等待」，避免記憶體洩漏。

### 8.3 序列號的用途

事件框架的 `seq` 欄位讓客戶端能偵測事件遺漏。如果客戶端收到 `seq: 5` 後直接收到 `seq: 8`，它就知道中間漏了兩個事件，可以觸發狀態同步。

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/plugin-sdk/channel-streaming.ts:1-110` | 通道串流型別定義 |
| `source-repo/src/channels/draft-stream-controls.ts:1-100` | Draft Stream 控制結構 |
| `source-repo/src/channels/draft-stream-loop.ts:1-110` | 串流迴圈狀態機 |
| `source-repo/src/plugin-sdk/provider-stream-shared.ts:1-120` | Provider 串流抽象 |
| `source-repo/src/gateway/protocol/schema/frames.ts:128-137` | 錯誤結構 schema |
| `source-repo/src/gateway/openresponses-http.ts:793-797` | SSE 錯誤回退 |
| `source-repo/src/gateway/reconnect-gating.ts` | 重連閘門 |
| `source-repo/src/gateway/sessions-history-http.ts:207-217` | SSE 重連與心跳 |
