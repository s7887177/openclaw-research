# Extensions 架構：目錄結構、Manifest 格式與 Plugin SDK 邊界

> **本章摘要**：本文從第一性原理出發，剖析 OpenClaw 擴充套件（Extension）系統的底層架構。首先描述 `extensions/` 目錄的組織方式，然後深入解析 `openclaw.plugin.json` 清單檔（manifest）的完整欄位規格，最後說明擴充套件與核心之間唯一的互動界面——Plugin SDK。讀完本章，你將理解一個擴充套件如何從原始碼變成 OpenClaw 的一部分。

---

## 1. extensions/ 目錄總覽

OpenClaw 原始碼樹的根目錄下有一個 `extensions/` 資料夾，它是所有擴充套件的容器。截至 v2026.4.12，共有 **111 個子目錄**，每個子目錄就是一個獨立的擴充套件。

### 1.1 根層級檔案

在 `extensions/` 根層級有兩個 TypeScript 設定檔，用於建立擴充套件之間的套件邊界（package boundary）：

| 檔案 | 用途 |
|------|------|
| `tsconfig.package-boundary.base.json` | 基礎 TypeScript 設定，繼承自父層級 |
| `tsconfig.package-boundary.paths.json` | 路徑對應設定，確保擴充套件之間的匯入邊界正確 |

> **來源**：`source-repo/extensions/tsconfig.package-boundary.base.json`、`source-repo/extensions/tsconfig.package-boundary.paths.json`

### 1.2 典型擴充套件目錄結構

每個擴充套件都是一個標準的 npm 套件，遵循相似的結構模式。以下以幾個代表性擴充套件為例：

**LLM Provider 型（以 anthropic 為例）**：

```
extensions/anthropic/
├── openclaw.plugin.json        # 清單檔（必要）
├── package.json                # npm 套件定義
├── tsconfig.json               # TypeScript 設定
├── index.ts                    # 進入點，呼叫 definePluginEntry()
├── index.test.ts               # 測試
├── register.runtime.ts         # 核心註冊邏輯（496 行）
├── api.ts                      # 公開 API 匯出
├── setup-api.ts                # 設定邏輯
├── cli-backend.ts              # CLI 後端整合
├── cli-auth-seam.ts            # CLI 認證接縫
├── cli-shared.ts               # CLI 共用工具
├── cli-migration.ts            # 遷移邏輯
├── config-defaults.ts          # 預設配置
├── media-understanding-provider.ts  # 媒體理解能力
├── provider-policy-api.ts      # Provider 策略
├── replay-policy.ts            # 重播策略
├── contract-api.ts             # 合約 API
└── stream-wrappers.ts          # 串流包裝器
```

> **來源**：`source-repo/extensions/anthropic/` 目錄結構

**Channel 型（以 discord 為例）**：

```
extensions/discord/
├── openclaw.plugin.json
├── package.json
├── index.ts                    # 呼叫 defineBundledChannelEntry()
└── src/
    ├── channel.ts              # 通道核心（918 行）
    ├── api.ts                  # Discord API v10 封裝
    ├── client.ts               # Discord 客戶端
    ├── runtime.ts              # 執行時邏輯
    ├── monitor.ts              # 訊息監聽器
    ├── send.ts                 # 訊息發送
    ├── components.ts           # Discord Components v2
    ├── security-audit.ts       # 安全審計
    ├── voice/                  # 語音通道支援
    └── actions/                # 通道動作（roles、moderation 等）
```

> **來源**：`source-repo/extensions/discord/` 目錄結構

**Core Runtime 型（以 memory-core 為例）**：

```
extensions/memory-core/
├── openclaw.plugin.json
├── package.json
├── index.ts
└── src/
    ├── cli.ts                  # CLI 指令
    ├── dreaming-phases.ts      # 夢境階段（light/REM/deep）
    ├── dreaming-command.ts     # /dreaming 指令
    ├── dreaming-narrative.ts   # 夢境敘事生成
    ├── dreaming-markdown.ts    # Markdown 格式輸出
    ├── flush-plan.ts           # 記憶刷新計劃
    ├── prompt-section.ts       # Prompt 注入區段
    ├── runtime-provider.ts     # 執行時供應者
    ├── tools.ts                # memory_search、memory_get 工具
    └── memory/
        └── provider-adapters.ts  # 嵌入式供應商適配器
```

> **來源**：`source-repo/extensions/memory-core/` 目錄結構

### 1.3 shared/ 目錄

`extensions/shared/` 是一個特殊的非套件目錄，不含 `package.json` 或 `openclaw.plugin.json`。它提供多個擴充套件共用的工具模組：

| 檔案 | 用途 |
|------|------|
| `runtime.ts` | 匯出 `resolveLoggerBackedRuntime`，源自 `openclaw/plugin-sdk/extension-shared` |
| `channel-status-summary.ts` | 通道狀態摘要工具 |
| `config-schema-helpers.ts` | 配置 schema 輔助工具 |
| `deferred.ts` | 延遲載入工具 |
| `passive-monitor.ts` | 被動監控工具 |
| `status-issues.ts` | 狀態問題回報工具 |
| `windows-cmd-shim-test-fixtures.ts` | Windows 命令 shim 測試固件 |

> **來源**：`source-repo/extensions/shared/` 目錄內容；`source-repo/extensions/shared/runtime.ts:1`

---

## 2. openclaw.plugin.json 清單檔完整規格

每個擴充套件的 `openclaw.plugin.json` 是它向 OpenClaw 核心「自我介紹」的方式。這個 JSON 檔案定義了擴充套件的身份、能力、依賴和配置 schema。

### 2.1 完整欄位清單

以下是從 111 個擴充套件中歸納出的所有可能欄位：

| 欄位 | 型別 | 說明 | 範例 |
|------|------|------|------|
| `id` | `string` | **必要**。擴充套件唯一識別碼 | `"anthropic"` |
| `name` | `string` | 人類可讀名稱 | `"Anthropic Provider"` |
| `description` | `string` | 簡短描述 | `"Bundled Anthropic provider plugin"` |
| `enabledByDefault` | `boolean` | 是否預設啟用 | `true` |
| `kind` | `string` | 擴充套件種類 | `"memory"` |
| `providers` | `string[]` | 此擴充套件提供的 LLM Provider ID | `["anthropic"]` |
| `channels` | `string[]` | 此擴充套件提供的通道 ID | `["discord"]` |
| `modelSupport` | `object` | 模型前綴匹配 | `{"modelPrefixes": ["claude-"]}` |
| `cliBackends` | `string[]` | CLI 後端 ID | `["claude-cli"]` |
| `providerAuthEnvVars` | `object` | 供應商認證環境變數對應 | `{"anthropic": ["ANTHROPIC_API_KEY"]}` |
| `providerAuthChoices` | `array` | 認證方式選項（method、choiceId 等） | 見下方 |
| `channelEnvVars` | `object` | 通道認證環境變數對應 | `{"discord": ["DISCORD_BOT_TOKEN"]}` |
| `autoEnableWhenConfiguredProviders` | `string[]` | 當這些 provider 被配置時自動啟用 | `["google-gemini-cli"]` |
| `providerDiscoveryEntry` | `string` | 動態 provider 發現的進入點 | `"./provider-discovery.ts"` |
| `contracts` | `object` | 宣告擴充套件實作的能力合約 | 見下方 |
| `configContracts` | `object` | 配置合約（遷移路徑等） | 見下方 |
| `configSchema` | `object` | JSON Schema 格式的配置定義 | 見下方 |
| `skills` | `string[]` | 擴充套件捆綁的技能路徑 | `["./skills"]` |
| `commandAliases` | `array` | 斜線指令別名 | 見下方 |
| `uiHints` | `object` | Control UI 的顯示提示 | 見下方 |

> **來源**：從 `source-repo/extensions/*/openclaw.plugin.json` 歸納

### 2.2 providerAuthChoices 結構

認證選項陣列允許擴充套件定義多種登入方式。每個選項的結構：

```json
{
  "provider": "anthropic",
  "method": "cli",                    // "cli" | "api-key" | "oauth" | "local" | "custom"
  "choiceId": "anthropic-cli",
  "deprecatedChoiceIds": ["claude-cli"],
  "choiceLabel": "Anthropic Claude CLI",
  "choiceHint": "Reuse a local Claude CLI login on this host",
  "assistantPriority": -20,
  "groupId": "anthropic",
  "groupLabel": "Anthropic",
  "groupHint": "Claude CLI + API key",
  "optionKey": "anthropicApiKey",      // 配置鍵
  "cliFlag": "--anthropic-api-key",    // CLI 旗標
  "cliOption": "--anthropic-api-key <key>",
  "cliDescription": "Anthropic API key",
  "onboardingScopes": ["image-generation"]  // 引導範圍
}
```

> **來源**：`source-repo/extensions/anthropic/openclaw.plugin.json:14-37`

### 2.3 contracts 結構

`contracts` 欄位宣告擴充套件實作了哪些能力合約。這是 OpenClaw 的「能力發現」機制——核心不需要硬編碼知道哪個擴充套件能做什麼，而是透過合約宣告來動態發現。

已發現的合約類型：

| 合約名稱 | 說明 | 範例擴充套件 |
|----------|------|------------|
| `speechProviders` | TTS/STT 語音供應商 | elevenlabs、openai、microsoft、minimax、vydra |
| `realtimeTranscriptionProviders` | 即時語音轉文字 | openai |
| `realtimeVoiceProviders` | 即時語音對話 | openai |
| `mediaUnderstandingProviders` | 媒體（圖片/影片）理解 | anthropic、openai、google、groq、mistral、deepgram |
| `imageGenerationProviders` | 圖片生成 | openai、google、fal、comfy、minimax、vydra |
| `videoGenerationProviders` | 影片生成 | runway、fal、alibaba、byteplus、together、xai、vydra |
| `musicGenerationProviders` | 音樂生成 | google、comfy、minimax |
| `webSearchProviders` | 網路搜尋 | tavily、exa、duckduckgo、brave、searxng、perplexity、firecrawl |
| `webFetchProviders` | 網頁抓取 | firecrawl |
| `memoryEmbeddingProviders` | 記憶向量嵌入 | ollama |
| `tools` | 註冊的工具名稱 | `["tavily_search", "tavily_extract"]` |

> **來源**：從所有 `source-repo/extensions/*/openclaw.plugin.json` 的 `contracts` 欄位歸納

### 2.4 configSchema 結構

每個擴充套件可以定義自己的 JSON Schema 配置結構。以 `memory-core` 為例，它定義了複雜的巢狀夢境配置：

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "dreaming": {
      "type": "object",
      "properties": {
        "enabled": { "type": "boolean" },
        "frequency": { "type": "string" },       // Cron 表達式
        "timezone": { "type": "string" },
        "phases": {
          "type": "object",
          "properties": {
            "light": {
              "properties": {
                "enabled": { "type": "boolean" },
                "lookbackDays": { "type": "integer", "minimum": 0 },
                "dedupeSimilarity": { "type": "number", "minimum": 0, "maximum": 1 }
              }
            },
            "rem": { "..." },
            "deep": { "..." }
          }
        }
      }
    }
  }
}
```

> **來源**：`source-repo/extensions/memory-core/openclaw.plugin.json:10-105`

### 2.5 uiHints 結構

`uiHints` 告訴 Control UI 如何為配置欄位生成表單元素：

```json
{
  "webSearch.apiKey": {
    "label": "Tavily API Key",
    "help": "Tavily API key for web search and extraction (fallback: TAVILY_API_KEY env var).",
    "sensitive": true,          // 密碼遮罩
    "placeholder": "tvly-..."
  },
  "webSearch.baseUrl": {
    "label": "Tavily Base URL",
    "help": "Tavily API base URL override."
  }
}
```

> **來源**：`source-repo/extensions/tavily/openclaw.plugin.json:12-23`

### 2.6 commandAliases 結構

擴充套件可以將斜線指令映射到核心 CLI 指令：

```json
{
  "commandAliases": [
    {
      "name": "dreaming",
      "kind": "runtime-slash",
      "cliCommand": "memory"
    }
  ]
}
```

這使得用戶可以在聊天中輸入 `/dreaming` 來觸發記憶相關操作。

> **來源**：`source-repo/extensions/memory-core/openclaw.plugin.json:4-10`

---

## 3. 擴充套件進入點模式

所有擴充套件的 `index.ts` 都遵循兩種模式之一：

### 3.1 一般擴充套件：definePluginEntry()

絕大多數擴充套件使用 `definePluginEntry()` 註冊：

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

`definePluginEntry()` 接受的選項：

| 欄位 | 說明 |
|------|------|
| `id` | 擴充套件 ID |
| `name` | 顯示名稱 |
| `description` | 描述 |
| `kind` | 種類（如 `"memory"`） |
| `register(api)` | 註冊回呼，接收 Plugin API |
| `reload` | 重新載入回呼 |
| `nodeHostCommands` | Node host 指令 |
| `securityAuditCollectors` | 安全審計收集器 |

`register(api)` 中的 `api` 物件是 OpenClaw 核心暴露給擴充套件的介面，允許：
- `api.registerWebSearchProvider()` — 註冊搜尋供應商
- `api.registerTool()` — 註冊工具
- `api.registerMemoryCapability()` — 註冊記憶能力
- `api.on()` — 訂閱事件

> **來源**：`source-repo/extensions/tavily/index.ts:1-15`、`source-repo/extensions/memory-core/index.ts:1-40`、`source-repo/extensions/browser/index.ts:1-15`

### 3.2 通道擴充套件：defineBundledChannelEntry()

通道類擴充套件使用專門的 `defineBundledChannelEntry()`：

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
    api.on("subagent_spawning", async (event) => { ... });
    api.on("subagent_ended", async (event) => { ... });
    api.on("subagent_delivery_target", async (event) => { ... });
  },
});
```

通道進入點的差異在於：
- `importMetaUrl`：用於解析相對路徑
- `plugin`/`runtime`：延遲載入的模組規格（specifier + exportName）
- `registerFull(api)`：完整註冊回呼（含 subagent 事件掛鉤）

> **來源**：`source-repo/extensions/discord/index.ts:1-41`

---

## 4. Plugin SDK：擴充套件與核心的唯一邊界

擴充套件**不直接匯入**核心原始碼（`src/` 目錄）。所有互動都透過 `openclaw/plugin-sdk/*` 命名空間模組。這是一個精心設計的 API 邊界（API boundary），確保核心可以獨立演進而不破壞擴充套件。

### 4.1 已發現的 Plugin SDK 子模組

從所有擴充套件的匯入語句中歸納出以下 SDK 子模組：

| 子模組路徑 | 用途 | 使用者範例 |
|-----------|------|----------|
| `openclaw/plugin-sdk/plugin-entry` | 核心進入點定義 | anthropic、tavily、browser |
| `openclaw/plugin-sdk/channel-entry-contract` | 通道進入點合約 | discord、slack、telegram |
| `openclaw/plugin-sdk/extension-shared` | 擴充套件共用工具 | shared/runtime.ts |
| `openclaw/plugin-sdk/fetch-runtime` | HTTP fetch 封裝 | discord |
| `openclaw/plugin-sdk/retry-runtime` | 重試邏輯 | discord |
| `openclaw/plugin-sdk/cli-runtime` | CLI 工具格式化 | anthropic |
| `openclaw/plugin-sdk/cli-backend` | CLI 後端介面 | anthropic |
| `openclaw/plugin-sdk/provider-auth` | 供應商認證 | anthropic |
| `openclaw/plugin-sdk/provider-model-shared` | 模型定義共用 | anthropic |
| `openclaw/plugin-sdk/provider-model-types` | 模型型別定義 | anthropic |
| `openclaw/plugin-sdk/provider-usage` | 用量追蹤 | anthropic |
| `openclaw/plugin-sdk/text-runtime` | 文字處理工具 | anthropic、memory-wiki |
| `openclaw/plugin-sdk/error-runtime` | 錯誤格式化 | memory-wiki |
| `openclaw/plugin-sdk/temp-path` | 暫存路徑解析 | memory-wiki |
| `openclaw/plugin-sdk/speech-core` | 語音核心介面 | speech-core |
| `openclaw/plugin-sdk/media-understanding` | 媒體理解介面 | anthropic |
| `openclaw/plugin-sdk/memory-core` | 記憶核心介面 | memory-core |
| `openclaw/plugin-sdk/memory-host-core` | 記憶主機核心 | memory-wiki |
| `openclaw/plugin-sdk/memory-host-files` | 記憶檔案存取 | memory-wiki |
| `openclaw/plugin-sdk/memory-host-search` | 記憶搜尋管理器 | memory-wiki |
| `openclaw/plugin-sdk/memory-host-markdown` | 記憶 Markdown 處理 | memory-wiki |
| `openclaw/plugin-sdk/memory-core-host-engine-qmd` | 記憶引擎 QMD | memory-core |
| `openclaw/plugin-sdk/memory-core-host-runtime-files` | 記憶執行時檔案 | memory-core |
| `openclaw/plugin-sdk/memory-core-host-status` | 記憶狀態 | memory-core |
| `openclaw/plugin-sdk/testing` | 測試工具 | anthropic |
| `openclaw/plugin-sdk/zod` | Zod schema 驗證 | memory-wiki |

> **來源**：`source-repo/extensions/` 中所有 `*.ts` 檔案的 `from "openclaw/plugin-sdk/*"` 匯入語句

### 4.2 邊界設計原則

1. **單向依賴**：擴充套件 → SDK → 核心，反向不存在
2. **懶載入**：通道擴充套件使用 `specifier` + `exportName` 進行延遲載入
3. **型別安全**：所有 SDK 模組都有完整的 TypeScript 型別定義
4. **合約驅動**：擴充套件透過 `contracts` 宣告能力，核心透過合約發現擴充套件

---

## 5. 擴充套件啟用機制

從 `openclaw.plugin.json` 中的 `enabledByDefault` 欄位可以看出三種啟用模式：

| 模式 | 行為 | 範例 |
|------|------|------|
| `enabledByDefault: true` | 安裝即啟用，無需配置 | anthropic、openai、google、browser |
| `enabledByDefault` 缺失 | 需要明確啟用或配置 | discord、slack、tavily、elevenlabs |
| `autoEnableWhenConfiguredProviders` | 當指定 provider 被配置時自動啟用 | google（`["google-gemini-cli"]`） |

---

## 引用來源

| 來源路徑 | 引用內容 |
|----------|----------|
| `source-repo/extensions/tsconfig.package-boundary.base.json` | 根層級設定檔 |
| `source-repo/extensions/anthropic/` | Anthropic 擴充套件完整結構 |
| `source-repo/extensions/discord/` | Discord 擴充套件完整結構 |
| `source-repo/extensions/memory-core/` | Memory Core 擴充套件完整結構 |
| `source-repo/extensions/shared/` | 共用模組 |
| `source-repo/extensions/shared/runtime.ts:1` | Plugin SDK 匯出 |
| `source-repo/extensions/anthropic/openclaw.plugin.json` | Manifest 完整範例 |
| `source-repo/extensions/discord/openclaw.plugin.json` | Channel manifest 範例 |
| `source-repo/extensions/memory-core/openclaw.plugin.json:10-105` | 複雜 configSchema 範例 |
| `source-repo/extensions/tavily/openclaw.plugin.json:12-23` | uiHints 範例 |
| `source-repo/extensions/anthropic/index.ts:1-11` | definePluginEntry 範例 |
| `source-repo/extensions/discord/index.ts:1-41` | defineBundledChannelEntry 範例 |
| `source-repo/extensions/tavily/index.ts:1-15` | 搜尋擴充套件註冊 |
| `source-repo/extensions/browser/index.ts:1-15` | 瀏覽器擴充套件註冊 |
| `source-repo/extensions/memory-core/index.ts:1-40` | 記憶擴充套件註冊 |
| `source-repo/extensions/anthropic/register.runtime.ts:1-80` | Anthropic 註冊邏輯 |
| `source-repo/extensions/discord/src/channel.ts:1-40` | Discord API 封裝 |
| `source-repo/extensions/memory-core/src/dreaming-phases.ts:1-50` | 夢境階段邏輯 |
