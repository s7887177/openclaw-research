# Plugin 啟動生命週期（二）：載入、註冊、執行與卸載

> **摘要**：本章接續前章的靜態元資料階段，深入 Plugin 的動態生命週期——模組載入
> （Loading）、函式註冊（Registration）、全域啟用（Activation）、Hook 觸發執行
> （Execution）、以及卸載清理（Unloading）。這些步驟涉及 Jiti 動態載入器、
> Registry 資料結構、和 28 種 Plugin Hook 的執行機制。

---

## 1. Loading：Plugin 模組的載入

### 1.1 載入的入口函式

Plugin 載入的核心函式是 `loadOpenClawPlugins()`，定義在 `src/plugins/loader.ts` 中。
這是一個超過 700 行的大型函式，負責編排整個載入流程：

```typescript
// source-repo/src/plugins/loader.ts:105-129
export type PluginLoadOptions = {
  config?: OpenClawConfig;
  activationSourceConfig?: OpenClawConfig;
  autoEnabledReasons?: Readonly<Record<string, string[]>>;
  workspaceDir?: string;
  env?: NodeJS.ProcessEnv;
  logger?: PluginLogger;
  runtimeOptions?: CreatePluginRuntimeOptions;
  pluginSdkResolution?: PluginSdkResolutionPreference;
  cache?: boolean;
  mode?: "full" | "validate";          // full = 載入並執行；validate = 只驗證
  onlyPluginIds?: string[];             // 只載入指定的 Plugin
  includeSetupOnlyChannelPlugins?: boolean;
  preferSetupRuntimeForChannelPlugins?: boolean;
  activate?: boolean;                   // 是否啟用（寫入全域 registry）
  loadModules?: boolean;                // 是否載入模組（可以只驗證 manifest）
  throwOnLoadError?: boolean;           // 載入錯誤是否拋出例外
};
```

### 1.2 載入流程的十個步驟

根據 `source-repo/src/plugins/loader.ts:1100-1855`，完整載入流程如下：

**步驟 1：快取檢查**（~line 1126-1149）
如果已有相同配置的載入結果在快取中，直接返回。

**步驟 2：重入保護**（~line 1151-1154）
使用 flag 防止遞迴載入——某些 Plugin 的 `register()` 函式可能觸發其他 Plugin 的載入。

**步驟 3：狀態清理**（~line 1158-1163）
清除先前註冊的 Plugin，確保新載入不會與舊狀態衝突。

**步驟 4：Runtime Factory 準備**（~line 1168-1197）
懶載入 Plugin Runtime 模組，建立 Plugin 與核心之間的通訊橋樑。

**步驟 5：Plugin 發現**（~line 1262-1267）
呼叫前章介紹的 `discoverOpenClawPlugins()` 取得所有候選者。

**步驟 6：Manifest Registry 建置**（~line 1268-1275）
載入所有候選者的 manifest 並建立查詢索引。

**步驟 7：配置驗證與啟動決策**（~line 1300+）
對每個 Plugin：
- 驗證配置是否符合 `configSchema`
- 根據 activation planner 決定是否啟動
- 檢查 `enabledByDefault`、用戶配置、deny list 等

**步驟 8：模組載入與 register() 執行**（~line 1549+）
使用 Jiti（Just-In-Time TypeScript/JS 載入器）動態載入 Plugin 程式碼，
然後呼叫 Plugin 匯出的 `register()` 函式。

**步驟 9：Registry 快取**（~line 1834-1847）
將完成的 Registry 存入快取。

**步驟 10：全域啟用**（~line 1848-1850）
呼叫 `setActivePluginRegistry(registry)` 將 Registry 安裝為全域有效。

### 1.3 Jiti 載入器

OpenClaw 使用 [Jiti](https://github.com/unjs/jiti) 作為動態模組載入器。
Jiti 能夠在運行時直接載入 TypeScript 檔案，不需要預先編譯：

```typescript
// source-repo/src/plugins/jiti-loader-cache.ts（快取機制）
// Jiti 實例被快取，避免重複初始化
```

載入時會進行邊界檔案驗證（boundary file validation），確保 Plugin 程式碼
不會嘗試存取系統外的檔案。

### 1.4 Setup Registry：輕量級預載

在完整載入之前，系統可以只載入 Plugin 的「Setup Surface」：

```typescript
// source-repo/src/plugins/setup-registry.ts:29-54（型別定義）
type PluginSetupRegistry = {
  providers: SetupProviderEntry[];          // Setup 階段可用的 Provider
  cliBackends: SetupCliBackendEntry[];      // Setup 階段可用的 CLI 後端
  configMigrations: SetupConfigMigrationEntry[]; // 配置遷移
  autoEnableProbes: SetupAutoEnableProbeEntry[]; // 自動啟用探測器
};
```

Setup Registry 使用獨立的 Jiti 實例載入 Plugin 的 `setupEntry`（如果有的話），
提取出 Provider、CLI Backend 等資訊，而不需要載入完整的 Plugin runtime。
結果會被快取（最多 128 個條目）。

Setup Descriptor 函式用於從 manifest 中提取設定時元資料：

```typescript
// source-repo/src/plugins/setup-descriptors.ts:5-11
export function listSetupProviderIds(record): string[]
export function listSetupCliBackendIds(record): string[]
```

---

## 2. Registration：Plugin 的註冊

### 2.1 Plugin Registry 資料結構

Registry 是所有已載入 Plugin 的集中管理處：

```typescript
// source-repo/src/plugins/registry-types.ts（部分）
export type PluginRecord = {
  id: string;
  source: string;
  status: "error" | "disabled" | "enabled";
  error?: string;
  failedAt?: Date;
  failurePhase?: string;
  // ...其他元資料
};
```

`createPluginRegistry()` 建立 Registry 實例（`source-repo/src/plugins/registry.ts`），
接受 `logger`、`runtime`、`coreGatewayHandlers`、`activateGlobalSideEffects` 參數，
並返回包含以下能力的物件：
- `plugins[]` — Plugin 記錄陣列
- `diagnostics[]` — 累積的診斷問題
- 各種能力類型的註冊方法

### 2.2 registerPlugin() 的呼叫

每個 Plugin 的主要入口是一個 `register(api)` 函式。`api` 參數是 `OpenClawPluginApi`，
提供了所有註冊方法：

```typescript
// source-repo/src/plugins/types.ts（OpenClawPluginApi 介面概要）
// api.registerProvider(provider)      — 註冊 LLM Provider
// api.registerHook(hookName, handler) — 註冊 Hook
// api.registerTool(tool)              — 註冊工具
// api.registerChannel(channel)        — 註冊 Channel
// api.registerCliBackend(backend)     — 註冊 CLI 後端
// api.registerSpeechProvider(p)       — 註冊語音 Provider
// api.registerImageGenerationProvider(p) — 註冊圖片生成 Provider
// ...等等
```

### 2.3 Captured Registration：註冊擷取

`CapturedPluginRegistration` 是一個用於測試和內省的機制——它建立一個 no-op 的
API，捕獲所有註冊呼叫而不實際執行它們：

```typescript
// source-repo/src/plugins/captured-registration.ts:31-49
export type CapturedPluginRegistration = {
  api: OpenClawPluginApi;
  providers: ProviderPlugin[];
  agentHarnesses: AgentHarness[];
  cliRegistrars: CapturedPluginCliRegistration[];
  cliBackends: CliBackendPlugin[];
  textTransforms: PluginTextTransformRegistration[];
  speechProviders: SpeechProviderPlugin[];
  realtimeTranscriptionProviders: RealtimeTranscriptionProviderPlugin[];
  realtimeVoiceProviders: RealtimeVoiceProviderPlugin[];
  mediaUnderstandingProviders: MediaUnderstandingProviderPlugin[];
  imageGenerationProviders: ImageGenerationProviderPlugin[];
  videoGenerationProviders: VideoGenerationProviderPlugin[];
  musicGenerationProviders: MusicGenerationProviderPlugin[];
  webFetchProviders: WebFetchProviderPlugin[];
  webSearchProviders: WebSearchProviderPlugin[];
  memoryEmbeddingProviders: MemoryEmbeddingProviderAdapter[];
  tools: AnyAgentTool[];
};
```

從這個型別可以看出 OpenClaw 支援 17 種以上的能力類型。每一種都有專門的
註冊方法和 Runtime 處理邏輯。

---

## 3. Activation：全域啟用

### 3.1 全域 Registry 管理

Plugin 載入完成後，需要「啟用」——也就是將 Registry 安裝為全域有效：

```typescript
// source-repo/src/plugins/runtime.ts:76-89
export function setActivePluginRegistry(
  registry: PluginRegistry,
  cacheKey?: string,
  mode?: string,
  workspaceDir?: string
): void

// source-repo/src/plugins/runtime.ts:91-93
export function getActivePluginRegistry(): PluginRegistry | null
```

啟用時會更新：
- HTTP 路由表（Plugin 註冊的 HTTP endpoint）
- Channel 表面 Registry
- 版本號（用於變更檢測）

### 3.2 Runtime 狀態追蹤

`recordImportedPluginId()` 追蹤動態載入的 Plugin：

```typescript
// source-repo/src/plugins/runtime.ts:42-44
export function recordImportedPluginId(pluginId: string): void
```

Runtime 支援多種子 Agent 模式：`"default"`、`"explicit"`、`"gateway-bindable"`。

---

## 4. Tools 解析：工具的註冊與解析

### 4.1 工具解析流程

Plugin 註冊的工具透過 `resolvePluginTools()` 被解析為可用工具清單：

```typescript
// source-repo/src/plugins/tools.ts:71-100+
export function resolvePluginTools(params): AnyAgentTool[]
```

解析過程：
1. 檢查 Plugin 系統是否啟用
2. 載入 Plugin Registry
3. 按 allowlist 過濾工具
4. 處理可選工具（optional tools）
5. 檢查工具名稱衝突
6. 支援 Gateway 子 Agent 綁定

---

## 5. Execution：Plugin Hook 的執行

### 5.1 28 種生命週期 Hook

OpenClaw 定義了 28 種 Plugin Hook，涵蓋了 Agent 生命週期的每一個階段：

```typescript
// source-repo/src/plugins/hook-types.ts:55-84
export type PluginHookName =
  | "before_model_resolve"        // 模型解析前
  | "before_prompt_build"         // Prompt 建構前
  | "before_agent_start"          // Agent 啟動前
  | "before_agent_reply"          // Agent 回覆前
  | "llm_input"                   // LLM 輸入
  | "llm_output"                  // LLM 輸出
  | "agent_end"                   // Agent 結束
  | "before_compaction"           // 上下文壓縮前
  | "after_compaction"            // 上下文壓縮後
  | "before_reset"                // 重置前
  | "inbound_claim"               // 入站訊息認領
  | "message_received"            // 收到訊息
  | "message_sending"             // 正在發送訊息
  | "message_sent"                // 訊息已發送
  | "before_tool_call"            // 工具調用前
  | "after_tool_call"             // 工具調用後
  | "tool_result_persist"         // 工具結果持久化
  | "before_message_write"        // 訊息寫入前
  | "session_start"               // Session 開始
  | "session_end"                 // Session 結束
  | "subagent_spawning"           // 子 Agent 生成中
  | "subagent_delivery_target"    // 子 Agent 遞送目標
  | "subagent_spawned"            // 子 Agent 已生成
  | "subagent_ended"              // 子 Agent 已結束
  | "gateway_start"               // Gateway 啟動
  | "gateway_stop"                // Gateway 停止
  | "before_dispatch"             // 派發前
  | "reply_dispatch"              // 回覆派發
  | "before_install";             // 安裝前
```

編譯時型別檢查確保常數陣列與型別完全同步：

```typescript
// source-repo/src/plugins/hook-types.ts:118-121
type MissingPluginHookNames = Exclude<PluginHookName, (typeof PLUGIN_HOOK_NAMES)[number]>;
type AssertAllPluginHookNamesListed = MissingPluginHookNames extends never ? true : never;
const assertAllPluginHookNamesListed: AssertAllPluginHookNamesListed = true;
```

### 5.2 Hook 分類

這些 Hook 可以按功能分為幾個群組：

| 群組 | Hook | 用途 |
|------|------|------|
| **Agent 生命週期** | `before_agent_start`, `before_agent_reply`, `agent_end` | 控制 Agent 行為 |
| **模型與 Prompt** | `before_model_resolve`, `before_prompt_build` | 修改模型選擇和 Prompt |
| **LLM 通訊** | `llm_input`, `llm_output` | 攔截 LLM 請求和回應 |
| **工具調用** | `before_tool_call`, `after_tool_call`, `tool_result_persist` | 控制工具執行 |
| **訊息流** | `message_received`, `message_sending`, `message_sent` | 攔截訊息 |
| **Session** | `session_start`, `session_end`, `before_reset` | 管理 Session 生命週期 |
| **壓縮** | `before_compaction`, `after_compaction` | 控制上下文壓縮 |
| **子 Agent** | `subagent_spawning/spawned/ended`, `subagent_delivery_target` | 管理子 Agent |
| **系統** | `gateway_start/stop`, `before_dispatch`, `reply_dispatch`, `before_install` | 系統級事件 |
| **入站** | `inbound_claim` | 訊息認領/路由 |

### 5.3 Hook 的執行機制

Hook Runner（`source-repo/src/plugins/hooks.ts`）負責實際執行 Hook。
它支援兩種執行模式：

**修改式 Hook（Modifying Hook）**：可以修改事件參數，支援短路（Short-circuit）：

```typescript
// source-repo/src/plugins/hooks.ts:774-811（before_tool_call 為例）
// mergeResults 合併多個 Plugin 的結果
// shouldStop 判斷是否短路（例如 block=true 會立即停止後續 Plugin）
```

**通知式 Hook（Void Hook）**：只通知，不修改：

```typescript
// source-repo/src/plugins/hooks.ts:814-822
async function runAfterToolCall(event, ctx): Promise<void> {
  return runVoidHook("after_tool_call", event, ctx);
}
```

### 5.4 Prompt Injection 防護

有些 Hook 具有特殊的安全考量——`before_prompt_build` 和 `before_agent_start` 
被標記為「Prompt Injection Hook」：

```typescript
// source-repo/src/plugins/hook-types.ts:128-131
export const PROMPT_INJECTION_HOOK_NAMES = [
  "before_prompt_build",
  "before_agent_start",
] as const satisfies readonly PluginHookName[];
```

這意味著系統會對這兩個 Hook 的結果進行額外的安全檢查。

---

## 6. Enable/Disable：啟用與停用

### 6.1 啟用邏輯

Plugin 的啟用/停用由 `enablePluginInConfig()` 管理：

```typescript
// source-repo/src/plugins/enable.ts:12-24
export type PluginEnableResult = {
  config: OpenClawConfig;
  enabled: boolean;
  reason?: string;
};
```

啟用流程：
1. 檢查 Plugin 系統是否全域停用 → 如果是，返回 `disabled`
2. 檢查 Plugin 是否在 deny list 中 → 如果是，返回 `disabled`
3. 在配置中設置 Plugin 為啟用
4. 加入 allowlist

### 6.2 配置策略

`config-policy.ts` 定義了配置合併的策略（`source-repo/src/plugins/config-policy.ts`），
包括哪些配置優先、哪些可以被覆蓋等。

---

## 7. Unloading：Plugin 的卸載與清理

### 7.1 卸載動作

Plugin 卸載定義了 7 個可選動作：

```typescript
// source-repo/src/plugins/uninstall.ts:9-17
export type UninstallActions = {
  entry: boolean;          // 從 plugins.entries 移除
  install: boolean;        // 從 plugins.installs 移除
  allowlist: boolean;      // 從 plugins.allow 移除
  loadPath: boolean;       // 從 plugins.load.paths 移除
  memorySlot: boolean;     // 清除記憶體 slot
  channelConfig: boolean;  // 移除擁有的 Channel 配置
  directory: boolean;      // 刪除 Plugin 目錄
};
```

### 7.2 卸載函式

```typescript
// source-repo/src/plugins/uninstall.ts:92-100+
export function removePluginFromConfig(cfg, pluginId, opts?): OpenClawConfig

// source-repo/src/plugins/uninstall.ts:29-61
export function resolveUninstallDirectoryTarget(params): string | null
```

卸載流程：
1. `removePluginFromConfig()` — 從配置中移除 Plugin 相關條目
2. `clearPluginLoaderCache()` — 清除載入快取
3. `clearAgentHarnesses()` — 取消註冊 Runtime 物件
4. 如果 `directory: true`，刪除 Plugin 目錄

---

## 8. 完整生命週期流程圖

```
┌─── STATIC PHASE（靜態階段）──────────────────────────────────────┐
│                                                                    │
│  1. MANIFEST PARSING                                              │
│     loadPluginManifest(rootDir)                                   │
│     ├─ 讀取 openclaw.plugin.json                                  │
│     ├─ JSON5 解析（支援註解）                                       │
│     └─ 驗證：id、configSchema 必填                                 │
│                                                                    │
│  2. DISCOVERY                                                     │
│     discoverOpenClawPlugins()                                     │
│     ├─ 掃描 bundled / workspace / config / global 來源            │
│     ├─ 安全檢查（路徑逃逸、world-writable、hardlink）              │
│     └─ 快取結果（1000ms 預設）                                     │
│                                                                    │
│  3. ACTIVATION PLANNING                                           │
│     resolveManifestActivationPluginIds(trigger)                   │
│     ├─ 比對 trigger 與 manifest 的 activation 欄位                 │
│     ├─ Fallback 到 contracts.*                                    │
│     └─ 返回需要啟動的 Plugin ID 清單                               │
│                                                                    │
└──── DYNAMIC PHASE（動態階段）─────────────────────────────────────┘
│                                                                    │
│  4. SETUP（可選）                                                 │
│     PluginSetupRegistry                                           │
│     ├─ 載入 setupEntry（輕量級）                                   │
│     └─ 提取 providers / cliBackends / configMigrations            │
│                                                                    │
│  5. MODULE LOADING                                                │
│     loadOpenClawPlugins(options)                                   │
│     ├─ 快取檢查 → 重入保護 → 狀態清理                              │
│     ├─ 發現候選者 → 載入 manifest → 驗證配置                       │
│     └─ Jiti 動態載入 Plugin TypeScript/JS 程式碼                   │
│                                                                    │
│  6. REGISTRATION                                                  │
│     plugin.register(api)                                          │
│     ├─ api.registerProvider()                                     │
│     ├─ api.registerHook()                                         │
│     ├─ api.registerTool()                                         │
│     ├─ api.registerChannel()                                      │
│     └─ ...17+ 種能力類型                                          │
│                                                                    │
│  7. GLOBAL ACTIVATION                                             │
│     setActivePluginRegistry(registry)                             │
│     ├─ 更新 HTTP 路由表                                           │
│     ├─ 更新 Channel 表面 Registry                                 │
│     └─ 遞增版本號                                                  │
│                                                                    │
│  8. EXECUTION                                                     │
│     28 種 Hook 在 Agent 生命週期中觸發                             │
│     ├─ 修改式 Hook（可短路、可修改參數）                            │
│     └─ 通知式 Hook（fire-and-forget）                             │
│                                                                    │
│  9. UNLOADING                                                     │
│     removePluginFromConfig() + clearPluginLoaderCache()           │
│     ├─ 從配置移除                                                  │
│     ├─ 清除快取                                                    │
│     └─ 可選：刪除目錄                                              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 9. 跨元件的協作模式

### 9.1 Manifest 驅動 vs Runtime 驅動

OpenClaw Plugin 系統的核心設計原則是「能在 manifest 解決的，就不要等到 runtime」。
這體現在：

- `providerAuthEnvVars`：在 manifest 中宣告環境變數需求，不需載入程式碼就能判斷認證狀態
- `modelSupport.modelPrefixes`：在 manifest 中宣告模型匹配規則，不需載入就能路由模型請求
- `contracts`：在 manifest 中宣告能力，不需載入就能查詢 Plugin 功能

### 9.2 懶啟動模式

Plugin 不是在系統啟動時全部載入的。Activation Planner 根據當前的觸發條件
（正在使用的 Provider、Channel、Command）動態決定需要啟動哪些 Plugin。
這讓系統啟動時間與安裝的 Plugin 數量無關。

### 9.3 安全分層

安全檢查分布在生命週期的每個階段：
- **發現階段**：路徑逃逸、world-writable、hardlink 檢查
- **載入階段**：邊界檔案驗證、Jiti sandbox
- **註冊階段**：配置 schema 驗證、dependency deny list
- **執行階段**：Prompt Injection Hook 標記、Hook 短路機制

---

## 引用來源

| 來源檔案 | 行號 | 內容 |
|---------|------|------|
| `source-repo/src/plugins/loader.ts` | 105-129 | `PluginLoadOptions` 型別定義 |
| `source-repo/src/plugins/loader.ts` | 1100-1855 | `loadOpenClawPlugins()` 完整流程 |
| `source-repo/src/plugins/registry.ts` | — | `createPluginRegistry()` |
| `source-repo/src/plugins/registry-types.ts` | — | `PluginRecord` 型別定義 |
| `source-repo/src/plugins/captured-registration.ts` | 31-49 | `CapturedPluginRegistration` 型別 |
| `source-repo/src/plugins/runtime.ts` | 42-44, 76-93 | 全域 Registry 管理函式 |
| `source-repo/src/plugins/setup-registry.ts` | 29-54 | `PluginSetupRegistry` 型別 |
| `source-repo/src/plugins/setup-descriptors.ts` | 5-11 | Setup descriptor 函式 |
| `source-repo/src/plugins/tools.ts` | 71-100+ | `resolvePluginTools()` |
| `source-repo/src/plugins/hooks.ts` | 774-822 | Hook 執行函式 |
| `source-repo/src/plugins/hook-types.ts` | 55-84 | 28 種 `PluginHookName` |
| `source-repo/src/plugins/hook-types.ts` | 86-121 | Hook 名稱常數與型別檢查 |
| `source-repo/src/plugins/hook-types.ts` | 128-131 | `PROMPT_INJECTION_HOOK_NAMES` |
| `source-repo/src/plugins/enable.ts` | 12-24 | `PluginEnableResult` |
| `source-repo/src/plugins/uninstall.ts` | 9-17 | `UninstallActions` 型別 |
| `source-repo/src/plugins/uninstall.ts` | 29-100+ | 卸載函式 |
| `source-repo/src/plugins/config-contracts.ts` | 37-98 | Config contract 匹配 |
