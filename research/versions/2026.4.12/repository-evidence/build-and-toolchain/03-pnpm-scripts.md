# 03 — pnpm 腳本參考手冊（pnpm Scripts Reference）

> **層級**：Repository Evidence · Topic 4 · Chapter 03  
> **版本快照**：2026.4.12  
> **最後更新**：2026-07-21

---

## 本章摘要

OpenClaw 的 `package.json` 中定義了 100 餘個 pnpm 腳本（Scripts），構成了整個開發工作流的自動化基礎。這些腳本涵蓋了從建置、測試、靜態分析、格式化、到架構守衛（Architectural Guards）的完整鏈路。本章將這些腳本按功能分類整理，逐一解析其命令內容、用途與彼此之間的依賴關係，為後續理解 CI/CD 流水線提供必要的知識基礎。

值得特別關注的是，OpenClaw 的腳本體系展現了一種「多層防線」設計哲學——從格式化、靜態分析、架構約束、到完整測試，每一層都有專門的腳本，且它們被精心編排成可以單獨執行或組合執行的流水線。

---

## 1. 腳本分類概覽

OpenClaw 的 100+ 腳本可分為以下 8 大類：

| 分類 | 腳本數量（約） | 代表腳本 |
|------|---------------|----------|
| 核心建置（Build） | 5+ | `build`, `build:docker`, `build:plugin-sdk:dts` |
| 開發模式（Dev） | 2 | `dev`, `start` |
| 檢查流水線（Check） | 10+ | `check`, `check:import-cycles`, `check:loc` |
| 格式化（Format） | 2 | `format`, `format:check` |
| 靜態分析（Lint） | 30+ | `lint`, `lint:fix`, `lint:extensions:*`, `lint:plugins:*` |
| 測試（Test） | 5+ | `test`, `test:all`, `test:e2e`, `test:live` |
| 生命週期鉤子（Lifecycle） | 2 | `postinstall`, `prepare` |
| 其他工具 | 5+ | `tsgo`, 各種 `tool-*` 腳本 |

---

## 2. 核心建置腳本

### 2.1 主要建置腳本

| 腳本 | 命令 | 說明 |
|------|------|------|
| `build` | `node scripts/build-all.mjs` | 完整建置。執行 `scripts/build-all.mjs`，此腳本統籌所有建置步驟。 |
| `build:docker` | `node scripts/tsdown-build.mjs && node scripts/runtime-postbuild.mjs && ...` | Docker 映像專用建置。包含打包、後處理、以及 Docker 特定的最佳化步驟。 |
| `build:plugin-sdk:dts` | `tsc -p tsconfig.plugin-sdk.dts.json` | 使用 TypeScript 編譯器產生 Plugin SDK 的 `.d.ts` 型別宣告檔案。參見 [01-tsconfig-configuration.md](./01-tsconfig-configuration.md) 第 2 節。 |
| `build:strict-smoke` | _(多步驟，以 `check-plugin-sdk-exports` 結尾)_ | 嚴格煙霧測試建置。確保建置產物的完整性和正確性，最終檢查 Plugin SDK 的匯出是否完整。 |

### 2.2 建置流程解析

#### `pnpm build` 完整流程

```
pnpm build
  └── node scripts/build-all.mjs
        ├── 1. tsdown 打包（讀取 tsdown.config.ts）
        │     └── 產生 dist/ 目錄下的所有模組
        ├── 2. Plugin SDK 型別宣告（tsc -p tsconfig.plugin-sdk.dts.json）
        │     └── 產生 dist/plugin-sdk/ 下的 .d.ts 檔案
        └── 3. 其他後處理步驟
```

#### `pnpm build:docker` 特殊之處

Docker 建置使用不同的腳本入口（`scripts/tsdown-build.mjs` 而非 `scripts/build-all.mjs`），並且包含額外的後處理步驟（`scripts/runtime-postbuild.mjs`）。這暗示 Docker 產物有不同的需求，例如：
- 可能需要更小的產物體積
- 可能需要額外的執行時期檔案處理
- 可能需要不同的環境變數注入

---

## 3. 開發模式腳本

| 腳本 | 命令 | 說明 |
|------|------|------|
| `dev` | `node scripts/run-node.mjs` | 開發模式啟動。透過 `tsx`（TypeScript 直接執行器）載入原始碼，無需預先打包。適合快速開發迭代。 |
| `start` | `node scripts/run-node.mjs` | 啟動 OpenClaw。與 `dev` 使用相同命令，可能透過環境變數區分行為。 |

**觀察**：`dev` 和 `start` 使用完全相同的命令 `node scripts/run-node.mjs`，這表示區分開發模式和正式啟動的邏輯在 `scripts/run-node.mjs` 內部，可能透過環境變數（如 `NODE_ENV`）或命令列參數來決定。

---

## 4. 檢查流水線腳本（Check Pipeline）

### 4.1 主檢查腳本

```
pnpm check = 
  pnpm check:no-conflict-markers
  && pnpm tool-display:check
  && ...
  && pnpm tsgo
  && ...
  && pnpm lint
  && ...
```

`check` 是一個串接多個子檢查的「閘門腳本」（Gate Script），依序執行所有品質檢查。任何一個子檢查失敗都會中斷整個流水線（因為使用 `&&` 串接）。

### 4.2 子檢查腳本一覽

| 腳本 | 用途 | 說明 |
|------|------|------|
| `check:no-conflict-markers` | 衝突標記偵測 | 檢查原始碼中是否殘留 Git 合併衝突標記（`<<<<<<<`、`=======`、`>>>>>>>`）。防止未解決的衝突被意外提交。 |
| `check:base-config-schema` | 基礎設定 Schema 驗證 | 驗證 OpenClaw 的基礎設定檔是否符合定義的 JSON Schema。 |
| `check:bundled-channel-config-metadata` | 通道設定元資料檢查 | 確認 Bundled 通道（如 Telegram、Discord）的設定元資料是否完整且一致。 |
| `check:docs` | 文件檢查 | 確認文件的完整性和一致性。 |
| `check:host-env-policy:swift` | Swift 宿主環境政策檢查 | 檢查 Swift 相關程式碼（iOS/macOS 宿主應用）是否符合環境政策。 |
| `check:import-cycles` | 匯入循環偵測（自訂） | 使用自訂腳本偵測模組之間的循環匯入（Circular Imports）。 |
| `check:madge-import-cycles` | 匯入循環偵測（Madge） | 使用 Madge 工具偵測匯入循環。與上方的自訂腳本形成雙重檢查。 |
| `check:loc` | 程式碼行數限制 | 確認單一檔案不超過 500 行。這是 OpenClaw 的架構約束——強制小檔案（Small File）原則，防止出現「上帝物件」（God Object）或過度臃腫的模組。 |

### 4.3 型別檢查

`check` 流水線中包含 `pnpm tsgo`，這是使用 `@typescript/native-preview`（版本 7.0.0-dev）的原生型別檢查器。tsgo 是 TypeScript 團隊正在開發的 Rust/Go 原生實作，速度遠快於傳統的 `tsc`。

**觀察**：OpenClaw 使用 tsgo 作為主要型別檢查器而非 tsc，說明專案積極擁抱新技術以加速開發體驗。tsgo 尚處於開發預覽階段，OpenClaw 願意承擔一定的前沿風險。

### 4.4 架構守衛的意義

`check:loc`（500 行限制）和 `check:import-cycles`（循環匯入偵測）不是傳統的「語法檢查」或「型別檢查」，而是「架構守衛」（Architectural Guards）。它們將架構原則自動化為可執行的規則：

```
架構原則：「每個模組應保持小而專注」
  → 自動化規則：check:loc（最多 500 行/檔案）

架構原則：「模組之間應有清晰的層次結構」
  → 自動化規則：check:import-cycles + check:madge-import-cycles
```

---

## 5. 格式化腳本

| 腳本 | 命令 | 說明 |
|------|------|------|
| `format` | `oxfmt --write` | 使用 oxfmt 格式化所有程式碼，直接寫入修改。oxfmt 是 Oxc 工具鏈的一部分，使用 Rust 實作，速度極快。 |
| `format:check` | `oxfmt --check --threads=1` | 檢查格式是否一致，但不修改檔案。使用 `--threads=1` 確保輸出的確定性（Deterministic Output）——多執行緒模式下輸出順序可能不一致。 |

**工具選擇**：OpenClaw 使用 oxfmt 而非 Prettier。oxfmt 是基於 Rust 的格式化工具，屬於 Oxc（Oxidation Compiler）生態系。與 Prettier 相比，oxfmt 的優勢在於：
- 速度快數十倍（Rust vs. JavaScript）
- 與 oxlint 同屬 Oxc 生態系，設定和行為更一致

---

## 6. 靜態分析腳本（Lint Scripts）

### 6.1 主要 Lint 腳本

| 腳本 | 命令 | 說明 |
|------|------|------|
| `lint` | `node scripts/run-oxlint.mjs` | 使用 oxlint 進行靜態分析。oxlint 是 Oxc 工具鏈的 Linter，基於 Rust 實作。 |
| `lint:fix` | `node scripts/run-oxlint.mjs --fix && pnpm format` | 自動修復 lint 問題並重新格式化。注意：修復後會接續格式化，確保自動修復的程式碼也符合格式規範。 |

### 6.2 擴充套件 Lint 腳本

這類腳本專門檢查 `extensions/` 目錄下的程式碼，確保擴充套件遵循架構邊界：

| 腳本 | 用途 |
|------|------|
| `lint:extensions:bundled` | 檢查 Bundled Extensions 的合規性 |
| `lint:extensions:channels` | 檢查通道擴充套件（Channel Extensions）的合規性 |
| `lint:extensions:no-plugin-sdk-internal` | **禁止擴充套件匯入 Plugin SDK 的內部模組**。擴充套件只能使用 Plugin SDK 的公開 API。 |
| `lint:extensions:no-relative-outside-package` | **禁止擴充套件使用相對路徑匯入套件外的模組**。防止跨套件的緊耦合。 |
| `lint:extensions:no-src-outside-plugin-sdk` | **禁止擴充套件直接匯入 `src/` 目錄的模組**（Plugin SDK 除外）。擴充套件必須透過正式 API 存取核心功能。 |

### 6.3 Plugin Lint 腳本

這類腳本檢查 Plugin 系統的程式碼：

| 腳本 | 用途 |
|------|------|
| `lint:plugins:no-extension-imports` | **禁止 Plugin 匯入 Extension 模組**。Plugin 和 Extension 是不同的抽象層，不應直接依賴。 |
| `lint:plugins:no-extension-src-imports` | **禁止 Plugin 匯入 Extension 的原始碼目錄** |
| `lint:plugins:no-monolithic-plugin-sdk-entry-imports` | **禁止從 Plugin SDK 的整體進入點匯入**。強制使用細粒度的子路徑匯入（如 `openclaw/plugin-sdk/auth` 而非 `openclaw/plugin-sdk`），以利於 Tree Shaking。 |
| `lint:plugins:no-register-http-handler` | **禁止 Plugin 直接註冊 HTTP 處理器**。HTTP 處理可能需透過特定的 API 層進行。 |
| `lint:plugins:plugin-sdk-subpaths-exported` | 確認 Plugin SDK 的所有子路徑都已正確匯出 |

### 6.4 其他 Lint 腳本

| 腳本 | 用途 |
|------|------|
| `lint:webhook:no-low-level-body-read` | 禁止 Webhook 處理器使用低層級的 body 讀取方式 |
| `lint:auth:no-pairing-store-group` | 認證模組的特定約束 |
| `lint:auth:pairing-account-scope` | 確認認證模組的帳號範圍設定 |

### 6.5 Lint 腳本的架構邊界映射

OpenClaw 的 30+ lint 腳本實質上定義了一套嚴格的架構邊界規則。以下圖表展示了主要的匯入限制：

```
┌────────────────────────────────────────────┐
│                  核心（src/）                │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │        Plugin SDK（公開 API）         │  │
│  │    src/plugin-sdk/                   │  │
│  └──────────┬───────────────────────────┘  │
│             │ 唯一允許的介面                 │
│             ▼                              │
│  ┌──────────────────────────────────────┐  │
│  │         Plugins（src/plugins/）       │  │
│  │  ✗ 不可匯入 extensions/              │  │
│  │  ✗ 不可整體匯入 plugin-sdk           │  │
│  │  ✗ 不可註冊 HTTP handler             │  │
│  └──────────────────────────────────────┘  │
│                                            │
├────────────────────────────────────────────┤
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │      Extensions（extensions/）        │  │
│  │  ✗ 不可匯入 plugin-sdk 內部          │  │
│  │  ✗ 不可相對匯入套件外模組            │  │
│  │  ✗ 不可直接匯入 src/                 │  │
│  └──────────────────────────────────────┘  │
│                                            │
└────────────────────────────────────────────┘
```

**設計哲學**：這些 lint 規則形成了一個「分層架構」（Layered Architecture）的自動化守衛。核心透過 Plugin SDK 向外暴露 API，Plugin 和 Extension 只能透過此 API 存取核心功能，不得跨層或反向依賴。這確保了模組化和可替換性。

---

## 7. 測試腳本

### 7.1 測試腳本一覽

| 腳本 | 命令 | 說明 |
|------|------|------|
| `test` | `node scripts/test-projects.mjs` | 執行主要測試套件。透過自訂腳本管理多專案測試。 |
| `test:all` | `pnpm lint && pnpm build && pnpm test && pnpm test:e2e && pnpm test:live && pnpm test:docker:all` | 完整測試流水線——從 lint 到單元測試、端對端測試、即時測試、Docker 測試。 |
| `test:e2e` | _(未詳列)_ | 端對端測試（End-to-End Tests） |
| `test:live` | _(未詳列)_ | 即時測試——可能涉及實際的 API 呼叫或服務互動 |
| `test:docker:all` | _(未詳列)_ | Docker 環境下的完整測試 |

### 7.2 `test:all` 的執行順序

```
pnpm test:all 的完整流程：

1. pnpm lint          ← 先通過靜態分析
2. pnpm build         ← 再確認建置成功
3. pnpm test          ← 然後執行單元測試
4. pnpm test:e2e      ← 端對端測試
5. pnpm test:live     ← 即時測試（可能需要網路）
6. pnpm test:docker:all ← Docker 環境測試
```

**設計解讀**：`test:all` 的執行順序遵循「快速失敗」（Fail Fast）原則——最快且最可能失敗的檢查（lint）放在最前面，最慢且最昂貴的檢查（Docker 測試）放在最後。如果 lint 都沒過，就沒必要跑建置和測試了。

### 7.3 測試框架

OpenClaw 使用 Vitest 作為測試框架（版本 ^4.1.4），搭配以下工具：

| 工具 | 版本 | 用途 |
|------|------|------|
| `vitest` | ^4.1.4 | 測試框架與測試執行器 |
| `@vitest/coverage-v8` | ^4.1.4 | 基於 V8 引擎的程式碼覆蓋率 |
| `jsdom` | ^29.0.2 | DOM 模擬環境（用於 UI 組件測試） |

---

## 8. 生命週期鉤子腳本

### 8.1 `postinstall`

```
postinstall: node scripts/postinstall-bundled-plugins.mjs
```

此腳本在 `pnpm install` 完成後自動執行，處理 Bundled Plugin 的安裝後置作業。可能的功能包括：
- 將 Bundled Plugin 的依賴連結到正確位置
- 產生 Plugin 的設定檔或元資料
- 編譯或預處理 Plugin 資源

### 8.2 `prepare`

```
prepare: git config core.hooksPath git-hooks
```

此腳本在 `pnpm install` 後（以及 `git clone` 後首次 `pnpm install` 時）自動執行。它將 Git 的 hooks 路徑指向專案根目錄的 `git-hooks/` 目錄。

**意義**：OpenClaw 使用自訂的 Git hooks 目錄（`git-hooks/`）而非預設的 `.git/hooks/`。這允許 Git hooks 被版本控制追蹤，確保所有開發者使用相同的 hooks（如 pre-commit 檢查）。

---

## 9. 完整檢查流水線重建

### 9.1 `pnpm check` 的完整步驟

根據所有已知的子腳本，可以重建 `pnpm check` 的完整檢查流程：

```
pnpm check（完整檢查流水線）
  │
  ├─ 1. check:no-conflict-markers    ← 確認無 Git 衝突標記
  ├─ 2. tool-display:check           ← 工具版本/狀態檢查
  ├─ 3. ...                          ← 其他前置檢查
  ├─ 4. tsgo                         ← 原生 TypeScript 型別檢查
  ├─ 5. ...                          ← 中間步驟
  ├─ 6. lint                         ← oxlint 靜態分析
  │     ├─ lint:extensions:*         ← 擴充套件架構邊界
  │     ├─ lint:plugins:*            ← Plugin 架構邊界
  │     └─ lint:auth:*, lint:webhook:* ← 領域特定規則
  └─ 7. ...                          ← 其他後置檢查
```

### 9.2 品質閘門的多層防線

```
第一層：格式化
  └─ format:check         → 確保程式碼格式一致

第二層：靜態分析
  └─ lint (oxlint)        → 捕捉常見錯誤和反模式

第三層：型別安全
  └─ tsgo                 → 完整的型別檢查

第四層：架構約束
  ├─ check:import-cycles  → 無循環依賴
  ├─ check:loc            → 每檔不超過 500 行
  ├─ lint:extensions:*    → 擴充套件邊界
  └─ lint:plugins:*       → Plugin 邊界

第五層：建置驗證
  └─ build:strict-smoke   → 建置產物完整性

第六層：測試
  ├─ test                 → 單元測試
  ├─ test:e2e             → 端對端測試
  ├─ test:live            → 即時測試
  └─ test:docker:all      → Docker 環境測試
```

---

## 10. 腳本命名慣例

### 10.1 命名模式

| 模式 | 範例 | 意義 |
|------|------|------|
| `verb` | `build`, `test`, `lint`, `format` | 主要動作 |
| `verb:target` | `build:docker`, `test:e2e` | 針對特定目標的動作 |
| `verb:target:detail` | `build:plugin-sdk:dts`, `test:docker:all` | 更細粒度的動作 |
| `check:rule` | `check:loc`, `check:import-cycles` | 特定的檢查規則 |
| `lint:scope:rule` | `lint:extensions:no-plugin-sdk-internal` | 特定範圍的 lint 規則 |

### 10.2 慣例總結

1. **冒號分隔**（`:`）：用於建立腳本的層級結構。pnpm 支援使用 `pnpm lint:extensions:*` 這類模式來篩選和批次執行腳本。
2. **`no-` 前綴**：表示禁止性規則（如 `no-plugin-sdk-internal`、`no-relative-outside-package`）。
3. **`all` 後綴**：表示完整/全部（如 `test:all`、`test:docker:all`）。
4. **動詞在前**：所有腳本以動作動詞開頭，明確表達意圖。

---

## 11. 與 CI/CD 的對應關係

| pnpm 腳本 | CI 工作（Job） | 說明 |
|-----------|---------------|------|
| `check` | `check`, `checks-fast-core` | 主檢查流水線 |
| `lint` | `checks-fast-core` | 靜態分析 |
| `build` | `build-artifacts` | 建置產物 |
| `build:strict-smoke` | `build-smoke` | 煙霧測試建置 |
| `test` | `checks-node-core-test-shard` | 核心測試（分片） |
| `test:all` | 多個 CI 工作組合 | 完整測試套件 |
| `lint:extensions:*` | `checks-node-extensions-shard` | 擴充套件檢查（分片） |

---

## 12. 關鍵設計決策總結

### 12.1 自訂腳本包裝器

OpenClaw 的許多核心腳本不是直接呼叫工具，而是透過 `node scripts/*.mjs` 包裝。這提供了：
- 更靈活的參數處理
- 統一的錯誤處理和輸出格式
- 環境變數的動態設定
- 多步驟建置的流程控制

### 12.2 架構約束自動化

30+ lint 腳本將架構決策編碼為可執行的規則。這意味著每次提交都會自動驗證架構邊界是否被維護，無需人工程式碼審查來捕捉架構違規。

### 12.3 500 行限制的實踐意義

`check:loc` 強制每個檔案不超過 500 行，這是一個相當嚴格的限制。在 100+ 進入點的大型專案中，這個規則有效防止了程式碼的無序增長，迫使開發者將邏輯拆分為更小、更專注的模組。

### 12.4 雙重循環偵測

同時使用自訂腳本（`check:import-cycles`）和 Madge（`check:madge-import-cycles`）來偵測匯入循環，形成雙重驗證。這可能是因為兩種工具的偵測演算法不同，互相補充可以提高偵測率。

---

## 引用來源

| 來源檔案 | 行號範圍 | 引用內容 |
|----------|----------|----------|
| `source-repo/package.json` | scripts 區段 | 所有 pnpm 腳本定義 |
| `source-repo/package.json` | scripts.build | `node scripts/build-all.mjs` |
| `source-repo/package.json` | scripts.build:docker | Docker 建置命令 |
| `source-repo/package.json` | scripts.build:plugin-sdk:dts | `tsc -p tsconfig.plugin-sdk.dts.json` |
| `source-repo/package.json` | scripts.dev | `node scripts/run-node.mjs` |
| `source-repo/package.json` | scripts.start | `node scripts/run-node.mjs` |
| `source-repo/package.json` | scripts.check | 完整檢查流水線 |
| `source-repo/package.json` | scripts.format | `oxfmt --write` |
| `source-repo/package.json` | scripts.format:check | `oxfmt --check --threads=1` |
| `source-repo/package.json` | scripts.lint | `node scripts/run-oxlint.mjs` |
| `source-repo/package.json` | scripts.lint:fix | `node scripts/run-oxlint.mjs --fix && pnpm format` |
| `source-repo/package.json` | scripts.test | `node scripts/test-projects.mjs` |
| `source-repo/package.json` | scripts.test:all | 完整測試命令 |
| `source-repo/package.json` | scripts.postinstall | `node scripts/postinstall-bundled-plugins.mjs` |
| `source-repo/package.json` | scripts.prepare | `git config core.hooksPath git-hooks` |
| `source-repo/package.json` | scripts.check:loc | 500 行限制檢查 |
| `source-repo/package.json` | scripts.check:import-cycles | 匯入循環偵測 |
| `source-repo/package.json` | scripts.check:madge-import-cycles | Madge 匯入循環偵測 |
| `source-repo/package.json` | devDependencies | 開發依賴清單 |
| `source-repo/tsconfig.plugin-sdk.dts.json` | 1-15 | Plugin SDK 型別宣告設定 |
| `source-repo/vitest.config.ts` | 1-7 | Vitest 設定入口 |
