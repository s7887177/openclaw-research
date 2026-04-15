# Channels 子章節 04：平台擴充生態

> **引用範圍**：`extensions/` 目錄、`src/channels/plugins/` 架構

## 1. 擴充架構

OpenClaw 的通道 Plugin 以 **extensions/** 目錄下的獨立套件形式存在。每個擴充是一個獨立的 TypeScript 模組，實作 `ChannelPlugin` 介面並向 Registry 註冊自己。

### 1.1 擴充載入流程

```
1. 啟動時掃描 extensions/ 目錄
2. 透過 import() 動態載入各擴充模組
3. 每個模組呼叫 registerChannelPlugin(plugin)
4. Registry 建立 Map<string, ChannelPlugin>
5. 高階 Registry 提供配置感知查詢
```

> **引用來源**：`source-repo/src/channels/plugins/registry.ts:1-31`

### 1.2 擴充結構模板

典型的通道擴充目錄結構：

```
extensions/{channel-name}/
├── src/
│   ├── index.ts              # 入口，建立並註冊 ChannelPlugin
│   ├── config.ts             # ChannelConfigAdapter 實作
│   ├── security.ts           # ChannelSecurityAdapter 實作
│   ├── outbound.ts           # 出站訊息處理
│   ├── lifecycle.ts          # 連線生命週期
│   ├── commands.ts           # 原生指令
│   └── types.ts              # 平台特定型別
├── package.json
└── tsconfig.json
```

---

## 2. 完整平台清單

以下根據 `extensions/` 目錄列出所有 80+ 擴充，按功能分類：

### 2.1 主流即時通訊（10 個通道）

| 擴充名稱 | 平台 | 說明 |
|----------|------|------|
| `discord` | Discord | 最完整的通道之一；支援 Slash Commands、Thread、Reactions、Voice |
| `telegram` | Telegram | Bot API；支援群組、超級群組、頻道 |
| `whatsapp` | WhatsApp | 透過 WhatsApp Business API 或 Baileys |
| `signal` | Signal | 透過 signal-cli 或 signald |
| `slack` | Slack | 支援 Socket Mode 和 Events API；Slash Commands、Thread |
| `line` | LINE | LINE Messaging API |
| `zalo` | Zalo | 越南最大即時通訊 |
| `zalouser` | Zalo User | Zalo 個人帳號模式 |
| `imessage` | iMessage | macOS 限定；透過 AppleScript 或 BlueBubbles |
| `bluebubbles` | BlueBubbles | iMessage 橋接伺服器 |

### 2.2 企業協作（6 個通道）

| 擴充名稱 | 平台 | 說明 |
|----------|------|------|
| `googlechat` | Google Chat | Google Workspace 整合 |
| `feishu` | 飛書（Lark） | 字節跳動企業通訊 |
| `qqbot` | QQ Bot | 騰訊 QQ 機器人 |
| `synology-chat` | Synology Chat | 群暉 NAS 聊天 |
| `msteams`* | Microsoft Teams | 企業通訊（如果存在） |
| `mattermost`* | Mattermost | 開源企業通訊 |

### 2.3 社群平台（3 個通道）

| 擴充名稱 | 平台 | 說明 |
|----------|------|------|
| `twitch` | Twitch | 直播平台聊天室 |
| `nostr` | Nostr | 去中心化社交協議 |
| `tlon` | Tlon/Urbit | Urbit 生態社交 |

### 2.4 開源通訊協議（2 個通道）

| 擴充名稱 | 平台 | 說明 |
|----------|------|------|
| `irc` | IRC | 經典網路聊天協議 |
| `matrix`* | Matrix | 開源去中心化通訊 |

### 2.5 通用介面（3 個通道）

| 擴充名稱 | 用途 |
|----------|------|
| `webhooks` | 通用 Webhook 入站/出站 |
| `qa-channel` | 測試用通道 |
| `synthetic` | 合成/模擬通道 |

### 2.6 語音與電話（3 個通道）

| 擴充名稱 | 用途 |
|----------|------|
| `talk-voice` | 語音對話模式 |
| `voice-call` | 語音通話 |
| `phone-control` | 電話控制 |

---

## 3. AI/LLM Provider 擴充

`extensions/` 中也包含大量 AI Provider 擴充（非通道，但共用擴充基礎設施）：

### 3.1 主流 Provider

| 擴充名稱 | Provider |
|----------|----------|
| `openai` | OpenAI (GPT-4o, o1, etc.) |
| `anthropic` | Anthropic (Claude) |
| `anthropic-vertex` | Anthropic via Google Vertex |
| `google` | Google (Gemini) |
| `deepseek` | DeepSeek |
| `xai` | xAI (Grok) |
| `codex` | OpenAI Codex |

### 3.2 雲端 Provider

| 擴充名稱 | Provider |
|----------|----------|
| `amazon-bedrock` | AWS Bedrock |
| `amazon-bedrock-mantle` | AWS Bedrock Mantle |
| `cloudflare-ai-gateway` | Cloudflare AI Gateway |
| `vercel-ai-gateway` | Vercel AI Gateway |
| `github-copilot` | GitHub Copilot |
| `copilot-proxy` | Copilot Proxy |

### 3.3 本地/開源 Provider

| 擴充名稱 | Provider |
|----------|----------|
| `ollama` | Ollama（本地 LLM） |
| `vllm` | vLLM 推理引擎 |
| `sglang` | SGLang 推理框架 |
| `together` | Together AI |
| `groq` | Groq（LPU 加速） |
| `fireworks` | Fireworks AI |
| `openrouter` | OpenRouter（聚合） |

### 3.4 中國 Provider

| 擴充名稱 | Provider |
|----------|----------|
| `qwen` | 通義千問（阿里巴巴） |
| `qianfan` | 千帆（百度） |
| `alibaba` | 阿里巴巴 AI |
| `volcengine` | 火山引擎（字節跳動） |
| `xiaomi` | 小米 AI |
| `stepfun` | 階躍星辰 |
| `byteplus` | BytePlus |

### 3.5 媒體生成

| 擴充名稱 | 用途 |
|----------|------|
| `image-generation-core` | 圖片生成核心 |
| `video-generation-core` | 影片生成核心 |
| `video-generation-providers.live.test.ts` | 影片生成 Provider 測試 |
| `fal` | Fal.ai（圖片/影片生成） |
| `comfy` | ComfyUI 整合 |
| `runway` | Runway（影片生成） |
| `venice` | Venice AI |
| `vydra` | Vydra AI |

### 3.6 搜尋與工具

| 擴充名稱 | 用途 |
|----------|------|
| `brave` | Brave Search |
| `duckduckgo` | DuckDuckGo Search |
| `exa` | Exa 搜尋 |
| `tavily` | Tavily 搜尋 |
| `searxng` | SearXNG 元搜尋 |
| `firecrawl` | Firecrawl 網頁爬蟲 |
| `perplexity` | Perplexity AI |
| `browser` | 瀏覽器控制 |

### 3.7 其他

| 擴充名稱 | 用途 |
|----------|------|
| `deepgram` | 語音辨識（STT） |
| `elevenlabs` | 語音合成（TTS） |
| `speech-core` | 語音核心 |
| `active-memory` | Active Memory 引擎 |
| `diffs` | 差異比對 |
| `diagnostics-otel` | OpenTelemetry 診斷 |
| `device-pair` | 設備配對 |
| `acpx` | ACP 擴充 |
| `open-prose` | 文件處理 |
| `openshell` | Shell 整合 |
| `opencode` | 程式碼分析 |
| `opencode-go` | Go 語言程式碼分析 |
| `thread-ownership` | Thread 擁有權管理 |
| `qa-lab` | QA 測試實驗室 |
| `shared` | 共享工具函式 |
| `zai` | Zai AI |
| `huggingface` | Hugging Face |
| `nvidia` | NVIDIA NIM |
| `chutes` | Chutes AI |
| `arcee` | Arcee AI |

---

## 4. 通道能力矩陣

以下列出主要通道的能力差異：

| 平台 | DM | Group | Thread | Reactions | Edit | Media | Commands | Streaming |
|------|:--:|:-----:|:------:|:---------:|:----:|:-----:|:--------:|:---------:|
| Discord | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Slack | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Telegram | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| WhatsApp | ✅ | ✅ | — | ✅ | — | ✅ | — | — |
| Signal | ✅ | ✅ | — | ✅ | — | ✅ | — | — |
| LINE | ✅ | ✅ | — | — | — | ✅ | — | — |
| IRC | ✅ | ✅ | — | — | — | — | — | — |
| Twitch | — | ✅ | — | — | — | — | ✅ | — |
| Webhooks | ✅ | — | — | — | — | — | — | — |

（✅ = 支援, — = 不支援/有限支援）

> **注意**：此矩陣基於 `ChannelCapabilities` 型別推斷，實際支援程度取決於各擴充的實作完整度。

---

## 5. Discord 擴充深度分析

作為 Eason 應用目標（Discord 語音社交機器人）最相關的通道，Discord 擴充值得特別關注：

### 5.1 能力

- **chatTypes**：direct, group, thread
- **reactions**：✅（Discord 原生反應）
- **edit**：✅
- **unsend**：✅
- **reply**：✅
- **threads**：✅（Discord 論壇/Thread）
- **media**：✅（圖片、影片、檔案）
- **nativeCommands**：✅（Slash Commands）
- **blockStreaming**：✅

### 5.2 語音相關

`extensions/talk-voice/` 和 `extensions/voice-call/` 提供語音能力，可能與 Discord 語音頻道整合。這是實現「Discord 語音社交機器人」的關鍵擴充。

### 5.3 角色路由

`AgentBindingMatch` 支援 `roles?: string[]`，可根據 Discord 角色將不同使用者路由到不同 Agent。

```typescript
// source-repo/src/config/types.agents.ts:39
roles?: string[];  // Discord role IDs used for role-based routing.
```

> **引用來源**：`source-repo/src/config/types.agents.ts:39`

---

## 6. 擴充開發指南

### 6.1 最小可行 Plugin

要建立新通道 Plugin，最少需要：

```typescript
import { registerChannelPlugin } from "../../src/channels/plugins/registry.js";
import type { ChannelPlugin } from "../../src/channels/plugins/types.plugin.js";

const myPlugin: ChannelPlugin = {
  id: "my-channel",
  meta: {
    id: "my-channel",
    label: "My Channel",
    selectionLabel: "My Channel",
    docsPath: "/docs/my-channel",
    blurb: "A custom channel",
  },
  capabilities: {
    chatTypes: ["direct"],
  },
  config: {
    resolveAccount: (cfg) => ({ /* ... */ }),
    // ... 配置解析邏輯
  },
};

registerChannelPlugin(myPlugin);
```

### 6.2 漸進式擴充

可以逐步添加適配器來增強能力：

```
1. config（必填）→ 帳號解析
2. + security → 允許清單、DM 策略
3. + outbound → 出站訊息
4. + lifecycle → 連線管理
5. + threading → Thread 支援
6. + streaming → 串流回覆
7. + commands → 原生指令
8. + agentTools → 平台特定工具
```

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| Plugin Registry | `source-repo/src/channels/plugins/registry.ts` | 1-31 |
| ChannelCapabilities | `source-repo/src/channels/plugins/types.core.ts` | 253-266 |
| Discord 角色路由 | `source-repo/src/config/types.agents.ts` | 39 |
| extensions/ 目錄 | `source-repo/extensions/` | — |
| ChannelPlugin 介面 | `source-repo/src/channels/plugins/types.plugin.ts` | 53-96 |
