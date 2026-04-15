# Channel 適配器模式（一）：ChannelPlugin 介面與適配器組合

> **摘要**：本章深入解析 OpenClaw 如何透過組合式適配器模式（Composition-based Adapter Pattern）
> 將 25+ 通訊平台統一為相同介面。核心是 `ChannelPlugin` 型別——一個由 25+ 個可選
> 適配器組合而成的巨型介面。每個適配器負責一個特定的橫切關注點（Cross-cutting Concern），
> 從帳號管理到訊息發送，從安全策略到核准流程。

---

## 1. 問題背景：為什麼需要 Adapter Pattern？

OpenClaw 支援的通訊平台包括但不限於：

- **即時通訊**：Discord、Slack、Telegram、WhatsApp、Line、Matrix、Signal、IRC
- **企業協作**：Microsoft Teams、Google Chat、Feishu（飛書）、Mattermost
- **社群平台**：Twitch、Nostr、Apple iMessage
- **其他**：Email（Gmail）、SMS、QQ Bot、Zalo 等

每個平台都有完全不同的：
- **認證方式**：Bot Token、OAuth、QR Code、SMS 驗證
- **訊息格式**：Markdown、HTML、富文字、純文字
- **能力集合**：Thread 支援、Reaction 支援、檔案上傳、語音通話
- **傳輸協定**：WebSocket、HTTP Webhook、Long Polling

如果為每個平台寫一套獨立的處理邏輯，核心 Agent 程式碼會變成一團無法維護的條件分支。
OpenClaw 的解決方案是 **Channel Adapter Pattern**。

---

## 2. ChannelPlugin：核心介面定義

### 2.1 完整型別定義

`ChannelPlugin` 是所有 Channel 的統一介面，定義在 `src/channels/plugins/types.plugin.ts`：

```typescript
// source-repo/src/channels/plugins/types.plugin.ts:53-96
export type ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown> = {
  id: ChannelId;                           // Channel 唯一識別碼
  meta: ChannelMeta;                       // 人類可讀元資料（名稱、圖示等）
  capabilities: ChannelCapabilities;       // 靜態能力宣告

  // 預設值
  defaults?: { queue?: { debounceMs?: number } };
  reload?: { configPrefixes: string[]; noopPrefixes?: string[] };

  // 設定嚮導
  setupWizard?: ChannelPluginSetupWizard;

  // === 核心適配器 ===
  config: ChannelConfigAdapter<ResolvedAccount>;      // 帳號配置（必填）
  configSchema?: ChannelConfigSchema;                 // 配置 Schema
  setup?: ChannelSetupAdapter;                        // 設定流程
  pairing?: ChannelPairingAdapter;                    // 裝置配對
  security?: ChannelSecurityAdapter<ResolvedAccount>; // 安全策略
  groups?: ChannelGroupAdapter;                       // 群組管理
  mentions?: ChannelMentionAdapter;                   // @提及

  // === 訊息適配器 ===
  outbound?: ChannelOutboundAdapter;                  // 訊息發送
  streaming?: ChannelStreamingAdapter;                // 串流回應
  threading?: ChannelThreadingAdapter;                // 對話串
  messaging?: ChannelMessagingAdapter;                // 訊息路由

  // === 生命週期適配器 ===
  status?: ChannelStatusAdapter<ResolvedAccount, Probe, Audit>; // 狀態探測
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;   // Gateway 生命週期
  lifecycle?: ChannelLifecycleAdapter;                // 配置變更處理

  // === 認證與安全適配器 ===
  auth?: ChannelAuthAdapter;                          // 登入/登出
  approvalCapability?: ChannelApprovalCapability;     // 工具核准
  elevated?: ChannelElevatedAdapter;                  // 提權操作
  secrets?: ChannelSecretsAdapter;                    // 密鑰管理
  allowlist?: ChannelAllowlistAdapter;                // 允許清單

  // === 進階適配器 ===
  commands?: ChannelCommandAdapter;                   // 斜線命令
  doctor?: ChannelDoctorAdapter;                      // 診斷修復
  bindings?: ChannelConfiguredBindingProvider;         // 配置綁定
  conversationBindings?: ChannelConversationBindingSupport; // 對話綁定
  agentPrompt?: ChannelAgentPromptAdapter;            // Agent Prompt 修改
  directory?: ChannelDirectoryAdapter;                // 使用者/群組目錄
  resolver?: ChannelResolverAdapter;                  // 目標解析
  actions?: ChannelMessageActionAdapter;              // 訊息動作
  heartbeat?: ChannelHeartbeatAdapter;                // 心跳檢測
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[]; // Channel 專屬工具
};
```

### 2.2 關鍵設計模式：組合優於繼承

這個介面最重要的設計決策是 **組合（Composition）而非繼承（Inheritance）**。
Channel Plugin 不是繼承自某個 base class，而是由 25+ 個可選的適配器組合而成。

只有三個欄位是必填的：
- `id` — Channel 識別碼
- `meta` — 元資料
- `capabilities` — 能力宣告
- `config` — 帳號配置適配器

其他所有適配器都是可選的。不支援 Thread 的平台不需要實作 `threading`；
不支援檔案發送的平台不需要實作完整的 `outbound`。

### 2.3 泛型參數的用意

```typescript
ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown>
```

- **`ResolvedAccount`**：平台特定的帳號型別。Discord 的帳號有 `guildId`、`botToken`；
  Slack 的帳號有 `botToken`、`userToken`、`appId`。泛型讓各平台定義自己的帳號結構。
- **`Probe`**：狀態探測的結果型別。
- **`Audit`**：安全稽核的結果型別。

---

## 3. 核心適配器詳解

### 3.1 Configuration Adapter（配置適配器）

這是唯一必填的適配器，負責將平台特定的帳號模型正規化：

```typescript
// source-repo/src/channels/plugins/types.adapters.ts:76-150（概要）
export type ChannelConfigAdapter<ResolvedAccount> = {
  listAccountIds: (cfg: OpenClawConfig) => string[];
  resolveAccount: (cfg: OpenClawConfig, accountId?: string | null) => ResolvedAccount;
  inspectAccount?: (cfg: OpenClawConfig, accountId?: string | null) => unknown;
  defaultAccountId?: (cfg: OpenClawConfig) => string;
  setAccountEnabled?: (params) => OpenClawConfig;
  deleteAccount?: (params) => OpenClawConfig;
  isEnabled?: (account: ResolvedAccount, cfg) => boolean;
  disabledReason?: (account: ResolvedAccount, cfg) => string;
  isConfigured?: (account: ResolvedAccount, cfg) => boolean | Promise<boolean>;
  unconfiguredReason?: (account: ResolvedAccount, cfg) => string;
  describeAccount?: (account: ResolvedAccount, cfg) => ChannelAccountSnapshot;
  resolveAllowFrom?: (cfg, accountId?) => Array<string | number>;
  formatAllowFrom?: (cfg, accountId, allowFrom) => string[];
  hasConfiguredState?: (cfg, env?) => boolean;
  hasPersistedAuthState?: (cfg, env?) => boolean;
  resolveDefaultTo?: (cfg, accountId?) => string | undefined;
};
```

**設計亮點**：所有方法都接受 `OpenClawConfig` 並返回新的 `OpenClawConfig`，
遵循不可變（Immutable）原則。這確保了配置變更的可追蹤性和可回滾性。

**多帳號支援**：`listAccountIds()` 和 `resolveAccount()` 支援一個 Channel 有多個帳號。
例如，一個組織可以同時有多個 Slack workspace 連接。

### 3.2 Outbound Adapter（訊息發送適配器）

負責將正規化的回覆酬載（Reply Payload）轉換為平台特定的訊息格式：

```typescript
// source-repo/src/channels/plugins/outbound.types.ts:53-112（概要）
export type ChannelOutboundAdapter = {
  deliveryMode: "direct" | "gateway" | "hybrid";
  chunker?: ((text: string, limit: number) => string[]) | null;
  textChunkLimit?: number;
  sanitizeText?: (params) => string;
  pollMaxOptions?: number;
  normalizePayload?: (params) => ReplyPayload | null;
  resolveTarget?: (cfg?, to?, ...) => { ok, to } | { ok: false, error };
  sendPayload?: (ctx) => Promise<OutboundDeliveryResult>;
  sendFormattedText?: (ctx) => Promise<OutboundDeliveryResult[]>;
  sendFormattedMedia?: (ctx) => Promise<OutboundDeliveryResult>;
  sendText?: (ctx) => Promise<OutboundDeliveryResult>;
  sendMedia?: (ctx) => Promise<OutboundDeliveryResult>;
  sendPoll?: (ctx) => Promise<ChannelPollResult>;
};
```

**三種遞送模式**：
- `"direct"` — 直接透過平台 API 發送（大多數平台）
- `"gateway"` — 透過 OpenClaw Gateway 集中發送
- `"hybrid"` — 根據情況選擇直接或 Gateway

**文字分塊**：不同平台有不同的訊息長度限制。`chunker` 和 `textChunkLimit` 讓
每個平台定義自己的分塊策略。

### 3.3 Security Adapter（安全適配器）

處理平台特定的安全策略：

```typescript
// source-repo/src/channels/plugins/types.adapters.ts:817-853
export type ChannelSecurityAdapter<ResolvedAccount = unknown> = {
  applyConfigFixes?: (cfg, env) => ChannelDoctorConfigMutation | Promise<...>;
  resolveDmPolicy?: (ctx: ChannelSecurityContext) => ChannelSecurityDmPolicy | null;
  collectWarnings?: (ctx) => Promise<string[]> | string[];
  collectAuditFindings?: (ctx) => Promise<Array<{
    checkId, severity, title, detail, remediation?
  }>>;
};
```

**DM 策略**：`resolveDmPolicy()` 決定 Bot 是否可以主動發送私人訊息。
這在不同平台有不同的限制（例如 Discord 需要共同伺服器才能 DM）。

### 3.4 Gateway Adapter（Gateway 生命週期適配器）

管理 Channel 在 OpenClaw Gateway（長駐程序）中的生命週期：

```typescript
// source-repo/src/channels/plugins/types.adapters.ts:335-350
export type ChannelGatewayAdapter<ResolvedAccount = unknown> = {
  startAccount?: (ctx: ChannelGatewayContext<ResolvedAccount>) => Promise<unknown>;
  stopAccount?: (ctx: ChannelGatewayContext<ResolvedAccount>) => Promise<void>;
  resolveGatewayAuthBypassPaths?: (cfg) => string[];
  loginWithQrStart?: (accountId?, force?, timeoutMs?, verbose?)
    => Promise<ChannelLoginWithQrStartResult>;
  loginWithQrWait?: (accountId?, timeoutMs?)
    => Promise<ChannelLoginWithQrWaitResult>;
  logoutAccount?: (ctx) => Promise<ChannelLogoutResult>;
};
```

**QR Code 認證**：某些平台（如 WhatsApp、WeChat）使用 QR Code 登入。
`loginWithQrStart()` 和 `loginWithQrWait()` 提供了統一的 QR 認證介面。

### 3.5 Threading Adapter（對話串適配器）

正規化不同平台的對話串（Thread）模型：

```typescript
// source-repo/src/channels/plugins/types.core.ts:345-386（概要推導）
export type ChannelThreadingAdapter = {
  resolveReplyToMode?: (cfg, accountId?, chatType?)
    => "off" | "first" | "all" | "batched";
  allowExplicitReplyTagsWhenOff?: boolean;
  buildToolContext?: (cfg, accountId?, context, hasRepliedRef?)
    => ChannelThreadingToolContext | undefined;
  resolveAutoThreadId?: (cfg, accountId?, to, toolContext?, replyToId?)
    => string | undefined;
  resolveReplyTransport?: (cfg, accountId?, threadId?, replyToId?)
    => ChannelReplyTransport | null;
  resolveFocusedBinding?: (cfg, accountId?, context)
    => ChannelFocusedBindingContext | null;
};
```

**四種回覆模式**：
- `"off"` — 不使用 Thread
- `"first"` — 只在第一則訊息建立 Thread
- `"all"` — 每則回覆都在 Thread 中
- `"batched"` — 批次收集後統一回覆

### 3.6 Lifecycle Adapter（生命週期適配器）

處理 Channel 的配置變更和啟動維護：

```typescript
// source-repo/src/channels/plugins/types.adapters.ts:523-550
export type ChannelLifecycleAdapter = {
  onAccountConfigChanged?: (prevCfg, nextCfg, accountId, runtime) => Promise<void> | void;
  onAccountRemoved?: (prevCfg, accountId, runtime) => Promise<void> | void;
  runStartupMaintenance?: (cfg, env?, log?, trigger?, logPrefix?) => Promise<void> | void;
  detectLegacyStateMigrations?: (cfg, env, stateDir, oauthDir)
    => ChannelLegacyStateMigrationPlan[] | Promise<...>;
};
```

---

## 4. Capabilities 宣告

### 4.1 靜態能力聲明

`ChannelCapabilities` 型別（定義在 `src/channels/plugins/types.core.ts`）
讓每個 Channel 宣告它支援的功能。核心使用這些宣告來決定對特定平台啟用哪些功能。

能力包括但不限於：
- **Media**：支援圖片、檔案、影片上傳
- **Reactions**：支援 Emoji 反應
- **Threads**：支援對話串
- **Voice**：支援語音通話
- **Polls**：支援投票
- **Buttons**：支援互動按鈕

### 4.2 Runtime 能力狀態

除了靜態宣告，某些能力的可用性取決於 Runtime 狀態：

```typescript
// source-repo/src/channels/plugins/types.adapters.ts:605-645（概要）
export type ChannelApprovalCapability = ChannelApprovalAdapter & {
  getActionAvailabilityState?: (cfg, accountId?, action, approvalKind?)
    => ChannelActionAvailabilityState;
  // ChannelActionAvailabilityState: "enabled" | "disabled" | "unsupported"
};
```

---

## 5. Entry Contract：Channel Plugin 的入口契約

### 5.1 BundledChannelEntryContract

每個內建 Channel Plugin 都遵循統一的入口契約：

```typescript
// source-repo/src/plugin-sdk/channel-entry-contract.ts:50-61
export type BundledChannelEntryContract<TPlugin = ChannelPlugin> = {
  kind: "bundled-channel-entry";
  id: string;
  name: string;
  description: string;
  configSchema: ChannelEntryConfigSchema<TPlugin>;
  register: (api: OpenClawPluginApi) => void;
  loadChannelPlugin: () => TPlugin;
  loadChannelSecrets?: () => ChannelPlugin["secrets"] | undefined;
  setChannelRuntime?: (runtime: PluginRuntime) => void;
};
```

`loadChannelPlugin()` 使用懶載入——Channel Plugin 的完整程式碼只在需要時才被載入。
`defineBundledChannelEntry()` 是建立入口契約的工廠函式（`source-repo/src/plugin-sdk/channel-entry-contract.ts:327-380`）。

### 5.2 Architecture Layer 概覽

Channel 系統的分層架構如下（來自 `source-repo/extensions/AGENTS.md`）：

```
extensions/<channel>/                    ← 具體平台實作
    ↓ import
src/plugin-sdk/*                         ← 公開契約（Plugin SDK）
    ↓ use
src/channels/*                           ← 核心 Channel 實作
    ↓ use
src/plugins/*                            ← 發現、Registry、Loader
    ↓ use
src/gateway/protocol/*                   ← Wire 協定
```

**核心不變式（Core Invariant）**：核心程式碼必須保持 extension-agnostic——
不能有任何特定 Channel 的硬編碼邏輯。

---

## 6. 實際 Channel 實作範例

### 6.1 Discord Channel

Discord Plugin 的 manifest（`source-repo/extensions/discord/openclaw.plugin.json`）：

```json
{
  "id": "discord",
  "channels": ["discord"],
  "channelEnvVars": {
    "discord": ["DISCORD_BOT_TOKEN"]
  }
}
```

Discord Channel 的實作展示了幾個典型模式：

```typescript
// source-repo/extensions/discord/src/channel.ts（概要）
// 1. 使用 createScopedDmSecurityResolver 建立 DM 策略解析器
const resolveDiscordDmPolicy = createScopedDmSecurityResolver<ResolvedDiscordAccount>({
  channelKey: "discord",
  resolvePolicy: (account) => account.dm?.policy,
  resolveAllowFrom: (account) => account.dm?.allowFrom,
  allowFromPathSuffix: "dm.",
  normalizeEntry: (raw) => raw.trim().replace(/^discord:/i, "").trim(),
});

// 2. 懶載入模式——Runtime 模組只在需要時才 import
// discordProviderRuntimePromise, discordProbeRuntimePromise 等
```

### 6.2 Slack Channel

Slack Channel 展示了雙 Token 支援模式：

```typescript
// source-repo/extensions/slack/src/channel.ts（概要）
function getTokenForOperation(
  account: ResolvedSlackAccount,
  operation: "read" | "write",
): string | undefined {
  const userToken = normalizeOptionalString(account.config.userToken);
  const botToken = normalizeOptionalString(account.botToken);
  const allowUserWrites = account.config.userTokenReadOnly === false;
  if (operation === "read") return userToken ?? botToken;
  if (!allowUserWrites) return botToken;
  return botToken ?? userToken;
}
```

Slack 支援 Bot Token 和 User Token 兩種認證，某些操作（如讀取訊息歷史）
可能需要 User Token，而發送訊息使用 Bot Token。這種平台特定的複雜性被
封裝在 Adapter 內部。

---

## 7. Channel 目錄與 Registry

### 7.1 Channel Catalog Registry

系統會維護一份所有可用 Channel 的目錄：

```typescript
// source-repo/src/plugins/channel-catalog-registry.ts:1-56
export type PluginChannelCatalogEntry = {
  pluginId: string;
  origin: PluginOrigin;
  packageName?: string;
  workspaceDir?: string;
  rootDir: string;
  channel: PluginPackageChannel;
  install?: PluginPackageInstall;
};

export function listChannelCatalogEntries(params?): PluginChannelCatalogEntry[]
```

### 7.2 Channel ID 與 Plugin 所有權

Channel ID 到 Plugin ID 的映射由 `channel-plugin-ids.ts` 管理：

```typescript
// source-repo/src/plugins/channel-plugin-ids.ts:92-100
export function resolveScopedChannelOwnerPluginIds(): Map<string, string>
// 例如："discord" → "discord", "slack" → "slack"
```

`isChannelPluginEligibleForScopedOwnership()` 確保一個 Channel ID 只能被一個
Plugin 擁有（`source-repo/src/plugins/channel-plugin-ids.ts:63-90`）。

---

## 引用來源

| 來源檔案 | 行號 | 內容 |
|---------|------|------|
| `source-repo/src/channels/plugins/types.plugin.ts` | 53-96 | `ChannelPlugin` 完整型別 |
| `source-repo/src/channels/plugins/types.adapters.ts` | 76-150 | `ChannelConfigAdapter` |
| `source-repo/src/channels/plugins/types.adapters.ts` | 335-350 | `ChannelGatewayAdapter` |
| `source-repo/src/channels/plugins/types.adapters.ts` | 523-550 | `ChannelLifecycleAdapter` |
| `source-repo/src/channels/plugins/types.adapters.ts` | 605-645 | `ChannelApprovalCapability` |
| `source-repo/src/channels/plugins/types.adapters.ts` | 817-853 | `ChannelSecurityAdapter` |
| `source-repo/src/channels/plugins/outbound.types.ts` | 53-112 | `ChannelOutboundAdapter` |
| `source-repo/src/plugin-sdk/channel-entry-contract.ts` | 50-61 | `BundledChannelEntryContract` |
| `source-repo/src/plugin-sdk/channel-entry-contract.ts` | 327-380 | `defineBundledChannelEntry()` |
| `source-repo/src/plugin-sdk/channel-contract.ts` | 1-39 | Channel 公開型別匯出 |
| `source-repo/src/plugins/channel-catalog-registry.ts` | 1-56 | Channel 目錄 Registry |
| `source-repo/src/plugins/channel-plugin-ids.ts` | 63-100 | Channel 所有權解析 |
| `source-repo/extensions/discord/openclaw.plugin.json` | — | Discord manifest |
| `source-repo/extensions/discord/src/channel.ts` | — | Discord Channel 實作 |
| `source-repo/extensions/slack/src/channel.ts` | — | Slack Channel 實作 |
| `source-repo/extensions/AGENTS.md` | — | Architecture 分層規則 |
