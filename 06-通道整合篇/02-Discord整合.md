# Discord 深度整合指南——從基礎到語音機器人

## 目錄

- [1. 引言：為什麼 Discord 是首選通道](#1-引言為什麼-discord-是首選通道)
- [2. Discord Bot 建立](#2-discord-bot-建立)
  - [2.1 前置需求](#21-前置需求)
  - [2.2 Developer Portal 完整步驟](#22-developer-portal-完整步驟)
  - [2.3 Bot 設定與功能開關](#23-bot-設定與功能開關)
  - [2.4 Privileged Gateway Intents](#24-privileged-gateway-intents)
- [3. Bot Token 安全管理](#3-bot-token-安全管理)
  - [3.1 Token 是什麼](#31-token-是什麼)
  - [3.2 安全存放方式](#32-安全存放方式)
  - [3.3 Token 洩漏的處理](#33-token-洩漏的處理)
  - [3.4 Token 的生命週期管理](#34-token-的生命週期管理)
- [4. OAuth2 URL 生成與權限設定](#4-oauth2-url-生成與權限設定)
  - [4.1 OAuth2 流程說明](#41-oauth2-流程說明)
  - [4.2 權限位元計算](#42-權限位元計算)
  - [4.3 邀請連結生成](#43-邀請連結生成)
  - [4.4 最小權限原則](#44-最小權限原則)
- [5. 開發者模式與 ID 蒐集](#5-開發者模式與-id-蒐集)
  - [5.1 啟用開發者模式](#51-啟用開發者模式)
  - [5.2 Guild ID（伺服器 ID）](#52-guild-id伺服器-id)
  - [5.3 Channel ID（頻道 ID）](#53-channel-id頻道-id)
  - [5.4 User ID（使用者 ID）](#54-user-id使用者-id)
  - [5.5 Role ID（角色 ID）](#55-role-id角色-id)
  - [5.6 使用 CLI 取得 ID](#56-使用-cli-取得-id)
- [6. OpenClaw Discord 配置](#6-openclaw-discord-配置)
  - [6.1 基礎配置](#61-基礎配置)
  - [6.2 完整配置結構](#62-完整配置結構)
  - [6.3 配置欄位詳解](#63-配置欄位詳解)
  - [6.4 環境變數引用](#64-環境變數引用)
- [7. DM 配對流程](#7-dm-配對流程)
  - [7.1 配對的必要性](#71-配對的必要性)
  - [7.2 配對方式](#72-配對方式)
  - [7.3 配對配置](#73-配對配置)
  - [7.4 取消配對](#74-取消配對)
- [8. 群組頻道配置](#8-群組頻道配置)
  - [8.1 頻道白名單](#81-頻道白名單)
  - [8.2 觸發模式](#82-觸發模式)
  - [8.3 討論串策略](#83-討論串策略)
  - [8.4 頻道特定行為](#84-頻道特定行為)
- [9. 語音頻道配置](#9-語音頻道配置)
  - [9.1 語音功能啟用](#91-語音功能啟用)
  - [9.2 語音配置詳解](#92-語音配置詳解)
  - [9.3 語音技術要求](#93-語音技術要求)
  - [9.4 語音品質調整](#94-語音品質調整)
- [10. Slash Commands 註冊](#10-slash-commands-註冊)
  - [10.1 什麼是 Slash Commands](#101-什麼是-slash-commands)
  - [10.2 內建 Slash Commands](#102-內建-slash-commands)
  - [10.3 自訂 Slash Commands](#103-自訂-slash-commands)
  - [10.4 Commands 同步](#104-commands-同步)
- [11. 進階：語音頻道整合的架構設計](#11-進階語音頻道整合的架構設計)
  - [11.1 語音處理管線](#111-語音處理管線)
  - [11.2 音訊編解碼](#112-音訊編解碼)
  - [11.3 延遲優化策略](#113-延遲優化策略)
  - [11.4 多人語音的技術挑戰](#114-多人語音的技術挑戰)
- [12. 進階：多人語音頻道中的禮貌插話機制設計](#12-進階多人語音頻道中的禮貌插話機制設計)
  - [12.1 問題定義](#121-問題定義)
  - [12.2 插話策略](#122-插話策略)
  - [12.3 話輪管理（Turn-Taking）](#123-話輪管理turn-taking)
  - [12.4 實作方案](#124-實作方案)
- [13. 進階：使用者識別與個性記憶](#13-進階使用者識別與個性記憶)
  - [13.1 多人環境下的身份識別](#131-多人環境下的身份識別)
  - [13.2 個人化互動](#132-個人化互動)
  - [13.3 群組動態記憶](#133-群組動態記憶)
- [14. 實戰：打造 Discord 社交友善機器人的完整藍圖](#14-實戰打造-discord-社交友善機器人的完整藍圖)
  - [14.1 藍圖概述](#141-藍圖概述)
  - [14.2 第一階段：基礎文字互動](#142-第一階段基礎文字互動)
  - [14.3 第二階段：語音整合](#143-第二階段語音整合)
  - [14.4 第三階段：社交增強](#144-第三階段社交增強)
  - [14.5 完整配置範例](#145-完整配置範例)
  - [14.6 部署與維護](#146-部署與維護)
- [15. 總結](#15-總結)

---

## 1. 引言：為什麼 Discord 是首選通道

在所有 OpenClaw 支援的通道中，Discord 具有獨特的地位。它不僅是一個文字通訊平台，更是一個完整的社交生態系統——文字頻道、語音頻道、視訊通話、螢幕分享、社群管理、Bot 生態⋯⋯ Discord 提供了最豐富的互動能力。

對於我們的 EasonClaw 專案來說，Discord 更是核心中的核心：

- **語音頻道**：Discord 是最成熟的即時語音平台之一，語音頻道讓 AI 能「開口說話」
- **Bot API 成熟度**：Discord 的 Bot API 經過多年發展，功能完善且文件齊全
- **社群生態**：Discord 的社群文化鼓勵互動，是 AI 社交機器人的理想棲息地
- **Slash Commands**：結構化的指令讓使用者能精確地與 AI 互動
- **多伺服器支援**：一個 Bot 可以加入多個伺服器，服務不同的社群
- **免費且開放**：不像某些平台需要商業帳號或付費 API

本章將從建立 Discord Bot 開始，逐步深入到語音整合、多人互動、個性化記憶，最終描繪一個完整的 Discord 社交友善機器人的藍圖。

---

## 2. Discord Bot 建立

### 2.1 前置需求

在開始之前，你需要：

1. 一個 Discord 帳號（[discord.com](https://discord.com) 註冊）
2. 至少一個 Discord 伺服器（你擁有管理權限的）
3. 穩定的網路連線
4. OpenClaw 已安裝且可運行

### 2.2 Developer Portal 完整步驟

**Step 1：進入 Discord Developer Portal**

開啟瀏覽器，前往 [https://discord.com/developers/applications](https://discord.com/developers/applications)。使用你的 Discord 帳號登入。

**Step 2：建立新應用程式**

1. 點擊右上角的 **「New Application」** 按鈕
2. 在彈出的對話框中輸入應用程式名稱（例如 `EasonClaw`）
3. 勾選同意 Developer Terms of Service 和 Developer Policy
4. 點擊 **「Create」**

你會被帶到應用程式的 General Information 頁面。這裡可以：

- 上傳應用程式圖示（建議 512x512 PNG）
- 填寫描述
- 記錄 **Application ID**（後續會用到）

**Step 3：建立 Bot 使用者**

1. 在左側選單中點擊 **「Bot」**
2. 點擊 **「Add Bot」** 按鈕
3. 確認對話框中的提示
4. Bot 建立完成！

**Step 4：取得 Bot Token**

在 Bot 頁面中：

1. 找到 **「TOKEN」** 區塊
2. 點擊 **「Reset Token」**（如果之前沒有記錄過）
3. 確認重置
4. **立即複製**顯示的 Token（它只會顯示一次！）
5. 安全地存放 Token（見第 3 節）

```
Bot Token 格式範例：
MTIzNDU2Nzg5MDEyMzQ1Njc4OQ.GaBcDe.AbCdEfGhIjKlMnOpQrStUvWxYz-0123456789
```

**Step 5：配置 Bot 設定**

在 Bot 頁面中，調整以下設定：

- **Username**：Bot 的顯示名稱
- **Icon**：Bot 的頭像
- **Public Bot**：
  - ✅ 開啟：任何人都能邀請你的 Bot
  - ❌ 關閉（推薦）：只有你能邀請 Bot
- **Requires OAuth2 Code Grant**：
  - ❌ 關閉（通常不需要）

### 2.3 Bot 設定與功能開關

在 Bot 頁面下方，有幾個重要的功能開關：

**Presence Intent**

```
□ PRESENCE INTENT
控制 Bot 是否接收使用者的線上/離線/遊戲狀態更新
```

- 啟用：Bot 能知道使用者是否在線（用於智慧通知）
- 停用（預設）：節省頻寬，如果不需要使用者狀態資訊

**Server Members Intent**

```
☑ SERVER MEMBERS INTENT
控制 Bot 是否接收伺服器成員的加入/離開/更新事件
```

- **需要啟用**：OpenClaw 需要知道誰在伺服器中才能正確管理權限和記憶

**Message Content Intent**

```
☑ MESSAGE CONTENT INTENT
控制 Bot 是否能讀取訊息的文字內容
```

- **必須啟用**：OpenClaw 需要讀取使用者的訊息內容才能回應。如果不啟用，Bot 只能看到空白的訊息內容（除了 Slash Commands 和 DM）。

> ⚠️ **重要**：如果你的 Bot 加入超過 100 個伺服器，這些 Privileged Intents 需要通過 Discord 的審核才能使用。

### 2.4 Privileged Gateway Intents

Discord 的 Gateway Intents 系統讓 Bot 只接收需要的事件，減少不必要的資料傳輸。

OpenClaw 需要的 Intents：

| Intent | 是否必要 | 用途 |
|--------|---------|------|
| `GUILDS` | ✅ 必要 | 接收伺服器相關事件 |
| `GUILD_MESSAGES` | ✅ 必要 | 接收伺服器中的訊息 |
| `DIRECT_MESSAGES` | ✅ 必要 | 接收私訊 |
| `MESSAGE_CONTENT` | ✅ 必要 | 讀取訊息內容 |
| `GUILD_MEMBERS` | ⬜ 建議 | 成員管理和識別 |
| `GUILD_VOICE_STATES` | ⬜ 語音需要 | 語音頻道事件 |
| `GUILD_PRESENCES` | ⬜ 選擇性 | 使用者在線狀態 |

在 OpenClaw 的配置中設定 Intents：

```json
{
  "channels": {
    "discord": {
      "gateway": {
        "intents": [
          "Guilds",
          "GuildMessages", 
          "DirectMessages",
          "MessageContent",
          "GuildMembers",
          "GuildVoiceStates"
        ]
      }
    }
  }
}
```

---

## 3. Bot Token 安全管理

### 3.1 Token 是什麼

Bot Token 是你的 Bot 在 Discord 上的「身份證+密碼」。擁有 Token 的人可以完全控制你的 Bot：發送訊息、加入/離開伺服器、讀取所有可訪問的訊息⋯⋯

**Token 的結構**：

```
MTIzNDU2Nzg5MDEyMzQ1Njc4OQ.GaBcDe.AbCdEfGhIjKlMnOpQrStUvWxYz-0123456789
│                             │       │
├── Base64 編碼的 User ID     │       │
│                             │       │
├── 時間戳                     │       │
│                             │       │
└── HMAC 簽名─────────────────┘───────┘
```

### 3.2 安全存放方式

**❌ 絕對不要做的事**：

```bash
# ❌ 不要硬編碼在配置文件中
{
  "discord": {
    "botToken": "MTIzNDU2Nzg5.GaBcDe.AbCdEf..."  // 超級危險！
  }
}

# ❌ 不要提交到 Git
git add openclaw.json  # 如果裡面有明文 Token

# ❌ 不要在公開場合分享
# 論壇、Discord 頻道、GitHub Issues 中都不要
```

**✅ 正確的做法**：

**方法一：環境變數（推薦）**

```bash
# 在 .env 文件中存放
echo "DISCORD_BOT_TOKEN=MTIzNDU2Nzg5.GaBcDe.AbCdEf..." >> ~/.openclaw/.env

# 在 openclaw.json 中引用
{
  "channels": {
    "discord": {
      "botToken": "${DISCORD_BOT_TOKEN}"
    }
  }
}
```

**方法二：系統金鑰管理工具**

```bash
# macOS Keychain
security add-generic-password -a "openclaw" -s "discord-bot-token" -w "你的Token"

# Linux secret-tool
secret-tool store --label="OpenClaw Discord Token" service openclaw key discord-token

# 然後在啟動腳本中讀取
export DISCORD_BOT_TOKEN=$(secret-tool lookup service openclaw key discord-token)
```

**方法三：加密的配置文件**

```bash
# 使用 OpenClaw 的加密配置功能
openclaw config encrypt --key DISCORD_BOT_TOKEN

# 系統會加密存放，運行時自動解密
```

### 3.3 Token 洩漏的處理

如果你的 Token 不小心洩漏了（例如意外提交到公開的 GitHub 倉庫），立即：

```bash
# Step 1：立即重置 Token
# 前往 Discord Developer Portal → Bot → Reset Token
# 這會立即使舊 Token 失效

# Step 2：更新所有使用舊 Token 的地方
echo "DISCORD_BOT_TOKEN=新的Token" > ~/.openclaw/.env

# Step 3：檢查是否有可疑活動
# 查看 Bot 的最近活動、加入的伺服器等

# Step 4：如果 Token 在 Git 歷史中
# 使用 git filter-branch 或 BFG Repo-Cleaner 清除
```

> 💡 **小提示**：GitHub 有自動偵測 Discord Bot Token 的功能。如果你不小心提交了 Token，Discord 可能會自動重置你的 Token 並通知你。

### 3.4 Token 的生命週期管理

- Token 不會過期（除非手動重置）
- 建議定期（例如每季度）輪換 Token
- 建立 Token 輪換的 SOP：更新 Token → 更新所有配置 → 重啟服務 → 驗證正常

---

## 4. OAuth2 URL 生成與權限設定

### 4.1 OAuth2 流程說明

要讓 Bot 加入一個 Discord 伺服器，你需要生成一個 OAuth2 邀請連結。使用者（伺服器管理員）點擊這個連結後，會看到權限確認頁面，確認後 Bot 就會加入伺服器。

### 4.2 權限位元計算

Discord 使用位元運算來表示權限。每個權限是一個特定的位元位：

| 權限 | 位元值 | 十六進位 | 說明 |
|------|--------|---------|------|
| View Channels | `1 << 10` | `0x400` | 查看頻道 |
| Send Messages | `1 << 11` | `0x800` | 發送訊息 |
| Send Messages in Threads | `1 << 38` | - | 在討論串中發送 |
| Embed Links | `1 << 14` | `0x4000` | 嵌入連結 |
| Attach Files | `1 << 15` | `0x8000` | 附加文件 |
| Read Message History | `1 << 16` | `0x10000` | 讀取歷史訊息 |
| Use Slash Commands | `1 << 31` | - | 使用斜線命令 |
| Connect (Voice) | `1 << 20` | `0x100000` | 加入語音頻道 |
| Speak (Voice) | `1 << 21` | `0x200000` | 在語音頻道說話 |
| Use Voice Activity | `1 << 25` | - | 語音活動偵測 |

### 4.3 邀請連結生成

**方法一：Developer Portal 生成（最簡單）**

1. 進入 Developer Portal → 你的應用 → **OAuth2** → **URL Generator**
2. 在 **Scopes** 中勾選：
   - `bot`
   - `applications.commands`（用於 Slash Commands）
3. 在 **Bot Permissions** 中勾選需要的權限
4. 複製底部生成的 URL

**方法二：手動構建**

```
https://discord.com/api/oauth2/authorize?client_id=YOUR_APPLICATION_ID&permissions=PERMISSION_INTEGER&scope=bot%20applications.commands
```

**OpenClaw 推薦的權限集合**：

```
基本互動權限（文字）：
  View Channels          (1 << 10)  =      1,024
  Send Messages          (1 << 11)  =      2,048
  Send Messages in Threads (1 << 38) = 274,877,906,944
  Embed Links            (1 << 14)  =     16,384
  Attach Files           (1 << 15)  =     32,768
  Read Message History   (1 << 16)  =     65,536
  Use Slash Commands     (1 << 31)  = 2,147,483,648

語音權限（如果需要語音功能）：
  Connect                (1 << 20)  =  1,048,576
  Speak                  (1 << 21)  =  2,097,152
  Use Voice Activity     (1 << 25)  = 33,554,432

計算總和後填入 permissions 參數。
```

使用 OpenClaw 自動生成：

```bash
# 只有文字功能
openclaw discord invite-url --text-only
# Output: https://discord.com/api/oauth2/authorize?client_id=...&permissions=2147534912&scope=bot%20applications.commands

# 文字 + 語音
openclaw discord invite-url --with-voice
# Output: https://discord.com/api/oauth2/authorize?client_id=...&permissions=2184234048&scope=bot%20applications.commands
```

### 4.4 最小權限原則

**只請求你實際需要的權限**。過多的權限會讓伺服器管理員不安，也增加了安全風險。

```
✅ 推薦：只請求必要權限
   - 大多數場景只需要：讀+寫訊息、斜線命令
   - 只在確實需要語音功能時才請求語音權限
   - 避免請求 Administrator 權限（管理者權限）

❌ 避免：
   - 不要請求 Administrator（管理者）權限
   - 不要請求 Manage Server（管理伺服器）
   - 不要請求 Ban Members（封禁成員）
   - 除非你的 Bot 確實需要這些功能
```

---

## 5. 開發者模式與 ID 蒐集

### 5.1 啟用開發者模式

Discord 的開發者模式讓你能在 UI 中直接複製各種 ID。啟用方式：

**桌面版**：
1. 點擊左下角的 ⚙️（設定）
2. 進入 **App Settings** → **Advanced**
3. 開啟 **Developer Mode**

**手機版**：
1. 點擊右下角的頭像（個人資料）
2. 向下滑到 **App Settings**
3. 點擊 **Advanced**
4. 開啟 **Developer Mode**

啟用後，在 Discord 中右鍵點擊各種元素時會出現「Copy ID」選項。

### 5.2 Guild ID（伺服器 ID）

Guild 是 Discord 中「伺服器」的內部名稱。

**取得方式**：

1. 右鍵點擊伺服器圖示
2. 選擇 **「Copy Server ID」**

```
範例：1234567890123456789
```

**用途**：在 `allowFrom.guilds` 中設定允許的伺服器。

### 5.3 Channel ID（頻道 ID）

**取得方式**：

1. 右鍵點擊頻道名稱
2. 選擇 **「Copy Channel ID」**

```
範例：9876543210987654321
```

**用途**：在 `allowFrom.channels` 或 `groupPolicy.allowedChannels` 中設定。

### 5.4 User ID（使用者 ID）

**取得方式**：

1. 右鍵點擊使用者名稱或頭像
2. 選擇 **「Copy User ID」**

```
範例：1111222233334444555
```

**用途**：在 `allowFrom.users` 中設定允許的使用者。

### 5.5 Role ID（角色 ID）

**取得方式**：

1. 進入伺服器設定 → Roles
2. 右鍵點擊角色
3. 選擇 **「Copy Role ID」**

或者在聊天中 @ 角色（加上反斜線 `\`）：
```
輸入：\@moderator
顯示：<@&1234567890>  ← 數字就是 Role ID
```

**用途**：在 `allowFrom.roles` 中設定允許的角色。

### 5.6 使用 CLI 取得 ID

```bash
# 列出 Bot 加入的伺服器
openclaw discord guilds

# 輸出：
# Guild ID             Name              Members
# 1234567890123456789  My Server         45
# 9876543210987654321  Test Server       12

# 列出伺服器的頻道
openclaw discord channels --guild 1234567890123456789

# 輸出：
# Channel ID            Name              Type
# 1111111111111111111  general           text
# 2222222222222222222  bot-commands      text
# 3333333333333333333  General Voice     voice
```

---

## 6. OpenClaw Discord 配置

### 6.1 基礎配置

最小可行的 Discord 配置：

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "botToken": "${DISCORD_BOT_TOKEN}",
      "allowFrom": {
        "users": ["discord:YOUR_USER_ID"]
      }
    }
  }
}
```

只需要三個設定：啟用 Discord、提供 Token、允許你自己使用。

### 6.2 完整配置結構

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "botToken": "${DISCORD_BOT_TOKEN}",
      "applicationId": "${DISCORD_APP_ID}",
      
      "allowFrom": {
        "users": [
          "discord:123456789012345678",
          "discord:234567890123456789"
        ],
        "guilds": [
          "guild:345678901234567890"
        ],
        "channels": [
          "channel:456789012345678901"
        ],
        "roles": [
          "role:567890123456789012"
        ]
      },
      
      "dmPolicy": {
        "enabled": true,
        "allowUnknownUsers": false,
        "requirePairing": false,
        "greeting": "你好！我是 EasonClaw，有什麼我能幫你的嗎？",
        "rateLimitPerUser": {
          "messages": 60,
          "period": "1m"
        }
      },
      
      "groupPolicy": {
        "enabled": true,
        "triggerMode": "mention",
        "triggerPrefix": "!claw",
        "respondInThread": true,
        "allowedChannels": [
          "channel:111222333444555666"
        ],
        "ignoredChannels": [
          "channel:666555444333222111"
        ],
        "cooldown": 3
      },
      
      "voice": {
        "enabled": true,
        "autoJoin": false,
        "allowedChannels": [
          "channel:777888999000111222"
        ],
        "stt": {
          "provider": "whisper",
          "language": "zh"
        },
        "tts": {
          "provider": "elevenlabs",
          "voice": "custom-voice-id"
        },
        "vadSensitivity": 0.6,
        "silenceTimeout": 1500,
        "bargeIn": true
      },
      
      "slashCommands": {
        "enabled": true,
        "syncOnStartup": true,
        "guildSpecific": true,
        "guildIds": ["345678901234567890"]
      },
      
      "presence": {
        "status": "online",
        "activity": {
          "type": "LISTENING",
          "name": "你的訊息"
        }
      },
      
      "gateway": {
        "intents": [
          "Guilds",
          "GuildMessages",
          "DirectMessages",
          "MessageContent",
          "GuildMembers",
          "GuildVoiceStates"
        ]
      },
      
      "resilience": {
        "reconnect": {
          "enabled": true,
          "maxRetries": 10,
          "baseDelay": 1000,
          "maxDelay": 30000
        }
      }
    }
  }
}
```

### 6.3 配置欄位詳解

**基本欄位**：

| 欄位 | 類型 | 必要 | 說明 |
|------|------|------|------|
| `enabled` | bool | ✅ | 是否啟用 Discord 通道 |
| `botToken` | string | ✅ | Bot Token（建議用環境變數） |
| `applicationId` | string | ⬜ | Application ID（Slash Commands 需要） |

**allowFrom 欄位**：

| 欄位 | 類型 | 說明 |
|------|------|------|
| `users` | string[] | 允許的使用者 ID，格式：`discord:ID` |
| `guilds` | string[] | 允許的伺服器 ID，格式：`guild:ID` |
| `channels` | string[] | 允許的頻道 ID，格式：`channel:ID` |
| `roles` | string[] | 允許的角色 ID，格式：`role:ID` |

權限判斷為 OR 邏輯——只要符合任一條件就允許。

**voice 欄位**：

| 欄位 | 類型 | 預設 | 說明 |
|------|------|------|------|
| `enabled` | bool | `false` | 啟用語音功能 |
| `autoJoin` | bool | `false` | 是否自動加入語音頻道 |
| `allowedChannels` | string[] | `[]` | 允許加入的語音頻道（空=全部） |
| `stt.provider` | string | `whisper` | 語音轉文字供應商 |
| `tts.provider` | string | `openai-tts` | 文字轉語音供應商 |
| `vadSensitivity` | number | `0.5` | 語音活動偵測靈敏度（0-1） |
| `silenceTimeout` | number | `2000` | 靜音超時（毫秒）——多久沒聲音算說完 |
| `bargeIn` | bool | `true` | 是否允許使用者打斷 AI 說話 |

### 6.4 環境變數引用

OpenClaw 的配置文件支援環境變數引用：

```json
{
  "botToken": "${DISCORD_BOT_TOKEN}",
  "applicationId": "${DISCORD_APP_ID}"
}
```

系統在啟動時會自動替換 `${...}` 為對應的環境變數值。

---

## 7. DM 配對流程

### 7.1 配對的必要性

在開放的 Discord 環境中，任何人都可能發送私訊給你的 Bot。如果不加限制，Bot 可能：
- 被陌生人濫用
- 產生大量 API 費用
- 接觸到不當內容

DM 配對機制解決了這個問題——只有通過配對的使用者才能與 Bot 私訊。

### 7.2 配對方式

**方式一：白名單模式（最簡單）**

直接在 `allowFrom.users` 中列出允許的使用者 ID：

```json
{
  "allowFrom": {
    "users": [
      "discord:123456789",
      "discord:987654321"
    ]
  }
}
```

**方式二：配對碼模式**

1. 管理員通過 CLI 生成配對碼：
```bash
openclaw discord pair generate
# Output: Pairing code: CLAW-A1B2-C3D4 (expires in 10 minutes)
```

2. 使用者在 DM 中發送配對碼給 Bot：
```
使用者: CLAW-A1B2-C3D4
Bot: 配對成功！你好 Eason，我是 EasonClaw。有什麼我能幫你的嗎？
```

**方式三：伺服器成員自動配對**

如果使用者在允許的伺服器中，自動允許 DM：

```json
{
  "dmPolicy": {
    "autoApproveFromGuild": true
  }
}
```

### 7.3 配對配置

```json
{
  "dmPolicy": {
    "enabled": true,
    "requirePairing": true,
    "pairingMethod": "code",
    "pairingCodeExpiry": 600,
    "maxPairedUsers": 50,
    "unpairInactiveAfter": "30d",
    "greeting": "你好！我是 EasonClaw。要開始使用，請輸入你的配對碼。"
  }
}
```

### 7.4 取消配對

```bash
# 查看已配對的使用者
openclaw discord pair list

# 取消配對
openclaw discord pair remove discord:123456789

# 清除所有配對
openclaw discord pair clear --confirm
```

---

## 8. 群組頻道配置

### 8.1 頻道白名單

限制 Bot 只在特定頻道中回應：

```json
{
  "groupPolicy": {
    "allowedChannels": [
      "channel:111222333",
      "channel:444555666"
    ],
    "ignoredChannels": [
      "channel:777888999"
    ]
  }
}
```

- `allowedChannels`：如果設定，Bot **只在**這些頻道中回應
- `ignoredChannels`：Bot **不在**這些頻道中回應（即使在 allowedChannels 中）

### 8.2 觸發模式

```json
{
  "groupPolicy": {
    "triggerMode": "mention"
  }
}
```

三種觸發模式：

| 模式 | 說明 | 範例 |
|------|------|------|
| `mention` | 被 @ 提及時回應 | `@EasonClaw 今天天氣如何？` |
| `prefix` | 特定前綴觸發 | `!claw 今天天氣如何？` |
| `always` | 回應所有訊息 | `今天天氣如何？`（Bot 直接回應） |

**建議**：在群組中使用 `mention` 模式。`always` 模式會讓 Bot 回應每一條訊息，非常吵雜。

### 8.3 討論串策略

```json
{
  "groupPolicy": {
    "respondInThread": true,
    "threadAutoArchive": 60,
    "threadNameFormat": "{user} 的對話"
  }
}
```

啟用 `respondInThread` 後，Bot 會在觸發訊息下建立一個討論串，並在串中回應。這避免了在主頻道刷頻，保持頻道整潔。

### 8.4 頻道特定行為

你可以為不同頻道設定不同的行為：

```json
{
  "groupPolicy": {
    "channelOverrides": {
      "channel:111222333": {
        "triggerMode": "always",
        "personality": "casual"
      },
      "channel:444555666": {
        "triggerMode": "prefix",
        "triggerPrefix": "/ask",
        "personality": "professional"
      }
    }
  }
}
```

---

## 9. 語音頻道配置

### 9.1 語音功能啟用

語音是 Discord 整合中最令人興奮的功能。讓 AI 能在語音頻道中「聽」和「說」。

```json
{
  "voice": {
    "enabled": true
  }
}
```

啟用語音後，Bot 可以：
- 加入語音頻道
- 聽取使用者的語音並轉為文字（STT）
- 用語音回應使用者（TTS）
- 支援即時對話（Talk Mode）

### 9.2 語音配置詳解

```json
{
  "voice": {
    "enabled": true,
    "autoJoin": false,
    "autoLeaveOnEmpty": true,
    "autoLeaveTimeout": 300,
    
    "allowedChannels": [],
    
    "stt": {
      "provider": "whisper",
      "model": "whisper-1",
      "language": "zh",
      "prompt": "這是一段繁體中文的對話。"
    },
    
    "tts": {
      "provider": "elevenlabs",
      "voiceId": "your-voice-id",
      "model": "eleven_multilingual_v2",
      "stability": 0.5,
      "similarityBoost": 0.75,
      "speed": 1.0
    },
    
    "vad": {
      "sensitivity": 0.6,
      "silenceThreshold": 1500,
      "minSpeechDuration": 300
    },
    
    "bargeIn": {
      "enabled": true,
      "sensitivity": 0.7
    },
    
    "multiUser": {
      "enabled": true,
      "speakerIdentification": true,
      "maxConcurrentSpeakers": 5,
      "politeTurnTaking": true
    }
  }
}
```

### 9.3 語音技術要求

語音功能需要額外的系統依賴：

```bash
# 必要的系統工具
- ffmpeg           # 音訊編解碼
- opus-tools       # Opus 編碼（Discord 使用 Opus）

# 安裝（Ubuntu/Debian）
sudo apt-get install ffmpeg libopus0 libopus-dev

# 安裝（macOS）
brew install ffmpeg opus

# Python 依賴
pip3 install discord.py[voice] PyNaCl
```

### 9.4 語音品質調整

```json
{
  "voice": {
    "audio": {
      "sampleRate": 48000,
      "channels": 2,
      "bitrate": 64000,
      "frameSize": 960,
      "noiseReduction": true,
      "echoCancellation": true,
      "autoGainControl": true
    }
  }
}
```

---

## 10. Slash Commands 註冊

### 10.1 什麼是 Slash Commands

Slash Commands 是 Discord 的結構化指令系統。使用者輸入 `/` 時，Discord 會顯示可用的命令列表，包括命令描述和參數提示。

```
/ask question:今天天氣如何？
/remind time:明天下午3點 message:開會
/voice join channel:General Voice
```

### 10.2 內建 Slash Commands

OpenClaw 自動註冊以下 Slash Commands：

| 命令 | 說明 |
|------|------|
| `/ask` | 向 AI 提問 |
| `/chat` | 開始/繼續對話 |
| `/voice join` | 讓 Bot 加入語音頻道 |
| `/voice leave` | 讓 Bot 離開語音頻道 |
| `/voice mute` | 靜音 Bot |
| `/settings` | 調整個人設定 |
| `/memory` | 管理 AI 的記憶 |
| `/help` | 顯示幫助資訊 |
| `/ping` | 檢查 Bot 延遲 |

### 10.3 自訂 Slash Commands

你可以通過技能定義自訂的 Slash Commands：

```json
{
  "slashCommands": {
    "custom": [
      {
        "name": "weather",
        "description": "查詢天氣",
        "options": [
          {
            "name": "city",
            "description": "城市名稱",
            "type": "STRING",
            "required": true
          },
          {
            "name": "days",
            "description": "預報天數",
            "type": "INTEGER",
            "required": false,
            "choices": [
              { "name": "今天", "value": 1 },
              { "name": "3天", "value": 3 },
              { "name": "7天", "value": 7 }
            ]
          }
        ]
      }
    ]
  }
}
```

### 10.4 Commands 同步

Slash Commands 需要與 Discord 同步才能使用：

```bash
# 同步到全域（所有伺服器，可能需要 1 小時生效）
openclaw discord commands sync --global

# 同步到特定伺服器（立即生效，推薦開發時使用）
openclaw discord commands sync --guild 1234567890

# 查看已註冊的命令
openclaw discord commands list

# 清除所有命令
openclaw discord commands clear
```

---

## 11. 進階：語音頻道整合的架構設計

### 11.1 語音處理管線

Discord 語音頻道的整合涉及一個完整的音訊處理管線：

```
使用者語音 → Discord Voice Gateway → Opus 解碼 → PCM 音訊
    │
    ▼
VAD（語音活動偵測）── 非語音 ──→ 忽略
    │
    └─── 語音 ──→ 音訊緩衝
                    │
                    ▼ （靜音超時）
                 STT 轉錄
                    │
                    ▼
              OpenClaw Core
              （ReAct 推理）
                    │
                    ▼
              生成文字回應
                    │
                    ▼
              TTS 語音合成
                    │
                    ▼
              Opus 編碼
                    │
                    ▼
        Discord Voice Gateway → 使用者耳機
```

### 11.2 音訊編解碼

Discord 使用 **Opus** 編碼傳輸語音。整個編解碼流程：

```
接收端：
Opus (48kHz, Stereo) → 解碼 → PCM (16-bit, 48kHz) → 降採樣 → PCM (16kHz, Mono)
                                                                       │
                                                                       ▼
                                                                    STT 引擎

發送端：
TTS 輸出 → PCM (24kHz) → 升採樣 → PCM (48kHz) → Opus 編碼 → Discord
```

### 11.3 延遲優化策略

語音互動中，延遲（Latency）是用戶體驗的關鍵。目標是將整體延遲控制在 2 秒以內：

```
延遲預算分配：

VAD + 靜音偵測：  ~200ms  ← 不可避免，需要等使用者說完
STT 轉錄：       ~500ms  ← 使用 streaming STT 可降低
ReAct 推理：     ~800ms  ← 取決於 LLM 和任務複雜度
TTS 合成：       ~300ms  ← 使用 streaming TTS 可降低
網路傳輸：       ~200ms  ← 取決於網路品質
────────────────────────
總計：           ~2000ms
```

**優化技巧**：

1. **Streaming STT**：不等整句話說完就開始轉錄
2. **Streaming TTS**：不等整段回應生成就開始合成語音
3. **LLM Streaming**：使用 streaming API，Token 一生成就開始 TTS
4. **預熱**：保持 STT/TTS 連線不斷開
5. **本地 STT**：使用本地 Whisper 而非 API，省去網路延遲
6. **短回應優先**：語音回應應該比文字回應更簡短

### 11.4 多人語音的技術挑戰

在多人語音頻道中，額外的挑戰包括：

**說話者識別（Speaker Identification）**：

Discord 的語音 Gateway 會標記每個音訊封包的來源使用者 ID。OpenClaw 利用這個資訊來區分不同的說話者：

```
Audio Packet → Discord User ID → OpenClaw User Identity → 個人化回應
```

**混音問題**：

當多人同時說話時，需要決定處理策略：

```
策略一：只處理最先開始說話的人
策略二：嘗試分離不同說話者的音訊（Speaker Separation）
策略三：等所有人都停止說話後再處理
```

OpenClaw 預設使用策略一，因為 Speaker Separation 在即時場景中計算成本過高。

---

## 12. 進階：多人語音頻道中的禮貌插話機制設計

### 12.1 問題定義

在多人語音頻道中，AI 面臨一個獨特的社交挑戰：**什麼時候該說話？**

不像文字通道中 AI 只需回應 @提及，在語音頻道中 AI 需要像一個真正的對話參與者一樣——知道何時該聽、何時該說、何時不該打斷。

### 12.2 插話策略

OpenClaw 設計了一套基於社交禮儀的插話策略：

**等級一：被點名時回應（Addressed）**

```
使用者A: "EasonClaw，你覺得呢？"
→ AI 立即回應
```

**等級二：問題導向回應（Question-Directed）**

```
使用者A: "有人知道台北今天天氣嗎？"
→ AI 判斷：這是一個 AI 能回答的問題
→ 等待 2-3 秒（讓人類有機會先回答）
→ 如果沒人回答，AI 回應
```

**等級三：補充資訊（Supplementary）**

```
使用者A: "我覺得那家餐廳不錯"
使用者B: "是啊，他們的牛排很好吃"
→ AI 判斷：可以補充相關資訊
→ 等待對話自然停頓
→ "那家餐廳的甜點也很推薦喔"
```

**等級四：靜默觀察（Silent Observation）**

```
使用者A: "最近工作好累"
使用者B: "我也是，加油啊"
→ AI 判斷：這是人際間的情感交流
→ 不插話（除非被問到）
```

### 12.3 話輪管理（Turn-Taking）

```json
{
  "voice": {
    "turnTaking": {
      "model": "polite",
      "addressedResponseDelay": 200,
      "unaddressedResponseDelay": 3000,
      "maxSilenceBeforeResponse": 5000,
      "interruptionThreshold": 0.8,
      "personalityWeight": {
        "talkative": 0.3,
        "reserved": 0.7
      }
    }
  }
}
```

話輪管理的決策流程：

```
音訊輸入
    │
    ├── 包含 Bot 名字？
    │   └── 是 → 等級一：立即回應
    │
    ├── 是問句？
    │   ├── AI 能回答？
    │   │   └── 是 → 等級二：延遲 2-3 秒後回應
    │   └── 否 → 繼續監聽
    │
    ├── 長時間靜默（>5秒）？
    │   └── 是 → 考慮主動發言（如果有相關話題）
    │
    └── 其他 → 繼續監聽，更新上下文
```

### 12.4 實作方案

```python
class PoliteTurnManager:
    """禮貌的話輪管理器"""
    
    def __init__(self, config):
        self.addressed_delay = config.get("addressedResponseDelay", 200)
        self.unaddressed_delay = config.get("unaddressedResponseDelay", 3000)
        self.last_speech_time = {}
        self.conversation_context = []
    
    async def should_respond(self, transcript, speaker_id, is_addressed):
        """判斷是否應該回應"""
        
        if is_addressed:
            # 被直接提到，短暫延遲後回應
            await asyncio.sleep(self.addressed_delay / 1000)
            return True
        
        if self._is_question(transcript) and self._can_answer(transcript):
            # 是問句且 AI 能回答，等待看是否有人先回答
            await asyncio.sleep(self.unaddressed_delay / 1000)
            if not self._someone_answered_since(self.unaddressed_delay):
                return True
        
        return False
    
    def _is_question(self, text):
        question_markers = ["嗎", "呢", "?", "？", "什麼", "怎麼", "哪裡", "多少"]
        return any(m in text for m in question_markers)
    
    def _can_answer(self, text):
        # 使用 LLM 判斷是否在 AI 的能力範圍內
        pass
    
    def _someone_answered_since(self, delay_ms):
        # 檢查在等待期間是否有其他人回應
        pass
```

---

## 13. 進階：使用者識別與個性記憶

### 13.1 多人環境下的身份識別

在 Discord 群組和語音頻道中，AI 需要識別和記住每個使用者。OpenClaw 利用 Discord 的使用者系統實現這一點：

```
Discord 使用者 → Discord User ID → OpenClaw 身份映射 → 個人記憶和偏好
```

每個使用者都有獨立的：

- **記憶存儲**：AI 對這個人的認識和記憶
- **偏好設定**：語言、語氣、互動風格
- **對話歷史**：與 AI 的歷史互動記錄
- **關係模型**：AI 與這個人的「關係」定義

### 13.2 個人化互動

```json
{
  "memory": {
    "perUser": {
      "discord:123456789": {
        "name": "Eason",
        "language": "zh-TW",
        "interests": ["程式設計", "咖啡", "攝影"],
        "conversationStyle": "casual",
        "lastInteraction": "2024-03-15T10:30:00Z"
      }
    }
  }
}
```

在群組對話中，AI 能根據個人化記憶提供不同的互動方式：

```
使用者A（技術背景）: 「這個 API 怎麼用？」
AI: 「你可以用 curl 測試：curl -X GET https://api.example.com/v2/...」

使用者B（非技術背景）: 「這個 API 怎麼用？」
AI: 「簡單來說，API 就像是一個服務窗口，你可以通過它來...
      我可以幫你設定一個簡單的範例。」
```

### 13.3 群組動態記憶

除了個人記憶外，OpenClaw 還維護群組層級的動態記憶：

```json
{
  "groupMemory": {
    "guild:345678901234567890": {
      "topics": ["OpenClaw 開發", "AI 技術"],
      "culture": "技術導向、友善、互助",
      "inside_jokes": ["那個 bug 又來了"],
      "active_members": ["Eason", "Alice", "Bob"],
      "recent_discussions": [...]
    }
  }
}
```

這讓 AI 能理解群組的文化和氛圍，做出更適當的回應。

---

## 14. 實戰：打造 Discord 社交友善機器人的完整藍圖

### 14.1 藍圖概述

現在讓我們把所有知識整合起來，設計一個完整的 Discord 社交友善機器人——EasonClaw。

**目標**：

- 在文字頻道中提供智慧互動
- 在語音頻道中進行即時對話
- 記住每個使用者的偏好和歷史
- 禮貌地參與多人對話
- 提供實用工具（天氣、翻譯、提醒等）

**分階段實施**：

| 階段 | 功能 | 預計時間 |
|------|------|---------|
| 第一階段 | 基礎文字互動 | 1-2 天 |
| 第二階段 | 語音整合 | 3-5 天 |
| 第三階段 | 社交增強 | 1-2 週 |

### 14.2 第一階段：基礎文字互動

**目標**：讓 EasonClaw 能在 Discord 中進行基本的文字對話。

**步驟**：

1. 建立 Discord Bot（完成第 2 節的步驟）
2. 設定基礎配置
3. 設定允許的使用者和伺服器
4. 測試基本互動

**基礎配置**：

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "botToken": "${DISCORD_BOT_TOKEN}",
      "applicationId": "${DISCORD_APP_ID}",
      "allowFrom": {
        "users": ["discord:YOUR_USER_ID"],
        "guilds": ["guild:YOUR_GUILD_ID"]
      },
      "dmPolicy": {
        "enabled": true,
        "greeting": "嗨！我是 EasonClaw 🐾 有什麼我能幫你的嗎？"
      },
      "groupPolicy": {
        "enabled": true,
        "triggerMode": "mention",
        "respondInThread": true
      },
      "slashCommands": {
        "enabled": true,
        "syncOnStartup": true,
        "guildSpecific": true,
        "guildIds": ["YOUR_GUILD_ID"]
      }
    }
  }
}
```

**驗證步驟**：

```bash
# 啟動 OpenClaw
openclaw start

# 檢查 Discord 連線狀態
openclaw channels status

# 在 Discord 中測試
# 1. 私訊 Bot → 應收到問候語
# 2. 在群組 @Bot → 應在討論串中回應
# 3. 使用 /ask 命令 → 應正常回應
```

### 14.3 第二階段：語音整合

**目標**：讓 EasonClaw 能在語音頻道中聽和說。

**前置準備**：

```bash
# 安裝語音依賴
sudo apt-get install ffmpeg libopus0

# 設定 TTS 供應商（ElevenLabs）
echo "ELEVENLABS_API_KEY=your-key" >> ~/.openclaw/.env
```

**語音配置**：

```json
{
  "voice": {
    "enabled": true,
    "autoJoin": false,
    "autoLeaveOnEmpty": true,
    "autoLeaveTimeout": 300,
    "stt": {
      "provider": "whisper",
      "language": "zh"
    },
    "tts": {
      "provider": "elevenlabs",
      "voiceId": "your-custom-voice-id",
      "model": "eleven_multilingual_v2"
    },
    "vad": {
      "sensitivity": 0.6,
      "silenceThreshold": 1500
    },
    "bargeIn": {
      "enabled": true
    }
  }
}
```

**驗證步驟**：

```bash
# 在 Discord 中使用 Slash Command 讓 Bot 加入語音頻道
/voice join

# 對著麥克風說話
# Bot 應該能聽懂並用語音回應

# 讓 Bot 離開
/voice leave
```

### 14.4 第三階段：社交增強

**目標**：讓 EasonClaw 成為一個真正的社交夥伴。

**增強功能**：

1. **個性化記憶**：記住每個人的名字、偏好和互動歷史
2. **禮貌插話**：在多人對話中適時參與
3. **情境感知**：根據對話主題和氛圍調整行為
4. **主動關心**：在合適的時機問候或提供幫助

**社交增強配置**：

```json
{
  "personality": {
    "base": "friendly-companion",
    "traits": {
      "warmth": 0.8,
      "humor": 0.6,
      "helpfulness": 0.9,
      "proactivity": 0.4
    }
  },
  "socialBehavior": {
    "rememberNames": true,
    "useNicknames": true,
    "celebrateMilestones": true,
    "sendGreetings": {
      "onJoinVoice": true,
      "message": "{name}，歡迎回來！"
    }
  }
}
```

### 14.5 完整配置範例

以下是 EasonClaw Discord 整合的完整配置：

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "botToken": "${DISCORD_BOT_TOKEN}",
      "applicationId": "${DISCORD_APP_ID}",
      
      "allowFrom": {
        "users": ["discord:YOUR_ID"],
        "guilds": ["guild:YOUR_GUILD_ID"]
      },
      
      "dmPolicy": {
        "enabled": true,
        "allowUnknownUsers": false,
        "greeting": "嗨！我是 EasonClaw 🐾"
      },
      
      "groupPolicy": {
        "enabled": true,
        "triggerMode": "mention",
        "respondInThread": true,
        "cooldown": 3,
        "channelOverrides": {
          "channel:BOT_CHANNEL_ID": {
            "triggerMode": "always"
          }
        }
      },
      
      "voice": {
        "enabled": true,
        "autoJoin": false,
        "autoLeaveOnEmpty": true,
        "autoLeaveTimeout": 300,
        "stt": {
          "provider": "whisper",
          "language": "zh"
        },
        "tts": {
          "provider": "elevenlabs",
          "voiceId": "YOUR_VOICE_ID",
          "model": "eleven_multilingual_v2"
        },
        "vad": {
          "sensitivity": 0.6,
          "silenceThreshold": 1500
        },
        "bargeIn": {
          "enabled": true
        },
        "multiUser": {
          "enabled": true,
          "speakerIdentification": true,
          "politeTurnTaking": true
        }
      },
      
      "slashCommands": {
        "enabled": true,
        "syncOnStartup": true,
        "guildSpecific": true,
        "guildIds": ["YOUR_GUILD_ID"]
      },
      
      "presence": {
        "status": "online",
        "activity": {
          "type": "LISTENING",
          "name": "你的聲音 🎧"
        }
      },
      
      "gateway": {
        "intents": [
          "Guilds",
          "GuildMessages",
          "DirectMessages",
          "MessageContent",
          "GuildMembers",
          "GuildVoiceStates"
        ]
      }
    }
  }
}
```

### 14.6 部署與維護

**部署建議**：

```bash
# 使用 systemd 保持 OpenClaw 持續運行
sudo cat > /etc/systemd/system/openclaw.service << 'EOF'
[Unit]
Description=OpenClaw AI Assistant
After=network.target

[Service]
Type=simple
User=openclaw
WorkingDirectory=/home/openclaw
ExecStart=/usr/local/bin/openclaw start
Restart=always
RestartSec=10
EnvironmentFile=/home/openclaw/.openclaw/.env

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable openclaw
sudo systemctl start openclaw
```

**監控**：

```bash
# 查看狀態
sudo systemctl status openclaw

# 查看日誌
openclaw logs --follow --filter channel:discord

# 查看指標
openclaw metrics --channel discord
```

**維護清單**：

- [ ] 每週檢查 Bot 在線狀態
- [ ] 每月檢查 Discord API 更新
- [ ] 定期更新 OpenClaw 版本
- [ ] 監控 API 用量（LLM、TTS 等）
- [ ] 備份記憶和配置

---

## 15. 總結

Discord 是 OpenClaw 生態系統中最強大、最靈活的通道。通過本章，我們完整走過了：

1. **基礎設置**：從建立 Bot 到安全管理 Token
2. **權限管理**：OAuth2 邀請和最小權限原則
3. **ID 蒐集**：開發者模式和 CLI 工具
4. **完整配置**：DM、群組、語音、Slash Commands
5. **語音架構**：從音訊管線到延遲優化
6. **社交設計**：禮貌插話、使用者識別、個性記憶
7. **實戰藍圖**：三階段打造社交友善機器人

Discord 整合不是一次性的工作，而是一個持續演進的過程。隨著你對使用者需求的理解越來越深，你的 Bot 也會越來越smart、越來越有「人味」。

> **下一步**：配置好 Discord 後，你可能還想整合其他通道。下一章我們將介紹 WhatsApp 的整合指南。
