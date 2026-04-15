# 01 — TypeScript 編譯設定詳解（tsconfig Configuration）

> **層級**：Repository Evidence · Topic 4 · Chapter 01  
> **版本快照**：2026.4.12  
> **最後更新**：2026-07-21

---

## 本章摘要

OpenClaw 專案使用三份 `tsconfig.json` 設定檔來管控不同面向的 TypeScript 編譯行為。主設定檔 `tsconfig.json` 定義了整個專案的基準編譯選項，包括 ES2023 目標、NodeNext 模組解析、嚴格模式，以及一套精心設計的路徑別名（Path Aliases）系統；第二份 `tsconfig.plugin-sdk.dts.json` 專門用於產生 Plugin SDK 的型別宣告檔（`.d.ts`）；第三份 `tsconfig.oxlint.json` 則為 oxlint 靜態分析工具擴展了檔案範圍。三份設定檔以繼承（extends）方式串接，形成層級化的設定架構。

本章逐欄位解析每一份設定檔的技術細節，說明各選項在 OpenClaw 專案脈絡中的具體意義。

---

## 1. 主設定檔 — `tsconfig.json`

### 1.1 完整設定內容

```json
// source-repo/tsconfig.json:1-33
{
  "compilerOptions": {
    "allowImportingTsExtensions": true,
    "allowSyntheticDefaultImports": true,
    "declaration": true,
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "forceConsistentCasingInFileNames": true,
    "lib": ["DOM", "DOM.Iterable", "ES2023", "ScriptHost"],
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "noEmit": true,
    "noEmitOnError": true,
    "outDir": "dist",
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "strict": true,
    "target": "es2023",
    "types": ["node"],
    "useDefineForClassFields": false,
    "paths": {
      "openclaw/extension-api": ["./src/extensionAPI.ts"],
      "openclaw/plugin-sdk": ["./src/plugin-sdk/index.ts"],
      "openclaw/plugin-sdk/*": ["./src/plugin-sdk/*.ts"],
      "@openclaw/plugin-sdk": ["./src/plugin-sdk/index.ts"],
      "@openclaw/plugin-sdk/*": ["./src/plugin-sdk/*.ts"],
      "openclaw/plugin-sdk/account-id": ["./src/plugin-sdk/account-id.ts"],
      "@openclaw/*": ["./extensions/*"]
    }
  },
  "include": ["src/**/*", "ui/**/*", "extensions/**/*", "packages/**/*"],
  "exclude": ["node_modules", "dist", "**/dist/**"]
}
```

### 1.2 編譯目標與語言標準

| 選項 | 值 | 意義 |
|------|----|------|
| `target` | `"es2023"` | 編譯輸出目標為 ECMAScript 2023。這意味著 TypeScript 不會對 ES2023 及更早的語法特性進行降級轉譯（Downlevel Compilation），例如 `Array.prototype.findLast()`、`Array.prototype.toSorted()` 等 ES2023 新增方法將直接保留。這也代表 OpenClaw 的最低執行環境需支援 ES2023（Node.js 20+）。 |
| `lib` | `["DOM", "DOM.Iterable", "ES2023", "ScriptHost"]` | 載入的型別定義庫。`ES2023` 提供所有 ES2023 標準 API 的型別定義；`DOM` 和 `DOM.Iterable` 提供瀏覽器 DOM API 的型別定義——這暗示 OpenClaw 有部分程式碼涉及 UI 層（`ui/**/*` 目錄）；`ScriptHost` 提供 Windows Script Host 的型別定義，這在現代專案中較少見，可能是歷史遺留。 |

**觀察**：target 設為 `es2023` 而非更新的版本（如 `es2024` 或 `esnext`），說明專案有意識地選擇了一個穩定的 ES 版本作為基線，確保在主流 Node.js LTS 版本上都能執行。

### 1.3 模組系統設定

| 選項 | 值 | 意義 |
|------|----|------|
| `module` | `"NodeNext"` | 使用 Node.js 原生的模組解析策略。TypeScript 會根據 `package.json` 中的 `"type"` 欄位和副檔名來決定每個檔案是 ESM（`.mjs`）還是 CJS（`.cjs`）。這是 TypeScript 5.x 以來推薦的 Node.js 專案模組設定。 |
| `moduleResolution` | `"NodeNext"` | 與 `module: "NodeNext"` 配對使用。啟用 Node.js 的 ESM 解析演算法，包括 `package.json` 的 `"exports"` 欄位解析、條件匯出（Conditional Exports）等。 |
| `esModuleInterop` | `true` | 啟用 ES Module 互操作性（Interoperability）。當 ESM 程式碼匯入 CJS 模組時，TypeScript 會自動產生相容性包裝程式碼，讓 `import fs from 'node:fs'` 這類語法能正確運作（即使 CJS 模組沒有 `default` 匯出）。 |
| `allowSyntheticDefaultImports` | `true` | 允許從沒有明確 `default` 匯出的模組使用預設匯入語法。這是 `esModuleInterop` 的型別層面補充。 |
| `allowImportingTsExtensions` | `true` | 允許在匯入路徑中使用 `.ts` 副檔名（如 `import { foo } from './bar.ts'`）。此選項必須搭配 `noEmit: true` 或 `emitDeclarationOnly: true` 使用，因為 TypeScript 編譯器本身不會轉換這些路徑。在 OpenClaw 中，實際的打包由 tsdown 處理，因此這裡只做型別檢查。 |
| `resolveJsonModule` | `true` | 允許匯入 `.json` 檔案，TypeScript 會自動推斷其型別。 |

**設計解讀**：`module: "NodeNext"` + `moduleResolution: "NodeNext"` 的組合表明 OpenClaw 採用現代 Node.js ESM 策略。搭配 `allowImportingTsExtensions: true` 和 `noEmit: true`，說明 TypeScript 在此僅作為「型別檢查器」（Type Checker），實際的程式碼轉譯和打包完全交由 tsdown 負責。這是大型 TypeScript 專案的常見最佳實踐：分離型別檢查與打包職責。

### 1.4 嚴格性與品質控制

| 選項 | 值 | 意義 |
|------|----|------|
| `strict` | `true` | 啟用所有嚴格型別檢查選項的總開關。包括 `strictNullChecks`、`strictFunctionTypes`、`strictBindCallApply`、`strictPropertyInitialization`、`noImplicitAny`、`noImplicitThis`、`alwaysStrict` 等。這是 TypeScript 官方建議的最佳實踐。 |
| `forceConsistentCasingInFileNames` | `true` | 強制檔案路徑大小寫一致性。防止在大小寫不敏感的檔案系統（如 macOS 預設、Windows）上因路徑大小寫不一致而導致的難以追蹤的錯誤。 |
| `noEmitOnError` | `true` | 當存在型別錯誤時不產生輸出檔案。在主設定中搭配 `noEmit: true` 使用時，此選項對 tsc 本身無實際效果，但對繼承此設定的子設定（如 `tsconfig.plugin-sdk.dts.json`，其將 `noEmit` 覆寫為 `false`）仍然生效。 |

### 1.5 輸出控制

| 選項 | 值 | 意義 |
|------|----|------|
| `noEmit` | `true` | TypeScript 編譯器不產生任何輸出檔案（`.js`、`.d.ts` 等）。tsc 僅用於型別檢查。 |
| `declaration` | `true` | 產生 `.d.ts` 型別宣告檔。雖然在主設定中搭配 `noEmit: true` 使此選項無實際效果，但它被子設定繼承時會生效。 |
| `outDir` | `"dist"` | 輸出目錄。同上，雖然 `noEmit` 使此選項在主設定中無效，但子設定繼承後會使用。 |

**設計模式**：主設定檔中設置了 `declaration: true` 和 `outDir: "dist"`，但同時設置 `noEmit: true` 使其不生效。這是一種「預設值繼承」模式——子設定只需覆寫 `noEmit: false` 即可啟用輸出，無需重複設定 `declaration` 和 `outDir`。

### 1.6 裝飾器與類別欄位

| 選項 | 值 | 意義 |
|------|----|------|
| `experimentalDecorators` | `true` | 啟用 TypeScript 的實驗性裝飾器語法（Stage 2 提案版本，非 TC39 Stage 3 標準版本）。這表示 OpenClaw 中有使用 `@decorator` 語法的程式碼，可能用於依賴注入（Dependency Injection）、元資料標注等場景。 |
| `useDefineForClassFields` | `false` | 不使用 ECMAScript 標準的 `Object.defineProperty` 來定義類別欄位，改用舊式的賦值語法（Assignment Semantics）。這是為了與 `experimentalDecorators: true` 配合——實驗性裝飾器與 ES 標準的類別欄位行為不相容。 |

**重要觀察**：`experimentalDecorators: true` 與 `useDefineForClassFields: false` 必須搭配使用。如果 `useDefineForClassFields` 設為 `true`，裝飾器在修改類別屬性時的行為會出現問題。這是 TypeScript 社群中一個常見的相容性陷阱。

### 1.7 其他選項

| 選項 | 值 | 意義 |
|------|----|------|
| `skipLibCheck` | `true` | 跳過 `node_modules` 中 `.d.ts` 檔案的型別檢查。這可以大幅加速編譯速度，代價是不會發現第三方套件型別定義中的錯誤。對大型專案而言這是常見的效能優化。 |
| `types` | `["node"]` | 只自動引入 `@types/node` 的全域型別定義。其他 `@types/*` 套件需要明確匯入才能使用。這避免了型別命名空間汙染（Type Namespace Pollution）。 |

### 1.8 路徑別名系統（Path Aliases）

路徑別名是主設定檔中最值得深入分析的部分。OpenClaw 定義了一套精心設計的路徑映射：

```json
// source-repo/tsconfig.json:21-28
"paths": {
  "openclaw/extension-api": ["./src/extensionAPI.ts"],
  "openclaw/plugin-sdk": ["./src/plugin-sdk/index.ts"],
  "openclaw/plugin-sdk/*": ["./src/plugin-sdk/*.ts"],
  "@openclaw/plugin-sdk": ["./src/plugin-sdk/index.ts"],
  "@openclaw/plugin-sdk/*": ["./src/plugin-sdk/*.ts"],
  "openclaw/plugin-sdk/account-id": ["./src/plugin-sdk/account-id.ts"],
  "@openclaw/*": ["./extensions/*"]
}
```

#### 路徑別名分類

| 類型 | 別名模式 | 映射目標 | 用途 |
|------|----------|----------|------|
| Extension API | `openclaw/extension-api` | `./src/extensionAPI.ts` | 擴充套件 API 的進入點。外掛開發者透過此路徑匯入 OpenClaw 提供的擴充 API。 |
| Plugin SDK（主路徑） | `openclaw/plugin-sdk` | `./src/plugin-sdk/index.ts` | Plugin SDK 的主進入點。 |
| Plugin SDK（子路徑） | `openclaw/plugin-sdk/*` | `./src/plugin-sdk/*.ts` | Plugin SDK 的子模組。允許 `import { ... } from 'openclaw/plugin-sdk/auth'` 這類細粒度匯入。 |
| Plugin SDK（scoped 別名） | `@openclaw/plugin-sdk` | `./src/plugin-sdk/index.ts` | 與上方相同，但使用 npm scoped package 命名慣例（`@openclaw/`）。 |
| Plugin SDK（scoped 子路徑） | `@openclaw/plugin-sdk/*` | `./src/plugin-sdk/*.ts` | 同上的子路徑版本。 |
| 特定模組 | `openclaw/plugin-sdk/account-id` | `./src/plugin-sdk/account-id.ts` | 專門為 `account-id` 模組設置的精確路徑映射。 |
| Extensions | `@openclaw/*` | `./extensions/*` | 將 `@openclaw/` 前綴映射到 `extensions/` 目錄。所有官方擴充套件都可以透過 `@openclaw/extension-name` 的方式匯入。 |

#### 設計分析

1. **雙重命名空間（Dual Namespace）**：Plugin SDK 同時支援 `openclaw/plugin-sdk` 和 `@openclaw/plugin-sdk` 兩種匯入路徑。這可能是為了相容性考量——開發早期可能使用了 `openclaw/` 前綴，後來遷移到 npm scoped package 慣例 `@openclaw/`，兩者需要並存。

2. **精確路徑優先（Exact Path Priority）**：`openclaw/plugin-sdk/account-id` 被單獨定義，而不是依賴萬用字元 `openclaw/plugin-sdk/*`。TypeScript 的路徑解析在精確匹配和萬用字元匹配之間有優先順序，精確路徑總是優先。單獨定義可能是因為 `account-id` 模組有特殊的匯出需求。

3. **Monorepo 內部解析**：`@openclaw/*` → `./extensions/*` 的映射讓 TypeScript 在 monorepo 結構中能正確解析擴充套件之間的相互引用，無需透過 npm/pnpm 的 workspace 機制。

### 1.9 檔案範圍

```json
// source-repo/tsconfig.json:30-32
"include": ["src/**/*", "ui/**/*", "extensions/**/*", "packages/**/*"],
"exclude": ["node_modules", "dist", "**/dist/**"]
```

| 目錄 | 用途推斷 |
|------|----------|
| `src/**/*` | 核心原始碼 |
| `ui/**/*` | UI 層程式碼（使用 Lit Web Components） |
| `extensions/**/*` | 官方擴充套件 |
| `packages/**/*` | 內部共用套件（Monorepo 架構） |

排除項中 `**/dist/**` 的使用（除了頂層 `dist`）暗示子目錄中也可能存在各自的 `dist` 輸出目錄。

---

## 2. Plugin SDK 型別宣告設定 — `tsconfig.plugin-sdk.dts.json`

### 2.1 完整設定內容

```json
// source-repo/tsconfig.plugin-sdk.dts.json:1-15
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "declarationMap": false,
    "emitDeclarationOnly": true,
    "incremental": true,
    "noEmit": false,
    "noEmitOnError": false,
    "outDir": "dist/plugin-sdk",
    "rootDir": ".",
    "tsBuildInfoFile": "dist/plugin-sdk/.tsbuildinfo"
  },
  "include": [
    "src/plugin-sdk/**/*.ts",
    "src/video-generation/dashscope-compatible.ts",
    "src/video-generation/types.ts",
    "src/types/**/*.d.ts"
  ],
  "exclude": ["node_modules", "dist", "src/**/*.test.ts"]
}
```

### 2.2 與主設定的差異分析

此設定繼承自主設定（`extends: "./tsconfig.json"`），但覆寫了多個關鍵選項：

| 選項 | 主設定值 | Plugin SDK 設定值 | 覆寫原因 |
|------|----------|-------------------|----------|
| `noEmit` | `true` | `false` | 此設定需要產生輸出檔案（`.d.ts` 型別宣告） |
| `emitDeclarationOnly` | _(未設定)_ | `true` | 只產生 `.d.ts` 檔案，不產生 `.js` 檔案。JavaScript 打包由 tsdown 負責。 |
| `noEmitOnError` | `true` | `false` | 即使存在型別錯誤也產生 `.d.ts`。這可能是因為 Plugin SDK 的使用者需要型別提示，即使內部有些型別問題尚未解決。 |
| `declarationMap` | _(未設定)_ | `false` | 不產生 `.d.ts.map` 原始碼映射檔案。SDK 使用者不需要追溯到原始 `.ts` 原始碼。 |
| `incremental` | _(未設定)_ | `true` | 啟用增量編譯，加速重複建置。 |
| `outDir` | `"dist"` | `"dist/plugin-sdk"` | 輸出到專屬子目錄。 |
| `rootDir` | _(未設定)_ | `"."` | 以專案根目錄作為根，保持輸出的目錄結構與原始碼一致。 |
| `tsBuildInfoFile` | _(未設定)_ | `"dist/plugin-sdk/.tsbuildinfo"` | 增量編譯的快取檔案位置。 |

### 2.3 包含的原始碼範圍

```json
"include": [
  "src/plugin-sdk/**/*.ts",
  "src/video-generation/dashscope-compatible.ts",
  "src/video-generation/types.ts",
  "src/types/**/*.d.ts"
]
```

| 路徑 | 意義 |
|------|------|
| `src/plugin-sdk/**/*.ts` | Plugin SDK 的核心程式碼 |
| `src/video-generation/dashscope-compatible.ts` | 影片生成功能的 DashScope 相容層——暗示 Plugin SDK 對外暴露了影片生成能力 |
| `src/video-generation/types.ts` | 影片生成的型別定義 |
| `src/types/**/*.d.ts` | 共用型別宣告檔 |

**觀察**：Plugin SDK 的型別宣告不只包含 `src/plugin-sdk/` 目錄，還包含了 `video-generation` 的部分檔案。這說明 Plugin SDK 的公開 API 表面（Public API Surface）跨越了多個原始碼目錄。

### 2.4 排除項

```json
"exclude": ["node_modules", "dist", "src/**/*.test.ts"]
```

測試檔案被排除在型別宣告產生範圍之外——這是合理的，因為測試程式碼不應成為 SDK 公開 API 的一部分。

---

## 3. oxlint 專用設定 — `tsconfig.oxlint.json`

### 3.1 完整設定內容

```json
// source-repo/tsconfig.oxlint.json:1-5
{
  "extends": "./tsconfig.json",
  "include": ["src/**/*", "ui/**/*", "packages/**/*", "extensions/**/*", "scripts/**/*"],
  "exclude": ["node_modules", "dist", "dist-runtime"]
}
```

### 3.2 與主設定的差異

此設定同樣繼承自主設定，差異集中在檔案範圍：

| 比較項目 | 主設定 | oxlint 設定 | 差異 |
|----------|--------|-------------|------|
| include | `src, ui, extensions, packages` | `src, ui, packages, extensions, scripts` | **新增 `scripts/**/*`** |
| exclude | `node_modules, dist, **/dist/**` | `node_modules, dist, dist-runtime` | **新增 `dist-runtime`**；移除 `**/dist/**` |

#### 設計解讀

1. **`scripts/**/*` 被加入**：oxlint 需要分析建置腳本（如 `scripts/build-all.mjs`、`scripts/run-oxlint.mjs` 等），但主設定不包含這些腳本，因為它們不需要型別檢查（多為 `.mjs` 檔案）。

2. **`dist-runtime` 被排除**：這暗示 OpenClaw 有一個 `dist-runtime` 目錄存放執行時期產物，與主要的 `dist` 目錄分開。此目錄不應被 lint。

3. **移除 `**/dist/**`**：可能是因為 oxlint 有自己的忽略機制（`.oxlintrc.json` 中的 `ignorePatterns`），不需要在 tsconfig 層面重複排除。

---

## 4. 三份設定檔的關係圖

```
tsconfig.json (主設定)
├── tsconfig.plugin-sdk.dts.json (繼承 + 覆寫輸出選項)
│   └── 用途：tsc -p tsconfig.plugin-sdk.dts.json → 產生 .d.ts
└── tsconfig.oxlint.json (繼承 + 擴展範圍)
    └── 用途：oxlint --tsconfig tsconfig.oxlint.json → 靜態分析
```

| 設定檔 | 觸發腳本 | 主要功能 |
|--------|----------|----------|
| `tsconfig.json` | `pnpm tsgo`（型別檢查） | 型別檢查基準，不產生任何輸出 |
| `tsconfig.plugin-sdk.dts.json` | `pnpm build:plugin-sdk:dts` | 產生 Plugin SDK 的 `.d.ts` 型別宣告 |
| `tsconfig.oxlint.json` | `pnpm lint`（間接透過 oxlint） | 為 oxlint 提供擴展的型別資訊 |

---

## 5. 關鍵設計決策總結

### 5.1 型別檢查與打包分離

OpenClaw 將 TypeScript 嚴格定位為「型別檢查器」而非「編譯器」：

- 主設定 `noEmit: true` → tsc 不產生任何 `.js` 檔案
- `allowImportingTsExtensions: true` → 原始碼中直接使用 `.ts` 副檔名
- 實際的打包由 tsdown（基於 Rolldown）處理

這種「關注點分離」（Separation of Concerns）讓每個工具做自己最擅長的事：tsc 負責型別安全，tsdown 負責高效打包。

### 5.2 ES2023 作為穩定基線

選擇 `target: "es2023"` 而非 `esnext`，表明 OpenClaw 重視穩定性與可預測性。`esnext` 會隨 TypeScript 版本變化而改變支援的語法，而 `es2023` 是一個固定的規範版本。

### 5.3 裝飾器的歷史包袱

使用 `experimentalDecorators: true` + `useDefineForClassFields: false` 的組合，暗示 OpenClaw 的核心程式碼庫中存在使用舊式 TypeScript 裝飾器的程式碼。遷移到 TC39 Stage 3 標準裝飾器需要同步修改所有裝飾器使用處，這在大型專案中是一項重大工程。

### 5.4 Plugin SDK 的雙重命名空間

`openclaw/plugin-sdk` 和 `@openclaw/plugin-sdk` 同時存在，反映了專案命名慣例的演進。在 npm 生態系中，scoped package（`@scope/name`）是現代推薦做法，但舊式路徑可能仍有大量程式碼引用。

---

## 引用來源

| 來源檔案 | 行號範圍 | 引用內容 |
|----------|----------|----------|
| `source-repo/tsconfig.json` | 1-33 | 主 TypeScript 設定檔完整內容 |
| `source-repo/tsconfig.json` | 3 | `allowImportingTsExtensions: true` |
| `source-repo/tsconfig.json` | 9 | `lib` 欄位定義 |
| `source-repo/tsconfig.json` | 10-11 | `module` 與 `moduleResolution` 設定 |
| `source-repo/tsconfig.json` | 12-13 | `noEmit` 與 `noEmitOnError` |
| `source-repo/tsconfig.json` | 17 | `strict: true` |
| `source-repo/tsconfig.json` | 18 | `target: "es2023"` |
| `source-repo/tsconfig.json` | 5 | `experimentalDecorators: true` |
| `source-repo/tsconfig.json` | 20 | `useDefineForClassFields: false` |
| `source-repo/tsconfig.json` | 21-28 | `paths` 路徑別名系統 |
| `source-repo/tsconfig.json` | 30-32 | `include` 與 `exclude` 範圍定義 |
| `source-repo/tsconfig.plugin-sdk.dts.json` | 1-15 | Plugin SDK 型別宣告設定完整內容 |
| `source-repo/tsconfig.plugin-sdk.dts.json` | 4-12 | 覆寫的 `compilerOptions` |
| `source-repo/tsconfig.plugin-sdk.dts.json` | 13-18 | Plugin SDK 的 `include` 範圍 |
| `source-repo/tsconfig.oxlint.json` | 1-5 | oxlint 專用設定完整內容 |
| `source-repo/package.json` | scripts 區段 | `build:plugin-sdk:dts` 腳本定義 |
