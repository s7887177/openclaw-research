# 02 — tsdown 打包器設定詳解（tsdown Build Configuration）

> **層級**：Repository Evidence · Topic 4 · Chapter 02  
> **版本快照**：2026.4.12  
> **最後更新**：2026-07-21

---

## 本章摘要

OpenClaw 使用 `tsdown` 作為其核心打包器（Bundler）。tsdown 是基於 Rolldown 引擎的 TypeScript-first 打包工具，其底層使用 Rust 實作，兼具高效能與 TypeScript 原生支援。OpenClaw 的 `tsdown.config.ts` 共約 195 行，定義了 25+ 個進入點（Entry Points）、外部依賴排除清單、警告抑制策略、以及 Plugin SDK 子路徑處理等細節。

本章完整走讀此設定檔，從匯入宣告到最終的 `defineConfig` 匯出，逐一解析每個函式與設定區塊的技術意涵。

---

## 1. tsdown 是什麼？

在深入設定細節之前，先建立對 tsdown 的基本認識：

| 面向 | 說明 |
|------|------|
| **定位** | TypeScript-first 的打包工具，專為 Node.js 函式庫和應用程式設計 |
| **底層引擎** | Rolldown — 以 Rust 實作的 Rollup 相容打包器，效能極高 |
| **與 Rollup 的關係** | API 與 Rollup 高度相容，但底層完全重寫為 Rust |
| **版本** | OpenClaw 使用 0.21.7 |
| **設定檔** | `tsdown.config.ts`，支援 TypeScript 原生撰寫 |
| **核心 API** | `defineConfig()` — 定義一或多個建置設定 |

**為什麼選擇 tsdown 而非 esbuild 或 Rollup？**

tsdown 基於 Rolldown，結合了 Rollup 的成熟生態（Plugin 系統、Tree Shaking 演算法）與 Rust 的極致效能。對於 OpenClaw 這種包含 25+ 進入點的大型專案而言，打包速度是關鍵考量。

---

## 2. 設定檔頂層結構

### 2.1 匯入宣告

```typescript
// source-repo/tsdown.config.ts:1-9
import fs from "node:fs";
import path from "node:path";
import { defineConfig, type UserConfig } from "tsdown";
import {
  listBundledPluginBuildEntries,
  listBundledPluginRuntimeDependencies,
} from "./scripts/lib/bundled-plugin-build-entries.mjs";
import { buildPluginSdkEntrySources } from "./scripts/lib/plugin-sdk-entries.mjs";
```

#### 匯入分析

| 匯入來源 | 匯入項目 | 用途 |
|----------|----------|------|
| `node:fs` | `fs` | 檔案系統操作（可能用於動態讀取進入點清單） |
| `node:path` | `path` | 路徑處理 |
| `tsdown` | `defineConfig`, `UserConfig` | tsdown 的設定定義函式與型別 |
| `./scripts/lib/bundled-plugin-build-entries.mjs` | `listBundledPluginBuildEntries`, `listBundledPluginRuntimeDependencies` | 動態列出所有 Bundled Plugin 的建置進入點和執行時期依賴 |
| `./scripts/lib/plugin-sdk-entries.mjs` | `buildPluginSdkEntrySources` | 建置 Plugin SDK 子路徑的進入點原始碼 |

**觀察**：匯入了兩個自定義腳本模組，說明 OpenClaw 的建置系統具有高度動態性——進入點和依賴清單並非全部寫死在設定檔中，部分是由腳本動態生成的。

---

## 3. 核心設定函式

### 3.1 環境變數設定

tsdown 設定中定義了 `env` 物件：

```
env: NODE_ENV = "production"
```

這確保所有打包產物都以生產模式編譯。在程式碼中使用 `process.env.NODE_ENV` 的地方，tsdown 會在建置時進行常數替換（Constant Folding），將 `process.env.NODE_ENV === "production"` 替換為 `true`，從而讓打包器能夠消除開發模式專用的程式碼（Dead Code Elimination）。

### 3.2 `nodeBuildConfig()` — Node.js 建置基礎設定

```
nodeBuildConfig(): platform=node, fixedExtension=false
```

| 參數 | 值 | 意義 |
|------|----|------|
| `platform` | `"node"` | 目標平台為 Node.js。tsdown 會自動將 Node.js 內建模組（如 `fs`、`path`、`http`）標記為外部（External），不打包進產物中。 |
| `fixedExtension` | `false` | 不強制使用固定副檔名。輸出檔案的副檔名會根據 `package.json` 的 `"type"` 欄位自動決定（`"module"` → `.js`，`"commonjs"` → `.cjs`）。 |

### 3.3 `buildInputOptions()` — 輸入選項與警告抑制

此函式配置 Rolldown 的輸入選項，最重要的功能是警告抑制策略：

```
buildInputOptions(): 
  - 抑制 EVAL 警告：protobufjs, bottleneck
  - 抑制 UNRESOLVED_IMPORT 警告：extensions/
  - 抑制 PLUGIN_TIMINGS 警告
```

#### 警告抑制詳解

| 警告碼 | 涉及套件/路徑 | 抑制原因 |
|--------|---------------|----------|
| `EVAL` | `protobufjs` | protobufjs 在執行時期使用 `eval()` 來動態編譯 Protocol Buffer 解碼器。這是該套件的已知行為，無法避免。 |
| `EVAL` | `bottleneck` | bottleneck（速率限制器套件）內部使用了 `eval()` 相關的動態程式碼執行。 |
| `UNRESOLVED_IMPORT` | `extensions/` | 擴充套件目錄中的匯入在打包時無法解析，因為這些匯入在執行時期才會被解析。這是 OpenClaw 擴充套件架構的設計特性。 |
| `PLUGIN_TIMINGS` | _(全域)_ | 隱藏 Rolldown 插件計時資訊，減少建置輸出的雜訊。 |

**設計解讀**：警告抑制策略反映了 OpenClaw 對第三方依賴的務實態度——不是消除所有警告，而是有意識地接受已知且無害的警告，同時確保真正的問題不會被淹沒。

---

## 4. 進入點定義

### 4.1 `buildCoreDistEntries()` — 核心進入點

這是設定檔中最核心的部分，定義了 OpenClaw 的所有建置進入點。共 25+ 個進入點，每個對應一個獨立的輸出模組：

```typescript
// source-repo/tsdown.config.ts:138-165（概念還原）
function buildCoreDistEntries() {
  return {
    // === 主進入點 ===
    "index":                         "src/index.ts",
    "entry":                         "src/entry.ts",

    // === CLI ===
    "cli/daemon-cli":                "src/cli/daemon-cli.ts",

    // === Agents（代理人模組） ===
    "agents/auth-profiles.runtime":         "src/agents/auth-profiles.runtime.ts",
    "agents/model-catalog.runtime":         "src/agents/model-catalog.runtime.ts",
    "agents/models-config.runtime":         "src/agents/models-config.runtime.ts",
    "agents/pi-model-discovery-runtime":    "src/agents/pi-model-discovery-runtime.ts",

    // === Commands ===
    "commands/status.summary.runtime":      "src/commands/status.summary.runtime.ts",

    // === Infrastructure ===
    "infra/boundary-file-read":             "src/infra/boundary-file-read.ts",
    "infra/warning-filter":                 "src/infra/warning-filter.ts",

    // === Plugin 系統 ===
    "plugins/provider-discovery.runtime":   "src/plugins/provider-discovery.runtime.ts",
    "plugins/provider-runtime.runtime":     "src/plugins/provider-runtime.runtime.ts",
    "plugins/public-surface-runtime":       "src/plugins/public-surface-runtime.ts",
    "plugins/sdk-alias":                    "src/plugins/sdk-alias.ts",
    "plugins/build-smoke-entry":            "src/plugins/build-smoke-entry.ts",
    "plugins/runtime/index":                "src/plugins/runtime/index.ts",

    // === 門面啟動檢查 ===
    "facade-activation-check.runtime":      "src/facade-activation-check.runtime.ts",

    // === Extension API ===
    "extensionAPI":                         "src/extensionAPI.ts",

    // === Telegram 擴充 ===
    "telegram/audit":                       "extensions/telegram/audit.ts",
    "telegram/token":                       "extensions/telegram/token.ts",

    // === 其他工具 ===
    "llm-slug-generator":                   "src/llm-slug-generator.ts",
    "mcp/plugin-tools-serve":               "src/mcp/plugin-tools-serve.ts",
  };
}
```

### 4.2 進入點分類分析

#### 按功能分類

| 分類 | 進入點數量 | 代表進入點 | 說明 |
|------|-----------|-----------|------|
| 主進入點 | 2 | `index`, `entry` | 應用程式的核心啟動點。`index` 是對外匯出的主模組，`entry` 可能是實際的啟動邏輯。 |
| CLI | 1 | `cli/daemon-cli` | 命令列守護行程介面（Daemon CLI） |
| Agents | 4 | `agents/*.runtime` | 代理人模組——認證設定檔、模型目錄、模型設定、模型探索 |
| Commands | 1 | `commands/status.summary.runtime` | 命令模組——狀態摘要 |
| Infrastructure | 2 | `infra/boundary-file-read`, `infra/warning-filter` | 基礎設施模組——邊界檔案讀取、警告過濾器 |
| Plugin 系統 | 6 | `plugins/*.runtime`, `plugins/sdk-alias` 等 | 外掛系統的各個子模組 |
| Extension API | 1 | `extensionAPI` | 擴充套件 API |
| Telegram | 2 | `telegram/audit`, `telegram/token` | Telegram 擴充的審計與權杖模組 |
| 工具 | 2 | `llm-slug-generator`, `mcp/plugin-tools-serve` | 獨立工具模組 |

#### `.runtime` 命名慣例

大量進入點使用 `.runtime` 後綴（如 `auth-profiles.runtime`、`model-catalog.runtime`），這暗示 OpenClaw 將程式碼分為「建置時期」和「執行時期」兩個階段。`.runtime` 檔案是專門在應用程式執行時被載入的模組，而非建置工具或腳本。

#### 跨目錄進入點

值得注意的是，大部分進入點來自 `src/` 目錄，但 Telegram 相關的進入點來自 `extensions/` 目錄：

```
"telegram/audit":  "extensions/telegram/audit.ts"
"telegram/token":  "extensions/telegram/token.ts"
```

這意味著某些擴充套件被視為核心建置的一部分（Bundled Extensions），而非獨立的套件。

### 4.3 `buildUnifiedDistEntries()` — 統一進入點

```
buildUnifiedDistEntries() = 
  核心進入點（buildCoreDistEntries）
  + Plugin SDK 子路徑進入點（buildPluginSdkEntrySources）
  + Bundled Plugin 進入點（listBundledPluginBuildEntries）
  + Bundled Hooks 進入點
```

此函式將四個來源的進入點合併為一個統一的進入點清單：

| 來源 | 函式/資料 | 說明 |
|------|-----------|------|
| 核心進入點 | `buildCoreDistEntries()` | 上述 25+ 個靜態定義的進入點 |
| Plugin SDK 子路徑 | `buildPluginSdkEntrySources()` | 動態生成的 Plugin SDK 子模組進入點 |
| Bundled Plugins | `listBundledPluginBuildEntries()` | 動態列出的 Bundled Plugin 進入點 |
| Bundled Hooks | _(內嵌)_ | Bundled 生命週期鉤子（Hooks）的進入點 |

**設計解讀**：OpenClaw 的建置產物是一個「統一分發包」（Unified Distribution），包含核心程式碼、Plugin SDK、以及所有 Bundled Plugin。這使得安裝後即可使用完整功能，無需額外安裝外掛。

---

## 5. 外部依賴排除

### 5.1 `explicitNeverBundleDependencies`

```typescript
// 明確永不打包的依賴
explicitNeverBundleDependencies = [
  "@lancedb/lancedb",
  "@matrix-org/matrix-sdk-crypto-nodejs",
  "matrix-js-sdk",
  // + bundledPluginRuntimeDependencies（動態列表）
]
```

#### 排除原因分析

| 依賴 | 排除原因 |
|------|----------|
| `@lancedb/lancedb` | LanceDB 是一個向量資料庫（Vector Database），其 npm 套件包含原生二進位模組（Native Binary Modules，如 `.node` 檔案），無法被 JavaScript 打包器處理。必須由使用者在安裝時透過 npm/pnpm 安裝，以取得與其作業系統和 CPU 架構相符的原生模組。 |
| `@matrix-org/matrix-sdk-crypto-nodejs` | Matrix 通訊協定的加密 SDK，同樣包含 Rust 編譯的原生模組（透過 napi-rs），無法打包。 |
| `matrix-js-sdk` | Matrix 通訊協定的 JavaScript SDK。可能因為體積過大或有動態 `require()` 等無法靜態分析的匯入模式而被排除。 |
| `bundledPluginRuntimeDependencies` | 由 `listBundledPluginRuntimeDependencies()` 動態產生的依賴清單。這些是 Bundled Plugin 在執行時期需要的依賴，但不應被打包進主產物中（可能包含原生模組或體積過大的套件）。 |

### 5.2 外部化策略的層次

tsdown/Rolldown 的外部化（Externalization）在 OpenClaw 中分為三個層次：

```
第一層：Node.js 內建模組（自動）
  ↓ platform: "node" 自動處理
第二層：package.json dependencies（自動）
  ↓ tsdown 預設行為
第三層：explicitNeverBundleDependencies（手動）
  ↓ 明確排除特定套件
```

1. **自動外部化**：`platform: "node"` 使 tsdown 自動將 `node:fs`、`node:path` 等內建模組標記為外部。
2. **依賴外部化**：tsdown 預設會將 `package.json` 中 `dependencies`（非 `devDependencies`）列出的套件標記為外部。
3. **手動外部化**：`explicitNeverBundleDependencies` 提供額外的手動排除清單，用於處理上述兩層無法覆蓋的特殊情況。

---

## 6. 最終設定匯出

### 6.1 `defineConfig` 結構

```typescript
// source-repo/tsdown.config.ts（概念還原）
export default defineConfig([
  nodeBuildConfig({
    entry: buildUnifiedDistEntries(),
    inputOptions: buildInputOptions(),
    external: explicitNeverBundleDependencies,
    env: { NODE_ENV: "production" },
    // ...其他選項
  })
]);
```

**關鍵觀察**：`defineConfig` 接收一個陣列，但 OpenClaw 只定義了一個建置設定。這意味著：

- 所有進入點共用同一組建置選項（platform、external、env 等）
- 不需要針對不同目標（如瀏覽器 vs. Node.js）產生不同的產物
- 整個專案的打包是一次完成的，而非多步驟

---

## 7. 建置產物結構推斷

根據進入點定義，可以推斷出 `dist/` 目錄的結構：

```
dist/
├── index.js                              ← 主進入點
├── entry.js                              ← 啟動邏輯
├── extensionAPI.js                       ← Extension API
├── facade-activation-check.runtime.js    ← 門面啟動檢查
├── llm-slug-generator.js                 ← LLM slug 生成器
├── cli/
│   └── daemon-cli.js                     ← CLI 守護行程
├── agents/
│   ├── auth-profiles.runtime.js          ← 認證設定代理人
│   ├── model-catalog.runtime.js          ← 模型目錄代理人
│   ├── models-config.runtime.js          ← 模型設定代理人
│   └── pi-model-discovery-runtime.js     ← 模型探索代理人
├── commands/
│   └── status.summary.runtime.js         ← 狀態摘要命令
├── infra/
│   ├── boundary-file-read.js             ← 邊界檔案讀取
│   └── warning-filter.js                 ← 警告過濾器
├── plugins/
│   ├── provider-discovery.runtime.js     ← Provider 探索
│   ├── provider-runtime.runtime.js       ← Provider 執行時期
│   ├── public-surface-runtime.js         ← 公開表面執行時期
│   ├── sdk-alias.js                      ← SDK 路徑別名
│   ├── build-smoke-entry.js              ← 建置煙霧測試
│   └── runtime/
│       └── index.js                      ← Plugin 執行時期核心
├── telegram/
│   ├── audit.js                          ← Telegram 審計
│   └── token.js                          ← Telegram 權杖
├── mcp/
│   └── plugin-tools-serve.js             ← MCP 工具服務
├── plugin-sdk/                           ← Plugin SDK 子路徑
│   └── ...（動態生成）
└── ...（Bundled Plugin 產物）
```

---

## 8. 建置流程整合

### 8.1 建置腳本調用鏈

```
pnpm build
  └── node scripts/build-all.mjs
        ├── tsdown（讀取 tsdown.config.ts）
        │   ├── buildUnifiedDistEntries()  → 收集所有進入點
        │   ├── buildInputOptions()        → 設定輸入選項
        │   ├── nodeBuildConfig()          → 設定平台選項
        │   └── → 產生 dist/
        └── （其他建置步驟）

pnpm build:docker
  └── node scripts/tsdown-build.mjs
        └── 直接調用 tsdown API
      && node scripts/runtime-postbuild.mjs
        └── 後處理建置產物
      && ...（Docker 特定步驟）
```

### 8.2 開發模式 vs. 生產模式

| 面向 | 開發模式 (`pnpm dev`) | 生產模式 (`pnpm build`) |
|------|----------------------|------------------------|
| 執行方式 | `node scripts/run-node.mjs`（直接透過 tsx 執行 TypeScript） | `node scripts/build-all.mjs`（tsdown 打包） |
| 打包 | 不打包 | 完整打包（25+ 進入點） |
| NODE_ENV | _(未設定或 development)_ | `"production"` |
| 速度 | 快（無打包步驟） | 較慢（需完整打包） |
| 產物 | 無（直接從原始碼執行） | `dist/` 目錄 |

---

## 9. 關鍵設計決策總結

### 9.1 單一設定，多進入點

OpenClaw 選擇在一個 `defineConfig` 中定義所有進入點，而非為每個模組建立獨立的建置設定。這簡化了建置流程，但也意味著所有模組必須共用相同的建置選項。

### 9.2 動態進入點收集

透過 `listBundledPluginBuildEntries()` 和 `buildPluginSdkEntrySources()` 動態收集進入點，使建置系統能自動感知新增的 Plugin 和 SDK 子路徑，無需手動更新設定檔。

### 9.3 原生模組的明確排除

對於包含原生二進位模組的依賴（LanceDB、Matrix Crypto），OpenClaw 採用明確排除策略。這是 Node.js 生態系中的常見做法——原生模組必須在使用者的機器上編譯或下載預編譯版本，無法被 JavaScript 打包器處理。

### 9.4 警告即文檔

`buildInputOptions()` 中的警告抑制清單，實質上也是一份「已知問題文檔」——記錄了哪些第三方套件有已知的打包警告，以及為什麼這些警告是安全的。

### 9.5 `.runtime` 慣例的架構含義

大量使用 `.runtime` 後綴暗示 OpenClaw 有意識地區分「建置時期程式碼」和「執行時期程式碼」。這種分離有助於：
- 減少執行時期的載入量（只載入 `.runtime` 模組）
- 清楚標識每個模組的生命週期
- 便於進行 Tree Shaking（搖樹優化）

---

## 引用來源

| 來源檔案 | 行號範圍 | 引用內容 |
|----------|----------|----------|
| `source-repo/tsdown.config.ts` | 1-195 | 完整設定檔 |
| `source-repo/tsdown.config.ts` | 1-9 | 匯入宣告 |
| `source-repo/tsdown.config.ts` | 138-165 | `buildCoreDistEntries()` 進入點定義 |
| `source-repo/scripts/lib/bundled-plugin-build-entries.mjs` | — | Bundled Plugin 建置進入點腳本 |
| `source-repo/scripts/lib/plugin-sdk-entries.mjs` | — | Plugin SDK 子路徑進入點腳本 |
| `source-repo/package.json` | scripts 區段 | `build`、`build:docker` 腳本定義 |
| `source-repo/tsconfig.json` | 1-33 | TypeScript 設定（影響打包行為） |
