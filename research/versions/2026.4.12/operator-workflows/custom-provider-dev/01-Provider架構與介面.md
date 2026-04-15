# Provider 架構與介面

## 本章摘要

本章從第一性原理介紹 OpenClaw 的 LLM Provider 架構——Provider 如何被定義、註冊、認證與發現。OpenClaw 的 Provider 系統是高度模組化的 Plugin 架構，每個 Provider 透過 `openclaw.plugin.json` 宣告自身的能力與認證方式，在啟動時透過 `api.registerProvider()` 註冊到系統中。

---

## 1. Provider 是什麼？

在 OpenClaw 中，Provider 是 AI 模型的來源。每個 Provider 代表一個 LLM 供應商（如 OpenAI、Anthropic、Ollama）或存取方式。Provider 負責：

1. **認證**：管理 API 金鑰、OAuth Token 等
2. **模型發現**：告訴系統有哪些模型可用
3. **請求處理**：將使用者的 Prompt 轉換為 API 請求
4. **串流回應**：將 API 的串流回應傳回系統

---

## 2. Plugin 清單（openclaw.plugin.json）

### 2.1 基本結構

每個 Provider Extension 都有一個 `openclaw.plugin.json` 清單：

```json
{
  "id": "provider-id",
  "enabledByDefault": true,
  "providers": ["provider-name"],
  "modelSupport": {
    "modelPrefixes": ["prefix-"]
  },
  "providerAuthEnvVars": {},
  "providerAuthChoices": [],
  "contracts": {},
  "configSchema": {}
}
```

### 2.2 OpenAI 範例

```json
{
  "id": "openai",
  "enabledByDefault": true,
  "providers": ["openai", "openai-codex"],
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"]
  },
  "cliBackends": ["codex-cli"],
  "providerAuthEnvVars": {
    "openai": ["OPENAI_API_KEY"]
  },
  "providerAuthChoices": [
    {
      "provider": "openai-codex",
      "method": "oauth",
      "choiceId": "openai-codex",
      "choiceLabel": "OpenAI Codex (ChatGPT OAuth)",
      "choiceHint": "Browser sign-in",
      "groupId": "openai",
      "groupLabel": "OpenAI"
    },
    {
      "provider": "openai",
      "method": "api-key",
      "choiceId": "openai-api-key",
      "choiceLabel": "OpenAI API key",
      "groupId": "openai",
      "groupLabel": "OpenAI",
      "optionKey": "openaiApiKey",
      "cliFlag": "--openai-api-key",
      "cliOption": "--openai-api-key <key>",
      "cliDescription": "OpenAI API key"
    }
  ],
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai", "openai-codex"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["openai"]
  }
}
```
> 引用：`source-repo/extensions/openai/openclaw.plugin.json:1-58`

### 2.3 Anthropic 範例

```json
{
  "id": "anthropic",
  "enabledByDefault": true,
  "providers": ["anthropic"],
  "modelSupport": {
    "modelPrefixes": ["claude-"]
  },
  "cliBackends": ["claude-cli"],
  "providerAuthEnvVars": {
    "anthropic": ["ANTHROPIC_OAUTH_TOKEN", "ANTHROPIC_API_KEY"]
  },
  "providerAuthChoices": [
    {
      "provider": "anthropic",
      "method": "cli",
      "choiceId": "anthropic-cli",
      "choiceLabel": "Anthropic Claude CLI",
      "choiceHint": "Reuse a local Claude CLI login on this host",
      "groupId": "anthropic",
      "groupLabel": "Anthropic"
    },
    {
      "provider": "anthropic",
      "method": "api-key",
      "choiceId": "apiKey",
      "choiceLabel": "Anthropic API key",
      "groupId": "anthropic",
      "groupLabel": "Anthropic",
      "optionKey": "anthropicApiKey",
      "cliFlag": "--anthropic-api-key"
    }
  ],
  "contracts": {
    "mediaUnderstandingProviders": ["anthropic"]
  }
}
```
> 引用：`source-repo/extensions/anthropic/openclaw.plugin.json:1-47`

### 2.4 Ollama 範例

```json
{
  "id": "ollama",
  "enabledByDefault": true,
  "providers": ["ollama"],
  "providerDiscoveryEntry": "./provider-discovery.ts",
  "providerAuthEnvVars": {
    "ollama": ["OLLAMA_API_KEY"]
  },
  "providerAuthChoices": [
    {
      "provider": "ollama",
      "method": "local",
      "choiceId": "ollama",
      "choiceLabel": "Ollama",
      "choiceHint": "Cloud and local open models"
    }
  ],
  "contracts": {
    "memoryEmbeddingProviders": ["ollama"],
    "webSearchProviders": ["ollama"]
  },
  "configSchema": {
    "type": "object",
    "properties": {
      "discovery": {
        "type": "object",
        "properties": {
          "enabled": { "type": "boolean" }
        }
      }
    }
  }
}
```
> 引用：`source-repo/extensions/ollama/openclaw.plugin.json:1-48`

### 2.5 清單欄位說明

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | string | 唯一 Plugin ID |
| `enabledByDefault` | boolean | 是否預設啟用 |
| `providers` | string[] | 提供的 Provider ID 列表 |
| `modelSupport.modelPrefixes` | string[] | 模型名稱前綴，用於自動路由 |
| `providerDiscoveryEntry` | string | 模型自動發現模組路徑 |
| `cliBackends` | string[] | CLI 後端 ID |
| `providerAuthEnvVars` | object | 每個 Provider 的認證環境變數 |
| `providerAuthChoices` | array | 認證選項（用於 Onboarding 精靈）|
| `contracts` | object | 能力契約宣告 |
| `configSchema` | object | Plugin 配置的 JSON Schema |
| `uiHints` | object | UI 顯示提示 |

---

## 3. Provider 認證機制

### 3.1 認證方法類型

```typescript
type ProviderAuthMethod = {
  id: string;
  label: string;
  hint?: string;
  kind: ProviderAuthKind;  // "api_key" | "oauth" | "cli" | "token" | "custom"
  wizard?: ProviderPluginWizardSetup;
  run: (ctx: ProviderAuthContext) => Promise<ProviderAuthResult>;
  runNonInteractive?: (ctx) => Promise<OpenClawConfig | null>;
};
```
> 引用：`source-repo/src/plugins/types.ts:316-333`

### 3.2 認證種類（AuthKind）

| 種類 | 說明 | 範例 |
|------|------|------|
| `api_key` | API 金鑰 | OpenAI、Anthropic |
| `oauth` | OAuth 登入 | OpenAI Codex |
| `cli` | 重用 CLI 登入 | Anthropic Claude CLI |
| `token` | Bearer Token | 自訂 Provider |
| `custom` | 自訂認證 | 複雜的認證流程 |

### 3.3 認證上下文

```typescript
type ProviderAuthContext = {
  config: OpenClawConfig;
  prompter: WizardPrompter;
  isRemote: boolean;
  opts: ProviderAuthOptionBag;
  openUrl: (url: string) => Promise<void>;
};
```
> 引用：`source-repo/src/plugins/types.ts:300+`

### 3.4 Wizard 設定

認證選項在 Onboarding 精靈中的呈現：

```typescript
type ProviderPluginWizardSetup = {
  choiceId: string;      // 選項 ID
  choiceLabel: string;   // 顯示標籤
  groupId: string;       // 群組 ID
  groupLabel: string;    // 群組標籤
  methodId: string;      // 認證方法 ID
};
```
> 引用：`source-repo/src/plugins/types.ts:949+`

---

## 4. Model Discovery（模型發現）

### 4.1 Model Prefixes

`modelPrefixes` 讓系統自動將模型名稱路由到正確的 Provider：

| Provider | Prefixes | 匹配範例 |
|----------|----------|---------|
| OpenAI | `gpt-`, `o1`, `o3`, `o4` | `gpt-4o`, `o3-mini` |
| Anthropic | `claude-` | `claude-3.5-sonnet` |

> 引用：`source-repo/extensions/openai/openclaw.plugin.json:6`, `source-repo/extensions/anthropic/openclaw.plugin.json:6`

### 4.2 Provider Discovery

Ollama 使用 `providerDiscoveryEntry` 實作動態模型發現：

```json
"providerDiscoveryEntry": "./provider-discovery.ts"
```
> 引用：`source-repo/extensions/ollama/openclaw.plugin.json:5`

Discovery 模組會自動探測本地或遠端的 Ollama 實例，列出可用的模型。

```typescript
// extensions/ollama/provider-discovery.ts
// ProviderDiscoveryContext 型別
// 自動發現 ambient Ollama 實例
```
> 引用：`source-repo/extensions/ollama/provider-discovery.ts:1-80+`

---

## 5. Provider 配置

### 5.1 Plugin 配置類型

```typescript
// src/config/types.plugins.ts

type PluginEntryConfig = {
  enabled?: boolean;
  hooks?: { allowPromptInjection?: boolean };
  subagent?: { allowModelOverride?: boolean; allowedModels?: string[] };
  config?: Record<string, unknown>;
};

type PluginSlotsConfig = {
  memory?: string;        // 記憶系統 Plugin
  contextEngine?: string; // 上下文引擎 Plugin
};

type PluginsConfig = {
  enabled?: boolean;
  allow?: string[];       // 白名單
  deny?: string[];        // 黑名單
  load?: { paths?: string[] };  // 額外 Plugin 路徑
  slots?: PluginSlotsConfig;
  entries?: Record<string, PluginEntryConfig>;
  installs?: Record<string, PluginInstallRecord>;
};
```
> 引用：`source-repo/src/config/types.plugins.ts:1-49`

### 5.2 在 openclaw.json 中配置

```json
{
  "plugins": {
    "entries": {
      "ollama": {
        "enabled": true,
        "config": {
          "discovery": { "enabled": true }
        }
      }
    }
  }
}
```

---

## 6. Contracts（能力契約）

`contracts` 欄位宣告 Provider 除了 LLM 之外還支援哪些能力：

| 契約 | 說明 |
|------|------|
| `speechProviders` | 語音合成（TTS）|
| `realtimeTranscriptionProviders` | 即時語音轉文字 |
| `realtimeVoiceProviders` | 即時語音對話 |
| `mediaUnderstandingProviders` | 圖片/影片理解 |
| `imageGenerationProviders` | 圖片生成 |
| `videoGenerationProviders` | 影片生成 |
| `memoryEmbeddingProviders` | 記憶嵌入向量 |
| `webSearchProviders` | 網頁搜尋 |

> 引用：`source-repo/extensions/openai/openclaw.plugin.json:38-45`, `source-repo/extensions/ollama/openclaw.plugin.json:21-24`

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/extensions/openai/openclaw.plugin.json:1-58` | OpenAI Plugin 清單 |
| `source-repo/extensions/anthropic/openclaw.plugin.json:1-47` | Anthropic Plugin 清單 |
| `source-repo/extensions/ollama/openclaw.plugin.json:1-48` | Ollama Plugin 清單 |
| `source-repo/src/plugins/types.ts:300-333, 949+` | Provider 認證類型 |
| `source-repo/src/config/types.plugins.ts:1-49` | Plugin 配置類型 |
| `source-repo/extensions/ollama/provider-discovery.ts:1-80+` | Ollama 模型發現 |
