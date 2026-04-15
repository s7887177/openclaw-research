# Telegram 整合指南

## 目錄

- [1. Telegram Bot 概述](#1-telegram-bot-概述)
  - [1.1 為什麼選擇 Telegram](#11-為什麼選擇-telegram)
  - [1.2 Telegram Bot 的獨特優勢](#12-telegram-bot-的獨特優勢)
  - [1.3 Telegram vs 其他通道](#13-telegram-vs-其他通道)
- [2. BotFather 建立 Bot](#2-botfather-建立-bot)
  - [2.1 與 BotFather 對話](#21-與-botfather-對話)
  - [2.2 取得 botToken](#22-取得-bottoken)
  - [2.3 設定 Bot 資訊](#23-設定-bot-資訊)
  - [2.4 設定 Bot Commands](#24-設定-bot-commands)
- [3. OpenClaw Telegram 配置](#3-openclaw-telegram-配置)
  - [3.1 基礎配置](#31-基礎配置)
  - [3.2 完整配置範例](#32-完整配置範例)
  - [3.3 配置欄位詳解](#33-配置欄位詳解)
- [4. Webhook vs Long Polling](#4-webhook-vs-long-polling)
  - [4.1 Long Polling 模式](#41-long-polling-模式)
  - [4.2 Webhook 模式](#42-webhook-模式)
  - [4.3 選擇建議](#43-選擇建議)
- [5. 使用者白名單](#5-使用者白名單)
  - [5.1 allowFrom 配置](#51-allowfrom-配置)
  - [5.2 取得 Telegram User ID](#52-取得-telegram-user-id)
  - [5.3 動態白名單](#53-動態白名單)
- [6. 群組整合](#6-群組整合)
  - [6.1 Bot 加入群組](#61-bot-加入群組)
  - [6.2 群組觸發模式](#62-群組觸發模式)
  - [6.3 群組權限設定](#63-群組權限設定)
  - [6.4 頻道整合（Channel）](#64-頻道整合channel)
- [7. 進階功能](#7-進階功能)
  - [7.1 Inline 查詢](#71-inline-查詢)
  - [7.2 鍵盤按鈕](#72-鍵盤按鈕)
  - [7.3 語音訊息](#73-語音訊息)
  - [7.4 文件處理](#74-文件處理)
- [8. Telegram 特色整合](#8-telegram-特色整合)
  - [8.1 Markdown 格式化](#81-markdown-格式化)
  - [8.2 長訊息處理](#82-長訊息處理)
  - [8.3 訊息編輯](#83-訊息編輯)
- [9. 安全考量](#9-安全考量)
- [10. 部署與維護](#10-部署與維護)

---

## 1. Telegram Bot 概述

### 1.1 為什麼選擇 Telegram

Telegram 是對 Bot 最友好的即時通訊平台。它提供了完整且免費的 Bot API，沒有訊息費用，沒有審核流程，而且 API 文件非常詳細。對於技術人員來說，Telegram 是最理想的 AI 助手通道之一。

### 1.2 Telegram Bot 的獨特優勢

- **免費且無限制**：Bot API 完全免費，不限訊息數量
- **豐富的 Bot API**：支援按鈕、內聯查詢、支付、遊戲等
- **原生 Markdown 支援**：回覆可使用 Markdown 格式化
- **隱私模式**：Bot 在群組中可選擇只接收 @提及的訊息
- **文件大小限制寬鬆**：支援最大 2GB 的文件
- **全平台客戶端**：桌面、手機、網頁全覆蓋
- **開放原始碼客戶端**：協議公開，第三方客戶端眾多

### 1.3 Telegram vs 其他通道

| 特性 | Telegram | Discord | WhatsApp |
|------|----------|---------|----------|
| Bot API 費用 | 免費 | 免費 | 付費（Business API） |
| 設定複雜度 | 非常低 | 中等 | 高 |
| Markdown 回覆 | 原生 | 原生 | 不支援 |
| 文件大小限制 | 2GB | 25MB | 100MB |
| 語音通話 | 支援（但 Bot API 有限） | 完整支援 | 支援 |
| 審核流程 | 無 | 無 | 需要 |
| 隱私保護 | 雲端加密 | 標準 | 端到端加密 |

---

## 2. BotFather 建立 Bot

### 2.1 與 BotFather 對話

BotFather 是 Telegram 的官方 Bot 管理工具，用來建立和管理 Bot：

```
1. 在 Telegram 中搜尋 @BotFather
2. 開始對話，發送 /start
3. 發送 /newbot 建立新 Bot
```

### 2.2 取得 botToken

```
你: /newbot
BotFather: Alright, a new bot. How are we going to call it? 
          Please choose a name for your bot.

你: EasonClaw Assistant
BotFather: Good. Now let's choose a username for your bot. 
          It must end in `bot`. Like this, for example: 
          TetrisBot or tetris_bot.

你: EasonClawBot
BotFather: Done! Congratulations on your new bot. You will find it at 
          t.me/EasonClawBot. You can now add a description, about 
          section and profile picture for your bot, see /help for a 
          list of commands. By the way, when you've finished creating 
          your cool bot, ping our Bot Support if you want a better 
          username for it. Don't forget to disable joining groups if 
          you don't need that functionality.
          
          Use this token to access the HTTP API:
          7123456789:AAHdqTcvCH1vGWJxfSeofSAs0K5PALDsaw
          
          Keep your token secure and store it safely.
```

```bash
# 安全存放 Token
echo "TELEGRAM_BOT_TOKEN=7123456789:AAHdqTcvCH1vGWJxfSeofSAs0K5PALDsaw" >> ~/.openclaw/.env
```

> ⚠️ **永遠不要將 botToken 提交到版本控制系統中！**

### 2.3 設定 Bot 資訊

```
/setdescription - 設定 Bot 的說明文字（在搜尋結果中顯示）
/setabouttext   - 設定 Bot 的關於文字（在 Bot 資訊頁面顯示）
/setuserpic     - 設定 Bot 的頭像
/setcommands    - 設定 Bot 的命令選單
```

### 2.4 設定 Bot Commands

向 BotFather 發送 `/setcommands`，然後輸入你的命令列表：

```
help - 顯示使用說明
status - 查看系統狀態
reset - 重置對話記憶
voice - 切換語音回覆模式
lang - 切換語言
```

設定後，使用者在對話框中輸入 `/` 就會看到命令選單。

---

## 3. OpenClaw Telegram 配置

### 3.1 基礎配置

最簡單的 Telegram 配置只需要 botToken：

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "${TELEGRAM_BOT_TOKEN}"
    }
  }
}
```

### 3.2 完整配置範例

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      
      "connectionMode": "polling",
      
      "allowFrom": {
        "users": [
          "telegram:123456789",
          "telegram:987654321"
        ],
        "groups": [
          "group:-1001234567890"
        ]
      },
      
      "dmPolicy": {
        "enabled": true,
        "allowUnknownUsers": false,
        "unauthorizedReply": "抱歉，你不在我的白名單中。請聯繫管理員。",
        "greeting": "你好！我是 EasonClaw 🐾\n輸入 /help 查看使用說明。"
      },
      
      "groupPolicy": {
        "enabled": true,
        "triggerMode": "mention",
        "privacyMode": true,
        "respondInReply": true,
        "cooldown": 3
      },
      
      "formatting": {
        "parseMode": "MarkdownV2",
        "disableWebPagePreview": false,
        "disableNotification": false
      },
      
      "voice": {
        "enabled": true,
        "autoTranscribe": true,
        "replyWithVoice": false
      },
      
      "inline": {
        "enabled": false,
        "cacheTime": 30,
        "isPersonal": true
      },
      
      "commands": {
        "/help": "顯示使用說明",
        "/status": "查看系統狀態",
        "/reset": "重置對話記憶",
        "/voice": "切換語音回覆",
        "/lang": "切換語言"
      },
      
      "resilience": {
        "pollingTimeout": 30,
        "retryOnError": true,
        "maxRetries": 10
      }
    }
  }
}
```

### 3.3 配置欄位詳解

| 欄位 | 類型 | 預設值 | 說明 |
|------|------|--------|------|
| `botToken` | string | — | BotFather 提供的 Token |
| `connectionMode` | string | `"polling"` | `"polling"` 或 `"webhook"` |
| `groupPolicy.privacyMode` | bool | `true` | 群組中是否只接收 @提及的訊息 |
| `formatting.parseMode` | string | `"MarkdownV2"` | 回覆的格式化模式 |
| `inline.enabled` | bool | `false` | 是否啟用 Inline 查詢功能 |

---

## 4. Webhook vs Long Polling

### 4.1 Long Polling 模式

Long Polling 是預設且最簡單的連接方式。Bot 主動向 Telegram 伺服器詢問是否有新訊息：

```
OpenClaw → Telegram API: "有新訊息嗎？"
           ← 等待最多 30 秒
           ← 有訊息時立即返回
OpenClaw 處理訊息
OpenClaw → Telegram API: "有新訊息嗎？"
           ← 繼續等待...
```

**優點**：不需要公開 IP 或域名，防火牆友好
**缺點**：有輕微延遲，不適合高流量

### 4.2 Webhook 模式

Webhook 模式下，Telegram 主動推送訊息到你指定的 URL：

```json
{
  "channels": {
    "telegram": {
      "connectionMode": "webhook",
      "webhook": {
        "url": "https://your-domain.com/webhook/telegram",
        "port": 8443,
        "secretToken": "your-secret-token"
      }
    }
  }
}
```

**優點**：即時收到訊息，適合高流量
**缺點**：需要 HTTPS 端點和公開可存取的 URL

### 4.3 選擇建議

| 場景 | 建議模式 |
|------|----------|
| 本地開發/測試 | Long Polling |
| 個人使用（1-3 人） | Long Polling |
| 團隊使用（10+ 人） | Webhook |
| 部署在雲端伺服器 | Webhook |
| 部署在家用電腦 | Long Polling |

---

## 5. 使用者白名單

### 5.1 allowFrom 配置

OpenClaw 預設不允許陌生人使用。你需要明確設定白名單：

```json
{
  "allowFrom": {
    "users": [
      "telegram:123456789"
    ]
  }
}
```

### 5.2 取得 Telegram User ID

方法一：使用 @userinfobot

```
1. 在 Telegram 搜尋 @userinfobot
2. 發送 /start
3. Bot 會回覆你的 User ID
```

方法二：轉發訊息到 @userinfobot

```
1. 從目標使用者轉發一則訊息到 @userinfobot
2. Bot 會回覆該使用者的 User ID
```

方法三：OpenClaw 的 debug 模式

```bash
# 啟動時啟用 debug 模式，會顯示所有收到訊息的 User ID
openclaw start --channel telegram --debug

# 輸出範例：
# [DEBUG] Received message from telegram:123456789 (John): "Hello"
# [DEBUG] User telegram:123456789 is not in allowFrom list, ignoring
```

### 5.3 動態白名單

OpenClaw 也支援動態白名單管理：

```json
{
  "allowFrom": {
    "users": ["telegram:123456789"],
    "inviteCode": {
      "enabled": true,
      "code": "join-easonclaw-2024",
      "maxUses": 10,
      "autoApprove": true
    }
  }
}
```

使用者只要發送邀請碼就能自動加入白名單。

---

## 6. 群組整合

### 6.1 Bot 加入群組

1. 在 BotFather 中確認 Bot 允許加入群組
2. 在群組中邀請 Bot
3. 將 Bot 設為管理員（如果需要讀取所有訊息）

### 6.2 群組觸發模式

```json
{
  "groupPolicy": {
    "triggerMode": "mention",
    "privacyMode": true
  }
}
```

Telegram 的 **Privacy Mode** 決定了 Bot 在群組中能看到哪些訊息：

- **Privacy Mode ON**：只能看到 @提及、命令 (`/`)、回覆 Bot 的訊息
- **Privacy Mode OFF**：可以看到群組中的所有訊息

### 6.3 群組權限設定

```json
{
  "groupPolicy": {
    "permissions": {
      "adminOnly": false,
      "allowedUsers": ["telegram:123456789"],
      "cooldownPerUser": 10,
      "maxResponseLength": 2000
    }
  }
}
```

### 6.4 頻道整合（Channel）

Telegram 的 Channel 與 Group 不同 — Channel 是單向廣播，OpenClaw 可以作為 Channel 的管理者發布內容：

```json
{
  "channels": {
    "telegram": {
      "channelPost": {
        "enabled": true,
        "channelId": "@my_channel",
        "allowScheduled": true
      }
    }
  }
}
```

---

## 7. 進階功能

### 7.1 Inline 查詢

Telegram 的 Inline 查詢允許使用者在任何對話中直接 @你的 Bot 進行查詢：

```
在任何對話框中輸入：
@EasonClawBot 今天天氣如何

Bot 會即時顯示回應供使用者選擇發送
```

```json
{
  "inline": {
    "enabled": true,
    "cacheTime": 30,
    "isPersonal": true,
    "switchPmText": "打開完整對話",
    "switchPmParameter": "inline_start"
  }
}
```

### 7.2 鍵盤按鈕

OpenClaw 可以在回覆中附帶互動按鈕：

```json
{
  "interactiveButtons": {
    "enabled": true,
    "style": "inline",
    "confirmActions": true
  }
}
```

按鈕類型：

- **Inline Keyboard**：附在訊息下方的按鈕
- **Reply Keyboard**：替換鍵盤的自定義按鈕
- **Callback Button**：按下後觸發特定動作

### 7.3 語音訊息

```
使用者發送語音訊息 → Telegram 傳遞 OGG/Opus 音訊
    │
    ▼
OpenClaw 下載 → 格式轉換 → STT 轉錄 → 處理 → 回覆
```

### 7.4 文件處理

Telegram 支援最大 2GB 的文件傳輸：

```json
{
  "media": {
    "acceptDocuments": true,
    "acceptPhotos": true,
    "acceptStickers": false,
    "maxProcessableSize": "50MB"
  }
}
```

---

## 8. Telegram 特色整合

### 8.1 Markdown 格式化

Telegram 原生支援 MarkdownV2 格式，OpenClaw 的回覆可以自動格式化：

```markdown
*粗體文字*
_斜體文字_
__底線文字__
~刪除線文字~
||隱藏文字（Spoiler）||
`行內程式碼`
```程式碼區塊```
[超連結](https://example.com)
```

### 8.2 長訊息處理

Telegram 訊息有 4096 字元的限制。OpenClaw 會自動處理長回覆：

```json
{
  "formatting": {
    "maxMessageLength": 4096,
    "splitStrategy": "paragraph",
    "splitDelay": 500
  }
}
```

分割策略：
- `"paragraph"`：在段落邊界處分割
- `"sentence"`：在句子邊界處分割
- `"hard"`：硬切在字元限制處

### 8.3 訊息編輯

OpenClaw 可以編輯已發送的訊息（用於串流回覆效果）：

```json
{
  "streaming": {
    "enabled": true,
    "editInterval": 1000,
    "showTypingIndicator": true
  }
}
```

---

## 9. 安全考量

- **botToken 安全**：Token 洩露等同帳號被盜，務必使用環境變數
- **白名單控管**：預設拒絕所有未知使用者
- **群組隱私模式**：建議開啟 Privacy Mode
- **Webhook 安全**：使用 secretToken 驗證來源
- **Rate Limiting**：防止濫用

```json
{
  "security": {
    "tokenFromEnv": "TELEGRAM_BOT_TOKEN",
    "rateLimiting": {
      "maxMessagesPerMinute": 20,
      "maxMessagesPerDay": 500
    },
    "blockList": [
      "telegram:000000000"
    ]
  }
}
```

---

## 10. 部署與維護

```bash
# 啟動 Telegram 通道
openclaw start --channel telegram

# 測試 Bot 是否正常運作
openclaw telegram test

# 查看 Bot 資訊
openclaw telegram info

# 設定 Webhook（如果使用 Webhook 模式）
openclaw telegram set-webhook --url https://your-domain.com/webhook/telegram

# 移除 Webhook（切換回 Polling）
openclaw telegram delete-webhook

# 查看最近的訊息日誌
openclaw logs --filter channel:telegram --tail 20
```

**維護建議**：

- 定期檢查 Bot 的回應品質
- 監控錯誤日誌
- 根據使用情況調整 Rate Limiting
- 定期更新 Bot 的命令選單和說明文字
- 備份對話記錄（如果需要）
