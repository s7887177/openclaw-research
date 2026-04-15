# Channel 抽象的設計價值

> **所屬 Topic**：多通道範式（Multi-Channel Paradigm）  
> **層級**：Level 6 — 概念綜合層  
> **前置閱讀**：Level 2 Channels 通道系統、Level 3 Channel 適配器模式

---

## 摘要

本章深入分析 OpenClaw 如何透過 `ChannelPlugin` 介面將 25+ 通訊平台統一為一致的抽象層。重點包括：介面設計中的 30+ 適配器插槽如何覆蓋從認證到串流的完整生命週期、訊息正規化機制如何處理各平台格式差異、以及能力矩陣（Capability Matrix）如何讓每個通道宣告自己的功能邊界。最後用大表格比較主要平台的能力差異。

---

## 1. ChannelPlugin 介面：30+ 適配器的統一合約

### 1.1 介面總覽

OpenClaw 的通道抽象核心是 `ChannelPlugin` 型別，定義於 `src/channels/plugins/types.plugin.ts:53-96`。這不是一個簡單的 interface，而是一個包含 30+ 個可選適配器（Adapter）的組合型別：

```typescript
// source-repo/src/channels/plugins/types.plugin.ts:53-96
export type ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  defaults?: { queue?: { debounceMs?: number } };
  reload?: { configPrefixes: string[]; noopPrefixes?: string[] };
  setupWizard?: ChannelPluginSetupWizard;
  config: ChannelConfigAdapter<ResolvedAccount>;
  configSchema?: ChannelConfigSchema;
  setup?: ChannelSetupAdapter;
  pairing?: ChannelPairingAdapter;
  security?: ChannelSecurityAdapter<ResolvedAccount>;
  groups?: ChannelGroupAdapter;
  mentions?: ChannelMentionAdapter;
  outbound?: ChannelOutboundAdapter;
  status?: ChannelStatusAdapter<ResolvedAccount, Probe, Audit>;
  gatewayMethods?: string[];
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;
  auth?: ChannelAuthAdapter;
  approvalCapability?: ChannelApprovalCapability;
  elevated?: ChannelElevatedAdapter;
  commands?: ChannelCommandAdapter;
  lifecycle?: ChannelLifecycleAdapter;
  secrets?: ChannelSecretsAdapter;
  allowlist?: ChannelAllowlistAdapter;
  doctor?: ChannelDoctorAdapter;
  bindings?: ChannelConfiguredBindingProvider;
  conversationBindings?: ChannelConversationBindingSupport;
  streaming?: ChannelStreamingAdapter;
  threading?: ChannelThreadingAdapter;
  messaging?: ChannelMessagingAdapter;
  agentPrompt?: ChannelAgentPromptAdapter;
  directory?: ChannelDirectoryAdapter;
  resolver?: ChannelResolverAdapter;
  actions?: ChannelMessageActionAdapter;
  heartbeat?: ChannelHeartbeatAdapter;
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
};
```

### 1.2 適配器分類

這 30+ 個適配器可以按功能分為七大類：

| 類別 | 適配器 | 職責 |
|------|--------|------|
| **核心識別** | `id`, `meta`, `capabilities` | 通道身份、元資料、能力宣告 |
| **配置與設定** | `config`, `configSchema`, `setup`, `setupWizard`, `reload` | 配置解析、設定精靈、熱重載 |
| **認證與安全** | `auth`, `security`, `secrets`, `allowlist`, `pairing`, `elevated` | 認證流程、安全稽核、白名單、設備配對 |
| **訊息處理** | `outbound`, `messaging`, `mentions`, `commands`, `actions` | 訊息發送、提及偵測、指令處理 |
| **串流與互動** | `streaming`, `threading`, `status`, `groups`, `bindings` | 串流回應、執行緒、狀態反應、群組管理 |
| **閘道整合** | `gateway`, `gatewayMethods` | HTTP/WebSocket 閘道路由 |
| **診斷與生命週期** | `lifecycle`, `doctor`, `heartbeat`, `directory`, `resolver` | 健康檢查、心跳、名冊解析 |

### 1.3 設計哲學：「能力宣告」而非「強制實作」

這個介面的精妙之處在於**幾乎所有適配器都是可選的**。只有三個欄位是必要的：

1. `id`：通道的唯一識別碼
2. `meta`：元資料（標籤、別名等）
3. `capabilities`：能力宣告

一個最簡的通道 Plugin 只需要提供這三個，加上 `config` 適配器來處理配置。而像 Discord 這樣功能豐富的通道，則可以實作大部分適配器。

這個設計遵循了**介面隔離原則**（Interface Segregation Principle, ISP）——每個通道只需要關心自己能做的事，不需要為不支援的功能提供空實作。

---

## 2. 訊息正規化：彌合格式鴻溝

### 2.1 問題：每個平台「說不同的語言」

不同通訊平台對訊息的表示方式差異巨大：

| 平台 | 格式 | 粗體語法 | 連結語法 | 特殊格式 |
|------|------|---------|---------|---------|
| Discord | Markdown | `**text**` | `[text](url)` | Embed, Code Block |
| Telegram | MarkdownV2 / HTML | `**text**` 或 `<b>text</b>` | `[text](url)` 或 `<a href="url">text</a>` | Inline Keyboard |
| Slack | Block Kit JSON | `*text*` | `<url\|text>` | Block Kit Components |
| WhatsApp | 自訂標記 | `*text*` | 原生連結 | 無富文本 |
| Signal | Markdown | `**text**` | 原生連結 | 有限格式 |
| IRC | 控制碼 | `\x02text\x02` | 原生連結 | mIRC 色碼 |
| Teams | Adaptive Card | HTML 子集 | HTML `<a>` | Adaptive Cards |

### 2.2 入站正規化：將混亂統一

OpenClaw 的入站正規化（Inbound Normalization）在 `src/plugin-sdk/channel-inbound.ts` 中實作，核心流程：

```
平台原生訊息 ──→ 通道 Plugin 解析 ──→ 正規化信封（Envelope）──→ Agent 處理管線
```

**信封建構器**（`src/plugin-sdk/inbound-envelope.ts:28-56`）負責將每個平台的原始訊息包裝成統一格式：

```typescript
type InboundEnvelopeFormatParams<TEnvelope> = {
  channel: string;        // 來源通道 ID
  from: string;           // 發送者識別
  timestamp?: number;     // 訊息時間戳
  previousTimestamp?: number;
  envelope: TEnvelope;    // 平台原生信封
  body: string;           // 正規化後的文字內容
};
```

每個通道 Plugin 負責：
1. 從平台 API 接收原始訊息
2. 提取文字內容並轉換為統一格式
3. 處理提及（Mention）偵測與正規化
4. 包裝成正規化信封交給 Agent

### 2.3 出站適配：從統一到多樣

出站適配（Outbound Adaptation）由 `ChannelOutboundAdapter`（`src/channels/plugins/outbound.types.ts`）處理：

```typescript
export type ChannelOutboundAdapter = {
  deliveryMode: "direct" | "gateway" | "hybrid";
  chunker?: ((text: string, limit: number) => string[]) | null;
  chunkerMode?: "text" | "markdown";
  textChunkLimit?: number;
  sanitizeText?: (params: { text: string; payload: ReplyPayload }) => string;
  normalizePayload?: (params: { payload: ReplyPayload }) => ReplyPayload | null;
  sendPayload?: (ctx: ChannelOutboundPayloadContext) => Promise<OutboundDeliveryResult>;
  sendText?: (ctx: ChannelOutboundContext) => Promise<OutboundDeliveryResult>;
  sendMedia?: (ctx: ChannelOutboundContext) => Promise<OutboundDeliveryResult>;
};
```

關鍵設計點：

| 機制 | 說明 | 範例 |
|------|------|------|
| **deliveryMode** | 訊息如何送達：直連（direct）、經閘道（gateway）、或混合（hybrid） | Discord: direct, Synology Chat: gateway |
| **chunkerMode** | 長訊息切分策略：保持 Markdown 結構（markdown）或純文字切分（text） | Discord/Slack: markdown, WhatsApp: text |
| **textChunkLimit** | 單次訊息最大長度 | IRC: 350, QQ: 5000, BlueBubbles: 4000 |
| **sanitizeText** | 平台特定的文字清理 | 移除 Markdown、轉義特殊字元 |

### 2.4 串流回應的平台適配

當 Agent 正在思考並逐步產生回應時，不同平台對「串流」的處理方式完全不同：

- **Discord**：支援訊息編輯，可以不斷更新同一條訊息 → 真正的串流體驗
- **Telegram**：也支援訊息編輯 → 類似 Discord 的串流
- **Signal**：不支援訊息編輯 → 使用 `blockStreaming` 模式，等完整回應再發送
- **IRC**：純文字協議 → 必須 `blockStreaming`，且受 350 字元限制

OpenClaw 透過兩個機制處理這個差異：

1. **`blockStreaming` 能力宣告**：通道可以宣告自己不支援串流，系統會自動切換為完整訊息模式
2. **Draft Stream Loop**（`src/channels/draft-stream-loop.ts`）：為支援串流的通道提供節流（Throttle）機制，避免 API 過載

```typescript
// source-repo/src/channels/draft-stream-loop.ts:10-104
type DraftStreamLoop = {
  update: (text: string) => void;   // 更新待送文字
  flush: () => Promise<void>;       // 立即送出
  stop: () => void;                 // 停止串流
  resetThrottleWindow: () => void;  // 重設節流窗口
  waitForInFlight: () => Promise<void>; // 等待進行中的請求
};
```

---

## 3. 能力矩陣：「我能做什麼」的自我宣告

### 3.1 ChannelCapabilities 型別

每個通道 Plugin 透過 `capabilities` 欄位宣告自己支援哪些功能（`src/channels/plugins/types.core.ts:253-266`）：

```typescript
export type ChannelCapabilities = {
  chatTypes: Array<ChatType | "thread">;  // 支援的聊天類型
  polls?: boolean;         // 投票功能
  reactions?: boolean;     // 表情反應
  edit?: boolean;          // 訊息編輯
  unsend?: boolean;        // 訊息撤回
  reply?: boolean;         // 引用回覆
  effects?: boolean;       // 訊息特效
  groupManagement?: boolean; // 群組管理
  threads?: boolean;       // 執行緒（子對話）
  media?: boolean;         // 媒體（圖片/檔案）
  nativeCommands?: boolean; // 原生斜線命令
  blockStreaming?: boolean; // 是否阻擋串流
};
```

### 3.2 ChatType 分類

`chatTypes` 是最重要的能力宣告，決定了通道支援哪些對話形態：

| ChatType | 說明 | 典型平台 |
|----------|------|---------|
| `"direct"` | 一對一私訊 | 幾乎所有平台 |
| `"group"` | 多人群組對話 | WhatsApp, Signal, LINE |
| `"channel"` | 公開/私有頻道 | Discord, Slack, Telegram |
| `"thread"` | 執行緒（巢狀對話） | Discord, Slack, Matrix |

### 3.3 完整能力比較矩陣

以下表格基於各通道 Plugin 原始碼中的 `capabilities` 宣告：

| 平台 | chatTypes | 反應 | 執行緒 | 媒體 | 投票 | 原生指令 | 串流 | Markdown |
|------|-----------|------|--------|------|------|---------|------|----------|
| **Discord** | direct, channel, thread | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ 即時串流 | ✅ |
| **Telegram** | direct, group, channel, thread | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ 阻塞 | ✅ |
| **Slack** | direct, channel, thread | ✅ | ✅ | ✅ | — | ✅ | ✅ 即時串流 | ✅ |
| **WhatsApp** | direct, group | ✅ | — | ✅ | ✅ | — | ✅ 即時串流 | — |
| **Signal** | direct, group | ✅ | — | ✅ | — | — | ❌ 阻塞 | ✅ |
| **MS Teams** | direct, channel, thread | — | ✅ | ✅ | ✅ | — | ❌ 阻塞 | — |
| **Google Chat** | direct, group, thread | ✅ | ✅ | ✅ | — | — | ❌ 阻塞 | ✅ |
| **Feishu** | direct, channel | ✅ | ✅ | ✅ | — | — | ✅ 即時串流 | — |
| **Matrix** | direct, group, thread | ✅ | ✅ | ✅ | ✅ | — | ✅ 即時串流 | — |
| **Mattermost** | direct, channel, group, thread | ✅ | ✅ | ✅ | — | ✅ | ❌ 阻塞 | — |
| **IRC** | direct, group | — | — | ✅ | — | — | ❌ 阻塞 | ✅ |
| **LINE** | direct, group | — | — | ✅ | — | — | ❌ 阻塞 | — |
| **Nostr** | direct | — | — | — | — | — | ✅ 即時串流 | — |
| **Twitch** | group | — | — | — | — | — | ✅ 即時串流 | — |
| **BlueBubbles** | direct, group | ✅ | — | ✅ | — | — | ✅ 即時串流 | — |
| **iMessage** | direct, group | — | — | ✅ | — | — | ✅ 即時串流 | — |
| **QQ Bot** | direct, group | — | — | ✅ | — | — | ❌ 阻塞 | — |
| **Zalo** | direct, group | — | — | ✅ | — | — | ❌ 阻塞 | — |
| **Nextcloud Talk** | direct, group | ✅ | — | ✅ | — | — | ❌ 阻塞 | — |
| **Synology Chat** | direct | — | — | ✅ | — | — | ✅ 即時串流 | — |

> **圖例**：✅ = 支援，— = 不支援/未宣告，「阻塞」= `blockStreaming: true`（等完整回應再發送）

### 3.4 能力矩陣的架構意義

能力矩陣不僅是「功能清單」，它在 OpenClaw 的架構中扮演決策樞紐：

1. **狀態反應控制器**（`src/channels/status-reactions.ts`）會檢查 `reactions` 能力，決定是否顯示 emoji 狀態（👀🤔🔥👍）
2. **執行緒綁定策略**（`src/channels/thread-bindings-policy.ts`）會檢查 `threads` 能力，決定是否可以自動建立子對話
3. **串流迴圈**會檢查 `blockStreaming`，決定是即時串流還是等待完整回應
4. **指令閘門**（`src/channels/command-gating.ts`）會檢查 `nativeCommands`，決定是否啟用斜線命令

這種「基於宣告的行為適配」讓 OpenClaw 的核心邏輯可以保持通用，而不需要到處寫 `if (channel === 'discord') { ... }` 的硬編碼分支。

---

## 4. 互動機制的統一

### 4.1 打字指示器（Typing Indicator）

當 Agent 正在處理訊息時，使用者應該看到「對方正在輸入…」的提示。OpenClaw 透過 `createTypingCallbacks()`（`src/channels/typing.ts:23-99`）統一了這個機制：

```typescript
type CreateTypingCallbacksParams = {
  start: () => Promise<void>;          // 開始顯示「輸入中」
  stop?: () => Promise<void>;          // 停止顯示
  keepaliveIntervalMs?: number;        // 心跳間隔（預設 3 秒）
  maxConsecutiveFailures?: number;     // 最大連續失敗次數（預設 2）
  maxDurationMs?: number;              // 安全 TTL（預設 60 秒）
};
```

每個通道 Plugin 只需提供 `start` 和 `stop` 函式，框架自動處理心跳、失敗重試、超時保護。

### 4.2 狀態反應（Status Reactions）

OpenClaw 獨特的狀態反應系統（`src/channels/status-reactions.ts:129-415`）透過 emoji 向使用者即時顯示 Agent 的處理狀態：

| 狀態 | 預設 Emoji | 觸發時機 |
|------|-----------|---------|
| 排隊中（Queued） | 👀 | 收到訊息，進入處理佇列 |
| 思考中（Thinking） | 🤔 | LLM 正在生成回應 |
| 使用工具（Tool） | 🔥 | 呼叫工具執行中 |
| 寫程式（Coding） | 👨‍💻 | 使用程式碼相關工具 |
| 搜尋網頁（Web） | ⚡ | 使用網頁搜尋工具 |
| 壓縮中（Compacting） | ✍ | 壓縮對話歷史 |
| 完成（Done） | 👍 | 回應完成 |
| 錯誤（Error） | 😱 | 處理過程出錯 |
| 軟停滯（Stall Soft） | 🥱 | 超過 10 秒無進度 |
| 硬停滯（Stall Hard） | 😨 | 超過 30 秒無進度 |

這個系統對支援 `reactions` 的平台（Discord、Telegram、WhatsApp 等）自動啟用，讓使用者即使在長時間等待中也能知道 Agent 在「做什麼」。

### 4.3 確認反應（Ack Reaction）

`shouldAckReaction()`（`src/channels/ack-reactions.ts:16-43`）根據六種策略決定是否在收到訊息時給予反應：

| Scope | 行為 |
|-------|------|
| `"all"` | 所有訊息都確認 |
| `"direct"` | 只確認私訊 |
| `"group-all"` | 確認群組中的所有訊息 |
| `"group-mentions"` | 只確認群組中被提及的訊息 |
| `"off"` / `"none"` | 不確認 |

---

## 5. 文字切分限制比較

不同平台對單條訊息長度有不同限制，OpenClaw 需要智慧地切分長回應：

| 平台 | textChunkLimit | chunkerMode | 說明 |
|------|---------------|-------------|------|
| Discord | ~2000 | markdown | 保持 Markdown 結構完整 |
| Telegram | ~4096 | markdown | Telegram 的原生限制 |
| Slack | ~4000 | markdown | Block Kit 限制 |
| IRC | 350 | markdown | 受 IRC 協議限制，極短 |
| WhatsApp | ~4000 | text | 純文字，不保留格式 |
| QQ Bot | 5000 | markdown | 相對寬鬆 |
| Feishu | 4000 | markdown | 飛書限制 |
| Nextcloud Talk | 4000 | markdown | Nextcloud 限制 |
| BlueBubbles | 4000 | text | iMessage 限制 |
| Nostr | 4000 | text | 協議限制 |
| Synology Chat | 2000 | text | 較嚴格的限制 |

---

## 引用來源

| 來源 | 路徑 / 說明 |
|------|-------------|
| ChannelPlugin 型別定義 | `source-repo/src/channels/plugins/types.plugin.ts:53-96` |
| ChannelCapabilities 型別 | `source-repo/src/channels/plugins/types.core.ts:253-266` |
| ChannelOutboundAdapter 型別 | `source-repo/src/channels/plugins/outbound.types.ts:1-112` |
| Draft Stream Loop | `source-repo/src/channels/draft-stream-loop.ts:10-104` |
| 打字指示器 | `source-repo/src/channels/typing.ts:23-99` |
| 狀態反應控制器 | `source-repo/src/channels/status-reactions.ts:129-415` |
| 確認反應 | `source-repo/src/channels/ack-reactions.ts:16-43` |
| Discord capabilities | `source-repo/extensions/discord/src/shared.ts:91-97` |
| Telegram capabilities | `source-repo/extensions/telegram/src/shared.ts:136-143` |
| Slack capabilities | `source-repo/extensions/slack/src/shared.ts:187-193` |
| WhatsApp capabilities | `source-repo/extensions/whatsapp/src/shared.ts:132-137` |
| Signal capabilities | `source-repo/extensions/signal/src/shared.ts:85-89` |
| MS Teams capabilities | `source-repo/extensions/msteams/src/channel.ts:413-418` |
| Google Chat capabilities | `source-repo/extensions/googlechat/src/channel.ts:61` |
| Feishu capabilities | `source-repo/extensions/feishu/src/channel.ts` |
| Matrix capabilities | `source-repo/extensions/matrix/src/channel.ts` |
| Mattermost capabilities | `source-repo/extensions/mattermost/src/channel.ts` |
| IRC capabilities | `source-repo/extensions/irc/src/channel.ts:60` |
| LINE capabilities | `source-repo/extensions/line/src/channel-shared.ts` |
| Nostr capabilities | `source-repo/extensions/nostr/src/channel.ts` |
| Twitch capabilities | `source-repo/extensions/twitch/src/plugin.ts` |
| BlueBubbles capabilities | `source-repo/extensions/bluebubbles/src/channel-shared.ts` |
| iMessage capabilities | `source-repo/extensions/imessage/src/shared.ts` |
| QQ Bot capabilities | `source-repo/extensions/qqbot/src/channel.ts` |
| Zalo capabilities | `source-repo/extensions/zalo/src/channel.ts` |
| Nextcloud Talk capabilities | `source-repo/extensions/nextcloud-talk/src/channel.ts` |
| Synology Chat capabilities | `source-repo/extensions/synology-chat/src/channel.ts` |
| 入站信封建構器 | `source-repo/src/plugin-sdk/inbound-envelope.ts:28-56` |
| 入站正規化 | `source-repo/src/plugin-sdk/channel-inbound.ts:1-54` |
