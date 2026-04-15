# 01 — pnpm Workspace 配置

> **證據層級**：Repository Evidence
> **對應版本**：OpenClaw 2026.4.12
> **主要來源**：`source-repo/pnpm-workspace.yaml`、`source-repo/.npmrc`

---

## 本章摘要

OpenClaw 使用 **pnpm workspace** 來管理其 monorepo。本章完整記錄 `pnpm-workspace.yaml` 和 `.npmrc` 這兩個配置檔的內容，並逐一解釋每個欄位的含義與作用。這兩個檔案共同定義了三件事：（1）哪些目錄被視為 workspace 成員、（2）依賴更新的安全策略、（3）原生模組的建置許可清單。對於想理解 OpenClaw 如何管理數百個子套件的讀者，這是必要的起點。

---

## 1. pnpm-workspace.yaml 全文與逐段解析

### 1.1 檔案全文

以下是 `source-repo/pnpm-workspace.yaml:1-45` 的完整內容：

```yaml
packages:
  - .
  - ui
  - packages/*
  - extensions/*

minimumReleaseAge: 2880

minimumReleaseAgeExclude:
  - "acpx"
  - "axios"
  - "basic-ftp"
  - "hono"
  - "openclaw"
  - "@buape/carbon"
  - "vite"
  - "@cloudflare/workers-types"
  - "@hono/node-server"
  - "@mariozechner/*"
  - "@typescript/native-preview*"
  - "@types/node"
  - "@rolldown/*"
  - "@oxlint/*"
  - "@oxfmt/*"
  - "axios@1.15.0"
  - "discord-api-types"
  - "rolldown"
  - "sqlite-vec"
  - "sqlite-vec-*"

onlyBuiltDependencies:
  - "@discordjs/opus"
  - "@lydell/node-pty"
  - "@matrix-org/matrix-sdk-crypto-nodejs"
  - "@napi-rs/canvas"
  - "@tloncorp/api"
  - "@whiskeysockets/baileys"
  - authenticate-pam
  - esbuild
  - node-llama-cpp
  - protobufjs
  - sharp

ignoredBuiltDependencies:
  - koffi
```

### 1.2 `packages` 欄位——Workspace 成員定義

```yaml
packages:
  - .
  - ui
  - packages/*
  - extensions/*
```

這是整個 monorepo 組織的核心宣告。它告訴 pnpm：「以下 glob 模式匹配到的目錄，每一個都是一個獨立的 workspace 套件（workspace package）。」

#### 逐項解析

| Glob 模式 | 匹配對象 | 意義 |
|-----------|---------|------|
| `.` | 倉庫根目錄本身 | 根目錄的 `package.json`（名為 `openclaw`）也是一個 workspace 成員。這表示根目錄不僅僅是 workspace 的容器，它本身就是主要的可發佈套件。 |
| `ui` | `ui/` 目錄 | Web UI 介面，是一個獨立的 workspace 套件。注意它不使用 glob（沒有 `*`），表示 `ui/` 下只有一個套件。 |
| `packages/*` | `packages/` 下的所有第一層子目錄 | 共享的內部套件。目前有 3 個：`plugin-sdk`、`memory-host-sdk`、`plugin-package-contract`。 |
| `extensions/*` | `extensions/` 下的所有第一層子目錄 | 所有擴充套件。目前有 111 個項目（包含一個 `shared/` 公用目錄）。 |

#### 重要觀察：哪些目錄**不是** workspace 成員

以下目錄存在於倉庫中，但**沒有**被列入 `packages` 欄位：

| 目錄 | 內容 | 為何不是 workspace 成員 |
|------|------|----------------------|
| `src/` | 核心原始碼 | `src/` 是根套件 `.` 的一部分，不是獨立的 workspace 成員。它被編譯為根套件的 `dist/` 輸出。 |
| `skills/` | 53 個技能包 | 技能包不作為 workspace 套件管理。這暗示技能可能在運行時動態載入，而非在建置時作為依賴解析。 |
| `apps/` | 4 個原生應用 | 原生應用（Android, iOS, macOS）不參與 pnpm workspace 的依賴解析。 |
| `docs/` | 文件網站 | 文件不是可安裝的套件。 |
| `vendor/` | 第三方vendored程式碼 | 直接被原始碼引用，不走 npm 套件機制。 |
| `test/`、`test-fixtures/` | 測試程式碼與資料 | 測試程式碼屬於根套件的一部分。 |

> **關鍵發現**：`skills/` 不是 workspace 成員這一點非常值得注意。在 monorepo 中，workspace 成員之間可以互相引用且 pnpm 會自動做 symlink。`skills/` 被排除意味著它們走的是不同的機制——很可能是作為「發佈時附帶的資產」而非「建置時的依賴」。這與 `package.json` 的 `files` 欄位中包含 `"skills/"` 一致（詳見 [03-package-json-key-fields.md](./03-package-json-key-fields.md)）。

### 1.3 `minimumReleaseAge` 欄位——依賴更新安全閾值

```yaml
minimumReleaseAge: 2880
```

這個欄位是 pnpm 的安全功能，單位是**分鐘**。`2880` 分鐘 = **48 小時（2 天）**。

**含義**：當 pnpm 解析依賴並選擇版本時，它會拒絕使用發佈時間不到 48 小時的套件版本。這是一種供應鏈安全（Supply Chain Security）措施，目的是：

1. **防範惡意套件**：如果某個套件被攻擊者接管並發佈了惡意版本，48 小時的緩衝期給了社群足夠的時間來發現和撤回（unpublish）該版本
2. **避免不穩定版本**：剛發佈的版本可能存在未被發現的問題，等待 48 小時可以讓早期採用者先踩坑
3. **自動化安全**：這種保護是全自動的，不需要開發者手動審查每次依賴更新

### 1.4 `minimumReleaseAgeExclude` 欄位——豁免清單

```yaml
minimumReleaseAgeExclude:
  - "acpx"
  - "axios"
  - "basic-ftp"
  - "hono"
  - "openclaw"
  - "@buape/carbon"
  - "vite"
  - "@cloudflare/workers-types"
  - "@hono/node-server"
  - "@mariozechner/*"
  - "@typescript/native-preview*"
  - "@types/node"
  - "@rolldown/*"
  - "@oxlint/*"
  - "@oxfmt/*"
  - "axios@1.15.0"
  - "discord-api-types"
  - "rolldown"
  - "sqlite-vec"
  - "sqlite-vec-*"
```

這些套件被豁免於 48 小時等待期。原因各有不同，我們可以將它們分類：

| 分類 | 套件 | 可能的豁免原因 |
|------|------|--------------|
| **自身套件** | `openclaw`、`acpx` | 自家發佈的套件，不需要等待 |
| **核心工具鏈** | `vite`、`@rolldown/*`、`rolldown`、`@oxlint/*`、`@oxfmt/*`、`@typescript/native-preview*` | 建置工具鏈的更新通常需要即時跟進，且這些工具有活躍的社群把關 |
| **型別定義** | `@types/node`、`@cloudflare/workers-types`、`discord-api-types` | 型別定義套件風險極低（不含可執行程式碼），且需要即時匹配對應的運行時版本 |
| **特定版本釘選** | `axios@1.15.0` | 某個特定 axios 版本被豁免，可能是因為它修復了關鍵 bug 且需要立即採用 |
| **快速迭代的依賴** | `basic-ftp`、`hono`、`@hono/node-server`、`@buape/carbon`、`@mariozechner/*`、`sqlite-vec`、`sqlite-vec-*`、`axios` | 這些套件要麼迭代速度快，要麼是信任的上游維護者 |

> **觀察**：豁免清單中有不少與 Discord 相關（`@buape/carbon`、`discord-api-types`）和語音/多媒體相關（`sqlite-vec`）的套件，反映了 OpenClaw 作為多通道（multi-channel）AI 閘道器的核心定位。

### 1.5 `onlyBuiltDependencies` 欄位——建置腳本許可清單

```yaml
onlyBuiltDependencies:
  - "@discordjs/opus"
  - "@lydell/node-pty"
  - "@matrix-org/matrix-sdk-crypto-nodejs"
  - "@napi-rs/canvas"
  - "@tloncorp/api"
  - "@whiskeysockets/baileys"
  - authenticate-pam
  - esbuild
  - node-llama-cpp
  - protobufjs
  - sharp
```

這是 pnpm 的另一個安全功能。在 npm 生態系中，套件可以在安裝時透過 `postinstall`（安裝後腳本）等 lifecycle scripts 執行任意程式碼。這是一個嚴重的安全風險——惡意套件可以藉此在你的機器上執行任意命令。

`onlyBuiltDependencies` 的語義是：**只有列在這裡的套件被允許在安裝時執行建置腳本**。所有其他套件的安裝腳本都會被靜默跳過。

#### 逐一分析被允許的套件

| 套件 | 需要建置腳本的原因 | 在 OpenClaw 中的角色 |
|------|-------------------|---------------------|
| `@discordjs/opus` | 原生（C/C++）音訊編碼器，需要編譯 | Discord 語音通道的音訊編碼/解碼 |
| `@lydell/node-pty` | 偽終端機（pseudo-terminal）原生綁定 | 終端機互動功能（CLI、shell 整合） |
| `@matrix-org/matrix-sdk-crypto-nodejs` | Matrix 協議的加密函式庫，原生 Rust 綁定 | Matrix 通道的端到端加密 |
| `@napi-rs/canvas` | Canvas 2D 渲染引擎的原生綁定 | 圖像生成/處理功能 |
| `@tloncorp/api` | Tlon（Urbit 生態）的 API 綁定 | Tlon 通道整合 |
| `@whiskeysockets/baileys` | WhatsApp Web 協議的實作 | WhatsApp 通道整合 |
| `authenticate-pam` | Linux PAM 認證的原生綁定 | 系統層級的用戶認證 |
| `esbuild` | Go 語言實作的打包器，安裝時下載對應平台的二進位 | 快速的 JavaScript/TypeScript 打包 |
| `node-llama-cpp` | llama.cpp 的 Node.js 綁定，需要編譯 C++ 程式碼 | 本地端 LLM 推理（在設備上運行大型語言模型） |
| `protobufjs` | Protocol Buffers 的 JavaScript 實作 | 資料序列化（可能用於 gRPC 或高效資料傳輸） |
| `sharp` | libvips 影像處理函式庫的原生綁定 | 高效能圖像處理（縮放、格式轉換等） |

> **關鍵洞察**：從這份清單中可以看出 OpenClaw 的技術面貌——它不僅僅是「一個聊天機器人框架」。它需要**音訊編碼**（`@discordjs/opus`）、**本地 LLM 推理**（`node-llama-cpp`）、**影像處理**（`sharp`, `@napi-rs/canvas`）、**加密通訊**（`@matrix-org/matrix-sdk-crypto-nodejs`）、**系統認證**（`authenticate-pam`）。這些原生依賴的存在揭示了 OpenClaw 是一個對系統層級有深度整合需求的專案。

### 1.6 `ignoredBuiltDependencies` 欄位——被忽略的建置依賴

```yaml
ignoredBuiltDependencies:
  - koffi
```

`koffi` 是一個 FFI（Foreign Function Interface，外部函式介面）函式庫，用於在 Node.js 中呼叫 C 函式庫。它被列在 `ignoredBuiltDependencies` 而非 `onlyBuiltDependencies` 中，意味著 pnpm 會完全忽略它的建置腳本。

**與 `onlyBuiltDependencies` 的區別**：

| 欄位 | 效果 |
|------|------|
| `onlyBuiltDependencies` 中列出 | 允許執行建置腳本 |
| `ignoredBuiltDependencies` 中列出 | 明確禁止執行建置腳本，且不會發出警告 |
| 兩個清單都沒列出 | 禁止執行建置腳本，可能會發出警告 |

`koffi` 被特別忽略，可能是因為：它是某個其他依賴的傳遞依賴（transitive dependency），OpenClaw 不直接使用它的原生功能，但又不想在安裝時看到警告訊息。

---

## 2. .npmrc 配置——node-linker 策略

### 2.1 檔案全文

以下是 `source-repo/.npmrc:1-4` 的完整內容：

```
# pnpm build-script allowlist lives in package.json -> pnpm.onlyBuiltDependencies.
# TS 7 native-preview fails to resolve packages reliably from pnpm's isolated linker.
# Keep the workspace on a hoisted layout so pnpm check/build stay stable.
node-linker=hoisted
```

### 2.2 `node-linker=hoisted` 的含義

pnpm 有三種套件連結策略（node-linker modes）：

| 模式 | `node_modules` 結構 | 特點 |
|------|---------------------|------|
| `isolated`（預設） | 嵌套的 symlink 結構 | 嚴格的依賴隔離，套件只能存取自己宣告的依賴 |
| `hoisted` | 扁平化結構（類似 npm v3+） | 所有依賴都提升到根目錄的 `node_modules`，與 npm/yarn 行為相同 |
| `pnp` | 無 `node_modules` 目錄 | 使用 Plug'n'Play 機制（Yarn PnP 風格） |

OpenClaw 選擇了 `hoisted` 模式。根據 `.npmrc` 中的註解，原因是：

> **TypeScript 7 native-preview 無法從 pnpm 的 isolated linker 可靠地解析套件**

這是一個實際的工具相容性問題。TypeScript 的原生（native，用 Go/Rust 重寫的版本）預覽版在解析模組路徑時，對 pnpm 的 isolated linker 產生的 symlink 結構處理不佳。為了確保 `pnpm check`（型別檢查）和 `pnpm build`（建置）能穩定運作，OpenClaw 被迫放棄了 pnpm 預設的嚴格隔離模式。

### 2.3 Hoisted 模式的取捨

選擇 `hoisted` 模式意味著以下取捨：

**放棄的好處**：
- 依賴隔離：套件可以「偷偷」存取未宣告的依賴（phantom dependencies），這可能導致在獨立安裝時出現問題
- 磁碟空間效率：`isolated` 模式利用 symlink 減少重複，`hoisted` 模式可能導致更多重複

**獲得的好處**：
- 與 TypeScript 7 native-preview 的完全相容性
- 與依賴扁平化結構的所有工具（如某些 bundler、linter）的相容性
- 更簡單的除錯體驗——`node_modules` 結構與預期一致

> **歷史脈絡**：這個配置暗示 OpenClaw 團隊正在積極跟進 TypeScript 最新的原生（native）版本。這也與 `devDependencies` 中出現 `@typescript/native-preview` 7.x 一致（詳見 [03-package-json-key-fields.md](./03-package-json-key-fields.md)）。

---

## 3. Workspace 拓撲圖

根據以上配置，OpenClaw 的 workspace 拓撲可以用以下結構表示：

```
openclaw (workspace root = .)
├── ui/                          ← workspace member（具名）
├── packages/
│   ├── plugin-sdk/              ← workspace member (packages/*)
│   ├── memory-host-sdk/         ← workspace member (packages/*)
│   └── plugin-package-contract/ ← workspace member (packages/*)
├── extensions/
│   ├── discord/                 ← workspace member (extensions/*)
│   ├── telegram/                ← workspace member (extensions/*)
│   ├── openai/                  ← workspace member (extensions/*)
│   ├── anthropic/               ← workspace member (extensions/*)
│   ├── ... (共 111 個)          ← workspace member (extensions/*)
│   └── shared/                  ← workspace member (extensions/*)
│
│ ── 以下不是 workspace 成員 ──
├── src/                         ← 根套件的原始碼
├── skills/                      ← 不是 workspace（運行時資產）
├── apps/                        ← 不是 workspace（原生應用）
├── docs/                        ← 不是 workspace（文件）
├── vendor/                      ← 不是 workspace（vendored 依賴）
├── test/                        ← 不是 workspace（測試）
├── test-fixtures/               ← 不是 workspace（測試資料）
└── ...
```

workspace 成員的總數量：

| 區域 | 數量 |
|------|------|
| 根目錄 `.` | 1 |
| `ui` | 1 |
| `packages/*` | 3 |
| `extensions/*` | ~111 |
| **合計** | **~116** |

這表示 pnpm 在這個 monorepo 中管理著大約 **116 個 workspace 套件**。

---

## 4. 安全模型總結

`pnpm-workspace.yaml` 中的安全相關配置形成了一套三層防禦：

| 層級 | 機制 | 配置 | 防禦目標 |
|------|------|------|----------|
| **第一層** | 建置腳本許可清單 | `onlyBuiltDependencies`（11 個套件） | 防止惡意套件在安裝時執行任意程式碼 |
| **第二層** | 新版本冷卻期 | `minimumReleaseAge: 2880`（48 小時） | 防止攻擊者發佈惡意版本後立即被拉取 |
| **第三層** | 建置腳本靜默忽略 | `ignoredBuiltDependencies`（1 個套件） | 已知安全但不需要建置的依賴，避免警告噪音 |

這套安全模型在 npm 生態系中屬於相當嚴格的配置，反映了 OpenClaw 團隊對供應鏈安全的重視。

---

## 5. 與 pnpm 版本的關聯

`pnpm-workspace.yaml` 中使用的 `minimumReleaseAge`、`minimumReleaseAgeExclude`、`onlyBuiltDependencies`、`ignoredBuiltDependencies` 等欄位都是 pnpm 較新版本才支援的功能。這些功能的存在暗示 OpenClaw 使用的是 pnpm v9+ 或更高版本。具體的 pnpm 版本可在 `package.json` 的 `packageManager` 欄位或 CI 配置中確認。

---

## 引用來源

| 檔案路徑 | 行號範圍 | 本文引用位置 | 內容說明 |
|----------|----------|-------------|----------|
| `source-repo/pnpm-workspace.yaml` | 1-4 | §1.2 | `packages` 欄位——workspace 成員定義 |
| `source-repo/pnpm-workspace.yaml` | 6 | §1.3 | `minimumReleaseAge` 欄位 |
| `source-repo/pnpm-workspace.yaml` | 8-29 | §1.4 | `minimumReleaseAgeExclude` 豁免清單 |
| `source-repo/pnpm-workspace.yaml` | 31-42 | §1.5 | `onlyBuiltDependencies` 建置許可清單 |
| `source-repo/pnpm-workspace.yaml` | 44-45 | §1.6 | `ignoredBuiltDependencies` 忽略清單 |
| `source-repo/.npmrc` | 1-4 | §2.1, §2.2 | node-linker 配置與註解 |
| `source-repo/package.json` | — | §2.3（交叉引用） | devDependencies 中的 TypeScript 版本 |
| `source-repo/packages/` | — | §1.2 | workspace 套件目錄列表 |
| `source-repo/extensions/` | — | §1.2 | 擴充套件目錄列表 |
| `source-repo/skills/` | — | §1.2 | 技能包目錄列表（非 workspace 成員） |
| `source-repo/apps/` | — | §1.2 | 原生應用目錄列表（非 workspace 成員） |
