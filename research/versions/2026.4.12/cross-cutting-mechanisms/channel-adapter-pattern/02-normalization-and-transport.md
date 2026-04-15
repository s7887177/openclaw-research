# Channel 適配器模式（二）：訊息正規化、傳輸層與核准流程

> **摘要**：本章深入 Channel Adapter Pattern 的運行時面向——訊息如何從不同平台進入
> 系統並被正規化（Inbound Normalization）、回覆如何被格式化並發送回平台
> （Outbound Pipeline）、工具核准（Tool Approval）如何因平台而異、以及新增 Channel
> 的標準流程。

---

## 1. Inbound：入站訊息正規化

### 1.1 Envelope Builder 模式

當一則訊息從 Discord、Slack 或 Telegram 到達時，它的格式完全不同。
`createInboundEnvelopeBuilder()` 建立一個正規化器，將平台特定的訊息轉換為統一格式：

```typescript
// source-repo/src/plugin-sdk/inbound-envelope.ts:1-100+
export function createInboundEnvelopeBuilder<TConfig, TEnvelope>(params: {
  cfg: TConfig;
  route: RouteLike;
  sessionStore?: string;
  resolveStorePath: (store, opts) => string;
  readSessionUpdatedAt: (params) => number | undefined;
  resolveEnvelopeFormatOptions: (cfg) => TEnvelope;
  formatAgentEnvelope: (params) => string;
}): (input: { channel, from, body, timestamp? }) => { storePath, body }
```

**設計亮點**：這是一個工廠函式（Factory Function），每個 Channel 用自己的配置和格式化
邏輯建立一個專屬的 Envelope Builder。Builder 返回的函式接受標準化的輸入
（channel、from、body、timestamp），產出統一的結果（storePath、body）。

### 1.2 Webhook Ingress 模式

大多數 Channel 透過 HTTP Webhook 接收訊息。OpenClaw 提供了一套統一的 Webhook 
處理基礎設施：

```typescript
// source-repo/src/plugin-sdk/webhook-ingress.ts:1-45
// 提供以下工具：
// - createWebhookAnomalyTracker()  — 異常流量偵測
// - createFixedWindowRateLimiter() — 固定窗口限流
// - createWebhookInFlightLimiter() — 並行請求限制
// - applyBasicWebhookRequestGuards() — 基本請求驗證
// - readJsonWebhookBodyOrReject()  — JSON Body 解析
```

### 1.3 Webhook 路徑正規化

每個 Channel 的 Webhook 都有一個路徑。路徑正規化確保一致性：

```typescript
// source-repo/src/plugin-sdk/webhook-path.ts:1-34
export function normalizeWebhookPath(raw: string): string {
  const trimmed = raw.trim();
  if (!trimmed) return "/";
  const withSlash = trimmed.startsWith("/") ? trimmed : `/${trimmed}`;
  if (withSlash.length > 1 && withSlash.endsWith("/")) {
    return withSlash.slice(0, -1);
  }
  return withSlash;
}

export function resolveWebhookPath(params: {
  webhookPath?: string;
  webhookUrl?: string;
  defaultPath?: string | null;
}): string | null
```

### 1.4 Webhook Target Registry

Webhook 目標被按路徑分桶註冊：

```typescript
// source-repo/src/plugin-sdk/webhook-targets.ts:10-100+
export type RegisteredWebhookTarget<T> = {
  target: T;
  unregister: () => void;
};

export function registerWebhookTarget<T extends { path: string }>(
  targetsByPath: Map<string, T[]>,
  target: T,
  opts?: RegisterWebhookTargetOptions<T>,
): RegisteredWebhookTarget<T>
```

當一個 HTTP 請求到達時，系統根據路徑查找對應的 Channel 處理器。

---

## 2. Outbound：回覆訊息管道

### 2.1 Reply Pipeline

Reply Pipeline 將 Agent 的回覆組裝成可發送的格式：

```typescript
// source-repo/src/plugin-sdk/channel-reply-pipeline.ts:1-67
export type ChannelReplyPipeline = ReplyPrefixOptions & {
  typingCallbacks?: TypingCallbacks;
  transformReplyPayload?: (payload: ReplyPayload) => ReplyPayload | null;
};

export function createChannelReplyPipeline(params: {
  cfg: OpenClawConfig;
  agentId: string;
  channel?: string;
  accountId?: string;
  typing?: CreateTypingCallbacksParams;
  typingCallbacks?: TypingCallbacks;
  transformReplyPayload?: (payload: ReplyPayload) => ReplyPayload | null;
}): ChannelReplyPipeline
```

**Typing Callbacks**：許多平台支援「正在輸入...」指示器。Pipeline 整合了 Typing 
回調，讓 Agent 思考時顯示輸入狀態。

**Payload Transform**：每個 Channel 可以透過 `transformReplyPayload` 修改最終的回覆內容，
例如將 Markdown 轉換為平台特定的格式。

### 2.2 Outbound Runtime

```typescript
// source-repo/src/plugin-sdk/outbound-runtime.ts:1-5
// 匯出：
// - createRuntimeOutboundDelegates() — 建立平台發送委派
// - resolveOutboundSendDep()        — 解析發送依賴
// - resolveAgentOutboundIdentity()  — 正規化身份
// - sanitizeForPlainText()          — 文字清理
```

### 2.3 Media 處理

不同平台對媒體（圖片、影片、檔案）的支援和限制各不相同：

```typescript
// source-repo/src/plugin-sdk/outbound-media.ts:1-25
export type OutboundMediaLoadOptions = {
  maxBytes?: number;
  mediaAccess?: OutboundMediaAccess;
  mediaLocalRoots?: readonly string[];
  mediaReadFile?: (filePath: string) => Promise<Buffer>;
};

export async function loadOutboundMediaFromUrl(mediaUrl, options): Promise<...>
```

`mediaAccess` 和 `mediaLocalRoots` 控制 Agent 可以存取哪些本地檔案作為媒體發送，
這是一個安全邊界。

### 2.4 Channel Streaming

對於支援串流（Streaming）的平台，回覆可以逐步更新：

```typescript
// source-repo/src/plugin-sdk/channel-streaming.ts:1-120+
export type ChannelStreamingAdapter = {
  blockStreamingCoalesceDefaults?: {
    minChars: number;
    idleMs: number;
  };
};
```

串流有不同的分塊模式（Chunk Mode）：
- **newline** — 按換行符分塊
- **length** — 按字元數分塊

`resolveChannelStreamingChunkMode()` 和 `resolveChannelStreamingBlockEnabled()` 
根據 Channel 的配置選擇適當的策略（`source-repo/src/plugin-sdk/channel-streaming.ts:83-97`）。

---

## 3. Approval：工具核准流程

### 3.1 為什麼核准因平台而異？

當 Agent 決定執行一個可能有風險的操作（例如刪除檔案、發送 Email），它需要使用者核准。
但不同平台呈現核准 UI 的方式完全不同：

- **Discord**：可以發送帶有按鈕的訊息
- **Slack**：可以發送 Interactive Message 或 Block Kit
- **Telegram**：可以發送 Inline Keyboard
- **CLI**：在終端機顯示 prompt

### 3.2 Approval Handler Architecture

OpenClaw 將核准流程分為四個子適配器：

```typescript
// source-repo/src/plugin-sdk/approval-handler-runtime.ts:1-31（匯出概要）
// - createChannelApprovalHandler()                   — 主處理器
// - createChannelApprovalNativeRuntimeAdapter()       — 原生適配器
// - createLazyChannelApprovalNativeRuntimeAdapter()   — 懶載入適配器
```

原生適配器（Native Adapter）分為四個面向：
- **Transport** — 訊息遞送（如何發送核准請求給使用者）
- **Interaction** — 使用者互動（如何接收使用者的核准/拒絕回應）
- **Presentation** — UI 呈現（如何顯示核准請求的內容）
- **Observation** — 狀態觀察（如何追蹤核准請求的狀態）

### 3.3 Approval Capability

每個 Channel 可以宣告其核准能力：

```typescript
// source-repo/src/channels/plugins/types.adapters.ts:605-645（概要）
export type ChannelApprovalCapability = ChannelApprovalAdapter & {
  authorizeActorAction?: (cfg, accountId?, senderId?, action, approvalKind)
    => { authorized, reason? };
  getActionAvailabilityState?: (cfg, accountId?, action, approvalKind?)
    => ChannelActionAvailabilityState;  // "enabled" | "disabled" | "unsupported"
  getExecInitiatingSurfaceState?: (cfg, accountId?, action)
    => ChannelActionAvailabilityState;
  resolveApproveCommandBehavior?: (cfg, accountId?, senderId?, approvalKind)
    => ChannelApproveCommandBehavior | undefined;
};
```

### 3.4 Approval Renderers

核准請求的渲染也是 Channel-specific 的：

```typescript
// source-repo/src/plugin-sdk/approval-renderers.ts
// 負責將核准請求渲染為平台特定的格式
// Discord: Embed + Buttons
// Slack: Block Kit
// Telegram: Inline Keyboard
// CLI: Text prompt
```

---

## 4. Setup Wizard：Channel 配置嚮導

### 4.1 Setup Adapter

每個 Channel 可以定義自己的設定流程：

```typescript
// source-repo/src/plugin-sdk/channel-setup.ts:1-47
export type ChannelSetupAdapter = {
  resolveAccountId?: (cfg, accountId?, input?) => string;
  resolveBindingAccountId?: (cfg, agentId, accountId?) => string | undefined;
  applyAccountName?: (cfg, accountId, name?) => OpenClawConfig;
  applyAccountConfig: (cfg, accountId, input) => OpenClawConfig;
  afterAccountConfigWritten?: (prevCfg, cfg, accountId, input, runtime)
    => Promise<void> | void;
  validateInput?: (cfg, accountId, input) => string | null;
  singleAccountKeysToMove?: readonly string[];
  namedAccountPromotionKeys?: readonly string[];
  resolveSingleAccountPromotionTarget?: (channel) => string | undefined;
};
```

**不可變配置模式**：所有配置修改都透過返回新的 `OpenClawConfig` 實現。
`applyAccountConfig()` 是必填方法，負責將使用者輸入轉換為配置。

### 4.2 Setup Wizard 層級

Channel 的設定嚮導分為幾個層級：

1. **Binary Setup**（`source-repo/src/channels/plugins/setup-wizard-binary.ts`）
   — 需要安裝外部程式（如 WhatsApp 需要 Chromium）

2. **Proxy Setup**（`source-repo/src/channels/plugins/setup-wizard-proxy.ts`）
   — 需要配置代理

3. **Group Access Setup**（`source-repo/src/channels/plugins/setup-group-access.ts`）
   — 群組存取權限設定

---

## 5. Channel Actions：訊息動作

Channel 可以定義訊息級別的動作（如按鈕、卡片）：

```typescript
// source-repo/src/plugin-sdk/channel-actions.ts:1-57
export function createMessageToolButtonsSchema(): TSchema
export function createMessageToolCardSchema(): TSchema
```

按鈕支援三種樣式：`danger`、`success`、`primary`。
卡片（Card）是平台原生的富內容格式。

---

## 6. Channel Targets 與訊息路由

### 6.1 Target 正規化

不同平台的「發送目標」格式各異：

```typescript
// source-repo/src/plugin-sdk/channel-targets.ts:1-46
export function parseTargetPrefix(): ...
export function buildMessagingTarget(): ...
export function normalizeTargetId(): ...
```

### 6.2 Messaging Targets

```typescript
// source-repo/src/plugin-sdk/messaging-targets.ts:1-14
export type MessagingTargetKind = string;
export function buildMessagingTarget(...)
export function normalizeTargetId(...)
export function parseTargetMention(...)
```

目標可以是 Channel（`channel:12345`）、用戶（`user:67890`）、Thread（`thread:abc`）等，
統一為 `kind:id` 格式。

---

## 7. 新增 Channel 的標準流程

根據 `source-repo/extensions/AGENTS.md` 和架構分析，新增 Channel 的流程如下：

### 7.1 建立 Extension 目錄

```
extensions/my-platform/
├── openclaw.plugin.json     # Manifest
├── package.json             # NPM 套件定義
├── index.ts                 # Plugin 入口
└── src/
    └── channel.ts           # Channel 實作
```

### 7.2 撰寫 Manifest

```json
{
  "id": "my-platform",
  "channels": ["my-platform"],
  "channelEnvVars": {
    "my-platform": ["MY_PLATFORM_BOT_TOKEN"]
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

### 7.3 實作 ChannelPlugin

最小可行的 Channel Plugin 需要：
1. `id` — Channel 識別碼
2. `meta` — 名稱和描述
3. `capabilities` — 支援的功能
4. `config` — 帳號配置適配器（至少實作 `listAccountIds` 和 `resolveAccount`）

可選但常見的適配器：
5. `outbound` — 訊息發送
6. `security` — DM 策略
7. `setup` — 配置流程
8. `gateway` — Gateway 生命週期

### 7.4 遵循邊界規則

根據 `source-repo/extensions/AGENTS.md`：

1. **Import 邊界**：Extension 只能 import `openclaw/plugin-sdk/*` 和本地 `api.ts`/`runtime-api.ts`
2. **Core Invariant**：核心必須保持 extension-agnostic，不能硬編碼 Channel
3. **Manifest-Driven**：使用 manifest 元資料而非硬編碼的特殊處理
4. **Doctor Path**：使用 `openclaw doctor --fix` 修復舊配置
5. **Generic Seams**：新的 Plugin 介面必須文件化、向後相容、有版本契約

### 7.5 測試

```bash
pnpm test:extension my-platform     # Plugin 特定測試
pnpm test:contracts:channels         # 共享 Channel 介面測試
pnpm check-src-extension-import-boundary.mjs  # 邊界驗證
```

---

## 8. 十大設計模式總結

| # | 模式 | 說明 |
|---|------|------|
| 1 | **組合優於繼承** | `ChannelPlugin` 由 25+ 可選適配器組合 |
| 2 | **型別安全泛型** | `<ResolvedAccount>` 讓平台特定型別流過適配器 |
| 3 | **懶載入** | `BundledChannelEntryContract` 使用工廠延遲載入 |
| 4 | **不可變配置** | 所有配置修改返回新 `OpenClawConfig` |
| 5 | **能力旗標** | 靜態 `capabilities` + 動態 `ActionAvailabilityState` |
| 6 | **上下文驅動正規化** | `SecurityContext`、`GatewayContext` 攜帶完整上下文 |
| 7 | **雙傳輸模式** | `deliveryMode: "direct" | "gateway" | "hybrid"` |
| 8 | **多模式 Threading** | `"off" | "first" | "all" | "batched"` |
| 9 | **安全即適配器** | DM 策略、稽核發現、允許清單都是可插拔的 |
| 10 | **四面向核准** | Transport / Interaction / Presentation / Observation |

---

## 引用來源

| 來源檔案 | 行號 | 內容 |
|---------|------|------|
| `source-repo/src/plugin-sdk/inbound-envelope.ts` | 1-100+ | Envelope Builder |
| `source-repo/src/plugin-sdk/webhook-ingress.ts` | 1-45 | Webhook 基礎設施 |
| `source-repo/src/plugin-sdk/webhook-path.ts` | 1-34 | 路徑正規化 |
| `source-repo/src/plugin-sdk/webhook-targets.ts` | 10-100+ | Webhook Target Registry |
| `source-repo/src/plugin-sdk/channel-reply-pipeline.ts` | 1-67 | Reply Pipeline |
| `source-repo/src/plugin-sdk/outbound-runtime.ts` | 1-5 | Outbound Runtime |
| `source-repo/src/plugin-sdk/outbound-media.ts` | 1-25 | Media 處理 |
| `source-repo/src/plugin-sdk/channel-streaming.ts` | 1-120+ | 串流適配器 |
| `source-repo/src/plugin-sdk/channel-setup.ts` | 1-47 | Setup Adapter |
| `source-repo/src/plugin-sdk/channel-actions.ts` | 1-57 | 訊息動作 |
| `source-repo/src/plugin-sdk/channel-targets.ts` | 1-46 | Target 正規化 |
| `source-repo/src/plugin-sdk/messaging-targets.ts` | 1-14 | Messaging Target |
| `source-repo/src/plugin-sdk/approval-handler-runtime.ts` | 1-31 | 核准處理器 |
| `source-repo/src/plugin-sdk/approval-renderers.ts` | — | 核准渲染器 |
| `source-repo/src/channels/plugins/setup-wizard-binary.ts` | — | Binary Setup |
| `source-repo/src/channels/plugins/setup-wizard-proxy.ts` | — | Proxy Setup |
| `source-repo/src/channels/plugins/setup-group-access.ts` | — | Group Access Setup |
| `source-repo/extensions/AGENTS.md` | — | 邊界規則與新增流程 |
