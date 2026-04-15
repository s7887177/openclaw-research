# Channels 子章節 01：Registry 與 Plugin 型別

> **引用範圍**：`src/channels/plugins/types.plugin.ts`, `types.core.ts`, `types.adapters.ts`, `registry.ts`, `src/channels/registry.ts`

## 1. ChannelPlugin 介面

`ChannelPlugin` 是所有通道 Plugin 的核心合約，定義在 `types.plugin.ts`。每個平台（Discord、Telegram 等）必須實作此介面。

### 1.1 完整型別定義

```typescript
// source-repo/src/channels/plugins/types.plugin.ts:53-96
export type ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown> = {
  id: ChannelId;                           // 唯一識別碼（如 "discord", "telegram"）
  meta: ChannelMeta;                       // 使用者可見的後設資料
  capabilities: ChannelCapabilities;       // 靜態能力宣告
  defaults?: {
    queue?: { debounceMs?: number };       // 預設佇列防抖
  };
  reload?: {                                // 熱重載配置
    configPrefixes: string[];
    noopPrefixes?: string[];
  };
  setupWizard?: ChannelPluginSetupWizard;  // 設定精靈

  // ── 適配器（約 25 個） ──
  config: ChannelConfigAdapter<ResolvedAccount>;       // 配置解析（必填）
  configSchema?: ChannelConfigSchema;                  // 配置 Schema
  setup?: ChannelSetupAdapter;                         // 設定流程
  pairing?: ChannelPairingAdapter;                     // 配對
  security?: ChannelSecurityAdapter<ResolvedAccount>;  // 安全策略
  groups?: ChannelGroupAdapter;                        // 群組管理
  mentions?: ChannelMentionAdapter;                    // @提及處理
  outbound?: ChannelOutboundAdapter;                   // 出站訊息
  status?: ChannelStatusAdapter<ResolvedAccount, Probe, Audit>;  // 狀態檢查
  gatewayMethods?: string[];                           // 自訂 Gateway 方法
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;    // Gateway 互動
  auth?: ChannelAuthAdapter;                           // 認證
  approvalCapability?: ChannelApprovalCapability;      // 審批能力
  elevated?: ChannelElevatedAdapter;                   // 提升權限
  commands?: ChannelCommandAdapter;                    // 原生指令
  lifecycle?: ChannelLifecycleAdapter;                 // 生命週期
  secrets?: ChannelSecretsAdapter;                     // 密鑰管理
  allowlist?: ChannelAllowlistAdapter;                 // 允許清單
  doctor?: ChannelDoctorAdapter;                       // 診斷
  bindings?: ChannelConfiguredBindingProvider;          // 配置綁定
  conversationBindings?: ChannelConversationBindingSupport;  // 對話綁定
  streaming?: ChannelStreamingAdapter;                 // 串流
  threading?: ChannelThreadingAdapter;                 // Thread 管理
  messaging?: ChannelMessagingAdapter;                 // 訊息處理
  agentPrompt?: ChannelAgentPromptAdapter;             // Agent 提示注入
  directory?: ChannelDirectoryAdapter;                 // 目錄服務
  resolver?: ChannelResolverAdapter;                   // 解析器
  actions?: ChannelMessageActionAdapter;               // 訊息動作
  heartbeat?: ChannelHeartbeatAdapter;                 // 心跳
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];  // Agent 工具
};
```

唯一必填的適配器是 `config`，其餘皆為可選。這實現了**漸進式能力宣告**：簡單通道只需實作配置解析，複雜通道可以逐步添加能力。

> **引用來源**：`source-repo/src/channels/plugins/types.plugin.ts:53-96`

---

## 2. ChannelMeta：平台後設資料

```typescript
// source-repo/src/channels/plugins/types.core.ts:142-164
export type ChannelMeta = {
  id: ChannelId;                     // Plugin ID
  label: string;                     // 顯示名稱（如 "Discord"）
  selectionLabel: string;            // 選擇列表標籤
  docsPath: string;                  // 文件路徑
  docsLabel?: string;                // 文件標籤
  blurb: string;                     // 簡短描述
  order?: number;                    // 排序權重
  aliases?: readonly string[];       // 別名（如 "dc" → "discord"）
  selectionDocsPrefix?: string;
  selectionDocsOmitLabel?: boolean;
  selectionExtras?: readonly string[];
  detailLabel?: string;
  systemImage?: string;              // 系統圖片
  markdownCapable?: boolean;         // 是否支援 Markdown
  exposure?: ChannelExposure;        // 曝光設定
  showConfigured?: boolean;
  showInSetup?: boolean;
  quickstartAllowFrom?: boolean;
  forceAccountBinding?: boolean;
  preferSessionLookupForAnnounceTarget?: boolean;
  preferOver?: readonly string[];    // 優先於其他通道
};
```

`ChannelExposure` 控制通道在不同介面中的可見性：

```typescript
// source-repo/src/channels/plugins/types.core.ts:17-21
export type ChannelExposure = {
  configured?: boolean;    // 已配置時顯示
  setup?: boolean;         // 設定流程中顯示
  docs?: boolean;          // 文件中顯示
};
```

> **引用來源**：`source-repo/src/channels/plugins/types.core.ts:17-21, 142-164`

---

## 3. ChannelCapabilities：能力宣告

```typescript
// source-repo/src/channels/plugins/types.core.ts:253-266
export type ChannelCapabilities = {
  chatTypes: Array<ChatType | "thread">;  // 支援的聊天類型
  polls?: boolean;               // 投票
  reactions?: boolean;           // 反應 Emoji
  edit?: boolean;                // 編輯訊息
  unsend?: boolean;              // 撤回訊息
  reply?: boolean;               // 回覆訊息
  effects?: boolean;             // 特效
  groupManagement?: boolean;     // 群組管理
  threads?: boolean;             // Thread 支援
  media?: boolean;               // 媒體附件
  nativeCommands?: boolean;      // 原生指令（如 Slash Commands）
  blockStreaming?: boolean;      // 區塊串流
};
```

ChatType 的可能值：

```typescript
// source-repo/src/channels/chat-type.ts
export type ChatType = "direct" | "group" | "channel";
```

加上 `"thread"` 作為 capabilities 中的特殊類型。

> **引用來源**：`source-repo/src/channels/plugins/types.core.ts:253-266`

---

## 4. ChannelAccountSnapshot：帳號快照

`ChannelAccountSnapshot` 是通道帳號的完整狀態快照，包含 40+ 個欄位：

```typescript
// source-repo/src/channels/plugins/types.core.ts:167-230
export type ChannelAccountSnapshot = {
  accountId: string;
  name?: string;
  enabled?: boolean;
  configured?: boolean;
  linked?: boolean;
  running?: boolean;
  connected?: boolean;
  restartPending?: boolean;
  reconnectAttempts?: number;
  lastConnectedAt?: number | null;
  lastDisconnect?: string | { at: number; status?; error?; loggedOut? } | null;
  lastMessageAt?: number | null;
  lastEventAt?: number | null;
  lastError?: string | null;
  healthState?: string;
  lastStartAt?: number | null;
  lastStopAt?: number | null;
  lastInboundAt?: number | null;
  lastOutboundAt?: number | null;
  busy?: boolean;
  activeRuns?: number;
  lastRunActivityAt?: number | null;
  mode?: string;
  dmPolicy?: string;
  allowFrom?: string[];
  // ... 更多 token/credential 狀態欄位
};
```

此型別被 `channel status` 和生命週期介面使用，提供通道連線健康度的完整視圖。

> **引用來源**：`source-repo/src/channels/plugins/types.core.ts:167-230`

---

## 5. 適配器型別

### 5.1 Adapter 分類

適配器（Adapter）是 Plugin 介面的核心，約 25 個。以功能分類如下：

**配置與設定**：
| 適配器 | 用途 |
|--------|------|
| `ChannelConfigAdapter` | 配置解析、帳號解析（**必填**） |
| `ChannelConfigSchema` | JSON Schema 配置描述 |
| `ChannelSetupAdapter` | 初始設定流程 |
| `ChannelSecretsAdapter` | 密鑰管理 |

**安全與存取**：
| 適配器 | 用途 |
|--------|------|
| `ChannelSecurityAdapter` | DM 策略、安全上下文 |
| `ChannelAuthAdapter` | 登入/登出 |
| `ChannelAllowlistAdapter` | 允許清單 |
| `ChannelElevatedAdapter` | 權限提升 |
| `ChannelApprovalCapability` | 工具審批能力 |

**訊息處理**：
| 適配器 | 用途 |
|--------|------|
| `ChannelMessagingAdapter` | 訊息格式、解析 |
| `ChannelMentionAdapter` | @提及 strip/detect |
| `ChannelOutboundAdapter` | 出站訊息發送 |
| `ChannelStreamingAdapter` | 串流回覆 |
| `ChannelMessageActionAdapter` | 訊息動作（react、pin 等） |

**結構與組織**：
| 適配器 | 用途 |
|--------|------|
| `ChannelGroupAdapter` | 群組管理 |
| `ChannelThreadingAdapter` | Thread 策略 |
| `ChannelCommandAdapter` | 原生指令 |
| `ChannelDirectoryAdapter` | 使用者/通道目錄 |

**生命週期與監控**：
| 適配器 | 用途 |
|--------|------|
| `ChannelLifecycleAdapter` | 啟動/停止/重啟 |
| `ChannelStatusAdapter` | 狀態查詢、Probe |
| `ChannelHeartbeatAdapter` | 心跳檢查 |
| `ChannelDoctorAdapter` | 診斷工具 |

**其他**：
| 適配器 | 用途 |
|--------|------|
| `ChannelPairingAdapter` | 設備配對 |
| `ChannelGatewayAdapter` | Gateway 整合 |
| `ChannelResolverAdapter` | ID/名稱解析 |
| `ChannelAgentPromptAdapter` | Agent 提示注入 |
| `ChannelConfiguredBindingProvider` | Agent 路由綁定 |

> **引用來源**：`source-repo/src/channels/plugins/types.plugin.ts:10-32`, `types.adapters.ts:1-80`

### 5.2 ChannelMentionAdapter

```typescript
// source-repo/src/channels/plugins/types.core.ts:283-300
export type ChannelMentionAdapter = {
  stripRegexes?: (params) => RegExp[];       // 正則表達式清除 mention
  stripPatterns?: (params) => string[];      // 字串模式清除
  stripMentions?: (params) => string;        // 自訂清除函式
};
```

### 5.3 ChannelThreadingAdapter

```typescript
// source-repo/src/channels/plugins/types.core.ts:345-386
export type ChannelThreadingAdapter = {
  resolveReplyToMode?: (...) => "off" | "first" | "all" | "batched";
  allowExplicitReplyTagsWhenOff?: boolean;
  buildToolContext?: (...) => ChannelThreadingToolContext | undefined;
  resolveAutoThreadId?: (...) => string | undefined;
  resolveReplyTransport?: (...) => ChannelReplyTransport | null;
  resolveFocusedBinding?: (...) => ChannelFocusedBindingContext | null;
};
```

Thread 模式有四種：

| 模式 | 說明 |
|------|------|
| `off` | 不回覆到 Thread |
| `first` | 只回覆第一條到 Thread |
| `all` | 所有回覆到 Thread |
| `batched` | 批次回覆到 Thread |

> **引用來源**：`source-repo/src/channels/plugins/types.core.ts:345-386`

---

## 6. Plugin Registry

### 6.1 底層 Registry

```typescript
// source-repo/src/channels/plugins/registry.ts:1-31
const channelPlugins = new Map<string, ChannelPlugin>();

export function registerChannelPlugin(plugin: ChannelPlugin) { ... }
export function listChannelPlugins(): ChannelPlugin[] { ... }
export function getChannelPlugin(id: string): ChannelPlugin | undefined { ... }
export function normalizeChannelId(id: string): string { ... }
```

底層 Registry 是一個簡單的 `Map<string, ChannelPlugin>`，提供 CRUD 操作。

> **引用來源**：`source-repo/src/channels/plugins/registry.ts:1-31`

### 6.2 高階 Registry

```typescript
// source-repo/src/channels/registry.ts:1-113
export function normalizeAnyChannelId(id: string): string
export function listRegisteredChannelPluginIds(): string[]
export function getRegisteredChannelPlugin(id: string): ChannelPlugin | undefined
export function resolveChannelPluginForAccountId(
  cfg: OpenClawConfig, accountId: string
): ChannelPlugin | undefined
export function listConfiguredChannelPlugins(cfg: OpenClawConfig): ChannelPlugin[]
export function listConfiguredChannelPluginIds(cfg: OpenClawConfig): string[]
```

高階 Registry 在底層之上添加：
- **多源正規化**：處理別名、大小寫不敏感
- **配置感知查詢**：只回傳已配置的通道
- **帳號→通道映射**：根據 accountId 找到對應的通道 Plugin

> **引用來源**：`source-repo/src/channels/registry.ts:1-113`

---

## 7. Agent 工具擴充

通道 Plugin 可以貢獻自己的 Agent 工具：

```typescript
// source-repo/src/channels/plugins/types.plugin.ts:95
agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
```

可以是靜態工具陣列，或根據配置動態生成的工廠函式：

```typescript
// source-repo/src/channels/plugins/types.core.ts:31
export type ChannelAgentToolFactory = (params: { cfg?: OpenClawConfig }) => ChannelAgentTool[];
```

這允許通道為 Agent 提供平台特定工具（如 Discord 的角色管理、Slack 的頻道操作等）。

> **引用來源**：`source-repo/src/channels/plugins/types.core.ts:26-31`

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| ChannelPlugin | `source-repo/src/channels/plugins/types.plugin.ts` | 53-96 |
| ChannelMeta | `source-repo/src/channels/plugins/types.core.ts` | 142-164 |
| ChannelExposure | `source-repo/src/channels/plugins/types.core.ts` | 17-21 |
| ChannelCapabilities | `source-repo/src/channels/plugins/types.core.ts` | 253-266 |
| ChannelAccountSnapshot | `source-repo/src/channels/plugins/types.core.ts` | 167-230 |
| ChannelMentionAdapter | `source-repo/src/channels/plugins/types.core.ts` | 283-300 |
| ChannelThreadingAdapter | `source-repo/src/channels/plugins/types.core.ts` | 345-386 |
| ChannelAgentToolFactory | `source-repo/src/channels/plugins/types.core.ts` | 31 |
| Plugin Registry | `source-repo/src/channels/plugins/registry.ts` | 1-31 |
| 高階 Registry | `source-repo/src/channels/registry.ts` | 1-113 |
| Adapter 型別 | `source-repo/src/channels/plugins/types.adapters.ts` | 1-80 |
