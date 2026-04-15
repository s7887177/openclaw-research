# 02 — 外部依賴清單（External Dependencies）

> **本章摘要**：OpenClaw 的 `package.json` 列舉了約 70 個 production dependencies 與 23 個 devDependencies，合計近 100 個外部套件。本章逐一分類記錄這些依賴，分析各類別在 OpenClaw 架構中的角色，並特別標註那些影響系統行為的關鍵版本選擇（例如使用 Express 5 而非 4、使用 TypeScript 6 而非 5）。此外，本章也記錄 `tsdown.config.ts` 中的 `neverBundle` 清單——那些因技術限制無法被打包進 bundle 的特殊依賴。

---

## 目錄

- [1. Production Dependencies 分類總覽](#1-production-dependencies-分類總覽)
- [2. AI SDK 與模型提供者](#2-ai-sdk-與模型提供者)
- [3. 通訊通道整合（Chat Platforms）](#3-通訊通道整合chat-platforms)
- [4. Web 框架與網路工具](#4-web-框架與網路工具)
- [5. 協定整合（Protocol SDKs）](#5-協定整合protocol-sdks)
- [6. 向量資料庫與搜尋](#6-向量資料庫與搜尋)
- [7. CLI 與使用者介面工具](#7-cli-與使用者介面工具)
- [8. 驗證與 Schema 工具](#8-驗證與-schema-工具)
- [9. 媒體處理](#9-媒體處理)
- [10. 語音處理（TTS/STT）](#10-語音處理ttsstt)
- [11. 日誌與設定](#11-日誌與設定)
- [12. 雲端服務 SDK](#12-雲端服務-sdk)
- [13. 系統工具與雜項](#13-系統工具與雜項)
- [14. devDependencies 分析](#14-devdependencies-分析)
- [15. neverBundle 清單——無法打包的依賴](#15-neverbundle-清單無法打包的依賴)
- [16. 關鍵觀察與結論](#16-關鍵觀察與結論)
- [引用來源](#引用來源)

---

## 1. Production Dependencies 分類總覽

下表按功能類別彙整所有 70 個 production dependencies，提供一個全局視角：

| 類別 | 套件數量 | 代表套件 |
|------|---------|---------|
| AI SDK / 模型提供者 | 7 | `openai`, `@google/genai`, `@anthropic-ai/vertex-sdk` |
| 通訊通道整合 | 8 | `grammy`, `@buape/carbon`, `@slack/bolt`, `matrix-js-sdk` |
| Web 框架與網路 | 4 | `express`, `hono`, `ws`, `undici` |
| 協定整合 | 2 | `@modelcontextprotocol/sdk`, `@agentclientprotocol/sdk` |
| 向量資料庫 | 2 | `@lancedb/lancedb`, `sqlite-vec` |
| CLI / UI | 3 | `commander`, `chalk`, `@clack/prompts` |
| 驗證 / Schema | 3 | `zod`, `ajv`, `@sinclair/typebox` |
| 媒體處理 | 4 | `sharp`, `jimp`, `playwright-core`, `pdfjs-dist` |
| TTS / STT | 3 | `opusscript`, `mpg123-decoder`, `node-edge-tts` |
| 日誌 / 設定 | 4 | `tslog`, `dotenv`, `yaml`, `json5` |
| 雲端 SDK | 7 | `@aws-sdk/*`, `google-auth-library`, `gaxios` |
| 系統工具 / 雜項 | ~23 | `chokidar`, `croner`, `uuid`, `tar`, `proxy-agent` 等 |

> **70 個依賴橫跨 12 個功能類別**——這數字本身就說明了 OpenClaw 的野心：它不只是一個聊天機器人框架，而是一個企圖整合所有主流 AI 模型、所有主流通訊平台、並提供完整媒體處理能力的一站式 AI Agent 平台。

---

## 2. AI SDK 與模型提供者

OpenClaw 支援多個 LLM（Large Language Model，大型語言模型）供應商，透過各自的官方 SDK 進行整合：

| 套件名稱 | 版本 | 用途說明 |
|----------|------|---------|
| `openai` | ^6.34.0 | OpenAI 官方 SDK，存取 GPT 系列模型與 API |
| `@anthropic-ai/vertex-sdk` | ^0.15.0 | Anthropic 透過 Google Vertex AI 存取 Claude 模型 |
| `@google/genai` | ^1.49.0 | Google Generative AI SDK，存取 Gemini 系列模型 |
| `@mariozechner/pi-agent-core` | 0.66.1 | pi-agent 核心框架——Agent 推理迴圈的底層抽象 |
| `@mariozechner/pi-ai` | 0.66.1 | pi-agent AI 層——模型呼叫的統一介面 |
| `@mariozechner/pi-coding-agent` | 0.66.1 | pi-agent coding agent——程式碼生成與理解 |
| `@mariozechner/pi-tui` | 0.66.1 | pi-agent TUI 元件——終端圖形介面 |

**關鍵觀察**：

1. **三大 LLM 供應商完整覆蓋**：OpenAI、Anthropic（透過 Vertex）、Google，確保使用者不被鎖定在單一模型供應商。
2. **pi-agent 系列（@mariozechner）鎖定精確版本 0.66.1**——不使用 `^` 或 `~` 範圍，表明 OpenClaw 對這組套件的 API 穩定性有嚴格要求，或者這是內部團隊維護的套件。
3. **Anthropic 透過 Vertex 而非直接 SDK**：使用 `@anthropic-ai/vertex-sdk` 而不是 `@anthropic-ai/sdk`，暗示 OpenClaw 偏好或預設透過 Google Cloud 的 Vertex AI 平台來存取 Claude 模型，而非直接連接 Anthropic 的 API。這可能與計費、區域限制或企業合規有關。
4. **AWS Bedrock 也有支援**——見下方雲端 SDK 分類，OpenClaw 同時支援透過 AWS Bedrock 存取多種模型。

---

## 3. 通訊通道整合（Chat Platforms）

這是 OpenClaw 最具特色的依賴類別——支援的通訊平台數量之多，在開源 AI Agent 框架中罕見：

| 套件名稱 | 版本 | 對應平台 |
|----------|------|---------|
| `grammy` | ^1.42.0 | **Telegram** Bot API 框架 |
| `@buape/carbon` | 0.15.0 | **Discord** Bot 框架 |
| `discord-api-types` | ^0.38.45 | Discord API 型別定義 |
| `@line/bot-sdk` | ^11.0.0 | **LINE** Messaging API |
| `@slack/bolt` | ^4.7.0 | **Slack** Bot 框架（事件驅動） |
| `@slack/web-api` | ^7.15.0 | Slack Web API 客戶端 |
| `matrix-js-sdk` | 41.3.0 | **Matrix** 開放通訊協定 |
| `nostr-tools` | ^2.23.3 | **Nostr** 去中心化社群協定 |

**關鍵觀察**：

1. **八個套件覆蓋六大平台**：Telegram、Discord、LINE、Slack、Matrix、Nostr。其中 Discord 和 Slack 各需兩個套件。
2. **Matrix 和 Nostr 的加入尤為有趣**——Matrix 是去中心化即時通訊協定，Nostr 是去中心化社群媒體協定。支援這兩者意味著 OpenClaw 不僅面向主流商業平台，也積極擁抱去中心化生態。
3. **Discord 使用 `@buape/carbon`**——這不是最主流的 Discord SDK（更常見的是 `discord.js`）。Carbon 是一個較新的、以 TypeScript 為優先的 Discord 框架，選擇它可能反映了 OpenClaw 團隊對型別安全的偏好。
4. **`matrix-js-sdk` 鎖定精確版本 41.3.0**——Matrix SDK 的 API 變動頻繁，鎖定版本是合理的防禦策略。

**補充：Matrix 加密支援**

| 套件名稱 | 版本 | 用途 |
|----------|------|------|
| `@matrix-org/matrix-sdk-crypto-wasm` | 18.0.0 | Matrix 端到端加密（E2EE）的 WASM 實作 |

Matrix 的端到端加密需要獨立的密碼學模組。使用 WASM（WebAssembly）版本而非純 JavaScript 實作，表明對加密效能的重視。

**補充：飛書/Lark 支援**

| 套件名稱 | 版本 | 用途 |
|----------|------|------|
| `@larksuiteoapi/node-sdk` | ^1.60.0 | **飛書（Lark/Feishu）** 開放平台 SDK |

飛書是字節跳動旗下的企業協作平台，在中國大陸和部分亞洲市場使用廣泛。支援飛書反映了 OpenClaw 對亞洲市場的關注。

> 總計通道相關套件：**10 個**，涵蓋 **7 個通訊平台** + 1 個加密模組。

---

## 4. Web 框架與網路工具

| 套件名稱 | 版本 | 用途說明 |
|----------|------|---------|
| `express` | ^5.2.1 | HTTP 伺服器框架——注意是 **Express 5**，非廣泛使用的 v4 |
| `hono` | 4.12.12 | 輕量 Web 框架，支援多執行環境（Node.js / Edge / Bun） |
| `ws` | ^8.20.0 | WebSocket 伺服器與客戶端 |
| `undici` | 8.0.2 | Node.js 官方 HTTP 客戶端，取代舊的 `http`/`https` 模組 |

**關鍵觀察**：

1. **同時使用 Express 5 和 Hono**——這是一個值得注意的設計選擇。Express 和 Hono 都是 HTTP 框架，同時使用兩者暗示它們服務於不同的場景。可能的解釋：Express 用於傳統的 webhook 接收端點，Hono 用於高效能的 API 路由或 Edge 環境部署。
2. **Express 5 是最新的 major version**（於 2024 年底正式發佈），相較於 Express 4 有許多破壞性變更（移除了 `app.del()`、改變了 middleware 簽名等）。OpenClaw 選擇 v5 表明其代碼庫相對現代化。
3. **`undici` 版本 8.0.2**——undici 是 Node.js 內建 `fetch` API 的底層實作，直接依賴它（而非使用 `node-fetch`）提供更好的效能和更細粒度的控制（如全域代理設定，正如 `run-main.ts` 中引入的 `ensureGlobalUndiciEnvProxyDispatcher`）。

---

## 5. 協定整合（Protocol SDKs）

| 套件名稱 | 版本 | 用途說明 |
|----------|------|---------|
| `@modelcontextprotocol/sdk` | 1.29.0 | **MCP（Model Context Protocol）** SDK |
| `@agentclientprotocol/sdk` | 0.18.2 | **ACP（Agent Client Protocol）** SDK |

**關鍵觀察**：

1. **MCP** 是 Anthropic 主導的開放協定，定義了 AI 模型如何與外部工具和資料源互動。OpenClaw 對 MCP 的支援意味著它可以作為 MCP server 暴露功能給支援 MCP 的 AI 客戶端（如 Claude Desktop、Cursor 等），也可以作為 MCP client 呼叫外部 MCP server。
2. **ACP** 是一個較新的 Agent 間通訊協定。`0.18.2` 的版本號暗示該協定仍在早期階段，OpenClaw 是早期採用者。
3. 這兩個 SDK 對應 `src/` 目錄中的 `mcp/` 和 `acp/` 模組——OpenClaw 為每個協定分配了獨立的模組。

---

## 6. 向量資料庫與搜尋

| 套件名稱 | 版本 | 用途說明 |
|----------|------|---------|
| `@lancedb/lancedb` | ^0.27.2 | **LanceDB** 向量資料庫——用於 embedding 儲存與相似度搜尋 |
| `sqlite-vec` | 0.1.9 | **SQLite** 向量搜尋擴充——在 SQLite 中實現向量相似度查詢 |

**關鍵觀察**：

1. **雙向量引擎策略**：同時支援 LanceDB（專用向量資料庫）和 sqlite-vec（SQLite 擴充）兩種向量搜尋方案。這可能讓使用者依據部署環境選擇：sqlite-vec 適合輕量部署（單檔案），LanceDB 適合需要更高效能的場景。
2. **這兩個套件是記憶系統的基礎**——`src/memory-host-sdk/` 模組很可能使用這些向量資料庫來實現長期記憶的語義搜尋。
3. **`@lancedb/lancedb` 出現在 `neverBundle` 清單中**（見第 15 節），表明它包含原生二進位檔（native binary），無法被 JavaScript bundler 打包。

---

## 7. CLI 與使用者介面工具

| 套件名稱 | 版本 | 用途說明 |
|----------|------|---------|
| `commander` | ^14.0.3 | 命令列引數解析框架 |
| `chalk` | ^5.6.2 | 終端機文字著色 |
| `@clack/prompts` | ^1.2.0 | 精美的互動式 CLI 提示元件 |

**關鍵觀察**：

1. **Commander v14** 是最新 major version，支援 TypeScript 原生型別推斷。在 `run-main.ts` 中直接引入了 `CommanderError`。
2. **Chalk v5** 是 ESM-only 版本（不支援 CommonJS `require()`），進一步確認 OpenClaw 是純 ESM 專案。
3. **@clack/prompts** 是一個新興的 CLI 互動元件庫，提供比 `inquirer` 更現代的 UI。配合 `src/wizard/` 模組，推測用於初始設定引導精靈。

---

## 8. 驗證與 Schema 工具

| 套件名稱 | 版本 | 用途說明 |
|----------|------|---------|
| `zod` | ^4.3.6 | TypeScript-first schema 驗證庫——注意是 **Zod 4** |
| `ajv` | ^8.18.0 | JSON Schema 驗證器 |
| `@sinclair/typebox` | 0.34.49 | JSON Schema 型別建構工具 |

**關鍵觀察**：

1. **三套驗證工具共存**——Zod（TypeScript runtime 驗證）、Ajv（JSON Schema 驗證）、TypeBox（JSON Schema 型別建構）。這不太常見，通常專案只會選用其中一到兩個。可能的解釋：
   - Zod 用於內部業務邏輯的型別驗證（最常見用途）
   - Ajv + TypeBox 用於驗證外部輸入的 JSON Schema（如插件的 manifest 檔、設定檔等）
   - 不同模組在不同時期引入了不同的驗證工具，尚未統一
2. **Zod 4 是全新版本**（2025 年中發佈），相較 Zod 3 有重大 API 變更。使用 Zod 4 再次說明 OpenClaw 積極採用最新版本的依賴。

---

## 9. 媒體處理

| 套件名稱 | 版本 | 用途說明 |
|----------|------|---------|
| `sharp` | ^0.34.5 | 高效能圖片處理（基於 libvips 的原生綁定） |
| `jimp` | ^1.6.1 | 純 JavaScript 圖片處理 |
| `playwright-core` | 1.59.1 | 無頭瀏覽器自動化（Chromium/Firefox/WebKit） |
| `pdfjs-dist` | ^5.6.205 | PDF 文件解析與渲染 |

**關鍵觀察**：

1. **Sharp + Jimp 雙圖片引擎**：Sharp 是原生綁定（效能高但需要編譯），Jimp 是純 JavaScript（跨平台但效能較低）。同時使用兩者可能是為了容錯——如果 Sharp 的原生模組安裝失敗，可以退回到 Jimp。
2. **Playwright-core 的存在意義深遠**——它不只是一個測試工具，在 OpenClaw 的脈絡中，它很可能用於：
   - 網頁截圖與內容擷取（`src/web-fetch/`、`src/link-understanding/`）
   - 動態網頁的渲染與互動
   - 網頁搜尋結果的處理
3. **pdfjs-dist** 讓 OpenClaw 能理解 PDF 文件內容——這是 AI Agent 需要處理使用者上傳文件的重要能力。

---

## 10. 語音處理（TTS/STT）

| 套件名稱 | 版本 | 用途說明 |
|----------|------|---------|
| `opusscript` | ^0.1.1 | Opus 音訊編解碼（WebRTC/Discord 語音的標準格式） |
| `mpg123-decoder` | ^1.0.3 | MP3 解碼器 |
| `node-edge-tts` | ^1.2.10 | 微軟 Edge TTS API 的非官方 Node.js 封裝 |

**關鍵觀察**：

1. **Opus 是語音通話的基礎**——Discord、WebRTC 等平台都使用 Opus 編碼。`opusscript` 讓 OpenClaw 能在 Node.js 中編解碼 Opus 音訊，這是實現 Discord 語音機器人的必要條件。
2. **`node-edge-tts`** 使用微軟 Edge 瀏覽器內建的 TTS 引擎，免費且品質不錯。這是 `src/tts/` 模組的後端之一。
3. **`mpg123-decoder`** 用於解碼 MP3 格式的音訊——某些 TTS API 回傳 MP3 格式，需要解碼後才能轉為 PCM 送入語音通道。

> 這三個套件構成了 **Discord 語音機器人** 的技術基礎——正是我們「應用目標二」（智慧社交機器人）所需的核心能力。

---

## 11. 日誌與設定

| 套件名稱 | 版本 | 用途說明 |
|----------|------|---------|
| `tslog` | ^4.10.2 | TypeScript 日誌庫 |
| `dotenv` | ^17.4.1 | `.env` 檔案環境變數載入 |
| `yaml` | ^2.8.3 | YAML 解析與序列化 |
| `json5` | ^2.2.3 | JSON5 格式（支援註解與尾逗號的 JSON 超集）解析 |

**關鍵觀察**：

1. **支援三種設定檔格式**：YAML、JSON5、以及透過 dotenv 的 `.env`。這給予使用者最大的設定靈活性。
2. **tslog** 是一個功能豐富的 TypeScript 日誌庫，對應 `src/logging.js` 頂層模組。
3. **dotenv v17** 是相當新的版本，表明 OpenClaw 的環境變數管理跟上了最新的 dotenv 功能。

---

## 12. 雲端服務 SDK

| 套件名稱 | 版本 | 用途說明 |
|----------|------|---------|
| `@aws-sdk/client-bedrock` | 3.1028.0 | AWS Bedrock 服務控制面板 |
| `@aws-sdk/client-bedrock-runtime` | 3.1028.0 | AWS Bedrock 模型推論 |
| `@aws-sdk/credential-provider-node` | 3.972.30 | AWS 認證提供者 |
| `@aws/bedrock-token-generator` | ^1.1.0 | Bedrock token 生成 |
| `google-auth-library` | ^10.6.2 | Google Cloud 認證庫 |
| `gaxios` | 7.1.4 | Google 的 Axios 替代品（HTTP 客戶端） |

**關鍵觀察**：

1. **AWS Bedrock 四件套**：控制面板、推論執行、認證、token 生成——完整的 Bedrock 整合。這讓 OpenClaw 能透過 AWS Bedrock 存取 Claude、Llama、Titan 等模型。
2. **Google Cloud 認證**：`google-auth-library` 配合 `@google/genai` 和 `@anthropic-ai/vertex-sdk`，支援透過 Google Cloud 的服務帳號認證來存取 Gemini 和 Vertex AI 上的 Claude。
3. **三大雲端平台覆蓋**：透過 OpenAI API、Google Cloud（Vertex AI + Gemini）、AWS（Bedrock），OpenClaw 實現了 AI 模型存取的完整雲端覆蓋。

---

## 13. 系統工具與雜項

| 套件名稱 | 版本 | 用途說明 |
|----------|------|---------|
| `chokidar` | ^5.0.0 | 檔案系統監視（file watcher） |
| `croner` | ^10.0.1 | Cron 表達式排程器 |
| `uuid` | ^13.0.0 | UUID 生成 |
| `tar` | 7.5.13 | tar 壓縮檔操作 |
| `@lydell/node-pty` | 1.2.0-beta.12 | 虛擬終端機（PTY）——原生綁定 |
| `@homebridge/ciao` | ^1.3.6 | mDNS/Bonjour 服務發現 |
| `proxy-agent` | ^8.0.1 | 全協定 HTTP/HTTPS/SOCKS 代理 |
| `https-proxy-agent` | ^9.0.0 | HTTPS 代理 |

**關鍵觀察**：

1. **`@lydell/node-pty`** 是虛擬終端機的關鍵依賴——它讓 OpenClaw 能建立真正的 PTY session，這對 `src/terminal/` 模組的功能至關重要。Beta 版本號暗示這是一個活躍維護但尚未完全穩定的 fork。
2. **`@homebridge/ciao`** 提供 mDNS（Multicast DNS）服務發現——這是一個出乎意料的依賴。可能用於局域網內的裝置配對（`src/pairing/`），讓 OpenClaw 能被同一網路中的其他裝置自動發現。
3. **雙代理套件**：`proxy-agent`（全協定）+ `https-proxy-agent`（HTTPS 專用），配合 `undici` 的全域代理設定，OpenClaw 對代理的支援非常完整。這對在企業網路中部署 AI Agent 至關重要。
4. **`croner`** 對應 `src/cron/` 模組——OpenClaw 支援基於 Cron 表達式的定時任務排程。

---

## 14. devDependencies 分析

devDependencies 不會出現在生產環境中，但它們揭示了 OpenClaw 的開發工具鏈：

| 類別 | 套件 | 版本 | 用途 |
|------|------|------|------|
| **語言** | TypeScript | 6 | TypeScript 編譯器——注意是 **TS 6** |
| **語言** | `@typescript/native-preview` | 7.0.0-dev | TypeScript 原生編譯器預覽版 |
| **打包** | `tsdown` | 0.21.7 | TypeScript 打包工具 |
| **測試** | `vitest` | 4 | 單元測試框架 |
| **測試** | `jsdom` | 29 | DOM 模擬環境 |
| **程式碼品質** | `oxlint` | >=1.59 | 極高效能的 JavaScript/TypeScript linter |
| **程式碼品質** | `oxfmt` | 0.44.0 | Oxc 項目的 formatter |
| **程式碼品質** | `jscpd` | 4 | 程式碼重複檢測 |
| **程式碼品質** | `madge` | 8 | 模組依賴圖分析（循環依賴檢測） |
| **執行** | `tsx` | 4 | TypeScript 直接執行器 |
| **工具** | `semver` | 7 | 語義化版本處理 |
| **前端** | `lit` | 3 | Web Components 框架 |
| **前端** | `signal-utils` | — | Signals 響應式工具 |

**關鍵觀察**：

1. **TypeScript 6 + Native Preview 7.0.0-dev**——OpenClaw 不僅使用最新的 TypeScript，還在測試即將到來的原生 TypeScript 編譯器（用 Go/Rust 重寫的版本，號稱比現有編譯器快 10 倍）。
2. **oxlint + oxfmt 取代 ESLint + Prettier**——Oxc 是用 Rust 寫的 JavaScript 工具鏈，效能遠超傳統 JavaScript 工具。這個選擇反映了 OpenClaw 對建置效能的高度重視。
3. **madge 的存在很有意義**——madge 是一個分析模組依賴圖、檢測循環依賴的工具。它的存在證實 OpenClaw 團隊**主動管理模組依賴關係**，會定期檢查是否有不當的循環引用。
4. **Lit + signal-utils**——Lit 是 Google 的 Web Components 框架。它出現在 devDependencies 中，推測用於 `src/tui/` 或 `src/canvas-host/` 的某些前端元件（可能是 TUI 渲染引擎的一部分）。

---

## 15. neverBundle 清單——無法打包的依賴

`tsdown.config.ts` 定義了一組「永遠不打包」的依賴：

```typescript
// source-repo/tsdown.config.ts:125-130
const explicitNeverBundleDependencies = [
  "@lancedb/lancedb",
  "@matrix-org/matrix-sdk-crypto-nodejs",
  "matrix-js-sdk",
  ...bundledPluginRuntimeDependencies,
];
```

**解讀**：

| 套件 | 不打包的原因 |
|------|-------------|
| `@lancedb/lancedb` | 包含原生二進位模組（C++ / Rust 編譯產物），JavaScript bundler 無法處理 |
| `@matrix-org/matrix-sdk-crypto-nodejs` | 同上——Matrix 端到端加密的 Node.js 原生綁定 |
| `matrix-js-sdk` | Matrix SDK 體積龐大且有複雜的內部依賴樹 |
| `...bundledPluginRuntimeDependencies` | 插件執行時依賴——動態展開的變數，具體內容取決於已安裝的插件 |

**關鍵觀察**：

1. **neverBundle 的依賴在執行時仍需存在於 `node_modules/` 中**——它們不會被包進單一的 bundle 檔案，而是保持為外部依賴，由 Node.js 的 module resolution 機制在執行時動態載入。
2. **這影響部署策略**——部署 OpenClaw 時不能只上傳 bundle 檔案，還需要確保這些 neverBundle 依賴被正確安裝。
3. **`bundledPluginRuntimeDependencies` 是動態的**——具體內容取決於打包設定中定義的 bundled plugin 列表，這意味著不同的打包配置可能產生不同的 neverBundle 清單。

---

## 16. 關鍵觀察與結論

### 16.1 依賴版本策略的兩極化

OpenClaw 的依賴版本管理呈現明顯的兩極化：

- **積極更新派**：Express 5、TypeScript 6、Zod 4、Commander 14——全都是各自生態系中的最新 major version。
- **嚴格鎖定派**：pi-agent 系列（0.66.1）、matrix-js-sdk（41.3.0）、hono（4.12.12）——精確版本，不接受任何自動升級。

這種策略意味著：「對自己有信心能跟進的套件，永遠追最新；對 API 不穩定或改動風險高的套件，嚴格鎖定。」

### 16.2 功能覆蓋的野心

70 個 production dependencies 橫跨 AI 模型、通訊平台、媒體處理、語音、向量搜尋、協定整合等領域。OpenClaw 顯然企圖成為一個「All-in-One」的 AI Agent 平台，而非一個需要使用者自行整合各種功能的輕量框架。

### 16.3 對我們兩個應用目標的意義

| 目標 | 相關依賴 | 意義 |
|------|---------|------|
| Copilot CLI 插件 | `@modelcontextprotocol/sdk`, `commander`, `plugin-sdk` | MCP 整合已內建，CLI 工具鏈完善 |
| Discord 語音機器人 | `@buape/carbon`, `discord-api-types`, `opusscript`, `node-edge-tts` | Discord + 語音的技術棧完整，無需額外安裝套件 |

---

## 引用來源

| 檔案路徑 | 行號範圍 | 本章引用內容 |
|----------|---------|-------------|
| `source-repo/package.json` | dependencies 區段 | 全部 70 個 production dependencies 的名稱與版本 |
| `source-repo/package.json` | devDependencies 區段 | 全部 23 個 devDependencies 的名稱與版本 |
| `source-repo/tsdown.config.ts` | 125-130 | `explicitNeverBundleDependencies` 清單 |
| `source-repo/src/cli/run-main.ts` | 5 | `commander` 的 `CommanderError` 引入 |
| `source-repo/src/cli/run-main.ts` | 10 | `undici-global-dispatcher.js` 引入（undici 代理） |
