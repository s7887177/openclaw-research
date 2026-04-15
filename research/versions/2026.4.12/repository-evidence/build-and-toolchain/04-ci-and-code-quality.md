# 04 — CI/CD 與程式碼品質工具（CI/CD & Code Quality Tools）

> **層級**：Repository Evidence · Topic 4 · Chapter 04  
> **版本快照**：2026.4.12  
> **最後更新**：2026-07-21

---

## 本章摘要

OpenClaw 建立了一套極為完善的 CI/CD 與程式碼品質保障體系。其核心是一份長達 1,378 行的 GitHub Actions CI 設定檔（`ci.yml`），定義了 20 餘個工作（Jobs），涵蓋從安全掃描、建置產物、多平台測試到文件檢查的完整流水線。在 CI 之外，OpenClaw 還整合了 oxlint（靜態分析）、oxfmt（格式化）、Vitest（測試框架）、knip（死碼偵測）、pre-commit hooks、jscpd（複製貼上偵測）、madge（匯入循環偵測）等眾多品質工具，形成多層次的品質閘門。

本章從 CI/CD 流水線開始，逐一記錄每個品質工具的設定細節和設計考量。

---

## 1. GitHub Actions CI 流水線

### 1.1 概覽

| 面向 | 數據 |
|------|------|
| 設定檔 | `source-repo/.github/workflows/ci.yml` |
| 行數 | 1,378 行 |
| 觸發條件 | Push 到 `main` 分支 + Pull Request |
| 工作數量 | 20+ 個 |
| 平台覆蓋 | Linux、Windows、macOS |
| 分片策略 | 核心測試和擴充套件檢查均使用分片（Sharding） |

### 1.2 工作清單與依賴關係

以下是 CI 中所有工作的完整清單，按執行階段分組：

#### 第一階段：前置飛行（Preflight）

| 工作 | 說明 |
|------|------|
| `preflight` | 偵測本次提交/PR 變更的範圍（Changed Scopes），產生後續工作的矩陣（Job Matrices）。這是所有其他工作的前置條件。 |

`preflight` 的存在表明 CI 採用了「智慧觸發」策略——不是每次都跑所有檢查，而是根據變更的檔案範圍來決定需要執行哪些工作。例如，如果只修改了文件，就不需要跑建置和測試。

#### 第二階段：快速檢查（Fast Checks）

| 工作 | 說明 |
|------|------|
| `security-fast` | 快速安全檢查。可能包括依賴漏洞掃描、密鑰洩漏偵測等。放在最前面是因為安全問題具有最高優先順序。 |
| `checks-fast-core` | 快速核心檢查。包括 lint、格式化檢查、型別檢查等不需要完整建置的快速驗證。 |
| `extension-fast` | 擴充套件的快速檢查。 |

#### 第三階段：建置（Build）

| 工作 | 說明 |
|------|------|
| `build-artifacts` | 建置產物。執行完整的建置流程，產生用於後續測試的建置產物。 |
| `build-smoke` | 煙霧測試建置。使用 `build:strict-smoke` 驗證建置產物的完整性。 |

#### 第四階段：測試（分片）

| 工作 | 說明 |
|------|------|
| `checks-node-core-test-shard` | 核心 Node.js 測試，使用分片（Sharding）並行執行。分片策略將測試套件分成多個分片，每個分片在獨立的 CI 執行器上並行執行，大幅縮短總測試時間。 |
| `checks-node-core-test` | 核心測試的閘門工作（Gate Job）。等待所有分片完成後，匯總結果。 |
| `checks-node-extensions-shard` | 擴充套件檢查，同樣使用分片策略。 |
| `checks-node-extensions` | 擴充套件檢查的閘門工作。 |

#### 第五階段：綜合檢查

| 工作 | 說明 |
|------|------|
| `checks` | 完整檢查。可能等待前面所有快速檢查和測試完成後執行。 |
| `check` | 主要檢查工作。對應 `pnpm check` 腳本。 |
| `check-additional` | 額外檢查。可能包括不常執行的低頻檢查。 |
| `check-docs` | 文件檢查。確認文件的正確性和完整性。 |

#### 第六階段：多平台測試

| 工作 | 平台 | 說明 |
|------|------|------|
| `checks-windows` | Windows | Windows 平台檢查。確認 OpenClaw 在 Windows 上能正確運作。 |
| `macos-node` | macOS | macOS Node.js 檢查。 |
| `macos-swift` | macOS | macOS Swift 檢查。OpenClaw 有 macOS/iOS 原生應用的 Swift 程式碼。 |
| `android` | Android | Android 平台檢查。 |

#### 第七階段：特定語言/生態系

| 工作 | 說明 |
|------|------|
| `skills-python` | Python Skills 檢查。OpenClaw 的某些 Skills 使用 Python 撰寫。 |

### 1.3 分片策略（Sharding Strategy）

分片是 OpenClaw CI 的關鍵效能最佳化策略。原理如下：

```
傳統方式（串行）：
  測試 A → 測試 B → 測試 C → 測試 D
  總時間 = A + B + C + D

分片方式（並行）：
  分片 1：測試 A, B  ─┐
  分片 2：測試 C, D  ─┤→ 等待所有分片完成
                      │
  總時間 ≈ max(分片1, 分片2)
```

OpenClaw 對以下兩類測試使用分片：
1. **核心測試**（`checks-node-core-test-shard`）：核心 Node.js 單元測試
2. **擴充套件檢查**（`checks-node-extensions-shard`）：擴充套件相關檢查

每組分片都有對應的「閘門工作」（Gate Job），負責：
- 等待所有分片完成
- 匯總分片結果
- 作為後續工作的依賴目標（後續工作只需依賴閘門工作，而非所有分片）

### 1.4 `preflight` 的變更偵測機制

`preflight` 工作的核心功能是偵測本次提交變更了哪些「範圍」（Scopes），然後動態產生後續工作的矩陣。這種模式在大型 monorepo 中非常常見：

```
變更偵測邏輯（概念性）：

如果 src/ 有變更 → 啟動核心測試
如果 extensions/ 有變更 → 啟動擴充套件測試
如果 docs/ 有變更 → 啟動文件檢查
如果 .swift 檔案有變更 → 啟動 macOS Swift 檢查
如果 android/ 有變更 → 啟動 Android 檢查
如果 skills/python/ 有變更 → 啟動 Python skills 檢查
```

**效益**：這避免了每次 PR 都跑所有 20+ 個工作。如果一個 PR 只修改了文件，CI 可能只需要跑 `check-docs` 一個工作，而非全部。

### 1.5 其他工作流程檔案

除了主要的 `ci.yml`，OpenClaw 還有大量專用的工作流程檔案：

| 工作流程檔案 | 用途分類 | 說明 |
|-------------|----------|------|
| `auto-response.yml` | 社群管理 | 自動回應 Issue 或 PR |
| `labeler.yml` | 社群管理 | 自動為 PR 加標籤 |
| `stale.yml` | 社群管理 | 自動處理過期的 Issue/PR |
| `docker-release.yml` | 發布 | Docker 映像發布 |
| `macos-release.yml` | 發布 | macOS 應用發布 |
| `openclaw-npm-release.yml` | 發布 | npm 套件發布 |
| `openclaw-release-checks.yml` | 發布 | 發布前檢查 |
| `plugin-clawhub-release.yml` | 發布 | Plugin 發布到 ClawHub（Plugin 市集） |
| `plugin-npm-release.yml` | 發布 | Plugin 發布到 npm |
| `docs-translate-trigger-release.yml` | 文件 | 觸發文件翻譯和發布 |
| `docs-sync-publish.yml` | 文件 | 文件同步與發布 |
| `control-ui-locale-refresh.yml` | UI | 控制 UI 的語言檔案更新 |
| `install-smoke.yml` | 品質 | 安裝煙霧測試——確認 npm 安裝流程正常 |
| `sandbox-common-smoke.yml` | 品質 | 沙箱環境的煙霧測試 |
| `parity-gate.yml` | 品質 | 平台一致性閘門 |
| `workflow-sanity.yml` | 品質 | 工作流程自身的健全性檢查 |
| `codeql.yml` | 安全 | CodeQL 靜態應用安全測試（SAST） |

**觀察**：OpenClaw 有完善的多平台發布流水線（Docker、macOS、npm、ClawHub），反映了其作為跨平台 AI 自動化框架的定位。

---

## 2. oxlint — 靜態分析工具

### 2.1 工具概覽

| 面向 | 說明 |
|------|------|
| 工具名稱 | oxlint |
| 版本 | ^1.59.0 |
| 設定檔 | `source-repo/.oxlintrc.json`（1-67 行） |
| 底層引擎 | Oxc（Oxidation Compiler），以 Rust 實作 |
| 與 ESLint 的關係 | oxlint 是 ESLint 的高效能替代品，支援部分 ESLint 規則 |

### 2.2 設定檔完整內容

```json
// source-repo/.oxlintrc.json:1-67
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "plugins": ["unicorn", "typescript", "oxc"],
  "categories": {
    "correctness": "error",
    "perf": "error",
    "suspicious": "error"
  },
  "rules": {
    "curly": "error",
    "eslint-plugin-unicorn/prefer-array-find": "error",
    "typescript/no-explicit-any": "error",
    "typescript/no-extraneous-class": "error",
    "typescript/consistent-return": "error",
    "oxc/no-accumulating-spread": "error"
  },
  "ignorePatterns": [
    "assets/", "dist/", "node_modules/", "patches/",
    "skills/", "vendor/"
  ],
  "overrides": [
    {
      "files": ["**/*.test.ts"],
      "rules": {
        "typescript/no-explicit-any": "off"
      }
    }
  ]
}
```

### 2.3 Plugin 啟用

```json
"plugins": ["unicorn", "typescript", "oxc"]
```

| Plugin | 來源 | 提供的規則 |
|--------|------|-----------|
| `unicorn` | eslint-plugin-unicorn 移植 | 現代 JavaScript 最佳實踐（如偏好 `Array.find()` 而非 `Array.filter()[0]`） |
| `typescript` | @typescript-eslint 移植 | TypeScript 特有的規則（如禁止 `any`、禁止多餘的 class） |
| `oxc` | Oxc 原生規則 | Oxc 特有的效能和正確性規則（如禁止累積式展開運算） |

### 2.4 類別（Categories）設定

```json
"categories": {
  "correctness": "error",
  "perf": "error",
  "suspicious": "error"
}
```

| 類別 | 層級 | 涵蓋範圍 |
|------|------|----------|
| `correctness` | `error` | 正確性問題——可能導致程式行為錯誤的程式碼模式 |
| `perf` | `error` | 效能問題——可能導致不必要的效能損耗的程式碼模式 |
| `suspicious` | `error` | 可疑程式碼——看起來可能是 bug 但不一定的程式碼模式 |

**觀察**：三個類別全部設為 `error`（而非 `warn`），表示 OpenClaw 對程式碼品質採取零容忍策略。任何正確性、效能或可疑問題都會直接中斷 CI。

### 2.5 關鍵規則解析

| 規則 | 層級 | 意義 |
|------|------|------|
| `curly` | `error` | 強制所有控制結構（`if`、`else`、`for`、`while`）使用花括號。禁止 `if (x) return;` 這類無花括號的寫法。 |
| `eslint-plugin-unicorn/prefer-array-find` | `error` | 偏好使用 `Array.find()` 取代 `Array.filter()[0]`。後者會建立一個中間陣列，浪費記憶體。 |
| `typescript/no-explicit-any` | `error` | **禁止使用 `any` 型別**。這是 OpenClaw 型別安全策略的核心——不允許任何程式碼繞過型別系統。 |
| `typescript/no-extraneous-class` | `error` | 禁止建立只有靜態成員的 class。應使用普通物件或函式替代。 |
| `typescript/consistent-return` | `error` | 強制函式的所有返回路徑型別一致。 |
| `oxc/no-accumulating-spread` | `error` | 禁止在迴圈中使用累積式展開運算（如 `arr = [...arr, item]`）。這是一個 O(n²) 的反模式，應使用 `push()` 替代。 |

### 2.6 測試檔案的規則覆寫

```json
"overrides": [
  {
    "files": ["**/*.test.ts"],
    "rules": {
      "typescript/no-explicit-any": "off"
    }
  }
]
```

在測試檔案中，`no-explicit-any` 規則被關閉。這是合理的——測試程式碼經常需要使用 `any` 來建立 mock 物件或測試邊界情況。

### 2.7 忽略模式

```json
"ignorePatterns": [
  "assets/", "dist/", "node_modules/", "patches/",
  "skills/", "vendor/"
]
```

| 忽略目錄 | 原因 |
|----------|------|
| `assets/` | 靜態資產，非程式碼 |
| `dist/` | 建置產物 |
| `node_modules/` | 第三方依賴 |
| `patches/` | 補丁檔案（可能用於 pnpm 的 patch 功能） |
| `skills/` | Skills 目錄——可能使用不同的語言（如 Python）或有自己的 lint 規則 |
| `vendor/` | 第三方原始碼副本 |

---

## 3. oxfmt — 程式碼格式化工具

### 3.1 設定檔完整內容

```jsonc
// source-repo/.oxfmtrc.jsonc:1-20
{
  "$schema": "./node_modules/oxfmt/configuration_schema.json",
  "sortImports": { "newlinesBetween": false },
  "sortPackageJson": { "sortScripts": true },
  "tabWidth": 2,
  "useTabs": false,
  "ignorePatterns": [
    "apps/", "assets/", "dist/", "node_modules/",
    "patches/", "vendor/"
  ]
}
```

### 3.2 設定解析

| 選項 | 值 | 意義 |
|------|----|------|
| `sortImports.newlinesBetween` | `false` | 匯入排序時，不在不同群組之間插入空行。所有 `import` 語句緊密排列。 |
| `sortPackageJson.sortScripts` | `true` | 格式化 `package.json` 時，自動排序 `scripts` 區段。確保腳本按字母順序排列，便於查找。 |
| `tabWidth` | `2` | 縮排寬度為 2 個空格。 |
| `useTabs` | `false` | 使用空格而非 Tab 字元縮排。 |

### 3.3 忽略模式

```jsonc
"ignorePatterns": [
  "apps/", "assets/", "dist/", "node_modules/",
  "patches/", "vendor/"
]
```

注意與 oxlint 忽略模式的差異：
- oxfmt 額外忽略了 `apps/`（可能是獨立的應用程式目錄）
- oxfmt 不忽略 `skills/`（Skills 的格式化仍然由 oxfmt 管理）

---

## 4. Vitest — 測試框架

### 4.1 設定入口

```typescript
// source-repo/vitest.config.ts:1-7
export {
  default,
  resolveDefaultVitestPool,
  resolveLocalVitestMaxWorkers,
  resolveLocalVitestScheduling,
  rootVitestProjects,
} from "./test/vitest/vitest.config.ts";
```

### 4.2 設定架構分析

此檔案是一個**重匯出**（Re-export），將實際的設定邏輯委託給 `test/vitest/vitest.config.ts`。匯出的具名項目暗示了 Vitest 的設定結構：

| 匯出項目 | 推斷用途 |
|----------|----------|
| `default` | 預設的 Vitest 設定物件 |
| `resolveDefaultVitestPool` | 決定預設的測試執行池（Pool）。Vitest 支援多種 Pool 模式：threads（工作執行緒）、forks（子行程）、vmThreads（VM 隔離）等。 |
| `resolveLocalVitestMaxWorkers` | 決定本地開發時的最大工作者數量。可能根據 CPU 核心數動態計算。 |
| `resolveLocalVitestScheduling` | 決定本地開發時的測試排程策略。可能控制測試的執行順序和並行度。 |
| `rootVitestProjects` | 定義根層級的 Vitest 專案清單。Vitest 支援多專案模式（Workspace），每個專案可以有不同的設定。 |

### 4.3 測試基礎設施推斷

根據匯出項目名稱，可以推斷 OpenClaw 的測試基礎設施：

1. **多專案架構**：`rootVitestProjects` 暗示存在多個測試專案，每個可能對應不同的程式碼區域（核心、擴充套件、UI 等）。
2. **自訂排程**：`resolveLocalVitestScheduling` 暗示測試執行順序是經過最佳化的（可能優先執行最近失敗的測試）。
3. **動態並行度**：`resolveLocalVitestMaxWorkers` 暗示並行度根據環境動態調整（CI 環境 vs. 本地開發可能使用不同的工作者數量）。

### 4.4 覆蓋率工具

| 套件 | 版本 | 說明 |
|------|------|------|
| `@vitest/coverage-v8` | ^4.1.4 | 使用 V8 JavaScript 引擎的內建覆蓋率功能。相比 Istanbul，V8 覆蓋率的優勢在於不需要程式碼插裝（Instrumentation），對效能影響更小。 |

---

## 5. knip — 死碼偵測工具

### 5.1 概覽

| 面向 | 說明 |
|------|------|
| 設定檔 | `source-repo/knip.config.ts`（1-122 行） |
| 用途 | 偵測未使用的匯出（Unused Exports）、未使用的依賴（Unused Dependencies）、未使用的檔案（Unused Files） |
| 重要性 | 在 25+ 進入點的大型專案中，死碼的累積是一個嚴重的維護負擔 |

### 5.2 設定結構

knip 的設定檔長達 122 行，結構包括：

| 區塊 | 說明 |
|------|------|
| 根進入點（Root Entries） | 定義專案的主要進入點，knip 從這些進入點開始追蹤程式碼引用鏈 |
| Bundled Plugin 進入點 | 動態定義的 Bundled Plugin 進入點 |
| Workspace 設定 | 定義各個 Workspace 的進入點和忽略規則 |
| 忽略規則 | 排除測試檔案、腳本、fixture 檔案等不應被偵測的程式碼 |

### 5.3 與 tsdown 的互補

knip 和 tsdown 是互補的工具：
- **tsdown** 負責打包，需要知道所有進入點
- **knip** 負責偵測死碼，也需要知道所有進入點

兩者共用相同的進入點資訊，確保一致性。

---

## 6. pre-commit hooks

### 6.1 設定檔

```yaml
# source-repo/.pre-commit-config.yaml（概念還原）
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    hooks:
      - id: trailing-whitespace        # 移除行尾空白
      - id: end-of-file-fixer          # 確保檔案以換行結尾
      - id: check-yaml                 # YAML 語法檢查
      - id: check-added-large-files    # 大檔案偵測（500KB 上限）
        args: ['--maxkb=500']
      - id: check-merge-conflict       # 合併衝突標記偵測
      - id: detect-private-key         # 私鑰偵測

  - repo: https://github.com/Yelp/detect-secrets
    hooks:
      - id: detect-secrets             # 密鑰洩漏偵測
```

### 6.2 Hook 清單解析

| Hook | 類別 | 說明 |
|------|------|------|
| `trailing-whitespace` | 格式 | 自動移除行尾多餘的空白字元。防止無意義的 diff 差異。 |
| `end-of-file-fixer` | 格式 | 確保每個檔案都以換行字元結尾。這是 POSIX 標準的要求。 |
| `check-yaml` | 語法 | 驗證 YAML 檔案的語法正確性。防止格式錯誤的 YAML 被提交。 |
| `check-added-large-files` | 品質 | 偵測新增的大檔案（超過 500KB）。防止二進位檔案或大型資料檔被意外提交到 Git。 |
| `check-merge-conflict` | 品質 | 偵測合併衝突標記。與 `pnpm check:no-conflict-markers` 形成雙重防線。 |
| `detect-private-key` | 安全 | 偵測私鑰格式的內容。防止 SSH 私鑰、TLS 私鑰等被提交。 |
| `detect-secrets` | 安全 | Yelp 的密鑰偵測工具。使用啟發式演算法偵測 API 金鑰、密碼、權杖等。 |

### 6.3 與 CI 的關係

pre-commit hooks 是「第零層防線」——在程式碼被提交到 Git 之前就攔截問題。即使開發者繞過了 pre-commit hooks（如使用 `--no-verify`），CI 中的相應檢查仍然會捕捉到問題。

```
防線層級：
  ├─ 第零層：pre-commit hooks（本地，提交前）
  ├─ 第一層：CI 快速檢查（remote，PR 建立後）
  ├─ 第二層：CI 完整測試（remote，PR 建立後）
  └─ 第三層：CI 多平台測試（remote，PR 建立後）
```

---

## 7. 其他品質工具

### 7.1 jscpd — 複製貼上偵測

| 面向 | 說明 |
|------|------|
| 設定檔 | `source-repo/.jscpd.json` |
| 版本 | 4.0.9 |
| 用途 | 偵測程式碼中的重複片段（Code Duplication / Copy-Paste） |
| 重要性 | 重複程式碼是維護負擔的重要來源。自動偵測有助於及時發現並重構。 |

### 7.2 madge — 匯入循環偵測

| 面向 | 說明 |
|------|------|
| 版本 | ^8.0.0 |
| 用途 | 分析 JavaScript/TypeScript 模組的依賴圖，偵測循環匯入（Circular Imports） |
| 觸發腳本 | `pnpm check:madge-import-cycles` |
| 與自訂腳本的關係 | 與 `pnpm check:import-cycles`（自訂腳本）形成雙重偵測 |

### 7.3 markdownlint — Markdown 文件檢查

| 面向 | 說明 |
|------|------|
| 設定檔 | `source-repo/.markdownlint-cli2.jsonc` |
| 用途 | 確保 Markdown 文件遵循一致的格式規範 |

### 7.4 ShellCheck — Shell 腳本檢查

| 面向 | 說明 |
|------|------|
| 設定檔 | `source-repo/.shellcheckrc` |
| 用途 | 靜態分析 Shell 腳本（`.sh`），偵測常見錯誤和不安全的用法 |

### 7.5 detect-secrets — 密鑰偵測

| 面向 | 說明 |
|------|------|
| 設定檔 | `source-repo/.detect-secrets.cfg` |
| 來源 | Yelp |
| 用途 | 偵測程式碼中的密鑰、API 金鑰、密碼等敏感資訊 |
| 使用場景 | pre-commit hook + CI |

### 7.6 Swift 工具鏈

OpenClaw 有 iOS/macOS 原生應用程式碼，使用以下 Swift 工具：

| 工具 | 設定檔 | 用途 |
|------|--------|------|
| SwiftFormat | `source-repo/.swiftformat` | Swift 程式碼格式化 |
| SwiftLint | `source-repo/.swiftlint.yml` | Swift 程式碼靜態分析 |

### 7.7 zizmor — GitHub Actions 安全掃描

| 面向 | 說明 |
|------|------|
| 設定檔 | `source-repo/zizmor.yml` |
| 用途 | 專門掃描 GitHub Actions 工作流程檔案的安全問題 |
| 偵測範圍 | 不安全的 `${{ }}` 表達式、權限過大的 GITHUB_TOKEN、不安全的 Action 引用等 |

---

## 8. 品質工具全景圖

```
┌──────────────────────────────────────────────────────────┐
│                    品質保障全景圖                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌── 安全層 ──────────────────────────────────────────┐  │
│  │ detect-secrets · detect-private-key · codeql        │  │
│  │ zizmor（GitHub Actions 安全）                       │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌── 格式層 ──────────────────────────────────────────┐  │
│  │ oxfmt（TypeScript/JavaScript）                      │  │
│  │ SwiftFormat（Swift）                                │  │
│  │ markdownlint（Markdown）                            │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌── 靜態分析層 ──────────────────────────────────────┐  │
│  │ oxlint（TypeScript/JavaScript）                     │  │
│  │ SwiftLint（Swift）                                  │  │
│  │ ShellCheck（Shell）                                 │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌── 型別安全層 ──────────────────────────────────────┐  │
│  │ tsgo / tsc（TypeScript 型別檢查）                   │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌── 架構守衛層 ──────────────────────────────────────┐  │
│  │ import cycles（madge + 自訂腳本）                   │  │
│  │ LOC 限制（500 行/檔案）                             │  │
│  │ 架構邊界（30+ lint 腳本）                           │  │
│  │ 死碼偵測（knip）                                    │  │
│  │ 重複偵測（jscpd）                                   │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌── 測試層 ──────────────────────────────────────────┐  │
│  │ Vitest（單元/整合測試）                              │  │
│  │ E2E 測試 · Live 測試 · Docker 測試                  │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌── CI/CD 層 ────────────────────────────────────────┐  │
│  │ GitHub Actions（20+ 工作、多平台、分片策略）          │  │
│  │ 17+ 專用工作流程（發布、文件、安全等）                │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 9. 關鍵設計決策總結

### 9.1 全面擁抱 Oxc 生態系

OpenClaw 選擇了 oxlint + oxfmt 的組合，而非傳統的 ESLint + Prettier。Oxc 生態系的工具全部基於 Rust 實作，提供數量級的效能提升。這對擁有 100+ 進入點的大型專案而言，是直接影響開發體驗的關鍵選擇。

### 9.2 零容忍的 `any` 政策

`typescript/no-explicit-any: "error"` 在生產程式碼中禁止使用 `any`，只在測試程式碼中放寬。這是 TypeScript 專案中最嚴格的型別安全策略之一。

### 9.3 智慧 CI 觸發

`preflight` 工作的變更偵測機制確保 CI 資源不被浪費。在大型開源專案中，每個 PR 都跑完整的 20+ 工作會消耗大量 CI 分鐘數。智慧觸發策略可能將 CI 時間減少 50% 以上。

### 9.4 多語言品質保障

OpenClaw 的品質工具不限於 TypeScript——還包括 Swift（SwiftFormat、SwiftLint）、Shell（ShellCheck）、Python（Skills 檢查）、Markdown（markdownlint）。這反映了 OpenClaw 作為跨平台框架的完整性。

### 9.5 防線深度

從 pre-commit hooks 到 CI 的多平台測試，OpenClaw 建立了至少 4 層防線。這種「縱深防禦」（Defense in Depth）策略確保了即使某一層被繞過，後續的層仍能捕捉到問題。

---

## 引用來源

| 來源檔案 | 行號範圍 | 引用內容 |
|----------|----------|----------|
| `source-repo/.github/workflows/ci.yml` | 1-1378 | 完整 CI 流水線定義 |
| `source-repo/.oxlintrc.json` | 1-67 | oxlint 設定完整內容 |
| `source-repo/.oxlintrc.json` | 3 | plugins 清單 |
| `source-repo/.oxlintrc.json` | 4-8 | categories 設定 |
| `source-repo/.oxlintrc.json` | 9-16 | rules 設定 |
| `source-repo/.oxlintrc.json` | 17-20 | ignorePatterns |
| `source-repo/.oxlintrc.json` | 21-28 | overrides（測試檔案規則） |
| `source-repo/.oxfmtrc.jsonc` | 1-20 | oxfmt 設定完整內容 |
| `source-repo/vitest.config.ts` | 1-7 | Vitest 設定入口 |
| `source-repo/knip.config.ts` | 1-122 | knip 死碼偵測設定 |
| `source-repo/.pre-commit-config.yaml` | 1-30+ | pre-commit hooks 設定 |
| `source-repo/.jscpd.json` | — | 複製貼上偵測設定 |
| `source-repo/.markdownlint-cli2.jsonc` | — | Markdown lint 設定 |
| `source-repo/.shellcheckrc` | — | ShellCheck 設定 |
| `source-repo/.swiftformat` | — | Swift 格式化設定 |
| `source-repo/.swiftlint.yml` | — | Swift lint 設定 |
| `source-repo/.detect-secrets.cfg` | — | 密鑰偵測設定 |
| `source-repo/zizmor.yml` | — | GitHub Actions 安全掃描設定 |
| `source-repo/package.json` | devDependencies | 開發依賴版本資訊 |
