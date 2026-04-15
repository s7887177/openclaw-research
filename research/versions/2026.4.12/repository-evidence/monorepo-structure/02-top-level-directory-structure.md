# 02 — 頂層目錄結構與職責

> **證據層級**：Repository Evidence
> **對應版本**：OpenClaw 2026.4.12
> **主要來源**：`source-repo/` 根目錄下的目錄列表與各目錄內容觀察

---

## 本章摘要

本章記錄 OpenClaw 倉庫根目錄下所有頂層目錄的結構、內容物與職責劃分。OpenClaw 的原始碼倉庫是一個「巨型 monorepo」——所有東西都在同一個 Git 倉庫中：核心引擎、111 個通道/供應商擴充套件、53 個技能包、4 個原生行動應用、1 個 Web UI、文件網站、測試套件、CI/CD 配置。本章逐一列舉每個頂層目錄，說明它包含什麼、對應的 workspace 角色是什麼、以及它在整體架構中的位置。這是一張「地圖」——讓讀者在探索 OpenClaw 原始碼時，永遠知道自己身在何處。

---

## 1. 目錄總覽表

下表列出倉庫根目錄下的所有頂層目錄，依重要性排序：

| 目錄 | Workspace 角色 | 概要描述 | 子項目數量 |
|------|---------------|----------|-----------|
| `src/` | 根套件的一部分 | 核心引擎原始碼 | 60+ 子目錄 |
| `extensions/` | `extensions/*`（workspace 成員） | 擴充套件：通道適配器、供應商適配器等 | 111 個 |
| `skills/` | 非 workspace 成員 | 技能包：為 AI 提供各種工具與能力 | 53 個 |
| `packages/` | `packages/*`（workspace 成員） | 共享內部套件：SDK 與合約定義 | 3 個 |
| `apps/` | 非 workspace 成員 | 原生應用程式（行動端與桌面端） | 4 個 |
| `ui/` | `ui`（workspace 成員） | Web UI 前端介面 | 1 個 |
| `docs/` | 非 workspace 成員 | 文件網站原始碼 | 25+ 主題目錄 |
| `scripts/` | 非 workspace 成員 | 建置、維護、輔助腳本 | — |
| `test/` | 非 workspace 成員 | 整合測試與端對端測試 | — |
| `test-fixtures/` | 非 workspace 成員 | 測試用的固定資料（fixture data） | — |
| `vendor/` | 非 workspace 成員 | 被 vendor 進倉庫的第三方程式碼 | 1 個（a2ui） |
| `patches/` | 非 workspace 成員 | pnpm patch 補丁檔案 | — |
| `.github/` | 非 workspace 成員 | GitHub Actions 工作流程、Issue 模板等 | — |
| `git-hooks/` | 非 workspace 成員 | Git hooks 腳本 | — |
| `qa/` | 非 workspace 成員 | QA 測試場景 | — |
| `assets/` | 非 workspace 成員 | 靜態資源檔案 | — |
| `Swabble/` | 非 workspace 成員 | 相關的 Swabble 專案檔案 | — |

---

## 2. 核心原始碼——`src/` 目錄

### 2.1 概述

`src/` 是 OpenClaw 的心臟。它包含了核心引擎的所有原始碼，被編譯後輸出到根套件的 `dist/` 目錄。`src/` 不是一個獨立的 workspace 套件——它是根套件 `openclaw` 的組成部分。

### 2.2 子目錄完整列表

以下是 `src/` 下所有已觀察到的子目錄，按功能分類：

#### 核心基礎設施（Core Infrastructure）

| 子目錄 | 推測職責 |
|--------|---------|
| `bootstrap` | 應用程式啟動引導流程 |
| `config` | 配置管理系統 |
| `context-engine` | 上下文引擎——管理 AI 對話的上下文窗口 |
| `daemon` | 背景常駐程序（daemon）模式 |
| `gateway` | 閘道器——核心的訊息路由層 |
| `hooks` | 生命週期鉤子（hooks）系統 |
| `infra` | 基礎設施工具（資料庫、快取等） |
| `logging` | 日誌系統 |
| `process` | 程序管理 |
| `routing` | 訊息路由邏輯 |
| `secrets` | 機密資料（API keys 等）管理 |
| `security` | 安全機制 |
| `sessions` | 對話會話管理 |
| `shared` | 跨模組共享工具 |
| `types` | TypeScript 型別定義 |
| `utils` | 通用工具函式 |

#### AI 與推理（AI & Reasoning）

| 子目錄 | 推測職責 |
|--------|---------|
| `agents` | AI Agent 核心邏輯 |
| `chat` | 聊天對話處理 |
| `flows` | 流程編排（workflow orchestration） |
| `tasks` | 任務管理與排程 |
| `auto-reply` | 自動回覆邏輯 |
| `cron` | 排程任務（cron jobs） |

#### 通道與通訊（Channels & Communication）

| 子目錄 | 推測職責 |
|--------|---------|
| `channels` | 通道抽象層 |
| `bindings` | 外部系統綁定 |
| `pairing` | 設備配對 |
| `proxy-capture` | 代理捕獲（可能用於 HTTP 代理） |
| `web` | Web 相關功能 |
| `web-fetch` | HTTP 抓取功能 |
| `web-search` | 網頁搜尋功能 |

#### 多媒體與生成（Media & Generation）

| 子目錄 | 推測職責 |
|--------|---------|
| `image-generation` | 圖像生成 |
| `media` | 媒體處理通用層 |
| `media-generation` | 媒體生成（比 image-generation 更廣泛） |
| `media-understanding` | 媒體理解（圖像/影片分析） |
| `music-generation` | 音樂生成 |
| `video-generation` | 影片生成 |
| `canvas-host` | Canvas 渲染宿主 |
| `link-understanding` | 連結內容理解（URL 預覽等） |

#### 語音（Voice）

| 子目錄 | 推測職責 |
|--------|---------|
| `realtime-voice` | 即時語音通話 |
| `realtime-transcription` | 即時語音轉文字 |
| `tts` | 文字轉語音（Text-to-Speech） |

#### 記憶系統（Memory）

| 子目錄 | 推測職責 |
|--------|---------|
| `memory-host-sdk` | 記憶系統宿主端 SDK |

#### 使用者介面（User Interface）

| 子目錄 | 推測職責 |
|--------|---------|
| `cli` | 命令列介面 |
| `commands` | CLI 命令定義 |
| `interactive` | 互動式介面 |
| `terminal` | 終端機功能 |
| `tui` | 文字使用者介面（Text UI） |
| `wizard` | 設定精靈 |
| `ui-app-settings` | UI 應用程式設定 |
| `status` | 狀態顯示 |

#### 插件系統（Plugin System）

| 子目錄 | 推測職責 |
|--------|---------|
| `plugin-sdk` | 插件 SDK 核心程式碼 |
| `plugins` | 插件管理與載入 |
| `node-host` | Node.js 插件宿主 |

#### 國際化與文件（i18n & Docs）

| 子目錄 | 推測職責 |
|--------|---------|
| `i18n` | 國際化（Internationalization） |
| `docs` | 文件生成相關 |
| `markdown` | Markdown 處理 |

#### 整合與相容性（Integration & Compatibility）

| 子目錄 | 推測職責 |
|--------|---------|
| `acp` | Agent Communication Protocol 或類似協議 |
| `compat` | 向後相容性層 |
| `mcp` | Model Context Protocol 整合 |

#### 測試輔助（Test Helpers）

| 子目錄 | 推測職責 |
|--------|---------|
| `test-helpers` | 測試輔助工具 |
| `test-utils` | 測試通用工具函式 |

#### 其他（Miscellaneous）

| 子目錄 | 推測職責 |
|--------|---------|
| `scripts` | 內部腳本 |

### 2.3 `src/` 的架構特徵觀察

從子目錄的命名和組織方式，可以觀察到以下模式：

1. **功能垂直切分**：每個子目錄對應一個功能領域（domain），而非技術分層（如 controller/service/model）。這是典型的 **Feature-based Architecture**（基於功能的架構）。

2. **多媒體是一等公民**：有 6 個子目錄直接與多媒體處理相關（`image-generation`、`media`、`media-generation`、`media-understanding`、`music-generation`、`video-generation`），表示 OpenClaw 不僅處理文字——多媒體生成與理解是其核心能力。

3. **語音是一等公民**：3 個語音相關子目錄（`realtime-voice`、`realtime-transcription`、`tts`）表明即時語音功能是內建的，而非事後附加。

4. **MCP 整合是內建的**：`mcp` 子目錄的存在表示 Model Context Protocol（模型上下文協議）支援直接內建於核心引擎中。

5. **多層 UI**：同時存在 `cli`、`tui`、`interactive`、`terminal`、`wizard` 等子目錄，表示 OpenClaw 有多種使用者互動模式。

---

## 3. 擴充套件——`extensions/` 目錄

### 3.1 概述

`extensions/` 包含 111 個項目，每個都是一個 pnpm workspace 成員。擴充套件是 OpenClaw 的可插拔模組，用於連接外部服務（LLM 供應商、通訊平台、媒體服務等）。

### 3.2 擴充套件分類

我們可以將 111 個擴充套件按功能分為以下幾類：

#### LLM 供應商適配器（LLM Provider Adapters）——約 44 個

涵蓋主流國際供應商（`openai`、`anthropic`、`google`、`mistral`、`groq`、`amazon-bedrock`、`nvidia` 等）、本地推理引擎（`ollama`、`lmstudio`、`vllm`、`sglang`）、聚合閘道（`openrouter`、`litellm`、`cloudflare-ai-gateway`、`vercel-ai-gateway`）、中國大陸供應商（`qianfan`、`qwen`、`moonshot`、`minimax`、`stepfun`、`volcengine`、`alibaba`、`byteplus`）、以及程式碼專用模型（`codex`、`kimi-coding`、`kilocode`、`github-copilot`、`copilot-proxy`）。

#### 通訊通道適配器（Channel Adapters）——約 27 個

覆蓋主流通訊平台：`discord`、`telegram`、`whatsapp`、`slack`、`signal`、`matrix`、`irc`、`msteams`、`line`、`imessage`、`bluebubbles`、`googlechat`、`mattermost`、`twitch`、`feishu`（飛書）、`qqbot`、`zalo`/`zalouser`、`xiaomi`、`nextcloud-talk`、`synology-chat`、`nostr`（去中心化）、`tlon`（Urbit）、`webhooks`（通用），以及測試用的 `qa-channel`、`qa-lab`。

#### 語音與多媒體（Voice & Media）——約 12 個

核心模組：`speech-core`、`talk-voice`、`voice-call`。供應商：`deepgram`、`elevenlabs`。生成核心：`image-generation-core`、`video-generation-core`、`media-understanding-core`、`music-generation-providers`。外部服務：`fal`（fal.ai）、`comfy`（ComfyUI）、`runway`（影片生成）。

#### 其他類別

- **搜尋與資料**（7 個）：`brave`、`duckduckgo`、`exa`、`tavily`、`searxng`、`firecrawl`、`browser`
- **記憶系統**（4 個）：`memory-core`、`memory-lancedb`、`memory-wiki`、`active-memory`
- **任務與工具**（6 個）：`llm-task`、`diffs`、`open-prose`、`thread-ownership`、`synthetic`、`openshell`
- **設備與基礎設施**（5 個）：`device-pair`、`phone-control`、`diagnostics-otel`、`acpx`、`vydra`
- **共享程式碼**（1 個）：`shared`——擴充套件之間的公用程式碼

### 3.3 擴充套件的規模觀察

- **LLM 供應商**佔據了最大比例——超過 40 個擴充套件是 LLM 供應商適配器。這意味著 OpenClaw 的核心價值主張之一是「統一的 LLM 存取層」。
- **通道適配器**有 27 個之多，覆蓋了從主流平台（Discord、Telegram、Slack）到小眾平台（Nostr、Tlon、Synology Chat）的廣泛範圍。
- **中國大陸的 AI 供應商**（千帆、通義千問、月之暗面、MiniMax、階躍星辰、火山引擎、BytePlus）有完整的支援，反映了 OpenClaw 的全球化定位。

---

## 4. 技能包——`skills/` 目錄

### 4.1 概述

`skills/` 包含 53 個技能包，是 AI Agent 可以使用的「工具箱」。技能包不是 pnpm workspace 成員（詳見 [01-pnpm-workspace-config.md](./01-pnpm-workspace-config.md)），暗示它們在運行時被動態載入。

### 4.2 完整列表與分類

#### 開發者工具

`coding-agent`、`github`、`gh-issues`、`tmux`、`skill-creator`（元技能——用來建立新技能的技能）

#### 通訊

`discord`、`slack`、`imsg`（iMessage）、`bluebubbles`、`voice-call`

#### 筆記與知識管理

`obsidian`、`notion`、`bear-notes`、`apple-notes`、`apple-reminders`、`things-mac`（Things for macOS）、`trello`

#### 媒體與內容

`camsnap`、`gifgrep`、`video-frames`、`songsee`、`spotify-player`、`sonoscli`（Sonos 音響）、`nano-pdf`、`canvas`

#### AI 與模型

`gemini`、`openai-whisper`（本地語音辨識）、`openai-whisper-api`、`sherpa-onnx-tts`（本地 TTS）、`model-usage`（使用量追蹤）

#### 工作流與任務

`taskflow`、`taskflow-inbox-triage`、`summarize`、`blogwatcher`、`session-logs`

#### 網路、搜尋與其他

`weather`、`xurl`、`goplaces`、`gog`、`oracle`、`healthcheck`、`node-connect`、`1password`、`blucli`（Bluetooth）、`eightctl`、`himalaya`（郵件）、`peekaboo`（macOS 螢幕）、`sag`、`wacli`、`ordercli`

#### 生態系統

`clawhub`（技能市集整合）、`mcporter`（MCP 匯出工具）

---

## 5. 共享套件——`packages/` 目錄

### 5.1 概述

`packages/` 包含 3 個 workspace 成員，都是 OpenClaw 的內部套件（所有版本都是 `0.0.0-private`，標記為 `private: true`）。這些套件不會被發佈到 npm——它們存在的目的是提供 workspace 內部的程式碼共享與合約定義。

### 5.2 套件列表

| 套件名稱 | npm 名稱 | Export 路徑數 | 角色 |
|---------|----------|-------------|------|
| `plugin-sdk` | `@openclaw/plugin-sdk` | ~40 | 插件開發 SDK——定義了擴充套件與核心引擎之間的 API 合約 |
| `memory-host-sdk` | `@openclaw/memory-host-sdk` | 12 | 記憶系統宿主端 SDK——定義了記憶引擎的介面 |
| `plugin-package-contract` | `@openclaw/plugin-package-contract` | 1 | 插件套件合約——定義了一個插件套件需要滿足的結構 |

### 5.3 `@openclaw/plugin-sdk` 的 Export 路徑

`@openclaw/plugin-sdk` 是最大的共享套件，提供約 40 個子路徑匯出（subpath exports），包括：`./core`、`./plugin-runtime`、`./provider-entry` 等。這些路徑構成了擴充套件與核心引擎之間的 API 介面。

### 5.4 `@openclaw/memory-host-sdk` 的 Export 路徑

記憶系統 SDK 提供 12 個子路徑：

- `./runtime`、`./runtime-core`、`./runtime-cli`、`./runtime-files` — 運行時相關
- `./engine`、`./engine-foundation`、`./engine-storage`、`./engine-embeddings`、`./engine-qmd` — 引擎相關
- `./multimodal` — 多模態記憶
- `./query` — 記憶查詢
- `./secret` — 機密資料存取
- `./status` — 記憶系統狀態

### 5.5 `@openclaw/plugin-package-contract`

這是最小的套件，只有一個匯出路徑 `"."`，指向 `./src/index.ts`。它定義了「一個插件套件長什麼樣子」的合約，供核心引擎在載入插件時使用。

---

## 6. 原生應用——`apps/` 目錄

### 6.1 概述

`apps/` 包含 4 個目錄，對應原生應用程式：

| 目錄 | 平台 | 技術推測 |
|------|------|---------|
| `android` | Android | Kotlin/Java |
| `ios` | iOS | Swift |
| `macos` | macOS | Swift / AppKit |
| `shared` | 跨平台共享程式碼 | — |

`apps/` 不是 pnpm workspace 成員，表示原生應用程式有自己獨立的建置流程（如 Gradle for Android、Xcode for iOS/macOS），不參與 Node.js 的依賴管理。

---

## 7. Web UI——`ui/` 目錄

### 7.1 概述

`ui/` 是一個 pnpm workspace 成員，包含 OpenClaw 的 Web 前端介面。從目錄內容觀察到它包含 `src/`、`public/`、`package.json`、`vite.config.ts`——是一個基於 **Vite** 的現代前端應用。

---

## 8. 輔助目錄

### 8.1 `docs/` — 文件網站

包含 25+ 主題目錄，構成官方文件。不是 workspace 成員。

### 8.2 `scripts/` — 建置與維護腳本

一些重要腳本隨套件發佈：`scripts/npm-runner.mjs`、`scripts/postinstall-bundled-plugins.mjs`、`scripts/windows-cmd-helpers.mjs`。

### 8.3 `test/` 與 `test-fixtures/`

整合測試、E2E 測試及測試用固定資料。

### 8.4 `vendor/` — Vendored 依賴

目前包含 `a2ui`。直接被原始碼引用，不走 npm 機制。

### 8.5 其他目錄

- `patches/`：pnpm patch 補丁檔案，用於局部修改第三方依賴
- `.github/`：GitHub Actions 工作流程、Issue/PR 模板
- `git-hooks/`：Git 提交鉤子（pre-commit 等）
- `qa/`：QA 測試場景（`qa/scenarios/` 隨套件發佈）
- `assets/`：靜態資源（隨套件發佈）
- `Swabble/`：相關子專案

---

## 9. 架構模式總結

從頂層目錄結構中可以提煉出以下架構模式：

### 9.1 「Core + Extensions + Skills」三層模型

```
┌─────────────────────────────────────┐
│           skills/ (53 個)            │  ← 第三層：技能工具箱
│   AI Agent 可呼叫的具體能力          │     運行時動態載入
├─────────────────────────────────────┤
│        extensions/ (111 個)          │  ← 第二層：可插拔擴充
│   通道、供應商、功能模組的適配器      │     Workspace 成員
├─────────────────────────────────────┤
│            src/ (60+ 模組)           │  ← 第一層：核心引擎
│   路由、閘道、Agent、記憶、安全...    │     根套件的一部分
└─────────────────────────────────────┘
```

### 9.2 數字比例

| 組件 | 數量 | 比例 |
|------|------|------|
| 核心模組（src/） | 60+ | — |
| 擴充套件（extensions/） | 111 | 最龐大的部分 |
| 技能包（skills/） | 53 | 第二龐大 |
| 共享套件（packages/） | 3 | 精簡的共享層 |
| 原生應用（apps/） | 4 | 多平台覆蓋 |
| Web UI（ui/） | 1 | 單一前端 |

擴充套件數量（111）遠超核心模組數量（60+），體現了 OpenClaw 的設計哲學：**核心保持精簡，功能透過擴充套件擴展**。

---

## 引用來源

| 檔案路徑 | 行號範圍 | 本文引用位置 | 內容說明 |
|----------|----------|-------------|----------|
| `source-repo/pnpm-workspace.yaml` | 1-4 | §1（交叉引用） | workspace 成員定義 |
| `source-repo/src/` | — | §2 | 核心原始碼目錄列表 |
| `source-repo/extensions/` | — | §3 | 擴充套件完整列表 |
| `source-repo/skills/` | — | §4 | 技能包完整列表 |
| `source-repo/packages/plugin-sdk/package.json` | — | §5.3 | plugin-sdk 的 exports 定義 |
| `source-repo/packages/memory-host-sdk/package.json` | — | §5.4 | memory-host-sdk 的 exports 定義 |
| `source-repo/packages/plugin-package-contract/package.json` | — | §5.5 | plugin-package-contract 的 exports 定義 |
| `source-repo/apps/` | — | §6 | 原生應用目錄列表 |
| `source-repo/ui/` | — | §7 | Web UI 目錄內容 |
| `source-repo/docs/` | — | §8.1 | 文件目錄 |
| `source-repo/scripts/` | — | §8.2 | 腳本目錄 |
| `source-repo/vendor/` | — | §8.4 | Vendored 依賴 |
| `source-repo/package.json` | `files` 欄位 | §8.2, §8.8, §8.9 | 發佈時包含的檔案清單 |
