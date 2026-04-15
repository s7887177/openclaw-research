# OpenClaw 支援的 25+ 平台完整分類

> **所屬 Topic**：多通道範式（Multi-Channel Paradigm）  
> **層級**：Level 6 — 概念綜合層  
> **前置閱讀**：Level 2 Extensions 擴充系統、Level 2 Channels 通道系統

---

## 摘要

本章對 OpenClaw v2026.4.12 支援的所有通道進行系統性分類。基於 `extensions/` 目錄的逐一分析，我們將 21 個通道 Plugin 依用途分為五大類：即時通訊、團隊協作、社群平台、自託管平台、以及特殊/新興平台。每類分析其特性、限制、以及在 OpenClaw 生態中的成熟度。

---

## 1. 分類總覽

| 類別 | 平台數 | 平台列表 |
|------|--------|---------|
| 即時通訊（Instant Messaging） | 6 | WhatsApp, Telegram, Signal, iMessage, BlueBubbles, LINE |
| 團隊協作（Team Collaboration） | 5 | Slack, MS Teams, Google Chat, Mattermost, Feishu |
| 社群平台（Community） | 4 | Discord, IRC, Matrix, Nostr |
| 自託管平台（Self-Hosted） | 3 | Mattermost*, Nextcloud Talk, Synology Chat |
| 特殊/區域（Special/Regional） | 4 | Twitch, QQ Bot, Zalo, Zalo Personal |
| **合計** | **21+** | （部分平台跨類別，如 Mattermost 同時是團隊協作和自託管） |

> **註**：加上 Web Channel（`src/channel-web.ts`）和 QA Channel（`extensions/qa-channel/`），實際通道數超過 23 個。

---

## 2. 即時通訊類（Instant Messaging）

這類通道的共同特性：以一對一或小群組通訊為主，使用者基數龐大，安全/隱私要求高。

### 2.1 WhatsApp

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/whatsapp/` |
| **設定方式** | QR Code 配對 |
| **chatTypes** | direct, group |
| **特殊能力** | reactions ✅, polls ✅, media ✅ |
| **串流** | 即時串流（支援訊息編輯） |
| **限制** | 無 threads, 無原生指令, 自訂格式（非 Markdown） |
| **適合場景** | 家人朋友通訊、跨國通訊 |

WhatsApp 是全球使用者最多的即時通訊 App，OpenClaw 的 WhatsApp 通道使用 QR Code 配對模式連接個人帳號。值得注意的是 OpenClaw 為 WhatsApp 實作了專屬的確認反應邏輯（`WhatsAppAckReactionMode`），支援 `always`、`mentions`、`never` 三種模式，反映了 WhatsApp 群組中的特殊需求。

### 2.2 Telegram

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/telegram/` |
| **設定方式** | BotFather 建立 Bot |
| **chatTypes** | direct, group, channel, thread |
| **特殊能力** | reactions ✅, threads ✅ (Topics), polls ✅, media ✅, nativeCommands ✅ |
| **串流** | 阻塞模式（`blockStreaming: true`） |
| **Markdown** | ✅ (MarkdownV2 + HTML) |
| **適合場景** | 技術社群、隱私敏感用戶、大型群組 |

Telegram 是功能最豐富的通道之一，支援幾乎所有 `ChannelCapabilities` 中定義的能力。唯一的限制是 `blockStreaming`——由於 Telegram API 的編輯頻率限制，OpenClaw 選擇等待完整回應再發送。

### 2.3 Signal

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/signal/` |
| **設定方式** | signal-cli 橋接 |
| **chatTypes** | direct, group |
| **特殊能力** | reactions ✅, media ✅ |
| **串流** | 阻塞模式（`blockStreaming` 隱含，透過 coalesce 預設值 minChars: 1500, idleMs: 1000） |
| **Markdown** | ✅ |
| **適合場景** | 隱私優先用戶、端對端加密需求 |

### 2.4 iMessage / BlueBubbles

| 屬性 | iMessage | BlueBubbles |
|------|----------|-------------|
| **Extension** | `extensions/imessage/` | `extensions/bluebubbles/` |
| **設定方式** | 需要 macOS | macOS App + REST API |
| **chatTypes** | direct, group | direct, group |
| **特殊能力** | media ✅ | media ✅, reactions ✅, edit ✅, unsend ✅, effects ✅, groupManagement ✅ |
| **建議** | 開發中 | **推薦方式** |

BlueBubbles 是 iMessage 的推薦接入方式，提供完整的 iMessage 功能支援，包括訊息特效（Effects）和群組管理——這些是其他通道很少支援的獨特能力。

### 2.5 LINE

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/line/` |
| **設定方式** | LINE Messaging API |
| **chatTypes** | direct, group |
| **特殊能力** | media ✅ |
| **串流** | 阻塞模式 |
| **限制** | 無 reactions, 無 threads, 無原生指令 |
| **適合場景** | 日本/東南亞市場 |

---

## 3. 團隊協作類（Team Collaboration）

### 3.1 Slack

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/slack/` |
| **設定方式** | Socket Mode（App-level Token） |
| **chatTypes** | direct, channel, thread |
| **特殊能力** | reactions ✅, threads ✅, media ✅, nativeCommands ✅ |
| **串流** | 即時串流 |
| **Markdown** | ✅（Slack 自訂 mrkdwn 語法） |
| **適合場景** | 企業/團隊內部、DevOps 自動化 |

Slack 是最成熟的團隊協作通道之一，使用 Block Kit 作為富文本格式。OpenClaw 的 Slack 通道支援隱式 Thread（透過 Thread Binding）自動在對話串中回覆。

### 3.2 Microsoft Teams

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/msteams/` |
| **設定方式** | Teams Bot SDK + Azure AD |
| **chatTypes** | direct, channel, thread |
| **特殊能力** | threads ✅, media ✅, polls ✅ |
| **串流** | 阻塞模式（minChars: 1500, idleMs: 1000） |
| **適合場景** | Microsoft 365 企業環境 |

### 3.3 Google Chat

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/googlechat/` |
| **設定方式** | Google Chat API（Service Account） |
| **chatTypes** | direct, group, thread |
| **特殊能力** | reactions ✅, threads ✅, media ✅ |
| **串流** | 阻塞模式（minChars: 1500, idleMs: 1000） |
| **Markdown** | ✅ |
| **適合場景** | Google Workspace 環境 |

### 3.4 Feishu（飛書/Lark）

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/feishu/` |
| **設定方式** | Bot API + QR Code |
| **chatTypes** | direct, channel |
| **特殊能力** | reactions ✅, threads ✅, media ✅, edit ✅, reply ✅ |
| **串流** | 即時串流 |
| **適合場景** | 中國/亞洲企業市場、字節跳動生態 |

Feishu 的通道實作相對豐富，支援文件/Wiki/雲端硬碟整合，是面向中國市場的重要通道。

### 3.5 Mattermost

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/mattermost/` |
| **設定方式** | Plugin 模式 |
| **chatTypes** | direct, channel, group, thread |
| **特殊能力** | reactions ✅, threads ✅, media ✅, nativeCommands ✅ |
| **串流** | 阻塞模式（minChars: 1500, idleMs: 1000） |
| **適合場景** | Slack 的自託管替代方案 |

Mattermost 同時屬於團隊協作和自託管類別，是 chatTypes 最豐富的通道之一（同時支援 direct, channel, group, thread 四種）。

---

## 4. 社群平台類（Community）

### 4.1 Discord

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/discord/` |
| **設定方式** | Bot Token |
| **chatTypes** | direct, channel, thread |
| **特殊能力** | reactions ✅, threads ✅, media ✅, polls ✅, nativeCommands ✅ |
| **串流** | 即時串流 |
| **Markdown** | ✅ |
| **富互動** | Buttons, Modals, Dropdowns, Embeds |
| **語音** | ✅（@discordjs/voice） |
| **適合場景** | 遊戲社群、開源專案、朋友群組 |

Discord 是 OpenClaw 生態中功能最完整的通道——支援所有 `ChannelCapabilities` 中的能力，外加語音通道支援和富互動元件（Buttons、Modals、Select Menus）。它也是本研究「應用目標二：Discord 語音機器人」的核心通道。

### 4.2 IRC

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/irc/` |
| **設定方式** | Server + Nick |
| **chatTypes** | direct, group |
| **特殊能力** | media ✅ |
| **串流** | 阻塞模式 |
| **Markdown** | ✅ |
| **限制** | textChunkLimit: 350（極短）, 無 reactions, 無 threads |
| **適合場景** | 技術老手、開源社群傳統頻道 |

IRC 是歷史最悠久的通道，也是限制最多的——350 字元的訊息限制要求 OpenClaw 極度精簡回應。

### 4.3 Matrix

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/matrix/` |
| **設定方式** | Federation 帳號 |
| **chatTypes** | direct, group, thread |
| **特殊能力** | reactions ✅, threads ✅, media ✅, polls ✅ |
| **串流** | 即時串流 |
| **適合場景** | 去中心化通訊、自託管需求、E2E 加密 |

Matrix 的聯邦（Federation）架構意味著你可以在自己的伺服器上託管聊天，同時與其他 Matrix 伺服器互通。配合 OpenClaw 的 Local-first 哲學，是完整自主（Self-sovereign）AI 通訊的理想組合。

### 4.4 Nostr

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/nostr/` |
| **設定方式** | Nostr Key Pair |
| **chatTypes** | direct（MVP 階段） |
| **限制** | 無 media, 無 reactions, DM only |
| **適合場景** | 抗審查通訊、去中心化身份 |

Nostr 目前是 MVP 階段，僅支援 DM（無群組、無媒體）。但作為去中心化（Decentralized）和抗審查（Censorship-resistant）的通訊協議，它在概念上與 OpenClaw 的 Local-first 理念高度契合。

---

## 5. 自託管平台類（Self-Hosted）

| 平台 | 設定方式 | chatTypes | 特殊能力 | 串流 |
|------|---------|-----------|---------|------|
| **Mattermost** | Plugin | direct, channel, group, thread | reactions, threads, media, nativeCommands | 阻塞 |
| **Nextcloud Talk** | Webhook | direct, group | reactions, media | 阻塞 |
| **Synology Chat** | Webhook | direct | media | 即時（gateway 模式） |

自託管平台的共同特點：
- 完全控制資料，不經過第三方
- 通常功能較主流平台簡化
- 設定稍複雜（需要自行部署平台）
- 與 OpenClaw 的 Local-first 理念天然契合

Synology Chat 值得注意：它是唯一使用 `deliveryMode: "gateway"` 的通道，意味著訊息透過 OpenClaw 的 HTTP Gateway 中繼，而非直接連接 Synology Chat API。

---

## 6. 特殊/區域平台類（Special/Regional）

### 6.1 Twitch

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/twitch/` |
| **設定方式** | OAuth Token |
| **chatTypes** | group（只有直播聊天室） |
| **限制** | 無 reactions, 無 threads, 無 media, 無私訊 |
| **適合場景** | 直播互動、Twitch 聊天機器人 |

Twitch 是功能最受限的通道——僅支援群組聊天（直播間），無法私訊、無法發圖、無法反應。但在直播場景中，它的價值在於讓 AI 能即時參與觀眾互動。

### 6.2 QQ Bot

| 屬性 | 值 |
|------|-----|
| **Extension** | `extensions/qqbot/` |
| **設定方式** | 官方 Bot API |
| **chatTypes** | direct, group |
| **特殊能力** | media ✅, 語音（SILK 編解碼器） |
| **串流** | 阻塞模式 |
| **適合場景** | 中國市場 |

QQ Bot 的獨特之處在於語音訊息支援——使用 SILK 編解碼器（Codec），這是騰訊自有的音訊格式。

### 6.3 Zalo / Zalo Personal

| 屬性 | Zalo (Bot) | Zalo Personal |
|------|-----------|---------------|
| **Extension** | `extensions/zalo/` | `extensions/zalouser/` |
| **設定方式** | Bot API | QR Login |
| **chatTypes** | direct, group | direct, group |
| **串流** | 阻塞模式 | 阻塞模式 |
| **適合場景** | 越南市場 | 越南市場（個人帳號） |

---

## 7. 平台成熟度評估

| 成熟度 | 平台 | 說明 |
|--------|------|------|
| 🟢 **生產就緒** | Discord, Telegram, Slack, WhatsApp | 功能完整、經過充分測試 |
| 🟡 **穩定可用** | Signal, Matrix, MS Teams, Mattermost, IRC, Google Chat, Feishu, BlueBubbles | 核心功能可靠，部分進階功能待完善 |
| 🟠 **可用但有限** | LINE, Nostr, Twitch, QQ Bot, Zalo, Nextcloud Talk, Synology Chat | 基本功能可用，功能較有限 |
| 🔴 **開發中** | iMessage（直連模式） | 功能不完整，推薦使用 BlueBubbles 替代 |

---

## 引用來源

| 來源 | 路徑 / 說明 |
|------|-------------|
| 所有 Extension 目錄 | `source-repo/extensions/` — 逐一分析 package.json 和 capabilities 宣告 |
| WhatsApp Ack Reaction | `source-repo/src/channels/ack-reactions.ts:45-79` |
| Discord shared.ts | `source-repo/extensions/discord/src/shared.ts:91-97` |
| Telegram shared.ts | `source-repo/extensions/telegram/src/shared.ts:136-143` |
| Slack shared.ts | `source-repo/extensions/slack/src/shared.ts:187-193` |
| WhatsApp shared.ts | `source-repo/extensions/whatsapp/src/shared.ts:132-137` |
| Signal shared.ts | `source-repo/extensions/signal/src/shared.ts:85-89` |
| MS Teams channel.ts | `source-repo/extensions/msteams/src/channel.ts:413-418` |
| Google Chat channel.ts | `source-repo/extensions/googlechat/src/channel.ts:61` |
| Feishu channel.ts | `source-repo/extensions/feishu/src/channel.ts` |
| Matrix channel.ts | `source-repo/extensions/matrix/src/channel.ts` |
| Mattermost channel.ts | `source-repo/extensions/mattermost/src/channel.ts` |
| IRC channel.ts | `source-repo/extensions/irc/src/channel.ts:60` |
| LINE channel-shared.ts | `source-repo/extensions/line/src/channel-shared.ts` |
| Nostr channel.ts | `source-repo/extensions/nostr/src/channel.ts` |
| Twitch plugin.ts | `source-repo/extensions/twitch/src/plugin.ts` |
| BlueBubbles channel-shared.ts | `source-repo/extensions/bluebubbles/src/channel-shared.ts` |
| iMessage shared.ts | `source-repo/extensions/imessage/src/shared.ts` |
| QQ Bot channel.ts | `source-repo/extensions/qqbot/src/channel.ts` |
| Zalo channel.ts | `source-repo/extensions/zalo/src/channel.ts` |
| Nextcloud Talk channel.ts | `source-repo/extensions/nextcloud-talk/src/channel.ts` |
| Synology Chat channel.ts | `source-repo/extensions/synology-chat/src/channel.ts` |
