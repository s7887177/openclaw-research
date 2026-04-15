# 03 — 擴充/核心邊界與 plugin-sdk（Extension/Core Boundary）

> **本章摘要**：OpenClaw 的架構中存在一條至關重要的隱形邊界——「核心程式碼」（Core）與「擴充插件」（Extension）之間的分界線。這條邊界不是靠紳士協定維護的，而是透過自動化腳本強制執行的三條嚴格規則、TypeScript path alias 的精心設計、以及零依賴（zero-dependency）的 workspace packages 來保障。本章完整記錄這個邊界機制的每一個技術細節，包括邊界檢查腳本的三條規則、`extensionAPI.ts` 的棄用遷移路徑、`plugin-sdk` 的公開介面設計、以及三個零依賴 workspace packages 的角色。理解這條邊界，是我們未來開發自定義插件的前提。

---

## 目錄

- [1. 為什麼需要邊界？](#1-為什麼需要邊界)
- [2. 邊界檢查腳本——三條鐵律](#2-邊界檢查腳本三條鐵律)
  - [2.1 規則一：src-outside-plugin-sdk](#21-規則一src-outside-plugin-sdk)
  - [2.2 規則二：plugin-sdk-internal](#22-規則二plugin-sdk-internal)
  - [2.3 規則三：relative-outside-package](#23-規則三relative-outside-package)
  - [2.4 三條規則的交互作用](#24-三條規則的交互作用)
- [3. extensionAPI.ts——舊邊界的棄用](#3-extensionapits舊邊界的棄用)
- [4. TypeScript Path Alias——邊界的路徑映射](#4-typescript-path-alias邊界的路徑映射)
- [5. plugin-sdk 的公開介面設計](#5-plugin-sdk-的公開介面設計)
  - [5.1 根入口的極簡主義](#51-根入口的極簡主義)
  - [5.2 Subpath Exports 策略](#52-subpath-exports-策略)
- [6. 零依賴 Workspace Packages](#6-零依賴-workspace-packages)
  - [6.1 @openclaw/plugin-sdk](#61-openclawplugin-sdk)
  - [6.2 @openclaw/memory-host-sdk](#62-openclawmemory-host-sdk)
  - [6.3 @openclaw/plugin-package-contract](#63-openclawplugin-package-contract)
- [7. 邊界的全景圖](#7-邊界的全景圖)
- [8. 關鍵觀察與結論](#8-關鍵觀察與結論)
- [引用來源](#引用來源)

---

## 1. 為什麼需要邊界？

在深入技術細節之前，先理解「為什麼 OpenClaw 需要在核心與擴充之間畫一條邊界」。

**核心（Core）** 是 OpenClaw 本身的程式碼——`src/` 目錄下的 60+ 個模組，包含 AI 推理引擎、通道整合、記憶系統、CLI 介面等所有功能。核心程式碼的 API 可能隨版本更新而改變，內部模組之間的引用關係也可能重構。

**擴充（Extension / Plugin）** 是第三方開發者或使用者自行開發的插件。它們需要引用 OpenClaw 的某些功能（例如存取設定、發送訊息、讀取記憶），但**不能**依賴 OpenClaw 的內部實作細節——否則每次 OpenClaw 更新都可能破壞插件。

因此，OpenClaw 需要一個**明確的、穩定的、受控的 API 表面（API surface）**——這就是 `plugin-sdk` 的角色。它像一扇門，只暴露核心願意承諾長期維護的介面，將核心的內部變動隔離在門內。

```
┌──────────────────────────────────────────────────┐
│                  Core (src/)                       │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐     │
│  │ agents │ │ config │ │ infra  │ │channels│ ... │
│  └────────┘ └────────┘ └────────┘ └────────┘     │
│                       │                            │
│              ┌────────┴────────┐                   │
│              │   plugin-sdk    │  ← 唯一的出口     │
│              └────────┬────────┘                   │
│ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│                       │                            │
│  ┌────────────────────┴─────────────────────┐     │
│  │          Extensions / Plugins             │     │
│  │  ┌─────────┐ ┌──────────┐ ┌───────────┐  │     │
│  │  │ Plugin A│ │ Plugin B │ │ Plugin C  │  │     │
│  │  └─────────┘ └──────────┘ └───────────┘  │     │
│  └───────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘
```

---

## 2. 邊界檢查腳本——三條鐵律

OpenClaw 透過一個專門的腳本來自動化地強制執行邊界規則。這不是建議、不是 linting 警告——是**硬性約束**，違反即構建失敗。

> 來源：`source-repo/scripts/check-extension-plugin-sdk-boundary.mjs:1-55`

### 2.1 規則一：src-outside-plugin-sdk

**規則 ID**：`src-outside-plugin-sdk`

**規則內容**：Production bundled plugins（生產環境打包的插件）**不得** import `src/**` 目錄下位於 `src/plugin-sdk/**` 之外的任何模組。

**白話解釋**：如果你是一個插件，你能碰到的 `src/` 程式碼只有 `src/plugin-sdk/` 這一個子目錄。其他的——`src/agents/`、`src/config/`、`src/infra/`——全部禁止直接引用。

**違規範例**：

```typescript
// ❌ 違反 src-outside-plugin-sdk
// 在 extensions/my-plugin/index.ts 中：
import { loadConfig } from "../../src/config/config.js";
//                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
// 直接引用 src/config/ —— 觸犯禁令！
```

**合規範例**：

```typescript
// ✅ 正確做法
// 在 extensions/my-plugin/index.ts 中：
import { /* ... */ } from "openclaw/plugin-sdk";
// 或
import { /* ... */ } from "openclaw/plugin-sdk/config";
// 只透過 plugin-sdk 的公開介面存取所需功能
```

**設計意圖**：這條規則確保插件不會「偷偷地」依賴核心的內部模組。即使 TypeScript 編譯器允許你 import 任何檔案（只要路徑正確），邊界檢查腳本會在建置時攔截這種行為。

### 2.2 規則二：plugin-sdk-internal

**規則 ID**：`plugin-sdk-internal`

**規則內容**：Production bundled plugins **不得** import `src/plugin-sdk-internal/**` 目錄下的任何模組。

**白話解釋**：`plugin-sdk/` 之外，還存在一個 `plugin-sdk-internal/` 目錄。顧名思義，它包含「SDK 的內部實作」——那些 plugin-sdk 自身需要、但不願暴露給插件開發者的實作細節。

**為什麼需要這條額外規則？**

規則一已經禁止了 `src/**`（plugin-sdk 之外），但 `plugin-sdk-internal/` 的名稱可能讓開發者誤以為它是 `plugin-sdk` 的一部分而嘗試引用。這條規則明確地封死了這個可能的誤區。

```
src/
├── plugin-sdk/           ← ✅ 插件可以引用
├── plugin-sdk-internal/  ← ❌ 插件不可引用（即使名字看起來相關）
├── agents/               ← ❌ 插件不可引用
├── config/               ← ❌ 插件不可引用
└── ...
```

### 2.3 規則三：relative-outside-package

**規則 ID**：`relative-outside-package`

**規則內容**：Production bundled plugins **不得**使用相對 import 路徑（`../`）逃逸出自身的 package root。

**白話解釋**：插件不能用 `../../src/anything` 這樣的相對路徑來「翻牆」。

**違規範例**：

```typescript
// ❌ 違反 relative-outside-package
// 在 extensions/my-plugin/src/helper.ts 中：
import { something } from "../../../src/infra/env.js";
//                        ^^^^^^^^^^^
// 相對路徑逃逸出 extensions/my-plugin/ 目錄
```

**設計意圖**：即使規則一和規則二封鎖了以 `src/` 開頭的絕對 import，聰明的開發者可能嘗試用相對路徑繞過。規則三堵住了這個漏洞——你的相對 import 只能指向你自己 package 內部的檔案。

### 2.4 三條規則的交互作用

三條規則形成了一個完整的「圍欄」：

| 規則 | 防禦的攻擊向量 |
|------|---------------|
| `src-outside-plugin-sdk` | 透過 TypeScript path alias 或絕對 import 直接引用核心模組 |
| `plugin-sdk-internal` | 誤將 SDK 內部實作當作公開 API 引用 |
| `relative-outside-package` | 透過相對路徑 `../` 逃逸 package 邊界 |

**三層防禦的邏輯**：

```
                     插件的 import 語句
                            │
                            ▼
              ┌─ 是否引用了 src/** (非 plugin-sdk)? ─── 是 → ❌ 規則一
              │
              ├─ 是否引用了 plugin-sdk-internal/**? ─── 是 → ❌ 規則二
              │
              ├─ 相對路徑是否逃出 package root? ──────── 是 → ❌ 規則三
              │
              └─ 以上皆否 ─────────────────────────────── → ✅ 允許
```

---

## 3. extensionAPI.ts——舊邊界的棄用

在 `plugin-sdk` 成為正式的邊界介面之前，OpenClaw 曾有一個較早期的擴充 API 入口點：`extensionAPI.ts`。

```typescript
// source-repo/src/extensionAPI.ts:1-30
// `openclaw/extension-api` 已被棄用，使用時會發出警告
// 插件應改用 `openclaw/plugin-sdk/<subpath>` imports
// extensionAPI.ts 仍然 re-export 了 agents/*, config/*, infra/* 等模組
```

**棄用遷移路徑**：

```
舊方式（已棄用）：
import { something } from "openclaw/extension-api";
                          ^^^^^^^^^^^^^^^^^^^^^^^^
                          → 觸發 deprecation warning

新方式（建議）：
import { something } from "openclaw/plugin-sdk/subpath";
                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                          → 正確的 plugin-sdk 子路徑引用
```

**關鍵細節**：

1. **`extensionAPI.ts` 仍然存在且仍然可用**——它不是被移除了，而是被標記為「棄用」（deprecated）。使用它會在執行時產生警告訊息，但功能本身仍然運作。
2. **它 re-export 了跨越多個核心模組的功能**——`agents/*`、`config/*`、`infra/*` 等。這正是它被棄用的原因：它暴露了過多的核心內部，違反了最小介面原則。
3. **`plugin-sdk` 是它的替代品**——新的 SDK 透過精心設計的 subpath exports，只暴露「插件真正需要的功能」，而非核心的全部。

**這個棄用過程告訴我們什麼？**

OpenClaw 的邊界設計是**演進式的**——它不是一開始就完美的。最初的 `extensionAPI.ts` 是一個「大而全」的 re-export，隨著專案成熟，團隊意識到需要更精細的控制，才設計了 `plugin-sdk` 及其三條邊界規則。

---

## 4. TypeScript Path Alias——邊界的路徑映射

TypeScript 的 path alias 機制是邊界實作的關鍵技術基礎。它決定了插件開發者在 `import` 語句中寫的路徑會被解析到哪個實際檔案。

```json
// source-repo/tsconfig.json:22-28
{
  "compilerOptions": {
    "paths": {
      "openclaw/extension-api": ["./src/extensionAPI.ts"],
      "openclaw/plugin-sdk": ["./src/plugin-sdk/index.ts"],
      "openclaw/plugin-sdk/*": ["./src/plugin-sdk/*.ts"],
      "@openclaw/plugin-sdk": ["./src/plugin-sdk/index.ts"],
      "@openclaw/plugin-sdk/*": ["./src/plugin-sdk/*.ts"],
      "openclaw/plugin-sdk/account-id": ["./src/plugin-sdk/account-id.ts"],
      "@openclaw/*": ["./extensions/*"]
    }
  }
}
```

### Path Alias 映射表

| import 路徑 | 實際解析目標 | 用途 |
|-------------|-------------|------|
| `openclaw/extension-api` | `./src/extensionAPI.ts` | 舊版擴充 API（已棄用） |
| `openclaw/plugin-sdk` | `./src/plugin-sdk/index.ts` | plugin-sdk 根入口 |
| `openclaw/plugin-sdk/*` | `./src/plugin-sdk/*.ts` | plugin-sdk 的 subpath exports |
| `@openclaw/plugin-sdk` | `./src/plugin-sdk/index.ts` | 同上，npm scope 風格的別名 |
| `@openclaw/plugin-sdk/*` | `./src/plugin-sdk/*.ts` | 同上，npm scope 風格的 subpath |
| `openclaw/plugin-sdk/account-id` | `./src/plugin-sdk/account-id.ts` | 特定 subpath 的顯式映射 |
| `@openclaw/*` | `./extensions/*` | 擴充目錄——所有 `@openclaw/` 前綴映射到 `extensions/` |

**關鍵觀察**：

### 4.1 雙命名慣例並存

`openclaw/plugin-sdk` 和 `@openclaw/plugin-sdk` 都指向同一個檔案。這意味著插件開發者可以使用任一風格：

```typescript
// 兩種寫法等效：
import { X } from "openclaw/plugin-sdk";
import { X } from "@openclaw/plugin-sdk";
```

這可能是歷史演進的結果——早期使用 `openclaw/` 前綴，後來遷移到更標準的 npm scope `@openclaw/`，為了相容性保留了兩者。

### 4.2 `@openclaw/*` 是擴充的命名空間

```json
"@openclaw/*": ["./extensions/*"]
```

這行設定把 `@openclaw/` 前綴（排除已被更具體規則匹配的 `@openclaw/plugin-sdk`）映射到 `extensions/` 目錄。這意味著：

```typescript
import { X } from "@openclaw/my-custom-plugin";
// 實際解析到: ./extensions/my-custom-plugin
```

每個擴充在 `extensions/` 目錄下有自己的子目錄，以 `@openclaw/` scope 發佈或引用。

### 4.3 account-id 的顯式映射

```json
"openclaw/plugin-sdk/account-id": ["./src/plugin-sdk/account-id.ts"]
```

`account-id` 被單獨列出，而非依賴通配符 `openclaw/plugin-sdk/*` 的規則。這可能是因為 `account-id` 是一個特殊的子路徑，需要確保在所有環境中都能正確解析（某些 TypeScript 版本對通配符的處理可能有邊界情況）。

---

## 5. plugin-sdk 的公開介面設計

### 5.1 根入口的極簡主義

```typescript
// source-repo/src/plugin-sdk/index.ts:1-30
// 根 plugin-sdk 介面刻意保持極小（intentionally kept tiny）
// 匯出來源：
//   - channels/plugins/types.public.js   ← 通道插件的公開型別
//   - channels/plugins/types.adapters.js ← 通道適配器型別
//   - channels/plugins/types.config.js   ← 通道設定型別
//   - channels/plugins/types.plugin.js   ← 插件基礎型別
// 通道/提供者的 helper 函式放在專用的 subpath 上
```

**設計哲學——「小根入口，豐富子路徑」**：

`plugin-sdk/index.ts` 只匯出最基本的型別定義——那些幾乎所有插件都會用到的公共介面。更具體的功能（通道 helper、提供者 helper 等）被分散到各個 subpath：

```
openclaw/plugin-sdk               ← 極簡：只有核心型別
openclaw/plugin-sdk/config         ← 設定存取
openclaw/plugin-sdk/channels       ← 通道操作
openclaw/plugin-sdk/providers      ← 模型提供者
openclaw/plugin-sdk/account-id     ← 帳號識別
openclaw/plugin-sdk/...            ← 其他 ~36 個 subpath
```

**為什麼不把所有東西放在根入口？**

1. **Tree-shaking 友善**：如果根入口匯出一切，即使插件只需要其中一小部分，bundler 也需要分析整個 SDK。分散到 subpath 後，每個 import 只載入需要的部分。
2. **API 演進的靈活性**：新增一個 subpath 不會影響現有的根入口消費者。根入口保持穩定，新功能透過新 subpath 添加。
3. **明確的功能邊界**：`import from "openclaw/plugin-sdk/channels"` 比 `import { channelHelper } from "openclaw/plugin-sdk"` 更能表達意圖。

### 5.2 Subpath Exports 策略

根據 `@openclaw/plugin-sdk` workspace package 的定義，共有約 **40 個 subpath exports**：

```
packages/plugin-sdk/
└── 每個 subpath export 映射到 src/ 下的一個 .ts 檔案
    約 40 個 subpath，涵蓋型別定義、通道操作、設定存取等功能
```

這約 40 個 subpath 就是「門」——插件開發者能存取的全部功能範圍。任何不在這 40 個 subpath 中的核心功能，插件都無法（合法地）觸及。

**根入口匯出的四個型別來源**：

| 匯出來源 | 提供的功能 |
|----------|-----------|
| `channels/plugins/types.public.js` | 通道插件的公開型別定義——插件與通道系統互動時需要的型別 |
| `channels/plugins/types.adapters.js` | 通道適配器型別——讓插件能定義自己的通道適配器 |
| `channels/plugins/types.config.js` | 通道設定型別——插件宣告自己所需設定項目的型別 |
| `channels/plugins/types.plugin.js` | 插件基礎型別——所有插件共通的基礎介面定義 |

**觀察**：根入口完全聚焦在「型別」上，不匯出任何執行時函式。這是一個純型別介面——它定義了「插件長什麼樣子」，而非「插件能做什麼」。實際的功能 helper 全部在 subpath 上。

---

## 6. 零依賴 Workspace Packages

OpenClaw 的 monorepo 中有三個獨立的 workspace packages，它們有一個共同的顯著特徵：**零外部依賴（0 dependencies）**。

### 6.1 @openclaw/plugin-sdk

| 屬性 | 值 |
|------|---|
| **Package 名稱** | `@openclaw/plugin-sdk` |
| **位置** | `packages/plugin-sdk/` |
| **Dependencies** | 0 |
| **Subpath Exports** | ~40 個 |
| **角色** | 定義插件與核心之間的介面合約 |

**零依賴的意義**：

作為「邊界介面」，`plugin-sdk` 絕對不能依賴任何外部套件。原因有三：

1. **穩定性**：外部依賴的版本升級可能改變行為，但邊界介面必須穩定。
2. **可移植性**：插件可能在各種環境中運行，零依賴確保不會遇到套件安裝問題。
3. **介面純粹性**：SDK 應該只包含型別定義和最小的 helper 函式，不需要外部功能。

**~40 個 subpath exports** 全部映射到 `src/*.ts` 檔案，意味著 SDK 的實作完全是自包含的 TypeScript 程式碼。

### 6.2 @openclaw/memory-host-sdk

| 屬性 | 值 |
|------|---|
| **Package 名稱** | `@openclaw/memory-host-sdk` |
| **位置** | `packages/memory-host-sdk/` |
| **Dependencies** | 0 |
| **Subpath Exports** | 14 個 |
| **角色** | 定義記憶系統宿主的介面合約 |

**Memory Host SDK 的定位**：

OpenClaw 的記憶系統（Memory System）允許第三方實作自定義的記憶後端（memory backend）。`memory-host-sdk` 定義了記憶宿主（Memory Host）需要實作的介面——例如如何儲存記憶、如何查詢記憶、如何管理記憶的生命週期等。

14 個 subpath exports 覆蓋了記憶系統的各個面向，但具體的實作（例如使用 LanceDB 還是 sqlite-vec）則由核心程式碼或插件來完成。

**與 plugin-sdk 的關係**：

```
plugin-sdk        ← 插件的通用介面
memory-host-sdk   ← 記憶系統的專用介面
```

兩者是平行的 SDK——一個面向所有插件開發者，一個面向想要自定義記憶後端的開發者。

### 6.3 @openclaw/plugin-package-contract

| 屬性 | 值 |
|------|---|
| **Package 名稱** | `@openclaw/plugin-package-contract` |
| **位置** | `packages/plugin-package-contract/` |
| **Dependencies** | 0 |
| **Subpath Exports** | 1 個（single export） |
| **角色** | 定義插件 package 的結構合約 |

**Plugin Package Contract 的特殊性**：

這個 package 只有**一個匯出**——它定義的是「一個合法的 OpenClaw 插件 npm package 應該長什麼樣子」的合約（contract）。包括：

- 插件 package 的 `package.json` 必須包含哪些欄位
- 插件的入口檔案應該匯出什麼
- 插件的 manifest 格式要求

這是一個**元層級（Meta-level）的合約**——不是定義「插件能做什麼」，而是定義「什麼東西算是一個合法的插件」。

### 6.4 三個 SDK 的協作模型

```
┌─────────────────────────────────────────────────────────┐
│                    Plugin Developer                       │
│                                                           │
│  "我要開發一個 OpenClaw 插件"                              │
│                                                           │
│  1. 先看 plugin-package-contract → 知道 package 要怎麼打包 │
│  2. 再用 plugin-sdk              → 知道能呼叫哪些核心功能  │
│  3. 如需自定義記憶 → memory-host-sdk → 知道記憶介面怎麼實作 │
└─────────────────────────────────────────────────────────┘
```

三者形成一個完整的「插件開發者工具包」：

| SDK | 回答的問題 |
|-----|-----------|
| `plugin-package-contract` | 「我的插件 package 應該是什麼結構？」 |
| `plugin-sdk` | 「我的插件能呼叫 OpenClaw 的哪些功能？」 |
| `memory-host-sdk` | 「如果我想自定義記憶後端，需要實作什麼介面？」 |

---

## 7. 邊界的全景圖

把本章的所有元素組合起來，形成 OpenClaw 擴充/核心邊界的完整圖景：

```
┌───────────────────── OpenClaw Monorepo ─────────────────────┐
│                                                              │
│  src/ (Core)                                                 │
│  ├── agents/          ┐                                      │
│  ├── config/          │                                      │
│  ├── infra/           │ ← 核心模組（60+ 個）                  │
│  ├── channels/        │   插件不可直接引用                     │
│  ├── ...              ┘                                      │
│  │                                                           │
│  ├── plugin-sdk/      ← 唯一合法的插件引用目標                │
│  │   ├── index.ts         (根入口：核心型別)                   │
│  │   ├── config.ts        (subpath: 設定存取)                 │
│  │   ├── channels.ts      (subpath: 通道操作)                 │
│  │   ├── account-id.ts    (subpath: 帳號識別)                 │
│  │   └── ...              (~40 個 subpath)                    │
│  │                                                           │
│  ├── plugin-sdk-internal/ ← SDK 內部實作，插件不可引用        │
│  │                                                           │
│  └── extensionAPI.ts  ← 已棄用的舊入口（仍可用，發出警告）    │
│                                                              │
│  packages/ (Workspace Packages)                              │
│  ├── plugin-sdk/           (0 deps, ~40 exports)             │
│  ├── memory-host-sdk/      (0 deps, 14 exports)              │
│  └── plugin-package-contract/ (0 deps, 1 export)             │
│                                                              │
│  extensions/ (User/Third-party Plugins)                      │
│  ├── @openclaw/plugin-a/                                     │
│  ├── @openclaw/plugin-b/                                     │
│  └── ...                                                     │
│                                                              │
│  scripts/                                                    │
│  └── check-extension-plugin-sdk-boundary.mjs                 │
│      ├── Rule 1: src-outside-plugin-sdk                      │
│      ├── Rule 2: plugin-sdk-internal                         │
│      └── Rule 3: relative-outside-package                    │
│                                                              │
│  tsconfig.json                                               │
│  └── paths: openclaw/plugin-sdk → src/plugin-sdk/            │
│             @openclaw/plugin-sdk → src/plugin-sdk/            │
│             @openclaw/* → extensions/*                        │
│                                                              │
│  tsdown.config.ts                                            │
│  └── neverBundle: [lancedb, matrix-crypto, matrix-js-sdk,    │
│                    ...bundledPluginRuntimeDependencies]       │
└──────────────────────────────────────────────────────────────┘
```

### 邊界的多層防禦總結

| 層次 | 機制 | 作用 |
|------|------|------|
| **靜態分析** | `check-extension-plugin-sdk-boundary.mjs` 三條規則 | 在建置時掃描 import 語句，違反即失敗 |
| **路徑映射** | `tsconfig.json` path aliases | 控制 TypeScript 如何解析 import 路徑 |
| **API 設計** | `plugin-sdk/index.ts` 極簡根入口 + subpath exports | 只暴露必要的介面 |
| **Package 隔離** | 零依賴 workspace packages | 確保 SDK 不引入外部依賴，保持介面穩定 |
| **棄用遷移** | `extensionAPI.ts` deprecation warning | 引導舊插件遷移到新的 plugin-sdk |

---

## 8. 關鍵觀察與結論

### 8.1 邊界是被自動化強制執行的

不同於許多專案靠文件和 code review 維護架構邊界，OpenClaw 用腳本（`check-extension-plugin-sdk-boundary.mjs`）在 CI/CD 中自動檢查。這意味著即使有人嘗試在 PR 中引入違規的 import，CI 會自動攔截。這是一種「架構即程式碼」（Architecture as Code）的實踐。

### 8.2 零依賴 SDK 是刻意的設計決策

三個 workspace packages 全部零依賴——這不是巧合，而是刻意的設計。介面合約（contract）不應該有外部依賴，因為依賴會引入不穩定性。當你的插件只依賴零依賴的 SDK 時，SDK 永遠不會因為某個上游套件的破壞性更新而壞掉。

### 8.3 從 extensionAPI 到 plugin-sdk 的演進揭示了設計思路

早期的 `extensionAPI.ts` 是一個「把所有東西都暴露出來」的做法——簡單粗暴但缺乏控制。遷移到 `plugin-sdk` 後，OpenClaw 採取了「最小介面」原則——只暴露插件真正需要的功能，其他一概隱藏。這個演進過程是成熟軟體專案的典型特徵。

### 8.4 對我們開發插件的實際影響

理解了邊界機制，我們在開發 Copilot CLI 插件或 Discord 語音機器人插件時需要遵守以下原則：

1. **只從 `openclaw/plugin-sdk/*` 或 `@openclaw/plugin-sdk/*` import**——其他路徑都會被邊界檢查攔截。
2. **查閱 ~40 個 subpath exports 的清單**——這就是我們能用的全部 API。如果需要的功能不在其中，可能需要向上游提交 feature request。
3. **如果需要自定義記憶後端**，使用 `@openclaw/memory-host-sdk` 的 14 個 subpath exports。
4. **Package 結構遵循 `@openclaw/plugin-package-contract`**——確保我們的插件能被 OpenClaw 正確識別和載入。

### 8.5 neverBundle 與邊界的交叉影響

`tsdown.config.ts` 的 `neverBundle` 清單包含 `...bundledPluginRuntimeDependencies`——這表明插件的執行時依賴在打包時有特殊處理。插件的依賴不會被打包進核心的 bundle，而是保持為外部引用。這進一步強化了核心與擴充的分離——即使在 bundle 層面，兩者也是獨立的。

---

## 引用來源

| 檔案路徑 | 行號範圍 | 本章引用內容 |
|----------|---------|-------------|
| `source-repo/scripts/check-extension-plugin-sdk-boundary.mjs` | 1-55 | 三條邊界規則的定義與實作 |
| `source-repo/src/extensionAPI.ts` | 1-30 | 棄用的擴充 API、deprecation warning、re-export 來源 |
| `source-repo/tsconfig.json` | 22-28 | TypeScript path aliases（7 條映射規則） |
| `source-repo/src/plugin-sdk/index.ts` | 1-30 | 根入口的設計哲學、匯出的四個型別來源 |
| `source-repo/tsdown.config.ts` | 125-130 | `neverBundle` 清單中的 `bundledPluginRuntimeDependencies` |
| `packages/plugin-sdk/` | package.json | 0 dependencies、~40 subpath exports |
| `packages/memory-host-sdk/` | package.json | 0 dependencies、14 subpath exports |
| `packages/plugin-package-contract/` | package.json | 0 dependencies、single export |
