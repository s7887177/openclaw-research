# Provider 抽象層（二）：Runtime 執行、串流處理與工具正規化

> **摘要**：本章深入 Provider 的 Runtime 面向——Provider 如何在推理迴圈中被呼叫、
> 如何處理串流回應（Streaming）、如何正規化不同 Provider 的工具呼叫格式（Tool Schema 
> Normalization）、以及 Web Search Provider 的專門抽象。最後以 OpenAI 和 Anthropic 
> 兩個具體實作為案例，展示抽象層如何映射到真實的 LLM API。

---

## 1. Provider HTTP 通訊層

### 1.1 共享 HTTP 工具

所有 Provider 共享一套 HTTP 通訊基礎設施：

```typescript
// source-repo/src/plugin-sdk/provider-http.ts:1-37
// 匯出的函式：
// - fetchWithTimeout(url, options)       — 帶超時的 fetch
// - fetchWithTimeoutGuarded(url, options) — 帶超時和錯誤處理的 fetch
// - normalizeBaseUrl(url)                — 正規化 API 端點 URL
// - postJsonRequest(url, body, options)  — JSON POST 請求
// - postTranscriptionRequest(...)        — 語音轉寫 POST 請求
// - assertOkOrThrowHttpError(response)   — HTTP 錯誤處理
// - resolveProviderHttpRequestConfig()   — 解析 HTTP 請求配置

// 匯出的型別：
// - ProviderAttributionPolicy     — 歸因策略
// - ProviderRequestCapabilities   — 請求能力
// - ProviderEndpointClass         — 端點類別
// - ProviderEndpointResolution    — 端點解析
// - ProviderRequestAuthOverride   — 認證覆蓋
// - ProviderRequestProxyOverride  — 代理覆蓋
// - ProviderRequestTlsOverride    — TLS 覆蓋
```

**設計考量**：HTTP 層抽象了認證注入、代理配置、TLS 設定等基礎設施，
讓各 Provider 不需要重複實作這些通用邏輯。

---

## 2. Provider Tools：工具 Schema 正規化

### 2.1 問題：不同 Provider 的 Tool Schema 不相容

OpenAI、Anthropic、Google Gemini 對 JSON Schema 的支援各不相同。
例如 Gemini 不支援 `minLength`、`maxLength` 等關鍵字，OpenAI 要求 strict 模式。
如果直接將相同的 Tool Schema 發送給不同 Provider，會導致錯誤。

### 2.2 Schema 剝離

`provider-tools.ts` 提供了遞迴的 Schema 處理工具：

```typescript
// source-repo/src/plugin-sdk/provider-tools.ts:28-72
export function stripUnsupportedSchemaKeywords(
  schema: unknown,
  unsupportedKeywords: Set<string>
): unknown
// 遞迴移除不支援的 JSON Schema 關鍵字

export function stripXaiUnsupportedKeywords(schema: unknown): unknown
// 專為 X.AI 移除不支援的關鍵字
```

### 2.3 Schema 檢查

```typescript
// source-repo/src/plugin-sdk/provider-tools.ts:90-130
export function findUnsupportedSchemaKeywords(
  schema: unknown,
  path: string,
  keywords: Set<string>
): string[]
// 尋找 Schema 中不支援的關鍵字，返回路徑清單
```

### 2.4 Provider-specific 正規化

針對不同的 Provider「家族」有專門的正規化器：

```typescript
// source-repo/src/plugin-sdk/provider-tools.ts:132-183
export function normalizeGeminiToolSchemas(ctx): AnyAgentTool[]
export function normalizeOpenAIToolSchemas(ctx): AnyAgentTool[]
```

### 2.5 相容性家族

Provider 被分為「工具相容性家族」（Tool Compat Family）：

```typescript
// source-repo/src/plugin-sdk/provider-tools.ts:430-447
export function buildProviderToolCompatFamilyHooks(
  family: ProviderToolCompatFamily
): {
  normalizeToolSchemas: (ctx) => AnyAgentTool[];
  inspectToolSchemas: (ctx) => void;
}
// family: "gemini" | "openai"
```

這個函式返回兩個 hook——`normalizeToolSchemas` 在工具 Schema 發送給 LLM 前
進行正規化，`inspectToolSchemas` 在開發模式下記錄 Schema 相容性問題。

---

## 3. Provider Streaming：串流處理

### 3.1 Stream Wrapper Factory

不同 Provider 的串流回應格式各異。OpenClaw 使用「Stream Wrapper」模式來統一處理：

```typescript
// source-repo/src/plugin-sdk/provider-stream-shared.ts:4-8
export type ProviderStreamWrapperFactory =
  (baseStreamFn: StreamFn | undefined) => StreamFn | undefined;
```

### 3.2 Stream Wrapper 組合

多個 Wrapper 可以鏈式組合：

```typescript
// source-repo/src/plugin-sdk/provider-stream-shared.ts:10-18
export function composeProviderStreamWrappers(
  baseStreamFn: StreamFn,
  ...wrappers: ProviderStreamWrapperFactory[]
): StreamFn
```

這遵循裝飾器模式（Decorator Pattern）——每個 Wrapper 包裝前一個的輸出。

### 3.3 Provider-specific Stream Wrappers

不同 Provider 需要不同的串流處理：

```typescript
// source-repo/src/plugin-sdk/provider-stream-shared.ts（匯出概要）

// HTML Entity 解碼（某些 Provider 如 X.AI 會 HTML encode 工具參數）
export function createHtmlEntityToolCallArgumentDecodingWrapper(
  baseStreamFn: StreamFn
): StreamFn

// Anthropic payload policy
export function applyAnthropicPayloadPolicyToParams(...): ...

// Amazon Bedrock 快取控制
export function createBedrockNoCacheWrapper(...): StreamFn

// Moonshot thinking block 處理
export function createMoonshotThinkingWrapper(...): StreamFn

// ZAI 工具串流修正
export function createZaiToolStreamWrapper(...): StreamFn
```

### 3.4 HTML Entity 特殊處理

某些 Provider（如 X.AI）會將工具呼叫的參數進行 HTML entity 編碼，
這需要在串流接收端進行解碼：

```typescript
// source-repo/src/plugin-sdk/provider-stream-shared.ts:20-121
export function decodeHtmlEntitiesInObject(value: unknown): unknown
// 遞迴解碼物件中所有字串的 HTML entities
```

### 3.5 Provider Stream Family

```typescript
// source-repo/src/plugin-sdk/provider-stream-family.ts
// 根據 Provider 家族選擇串流處理策略
```

---

## 4. Provider Usage：使用量追蹤

### 4.1 Usage Snapshot

每個 Provider 有不同的 API 來查詢使用量/配額：

```typescript
// source-repo/src/plugin-sdk/provider-usage.ts:1-26
// 型別：
export type ProviderUsageSnapshot = { ... };
export type UsageProviderId = string;
export type UsageWindow = string;

// Provider-specific fetch 函式：
export function fetchClaudeUsage(...): Promise<ProviderUsageSnapshot>
export function fetchCodexUsage(...): Promise<ProviderUsageSnapshot>
export function fetchGeminiUsage(...): Promise<ProviderUsageSnapshot>
export function fetchMinimaxUsage(...): Promise<ProviderUsageSnapshot>
export function fetchZaiUsage(...): Promise<ProviderUsageSnapshot>

// 工具函式：
export function clampPercent(value: number): number
export function buildUsageErrorSnapshot(...): ProviderUsageSnapshot
export function buildUsageHttpErrorSnapshot(...): ProviderUsageSnapshot
```

注意這裡出現了具體 Provider 的名稱——這是因為每個 Provider 的 usage API 
格式完全不同，無法進一步抽象。

---

## 5. Provider Onboard：引導式配置

### 5.1 配置應用函式

Provider 的引導流程涉及將認證和模型選擇寫入配置：

```typescript
// source-repo/src/plugin-sdk/provider-onboard.ts:179-200
export function applyOnboardAuthAgentModelsAndProviders(
  cfg: OpenClawConfig,
  params: { auth, agentModels, providerConfig }
): OpenClawConfig
// 合併認證、Agent 模型和 Provider 配置

// source-repo/src/plugin-sdk/provider-onboard.ts:202-220
export function applyAgentDefaultModelPrimary(
  cfg: OpenClawConfig,
  primary: string
): OpenClawConfig
// 設定預設模型

// source-repo/src/plugin-sdk/provider-onboard.ts:379-409
export function applyProviderConfigWithModelCatalog(
  cfg: OpenClawConfig,
  params: { providerId, catalog, ... }
): OpenClawConfig
// 使用模型目錄配置 Provider
```

### 5.2 Model Alias

Agent 可以使用模型別名（如 `fast`、`smart`），映射到實際模型：

```typescript
// source-repo/src/plugin-sdk/provider-onboard.ts:164-177
export function withAgentModelAliases(
  existing: AgentModelAliasEntry[],
  aliases: AgentModelAliasEntry[]
): AgentModelAliasEntry[]
```

### 5.3 Model Allowlist

為安全考量，模型可以被限制在允許清單中：

```typescript
// source-repo/src/plugin-sdk/provider-onboard.ts:451-487
export function ensureModelAllowlistEntry(params: {
  cfg: OpenClawConfig;
  modelId: string;
  providerId: string;
}): OpenClawConfig
```

---

## 6. Web Search Provider 抽象

### 6.1 Web Search Contract

OpenClaw 也將 Web Search 功能抽象為 Provider 介面：

```typescript
// source-repo/src/plugin-sdk/provider-web-search-contract.ts:42-65
export function createWebSearchProviderContractFields(
  options: CreateWebSearchProviderSelectionOptions
): Pick<WebSearchProviderPlugin,
  'inactiveSecretPaths' | 'getCredentialValue' | 'setCredentialValue'
> & Partial<Pick<WebSearchProviderPlugin, 'applySelectionConfig' | ...>>
```

此函式建立統一的憑證管理介面，讓不同的 Web Search Provider（Brave、DuckDuckGo、
Exa、Perplexity、SearXNG、Tavily 等）共享相同的認證儲存模式。

### 6.2 Web Fetch Contract

類似地，Web Fetch 也有抽象契約：

```typescript
// source-repo/src/plugin-sdk/provider-web-fetch-contract.ts:1-6
export type WebFetchProviderPlugin = { ... };
export { enablePluginInConfig };
```

---

## 7. 具體 Provider 實作：OpenAI

### 7.1 Manifest

```json
// source-repo/extensions/openai/openclaw.plugin.json
{
  "id": "openai",
  "enabledByDefault": true,
  "providers": ["openai", "openai-codex"],
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"]
  },
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["openai"]
  }
}
```

注意 OpenAI Plugin 同時註冊了 `openai` 和 `openai-codex` 兩個 Provider，
並且宣告了語音、圖片、影片等多種 contracts。

### 7.2 Registration

```typescript
// source-repo/extensions/openai/index.ts:20-50（概要）
export default definePluginEntry({
  id: "openai",
  name: "OpenAI Provider",
  description: "Bundled OpenAI provider plugins",
  register(api) {
    const openAIToolCompatHooks = buildProviderToolCompatFamilyHooks("openai");
    const buildProviderWithPromptContribution = (provider) => ({
      ...provider,
      ...openAIToolCompatHooks,  // Tool Schema 正規化
      resolveSystemPromptContribution: (ctx) =>
        resolveOpenAISystemPromptContribution(...)
    });

    api.registerProvider(buildProviderWithPromptContribution(buildOpenAIProvider()));
    api.registerProvider(buildProviderWithPromptContribution(buildOpenAICodexProviderPlugin()));
    api.registerImageGenerationProvider(buildOpenAIImageGenerationProvider());
    api.registerRealtimeTranscriptionProvider(...);
    api.registerSpeechProvider(...);
    api.registerMediaUnderstandingProvider(...);
    api.registerVideoGenerationProvider(...);
  }
});
```

**關鍵觀察**：
1. `buildProviderToolCompatFamilyHooks("openai")` 取得 OpenAI 家族的工具正規化 hook
2. `resolveSystemPromptContribution` 讓 Provider 能向 System Prompt 注入內容
3. 一個 Plugin 註冊了 7 種不同類型的能力

---

## 8. 具體 Provider 實作：Anthropic

### 8.1 Manifest

```json
// source-repo/extensions/anthropic/openclaw.plugin.json
{
  "id": "anthropic",
  "enabledByDefault": true,
  "providers": ["anthropic"],
  "modelSupport": {
    "modelPrefixes": ["claude-"]
  },
  "contracts": {
    "mediaUnderstandingProviders": ["anthropic"]
  }
}
```

### 8.2 Registration

```typescript
// source-repo/extensions/anthropic/index.ts:4-11
export default definePluginEntry({
  id: "anthropic",
  name: "Anthropic Provider",
  description: "Bundled Anthropic provider plugin",
  register(api) {
    return registerAnthropicPlugin(api);
  }
});
```

Anthropic 的註冊被封裝在 `registerAnthropicPlugin()` 中，暗示其內部邏輯
較為複雜（可能涉及 Claude CLI token 重用、Vertex AI 整合等）。

---

## 9. 架構設計原則總結

### 9.1 Provider Independence（Provider 獨立性）

每個 Provider Plugin 是完全自包含的——可以獨立停用或替換，
不影響其他 Provider 或核心系統。

### 9.2 Hook-Based Extensibility（Hook 驅動的可擴展性）

Provider 不是透過中央的 switch-case 來切換行為，而是透過 hook 
（`normalizeToolSchemas`、`createStreamFn`、`resolveSystemPromptContribution` 等）
來「注入」自己的行為。

### 9.3 Family-Based Compatibility（家族化相容性）

相似的 Provider 被分組為「家族」：
- **openai-compatible**：OpenAI、Azure OpenAI、Together 等
- **anthropic-by-model**：Anthropic、Anthropic Vertex
- **google-gemini**：Google Gemini

家族內的 Provider 共享重播（Replay）、工具正規化、串流處理等邏輯。

### 9.4 Catalog Ordering（目錄排序）

Provider 可以宣告 discovery 順序（`simple` → `profile` → `paired` → `late`），
讓高頻使用的 Provider 優先初始化。

### 9.5 Wizard Integration（引導整合）

Auth choices 透過 manifest 暴露到統一的引導 UI 中，
讓使用者在一個地方完成所有 Provider 的配置。

### 9.6 Auth Flexibility（認證彈性）

每個 Provider 可以支援多種認證策略（API Key、OAuth、CLI Token、Device Code），
系統根據優先級和可用性自動選擇最佳方式。

---

## 引用來源

| 來源檔案 | 行號 | 內容 |
|---------|------|------|
| `source-repo/src/plugin-sdk/provider-http.ts` | 1-37 | HTTP 通訊工具 |
| `source-repo/src/plugin-sdk/provider-tools.ts` | 28-447 | Tool Schema 正規化 |
| `source-repo/src/plugin-sdk/provider-stream-shared.ts` | 4-121 | Stream Wrapper |
| `source-repo/src/plugin-sdk/provider-stream-family.ts` | — | Stream Family |
| `source-repo/src/plugin-sdk/provider-usage.ts` | 1-26 | Usage 追蹤 |
| `source-repo/src/plugin-sdk/provider-onboard.ts` | 164-487 | 引導配置 |
| `source-repo/src/plugin-sdk/provider-web-search-contract.ts` | 42-65 | Web Search 契約 |
| `source-repo/src/plugin-sdk/provider-web-fetch-contract.ts` | 1-6 | Web Fetch 契約 |
| `source-repo/extensions/openai/openclaw.plugin.json` | — | OpenAI manifest |
| `source-repo/extensions/openai/index.ts` | 20-50 | OpenAI 註冊 |
| `source-repo/extensions/anthropic/openclaw.plugin.json` | — | Anthropic manifest |
| `source-repo/extensions/anthropic/index.ts` | 4-11 | Anthropic 註冊 |
| `source-repo/src/plugins/provider-runtime.ts` | — | Provider runtime registry |
