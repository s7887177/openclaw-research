# Channels 元件機制：OpenClaw 通道整合系統

> **層級**：Level 2 — Component Mechanics  
> **版本**：2026.4.12  
> **範圍**：`src/channels/`（約 128 個檔案）+ `extensions/`（80+ 平台擴充）

## 本章摘要

Channel 系統是 OpenClaw 與外部通訊平台的**橋接層**。每個平台（Discord、Telegram、Slack 等）透過 **ChannelPlugin** 介面接入，提供統一的訊息收發、狀態管理、安全控制能力。系統支援 80+ 個平台擴充，涵蓋即時通訊、社群平台、企業協作工具等。

本章涵蓋：

| 子章節 | 檔案 | 內容 |
|--------|------|------|
| [01-Registry 與 Plugin 型別](./01-registry-and-plugin.md) | `plugins/types.plugin.ts`, `registry.ts` | Plugin 介面、Registry、能力宣告 |
| [02-核心子系統](./02-core-subsystems.md) | `mention-gating.ts`, `status-reactions.ts`, `typing.ts` | Mention 門控、狀態反應、打字指示 |
| [03-安全與允許清單](./03-security-and-allowlist.md) | `allowlist-match.ts`, `allow-from.ts` | 允許清單匹配、安全策略 |
| [04-平台擴充生態](./04-platform-extensions.md) | `extensions/` | 80+ 平台概覽、擴充架構 |

---

## 目錄結構總覽

### src/channels/

| 檔案/子目錄 | 用途 |
|-------------|------|
| `plugins/` | Plugin 型別定義、Registry、適配器介面 |
| `allowlists/` | 允許清單管理 |
| `transport/` | 傳輸層（stall-watchdog） |
| `web/` | Web 介面相關 |
| `registry.ts` | 高階 Registry（正規化、查詢） |
| `mention-gating.ts` | Mention 門控邏輯 |
| `status-reactions.ts` | 狀態反應 Emoji 系統 |
| `typing.ts` | 打字指示控制 |
| `allowlist-match.ts` | 允許清單匹配引擎 |
| `allow-from.ts` | 來源允許邏輯 |
| `channel-config.ts` | 通道配置解析 |
| `session.ts` | Session 管理 |
| `thread-bindings-policy.ts` | Thread 綁定策略 |
| `inbound-debounce-policy.ts` | 入站防抖策略 |
| `draft-stream-controls.ts` | 草稿串流控制 |
| `run-state-machine.ts` | 執行狀態機 |
| `model-overrides.ts` | 模型覆寫 |

### extensions/（通道相關）

80+ 擴充目錄，主要通道平台包括：

| 類別 | 平台 |
|------|------|
| **即時通訊** | discord, telegram, whatsapp, signal, line, zalo |
| **企業協作** | slack, googlechat, feishu, qqbot, synology-chat |
| **社群** | twitch, nostr, tlon |
| **蘋果生態** | imessage, bluebubbles |
| **開源協議** | irc, matrix |
| **通用介面** | webhooks, qa-channel, synthetic |
| **語音** | talk-voice, voice-call, phone-control |

---

## 核心常數速查

| 常數 | 值 | 說明 | 來源 |
|------|-----|------|------|
| 狀態反應 — queued | 👀 | 排隊中 | `status-reactions.ts:58` |
| 狀態反應 — thinking | 🤔 | 思考中 | `status-reactions.ts:59` |
| 狀態反應 — tool | 🔥 | 使用工具 | `status-reactions.ts:60` |
| 狀態反應 — coding | 👨‍💻 | 寫程式 | `status-reactions.ts:61` |
| 狀態反應 — web | ⚡ | 瀏覽網頁 | `status-reactions.ts:62` |
| 狀態反應 — done | 👍 | 完成 | `status-reactions.ts:63` |
| 狀態反應 — error | 😱 | 錯誤 | `status-reactions.ts:64` |
| 狀態反應 — stallSoft | 🥱 | 輕度卡頓 | `status-reactions.ts:65` |
| 狀態反應 — stallHard | 😨 | 嚴重卡頓 | `status-reactions.ts:66` |
| 狀態反應 — compacting | ✍ | 壓縮中 | `status-reactions.ts:67` |
| debounceMs | 700 | 反應防抖 | `status-reactions.ts:71` |
| stallSoftMs | 10,000 | 輕度卡頓閾值 | `status-reactions.ts:72` |
| stallHardMs | 30,000 | 嚴重卡頓閾值 | `status-reactions.ts:73` |
| doneHoldMs | 1,500 | 完成保持時間 | `status-reactions.ts:74` |
| errorHoldMs | 2,500 | 錯誤保持時間 | `status-reactions.ts:75` |
| typing keepaliveMs | 3,000 | 打字保活間隔 | `typing.ts:25` |
| typing maxDurationMs | 60,000 | 打字最長持續 | `typing.ts:27` |
| typing maxFailures | 2 | 最大連續失敗 | `typing.ts:26` |

---

## 引用來源總表

| 引用 | 檔案路徑 | 行號 |
|------|----------|------|
| ChannelPlugin 型別 | `source-repo/src/channels/plugins/types.plugin.ts` | 53-96 |
| ChannelCapabilities | `source-repo/src/channels/plugins/types.core.ts` | 253-266 |
| ChannelMeta | `source-repo/src/channels/plugins/types.core.ts` | 142-164 |
| ChannelAccountSnapshot | `source-repo/src/channels/plugins/types.core.ts` | 167-230 |
| listChannelPlugins() | `source-repo/src/channels/plugins/registry.ts` | 11 |
| getChannelPlugin() | `source-repo/src/channels/plugins/registry.ts` | 19 |
| 高階 Registry | `source-repo/src/channels/registry.ts` | 1-113 |
| StatusReactionEmojis | `source-repo/src/channels/status-reactions.ts` | 19-30 |
| DEFAULT_EMOJIS | `source-repo/src/channels/status-reactions.ts` | 57-68 |
| DEFAULT_TIMING | `source-repo/src/channels/status-reactions.ts` | 70-76 |
| MentionGating | `source-repo/src/channels/mention-gating.ts` | 1-80 |
| AllowlistMatchSource | `source-repo/src/channels/allowlist-match.ts` | 6-16 |
| TypingCallbacks | `source-repo/src/channels/typing.ts` | 4-9 |
