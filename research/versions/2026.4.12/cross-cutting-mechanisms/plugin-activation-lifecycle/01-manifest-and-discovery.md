# Plugin 啟動生命週期（一）：Manifest 解析與 Plugin 發現

> **摘要**：本章從零開始解析 OpenClaw Plugin 的前半段生命週期——從 `openclaw.plugin.json` 
> manifest 檔案的格式定義與解析驗證，到 Plugin 候選者的發現（Discovery）與啟動規劃
> （Activation Planning）。這些步驟全部發生在 Plugin 程式碼被載入之前，屬於「靜態
> 元資料」階段，是理解 OpenClaw Plugin 系統的基礎。

---

## 1. Manifest：Plugin 的身分證

### 1.1 什麼是 Manifest？

每個 OpenClaw Plugin 的根目錄下都必須有一個 `openclaw.plugin.json` 檔案，這就是
Plugin 的 manifest（清單）。它是一份靜態的 JSON 宣告，描述了 Plugin 的身分、能力、
配置需求、以及啟動條件。

這個檔名被定義為常數：

```typescript
// source-repo/src/plugins/manifest.ts:17
export const PLUGIN_MANIFEST_FILENAME = "openclaw.plugin.json";
```

Manifest 使用 JSON5 格式解析（支援註解），這讓開發者可以在 manifest 中加入說明：

```typescript
// source-repo/src/plugins/manifest.ts:3（引入 JSON5）
import JSON5 from "json5";
```

### 1.2 Manifest 的完整型別定義

`PluginManifest` 型別定義了 manifest 的所有欄位：

```typescript
// source-repo/src/plugins/manifest.ts:134-190
export type PluginManifest = {
  id: string;                          // Plugin 唯一識別碼
  configSchema: Record<string, unknown>; // JSON Schema，定義 Plugin 的配置格式
  enabledByDefault?: boolean;           // 是否預設啟用
  legacyPluginIds?: string[];           // 舊版 Plugin ID，向後相容用
  autoEnableWhenConfiguredProviders?: string[]; // 當這些 provider 存在時自動啟用
  kind?: PluginKind | PluginKind[];     // Plugin 類型：memory 或 context-engine
  channels?: string[];                  // 擁有的 Channel ID
  providers?: string[];                 // 擁有的 Provider ID
  providerDiscoveryEntry?: string;      // 輕量級 Provider 元資料入口
  modelSupport?: PluginManifestModelSupport; // 模型家族匹配
  cliBackends?: string[];               // CLI 推理後端
  commandAliases?: PluginManifestCommandAlias[]; // 命令別名
  providerAuthEnvVars?: Record<string, string[]>; // 環境變數認證查找
  providerAuthAliases?: Record<string, string>;   // Provider 認證別名
  channelEnvVars?: Record<string, string[]>;      // Channel 環境變數
  providerAuthChoices?: PluginManifestProviderAuthChoice[]; // 認證選項
  activation?: PluginManifestActivation; // 啟動條件
  setup?: PluginManifestSetup;           // 設定/引導元資料
  skills?: string[];                     // 擁有的 Skill
  name?: string;                        // 人類可讀名稱
  description?: string;                 // 描述
  version?: string;                     // 版本號
  contracts?: PluginManifestContracts;   // 靜態能力宣告
  configContracts?: PluginManifestConfigContracts; // 配置行為契約
  channelConfigs?: Record<string, PluginManifestChannelConfig>; // Channel 配置 Schema
};
```

這裡有兩個值得特別注意的設計思想：

**思想一：「Cheap Metadata」（廉價元資料）**。manifest 中大量欄位的 JSDoc 註解都包含
「cheap」這個詞——意思是這些資訊可以在不載入 Plugin 程式碼的情況下取得。例如
`providerAuthEnvVars` 讓系統知道某個 Provider 需要哪些環境變數，而不需要實際執行
Plugin 的 `register()` 函式。這是一個重要的效能優化。

**思想二：「Contracts 先於 Runtime」**。`contracts` 欄位（`source-repo/src/plugins/manifest.ts:192-204`）
是一個靜態能力宣告快照：

```typescript
// source-repo/src/plugins/manifest.ts:192-204
export type PluginManifestContracts = {
  memoryEmbeddingProviders?: string[];
  speechProviders?: string[];
  realtimeTranscriptionProviders?: string[];
  realtimeVoiceProviders?: string[];
  mediaUnderstandingProviders?: string[];
  imageGenerationProviders?: string[];
  videoGenerationProviders?: string[];
  musicGenerationProviders?: string[];
  webFetchProviders?: string[];
  webSearchProviders?: string[];
  tools?: string[];
};
```

這讓系統在 Plugin runtime 載入之前就知道它提供哪些能力。

### 1.3 Manifest 的載入與驗證

Manifest 的載入由 `loadPluginManifest()` 函式負責：

```typescript
// source-repo/src/plugins/manifest.ts:606-710（函式簽名）
export function loadPluginManifest(rootDir: string, rejectHardlinks?: boolean): PluginManifest
```

載入流程：
1. 在 `rootDir` 下尋找 `openclaw.plugin.json` 檔案
2. 使用安全的邊界檔案讀取（`openBoundaryFileSync`，防止 symlink/hardlink 攻擊）
3. 用 JSON5 解析（支援註解）
4. 驗證必要欄位（`id`、`configSchema`）
5. 正規化所有字串列表欄位

### 1.4 Plugin Kind（類型）

Plugin 只有兩種特殊類型：

```typescript
// source-repo/src/plugins/plugin-kind.types.ts:1
export type PluginKind = "memory" | "context-engine";
```

大多數 Plugin 不宣告 `kind`，表示它們是通用型 Plugin（提供 Provider、Channel、Tool 
或 Hook 等能力）。`memory` 類型的 Plugin 提供記憶系統，`context-engine` 類型的 
Plugin 提供上下文引擎。

### 1.5 Plugin 的四種來源

每個 Plugin 都有一個來源標記：

```typescript
// source-repo/src/plugins/plugin-origin.types.ts:1
export type PluginOrigin = "bundled" | "global" | "workspace" | "config";
```

- **bundled**：隨 OpenClaw 原始碼一起發佈的 Plugin（在 `extensions/` 目錄下）
- **global**：安裝在全域目錄下的 Plugin
- **workspace**：安裝在工作區目錄下的 Plugin
- **config**：透過配置檔案指定路徑的 Plugin

---

## 2. Activation Descriptor：啟動條件描述

### 2.1 Manifest 中的 Activation 欄位

`PluginManifestActivation` 定義了 Plugin 的啟動條件：

```typescript
// source-repo/src/plugins/manifest.ts:44-58
export type PluginManifestActivation = {
  onProviders?: string[];   // 當這些 Provider 被請求時啟動
  onCommands?: string[];    // 當這些 Command 被調用時啟動
  onChannels?: string[];    // 當這些 Channel 需要時啟動
  onRoutes?: string[];      // 當這些 Route 匹配時啟動
  onCapabilities?: PluginManifestActivationCapability[]; // 能力層級觸發
};

// source-repo/src/plugins/manifest.ts:42
export type PluginManifestActivationCapability = "provider" | "channel" | "tool" | "hook";
```

這是一個「懶啟動」（Lazy Activation）機制——Plugin 不會在系統啟動時全部載入，
而是只在被需要時才啟動。這顯著減少了啟動時間和記憶體使用。

### 2.2 Setup Descriptor：安裝時元資料

`PluginManifestSetup` 描述了 Plugin 在安裝/設定階段需要暴露的資訊：

```typescript
// source-repo/src/plugins/manifest.ts:69-81
export type PluginManifestSetup = {
  providers?: PluginManifestSetupProvider[];  // 設定階段的 Provider
  cliBackends?: string[];                    // 設定階段的 CLI 後端
  configMigrations?: string[];               // 配置遷移 ID
  requiresRuntime?: boolean;                 // 是否需要完整 runtime（預設 false）
};
```

`requiresRuntime: false`（預設值）表示這個 Plugin 的設定流程不需要載入完整的
Plugin 程式碼。系統只需要讀取 manifest 和一個輕量級的 setup entry 就足夠了。

### 2.3 CHANGELOG 中的相關改動

OpenClaw 的 CHANGELOG 記錄了幾個與 Plugin 啟動相關的重要修改：

> Plugins/context engines: preserve `plugins.slots.contextEngine` through normalization
> and keep explicitly selected workspace context-engine plugins enabled, so loader
> diagnostics and plugin activation stop dropping that slot selection. (#64192)
> — `source-repo/CHANGELOG.md:242`

> Plugins/runtime: keep LINE reply directives and browser-backed cleanup/reset flows
> working even when those plugins are disabled while tightening bundled plugin
> activation guards. (#59412)
> — `source-repo/CHANGELOG.md:831`

這些修改顯示 activation 系統持續演進中，特別是在「啟用/停用」的邊界情況處理上。

---

## 3. Discovery：Plugin 發現機制

### 3.1 發現流程概述

Plugin 發現由 `discoverOpenClawPlugins()` 函式負責，定義在：

```typescript
// source-repo/src/plugins/discovery.ts:56-59
export type PluginDiscoveryResult = {
  candidates: PluginCandidate[];
  diagnostics: PluginDiagnostic[];
};
```

### 3.2 Plugin 候選者型別

每個被發現的 Plugin 被表示為一個 `PluginCandidate`：

```typescript
// source-repo/src/plugins/discovery.ts:38-54
export type PluginCandidate = {
  idHint: string;              // 推測的 Plugin ID
  source: string;              // 入口檔案路徑
  setupSource?: string;        // 可選的 setup 入口
  rootDir: string;             // Plugin 根目錄
  origin: PluginOrigin;        // 來源：bundled/global/workspace/config
  format?: PluginFormat;       // 格式：openclaw 或 bundle
  bundleFormat?: PluginBundleFormat; // Bundle 變體：codex/claude/cursor
  workspaceDir?: string;       // 所屬工作區
  packageName?: string;        // npm 套件名
  packageVersion?: string;     // npm 版本
  packageDescription?: string; // npm 描述
  packageDir?: string;         // npm 套件目錄
  packageManifest?: OpenClawPackageManifest; // OpenClaw 套件清單
  bundledManifest?: PluginManifest;     // 內建 manifest
  bundledManifestPath?: string;         // manifest 檔案路徑
};
```

注意 `format` 欄位——除了原生的 `"openclaw"` 格式，還支援 `"bundle"` 格式：

```typescript
// source-repo/src/plugins/manifest-types.ts:10-12
export type PluginFormat = "openclaw" | "bundle";
export type PluginBundleFormat = "codex" | "claude" | "cursor";
```

這表示 OpenClaw 可以載入其他 AI 工具的 Plugin 格式（Codex、Claude、Cursor），
體現了其開放性設計。

### 3.3 發現快取機制

發現結果會被快取，預設快取 1000ms：

```typescript
// source-repo/src/plugins/discovery.ts:61-64
const discoveryCache = new Map<string, { expiresAt: number; result: PluginDiscoveryResult }>();
const DEFAULT_DISCOVERY_CACHE_MS = 1000;
```

可以透過環境變數 `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` 調整（`source-repo/src/plugins/discovery.ts:71`）。
設為 `"0"` 可以完全停用快取，適合開發時使用。

### 3.4 安全檢查

發現過程中會進行多項安全檢查：
- **路徑逃逸檢測**：確保 Plugin 路徑不會逃出允許的目錄
- **World-writable 檢測**：拒絕來自任何人都可寫入的目錄的 Plugin
- **所有權驗證**：確保 Plugin 目錄的所有者是合理的
- **Hardlink 拒絕**：防止透過硬連結注入惡意程式碼
- **忽略目錄**：自動跳過 `node_modules`、`.git`、`dist` 等目錄

```typescript
// source-repo/src/plugins/discovery.ts:25-36
const SCANNED_DIRECTORY_IGNORE_NAMES = new Set([
  ".git", ".hg", ".svn", ".turbo", ".yarn",
  ".yarn-cache", "build", "coverage", "dist", "node_modules",
]);
```

### 3.5 Bundled Plugin 掃描

對於隨 OpenClaw 原始碼發佈的內建 Plugin，有一套專門的掃描機制：

```typescript
// source-repo/src/plugins/bundled-plugin-scan.ts:104-129（函式簽名）
export function resolveBundledPluginScanDir(params): string | null
```

掃描結果會被組裝成 `BundledPluginMetadata`：

```typescript
// source-repo/src/plugins/bundled-plugin-metadata.ts:38-50
export type BundledPluginMetadata = {
  dirName: string;
  idHint: string;
  source: BundledPluginPathPair;         // TypeScript 原始碼位置
  setupSource?: BundledPluginPathPair;    // Setup 入口位置
  publicSurfaceArtifacts?: string[];     // 建置產物入口
  runtimeSidecarArtifacts?: string[];    // Runtime 輔助檔案
  packageName?: string;
  packageVersion?: string;
  packageDescription?: string;
  packageManifest?: OpenClawPackageManifest;
  manifest: PluginManifest;
};
```

掃描流程：
1. 掃描 `extensions/` 目錄（或建置後的 `dist/extensions/`）
2. 讀取每個 Plugin 目錄的 `package.json`
3. 載入 `openclaw.plugin.json` manifest
4. 收集公開介面產物（entry files）
5. 按 package root 快取結果

### 3.6 Bundled Sources 映射

`BundledPluginSource` 建立了 Plugin ID 到本地路徑的映射：

```typescript
// source-repo/src/plugins/bundled-sources.ts:34-71（函式簽名）
export function resolveBundledPluginSources(params): Map<string, BundledPluginSource>
```

---

## 4. Activation Planning：啟動規劃

### 4.1 觸發器型別

啟動規劃器使用「觸發器」（Trigger）來決定哪些 Plugin 需要啟動：

```typescript
// source-repo/src/plugins/activation-planner.ts:9-14
export type PluginActivationPlannerTrigger =
  | { kind: "command"; command: string }
  | { kind: "provider"; provider: string }
  | { kind: "channel"; channel: string }
  | { kind: "route"; route: string }
  | { kind: "capability"; capability: PluginManifestActivationCapability };
```

### 4.2 匹配邏輯

`resolveManifestActivationPluginIds()` 是啟動規劃的核心函式：

```typescript
// source-repo/src/plugins/activation-planner.ts:16-44
export function resolveManifestActivationPluginIds(params: {
  trigger: PluginActivationPlannerTrigger;
  config?: OpenClawConfig;
  workspaceDir?: string;
  env?: NodeJS.ProcessEnv;
  cache?: boolean;
  origin?: PluginOrigin;
  onlyPluginIds?: readonly string[];
}): string[]
```

它的匹配邏輯在 `matchesManifestActivationTrigger()` 中實作：

```typescript
// source-repo/src/plugins/activation-planner.ts:46-64
function matchesManifestActivationTrigger(
  plugin: PluginManifestRecord,
  trigger: PluginActivationPlannerTrigger,
): boolean {
  switch (trigger.kind) {
    case "command":
      return listActivationCommandIds(plugin).includes(normalizeCommandId(trigger.command));
    case "provider":
      return listActivationProviderIds(plugin).includes(normalizeProviderId(trigger.provider));
    case "channel":
      return listActivationChannelIds(plugin).includes(normalizeCommandId(trigger.channel));
    case "route":
      return listActivationRouteIds(plugin).includes(normalizeCommandId(trigger.route));
    case "capability":
      return hasActivationCapability(plugin, trigger.capability);
  }
}
```

匹配來源不僅包含 `activation.*` 欄位，還包含 `providers`、`channels`、`contracts.*`
等其他 manifest 欄位（`source-repo/src/plugins/activation-planner.ts:75-80`），
作為 fallback。

### 4.3 Activation Context

啟動上下文攜帶了所有影響啟動決策的資訊：

```typescript
// source-repo/src/plugins/activation-context.ts（型別定義）
export type PluginActivationInputs = {
  rawConfig?: OpenClawConfig;
  config?: OpenClawConfig;
  normalized: NormalizedPluginsConfig;
  activationSourceConfig?: OpenClawConfig;
  activationSource: PluginActivationConfigSource;
  autoEnabledReasons: Record<string, string[]>;
};
```

`withActivatedPluginIds()` 可以動態地將 Plugin 加入啟動清單。

---

## 5. Plugin Package Contract：外部 Plugin 的契約

### 5.1 契約驗證

外部發佈到 ClawHub 的 Plugin 需要滿足一套相容性契約：

```typescript
// source-repo/packages/plugin-package-contract/src/index.ts
export type ExternalPluginCompatibility = {
  pluginApiRange?: string;           // 例如 "^1.0.0"
  builtWithOpenClawVersion?: string; // 建置時的 OpenClaw 版本
  pluginSdkVersion?: string;        // Plugin SDK 版本
  minGatewayVersion?: string;       // 最低 Gateway 版本
};
```

`validateExternalCodePluginPackageJson()` 函式驗證外部 Plugin 的 package.json 
是否滿足發佈要求。

### 5.2 與 Manifest 的關係

Package Contract 和 Manifest 是兩個不同層面的契約：
- **Manifest**：描述 Plugin 的能力和行為（「我能做什麼」）
- **Package Contract**：描述 Plugin 的相容性（「我能跑在哪個版本上」）

兩者共同確保了 Plugin 生態系統的穩定性。

---

## 引用來源

| 來源檔案 | 行號 | 內容 |
|---------|------|------|
| `source-repo/src/plugins/manifest.ts` | 17 | `PLUGIN_MANIFEST_FILENAME` 定義 |
| `source-repo/src/plugins/manifest.ts` | 42-58 | `PluginManifestActivation` 型別 |
| `source-repo/src/plugins/manifest.ts` | 69-81 | `PluginManifestSetup` 型別 |
| `source-repo/src/plugins/manifest.ts` | 134-190 | `PluginManifest` 完整型別 |
| `source-repo/src/plugins/manifest.ts` | 192-204 | `PluginManifestContracts` 型別 |
| `source-repo/src/plugins/manifest-types.ts` | 10-12 | `PluginFormat` / `PluginBundleFormat` |
| `source-repo/src/plugins/plugin-kind.types.ts` | 1 | `PluginKind` 型別 |
| `source-repo/src/plugins/plugin-origin.types.ts` | 1 | `PluginOrigin` 型別 |
| `source-repo/src/plugins/discovery.ts` | 38-54 | `PluginCandidate` 型別 |
| `source-repo/src/plugins/discovery.ts` | 56-59 | `PluginDiscoveryResult` 型別 |
| `source-repo/src/plugins/discovery.ts` | 61-64 | 發現快取機制 |
| `source-repo/src/plugins/discovery.ts` | 25-36 | 忽略目錄清單 |
| `source-repo/src/plugins/activation-planner.ts` | 9-14 | `PluginActivationPlannerTrigger` |
| `source-repo/src/plugins/activation-planner.ts` | 16-44 | `resolveManifestActivationPluginIds()` |
| `source-repo/src/plugins/activation-planner.ts` | 46-64 | `matchesManifestActivationTrigger()` |
| `source-repo/src/plugins/bundled-plugin-metadata.ts` | 38-50 | `BundledPluginMetadata` 型別 |
| `source-repo/src/plugins/bundled-plugin-scan.ts` | 104-129 | `resolveBundledPluginScanDir()` |
| `source-repo/packages/plugin-package-contract/src/index.ts` | — | 外部 Plugin 相容性契約 |
| `source-repo/CHANGELOG.md` | 242 | Plugin activation slot 保留修復 |
| `source-repo/CHANGELOG.md` | 831 | Bundled plugin activation guards |
