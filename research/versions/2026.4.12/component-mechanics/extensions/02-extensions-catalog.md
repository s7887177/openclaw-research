# Extensions 完整分類總覽：111 個擴充套件

> **本章摘要**：本文列出 OpenClaw v2026.4.12 中所有 111 個擴充套件的完整清單，按功能分為八大類別。每個擴充套件都附有其 ID、是否預設啟用、描述與提供的能力合約。這份文件是「擴充套件全景地圖」。

---

## 分類方法

分類依據是 `openclaw.plugin.json` 中的結構性欄位：
- **有 `providers` 欄位** → LLM Provider 或 Gateway Proxy
- **有 `channels` 欄位** → Chat/Messaging Platform
- **有 `contracts.speechProviders`** → Speech/Audio
- **有 `contracts.imageGenerationProviders` / `videoGenerationProviders`** → Media
- **有 `contracts.webSearchProviders`** → Web Search
- **有 `kind: "memory"`** → Memory
- **其他** → Tools/Utilities

---

## 1. LLM Providers（LLM 供應商）— 33 個

LLM 供應商擴充套件負責將各家 AI 模型 API 接入 OpenClaw。它們透過 `providers` 欄位宣告供應的 provider ID。

| # | ID | 預設啟用 | 描述 | providers | 特殊合約 |
|---|-----|---------|------|-----------|----------|
| 1 | `anthropic` | ✅ | Anthropic Claude 系列 | `["anthropic"]` | mediaUnderstanding |
| 2 | `anthropic-vertex` | ✅ | Anthropic on Google Vertex | `["anthropic-vertex"]` | — |
| 3 | `openai` | ✅ | OpenAI GPT/o-系列 | `["openai", "openai-codex"]` | speech, realtimeTranscription, realtimeVoice, mediaUnderstanding, imageGeneration, videoGeneration |
| 4 | `google` | ✅ | Google Gemini | `["google", "google-gemini-cli"]` | mediaUnderstanding, imageGeneration, musicGeneration, videoGeneration, webSearch |
| 5 | `deepseek` | ✅ | DeepSeek | `["deepseek"]` | — |
| 6 | `mistral` | ✅ | Mistral AI | `["mistral"]` | mediaUnderstanding |
| 7 | `groq` | ✅ | Groq 推理加速 | — | mediaUnderstanding |
| 8 | `xai` | ✅ | xAI (Grok) | `["xai"]` | webSearch, videoGeneration, tools |
| 9 | `ollama` | ✅ | Ollama 本地模型 | `["ollama"]` | memoryEmbedding, webSearch |
| 10 | `lmstudio` | ✅ | LM Studio 本地模型 | `["lmstudio"]` | — |
| 11 | `openrouter` | ✅ | OpenRouter 統一路由 | `["openrouter"]` | mediaUnderstanding |
| 12 | `fireworks` | ✅ | Fireworks AI | `["fireworks"]` | — |
| 13 | `together` | ✅ | Together AI | `["together"]` | videoGeneration |
| 14 | `amazon-bedrock` | ✅ | Amazon Bedrock | `["amazon-bedrock"]` | — |
| 15 | `amazon-bedrock-mantle` | ✅ | Bedrock Mantle (OpenAI 相容) | `["amazon-bedrock-mantle"]` | — |
| 16 | `nvidia` | ✅ | NVIDIA NIM | `["nvidia"]` | — |
| 17 | `huggingface` | ✅ | Hugging Face | `["huggingface"]` | — |
| 18 | `arcee` | ✅ | Arcee AI | `["arcee"]` | — |
| 19 | `chutes` | ✅ | Chutes.ai | `["chutes"]` | — |
| 20 | `venice` | ✅ | Venice AI | `["venice"]` | — |
| 21 | `sglang` | ✅ | SGLang 自託管 | `["sglang"]` | — |
| 22 | `vllm` | ✅ | vLLM 自託管 | `["vllm"]` | — |
| 23 | `moonshot` | ✅ | Moonshot/Kimi | `["moonshot"]` | mediaUnderstanding, webSearch |
| 24 | `qwen` | ✅ | 通義千問 (Qwen) | `["qwen"]` | mediaUnderstanding, videoGeneration |
| 25 | `qianfan` | ✅ | 百度千帆 | `["qianfan"]` | — |
| 26 | `minimax` | ✅ | MiniMax (海螺) | `["minimax", "minimax-portal"]` | speech, mediaUnderstanding, imageGeneration, musicGeneration, videoGeneration, webSearch |
| 27 | `volcengine` | ✅ | 火山引擎 (字節) | `["volcengine", "volcengine-plan"]` | — |
| 28 | `stepfun` | ✅ | 階躍星辰 (StepFun) | `["stepfun", "stepfun-plan"]` | — |
| 29 | `kimi-coding` | ✅ | Kimi Coding | `["kimi", "kimi-coding"]` | — |
| 30 | `xiaomi` | ✅ | 小米 AI | `["xiaomi"]` | — |
| 31 | `kilocode` | ✅ | Kilo Gateway | `["kilocode"]` | — |
| 32 | `opencode` | ✅ | OpenCode Zen | `["opencode"]` | — |
| 33 | `opencode-go` | ✅ | OpenCode Go | `["opencode-go"]` | — |

> **來源**：所有 `source-repo/extensions/*/openclaw.plugin.json` 中有 `providers` 欄位的擴充套件

### 觀察

- 所有 LLM Provider 都 `enabledByDefault: true`，表示安裝 OpenClaw 後即可選用任何供應商
- 中國系供應商密度高：qwen、qianfan、moonshot、minimax、volcengine、stepfun、kimi-coding、xiaomi 共 8 個
- `openai` 是功能最全面的，幾乎實作了所有合約類型
- `minimax` 是第二全面的，同時提供 LLM、語音、圖片、音樂、影片、搜尋

---

## 2. Chat/Messaging Platforms（聊天/通訊平台）— 22 個

通道擴充套件透過 `channels` 欄位宣告支援的通訊平台。

| # | ID | 描述 | channels | 認證環境變數 |
|---|-----|------|----------|-------------|
| 1 | `discord` | Discord 機器人 | `["discord"]` | `DISCORD_BOT_TOKEN` |
| 2 | `slack` | Slack 機器人 | `["slack"]` | `SLACK_BOT_TOKEN, SLACK_APP_TOKEN, SLACK_USER_TOKEN` |
| 3 | `telegram` | Telegram Bot | `["telegram"]` | `TELEGRAM_BOT_TOKEN` |
| 4 | `whatsapp` | WhatsApp | `["whatsapp"]` | — |
| 5 | `signal` | Signal | `["signal"]` | — |
| 6 | `matrix` | Matrix 協定 | `["matrix"]` | `MATRIX_HOMESERVER, MATRIX_USER_ID, MATRIX_ACCESS_TOKEN` 等 10 個 |
| 7 | `imessage` | Apple iMessage | `["imessage"]` | — |
| 8 | `bluebubbles` | BlueBubbles (iMessage 代理) | `["bluebubbles"]` | — |
| 9 | `msteams` | Microsoft Teams | `["msteams"]` | — |
| 10 | `googlechat` | Google Chat | `["googlechat"]` | — |
| 11 | `feishu` | 飛書/Lark | `["feishu"]` | — |
| 12 | `line` | LINE | `["line"]` | — |
| 13 | `irc` | IRC 協定 | `["irc"]` | — |
| 14 | `nostr` | Nostr NIP-04 加密 DM | `["nostr"]` | — |
| 15 | `twitch` | Twitch | `["twitch"]` | — |
| 16 | `mattermost` | Mattermost | `["mattermost"]` | — |
| 17 | `nextcloud-talk` | Nextcloud Talk | `["nextcloud-talk"]` | — |
| 18 | `synology-chat` | Synology Chat | `["synology-chat"]` | — |
| 19 | `qqbot` | QQ 機器人 | `["qqbot"]` | — |
| 20 | `tlon` | Tlon/Urbit | `["tlon"]` | — |
| 21 | `zalo` | Zalo | `["zalo"]` | — |
| 22 | `zalouser` | Zalo 個人帳號 (zca-js) | `["zalouser"]` | — |

> **來源**：所有 `source-repo/extensions/*/openclaw.plugin.json` 中有 `channels` 欄位的擴充套件

### 觀察

- 通道擴充套件全部**不**預設啟用（無 `enabledByDefault`），因為都需要 Bot Token 或特定配置
- Matrix 的環境變數最多（10 個），反映其去中心化架構的複雜性
- 亞洲市場覆蓋佳：飛書、LINE、QQ Bot、Zalo
- `qa-channel` 是內部測試用的合成通道（見 Tools 分類）

---

## 3. Speech/Audio（語音/音頻）— 7 個

語音類擴充套件透過 `contracts.speechProviders` 或相關合約宣告能力。

| # | ID | 預設啟用 | 描述 | 合約 |
|---|-----|---------|------|------|
| 1 | `speech-core` | — | 語音核心運行時套件 | 重新匯出 `openclaw/plugin-sdk/speech-core` |
| 2 | `elevenlabs` | — | ElevenLabs TTS | `speechProviders: ["elevenlabs"]` |
| 3 | `microsoft` | — | Microsoft Azure 語音 | `speechProviders: ["microsoft"]` |
| 4 | `deepgram` | ✅ | Deepgram 語音轉文字 | `mediaUnderstandingProviders: ["deepgram"]` |
| 5 | `talk-voice` | ✅ | Talk 語音管理（列表/設定） | — |
| 6 | `voice-call` | — | 語音通話（Twilio/Telnyx/Plivo） | — |
| 7 | `vydra` | ✅ | Vydra 媒體（含 TTS） | `speechProviders: ["vydra"]` |

> **注意**：`openai` 和 `minimax` 也提供 `speechProviders` 合約，但它們被歸類為 LLM Provider，因為語音只是其附帶能力。

> **來源**：`source-repo/extensions/elevenlabs/openclaw.plugin.json`、`source-repo/extensions/speech-core/package.json`、`source-repo/extensions/talk-voice/openclaw.plugin.json`

---

## 4. Media — Image/Video/Music（媒體生成）— 7 個

| # | ID | 預設啟用 | 描述 | 合約 |
|---|-----|---------|------|------|
| 1 | `image-generation-core` | — | 圖片生成核心運行時 | 核心套件，無 plugin.json |
| 2 | `video-generation-core` | — | 影片生成核心運行時 | 核心套件，無 plugin.json |
| 3 | `media-understanding-core` | — | 媒體理解核心運行時 | 核心套件 |
| 4 | `fal` | ✅ | fal.ai 圖片/影片 | `imageGeneration, videoGeneration` |
| 5 | `runway` | ✅ | Runway 影片 | `videoGeneration` |
| 6 | `comfy` | ✅ | ComfyUI 本地生成 | `imageGeneration, musicGeneration, videoGeneration` |
| 7 | `alibaba` | ✅ | 阿里巴巴影片 | `videoGeneration` |

> **注意**：`openai`、`google`、`minimax`、`byteplus`、`together`、`xai`、`qwen`、`vydra` 也提供媒體生成能力，但歸類為 LLM Provider。

> **來源**：`source-repo/extensions/fal/openclaw.plugin.json`、`source-repo/extensions/runway/openclaw.plugin.json`、`source-repo/extensions/comfy/openclaw.plugin.json`

---

## 5. Web Search（網路搜尋）— 8 個

| # | ID | 預設啟用 | 描述 | 合約 |
|---|-----|---------|------|------|
| 1 | `tavily` | — | Tavily 搜尋與擷取 | `webSearch, tools: [tavily_search, tavily_extract]` |
| 2 | `exa` | — | Exa 語意搜尋 | `webSearch` |
| 3 | `duckduckgo` | — | DuckDuckGo | `webSearch` |
| 4 | `brave` | — | Brave Search | `webSearch` |
| 5 | `searxng` | — | SearXNG 自託管 | `webSearch` |
| 6 | `perplexity` | — | Perplexity AI | `webSearch` |
| 7 | `firecrawl` | — | Firecrawl 抓取/搜尋 | `webFetch, webSearch, tools: [firecrawl_search, firecrawl_scrape]` |

> **注意**：`ollama`、`moonshot`、`minimax`、`xai` 也提供 `webSearch` 合約，但歸類為 LLM Provider。第 8 個搜尋來源是 `xai` 的 Grok 搜尋。

> **來源**：`source-repo/extensions/tavily/openclaw.plugin.json`、`source-repo/extensions/brave/openclaw.plugin.json`

---

## 6. Memory（記憶系統）— 4 個

| # | ID | kind | 描述 |
|---|-----|------|------|
| 1 | `memory-core` | `memory` | 檔案式記憶搜尋、夢境系統（light/REM/deep） |
| 2 | `memory-lancedb` | `memory` | LanceDB 向量資料庫長期記憶（自動召回/捕獲） |
| 3 | `memory-wiki` | — | 持久性 Wiki 知識庫 |
| 4 | `active-memory` | — | 回覆前自動注入相關記憶的子代理 |

> **來源**：`source-repo/extensions/memory-core/openclaw.plugin.json`、`source-repo/extensions/memory-lancedb/openclaw.plugin.json`、`source-repo/extensions/active-memory/openclaw.plugin.json`

---

## 7. Gateway Proxies（閘道代理）— 5 個

| # | ID | 預設啟用 | 描述 |
|---|-----|---------|------|
| 1 | `cloudflare-ai-gateway` | ✅ | Cloudflare AI Gateway |
| 2 | `vercel-ai-gateway` | ✅ | Vercel AI Gateway |
| 3 | `litellm` | ✅ | LiteLLM 統一閘道（100+ 供應商） |
| 4 | `copilot-proxy` | ✅ | GitHub Copilot Proxy |
| 5 | `github-copilot` | ✅ | GitHub Copilot 直接整合 |

> **來源**：`source-repo/extensions/cloudflare-ai-gateway/openclaw.plugin.json`、`source-repo/extensions/litellm/openclaw.plugin.json`

---

## 8. Tools/Utilities（工具/公用程式）— 25 個

| # | ID | 預設啟用 | 描述 |
|---|-----|---------|------|
| 1 | `browser` | ✅ | 瀏覽器自動化工具 |
| 2 | `openshell` | — | OpenShell 沙盒後端（鏡像/遠端 SSH） |
| 3 | `diagnostics-otel` | — | OpenTelemetry 診斷匯出器 |
| 4 | `diffs` | — | 唯讀 diff 檢視器 |
| 5 | `device-pair` | ✅ | 裝置配對（setup code 生成/審批） |
| 6 | `phone-control` | ✅ | 手機高風險指令控制（相機/螢幕/寫入） |
| 7 | `thread-ownership` | — | Slack 線程所有權防衝突 |
| 8 | `webhooks` | — | Webhook 橋接 |
| 9 | `acpx` | ✅ | ACP 嵌入式運行時後端 |
| 10 | `codex` | — | Codex 應用伺服器 harness |
| 11 | `llm-task` | — | JSON-only LLM 工具（工作流用） |
| 12 | `lobster` | — | Lobster 工作流引擎（型別化 pipeline + 可恢復審批） |
| 13 | `open-prose` | — | OpenProse VM 技能包（/prose 斜線指令） |
| 14 | `qa-channel` | — | QA 合成通道（測試用） |
| 15 | `qa-lab` | — | QA 實驗室（除錯 UI + 場景執行器） |
| 16 | `synthetic` | ✅ | 合成 Provider（測試/模擬用） |
| 17 | `byteplus` | ✅ | BytePlus 影片供應商 |
| 18 | `shared` | — | 跨擴充套件共用模組（非獨立套件） |
| 19 | `zai` | ✅ | Z.AI Provider | 
| 20 | `image-generation-core` | — | 圖片生成核心（無 plugin.json） |
| 21 | `video-generation-core` | — | 影片生成核心（無 plugin.json） |
| 22 | `media-understanding-core` | — | 媒體理解核心（無 plugin.json） |
| 23 | `speech-core` | — | 語音核心（無 plugin.json） |
| 24 | `microsoft-foundry` | ✅ | Microsoft Foundry Provider |
| 25 | `vydra` | ✅ | Vydra 媒體 Provider |

> **來源**：`source-repo/extensions/browser/openclaw.plugin.json`、`source-repo/extensions/openshell/openclaw.plugin.json`、`source-repo/extensions/acpx/openclaw.plugin.json`

---

## 統計摘要

| 類別 | 數量 | 預設啟用數 |
|------|------|-----------|
| LLM Providers | 33 | 33 |
| Chat/Messaging | 22 | 0 |
| Speech/Audio | 7 | 3 |
| Media (Image/Video/Music) | 7 | 4 |
| Web Search | 7 | 0 |
| Memory | 4 | 0 |
| Gateway Proxies | 5 | 5 |
| Tools/Utilities | 26 | 10 |
| **合計** | **111** | **55** |

> 約一半的擴充套件預設啟用。LLM Provider 和 Gateway Proxy 全部預設啟用（零配置即可用），而通道和搜尋擴充套件全部需要明確配置。

---

## 擴充套件命名慣例

從目錄名稱可以觀察到幾個模式：

1. **供應商名稱直接作為 ID**：`anthropic`、`openai`、`google`、`discord`
2. **核心運行時以 `-core` 結尾**：`memory-core`、`speech-core`、`image-generation-core`
3. **變體以供應商-平台命名**：`anthropic-vertex`、`amazon-bedrock-mantle`
4. **中國供應商保留英文名**：`qwen`（千問）、`qianfan`（千帆）、`volcengine`（火山引擎）
5. **多平台型以獨立擴充套件存在**：`zalo` + `zalouser`（OA vs 個人帳號）

---

## 引用來源

| 來源路徑 | 引用內容 |
|----------|----------|
| `source-repo/extensions/` | 完整目錄列表（111 個子目錄） |
| `source-repo/extensions/*/openclaw.plugin.json` | 所有擴充套件的清單檔 |
| `source-repo/extensions/*/package.json` | 所有擴充套件的 npm 套件描述 |
| `source-repo/extensions/anthropic/openclaw.plugin.json` | Anthropic 清單檔 |
| `source-repo/extensions/openai/openclaw.plugin.json` | OpenAI 清單檔 |
| `source-repo/extensions/google/openclaw.plugin.json` | Google 清單檔 |
| `source-repo/extensions/discord/openclaw.plugin.json` | Discord 清單檔 |
| `source-repo/extensions/slack/openclaw.plugin.json` | Slack 清單檔 |
| `source-repo/extensions/matrix/openclaw.plugin.json` | Matrix 清單檔 |
| `source-repo/extensions/memory-core/openclaw.plugin.json` | Memory Core 清單檔 |
| `source-repo/extensions/tavily/openclaw.plugin.json` | Tavily 清單檔 |
| `source-repo/extensions/fal/openclaw.plugin.json` | fal 清單檔 |
| `source-repo/extensions/comfy/openclaw.plugin.json` | ComfyUI 清單檔 |
| `source-repo/extensions/litellm/openclaw.plugin.json` | LiteLLM 清單檔 |
| `source-repo/extensions/acpx/openclaw.plugin.json` | ACPX 清單檔 |
| `source-repo/extensions/active-memory/openclaw.plugin.json` | Active Memory 清單檔 |
