# OpenClaw 插件系統（Plugin System）深度解析

---

## 本章摘要

OpenClaw 的插件系統（Plugin System）是整個框架的可擴展性基石。它允許開發者透過標準化的介面，為人工智慧代理（AI Agent）添加新的能力——包括工具（Tool）、通道（Channel）、記憶（Memory）、模型提供者（Provider）以及生命週期鉤子（Hook）。如果你想理解 OpenClaw 為何能支援如此多樣化的功能——從連接不同的大型語言模型（Large Language Model, LLM）到整合各種通訊平台——答案就藏在這套精密的插件架構之中。

本章將從零開始，假設讀者對 OpenClaw 完全陌生，逐步拆解插件系統的每一個組成部分。我們會深入探討以下十個核心主題：

1. **目錄結構與程式碼規模**：核心基礎設施、公開軟體開發套件、外部驗證合約，以及一百一十一個內建擴充套件的整體佈局。透過了解目錄結構，我們可以建立對整個系統的空間直覺。
2. **插件清單（Plugin Manifest）**：每個插件的自描述檔案 `openclaw.plugin.json` 的完整型別定義。清單是插件對外溝通的「名片」，系統透過它來了解插件的身份和能力。
3. **插件發現機制（Plugin Discovery）**：系統如何自動掃描檔案系統、辨識候選插件、並快取結果以提升效能。這是整個插件生命週期的起點。
4. **插件載入器（Plugin Loader）**：長達兩千兩百多行的核心調度器，如何將原始碼即時轉譯、驗證相容性、並匯入執行環境。這是連接「靜態檔案」與「活躍功能」的橋樑。
5. **插件註冊中心（Plugin Registry）**：連接插件到所有子系統的中央樞紐。註冊中心是整個插件系統的「神經中樞」。
6. **執行時期狀態（Runtime State）**：全域符號（Symbol）註冊、版本追蹤、以及表面釘選機制。這些低階機制確保了插件在併發環境中的正確運作。
7. **鉤子系統（Hook System）**：三十種生命週期鉤子的完整清單與執行邏輯。鉤子是插件介入代理行為的主要手段。
8. **插件軟體開發套件（Plugin SDK）**：面向外部開發者的公開應用程式介面（API）與入口函式。這是第三方開發者實際接觸到的介面層。
9. **外部插件驗證合約（Plugin Package Contract）**：確保第三方插件相容性的驗證機制。這是維護插件生態系統品質的守門員。
10. **完整生命週期流程**：從發現到卸載的五階段流程圖解。將所有元件串連起來，呈現全貌。

閱讀本章後，你將具備理解、開發、以及除錯 OpenClaw 插件所需的全部基礎知識。無論你是想開發自己的插件，還是想深入理解 OpenClaw 的內部運作原理，本章都將提供堅實的知識基礎。

---

## 1. 插件系統的整體架構（Architecture Overview）

在開始深入任何細節之前，讓我們先從宏觀的角度理解插件系統的整體佈局。OpenClaw 的插件系統由四個主要區域組成，各自承擔截然不同的職責，共同構成一個完整的插件生態系統基礎設施。

### 1.1 目錄結構與程式碼規模

以下是整體的目錄結構與程式碼規模概覽：

| 目錄路徑 | 用途 | 檔案數量 | 程式碼行數（約） |
|---|---|---|---|
| `src/plugins/` | 核心插件基礎設施（Core Plugin Infrastructure） | 218 個 TypeScript 檔案 | ~76,805 行 |
| `src/plugin-sdk/` | 公開軟體開發套件，供插件作者使用 | 326 個檔案 | ~29,500 行 |
| `packages/plugin-package-contract/` | 外部插件驗證合約 | 3 個檔案 | — |
| `extensions/` | 內建擴充套件目錄 | 111 個擴充套件目錄 | — |

光從數字來看，核心基礎設施就有將近七萬七千行程式碼，這個規模說明了插件系統在 OpenClaw 中佔據的核心地位。它不是一個附加功能，而是整個框架的骨幹。

### 1.2 各區域的職責劃分

讓我們逐一理解每個區域的設計目的與職責邊界：

**`src/plugins/`——核心基礎設施**：這是插件系統的心臟地帶。所有關於插件的發現（Discovery）、載入（Loading）、註冊（Registration）、執行時期管理（Runtime Management）、以及鉤子派發（Hook Dispatching）的邏輯都封裝在這裡。兩百一十八個檔案涵蓋了從底層的型別定義到高階的生命週期管理等各個層面。普通的插件開發者通常不需要直接接觸這些程式碼，但了解它們有助於在遇到問題時進行深度除錯。

**`src/plugin-sdk/`——公開軟體開發套件**：這是面向外部開發者的公開介面層。它將內部複雜的型別與功能包裝成簡潔、穩定的應用程式介面，讓第三方開發者不需要了解內部實作就能開發功能豐富的插件。三百二十六個檔案代表了軟體開發套件提供的廣泛功能覆蓋——從基礎的型別定義到進階的工具工廠模式，應有盡有。這也是內部與外部之間的「防火牆」：即使內部實作發生重大重構，只要軟體開發套件的介面保持不變，外部插件就不需要修改。

**`packages/plugin-package-contract/`——外部驗證合約**：這是一個獨立的套件，定義了外部插件必須滿足的相容性合約。它僅有三個檔案，但扮演著確保生態系統穩定的守門員角色。每當第三方開發者要發布插件到 ClawHub（官方的插件市集）時，插件的封裝配置檔案就必須通過這個合約的驗證。這個設計確保了所有公開發布的插件都明確聲明了自身的相容性資訊。

**`extensions/`——內建擴充套件**：包含一百一十一個內建擴充套件，每一個都是一個完整、獨立的插件。這些擴充套件既是 OpenClaw 核心功能的來源（例如記憶系統、各種通道整合、模型提供者），也是第三方開發者學習插件開發的最佳參考範例。「以插件形式實作核心功能」的設計哲學，意味著核心團隊和外部開發者使用的是完全相同的插件應用程式介面，這保證了應用程式介面的完整性和實用性。

---

## 2. 插件清單（Plugin Manifest）

每個 OpenClaw 插件都可以包含一個清單檔案，用來描述自身的元資料（Metadata）和能力聲明（Capability Declaration）。清單檔案是插件的「數位身份證」——系統在載入插件之前，會先讀取清單來判斷這個插件是什麼、能做什麼、以及在什麼條件下應該被啟動。

### 2.1 清單檔案名稱與格式

清單檔案的標準名稱定義在核心基礎設施中，這是一個不可變的常數：

```typescript
// source-repo/src/plugins/manifest.ts:17
export const PLUGIN_MANIFEST_FILENAME = "openclaw.plugin.json";
```

值得注意的是，雖然副檔名是 `.json`，但系統實際上使用 JSON5 格式來解析清單內容：

```typescript
// source-repo/src/plugins/manifest.ts:3
import JSON5 from "json5";
```

為什麼選擇 JSON5 而不是標準的 JSON 格式呢？這是一個以開發者體驗為中心的設計決策。JSON5 的好處在於它允許使用註解（Comments）、尾隨逗號（Trailing Commas）、以及單引號字串等便利語法。對於需要頻繁編輯清單檔案的插件開發者來說，能夠在配置中添加註解來解釋每個欄位的用途，大幅降低了維護的認知負擔。舉例來說，開發者可以在清單中寫下 `// 這個通道只支援文字訊息` 這樣的註解，幫助團隊成員理解配置的意圖。

### 2.2 清單型別系統

清單的型別系統定義了插件可以聲明的所有配置項目。以下我們逐一深入說明每個關鍵型別，並解釋其設計意圖。

#### 2.2.1 通道配置型別（PluginManifestChannelConfig）

通道配置描述了插件如何與外部通訊平台（如 Discord、WhatsApp、Telegram 等）進行整合：

```typescript
// source-repo/src/plugins/manifest.ts:20-27
type PluginManifestChannelConfig = {
  schema: ...;
  uiHints: ...;
  runtime: ...;
  label: string;
  description: string;
  preferOver: ...;
};
```

| 欄位 | 型別說明 | 設計目的 |
|---|---|---|
| `schema` | JSON Schema 物件 | 定義使用者配置此通道時需要填寫的欄位結構，系統可以用它來自動驗證使用者輸入的正確性 |
| `uiHints` | 顯示提示物件 | 提供給前端使用者介面的渲染提示，例如哪些欄位應該顯示為密碼框、哪些應該用下拉選單等 |
| `runtime` | 執行時期配置 | 通道在執行時期的行為參數配置 |
| `label` | 字串 | 通道的人類可讀顯示名稱，例如「Discord 機器人」 |
| `description` | 字串 | 通道的詳細文字描述，用於幫助使用者理解此通道的功能和限制 |
| `preferOver` | 優先順序宣告 | 當系統中存在多個可以處理相同類型訊息的通道時，此欄位聲明當前通道應該優先於哪些其他通道 |

#### 2.2.2 模型支援型別（PluginManifestModelSupport）

此型別讓插件明確聲明它支援哪些大型語言模型：

```typescript
// source-repo/src/plugins/manifest.ts:29-40
type PluginManifestModelSupport = {
  modelPrefixes: ...;
  modelPatterns: ...;
};
```

兩個欄位提供了不同精細度的匹配策略。`modelPrefixes` 使用前綴匹配，適合涵蓋整個模型家族（例如 `"gpt-4"` 可以匹配 `gpt-4`、`gpt-4-turbo`、`gpt-4o` 等所有變體）。`modelPatterns` 使用正則表達式模式匹配，適合需要更精確控制的場景（例如只匹配特定的模型版本號）。這種雙層匹配設計在保持簡單用法容易上手的同時，為進階使用者保留了完全的控制能力。

#### 2.2.3 啟動能力型別（PluginManifestActivationCapability）

這是插件可以聲明的四種基本能力類型，每一種都對應到插件系統的一個核心子系統：

```typescript
// source-repo/src/plugins/manifest.ts:42
type PluginManifestActivationCapability = "provider" | "channel" | "tool" | "hook";
```

| 能力類型 | 對應子系統 | 功能說明 | 典型範例 |
|---|---|---|---|
| `provider` | 模型提供者子系統 | 連接到大型語言模型的應用程式介面（例如 OpenAI、Anthropic、Google 等） | OpenAI 提供者插件 |
| `channel` | 通道子系統 | 連接到外部通訊平台，使代理可以在該平台上與使用者互動 | Discord 通道插件 |
| `tool` | 工具子系統 | 為代理提供新的可調用函式，擴展代理可以執行的操作 | 檔案搜尋工具插件 |
| `hook` | 鉤子子系統 | 在代理生命週期的特定時刻插入自訂邏輯，例如記錄日誌或修改行為 | 審計日誌插件 |

一個插件可以同時聲明多種能力類型。例如，一個「智慧記憶」插件可能同時提供工具（讓代理可以搜尋記憶）和鉤子（在每次對話結束時自動儲存記憶摘要）。

#### 2.2.4 啟動條件型別（PluginManifestActivation）

啟動條件定義了插件在何種情境下會被系統激活。這是一個重要的效能最佳化設計——延遲載入（Lazy Loading）機制：

```typescript
// source-repo/src/plugins/manifest.ts:44-58
type PluginManifestActivation = {
  onProviders: ...;
  onCommands: ...;
  onChannels: ...;
  onRoutes: ...;
  onCapabilities: ...;
};
```

| 啟動條件 | 觸發時機 | 實際應用場景 |
|---|---|---|
| `onProviders` | 當系統需要特定的模型提供者時 | 只有當使用者選擇了 Anthropic 模型時，才載入 Anthropic 提供者插件 |
| `onCommands` | 當使用者執行特定的命令列指令時 | 只有當使用者輸入了記憶相關指令時，才載入記憶管理插件 |
| `onChannels` | 當系統啟動特定的通訊通道時 | 只有當使用者配置了 Discord 通道時，才載入 Discord 插件 |
| `onRoutes` | 當超文本傳輸協定請求匹配特定路由時 | 只有當外部系統呼叫了特定的 API 端點時，才載入對應的處理插件 |
| `onCapabilities` | 當系統需要特定能力時 | 只有當代理需要圖片生成能力時，才載入圖片生成插件 |

為什麼延遲載入如此重要？想像一下 OpenClaw 有一百一十一個內建擴充套件，如果在啟動時全部載入，不僅會大幅增加啟動時間，還會佔用大量的記憶體。透過啟動條件機制，系統只會載入實際需要的插件，其餘的保持為「候選」狀態。這使得系統即使安裝了大量插件，仍能保持快速啟動和低記憶體佔用。

#### 2.2.5 安裝設定型別（PluginManifestSetup）

安裝設定描述了插件在系統中的基礎需求和安裝時期的行為：

```typescript
// source-repo/src/plugins/manifest.ts:69-80
type PluginManifestSetup = {
  providers: ...;
  cliBackends: ...;
  configMigrations: ...;
  requiresRuntime: ...;
};
```

| 欄位 | 說明 |
|---|---|
| `providers` | 此插件需要或提供的模型提供者定義 |
| `cliBackends` | 此插件需要的命令列後端配置 |
| `configMigrations` | 配置遷移腳本，用於在插件版本升級時自動轉換舊格式的配置檔案 |
| `requiresRuntime` | 是否需要特定的執行時期環境（例如特定版本的 Node.js 或特定的系統依賴） |

配置遷移（Config Migrations）是一個特別值得關注的設計。當插件的配置格式在不同版本之間發生變化時，遷移腳本可以自動將使用者的舊配置轉換為新格式，避免使用者在升級插件後需要手動修改配置。這對於維護良好的使用者體驗至關重要。

---

## 3. 插件發現機制（Plugin Discovery）

插件發現是整個生命週期的第一步，也是最基礎的一步。在任何插件能夠被載入和使用之前，系統首先需要知道「哪裡有插件」。發現機制就是負責回答這個問題的模組。

### 3.1 掃描規則與檔案過濾

系統在掃描目錄時，需要知道哪些檔案可能是插件的入口點，以及哪些目錄應該被跳過。這兩組規則定義在以下常數中：

```typescript
// source-repo/src/plugins/discovery.ts:24-36
const EXTENSION_EXTS = new Set([".ts", ".js", ".mts", ".cts", ".mjs", ".cjs"]);
const SCANNED_DIRECTORY_IGNORE_NAMES = new Set([
  ".git", ".hg", ".svn", ".turbo", ".yarn", ".yarn-cache",
  "build", "coverage", "dist", "node_modules",
]);
```

系統能辨識的原始碼副檔名涵蓋了 TypeScript 和 JavaScript 生態系統中的所有主要模組格式：

| 副檔名 | 模組系統 | 語言 | 說明 |
|---|---|---|---|
| `.ts` | 依編譯配置而定 | TypeScript | 最常見的 TypeScript 原始碼格式 |
| `.js` | 依編譯配置而定 | JavaScript | 最常見的 JavaScript 原始碼格式 |
| `.mts` | ECMAScript 模組 | TypeScript | 明確指定為 ES 模組的 TypeScript 檔案 |
| `.cts` | CommonJS 模組 | TypeScript | 明確指定為 CommonJS 模組的 TypeScript 檔案 |
| `.mjs` | ECMAScript 模組 | JavaScript | 明確指定為 ES 模組的 JavaScript 檔案 |
| `.cjs` | CommonJS 模組 | JavaScript | 明確指定為 CommonJS 模組的 JavaScript 檔案 |

這種全面的副檔名支援確保了無論插件作者使用哪種模組格式，系統都能正確辨識。對於剛接觸 JavaScript 生態的讀者，這裡簡要解釋：ECMAScript 模組和 CommonJS 模組是兩種不同的模組載入機制，前者使用 `import/export` 語法，後者使用 `require/module.exports` 語法。OpenClaw 同時支援兩者，減少了對插件作者的限制。

同時，掃描過程會自動跳過以下類型的目錄，避免不必要的效能損耗和誤判：

- **版本控制目錄**（`.git`、`.hg`、`.svn`）：這些目錄包含版本控制系統的內部資料，不可能包含插件
- **建置快取目錄**（`.turbo`）：Turborepo 的建置快取目錄
- **套件管理目錄**（`.yarn`、`.yarn-cache`、`node_modules`）：套件管理器的內部目錄
- **建置輸出目錄**（`build`、`dist`）：編譯後的產出物，通常是原始碼的副本
- **測試覆蓋目錄**（`coverage`）：測試覆蓋率報告目錄

### 3.2 候選插件的資料結構（PluginCandidate）

當掃描器找到一個潛在的插件時，不會立即載入它，而是將其封裝為一個「候選插件」（Plugin Candidate）物件。這個物件收集了在發現階段能獲取的所有資訊，為後續的載入階段提供基礎：

```typescript
// source-repo/src/plugins/discovery.ts:38-54
export type PluginCandidate = {
  idHint: string;
  source: string;
  setupSource?: string;
  rootDir: string;
  origin: PluginOrigin;
  format?: PluginFormat;
  bundleFormat?: PluginBundleFormat;
  workspaceDir?: string;
  packageName?: string;
  packageVersion?: string;
  packageDescription?: string;
  packageDir?: string;
  packageManifest?: OpenClawPackageManifest;
  bundledManifest?: PluginManifest;
  bundledManifestPath?: string;
};
```

每個欄位的設計目的如下——注意必填欄位和可選欄位的區分，這反映了不同來源的插件可能擁有不同程度的元資料：

| 欄位 | 是否必填 | 說明 |
|---|---|---|
| `idHint` | 必填 | 插件的建議識別碼，通常來自目錄名或套件名。注意這只是「提示」，最終的識別碼可能在載入階段被修改 |
| `source` | 必填 | 插件的主要入口檔案路徑，指向實際的程式碼位置 |
| `setupSource` | 可選 | 安裝腳本的路徑，如果插件有特殊的安裝邏輯 |
| `rootDir` | 必填 | 插件的根目錄路徑，作為解析相對路徑的基準 |
| `origin` | 必填 | 插件的來源類型，例如內建、本地開發、或外部套件安裝 |
| `format` | 可選 | 插件的程式碼格式（TypeScript 原始碼或已編譯的 JavaScript） |
| `bundleFormat` | 可選 | 如果插件是已打包的格式，描述打包方式 |
| `workspaceDir` | 可選 | 如果插件屬於某個工作區（Workspace），記錄該工作區的目錄 |
| `packageName` | 可選 | 來自 `package.json` 的套件名稱 |
| `packageVersion` | 可選 | 來自 `package.json` 的套件版本號 |
| `packageDescription` | 可選 | 來自 `package.json` 的套件描述文字 |
| `packageDir` | 可選 | 套件目錄的路徑 |
| `packageManifest` | 可選 | 完整的套件清單物件 |
| `bundledManifest` | 可選 | 已打包插件中內嵌的 OpenClaw 清單資訊 |
| `bundledManifestPath` | 可選 | 已打包清單的檔案路徑 |

### 3.3 發現結果與診斷資訊

發現過程的輸出是一個結構化的結果物件，包含成功辨識的候選插件和過程中產生的診斷資訊：

```typescript
// source-repo/src/plugins/discovery.ts:56-59
export type PluginDiscoveryResult = {
  candidates: PluginCandidate[];
  diagnostics: PluginDiagnostic[];
};
```

`candidates` 陣列包含所有被成功辨識為潛在插件的候選物件。`diagnostics` 陣列則包含掃描過程中發現的問題——這可能是清單格式錯誤、缺少必要欄位、或者檔案權限不足等情況。診斷資訊的設計允許發現過程「盡可能多地收集結果」，而不是在遇到第一個問題時就中斷。這對於除錯特別有幫助：即使某些插件有問題，其他正常的插件仍然可以被發現和載入。

### 3.4 發現結果的快取機制

為了避免在短時間內重複掃描同一目錄（這在許多工作流程中是常見的情況），系統實作了一個基於時間的快取（Cache）機制：

```typescript
// source-repo/src/plugins/discovery.ts:61-64
const discoveryCache = new Map<string, { expiresAt: number; result: PluginDiscoveryResult }>();
const DEFAULT_DISCOVERY_CACHE_MS = 1000;
```

快取使用 `Map` 資料結構，以目錄路徑為鍵，儲存發現結果和過期時間。預設的快取有效期為一千毫秒（一秒鐘）。這個時間窗口足以在一次操作流程中避免重複掃描，同時又足夠短，確保開發者修改了插件後能迅速看到效果。

快取的有效期可以透過環境變數進行客製化調整：

```typescript
// source-repo/src/plugins/discovery.ts:70-83
function resolveDiscoveryCacheMs(env: NodeJS.ProcessEnv): number {
  const raw = env.OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS?.trim();
  ...
}
```

透過設定 `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` 環境變數，開發者可以根據自身需求調整快取行為。在插件開發階段，將此值設為 `0` 可以完全停用快取，確保每次操作都重新掃描檔案系統，立即反映任何修改。在生產環境中，可以適當增大此值以進一步降低檔案系統的存取壓力。

---

## 4. 插件載入器（Plugin Loader）

載入器是插件系統中最複雜、程式碼量最大的元件。它的原始碼長達兩千兩百一十二行，負責將發現階段產出的候選插件轉化為可以實際執行的插件實例。可以把它想像成一個「工廠」——原材料（候選插件）在這裡經過一系列的加工步驟，最終成為「成品」（已註冊的活躍插件）。

### 4.1 核心依賴關係

載入器依賴多個關鍵模組來完成工作，讓我們逐一了解每個依賴的角色：

```typescript
// source-repo/src/plugins/loader.ts:4
import { createJiti } from "jiti";

// source-repo/src/plugins/loader.ts:46
import { discoverOpenClawPlugins } from "./discovery.js";

// source-repo/src/plugins/loader.ts:75
import { createPluginRegistry } from "./registry.js";

// source-repo/src/plugins/loader.ts:82-83
import { setActivePluginRegistry, getActivePluginRegistry } from "./runtime.js";
```

| 依賴項目 | 來源 | 在載入流程中的角色 |
|---|---|---|
| `jiti` | 外部開源套件 | 核心轉譯工具。`jiti` 能在執行時期將 TypeScript 原始碼即時轉譯（Transpile）為 JavaScript，這意味著插件作者可以直接用 TypeScript 編寫插件而不需要預先執行編譯步驟。這大幅降低了插件開發的摩擦力 |
| `discoverOpenClawPlugins` | `discovery.ts` | 執行第一階段的插件發現流程，獲取候選插件列表。載入器依賴發現模組來找到需要載入的插件 |
| `createPluginRegistry` | `registry.ts` | 創建新的插件註冊中心實例。每次完整的載入流程都會創建一個全新的註冊中心 |
| `setActivePluginRegistry` | `runtime.ts` | 將新建的註冊中心設為全域活躍狀態，使所有子系統都能存取到最新載入的插件集合 |
| `getActivePluginRegistry` | `runtime.ts` | 取得目前活躍的註冊中心，用於判斷是否需要重新載入 |

### 4.2 軟體開發套件路徑別名解析系統

載入器內部實作了一個路徑別名（Path Alias）解析系統，這是讓插件開發體驗保持一致的關鍵機制：

```typescript
// source-repo/src/plugins/loader.ts:88-99
// SDK alias resolution system for mapping `openclaw/plugin-sdk/*` paths
```

這個系統的作用是將插件程式碼中的 `openclaw/plugin-sdk/*` 匯入路徑，自動對應到實際的軟體開發套件原始碼在檔案系統中的位置。為什麼需要這個機制？因為插件的實際部署位置可能在任何地方——可能是 OpenClaw 專案內部的 `extensions/` 目錄，也可能是使用者本地的某個開發目錄，或者是通過套件管理器安裝的外部套件。路徑別名解析系統確保無論插件的物理位置在哪裡，它都能用相同的、穩定的匯入路徑來存取軟體開發套件的功能。

### 4.3 載入流程概述

載入器的完整工作流程可以概括為以下七個步驟，每個步驟都對應到特定的程式碼邏輯：

1. **呼叫發現模組**：透過 `discoverOpenClawPlugins()` 獲取候選插件列表，包括所有被辨識的候選插件和診斷資訊
2. **解析清單檔案**：逐一讀取每個候選插件的 `openclaw.plugin.json`，使用 JSON5 格式解析其內容
3. **驗證相容性**：根據清單中的版本資訊和能力聲明，判斷插件是否與當前環境相容
4. **即時轉譯與匯入**：使用 `jiti` 動態載入插件的入口檔案。如果入口是 TypeScript 原始碼，`jiti` 會在匯入過程中自動完成轉譯
5. **建立註冊中心**：呼叫 `createPluginRegistry()` 創建一個新的、空白的插件註冊中心
6. **執行插件註冊**：依次呼叫每個已載入插件的 `register()` 方法，讓插件將自身的能力注入到註冊中心中
7. **啟動註冊中心**：透過 `setActivePluginRegistry()` 將填充完成的註冊中心設為全域活躍狀態

---

## 5. 插件註冊中心（Plugin Registry）

註冊中心是所有插件能力匯聚的中央樞紐。可以把它想像成一個大型的「功能商場」——每個插件是一個「商鋪」，它們各自帶來不同的「商品」（能力），並在註冊中心這個「商場」裡統一陳列和管理。

### 5.1 註冊中心與子系統的連接

從 `source-repo/src/plugins/registry.ts:1-76` 的匯入宣告可以看出，註冊中心與系統中的多個子系統直接建立了連接：

| 子系統 | 匯入來源 | 對應的註冊函式 | 功能說明 |
|---|---|---|---|
| 上下文引擎（Context Engine） | `context-engine/registry.js` | `registerContextEngineForOwner` | 為特定的所有者註冊上下文引擎，管理代理的記憶和上下文 |
| 內部鉤子（Internal Hooks） | `hooks/internal-hooks.js` | `registerInternalHook` / `unregisterInternalHook` | 管理系統內部的鉤子註冊和反註冊 |
| 指令系統（Command System） | — | `registerPluginCommand` | 讓插件能夠新增自訂的命令列指令 |
| 壓縮提供者（Compaction Provider） | — | `registerCompactionProvider` | 註冊對話上下文壓縮策略的實作 |
| 記憶嵌入提供者（Memory Embedding Provider） | — | `registerMemoryEmbeddingProvider` | 註冊將文字轉換為向量嵌入的實作，用於記憶的語意搜尋 |
| 記憶能力（Memory Capability） | — | `registerMemoryCapability` | 註冊完整的記憶能力定義，包括儲存、檢索、刪除等操作 |

註冊中心的設計哲學是實現「控制反轉」（Inversion of Control）：插件不需要知道每個子系統的內部運作方式和記憶體位置，只需要透過註冊中心提供的標準介面，將自己的能力「插入」到正確的位置即可。這實現了插件與核心系統之間的鬆耦合（Loose Coupling），使得任何一方的內部修改都不會影響到另一方——只要註冊介面保持穩定。

---

## 6. 執行時期狀態管理（Runtime State）

OpenClaw 使用一套精密的狀態管理機制來確保插件在整個處理程序的生命週期中正確運作。這個機制涉及全域符號（Global Symbol）、版本追蹤、以及表面釘選等底層技術。

### 6.1 全域狀態的定義與存取

```typescript
// source-repo/src/plugins/runtime-state.ts:3
export const PLUGIN_REGISTRY_STATE = Symbol.for("openclaw.pluginRegistryState");

// source-repo/src/plugins/runtime-state.ts:13-22
export type RegistryState = {
  activeRegistry: RuntimeTrackedPluginRegistry | null;
  activeVersion: number;
  httpRoute: RegistrySurfaceState;
  channel: RegistrySurfaceState;
  key: string | null;
  workspaceDir: string | null;
  runtimeSubagentMode: "default" | "explicit" | "gateway-bindable";
  importedPluginIds: Set<string>;
};
```

為什麼使用 `Symbol.for()` 而不是普通的模組層級變數？這是一個精心考量的設計決策。在 Node.js 環境中，由於套件管理器的去重（Deduplication）行為，同一個套件有時會被安裝多份、因此被載入為多個模組實例。如果使用普通變數，每個模組實例都會有自己獨立的一份狀態，導致不同模組看到不同的插件註冊中心——這會造成難以追蹤的錯誤。`Symbol.for()` 創建的是處理程序級別的全域唯一符號，無論有多少個模組實例，透過相同的字串鍵 `"openclaw.pluginRegistryState"` 存取到的永遠是同一個符號，因此指向的是同一份狀態物件。

各個狀態欄位的詳細說明：

| 欄位 | 型別 | 說明 |
|---|---|---|
| `activeRegistry` | `RuntimeTrackedPluginRegistry \| null` | 目前活躍的插件註冊中心實例。`null` 表示尚未載入任何插件 |
| `activeVersion` | `number` | 版本計數器，每次設定新的註冊中心時遞增一。用於偵測註冊中心是否已被更換 |
| `httpRoute` | `RegistrySurfaceState` | 超文本傳輸協定路由表面的追蹤狀態，管理插件暴露的 API 端點 |
| `channel` | `RegistrySurfaceState` | 通道表面的追蹤狀態，管理插件提供的通訊通道 |
| `key` | `string \| null` | 快取鍵值，用於判斷當前的註冊中心是否已過期 |
| `workspaceDir` | `string \| null` | 當前工作區的目錄路徑 |
| `runtimeSubagentMode` | 聯合型別字串 | 子代理（Sub-agent）的執行模式設定 |
| `importedPluginIds` | `Set<string>` | 已匯入的插件識別碼集合，用於防止同一個插件被重複載入 |

### 6.2 活躍註冊中心的管理函式

系統提供了三個核心函式來管理活躍的插件註冊中心，它們各自服務於不同的使用場景：

```typescript
// source-repo/src/plugins/runtime.ts:76-89
export function setActivePluginRegistry(
  registry: PluginRegistry,
  cacheKey?: string,
  runtimeSubagentMode: "default" | "explicit" | "gateway-bindable" = "default",
  workspaceDir?: string,
) {
  state.activeRegistry = registry;
  state.activeVersion += 1;
  syncTrackedSurface(state.httpRoute, registry, true);
  syncTrackedSurface(state.channel, registry, true);
  state.key = cacheKey ?? null;
  state.workspaceDir = workspaceDir ?? null;
  state.runtimeSubagentMode = runtimeSubagentMode;
}
```

`setActivePluginRegistry()` 是最核心的函式。它在設定新的註冊中心時，會同時執行以下操作：遞增版本號（`activeVersion += 1`）、同步超文本傳輸協定路由和通道的表面狀態、更新快取鍵和工作區目錄、設定子代理模式。版本號的遞增是一個「樂觀鎖」（Optimistic Locking）機制——其他元件可以在操作開始前記錄版本號，操作完成後再次檢查，如果版本號已變更，就知道註冊中心在此期間被替換了，需要重新獲取。

```typescript
// source-repo/src/plugins/runtime.ts:91-93
export function getActivePluginRegistry(): PluginRegistry | null {
  return asPluginRegistry(state.activeRegistry);
}

// source-repo/src/plugins/runtime.ts:99-107
export function requireActivePluginRegistry(): PluginRegistry {
  if (!state.activeRegistry) {
    state.activeRegistry = createEmptyPluginRegistry();
    ...
  }
  return asPluginRegistry(state.activeRegistry)!;
}
```

`getActivePluginRegistry()` 是一個安全的查詢函式，可能回傳 `null`。而 `requireActivePluginRegistry()` 是一個「保證回傳」的便利函式——如果目前沒有活躍的註冊中心，它會自動建立一個空的。這個設計遵循了「不要讓呼叫端處理不必要的複雜度」的原則。

### 6.3 表面釘選機制（Surface Pinning）

```typescript
// source-repo/src/plugins/runtime.ts:109-118
pinActivePluginHttpRouteRegistry()     // 釘選超文本傳輸協定路由表面
releasePinnedPluginHttpRouteRegistry() // 釋放釘選的表面
```

表面釘選（Surface Pinning）是一個專為併發場景設計的進階機制。設想以下情境：一個超文本傳輸協定請求正在被處理，處理過程中需要存取插件暴露的路由表。如果在處理過程中，另一個執行緒觸發了插件的重新載入（例如因為偵測到插件的原始碼被修改了），路由表可能會被替換，導致正在處理的請求引用到已失效的路由——這會造成不可預測的行為。

釘選機制的解決方式是：在開始處理請求前，呼叫 `pinActivePluginHttpRouteRegistry()` 將當前的路由表面「凍結」。在釘選期間，即使註冊中心被替換，已釘選的路由表面仍然保持有效。處理完成後，呼叫 `releasePinnedPluginHttpRouteRegistry()` 釋放釘選。

### 6.4 子代理模式

子代理模式（`runtimeSubagentMode`）控制了子代理與父代理之間的插件共享策略：

| 模式 | 行為說明 | 適用場景 |
|---|---|---|
| `default` | 子代理自動繼承父代理的完整插件環境 | 大多數標準使用場景，子代理需要與父代理相同的工具和能力 |
| `explicit` | 子代理需要在建立時明確指定要使用的插件清單 | 需要為子代理定制精簡插件環境的場景，例如安全敏感的操作 |
| `gateway-bindable` | 子代理可以在執行時期動態綁定到閘道（Gateway）提供的插件集合 | 多租戶閘道環境，每個租戶可能有不同的插件配置 |

---

## 7. 鉤子系統（Hook System）

鉤子系統是插件影響代理行為的主要手段。它在代理生命週期的每個關鍵時刻都定義了「鉤子點」（Hook Point），插件可以在這些時刻插入自訂的程式碼邏輯。

### 7.1 鉤子執行器的設計

```
// source-repo/src/plugins/hooks.ts:1-6
/**
 * Plugin Hook Runner
 *
 * Provides utilities for executing plugin lifecycle hooks with proper
 * error handling, priority ordering, and async support.
 */
```

鉤子執行器（Hook Runner）的設計體現了三個核心原則：

1. **錯誤隔離（Error Isolation）**：當某個插件的鉤子函式拋出異常時，執行器會捕獲該異常並記錄錯誤日誌，但不會中斷其他插件的鉤子執行。這確保了一個有問題的插件不會拖垮整個系統。
2. **優先順序排序（Priority Ordering）**：當多個插件都註冊了同一個鉤子時，執行器會按照插件聲明的優先順序依序執行。這讓插件開發者可以精確控制自己的邏輯在鉤子鏈中的位置。
3. **非同步支援（Async Support）**：鉤子函式可以是非同步的（回傳 Promise），執行器會正確地等待每個非同步鉤子完成後再執行下一個。這對於需要進行網路請求或資料庫操作的鉤子至關重要。

### 7.2 三十種生命週期鉤子完整清單

```typescript
// source-repo/src/plugins/hook-types.ts:55-84
export type PluginHookName = ...;
```

以下按功能類別完整列出所有三十種鉤子，並說明每一種的觸發時機和典型用途：

#### 模型與推理流程鉤子

| 鉤子名稱 | 觸發時機 | 典型用途 |
|---|---|---|
| `before_model_resolve` | 在系統選擇要使用的大型語言模型之前 | 動態切換模型、根據對話內容選擇最適合的模型 |
| `before_prompt_build` | 在系統組建要送給模型的提示詞之前 | 注入額外的系統指令、修改提示詞模板 |
| `llm_input` | 在將完整輸入送給大型語言模型之前 | 記錄輸入內容、進行最後的內容審查或修改 |
| `llm_output` | 在收到大型語言模型的回覆之後 | 過濾或轉換回覆內容、記錄模型輸出 |

#### 代理生命週期鉤子

| 鉤子名稱 | 觸發時機 | 典型用途 |
|---|---|---|
| `before_agent_start` | 在代理啟動、開始處理使用者請求之前 | 初始化插件的內部狀態、載入必要的外部資源 |
| `before_agent_reply` | 在代理準備回覆使用者之前 | 修改回覆的格式或內容、新增額外資訊 |
| `agent_end` | 在代理完成當前請求的處理之後 | 清理資源、儲存統計數據、更新記憶 |

#### 記憶與上下文壓縮鉤子

| 鉤子名稱 | 觸發時機 | 典型用途 |
|---|---|---|
| `before_compaction` | 在系統執行上下文壓縮（縮減對話歷史以節省模型窗口空間）之前 | 標記重要的對話段落不被壓縮 |
| `after_compaction` | 在上下文壓縮完成之後 | 驗證壓縮結果、更新相關的索引資訊 |
| `before_reset` | 在代理的記憶或上下文被重置之前 | 備份重要資訊、執行優雅的清理操作 |

#### 訊息處理鉤子

| 鉤子名稱 | 觸發時機 | 典型用途 |
|---|---|---|
| `inbound_claim` | 當收到一則入站訊息，系統需要決定由哪個代理或通道來處理 | 實作自訂的訊息路由邏輯 |
| `message_received` | 當訊息被成功接收並分配給處理者 | 記錄訊息日誌、觸發通知、更新統計 |
| `message_sending` | 當代理準備發送回覆訊息時（發送之前） | 修改訊息格式、新增附件、內容審查 |
| `message_sent` | 當訊息已成功發送完成 | 確認送達、更新送達狀態、觸發後續流程 |
| `before_message_write` | 在訊息被寫入持久化儲存之前 | 加密訊息內容、新增元資料標記 |

#### 工具調用鉤子

| 鉤子名稱 | 觸發時機 | 典型用途 |
|---|---|---|
| `before_tool_call` | 在代理準備呼叫工具函式之前 | 權限檢查、參數驗證、記錄審計日誌 |
| `after_tool_call` | 在工具函式執行完成之後 | 處理工具回傳結果、記錄執行時間 |
| `tool_result_persist` | 在工具的執行結果被持久化儲存之前 | 過濾敏感資訊、壓縮大型結果 |

#### 會話管理鉤子

| 鉤子名稱 | 觸發時機 | 典型用途 |
|---|---|---|
| `session_start` | 當一個新的對話會話（Session）開始時 | 載入使用者偏好設定、初始化會話級別的狀態 |
| `session_end` | 當對話會話結束時 | 儲存會話摘要、釋放會話資源、更新使用者資料 |

#### 子代理管理鉤子

| 鉤子名稱 | 觸發時機 | 典型用途 |
|---|---|---|
| `subagent_spawning` | 當系統即將建立一個子代理時 | 配置子代理的插件環境、設定資源限制 |
| `subagent_delivery_target` | 當子代理的目標對象（要回覆的對象）被確定時 | 修改或重新導向目標 |
| `subagent_spawned` | 當子代理已成功建立並開始運行 | 記錄子代理的建立事件、設定監控 |
| `subagent_ended` | 當子代理完成任務並結束運行 | 收集子代理的結果、清理資源 |

#### 閘道管理鉤子

| 鉤子名稱 | 觸發時機 | 典型用途 |
|---|---|---|
| `gateway_start` | 當閘道（Gateway）服務啟動時 | 初始化閘道級別的共享資源、建立外部連線 |
| `gateway_stop` | 當閘道服務停止時 | 優雅地關閉連線、釋放共享資源 |

#### 派發與安裝鉤子

| 鉤子名稱 | 觸發時機 | 典型用途 |
|---|---|---|
| `before_dispatch` | 在訊息被派發到處理鏈之前 | 修改派發目標、進行負載均衡邏輯 |
| `reply_dispatch` | 在回覆被派發時 | 將回覆複製到多個目標、記錄回覆路徑 |
| `before_install` | 在新的插件被安裝到系統之前 | 驗證插件的安全性、檢查相容性衝突 |

---

## 8. 插件軟體開發套件（Plugin SDK）

軟體開發套件是面向外部開發者的公開應用程式介面層。它的設計目標是讓開發者能用最少的程式碼、最短的學習時間，開發出功能完整的插件。

### 8.1 核心匯出型別

軟體開發套件的核心模組匯出了以下關鍵型別，每一個都對應到插件開發中的一個重要概念：

```typescript
// source-repo/src/plugin-sdk/core.ts:29-80
// 主要匯出的型別
```

| 型別名稱 | 類別 | 說明 |
|---|---|---|
| `OpenClawPluginApi` | 註冊介面 | 最核心的型別。插件在 `register()` 方法中接收此介面的實例，用來呼叫 `registerTool()`、`registerCli()`、`registerMemoryCapability()` 等註冊方法 |
| `OpenClawPluginDefinition` | 定義介面 | 插件定義的完整型別，作為 `definePluginEntry()` 函式的參數。包含識別碼、名稱、描述、類型、以及註冊函式 |
| `ChannelPlugin` | 通道介面 | 通道插件需要實作的型別定義，描述如何連接和管理外部通訊平台 |
| `PluginRuntime` | 執行時期介面 | 插件在執行時期可以存取的環境資訊和功能 |
| `ProviderAuthContext` | 認證介面 | 模型提供者插件在進行身份認證時可用的上下文資訊，例如 API 金鑰的存取方式 |
| `MemoryPluginCapability` | 記憶介面 | 記憶插件需要定義的能力集合，包括記憶的儲存、檢索、刪除等操作的實作 |
| `OpenClawPluginToolContext` | 工具上下文 | 工具函式在被代理呼叫時，可以存取的上下文物件。提供了關於當前對話、使用者、以及代理狀態的資訊 |
| `OpenClawPluginToolFactory` | 工具工廠 | 工具的工廠函式型別。工廠模式允許延遲建立工具實例，只在實際需要時才進行初始化 |

### 8.2 插件入口函式

```typescript
// source-repo/src/plugin-sdk/plugin-entry.ts:1-20
import type { OpenClawPluginDefinition } from "../plugins/types.js";
// definePluginEntry is the main entry point for plugin authors
```

`definePluginEntry()` 是每個插件的入口函式，也是插件開發者最先接觸到的應用程式介面。它接受一個 `OpenClawPluginDefinition` 物件作為參數，這個物件完整描述了插件的身份和行為。函式回傳值作為模組的預設匯出（default export），供載入器在匯入時取得。

這個函式的設計遵循了「宣告式程式設計」（Declarative Programming）的理念——開發者只需要「宣告」插件是什麼、能做什麼，而不需要「命令」系統如何載入和管理插件。所有的載入、註冊、生命週期管理都由框架自動處理。

---

## 9. 外部插件驗證合約（Plugin Package Contract）

當第三方開發者打包並發布插件到 ClawHub 插件市集時，插件必須通過驗證合約的一系列檢查。這個合約確保了插件生態系統的品質底線和相容性保證。

### 9.1 相容性資訊型別

```typescript
// source-repo/packages/plugin-package-contract/src/index.ts:6-11
export type ExternalPluginCompatibility = {
  pluginApiRange?: string;
  builtWithOpenClawVersion?: string;
  pluginSdkVersion?: string;
  minGatewayVersion?: string;
};
```

| 欄位 | 格式 | 說明 |
|---|---|---|
| `pluginApiRange` | 語意化版本範圍（Semver Range） | 此插件支援的插件應用程式介面版本範圍。例如 `"^2.0.0"` 表示支援二點零到三點零之前的所有版本 |
| `builtWithOpenClawVersion` | 精確版本號 | 開發者建置此插件時使用的 OpenClaw 版本，用於追溯和除錯 |
| `pluginSdkVersion` | 精確版本號 | 插件使用的軟體開發套件版本 |
| `minGatewayVersion` | 最低版本號 | 如果插件需要閘道功能，聲明最低要求的閘道版本 |

### 9.2 驗證結果型別

```typescript
// source-repo/packages/plugin-package-contract/src/index.ts:18-21
export type ExternalCodePluginValidationResult = {
  compatibility?: ExternalPluginCompatibility;
  issues: ExternalPluginValidationIssue[];
};
```

驗證結果包含兩部分：成功解析出的相容性資訊（如果格式正確的話），以及發現的所有問題清單。問題可能是不同嚴重程度的——警告（Warning）表示建議修正但不阻擋發布，錯誤（Error）表示必須修正才能通過驗證。

### 9.3 必要欄位定義

```typescript
// source-repo/packages/plugin-package-contract/src/index.ts:23-26
export const EXTERNAL_CODE_PLUGIN_REQUIRED_FIELD_PATHS = [
  "openclaw.compat.pluginApi",
  "openclaw.build.openclawVersion",
] as const;
```

外部插件的 `package.json` 中必須包含以上兩個欄位路徑。這是最低限度的強制要求——每個外部插件都必須明確聲明它支援的插件應用程式介面版本（`openclaw.compat.pluginApi`）和建置時的 OpenClaw 版本（`openclaw.build.openclawVersion`）。有了這些資訊，系統就能在載入之前就判斷此插件是否與當前環境相容，避免載入不相容的插件導致執行時期錯誤。

### 9.4 驗證工具函式

驗證合約提供了三個層次分明的工具函式：

| 函式名稱 | 原始碼位置 | 職責 |
|---|---|---|
| `normalizeExternalPluginCompatibility()` | `source-repo/packages/plugin-package-contract/src/index.ts:37-66` | 從原始的 `package.json` 內容中提取並正規化相容性資訊為 `ExternalPluginCompatibility` 型別 |
| `listMissingExternalCodePluginFieldPaths()` | `source-repo/packages/plugin-package-contract/src/index.ts:68-78` | 檢查 `package.json` 中是否缺少必要欄位，回傳缺失欄位的路徑列表 |
| `validateExternalCodePluginPackageJson()` | `source-repo/packages/plugin-package-contract/src/index.ts:80-91` | 組合以上兩個函式，執行完整的驗證流程，回傳包含相容性資訊和問題列表的結構化結果 |

三個函式的層次關係是：`validateExternalCodePluginPackageJson()` 是最高階的入口，它在內部呼叫 `normalizeExternalPluginCompatibility()` 來提取資訊，並呼叫 `listMissingExternalCodePluginFieldPaths()` 來檢查必要欄位。這種分層設計使得開發者可以根據需求選擇呼叫適當層級的函式。

---

## 10. 實際範例：memory-core 擴充套件

理論知識需要透過實際範例來鞏固。以下是 OpenClaw 內建的 `memory-core` 擴充套件的入口檔案，它展示了一個完整插件的標準結構：

```typescript
// source-repo/extensions/memory-core/index.ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "memory-core",
  name: "Memory (Core)",
  description: "File-backed memory search tools and CLI",
  kind: "memory",
  register(api) {
    registerBuiltInMemoryEmbeddingProviders(api);
    registerShortTermPromotionDreaming(api);
    registerDreamingCommand(api);
    api.registerMemoryCapability({...});
    api.registerTool(...);
    api.registerCli(...);
  },
});
```

讓我們逐行解析這個範例，理解每一部分的設計意圖：

**第一行**——匯入入口函式：從軟體開發套件的固定路徑 `openclaw/plugin-sdk/plugin-entry` 匯入 `definePluginEntry`。注意這裡使用的是路徑別名，實際的解析由載入器的別名系統處理。

**預設匯出**——呼叫入口函式並匯出結果：`export default definePluginEntry({...})` 這個模式確保載入器在匯入此檔案時，能透過預設匯出取得插件的定義物件。

**元資料欄位**：
- `id: "memory-core"`——全域唯一的插件識別碼
- `name: "Memory (Core)"`——人類可讀的顯示名稱
- `description: "File-backed memory search tools and CLI"`——功能描述
- `kind: "memory"`——插件的類型分類

**`register(api)` 方法**——這是插件的核心。當載入器成功匯入此插件後，會呼叫此方法並傳入 `api` 物件。插件在此方法中完成所有的能力註冊：

| 註冊呼叫 | 註冊的能力類型 | 功能說明 |
|---|---|---|
| `registerBuiltInMemoryEmbeddingProviders(api)` | 記憶嵌入提供者 | 註冊內建的文字向量嵌入演算法，用於記憶的語意搜尋功能 |
| `registerShortTermPromotionDreaming(api)` | 內部處理邏輯 | 註冊「做夢」機制——在代理空閒時自動整理和提升短期記憶為長期記憶 |
| `registerDreamingCommand(api)` | 命令列指令 | 註冊 `dream` 命令列指令，讓使用者可以手動觸發記憶整理流程 |
| `api.registerMemoryCapability({...})` | 記憶能力 | 定義完整的記憶操作能力，包括如何儲存、檢索、更新、刪除記憶條目 |
| `api.registerTool(...)` | 代理工具 | 為代理提供記憶相關的工具函式，使代理在對話中可以主動搜尋和引用記憶 |
| `api.registerCli(...)` | 命令列介面 | 為使用者提供記憶相關的命令列工具，用於管理和診斷記憶系統 |

這個範例清楚地展示了一個中等複雜度的插件可以同時提供多種能力，以及如何透過 `api` 物件將這些能力統一註冊到系統中。

---

## 11. 完整生命週期流程（Full Lifecycle）

現在讓我們將以上所有元件串連起來，完整呈現一個插件從被發現到被卸載的五個階段：

### 階段一：發現（Discovery）

- **執行者**：`discovery.ts` 模組
- **輸入**：指定的掃描目錄路徑
- **過程**：遍歷目錄樹，根據副檔名規則和忽略清單過濾檔案，辨識潛在的插件候選
- **輸出**：`PluginDiscoveryResult` 物件，包含 `PluginCandidate[]` 候選列表和 `PluginDiagnostic[]` 診斷資訊
- **快取**：結果快取一千毫秒，可透過 `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` 環境變數調整

### 階段二：載入（Load）

- **執行者**：`loader.ts` 模組（兩千兩百一十二行）
- **輸入**：`PluginCandidate[]` 候選插件列表
- **過程**：解析每個候選的 `openclaw.plugin.json` 清單 → 驗證版本相容性 → 使用 `jiti` 即時轉譯並匯入入口檔案 → 呼叫 `createPluginRegistry()` 建立新的註冊中心
- **輸出**：已載入並準備好進行註冊的插件實例集合
- **關鍵技術**：`jiti` 即時轉譯、軟體開發套件路徑別名解析

### 階段三：註冊（Register）

- **執行者**：`registry.ts` 模組
- **輸入**：已載入的插件實例
- **過程**：依序呼叫每個插件的 `register(api)` 方法，插件透過 `api` 物件將自身的能力注入到註冊中心
- **輸出**：完整填充的 `PluginRegistry` 實例
- **註冊的能力類型**：指令（`registerPluginCommand`）、壓縮提供者（`registerCompactionProvider`）、記憶嵌入提供者（`registerMemoryEmbeddingProvider`）、記憶能力（`registerMemoryCapability`）、鉤子、工具、通道等

### 階段四：執行（Execute）

- **執行者**：`hooks.ts` 模組（鉤子派發）和 `provider-runtime.ts` 模組（提供者執行）
- **輸入**：系統事件（例如收到使用者訊息、代理即將回覆等）
- **過程**：根據事件類型，查找對應的已註冊鉤子 → 按優先順序依序執行 → 處理非同步結果和錯誤
- **輸出**：鉤子的執行結果可能修改事件的內容或行為
- **關鍵行為**：每次呼叫 `setActivePluginRegistry()` 時，`activeVersion` 遞增，所有表面狀態同步更新

### 階段五：卸載（Unload）

- **執行者**：`runtime.ts:232-241` 中的 `resetPluginRuntimeStateForTest()` 函式
- **輸入**：無
- **過程**：將 `RegistryState` 的所有欄位重置為初始值
- **輸出**：乾淨的、未初始化的執行時期狀態
- **備註**：此函式主要用於測試場景下的狀態清理，確保測試之間不會互相干擾

### 生命週期流程圖

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Discovery  │───▶│    Load     │───▶│  Register   │───▶│   Execute   │───▶│   Unload    │
│ discovery.ts│    │  loader.ts  │    │ registry.ts │    │  hooks.ts + │    │ runtime.ts  │
│             │    │             │    │             │    │ provider-   │    │ :232-241    │
│ 掃描目錄    │    │ jiti 轉譯   │    │ 註冊能力    │    │ runtime.ts  │    │ 狀態重置    │
│ 快取結果    │    │ 驗證相容性  │    │ 連接子系統  │    │ 鉤子派發    │    │             │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

---

## 12. 設計模式與架構洞察

綜觀整個插件系統的設計，可以辨識出以下七個核心設計模式。理解這些模式有助於在開發插件或進行系統級除錯時，快速定位相關的程式碼邏輯：

1. **註冊表模式（Registry Pattern）**：`PluginRegistry` 作為中央註冊表，所有子系統透過它來註冊和查詢能力。這是整個插件系統的骨幹模式。

2. **全域符號單例（Global Symbol Singleton）**：使用 `Symbol.for("openclaw.pluginRegistryState")` 確保跨模組實例的狀態唯一性，解決了 Node.js 環境中模組重複載入的問題。

3. **延遲載入（Lazy Loading）**：透過清單中的啟動條件（Activation）配置，控制插件的載入時機，只在需要時才載入對應的插件，大幅優化了啟動效能。

4. **樂觀鎖（Optimistic Locking）**：`activeVersion` 版本計數器使得其他元件可以偵測註冊中心是否在操作期間被替換，確保在併發環境中的資料一致性。

5. **表面釘選（Surface Pinning）**：在超文本傳輸協定請求處理期間凍結路由表面，防止進行中的請求因註冊中心更換而看到不一致的狀態。

6. **時間快取策略（Time-based Caching）**：發現結果的一秒快取，在效能（避免重複掃描）和即時性（反映最新變更）之間取得了良好的平衡，且可透過環境變數自由調整。

7. **合約驗證（Contract Validation）**：透過獨立的驗證合約套件定義外部插件必須滿足的最低品質標準，確保插件生態系統的穩定性和相容性。

---

## 引用來源

以下是本章所有引用的原始碼檔案與對應行號，供讀者進行交叉查驗：

| 引用編號 | 檔案路徑 | 行號範圍 | 內容說明 |
|---|---|---|---|
| M-1 | `source-repo/src/plugins/manifest.ts` | 17 | 插件清單檔案名稱常數 `PLUGIN_MANIFEST_FILENAME` 的定義 |
| M-2 | `source-repo/src/plugins/manifest.ts` | 3 | JSON5 套件的匯入語句 |
| M-3 | `source-repo/src/plugins/manifest.ts` | 20-27 | `PluginManifestChannelConfig` 通道配置型別的定義 |
| M-4 | `source-repo/src/plugins/manifest.ts` | 29-40 | `PluginManifestModelSupport` 模型支援型別的定義 |
| M-5 | `source-repo/src/plugins/manifest.ts` | 42 | `PluginManifestActivationCapability` 啟動能力聯合型別的定義 |
| M-6 | `source-repo/src/plugins/manifest.ts` | 44-58 | `PluginManifestActivation` 啟動條件型別的定義 |
| M-7 | `source-repo/src/plugins/manifest.ts` | 69-80 | `PluginManifestSetup` 安裝設定型別的定義 |
| RS-1 | `source-repo/src/plugins/runtime-state.ts` | 3 | `PLUGIN_REGISTRY_STATE` 全域符號的定義 |
| RS-2 | `source-repo/src/plugins/runtime-state.ts` | 13-22 | `RegistryState` 完整型別的定義 |
| RT-1 | `source-repo/src/plugins/runtime.ts` | 76-89 | `setActivePluginRegistry()` 活躍註冊中心設定函式 |
| RT-2 | `source-repo/src/plugins/runtime.ts` | 91-93 | `getActivePluginRegistry()` 活躍註冊中心查詢函式 |
| RT-3 | `source-repo/src/plugins/runtime.ts` | 99-107 | `requireActivePluginRegistry()` 強制取得註冊中心函式 |
| RT-4 | `source-repo/src/plugins/runtime.ts` | 109-118 | 超文本傳輸協定路由表面的釘選與釋放函式 |
| RT-5 | `source-repo/src/plugins/runtime.ts` | 232-241 | `resetPluginRuntimeStateForTest()` 測試狀態清理函式 |
| D-1 | `source-repo/src/plugins/discovery.ts` | 24-36 | 掃描規則常數：支援的副檔名和忽略的目錄名稱 |
| D-2 | `source-repo/src/plugins/discovery.ts` | 38-54 | `PluginCandidate` 候選插件完整型別定義 |
| D-3 | `source-repo/src/plugins/discovery.ts` | 56-59 | `PluginDiscoveryResult` 發現結果型別定義 |
| D-4 | `source-repo/src/plugins/discovery.ts` | 61-64 | 發現快取的資料結構和預設有效期定義 |
| D-5 | `source-repo/src/plugins/discovery.ts` | 70-83 | `resolveDiscoveryCacheMs()` 快取時間環境變數解析函式 |
| L-1 | `source-repo/src/plugins/loader.ts` | 4 | `jiti` 即時轉譯工具的匯入語句 |
| L-2 | `source-repo/src/plugins/loader.ts` | 46 | `discoverOpenClawPlugins` 發現函式的匯入語句 |
| L-3 | `source-repo/src/plugins/loader.ts` | 75 | `createPluginRegistry` 註冊中心建立函式的匯入語句 |
| L-4 | `source-repo/src/plugins/loader.ts` | 82-83 | 執行時期管理函式（設定與取得活躍註冊中心）的匯入語句 |
| L-5 | `source-repo/src/plugins/loader.ts` | 88-99 | 軟體開發套件路徑別名解析系統的實作位置 |
| H-1 | `source-repo/src/plugins/hook-types.ts` | 55-84 | `PluginHookName` 三十種鉤子名稱的完整聯合型別定義 |
| H-2 | `source-repo/src/plugins/hooks.ts` | 1-6 | 鉤子執行器模組的文件註解（描述核心設計原則） |
| SDK-1 | `source-repo/src/plugin-sdk/core.ts` | 29-80 | 軟體開發套件核心模組的型別匯出清單 |
| SDK-2 | `source-repo/src/plugin-sdk/plugin-entry.ts` | 1-20 | `definePluginEntry` 插件入口函式的定義位置 |
| PC-1 | `source-repo/packages/plugin-package-contract/src/index.ts` | 6-11 | `ExternalPluginCompatibility` 相容性資訊型別定義 |
| PC-2 | `source-repo/packages/plugin-package-contract/src/index.ts` | 18-21 | `ExternalCodePluginValidationResult` 驗證結果型別定義 |
| PC-3 | `source-repo/packages/plugin-package-contract/src/index.ts` | 23-26 | `EXTERNAL_CODE_PLUGIN_REQUIRED_FIELD_PATHS` 必要欄位常數 |
| PC-4 | `source-repo/packages/plugin-package-contract/src/index.ts` | 37-91 | 三個驗證工具函式的實作範圍 |
| REG-1 | `source-repo/src/plugins/registry.ts` | 1-76 | 註冊中心核心模組的匯入宣告與主要註冊函式 |
| EXT-1 | `source-repo/extensions/memory-core/index.ts` | — | `memory-core` 內建記憶擴充套件的完整入口檔案 |
