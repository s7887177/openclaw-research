# Gateway 協定框架、WebSocket 訊息格式與 SSE 串流

## 本章摘要

OpenClaw Gateway 是整個系統的通訊中樞，它透過 **WebSocket** 與客戶端保持持久連線，並透過 **HTTP SSE**（Server-Sent Events）提供串流 API。Gateway 定義了一套基於 JSON 的框架協定（frame protocol），所有的請求、回應與事件都遵循統一的結構。本章從協定的框架定義出發，深入分析 WebSocket 訊息流、SSE 串流機制、以及 Gateway 的廣播系統。

---

## 1. Gateway 框架協定

### 1.1 三種框架型別

Gateway 的通訊建立在三種核心框架（frame）型別之上，定義在 `src/gateway/protocol/schema/frames.ts`：

```typescript
// source-repo/src/gateway/protocol/schema/frames.ts:139-147
export const RequestFrameSchema = Type.Object({
  type: Type.Literal("req"),
  id: NonEmptyString,
  method: NonEmptyString,
  params: Type.Optional(Type.Unknown()),
}, { additionalProperties: false });
```

```typescript
// source-repo/src/gateway/protocol/schema/frames.ts:149-158
export const ResponseFrameSchema = Type.Object({
  type: Type.Literal("res"),
  id: NonEmptyString,
  ok: Type.Boolean(),
  payload: Type.Optional(Type.Unknown()),
  error: Type.Optional(ErrorShapeSchema),
}, { additionalProperties: false });
```

```typescript
// source-repo/src/gateway/protocol/schema/frames.ts:160-169
export const EventFrameSchema = Type.Object({
  type: Type.Literal("event"),
  event: NonEmptyString,
  payload: Type.Optional(Type.Unknown()),
  seq: Type.Optional(Type.Integer({ minimum: 0 })),
  stateVersion: Type.Optional(StateVersionSchema),
}, { additionalProperties: false });
```

三者透過**判別聯合型別**（discriminated union）統一：

```typescript
// source-repo/src/gateway/protocol/schema/frames.ts:174-177
export const GatewayFrameSchema = Type.Union(
  [RequestFrameSchema, ResponseFrameSchema, EventFrameSchema],
  { discriminator: "type" },
);
```

使用 `@sinclair/typebox` 定義 schema（`source-repo/src/gateway/protocol/schema/frames.ts:1`），這讓 schema 能同時用於 TypeScript 型別推斷和 JSON Schema 產出。

### 1.2 框架欄位解析

| 欄位 | 框架型別 | 說明 |
|------|----------|------|
| `type` | 全部 | `"req"` / `"res"` / `"event"` 判別碼 |
| `id` | req, res | 請求/回應的配對 ID |
| `method` | req | 呼叫的方法名稱 |
| `params` | req | 方法參數（任意型別） |
| `ok` | res | 成功或失敗 |
| `payload` | res, event | 回傳資料 |
| `error` | res | 錯誤資訊 |
| `event` | event | 事件名稱 |
| `seq` | event | 序列號（用於事件排序與遺漏偵測） |
| `stateVersion` | event | 狀態版本號（presence, health） |

### 1.3 錯誤結構

```typescript
// source-repo/src/gateway/protocol/schema/frames.ts:128-137
// ErrorShapeSchema 定義
{
  code: NonEmptyString,        // 錯誤碼
  message: NonEmptyString,     // 人類可讀訊息
  details: Type.Optional(...), // 額外細節
  retryable: Type.Optional(Type.Boolean()), // 是否可重試
  retryAfterMs: Type.Optional(Type.Integer({ minimum: 0 })), // 重試等待時間
}
```

`retryable` 和 `retryAfterMs` 欄位讓客戶端能智慧地決定是否重試以及等待多久。

### 1.4 連線參數

連線建立時，客戶端發送 `ConnectParams`（`source-repo/src/gateway/protocol/schema/frames.ts:20-100`）：

```typescript
// source-repo/src/gateway/protocol/schema/frames.ts:20-37
export const ConnectParamsSchema = Type.Object({
  minProtocol: Type.Integer({ minimum: 1 }),
  maxProtocol: Type.Integer({ minimum: 1 }),
  client: Type.Object({
    id: GatewayClientIdSchema,
    displayName: Type.Optional(NonEmptyString),
    version: NonEmptyString,
    platform: NonEmptyString,
    deviceFamily: Type.Optional(NonEmptyString),
    modelIdentifier: Type.Optional(NonEmptyString),
    mode: GatewayClientModeSchema,
    instanceId: Type.Optional(NonEmptyString),
  }),
  caps: Type.Optional(Type.Array(NonEmptyString)),    // 客戶端能力
  commands: Type.Optional(Type.Array(NonEmptyString)), // 支援的指令
  permissions: Type.Optional(Type.Record(...)),        // 權限設定
  role: Type.Optional(NonEmptyString),                 // 角色
  scopes: Type.Optional(Type.Array(NonEmptyString)),   // 存取範圍
  // ... 認證和裝置資訊
});
```

---

## 2. WebSocket 連線管理

### 2.1 客戶端追蹤

每個 WebSocket 連線以 `GatewayWsClient` 結構追蹤：

```typescript
// source-repo/src/gateway/server/ws-types.ts:4-15
type GatewayWsClient = {
  socket: WebSocket;
  connect: ConnectParams;
  connId: string;                      // UUID 連線 ID
  usesSharedGatewayAuth: boolean;
  sharedGatewaySessionGeneration?: string;
  presenceKey?: string;
  clientIp?: string;
  canvasHostUrl?: string;
  canvasCapability?: string;
  canvasCapabilityExpiresAtMs?: number;
};
```

### 2.2 連線生命週期

WebSocket 連線的生命週期管理位於 `src/gateway/server/ws-connection.ts`：

**挑戰階段**（Challenge Phase）：
```
Server → Client: connect.challenge { nonce }
Client → Server: connect.response { ... signed nonce ... }
```
（`source-repo/src/gateway/server/ws-connection.ts:244-249`）

**握手超時**（`source-repo/src/gateway/handshake-timeouts.ts:1-47`）：

```typescript
// source-repo/src/gateway/handshake-timeouts.ts:1-10
export const DEFAULT_PREAUTH_HANDSHAKE_TIMEOUT_MS = 10_000;

export function getPreauthHandshakeTimeoutMsFromEnv(
  env: NodeJS.ProcessEnv = process.env
): number {
  const configuredTimeout =
    env.OPENCLAW_HANDSHAKE_TIMEOUT_MS ||
    (env.VITEST && env.OPENCLAW_TEST_HANDSHAKE_TIMEOUT_MS);
  // 回傳 DEFAULT 或解析後的值
}
```

挑戰超時（Connect Challenge Timeout，`source-repo/src/gateway/handshake-timeouts.ts:12-34`）：
- 最小值：250ms
- 最大值：10000ms
- 可透過 `OPENCLAW_CONNECT_CHALLENGE_TIMEOUT_MS` 環境變數設定
- 超出範圍的值會被鉗制到安全範圍

**斷線追蹤**（`source-repo/src/gateway/server/ws-connection.ts:203-342`）：

系統詳細記錄斷線上下文：
```typescript
let closeCause: string | undefined;
let closeMeta: Record<string, unknown> = {};
let lastFrameType: string | undefined;
let lastFrameMethod: string | undefined;
let lastFrameId: string | undefined;
```

這些資訊在 `close` 事件中被記錄，包含時序和狀態資訊，用於診斷連線問題。

### 2.3 訊息處理

入站訊息的處理入口位於 `src/gateway/server/ws-connection/message-handler.ts`：

```typescript
// source-repo/src/gateway/server/ws-connection/message-handler.ts:159-220
// 所有入站 WebSocket 訊息的處理入口
// 包含：
// - 認證速率限制
// - Origin/Host 驗證
// - 裝置配對狀態協調
// - Presence 追蹤
```

---

## 3. SSE（Server-Sent Events）串流

### 3.1 SSE 標頭設定

```typescript
// source-repo/src/gateway/http-common.ts:102-108
export function setSseHeaders(res: ServerResponse) {
  res.statusCode = 200;
  res.setHeader("Content-Type", "text/event-stream; charset=utf-8");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");
  res.flushHeaders?.();
}

export function writeDone(res: ServerResponse) {
  res.write("data: [DONE]\n\n");
}
```

### 3.2 OpenAI 相容串流

OpenClaw 提供 OpenAI API 相容的串流端點（`source-repo/src/gateway/openai-http.ts`）：

```typescript
// SSE 寫入格式
function writeSse(res: ServerResponse, data: unknown) {
  res.write(`data: ${JSON.stringify(data)}\n\n`);
}
```

串流事件格式遵循 OpenAI 的 `chat.completion.chunk`：
```json
{
  "id": "run-123",
  "object": "chat.completion.chunk",
  "choices": [{"delta": {"content": "text"}}]
}
```

串流流程（`source-repo/src/gateway/openai-http.ts:620-700`）：
1. 設定 SSE 標頭（`setSseHeaders(res)`）
2. 訂閱 agent 事件（`onAgentEvent()`）
3. 接收文字差量（text delta）→ 透過 `writeAssistantContentChunk()` 送出
4. 工具串流：被處理但會過濾
5. 偵測完成 → 寫入最終 usage chunk → `writeDone(res)` + `res.end()`

### 3.3 Open Responses 串流

OpenClaw 也支援 Open Responses 格式的串流（`source-repo/src/gateway/openresponses-http.ts`）：

```typescript
// 具名 SSE 事件格式
function writeSseEvent(res: ServerResponse, event: StreamingEvent) {
  res.write(`event: ${event.type}\n`);
  res.write(`data: ${JSON.stringify(event)}\n\n`);
}
```

串流事件序列（`source-repo/src/gateway/openresponses-http.ts:790-880`）：
- `response.output_text.delta` — 文字差量（含 index 和 content_index）
- `response.output_text.done` — 最終文字確認
- `response.content_part.done` — 內容組裝標記
- `response.output_item.done` — 項目完成（含完整內容）
- `response.completed` — 最終回應（含 usage 指標）

錯誤處理（`source-repo/src/gateway/openresponses-http.ts:793-797`）：非串流模式的錯誤回退。

### 3.4 Session 歷史串流

Session 歷史也可以透過 SSE 串流推送（`source-repo/src/gateway/sessions-history-http.ts:83-250`）：

```typescript
// source-repo/src/gateway/sessions-history-http.ts:83-86
function sseWrite(res: ServerResponse, event: string, payload: unknown): void {
  res.write(`event: ${event}\n`);
  res.write(`data: ${JSON.stringify(payload)}\n\n`);
}
```

SSE 事件型別：
- `history` — 初始快照（session 的所有訊息）
- `message` — 增量訊息更新

**重連支援**（`source-repo/src/gateway/sessions-history-http.ts:207`）：
```typescript
res.write("retry: 1000\n\n"); // SSE 客戶端在 1000ms 後重試
```

**心跳保活**（`source-repo/src/gateway/sessions-history-http.ts:213-217`）：
每 15 秒發送 keepalive，防止代理伺服器超時斷開。

---

## 4. Gateway 廣播系統

### 4.1 廣播型別

```typescript
// source-repo/src/gateway/server-broadcast-types.ts:1-23
export type GatewayBroadcastStateVersion = {
  presence?: number;
  health?: number;
};

export type GatewayBroadcastOpts = {
  dropIfSlow?: boolean;
  stateVersion?: GatewayBroadcastStateVersion;
};

export type GatewayBroadcastFn = (
  event: string,
  payload: unknown,
  opts?: GatewayBroadcastOpts,
) => void;

export type GatewayBroadcastToConnIdsFn = (
  event: string,
  payload: unknown,
  connIds: ReadonlySet<string>,
  opts?: GatewayBroadcastOpts,
) => void;
```

### 4.2 廣播實作

```typescript
// source-repo/src/gateway/server-broadcast.ts:58-132
function createGatewayBroadcaster(params: { clients: Set<GatewayWsClient> }) {
  let seq = 0;

  const broadcastInternal = (
    event: string,
    payload: unknown,
    opts?: GatewayBroadcastOpts,
    targetConnIds?: ReadonlySet<string>,
  ) => {
    const eventSeq = isTargeted ? undefined : ++seq;
    const frame = JSON.stringify({
      type: "event",
      event,
      payload,
      seq: eventSeq,
      stateVersion: opts?.stateVersion,
    });
    // ... 發送給所有/指定客戶端
  };
}
```

### 4.3 廣播的關鍵特性

**範圍守衛**（Scope Guards，`source-repo/src/gateway/server-broadcast.ts:39-56`）：

每個事件都有存取範圍要求，廣播前會檢查客戶端是否擁有對應的 scope：
- `READ_SCOPE` — 讀取事件
- `WRITE_SCOPE` — 寫入事件
- `APPROVALS_SCOPE` — 審批事件
- `PAIRING_SCOPE` — 配對事件

**慢消費者偵測**（`source-repo/src/gateway/server-broadcast.ts:101-112`）：

當客戶端的緩衝區超過 `MAX_BUFFERED_BYTES` 閾值時：
- 預設行為：關閉該客戶端的連線
- 設定 `dropIfSlow: true` 時：跳過發送，不關閉連線

這個機制防止單一慢客戶端拖垮整個 Gateway。

**定向廣播**：可透過 `targetConnIds` 參數僅向特定連線發送，而非廣播給所有客戶端。

**狀態版本號**：事件可附帶 `presence` 和 `health` 版本號，客戶端據此判斷是否需要更新狀態。

### 4.4 Tick 事件

Gateway 定期發送 tick 事件：

```typescript
// source-repo/src/gateway/protocol/schema/frames.ts:5-10
export const TickEventSchema = Type.Object({
  ts: Type.Integer({ minimum: 0 }),
}, { additionalProperties: false });
```

Tick 事件包含時間戳，用於：
- 客戶端心跳確認
- 時鐘同步
- 連線活性檢測

### 4.5 關機事件

```typescript
// source-repo/src/gateway/protocol/schema/frames.ts:12-18
export const ShutdownEventSchema = Type.Object({
  reason: NonEmptyString,
  restartExpectedMs: Type.Optional(Type.Integer({ minimum: 0 })),
}, { additionalProperties: false });
```

Gateway 關機時會通知所有客戶端，並提供預期的重啟時間，讓客戶端能優雅地重連。

---

## 5. WebSocket 日誌與觀測

### 5.1 日誌系統

```typescript
// source-repo/src/gateway/ws-log.ts:260-443
export function logWs(
  direction: "in" | "out",
  kind: string,
  meta?: Record<string, unknown>
) {
  // 追蹤請求-回應配對與延遲時間
  // 支援多種日誌風格
}
```

### 5.2 日誌模式

```typescript
// source-repo/src/gateway/ws-logging.ts:1-14
export type GatewayWsLogStyle = "auto" | "full" | "compact";
export const DEFAULT_WS_SLOW_MS = 50;
```

| 模式 | 說明 |
|------|------|
| `"full"` | 完整的請求/回應生命週期含 metadata |
| `"compact"` | 最小化輸出，僅追蹤連線 ID |
| `"auto"` | 智慧模式——僅記錄錯誤和慢回應（>50ms） |

追蹤的 metadata 包括：
- 連線 ID、訊息 ID、方法名稱、OK/error 狀態
- 往返延遲（毫秒）
- 客戶端 IP、主機、Origin、User Agent
- 握手狀態、關閉原因、最後一個 frame 資訊

---

## 6. 協定 Schema 組織

Gateway 協定的 schema 檔案位於 `src/gateway/protocol/schema/` 目錄（`source-repo/src/gateway/protocol/schema/`）：

| 檔案 | 內容 |
|------|------|
| `frames.ts` | 核心框架型別（req/res/event） |
| `primitives.ts` | 基礎型別（ID、模式等） |
| `sessions.ts` | Session 相關 schema |
| `agents-models-skills.ts` | Agent、模型、技能 schema |
| `channels.ts` | 通道 schema |
| `commands.ts` | 指令 schema |
| `config.ts` | 設定 schema |
| `cron.ts` | 排程 schema |
| `devices.ts` | 裝置 schema |
| `error-codes.ts` | 錯誤碼定義 |
| `exec-approvals.ts` | 執行審批 schema |
| `logs-chat.ts` | 聊天日誌 schema |
| `nodes.ts` | 節點 schema |
| `plugin-approvals.ts` | 插件審批 schema |
| `push.ts` | 推播 schema |
| `secrets.ts` | 密鑰 schema |
| `snapshot.ts` | 快照與狀態版本 schema |
| `wizard.ts` | 設定精靈 schema |

所有 schema 都使用 `@sinclair/typebox`（TypeBox）定義，這是一個將 JSON Schema 和 TypeScript 型別統一的函式庫。

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/gateway/protocol/schema/frames.ts:1-177` | Gateway 框架 schema 完整定義 |
| `source-repo/src/gateway/protocol/schema/frames.ts:5-18` | Tick 和 Shutdown 事件 schema |
| `source-repo/src/gateway/protocol/schema/frames.ts:20-100` | ConnectParams schema |
| `source-repo/src/gateway/protocol/schema/frames.ts:128-137` | ErrorShape schema |
| `source-repo/src/gateway/server/ws-types.ts:4-15` | GatewayWsClient 型別 |
| `source-repo/src/gateway/server/ws-connection.ts:140-405` | WebSocket 連線生命週期 |
| `source-repo/src/gateway/server/ws-connection.ts:203-342` | 斷線追蹤 |
| `source-repo/src/gateway/server/ws-connection/message-handler.ts:159-220` | 訊息處理入口 |
| `source-repo/src/gateway/handshake-timeouts.ts:1-47` | 握手超時設定 |
| `source-repo/src/gateway/http-common.ts:98-108` | SSE 標頭與 writeDone |
| `source-repo/src/gateway/openai-http.ts:620-700` | OpenAI 相容串流 |
| `source-repo/src/gateway/openresponses-http.ts:790-880` | Open Responses 串流 |
| `source-repo/src/gateway/sessions-history-http.ts:83-250` | Session 歷史 SSE 串流 |
| `source-repo/src/gateway/server-broadcast-types.ts:1-23` | 廣播型別定義 |
| `source-repo/src/gateway/server-broadcast.ts:39-132` | 廣播實作 |
| `source-repo/src/gateway/ws-log.ts:260-443` | WebSocket 日誌 |
| `source-repo/src/gateway/ws-logging.ts:1-14` | 日誌模式設定 |
