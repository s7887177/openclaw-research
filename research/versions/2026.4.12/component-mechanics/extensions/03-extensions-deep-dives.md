# Extensions 重點深入分析

> **本章摘要**：本文從各類別中挑選代表性擴充套件，深入解析其原始碼實作。每個擴充套件從進入點追蹤到核心邏輯，揭示「一個擴充套件如何與 OpenClaw 核心互動」的完整路徑。這是理解 OpenClaw 可擴展性的關鍵章節。

---

## 1. LLM Provider 深入：Anthropic

Anthropic 擴充套件是 OpenClaw 最成熟的 LLM Provider 實作之一，展示了完整的 Provider 註冊流程。

### 1.1 進入點

```typescript
// source-repo/extensions/anthropic/index.ts:1-11
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { registerAnthropicPlugin } from "./register.runtime.js";

export default definePluginEntry({
  id: "anthropic",
  name: "Anthropic Provider",
  description: "Bundled Anthropic provider plugin",
  register(api) {
    return registerAnthropicPlugin(api);
  },
});
```

`definePluginEntry()` 向核心註冊一個擴充套件。當核心啟動並載入此擴充套件時，會呼叫 `register(api)` 函式，將 `OpenClawPluginApi` 物件傳入。

### 1.2 註冊邏輯

`register.runtime.ts`（496 行）是核心邏輯所在。它從 Plugin SDK 匯入大量模組：

```typescript
// source-repo/extensions/anthropic/register.runtime.ts:1-30
import { formatCliCommand, parseDurationMs } from "openclaw/plugin-sdk/cli-runtime";
import type {
  OpenClawPluginApi,
  ProviderAuthContext,
  ProviderAuthMethodNonInteractiveContext,
  ProviderResolveDynamicModelContext,
  ProviderRuntimeModel,
} from "openclaw/plugin-sdk/plugin-entry";
import {
  applyAuthProfileConfig,
  buildTokenProfileId,
  createProviderApiKeyAuthMethod,
  listProfilesForProvider,
  suggestOAuthProfileIdForLegacyDefault,
  upsertAuthProfile,
  validateAnthropicSetupToken,
} from "openclaw/plugin-sdk/provider-auth";
import { cloneFirstTemplateModel } from "openclaw/plugin-sdk/provider-model-shared";
import { fetchClaudeUsage } from "openclaw/plugin-sdk/provider-usage";
```

關鍵常數定義：

```typescript
// source-repo/extensions/anthropic/register.runtime.ts:40-56
const PROVIDER_ID = "anthropic";
const DEFAULT_ANTHROPIC_MODEL = "anthropic/claude-sonnet-4-6";
const ANTHROPIC_OPUS_46_MODEL_ID = "claude-opus-4-6";
const ANTHROPIC_SONNET_46_MODEL_ID = "claude-sonnet-4-6";
const ANTHROPIC_MODERN_MODEL_PREFIXES = [
  "claude-opus-4-6", "claude-sonnet-4-6",
  "claude-opus-4-5", "claude-sonnet-4-5", "claude-haiku-4-5",
] as const;
```

### 1.3 Plugin SDK 使用的關鍵 API

Anthropic 擴充套件展示了 Plugin SDK 的以下能力：

| SDK API | 用途 |
|---------|------|
| `api.registerProvider()` | 註冊 LLM Provider |
| `api.registerCliBackend()` | 註冊 CLI 後端（Claude CLI） |
| `createProviderApiKeyAuthMethod()` | 建立 API Key 認證方法 |
| `validateAnthropicSetupToken()` | 驗證 Setup Token |
| `cloneFirstTemplateModel()` | 從模板建立新模型定義 |
| `fetchClaudeUsage()` | 取得 Claude 用量資料 |
| `wrapAnthropicProviderStream()` | 包裝串流回應（注入 Beta headers、service tier 等） |

### 1.4 認證體系

Anthropic 支援三種認證方式（在 `openclaw.plugin.json` 中定義）：

1. **Claude CLI 重用**（`method: "cli"`）：偵測本機已登入的 Claude CLI，透過 `readClaudeCliCredentialsCached()` 讀取 token
2. **API Key**（`method: "api-key"`）：直接輸入 API 金鑰
3. **Setup Token**：Anthropic 專用的初始化 token（OpenClaw 專用路徑）

> **來源**：`source-repo/extensions/anthropic/register.runtime.ts:1-80`、`source-repo/extensions/anthropic/openclaw.plugin.json:14-37`、`source-repo/extensions/anthropic/cli-auth-seam.ts:1`

---

## 2. Channel 深入：Discord

Discord 擴充套件是 OpenClaw 最複雜的通道實作，原始碼規模超過 100 個檔案。

### 2.1 通道進入點

```typescript
// source-repo/extensions/discord/index.ts:1-41
import { defineBundledChannelEntry } from "openclaw/plugin-sdk/channel-entry-contract";

export default defineBundledChannelEntry({
  id: "discord",
  name: "Discord",
  description: "Discord channel plugin",
  importMetaUrl: import.meta.url,
  plugin: {
    specifier: "./channel-plugin-api.js",
    exportName: "discordPlugin",
  },
  runtime: {
    specifier: "./runtime-api.js",
    exportName: "setDiscordRuntime",
  },
  registerFull(api) {
    api.on("subagent_spawning", async (event) => {
      const { handleDiscordSubagentSpawning } = await loadDiscordSubagentHooksModule();
      return await handleDiscordSubagentSpawning(api, event);
    });
    api.on("subagent_ended", async (event) => { ... });
    api.on("subagent_delivery_target", async (event) => { ... });
  },
});
```

通道擴充套件與一般擴充套件的關鍵差異：
- 使用 `defineBundledChannelEntry()` 而非 `definePluginEntry()`
- 透過 `plugin` 和 `runtime` 物件進行**延遲載入**（specifier + exportName）
- 在 `registerFull()` 中掛鉤 subagent 事件（spawning、ended、delivery_target）

### 2.2 Discord API 封裝

```typescript
// source-repo/extensions/discord/src/channel.ts:1-7
import { resolveFetch } from "openclaw/plugin-sdk/fetch-runtime";
import {
  resolveRetryConfig,
  retryAsync,
  type RetryConfig,
} from "openclaw/plugin-sdk/retry-runtime";

const DISCORD_API_BASE = "https://discord.com/api/v10";
```

Discord 使用 Discord API v10，並透過 Plugin SDK 的 `resolveFetch()` 和 `retryAsync()` 來處理 HTTP 請求和重試邏輯（預設 3 次重試，500ms-30s 延遲）。

### 2.3 關鍵子系統

Discord 擴充套件的 `src/` 目錄包含以下子系統：

| 子系統 | 檔案（部分） | 功能 |
|--------|-------------|------|
| **通道核心** | `channel.ts` (918 行) | 通道生命週期管理 |
| **訊息監聽** | `monitor.ts`, `monitor.gateway.ts` | 接收和處理 Discord 事件 |
| **訊息發送** | `send.ts`, `send.messages.ts`, `send.components.ts` | 發送訊息、Discord Components v2 |
| **目錄/快取** | `directory-cache.ts`, `directory-live.ts` | 伺服器/頻道/用戶目錄 |
| **安全審計** | `security-audit.ts`, `security-doctor.ts` | 安全檢查和建議 |
| **語音** | `voice/` 子目錄 | Discord 語音通道支援 |
| **互動** | `shared-interactive.ts`, `interactive-dispatch.ts` | 按鈕/選單等互動元素 |
| **子代理鉤子** | `subagent-hooks.ts` | 子代理在 Discord 線程中的行為 |

### 2.4 Manifest 極簡性

```json
// source-repo/extensions/discord/openclaw.plugin.json
{
  "id": "discord",
  "channels": ["discord"],
  "channelEnvVars": {
    "discord": ["DISCORD_BOT_TOKEN"]
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

Discord 的 manifest 極其精簡——僅需一個 `DISCORD_BOT_TOKEN`。所有複雜度都在 TypeScript 實作中。

> **來源**：`source-repo/extensions/discord/openclaw.plugin.json`、`source-repo/extensions/discord/index.ts:1-41`、`source-repo/extensions/discord/src/channel.ts:1-40`、`source-repo/extensions/discord/src/` 目錄

---

## 3. Memory 深入：memory-core 的夢境系統

memory-core 是 OpenClaw 記憶系統的核心，其最獨特的設計是「夢境」（Dreaming）機制——模擬人類睡眠中的記憶鞏固過程。

### 3.1 進入點與能力註冊

```typescript
// source-repo/extensions/memory-core/index.ts:1-40
export default definePluginEntry({
  id: "memory-core",
  name: "Memory (Core)",
  description: "File-backed memory search tools and CLI",
  kind: "memory",
  register(api) {
    registerBuiltInMemoryEmbeddingProviders(api);
    registerShortTermPromotionDreaming(api);
    registerDreamingCommand(api);
    api.registerMemoryCapability({
      promptBuilder: buildPromptSection,
      flushPlanResolver: buildMemoryFlushPlan,
      runtime: memoryRuntime,
      publicArtifacts: {
        listArtifacts: listMemoryCorePublicArtifacts,
      },
    });
    // ... 註冊工具
  },
});
```

`registerMemoryCapability()` 是 memory 擴充套件的核心 API，它告訴 OpenClaw：
- **如何注入提示**（`promptBuilder`）：在每次對話前注入相關記憶
- **如何刷新**（`flushPlanResolver`）：何時將短期記憶寫入長期儲存
- **執行時行為**（`runtime`）：記憶搜尋和存取的實作

### 3.2 夢境三階段

夢境系統的配置在 `openclaw.plugin.json` 中定義，對應三個階段：

| 階段 | 英文 | 功能 | 關鍵參數 |
|------|------|------|----------|
| 淺眠 | Light | 掃描近期對話，去除重複 | `lookbackDays`, `dedupeSimilarity` |
| 快速動眼 | REM | 發現跨對話的模式和關聯 | `lookbackDays`, `minPatternStrength` |
| 深眠 | Deep | 鞏固重要記憶，淘汰不重要的 | `minScore`, `minRecallCount`, `maxAgeDays` |

```typescript
// source-repo/extensions/memory-core/src/dreaming-phases.ts:1-20
import type { OpenClawConfig, OpenClawPluginApi } from "openclaw/plugin-sdk/memory-core";
import {
  buildSessionEntry,
  listSessionFilesForAgent,
} from "openclaw/plugin-sdk/memory-core-host-engine-qmd";
import {
  formatMemoryDreamingDay,
  resolveMemoryDreamingWorkspaces,
  resolveMemoryLightDreamingConfig,
  resolveMemoryRemDreamingConfig,
} from "openclaw/plugin-sdk/memory-core-host-status";
```

夢境可以透過 cron 排程自動執行：

```json
// source-repo/extensions/memory-core/openclaw.plugin.json:6-8
"uiHints": {
  "dreaming.frequency": {
    "label": "Dreaming Frequency",
    "placeholder": "0 3 * * *",
    "help": "Optional cron cadence for the full dreaming sweep (light, REM, then deep)."
  }
}
```

> **來源**：`source-repo/extensions/memory-core/index.ts:1-40`、`source-repo/extensions/memory-core/src/dreaming-phases.ts:1-50`、`source-repo/extensions/memory-core/openclaw.plugin.json:4-10`

---

## 4. Web Search 深入：Tavily

Tavily 是 OpenClaw 中最「教科書式」的擴充套件——結構清晰，完整展示了一個工具型擴充套件的寫法。

### 4.1 完整註冊

```typescript
// source-repo/extensions/tavily/index.ts:1-15
import { definePluginEntry, type AnyAgentTool } from "openclaw/plugin-sdk/plugin-entry";
import { createTavilyExtractTool } from "./src/tavily-extract-tool.js";
import { createTavilyWebSearchProvider } from "./src/tavily-search-provider.js";
import { createTavilySearchTool } from "./src/tavily-search-tool.js";

export default definePluginEntry({
  id: "tavily",
  name: "Tavily Plugin",
  description: "Bundled Tavily search and extract plugin",
  register(api) {
    api.registerWebSearchProvider(createTavilyWebSearchProvider());
    api.registerTool(createTavilySearchTool(api) as AnyAgentTool);
    api.registerTool(createTavilyExtractTool(api) as AnyAgentTool);
  },
});
```

Tavily 同時註冊了三個能力：
1. **Web Search Provider**：作為 OpenClaw 的搜尋後端
2. **tavily_search 工具**：讓 AI Agent 可以主動搜尋網路
3. **tavily_extract 工具**：讓 AI Agent 可以擷取網頁內容

### 4.2 合約宣告

```json
// source-repo/extensions/tavily/openclaw.plugin.json:25-28
"contracts": {
  "webSearchProviders": ["tavily"],
  "tools": ["tavily_search", "tavily_extract"]
}
```

`contracts.tools` 欄位讓核心知道這個擴充套件會註冊哪些工具名稱。

> **來源**：`source-repo/extensions/tavily/index.ts:1-15`、`source-repo/extensions/tavily/openclaw.plugin.json`

---

## 5. Gateway Proxy 深入：LiteLLM

LiteLLM 是一個「元供應商」——透過 LiteLLM 代理層統一存取 100+ LLM 供應商。

### 5.1 Manifest

```json
// source-repo/extensions/litellm/openclaw.plugin.json
{
  "id": "litellm",
  "enabledByDefault": true,
  "providers": ["litellm"],
  "providerAuthEnvVars": {
    "litellm": ["LITELLM_API_KEY"]
  },
  "providerAuthChoices": [{
    "provider": "litellm",
    "method": "api-key",
    "choiceId": "litellm-api-key",
    "choiceLabel": "LiteLLM API key",
    "choiceHint": "Unified gateway for 100+ LLM providers",
    "groupId": "litellm",
    "groupLabel": "LiteLLM",
    "groupHint": "Unified LLM gateway (100+ providers)",
    "optionKey": "litellmApiKey",
    "cliFlag": "--litellm-api-key",
    "cliOption": "--litellm-api-key <key>",
    "cliDescription": "LiteLLM API key"
  }]
}
```

LiteLLM 的設計哲學是：OpenClaw 不需要為每個 LLM 供應商都寫擴充套件——只要用戶有一個 LiteLLM 代理，就可以透過單一 API Key 存取所有供應商。

> **來源**：`source-repo/extensions/litellm/openclaw.plugin.json`

---

## 6. Tool 深入：Browser

Browser 擴充套件讓 AI Agent 可以操控瀏覽器。

### 6.1 進入點

```typescript
// source-repo/extensions/browser/index.ts:1-15
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import {
  browserPluginNodeHostCommands,
  browserPluginReload,
  browserSecurityAuditCollectors,
  registerBrowserPlugin,
} from "./plugin-registration.js";

export default definePluginEntry({
  id: "browser",
  name: "Browser",
  description: "Default browser tool plugin",
  reload: browserPluginReload,
  nodeHostCommands: browserPluginNodeHostCommands,
  securityAuditCollectors: [...browserSecurityAuditCollectors],
  register: registerBrowserPlugin,
});
```

Browser 擴充套件的特殊之處：
- **`reload`**：支援熱重載
- **`nodeHostCommands`**：可以向 Node host（行動 app 或 macOS app）發送瀏覽器指令
- **`securityAuditCollectors`**：提供安全審計資料（瀏覽器是高風險工具）

> **來源**：`source-repo/extensions/browser/index.ts:1-15`、`source-repo/extensions/browser/openclaw.plugin.json`

---

## 7. Active Memory 深入

Active Memory 是一個自動記憶注入擴充套件，在每次回覆前執行一個「記憶子代理」（memory sub-agent）。

### 7.1 配置

```json
// source-repo/extensions/active-memory/openclaw.plugin.json (節選)
{
  "id": "active-memory",
  "name": "Active Memory",
  "description": "Runs a bounded blocking memory sub-agent before eligible conversational 
    replies and injects relevant memory into prompt context.",
  "configSchema": {
    "properties": {
      "enabled": { "type": "boolean" },
      "agents": { "type": "array", "items": { "type": "string" } },
      "model": { "type": "string" },
      "allowedChatTypes": {
        "type": "array",
        "items": { "type": "string", "enum": ["direct", "group", "channel"] }
      },
      "thinking": {
        "type": "string",
        "enum": ["off", "minimal", "low", "medium", "high", "xhigh", "adaptive"]
      }
    }
  }
}
```

Active Memory 的工作流程：
1. 用戶發送訊息 → 2. Active Memory 攔截 → 3. 用獨立 LLM 呼叫搜尋記憶 → 4. 將相關記憶注入 prompt context → 5. 主 LLM 帶著記憶回覆

可配置的 `thinking` 層級（從 off 到 xhigh）控制記憶子代理的推理深度。

> **來源**：`source-repo/extensions/active-memory/openclaw.plugin.json`

---

## 8. shared/ 擴充套件分析

`extensions/shared/` 不是一個獨立擴充套件（沒有 `package.json` 或 `openclaw.plugin.json`），而是提供跨擴充套件共用的工具函式。

| 檔案 | 匯出 | 用途 |
|------|------|------|
| `runtime.ts` | `resolveLoggerBackedRuntime` | 從 `openclaw/plugin-sdk/extension-shared` 重新匯出，提供帶日誌的執行時環境 |
| `channel-status-summary.ts` | 通道狀態摘要函式 | 標準化各通道的狀態回報格式 |
| `config-schema-helpers.ts` | Schema 輔助工具 | 配置 JSON Schema 的共用操作 |
| `deferred.ts` | Deferred promise | 延遲載入模式 |
| `passive-monitor.ts` | 被動監控基礎類 | 通道監控的共用邏輯 |
| `status-issues.ts` | 狀態問題回報 | 標準化問題/警告格式 |

```typescript
// source-repo/extensions/shared/runtime.ts:1
export { resolveLoggerBackedRuntime } from "openclaw/plugin-sdk/extension-shared";
```

> **來源**：`source-repo/extensions/shared/runtime.ts:1`、`source-repo/extensions/shared/` 目錄

---

## 設計模式總結

從深入分析中浮現的擴充套件設計模式：

| 模式 | 說明 | 範例 |
|------|------|------|
| **Manifest-first** | 靜態能力由 JSON 宣告，動態邏輯由 TypeScript 實作 | 所有擴充套件 |
| **Register callback** | 核心透過 `register(api)` 注入能力 | anthropic、tavily、memory-core |
| **Lazy loading** | 通道用 specifier 延遲載入大型模組 | discord、slack |
| **Contract discovery** | 核心透過合約發現擴充套件的能力 | tavily（webSearch）、elevenlabs（speech） |
| **Auth profile** | 認證資訊以 profile 管理（多帳號） | anthropic、openai |
| **Core runtime re-export** | `-core` 擴充套件重新匯出 SDK 模組 | speech-core、image-generation-core |

---

## 引用來源

| 來源路徑 | 引用內容 |
|----------|----------|
| `source-repo/extensions/anthropic/index.ts:1-11` | Anthropic 進入點 |
| `source-repo/extensions/anthropic/register.runtime.ts:1-80` | Anthropic 註冊邏輯 |
| `source-repo/extensions/anthropic/openclaw.plugin.json:14-37` | 認證選項 |
| `source-repo/extensions/anthropic/cli-auth-seam.ts:1` | CLI 認證 |
| `source-repo/extensions/discord/index.ts:1-41` | Discord 進入點 |
| `source-repo/extensions/discord/openclaw.plugin.json` | Discord manifest |
| `source-repo/extensions/discord/src/channel.ts:1-40` | Discord API |
| `source-repo/extensions/memory-core/index.ts:1-40` | Memory Core 進入點 |
| `source-repo/extensions/memory-core/src/dreaming-phases.ts:1-50` | 夢境階段 |
| `source-repo/extensions/memory-core/openclaw.plugin.json:4-10` | 夢境 UI 提示 |
| `source-repo/extensions/tavily/index.ts:1-15` | Tavily 進入點 |
| `source-repo/extensions/tavily/openclaw.plugin.json:25-28` | Tavily 合約 |
| `source-repo/extensions/litellm/openclaw.plugin.json` | LiteLLM manifest |
| `source-repo/extensions/browser/index.ts:1-15` | Browser 進入點 |
| `source-repo/extensions/active-memory/openclaw.plugin.json` | Active Memory 配置 |
| `source-repo/extensions/shared/runtime.ts:1` | Shared 匯出 |
