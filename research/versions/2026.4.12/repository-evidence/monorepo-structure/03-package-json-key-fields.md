# 03 — package.json 關鍵欄位分析

> **證據層級**：Repository Evidence
> **對應版本**：OpenClaw 2026.4.12
> **主要來源**：`source-repo/package.json:1-1523`

---

## 本章摘要

本章深入分析 OpenClaw 根 `package.json` 的關鍵欄位。這個長達 1,523 行的 `package.json` 不只是一份依賴清單——它是 OpenClaw 架構決策的集中體現。從 `name` 和 `version` 可以看出發佈策略；從 `bin` 和 `main` 可以看出入口點設計；從 `files` 可以看出發佈邊界；從 `exports` 的 264 條路徑可以看出 Plugin SDK 的 API 表面積；從 `dependencies` 和 `devDependencies` 可以看出技術棧的全貌。這份 1,523 行的 JSON 是理解 OpenClaw「它把自己當成什麼」的最佳起點。

---

## 1. 身份與元資料欄位

### 1.1 核心身份

```json
{
  "name": "openclaw",
  "version": "2026.4.12",
  "description": "Multi-channel AI gateway with extensible messaging integrations",
  "license": "MIT"
}
```

#### 逐欄位分析

| 欄位 | 值 | 意義 |
|------|-----|------|
| `name` | `"openclaw"` | 這是發佈到 npm 時的套件名稱。沒有 scope（如 `@openclaw/`），表示它佔用的是 npm 的全域命名空間。 |
| `version` | `"2026.4.12"` | 使用 **CalVer**（日曆版本）而非 SemVer（語意版本）。格式為 `YYYY.M.D`，表示這是 2026 年 4 月 12 日的版本。 |
| `description` | `"Multi-channel AI gateway with extensible messaging integrations"` | 官方自我定義：**多通道 AI 閘道器，具有可擴充的訊息整合**。 |
| `license` | `"MIT"` | MIT 授權——最寬鬆的開源授權之一，允許商業使用。 |

#### 版本策略深入分析

CalVer 版本號 `2026.4.12` 表明：

1. **不承諾向後相容性**：SemVer 的 major 版本號變更暗示 breaking changes，CalVer 則不帶這種語意。使用者不能從版本號推斷 API 是否有破壞性變更。
2. **發佈節奏以時間為基準**：版本號直接對應日期，暗示可能採用滾動發佈（rolling release）或定期發佈的模式。
3. **與 workspace 子套件的對比**：`packages/` 下的三個子套件版本都是 `0.0.0-private`，表示它們不獨立發佈。只有根套件 `openclaw` 使用有意義的版本號。

### 1.2 模組系統

```json
{
  "type": "module",
  "main": "dist/index.js"
}
```

| 欄位 | 值 | 意義 |
|------|-----|------|
| `type` | `"module"` | 使用 **ESM**（ES Modules）而非 CJS（CommonJS）作為預設模組系統。所有 `.js` 檔案預設以 ESM 解析。 |
| `main` | `"dist/index.js"` | 傳統的套件入口點。當其他程式碼 `import "openclaw"` 時，解析到 `dist/index.js`。注意這是編譯後（`dist/`）的路徑，不是原始碼（`src/`）。 |

---

## 2. 入口點與可執行檔

### 2.1 `bin` 欄位

```json
{
  "bin": {
    "openclaw": "openclaw.mjs"
  }
}
```

**含義**：當使用者透過 npm 全域安裝 OpenClaw（`npm install -g openclaw`）或在專案中安裝後，系統會建立一個名為 `openclaw` 的可執行命令，指向 `openclaw.mjs` 這個檔案。

**技術細節**：

- 檔案使用 `.mjs` 副檔名，明確標記為 ES Module（即使在 `"type": "module"` 的套件中，`.mjs` 仍然是一種明確的宣告）
- `openclaw.mjs` 是一個位於倉庫根目錄的檔案——它是整個應用程式的 CLI 入口腳本
- 這暗示了 OpenClaw 的主要使用方式是**命令列工具**（CLI）——使用者安裝後透過 `openclaw` 命令來操作

### 2.2 入口點的二元性

OpenClaw 同時定義了 `main`（程式庫入口）和 `bin`（CLI 入口），表示它既是一個**可以被其他程式引用的函式庫**，也是一個**獨立的命令列工具**。這種雙入口設計在成熟的 Node.js 專案中很常見（類似 `eslint`、`webpack` 等工具）。

---

## 3. `files` 欄位——發佈邊界定義

### 3.1 完整內容

```json
{
  "files": [
    "CHANGELOG.md", "LICENSE", "openclaw.mjs", "README.md",
    "assets/", "dist/", "!dist/**/*.map",
    "!dist/plugin-sdk/.tsbuildinfo",
    "!dist/extensions/qa-channel/**",
    "dist/extensions/qa-channel/runtime-api.js",
    "!dist/extensions/qa-lab/**",
    "dist/extensions/qa-lab/runtime-api.js",
    "docs/", "!docs/.generated/**", "!docs/.i18n/zh-CN.tm.jsonl",
    "qa/scenarios/",
    "skills/",
    "scripts/npm-runner.mjs",
    "scripts/postinstall-bundled-plugins.mjs",
    "scripts/windows-cmd-helpers.mjs"
  ]
}
```

### 3.2 逐項解析

`files` 欄位告訴 npm：「當執行 `npm publish` 時，只包含這些檔案和目錄。」以 `!` 開頭的是排除規則。

#### 包含的內容

| 項目 | 用途 |
|------|------|
| `CHANGELOG.md` | 變更日誌 |
| `LICENSE` | 授權文件 |
| `openclaw.mjs` | CLI 入口腳本 |
| `README.md` | 說明文件 |
| `assets/` | 靜態資源 |
| `dist/` | 編譯後的 JavaScript/TypeScript 輸出 |
| `docs/` | 文件（會隨套件一起安裝） |
| `qa/scenarios/` | QA 測試場景 |
| `skills/` | **所有 53 個技能包** |
| `scripts/npm-runner.mjs` | npm 執行器腳本 |
| `scripts/postinstall-bundled-plugins.mjs` | 安裝後的綁定插件處理腳本 |
| `scripts/windows-cmd-helpers.mjs` | Windows 命令列輔助腳本 |

#### 排除的內容

| 排除規則 | 排除對象 | 原因推測 |
|----------|---------|---------|
| `!dist/**/*.map` | Source Map 檔案 | 減少發佈大小，source map 只在開發時需要 |
| `!dist/plugin-sdk/.tsbuildinfo` | TypeScript 建置快取 | 建置中間產物，不需要發佈 |
| `!dist/extensions/qa-channel/**` | QA 通道擴充套件的全部 dist | QA 專用，不應發佈給終端使用者 |
| `dist/extensions/qa-channel/runtime-api.js` | 但保留 runtime-api.js | runtime API 仍需要被其他程式碼引用 |
| `!dist/extensions/qa-lab/**` | QA 實驗室擴充套件的全部 dist | 同上 |
| `dist/extensions/qa-lab/runtime-api.js` | 但保留 runtime-api.js | 同上 |
| `!docs/.generated/**` | 自動生成的文件 | 減少大小，這些可以重新生成 |
| `!docs/.i18n/zh-CN.tm.jsonl` | 簡體中文翻譯記憶 | 翻譯中間產物 |

### 3.3 關鍵架構觀察

#### `skills/` 是隨套件發佈的

技能包被完整包含在發佈中。這意味著：

1. 當使用者 `npm install openclaw` 時，53 個技能包會全部下載
2. 技能包不需要單獨安裝——它們是「內建」的
3. 這與 `skills/` 不是 workspace 成員一致——它們是**隨主套件一起發佈的資產**，不是獨立的 npm 套件

#### `docs/` 也隨套件發佈

文件不僅在網站上，也隨套件一起安裝。這可能是為了支援離線文件查閱或 CLI 內建的文件功能。

#### `postinstall-bundled-plugins.mjs` 的存在

這個腳本名稱暗示了一個安裝後處理機制——可能在使用者安裝 OpenClaw 後，自動設定或連結綁定的插件（bundled plugins）。

---

## 4. `exports` 欄位——Plugin SDK 的 API 表面積

### 4.1 規模與結構

根 `package.json` 定義了 **264 個 export 路徑**（subpath exports）。這些路徑大多以 `./plugin-sdk/` 為前綴。每條路徑都有兩個目標：

```json
{
  "exports": {
    "./plugin-sdk/core": {
      "types": "./dist/plugin-sdk/core.d.ts",
      "default": "./dist/plugin-sdk/core.js"
    },
    "./plugin-sdk/provider-entry": {
      "types": "./dist/plugin-sdk/provider-entry.d.ts",
      "default": "./dist/plugin-sdk/provider-entry.js"
    }
  }
}
```

| 條件 | 目標 | 用途 |
|------|------|------|
| `types` | `./dist/plugin-sdk/*.d.ts` | TypeScript 型別定義，供 IDE 和 TypeScript 編譯器使用 |
| `default` | `./dist/plugin-sdk/*.js` | JavaScript 運行時程式碼 |

### 4.2 `exports` 的架構意義

264 個 export 路徑意味著 OpenClaw 的 Plugin SDK 有一個**極大的 API 表面積**。這些路徑定義了「擴充套件可以從核心引擎引用什麼」。

#### 為什麼用根套件的 `exports` 而非 `@openclaw/plugin-sdk`？

`packages/plugin-sdk/` 作為一個 workspace 套件確實存在，但 264 個 export 路徑定義在根套件 `openclaw` 上。這意味著擴充套件開發者引用 SDK 的方式是：

```typescript
// 從根套件引用 Plugin SDK
import { something } from "openclaw/plugin-sdk/core";
```

而不是：

```typescript
// ❌ 不是這樣
import { something } from "@openclaw/plugin-sdk/core";
```

這種設計的好處是：

1. **單一依賴**：擴充套件只需要依賴 `openclaw` 一個套件，不需要額外安裝 `@openclaw/plugin-sdk`
2. **版本一致性**：SDK 的版本與核心引擎的版本綁定在一起，不會出現版本不匹配的問題
3. **發佈簡化**：只需要發佈一個套件到 npm

### 4.3 Export 路徑的分類推測

雖然我們沒有完整列出全部 264 條路徑，但從已知的部分可以推測分類：

| 路徑前綴 | 推測功能 |
|---------|---------|
| `./plugin-sdk/core` | 核心 SDK 介面 |
| `./plugin-sdk/plugin-runtime` | 插件運行時環境 |
| `./plugin-sdk/provider-entry` | 供應商適配器入口點 |
| `./plugin-sdk/*`（其他） | 各種 SDK 子模組，可能對應 `src/plugin-sdk/` 下的各個檔案 |

264 條路徑的存在表明 Plugin SDK 的模組化程度非常高——每個功能點都可以被單獨引入，支援 tree-shaking（搖樹優化），避免擴充套件引入不需要的程式碼。

---

## 5. `dependencies` 分析——生產依賴

### 5.1 概述

根 `package.json` 列出了 **70 個生產依賴**。這些是 OpenClaw 在運行時需要的外部套件。

### 5.2 依賴分類與技術棧

70 個生產依賴可分為以下主要類別：

| 類別 | 代表性依賴 | 版本 |
|------|-----------|------|
| Web 框架 | Express | 5.x（最新大版本） |
| WebSocket | ws | 8.x |
| 資料驗證 | Zod | 4.x（最新大版本） |
| CLI 框架 | commander | 14.x |
| AI/LLM SDK | openai、@anthropic-ai/vertex-sdk | 6.x |
| 向量資料庫 | @lancedb/lancedb | — |
| 通訊平台 SDK | grammy（Telegram）、matrix-js-sdk | — |
| 瀏覽器自動化 | playwright-core | — |
| 影像處理 | sharp | — |

### 5.3 依賴版本的激進程度

OpenClaw 使用了非常激進的版本策略——Express 5、Zod 4、OpenAI SDK 6、TypeScript 6 都是各自生態系中最新的大版本。這與 CalVer 版本號、TypeScript native-preview 的使用一致——OpenClaw 團隊偏好「跑在技術前沿」。

### 5.4 規模觀察

70 個生產依賴對於 OpenClaw 的功能範圍來說相當克制。大部分平台特定的 SDK 被放在 `extensions/` 中各自的 `package.json` 裡，而非根套件——這是 monorepo 架構帶來的依賴分散化好處。

---

## 6. `devDependencies` 分析——開發依賴

### 6.1 概述

根 `package.json` 列出了 **23 個開發依賴**。

### 6.2 工具鏈分類

| 類別 | 依賴 | 角色 |
|------|------|------|
| 語言 | `typescript` 6.x、`@typescript/native-preview` 7.x | 編譯器（同時使用傳統版與原生版） |
| 測試 | `vitest` 4.x | Vite 驅動的測試框架 |
| 打包 | `tsdown`、`tsx` | TS 打包與直接執行 |
| 程式碼品質 | `oxlint`、`oxfmt`、`jscpd`、`madge` | Linting、格式化、重複檢測、循環依賴檢測 |

### 6.3 原生工具偏好

OpenClaw 的工具鏈有明顯趨勢——**偏好 Rust/Go 原生實作**：用 `oxlint` 取代 ESLint、`oxfmt` 取代 Prettier、`tsdown`（基於 Rolldown）取代 webpack、`@typescript/native-preview` 取代傳統 tsc。對擁有 116 個 workspace 套件的大型 monorepo 來說，工具速度至關重要。

---

## 7. Workspace 子套件的 package.json

### 7.1 `@openclaw/plugin-sdk`

**位置**：`source-repo/packages/plugin-sdk/package.json`

```json
{
  "name": "@openclaw/plugin-sdk",
  "version": "0.0.0-private",
  "private": true,
  "type": "module"
}
```

**Export 路徑**：約 40 個子路徑，例如：

| 路徑 | 對應原始碼 | 推測功能 |
|------|----------|---------|
| `./core` | `src/core.ts` | SDK 核心型別與介面 |
| `./plugin-runtime` | `src/plugin-runtime.ts` | 插件運行時環境 |
| `./provider-entry` | `src/provider-entry.ts` | 供應商適配器的入口點定義 |

每條 export 路徑的結構為：

```json
{
  "./core": {
    "types": "./dist/core.d.ts",
    "default": "./src/core.ts"
  }
}
```

注意 `default` 指向 `./src/core.ts`（原始碼）而非 `./dist/core.js`（編譯後）。這在 workspace 內部是合理的——workspace 成員之間可以直接引用原始碼，pnpm 和 TypeScript 都能解析 symlink。

### 7.2 `@openclaw/memory-host-sdk`

**位置**：`source-repo/packages/memory-host-sdk/package.json`

```json
{
  "name": "@openclaw/memory-host-sdk",
  "version": "0.0.0-private",
  "private": true,
  "type": "module"
}
```

**Export 路徑**：12 個子路徑：

| 路徑 | 功能域 |
|------|--------|
| `./runtime` | 記憶運行時 |
| `./runtime-core` | 運行時核心 |
| `./runtime-cli` | CLI 運行時 |
| `./runtime-files` | 檔案運行時 |
| `./engine` | 記憶引擎 |
| `./engine-foundation` | 引擎基礎 |
| `./engine-storage` | 引擎儲存層 |
| `./engine-embeddings` | 引擎嵌入向量 |
| `./engine-qmd` | 引擎 QMD（可能是查詢-記憶-文件的縮寫） |
| `./multimodal` | 多模態記憶 |
| `./query` | 記憶查詢 |
| `./secret` | 機密資料存取 |
| `./status` | 記憶系統狀態 |

這個 SDK 的結構揭示了 OpenClaw 記憶系統的架構：

1. **Runtime/Engine 分離**：`runtime-*` 和 `engine-*` 的命名暗示記憶系統有「運行時」和「引擎」兩層抽象
2. **儲存與嵌入分離**：`engine-storage` 和 `engine-embeddings` 表示向量嵌入和持久化儲存是分開的
3. **多模態支援**：`multimodal` 路徑表示記憶系統不僅處理文字，也處理圖像、音訊等多模態資料

### 7.3 `@openclaw/plugin-package-contract`

**位置**：`source-repo/packages/plugin-package-contract/package.json`

```json
{
  "name": "@openclaw/plugin-package-contract",
  "version": "0.0.0-private",
  "private": true,
  "type": "module"
}
```

**Export 路徑**：僅 1 個：

```json
{
  ".": {
    "types": "./dist/index.d.ts",
    "default": "./src/index.ts"
  }
}
```

這是最小的套件——它只定義一件事：**一個插件套件需要符合什麼合約**。這種「合約套件」（contract package）模式在大型系統中很常見，用於確保核心引擎載入插件時有型別安全的保障。

### 7.4 三個套件的共同特徵

| 特徵 | 值 | 意義 |
|------|-----|------|
| `version` | `"0.0.0-private"` | 不會被發佈到 npm，僅在 workspace 內使用 |
| `private` | `true` | npm publish 會拒絕發佈此套件 |
| `type` | `"module"` | 全部使用 ESM |

---

## 8. 整體架構推論

### 8.1 「單一套件發佈」策略

綜合以上觀察，OpenClaw 採用的是「**單一套件發佈**」（single-package publish）策略：

```
npm install openclaw
         │
         ├── 核心引擎（dist/）
         ├── Plugin SDK（dist/plugin-sdk/，透過 264 個 exports 路徑暴露）
         ├── 111 個擴充套件（dist/extensions/）
         ├── 53 個技能包（skills/）
         ├── 文件（docs/）
         ├── QA 場景（qa/scenarios/）
         ├── 靜態資源（assets/）
         └── 輔助腳本（scripts/）
```

使用者只需安裝一個套件就獲得了完整的 OpenClaw。

### 8.2 SDK 的雙重暴露

Plugin SDK 透過兩種方式暴露：

1. **在 workspace 內部**：透過 `@openclaw/plugin-sdk` 套件（symlink），`default` 指向原始碼
2. **在發佈後**：透過根套件 `openclaw` 的 `exports` 欄位（264 條路徑），`default` 指向編譯後的程式碼

這是一種巧妙的設計——workspace 內的開發者享受到直接引用原始碼的便利（即時更新、支援 IDE 跳轉到定義），而外部使用者則透過編譯後的 JavaScript 獲得穩定的 API。

### 8.3 技術棧速覽

| 層面 | 選擇 |
|------|------|
| 語言 | TypeScript 6 + native-preview 7 |
| 模組 | ESM |
| 套件管理 | pnpm workspace (hoisted) |
| HTTP | Express 5 |
| 驗證 | Zod 4 |
| CLI | commander 14 |
| 測試 | vitest 4 |
| 打包 | tsdown（Rolldown-based） |
| Lint/Format | oxlint + oxfmt（Rust） |

---

## 9. 值得進一步調查的問題

1. **`openclaw.mjs` 的內容**：CLI 入口腳本的具體邏輯？
2. **264 個 exports 完整清單**：每條路徑暴露什麼 API？
3. **`scripts/postinstall-bundled-plugins.mjs`**：安裝後如何處理綁定插件？
4. **`scripts` 欄位**：定義了哪些建置命令？
5. **`pnpm.overrides`**：是否使用了依賴覆寫？

---

## 引用來源

| 檔案路徑 | 行號範圍 | 本文引用位置 | 內容說明 |
|----------|----------|-------------|----------|
| `source-repo/package.json` | 1-5 | §1.1 | name, version, description, license 欄位 |
| `source-repo/package.json` | 6-7 | §1.2 | type, main 欄位 |
| `source-repo/package.json` | 8-10 | §2.1 | bin 欄位 |
| `source-repo/package.json` | 11-28 | §3.1 | files 欄位完整內容 |
| `source-repo/package.json` | exports 區段 | §4 | 264 個 export 路徑 |
| `source-repo/package.json` | dependencies 區段 | §5 | 70 個生產依賴 |
| `source-repo/package.json` | devDependencies 區段 | §6 | 23 個開發依賴 |
| `source-repo/packages/plugin-sdk/package.json` | 全文 | §7.1 | plugin-sdk 套件定義 |
| `source-repo/packages/memory-host-sdk/package.json` | 全文 | §7.2 | memory-host-sdk 套件定義 |
| `source-repo/packages/plugin-package-contract/package.json` | 全文 | §7.3 | plugin-package-contract 套件定義 |
| `source-repo/.npmrc` | 1-4 | §6.3（交叉引用） | node-linker=hoisted 的原因 |
| `source-repo/pnpm-workspace.yaml` | 1-4 | §8.1（交叉引用） | workspace 成員定義 |
