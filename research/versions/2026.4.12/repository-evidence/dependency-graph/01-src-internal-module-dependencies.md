# 01 — src/ 內部模組依賴關係

> **本章摘要**：OpenClaw 的 `src/` 目錄包含超過 60 個子模組（subdirectory），從 AI agent 核心邏輯到 CLI 介面、從通道整合到媒體處理，幾乎所有功能都在這個單一目錄下共存。本章透過追蹤關鍵入口檔案的 `import` 語句，重建模組之間的靜態依賴拓撲，識別出哪些模組是「基礎設施骨幹」（被大量依賴）、哪些是「功能葉節點」（僅被少量引用），以及它們如何透過分層結構避免循環依賴。所有分析均基於原始碼中實際出現的 import 語句。

---

## 目錄

- [1. src/ 子目錄完整清單](#1-src-子目錄完整清單)
- [2. 入口層依賴分析](#2-入口層依賴分析)
  - [2.1 entry.ts——程式的第一行](#21-entryts程式的第一行)
  - [2.2 index.ts——雙模式入口](#22-indexts雙模式入口)
  - [2.3 runtime.ts——執行環境初始化](#23-runtimets執行環境初始化)
- [3. CLI 層依賴分析](#3-cli-層依賴分析)
  - [3.1 run-main.ts——CLI 的核心驅動器](#31-run-maints-cli-的核心驅動器)
- [4. Library 外觀層依賴分析](#4-library-外觀層依賴分析)
  - [4.1 library.ts——程式化 API 的入口](#41-libraryts程式化-api-的入口)
- [5. 模組分層模型](#5-模組分層模型)
- [6. 關鍵觀察與結論](#6-關鍵觀察與結論)
- [引用來源](#引用來源)

---

## 1. src/ 子目錄完整清單

在深入分析依賴關係之前，先列出 `src/` 下所有已知的子目錄。這 60+ 個目錄構成了 OpenClaw 的完整功能版圖：

| 分類 | 模組名稱 | 功能簡述 |
|------|----------|----------|
| **核心基礎** | `infra` | 基礎設施工具：環境變數、錯誤處理、網路、執行環境守衛 |
| **核心基礎** | `types` | 全域型別定義 |
| **核心基礎** | `shared` | 跨模組共享的工具函式 |
| **核心基礎** | `utils` | 通用工具函式（如電話號碼正規化） |
| **核心基礎** | `config` | 設定檔載入、session 路徑解析、session store |
| **核心基礎** | `logging` | 日誌系統 |
| **入口與啟動** | `cli` | 命令列介面：引數解析、profile 管理、容器目標解析 |
| **入口與啟動** | `bootstrap` | 應用程式啟動引導流程 |
| **入口與啟動** | `daemon` | 守護程序模式 |
| **入口與啟動** | `process` | 子程序管理、child-process bridge |
| **入口與啟動** | `terminal` | 終端機狀態管理（進度列、狀態還原） |
| **入口與啟動** | `wizard` | 初始設定引導精靈 |
| **AI 核心** | `agents` | Agent 定義與管理 |
| **AI 核心** | `chat` | 對話管理 |
| **AI 核心** | `context-engine` | 上下文引擎，決定送入 LLM 的資訊 |
| **AI 核心** | `sessions` | 對話 session 管理 |
| **AI 核心** | `memory-host-sdk` | 記憶系統的主機端 SDK |
| **AI 核心** | `flows` | 工作流程定義與執行 |
| **AI 核心** | `tasks` | 任務排程與管理 |
| **AI 核心** | `routing` | 訊息路由決策 |
| **AI 核心** | `auto-reply` | 自動回覆邏輯與模板 |
| **協定整合** | `mcp` | Model Context Protocol 整合 |
| **協定整合** | `acp` | Agent Client Protocol 整合 |
| **通道整合** | `channels` | 通道抽象層 |
| **通道整合** | `gateway` | 閘道器，統一接收外部訊息 |
| **通道整合** | `web` | Web 通道（HTTP API） |
| **通道整合** | `interactive` | 互動式介面 |
| **通道整合** | `pairing` | 裝置配對（WhatsApp 等） |
| **媒體處理** | `media` | 媒體檔案基礎處理 |
| **媒體處理** | `media-generation` | 媒體生成（圖片、影片等） |
| **媒體處理** | `media-understanding` | 媒體理解（圖片識別等） |
| **媒體處理** | `image-generation` | 圖片生成 |
| **媒體處理** | `video-generation` | 影片生成 |
| **媒體處理** | `music-generation` | 音樂生成 |
| **語音** | `tts` | 文字轉語音（Text-to-Speech） |
| **語音** | `realtime-voice` | 即時語音處理 |
| **語音** | `realtime-transcription` | 即時語音轉文字 |
| **擴充系統** | `plugin-sdk` | 插件 SDK，定義擴充的公開介面 |
| **擴充系統** | `plugins` | 插件載入與執行時環境 |
| **擴充系統** | `hooks` | 插件掛鉤（Hook）系統 |
| **擴充系統** | `bindings` | 原生綁定（Native bindings） |
| **擴充系統** | `node-host` | Node.js 插件宿主環境 |
| **網路** | `web-fetch` | HTTP 請求工具 |
| **網路** | `web-search` | 網頁搜尋功能 |
| **網路** | `link-understanding` | 連結內容理解 |
| **網路** | `proxy-capture` | 代理攔截與除錯 |
| **使用者介面** | `tui` | Terminal UI（終端圖形介面） |
| **使用者介面** | `markdown` | Markdown 渲染 |
| **使用者介面** | `status` | 系統狀態顯示 |
| **使用者介面** | `docs` | 文件生成 |
| **其他** | `commands` | 命令註冊系統 |
| **其他** | `compat` | 向後相容層 |
| **其他** | `cron` | 定時任務排程 |
| **其他** | `i18n` | 國際化 |
| **其他** | `scripts` | 內部腳本 |
| **其他** | `secrets` | 密鑰管理 |
| **其他** | `security` | 安全策略 |
| **其他** | `canvas-host` | Canvas 渲染宿主 |
| **測試** | `test-helpers` | 測試輔助工具 |
| **測試** | `test-utils` | 測試工具函式 |

> 以上分類為作者根據模組名稱與已知程式碼推斷的功能歸屬，部分分類可能隨後續深入分析而調整。

---

## 2. 入口層依賴分析

OpenClaw 的入口層由三個關鍵檔案組成：`entry.ts`、`index.ts`、`runtime.ts`。它們各自承擔不同的啟動職責，但共同特徵是：**只依賴少數幾個「基礎設施模組」**，形成一個薄薄的啟動層。

### 2.1 entry.ts——程式的第一行

`entry.ts` 是整個 OpenClaw 應用程式的真正起點。當使用者在終端機輸入 `openclaw` 命令時，Node.js 最先載入的就是這個檔案。

```typescript
// source-repo/src/entry.ts:1-16
import { spawn } from "node:child_process";
import { enableCompileCache } from "node:module";
import process from "node:process";
import { fileURLToPath } from "node:url";
import { isRootHelpInvocation, isRootVersionInvocation } from "./cli/argv.js";
import { parseCliContainerArgs, resolveCliContainerTarget } from "./cli/container-target.js";
import { applyCliProfileEnv, parseCliProfileArgs } from "./cli/profile.js";
import { normalizeWindowsArgv } from "./cli/windows-argv.js";
import { buildCliRespawnPlan } from "./entry.respawn.js";
import { isTruthyEnvValue, normalizeEnv } from "./infra/env.js";
import { isMainModule } from "./infra/is-main.js";
import { ensureOpenClawExecMarkerOnProcess } from "./infra/openclaw-exec-env.js";
import { installProcessWarningFilter } from "./infra/warning-filter.js";
import { attachChildProcessBridge } from "./process/child-process-bridge.js";
```

**依賴拓撲圖**：

```
entry.ts
├── [Node.js built-in] child_process, module, process, url
├── cli/
│   ├── argv.js          ← CLI 引數快速判斷（help / version）
│   ├── container-target.js  ← Docker 容器目標解析
│   ├── profile.js       ← CLI profile 環境變數套用
│   └── windows-argv.js  ← Windows 平台引數正規化
├── infra/
│   ├── env.js           ← 環境變數正規化與判斷
│   ├── is-main.js       ← 判斷是否為主模組
│   ├── openclaw-exec-env.js ← 執行環境標記
│   └── warning-filter.js   ← Node.js 警告過濾
├── process/
│   └── child-process-bridge.js ← 子程序橋接
└── entry.respawn.js     ← 程序重啟邏輯（同層級檔案）
```

**關鍵觀察**：

1. **`entry.ts` 只觸及三個內部模組**：`cli/`、`infra/`、`process/`。這表明入口點的職責被嚴格限制在「環境準備」層面。
2. **`infra/` 模組被引用 4 次**，是 entry 層最重要的依賴。`infra/` 可以說是整個 OpenClaw 的「地基」。
3. **Node.js 內建模組佔 4 項**（`child_process`、`module`、`process`、`url`），顯示 entry 層需要直接操作底層 Node.js API。
4. **`enableCompileCache`** 的引入暗示 OpenClaw 使用了 Node.js 的模組編譯快取功能來加速啟動。
5. **無任何第三方依賴**——`entry.ts` 完全依賴 Node.js 內建模組與自身程式碼，確保啟動的第一階段不受外部套件版本的影響。

### 2.2 index.ts——雙模式入口

`index.ts` 實現了一個巧妙的雙模式設計：同一個檔案既可以作為「被其他程式引用的 library」，也可以作為「直接執行的 CLI 工具」。

```typescript
// source-repo/src/index.ts:1-7
import process from "node:process";
import { fileURLToPath } from "node:url";
import { formatUncaughtError } from "./infra/errors.js";
import { isMainModule } from "./infra/is-main.js";
import { installUnhandledRejectionHandler } from "./infra/unhandled-rejections.js";
```

**雙模式的依賴分支**：

```
index.ts
├── [always] infra/errors.js
├── [always] infra/is-main.js
├── [always] infra/unhandled-rejections.js
│
├── [when NOT main module]  →  library.js (lazy import)
│   └── 公開程式化 API 給外部消費者
│
└── [when IS main module]
    ├── terminal/restore.js  ← 終端機狀態還原
    └── cli/run-main.js      ← CLI 主流程（透過 runLegacyCliEntry）
```

**關鍵觀察**：

1. **條件式 import（Conditional import）**：`index.ts` 根據 `isMainModule()` 的結果走不同的 import 路徑。這是一種優化策略——如果只是被其他程式當 library 引用，就不需要載入整個 CLI 框架。
2. **「Legacy」一詞的出現**：CLI 入口函式名為 `runLegacyCliEntry`，暗示可能存在新舊兩套 CLI 啟動方式。結合 `entry.ts` 的存在，推測 `entry.ts` 是較新的入口點，而 `index.ts` 的 CLI 模式是向後相容的舊路徑。
3. **`infra/` 再次出現**：三個 always-import 全部來自 `infra/`，再次確認其「基礎設施骨幹」的地位。

### 2.3 runtime.ts——執行環境初始化

`runtime.ts` 是最精簡的入口層檔案，僅兩行 import：

```typescript
// source-repo/src/runtime.ts:1-2
import { clearActiveProgressLine } from "./terminal/progress-line.js";
import { restoreTerminalState } from "./terminal/restore.js";
```

**依賴拓撲圖**：

```
runtime.ts
└── terminal/
    ├── progress-line.js  ← 清除終端機進度列
    └── restore.js        ← 還原終端機狀態
```

**關鍵觀察**：

1. **只依賴 `terminal/` 模組**——`runtime.ts` 的唯一職責是處理終端機的清理工作。
2. 這符合「執行環境」的語義：確保程式結束或中斷時，終端機能回復到正常狀態（例如隱藏的游標要恢復顯示、raw mode 要關閉等）。

---

## 3. CLI 層依賴分析

### 3.1 run-main.ts——CLI 的核心驅動器

`run-main.ts` 是 CLI 模式的真正心臟。相較於入口層的精簡，這裡的依賴範圍明顯擴大：

```typescript
// source-repo/src/cli/run-main.ts:1-40
import { existsSync } from "node:fs";
import path from "node:path";
import process from "node:process";
import { fileURLToPath } from "node:url";
import { CommanderError } from "commander";
import { resolveStateDir } from "../config/paths.js";
import { normalizeEnv } from "../infra/env.js";
import { formatUncaughtError } from "../infra/errors.js";
import { isMainModule } from "../infra/is-main.js";
import { ensureGlobalUndiciEnvProxyDispatcher } from "../infra/net/undici-global-dispatcher.js";
import { ensureOpenClawCliOnPath } from "../infra/path-env.js";
import { assertSupportedRuntime } from "../infra/runtime-guard.js";
import { enableConsoleCapture } from "../logging.js";
import { resolveManifestCommandAliasOwner } from "../plugins/manifest-command-aliases.runtime.js";
import { hasMemoryRuntime } from "../plugins/memory-state.js";
import { maybeWarnAboutDebugProxyCoverage } from "../proxy-capture/coverage.js";
import { finalizeDebugProxyCapture, initializeDebugProxyCapture } from "../proxy-capture/runtime.js";
```

**依賴拓撲圖**：

```
cli/run-main.ts
├── [Node.js built-in] fs, path, process, url
├── [external] commander (CommanderError)
├── config/
│   └── paths.js         ← 狀態目錄解析
├── infra/
│   ├── env.js           ← 環境變數正規化
│   ├── errors.js        ← 錯誤格式化
│   ├── is-main.js       ← 主模組判斷
│   ├── net/undici-global-dispatcher.js ← HTTP 代理設定
│   ├── path-env.js      ← PATH 環境變數管理
│   └── runtime-guard.js ← Node.js 版本檢查
├── logging.js           ← 日誌系統（頂層模組）
├── plugins/
│   ├── manifest-command-aliases.runtime.js ← 插件命令別名解析
│   └── memory-state.js  ← 記憶系統狀態檢查
└── proxy-capture/
    ├── coverage.js      ← 除錯代理覆蓋率警告
    └── runtime.js       ← 除錯代理捕獲初始化
```

**關鍵觀察**：

1. **首次出現外部依賴**：`commander` 是 CLI 層引入的第一個（也幾乎是唯一一個）外部套件，用於命令列引數解析。這與 entry 層「零外部依賴」形成鮮明對比。
2. **`infra/` 被引用 6 次**，遠超其他模組。`run-main.ts` 需要環境變數處理、錯誤格式化、模組判斷、網路代理、PATH 管理、版本守衛——這些全部由 `infra/` 提供。
3. **`plugins/` 模組首次出現**——CLI 層已經開始接觸插件系統，這表明插件的命令別名功能（manifest command aliases）在 CLI 引數解析階段就需要介入。
4. **`proxy-capture/` 模組出現兩次**——除錯代理的初始化和清理是 CLI 生命週期的一部分，表明 OpenClaw 內建了 HTTP 請求攔截與除錯功能。
5. **依賴範圍從 3 個模組擴展到 6 個**（config、infra、logging、plugins、proxy-capture，加上頂層 logging.js），反映了 CLI 層作為「業務邏輯與基礎設施的交匯點」的角色。

---

## 4. Library 外觀層依賴分析

### 4.1 library.ts——程式化 API 的入口

`library.ts` 是 OpenClaw 作為 Node.js library 被外部程式引用時的 API 外觀（Facade）。它的依賴模式與 CLI 層截然不同——大量使用**延遲載入（Lazy loading）**。

```typescript
// source-repo/src/library.ts:1-22 (靜態 import)
import { applyTemplate } from "./auto-reply/templating.js";
import { createDefaultDeps } from "./cli/deps.js";
import { waitForever } from "./cli/wait.js";
import { loadConfig } from "./config/config.js";
import { resolveStorePath } from "./config/sessions/paths.js";
import { deriveSessionKey, resolveSessionKey } from "./config/sessions/session-key.js";
import { loadSessionStore, saveSessionStore } from "./config/sessions/store.js";
import { describePortOwner, ensurePortAvailable, handlePortError, PortInUseError } from "./infra/ports.js";
import { normalizeE164 } from "./utils.js";
```

**延遲載入的模組**（在函式內透過 dynamic `import()` 載入）：

```
library.ts 延遲載入:
├── auto-reply/reply.runtime.js    ← 自動回覆執行時邏輯
├── cli/prompt.js                  ← 互動式提示
├── infra/binaries.js              ← 外部二進位檔管理
├── process/exec.js                ← 程序執行工具
└── plugins/runtime/runtime-web-channel-plugin.js ← Web 通道插件
```

**完整依賴拓撲圖**：

```
library.ts
├── [static imports]
│   ├── auto-reply/templating.js   ← 模板引擎
│   ├── cli/deps.js                ← 預設依賴建構
│   ├── cli/wait.js                ← 程序等待工具
│   ├── config/
│   │   ├── config.js              ← 設定檔載入
│   │   └── sessions/
│   │       ├── paths.js           ← Store 路徑解析
│   │       ├── session-key.js     ← Session 金鑰衍生
│   │       └── store.js           ← Session 儲存
│   ├── infra/ports.js             ← 連接埠管理
│   └── utils.js                   ← 通用工具（E.164 正規化）
│
└── [lazy imports]
    ├── auto-reply/reply.runtime.js
    ├── cli/prompt.js
    ├── infra/binaries.js
    ├── process/exec.js
    └── plugins/runtime/runtime-web-channel-plugin.js
```

**關鍵觀察**：

1. **延遲載入策略**：5 個模組被延遲載入，意味著它們只在「實際需要時」才被引入。這是一種效能優化——如果 library 消費者只需要讀取設定檔，就不需要載入自動回覆引擎或 Web 通道插件。
2. **`config/` 模組的依賴密度最高**：`sessions/paths.js`、`sessions/session-key.js`、`sessions/store.js` 加上 `config.js`，共 4 個 import 來自 config/。Library 的核心職責之一顯然是設定管理。
3. **`auto-reply/` 出現在靜態與延遲兩邊**：`templating.js`（模板引擎）是靜態載入的，而 `reply.runtime.js`（實際回覆邏輯）是延遲載入的。這暗示模板的編譯（輕量）和實際的回覆執行（重量）被刻意分離。
4. **`utils.js` 是頂層模組**——不在任何子目錄下，直接位於 `src/utils.js`。它提供的 `normalizeE164` 函式（E.164 是國際電話號碼格式）暗示通訊通道功能對 library 層是基本需求。

---

## 5. 模組分層模型

綜合以上分析，可以歸納出 OpenClaw `src/` 目錄的模組分層模型：

```
┌─────────────────────────────────────────────────────────┐
│  Layer 0: Entry Points（入口層）                          │
│  entry.ts, index.ts, runtime.ts                         │
│  特徵：零外部依賴，僅觸及 infra/ 與少數啟動模組           │
├─────────────────────────────────────────────────────────┤
│  Layer 1: Infrastructure（基礎設施層）                    │
│  infra/, types/, shared/, utils.js, logging.js          │
│  特徵：被所有其他層依賴，提供最基本的工具函式               │
├─────────────────────────────────────────────────────────┤
│  Layer 2: Configuration（設定層）                         │
│  config/, secrets/                                      │
│  特徵：依賴 Layer 1，提供設定檔與密鑰管理                  │
├─────────────────────────────────────────────────────────┤
│  Layer 3: Core Business（核心業務層）                     │
│  agents/, chat/, context-engine/, sessions/, routing/,  │
│  memory-host-sdk/, flows/, tasks/, auto-reply/          │
│  特徵：依賴 Layer 1-2，實現 AI Agent 的核心邏輯           │
├─────────────────────────────────────────────────────────┤
│  Layer 4: Integration（整合層）                           │
│  channels/, gateway/, web/, mcp/, acp/, pairing/,       │
│  media/, tts/, realtime-voice/, web-fetch/, web-search/ │
│  特徵：依賴 Layer 1-3，連接外部世界（通道、協定、媒體）    │
├─────────────────────────────────────────────────────────┤
│  Layer 5: Extension（擴充層）                             │
│  plugin-sdk/, plugins/, hooks/, bindings/, node-host/   │
│  特徵：定義與執行第三方擴充，plugin-sdk 是對外合約        │
├─────────────────────────────────────────────────────────┤
│  Layer 6: Presentation（呈現層）                          │
│  cli/, tui/, terminal/, wizard/, interactive/,          │
│  commands/, markdown/, status/, docs/                    │
│  特徵：面向使用者的介面層，依賴幾乎所有底層模組            │
└─────────────────────────────────────────────────────────┘
```

### 依賴方向原則

```
Layer 0 → Layer 1 → Layer 2 → Layer 3 → Layer 4
                                    ↓
                               Layer 5 (Extension)
                                    ↓
                               Layer 6 (Presentation)
```

**核心規則**：高層可以依賴低層，低層不應依賴高層。`infra/` 作為 Layer 1 被所有層引用但不引用任何業務模組——這正是它作為「基礎設施骨幹」的體現。

### 被依賴次數統計（基於入口檔案分析）

下表統計 `entry.ts`、`index.ts`、`runtime.ts`、`library.ts`、`cli/run-main.ts` 五個檔案中，各模組被 import 的次數：

| 模組 | 被引用次數 | 引用來源 |
|------|-----------|---------|
| `infra/` | **13** | entry(4), index(3), run-main(6) |
| `cli/` | **6** | entry(4), library(2) |
| `config/` | **5** | library(4), run-main(1) |
| `terminal/` | **3** | runtime(2), index(1) |
| `process/` | **2** | entry(1), library(1, lazy) |
| `plugins/` | **2** | run-main(2) |
| `auto-reply/` | **2** | library(1 static, 1 lazy) |
| `proxy-capture/` | **2** | run-main(2) |
| `logging.js` | **1** | run-main(1) |
| `utils.js` | **1** | library(1) |

> `infra/` 以 13 次引用遙遙領先，是當之無愧的「被依賴之王」。

---

## 6. 關鍵觀察與結論

### 6.1 infra/ 是全系統的地基

從入口層到 CLI 層再到 Library 層，`infra/` 無處不在。它至少提供以下子模組：

- `env.js` — 環境變數正規化與真值判斷
- `errors.js` — 未捕獲錯誤格式化
- `is-main.js` — 主模組判斷（ESM 環境下的 `require.main` 替代）
- `openclaw-exec-env.js` — 執行環境標記
- `warning-filter.js` — Node.js 警告過濾
- `net/undici-global-dispatcher.js` — 全域 HTTP 代理設定
- `path-env.js` — PATH 環境變數管理
- `runtime-guard.js` — Node.js 版本相容性守衛
- `ports.js` — 連接埠可用性檢查
- `binaries.js` — 外部二進位檔管理（延遲載入）
- `unhandled-rejections.js` — 未處理 Promise rejection 的攔截

### 6.2 延遲載入是重要的架構策略

`library.ts` 使用 dynamic `import()` 來延遲載入 5 個模組，`index.ts` 則根據 `isMainModule()` 條件載入不同模組。這不是偶然——OpenClaw 顯然有意識地控制啟動時的模組載入量，避免在不需要的情境下載入重型模組（如自動回覆引擎、Web 通道插件等）。

### 6.3 CLI 與 Library 是兩條平行路徑

```
entry.ts → index.ts ──┬── [as CLI]     → cli/run-main.ts → 完整功能
                      └── [as library] → library.ts → 精選 API
```

這兩條路徑依賴完全不同的模組集合，顯示 OpenClaw 的架構設計明確區分了「作為應用程式執行」和「作為函式庫被引用」兩種使用情境。

### 6.4 插件系統在 CLI 階段就開始介入

`run-main.ts` 引入 `plugins/manifest-command-aliases.runtime.js` 和 `plugins/memory-state.js`，說明插件不只是「被動載入的擴充」——它們的命令別名需要在 CLI 引數解析階段就生效，記憶系統的狀態也需要在啟動時就檢查。這暗示 OpenClaw 的插件系統與核心的耦合程度比一般的「插件架構」更深。

### 6.5 60+ 模組但入口層只觸及少數

60+ 個 `src/` 子目錄中，入口檔案直接引用的只有約 10 個。大量的功能模組（agents、channels、media、mcp 等）完全不出現在入口層，表明它們是在後續的啟動流程中按需載入的。這是一種健康的架構——入口層保持精簡，功能模組的引入被延遲到實際需要的時刻。

---

## 引用來源

| 檔案路徑 | 行號範圍 | 本章引用內容 |
|----------|---------|-------------|
| `source-repo/src/entry.ts` | 1-16 | entry.ts 完整 import 區塊，入口層依賴分析 |
| `source-repo/src/index.ts` | 1-7 | index.ts 靜態 import，雙模式入口設計 |
| `source-repo/src/index.ts` | （條件分支） | library.js 延遲載入、terminal/restore.js、cli/run-main.js 引用 |
| `source-repo/src/runtime.ts` | 1-2 | runtime.ts 完整 import，終端機清理職責 |
| `source-repo/src/cli/run-main.ts` | 1-40 | CLI 主驅動器完整 import 區塊 |
| `source-repo/src/library.ts` | 1-22 | Library 外觀層靜態 import |
| `source-repo/src/library.ts` | （函式內部） | 延遲載入的 5 個模組路徑 |
