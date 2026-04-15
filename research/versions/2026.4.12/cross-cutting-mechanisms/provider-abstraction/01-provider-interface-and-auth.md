# Provider 抽象層（一）：Provider 介面定義、認證與模型發現

> **摘要**：本章從 Plugin SDK 中的 Provider 介面出發，解析 OpenClaw 如何透過統一的
> `ProviderPlugin` 型別、多策略認證（Multi-strategy Auth）、以及模型目錄發現
> （Model Catalog Discovery）機制，將 50+ LLM 提供者抽象為相同的推理後端。
> 每個 Provider 都是一個 Plugin，遵循前章介紹的 Plugin 生命週期。

---

## 1. Provider 的角色定位

在 OpenClaw 的架構中，Provider 是連接 Agent 推理迴圈與外部 LLM API 的橋樑。
當 Agent 需要進行推理時，它不直接呼叫 OpenAI API 或 Anthropic API，
而是透過 Provider 抽象層——核心程式碼只知道「我要用一個 Provider 來做推理」。

Provider 作為 Plugin 的一部分，透過 `api.registerProvider(providerPlugin)` 註冊。
一個 Plugin 可以註冊多個 Provider（例如 OpenAI Plugin 同時註冊 `openai` 和 
`openai-codex` 兩個 Provider）。

---

## 2. Provider 介面定義

### 2.1 核心 ProviderPlugin 型別

`ProviderPlugin` 是 Provider 的完整介面（定義在 `src/plugins/types.ts`）：

```typescript
// source-repo/src/plugins/types.ts（ProviderPlugin 概要，實際有 50+ 個 hook）
export type ProviderPlugin = {
  id: string;                    // Provider 唯一識別碼
  pluginId?: string;             // 所屬 Plugin ID
  label: string;                 // 人類可讀名稱
  docsPath?: string;             // 文件路徑
  aliases?: string[];            // 別名
  hookAliases?: string[];        // Hook 別名
  envVars?: string[];            // 認證環境變數
  auth: ProviderAuthMethod[];    // 認證方法陣列

  // 模型目錄
  catalog?: ProviderPluginCatalog;
  discovery?: ProviderPluginDiscovery;

  // 50+ 個 provider-specific hook 函式...
};
```

### 2.2 Provider Entry Helper

對於只提供一個 Provider 的 Plugin，有一個簡化的定義方式：

```typescript
// source-repo/src/plugin-sdk/provider-entry.ts:41-60
export type SingleProviderPluginOptions = {
  id: string;
  name: string;
  description: string;
  kind?: OpenClawPluginDefinition["kind"];
  configSchema?: OpenClawPluginConfigSchema | (() => OpenClawPluginConfigSchema);
  provider?: {
    id?: string;
    label: string;
    docsPath: string;
    aliases?: string[];
    envVars?: string[];
    auth?: SingleProviderPluginApiKeyAuthOptions[];
    catalog: SingleProviderPluginCatalogOptions;
  } & Omit<ProviderPlugin, "id" | "label" | "docsPath" | "aliases" | "envVars" | "auth" | "catalog">;
  register?: (api: OpenClawPluginApi) => void;
};
```

`defineSingleProviderPluginEntry()` 將這些選項轉換為完整的 Plugin 定義
（`source-repo/src/plugin-sdk/provider-entry.ts:100-168`）：

1. 從 auth options 解析 wizard setup（lines 62-85）
2. 從 auth methods 建立環境變數清單（lines 87-98）
3. 建立 API key 認證方法（lines 115-128）
4. 建立 Provider 目錄配置（lines 130-147）
5. 透過 Plugin API 註冊 Provider（lines 149-163）

---

## 3. 認證抽象

### 3.1 Provider Auth 統一介面

Provider 認證的統一入口在 `provider-auth.ts`：

```typescript
// source-repo/src/plugin-sdk/provider-auth.ts:73-88
export function isProviderApiKeyConfigured(params: {
  provider: string;
  agentDir?: string;
}): boolean
```

此模組匯出了認證相關的所有子系統：
- Auth Profile Store 管理（持久化認證憑證）
- API Key 驗證和格式正規化
- OAuth 工具（Token 刷新、憑證儲存）
- Secret 存儲選項

### 3.2 多策略認證

每個 Provider 可以支援多種認證方式。以 manifest 中的 `providerAuthChoices` 為例：

**OpenAI 的認證選項**（`source-repo/extensions/openai/openclaw.plugin.json`）：

```json
{
  "providerAuthChoices": [
    {
      "provider": "openai-codex",
      "method": "oauth",
      "choiceId": "openai-codex",
      "choiceLabel": "OpenAI Codex (ChatGPT OAuth)",
      "groupId": "openai",
      "groupLabel": "OpenAI"
    },
    {
      "provider": "openai",
      "method": "api-key",
      "choiceId": "openai-api-key",
      "choiceLabel": "OpenAI API key",
      "optionKey": "openaiApiKey",
      "cliFlag": "--openai-api-key"
    }
  ]
}
```

**Anthropic 的認證選項**（`source-repo/extensions/anthropic/openclaw.plugin.json`）：

```json
{
  "providerAuthChoices": [
    {
      "provider": "anthropic",
      "method": "cli",
      "choiceId": "anthropic-cli",
      "choiceLabel": "Anthropic Claude CLI",
      "choiceHint": "Reuse a local Claude CLI login on this host",
      "assistantPriority": -20,
      "groupId": "anthropic",
      "groupLabel": "Anthropic"
    },
    {
      "provider": "anthropic",
      "method": "api-key",
      "choiceId": "apiKey",
      "choiceLabel": "Anthropic API key",
      "optionKey": "anthropicApiKey",
      "cliFlag": "--anthropic-api-key"
    }
  ]
}
```

認證方式包括：
- **API Key**：最常見，透過環境變數或配置提供
- **OAuth**：需要用戶授權（如 OpenAI Codex）
- **CLI Token**：重用本地 CLI 工具的登入狀態（如 Anthropic CLI）
- **Device Code**：裝置配對認證

### 3.3 Auth Choice 元資料與優先級

```typescript
// source-repo/src/plugins/provider-auth-choices.ts:7-25
export type ProviderAuthChoiceMetadata = {
  pluginId: string;
  providerId: string;
  methodId: string;
  choiceId: string;
  choiceLabel: string;
  assistantPriority: number;      // 在 UI 中的排序優先級
  assistantVisibility: string;
  deprecatedChoiceIds: string[];  // 已棄用的選項 ID
  groupId: string;                // 群組分類
  groupLabel: string;
  groupHint: string;
  optionKey: string;
  cliFlag: string;
  cliOption: string;
  cliDescription: string;
  onboardingScopes: string[];
};
```

**優先級系統**——不同來源的認證選項有不同的優先級：

```typescript
// source-repo/src/plugins/provider-auth-choices.ts:44-56
PROVIDER_AUTH_CHOICE_ORIGIN_PRIORITY = {
  config: 0,        // 最高——配置中指定的
  bundled: 1,       // 內建 Plugin
  global: 2,        // 全域安裝的 Plugin
  workspace: 3      // 最低——工作區 Plugin
};
```

### 3.4 環境變數認證

Provider 可以在 manifest 中宣告需要的環境變數：

```typescript
// source-repo/src/plugin-sdk/provider-env-vars.ts:1-7
export function getProviderEnvVars(env, provider): Record<string, string>
export function listKnownProviderAuthEnvVarNames(): string[]
export function omitEnvKeysCaseInsensitive(env, keys): NodeJS.ProcessEnv
```

這是「Cheap Metadata」原則的體現——系統不需要載入 Provider 的完整程式碼，
就能透過環境變數判斷認證是否可用。

---

## 4. Model Discovery：模型發現機制

### 4.1 Provider Discovery 流程

模型發現由 `provider-discovery.ts` 負責：

```typescript
// source-repo/src/plugins/provider-discovery.ts:18-26
export async function resolvePluginDiscoveryProviders(params: {
  config?: OpenClawConfig;
  workspaceDir?: string;
  env?: NodeJS.ProcessEnv;
  onlyPluginIds?: string[];
}): Promise<ProviderPlugin[]>
```

### 4.2 Discovery 排序

Provider 可以宣告自己的發現順序（Order）：

```typescript
// source-repo/src/plugins/provider-discovery.ts:29-49
export function groupPluginDiscoveryProvidersByOrder(
  providers: ProviderPlugin[]
): Record<ProviderDiscoveryOrder, ProviderPlugin[]>

// ProviderDiscoveryOrder: "simple" | "profile" | "paired" | "late"
```

四種順序的意義：
- **`simple`**（優先）：只需要 API Key 的 Provider，啟動最快
- **`profile`**：需要 auth profile 的 Provider
- **`paired`**：需要配對的 Provider（如 device code flow）
- **`late`**（最後）：啟動成本高的 Provider

這個排序讓系統能快速完成最常用 Provider 的初始化。

### 4.3 Catalog vs Discovery

Provider 有兩種模型枚舉方式：

```typescript
// source-repo/src/plugins/provider-catalog.ts:25-52
export function buildSingleProviderApiKeyCatalog(params: {
  ctx: ProviderCatalogContext;
  providerId: string;
  buildProvider: () => ModelProviderConfig | Promise<ModelProviderConfig>;
  allowExplicitBaseUrl?: boolean;
}): Promise<ProviderCatalogResult>
```

- **Catalog**（靜態目錄）：Provider 在啟動時提供完整的模型清單
- **Discovery**（動態發現）：Provider 按需查詢可用模型

兩者互斥——Provider validation 會檢查不能同時宣告 catalog 和 discovery
（`source-repo/src/plugins/provider-validation.ts:361-369`）。

### 4.4 Model Support 匹配

Manifest 中的 `modelSupport` 讓系統在不載入 Plugin 的情況下就能匹配模型：

```typescript
// source-repo/src/plugins/manifest.ts:29-40
export type PluginManifestModelSupport = {
  modelPrefixes?: string[];    // 模型 ID 前綴匹配
  modelPatterns?: string[];    // 正則表達式匹配
};
```

例如 OpenAI 的 manifest 宣告 `modelPrefixes: ["gpt-", "o1", "o3", "o4"]`，
Anthropic 宣告 `modelPrefixes: ["claude-"]`。當用戶指定 `gpt-5.4` 時，
系統不需要載入所有 Provider，只需要載入前綴匹配的 OpenAI Plugin。

### 4.5 Model Helpers

動態模型解析的工具函式：

```typescript
// source-repo/src/plugins/provider-model-helpers.ts:6-38
export function matchesExactOrPrefix(
  id: string, values: readonly string[]
): boolean
// 檢查模型 ID 是否精確匹配或前綴匹配

export function cloneFirstTemplateModel(params: {
  providerId, modelId, templateIds, ctx, patch?
}): ProviderRuntimeModel | undefined
// 從模板克隆模型定義，支援可選的覆蓋補丁
```

### 4.6 Provider Model Shared

Provider 被分為「Replay Family」來共享重播和歷史處理邏輯：

```typescript
// source-repo/src/plugin-sdk/provider-model-shared.ts:97-155
type ProviderReplayFamily =
  | "openai-compatible"
  | "anthropic-by-model"
  | "google-gemini"
  | "passthrough-gemini"
  | "hybrid-anthropic-openai";

export function buildProviderReplayFamilyHooks(
  options: ProviderReplayFamily
): ProviderReplayFamilyHooks
```

**Provider Hint 提取**：

```typescript
// source-repo/src/plugin-sdk/provider-model-shared.ts:75-89
export function getModelProviderHint(modelId: string): string | null
// 從 model ID 中提取 provider 前綴（例如 "anthropic/claude-3" → "anthropic"）
```

---

## 5. Provider Validation

### 5.1 載入時驗證

每個 Provider 在註冊時都會經過 `normalizeRegisteredProvider()` 的驗證：

```typescript
// source-repo/src/plugins/provider-validation.ts:319-393
export function normalizeRegisteredProvider(params: {
  pluginId: string;
  source: string;
  provider: ProviderPlugin;
  pushDiagnostic: (diag: PluginDiagnostic) => void;
}): ProviderPlugin | null
```

驗證步驟：
1. 檢查 Provider ID 存在（lines 325-335）
2. 正規化認證方法：
   - 驗證 method ID 唯一（lines 215-225）
   - 正規化 label/hints（lines 237-245）
   - 處理 wizard setup（lines 227-236）
3. 正規化字串欄位：aliases、envVars 等（lines 345-350）
4. **驗證 catalog XOR discovery**——不能同時宣告兩者（lines 361-369）
5. 驗證 wizard setup 與 auth methods 一致（lines 351-358）

---

## 6. Provider Wizard：引導式設定

### 6.1 Wizard Options 建構

Provider Wizard 為使用者提供互動式的 Provider 選擇介面：

```typescript
// source-repo/src/plugins/provider-wizard.ts:123-178
export function resolveProviderWizardOptions(params: {
  config?: OpenClawConfig;
  workspaceDir?: string;
  env?: NodeJS.ProcessEnv;
}): ProviderWizardOption[]

// source-repo/src/plugins/provider-wizard.ts:19-29
export type ProviderWizardOption = {
  value: string;
  label: string;
  hint: string;
  groupId: string;
  groupLabel: string;
  groupHint: string;
  onboardingScopes: string[];
  assistantPriority: number;
  assistantVisibility: string;
};
```

### 6.2 Model Picker

選定 Provider 後，Wizard 會呈現模型選擇器：

```typescript
// source-repo/src/plugins/provider-wizard.ts:194-215
export function resolveProviderModelPickerEntries(params): ProviderModelPickerEntry[]

export type ProviderModelPickerEntry = {
  value: string;
  label: string;
  hint: string;
};
```

### 6.3 Model Selected Hook

當使用者選定模型後，會觸發 Provider 的 hook：

```typescript
// source-repo/src/plugins/provider-wizard.ts:275-313
export async function runProviderModelSelectedHook(params: {
  config: OpenClawConfig;
  model: string;
  prompter: any;
  agentDir?: string;
  workspaceDir?: string;
  env?: NodeJS.ProcessEnv;
}): Promise<void>
```

Provider 可以在此 hook 中執行額外的配置（例如設定預設溫度、選擇端點等）。

---

## 7. Configured Provider Catalog

### 7.1 讀取已配置的模型

```typescript
// source-repo/src/plugin-sdk/provider-catalog-shared.ts:61-94
export function readConfiguredProviderCatalogEntries(params: {
  config?: OpenClawConfig;
  providerId: string;
  publishedProviderId?: string;
}): ConfiguredProviderCatalogEntry[]

// source-repo/src/plugin-sdk/provider-catalog-shared.ts:20-27
export type ConfiguredProviderCatalogEntry = {
  id: string;
  name: string;
  provider: string;
  contextWindow: number;
  reasoning: boolean;
  input: string;
};
```

### 7.2 Streaming Usage 相容性

不同 Provider 對串流中的 usage 報告支援不同：

```typescript
// source-repo/src/plugin-sdk/provider-catalog-shared.ts:96-142
export function withStreamingUsageCompat(provider): ModelProviderConfig
export function supportsNativeStreamingUsageCompat(params): boolean
export function applyProviderNativeStreamingUsageCompat(params): ModelProviderConfig
```

---

## 引用來源

| 來源檔案 | 行號 | 內容 |
|---------|------|------|
| `source-repo/src/plugin-sdk/provider-entry.ts` | 1-60 | `SingleProviderPluginOptions` |
| `source-repo/src/plugin-sdk/provider-entry.ts` | 100-168 | `defineSingleProviderPluginEntry()` |
| `source-repo/src/plugin-sdk/provider-auth.ts` | 73-88 | `isProviderApiKeyConfigured()` |
| `source-repo/src/plugin-sdk/provider-env-vars.ts` | 1-7 | 環境變數工具 |
| `source-repo/src/plugin-sdk/provider-model-types.ts` | 1-8 | 模型型別定義 |
| `source-repo/src/plugin-sdk/provider-model-shared.ts` | 75-155 | Provider Hint 與 Replay Family |
| `source-repo/src/plugin-sdk/provider-catalog-shared.ts` | 20-142 | Catalog 工具 |
| `source-repo/src/plugins/provider-discovery.ts` | 18-89 | Provider Discovery |
| `source-repo/src/plugins/provider-auth-choices.ts` | 7-56 | Auth Choice 元資料 |
| `source-repo/src/plugins/provider-catalog.ts` | 25-72 | Catalog Builder |
| `source-repo/src/plugins/provider-validation.ts` | 319-393 | Provider 驗證 |
| `source-repo/src/plugins/provider-model-helpers.ts` | 6-38 | 模型匹配工具 |
| `source-repo/src/plugins/provider-wizard.ts` | 19-313 | Wizard UI |
| `source-repo/src/plugins/manifest.ts` | 29-40 | `PluginManifestModelSupport` |
| `source-repo/extensions/openai/openclaw.plugin.json` | — | OpenAI manifest |
| `source-repo/extensions/anthropic/openclaw.plugin.json` | — | Anthropic manifest |
