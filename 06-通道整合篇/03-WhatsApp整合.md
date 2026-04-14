# WhatsApp 整合指南

## 目錄

- [1. WhatsApp 整合概述](#1-whatsapp-整合概述)
  - [1.1 為什麼選擇 WhatsApp](#11-為什麼選擇-whatsapp)
  - [1.2 WhatsApp Business API vs WhatsApp Web](#12-whatsapp-business-api-vs-whatsapp-web)
  - [1.3 限制與注意事項](#13-限制與注意事項)
- [2. WhatsApp Business API 配置](#2-whatsapp-business-api-配置)
  - [2.1 Meta Business Suite 設定](#21-meta-business-suite-設定)
  - [2.2 建立 WhatsApp Business 帳號](#22-建立-whatsapp-business-帳號)
  - [2.3 取得 API Token](#23-取得-api-token)
  - [2.4 Webhook 配置](#24-webhook-配置)
  - [2.5 電話號碼驗證](#25-電話號碼驗證)
- [3. QR Code 配對（WhatsApp Web 方式）](#3-qr-code-配對whatsapp-web-方式)
  - [3.1 使用 Baileys 函式庫](#31-使用-baileys-函式庫)
  - [3.2 QR Code 配對流程](#32-qr-code-配對流程)
  - [3.3 Session 持久化](#33-session-持久化)
  - [3.4 多裝置支援](#34-多裝置支援)
- [4. OpenClaw WhatsApp 配置](#4-openclaw-whatsapp-配置)
  - [4.1 基礎配置](#41-基礎配置)
  - [4.2 完整配置範例](#42-完整配置範例)
  - [4.3 配置欄位詳解](#43-配置欄位詳解)
- [5. 群組設定](#5-群組設定)
  - [5.1 群組觸發模式](#51-群組觸發模式)
  - [5.2 群組管理功能](#52-群組管理功能)
  - [5.3 群組隱私考量](#53-群組隱私考量)
- [6. 語音訊息支援](#6-語音訊息支援)
  - [6.1 接收語音訊息](#61-接收語音訊息)
  - [6.2 發送語音回覆](#62-發送語音回覆)
  - [6.3 語音處理管線](#63-語音處理管線)
- [7. 媒體處理](#7-媒體處理)
  - [7.1 圖片處理](#71-圖片處理)
  - [7.2 文件處理](#72-文件處理)
  - [7.3 位置分享](#73-位置分享)
- [8. 安全與隱私](#8-安全與隱私)
  - [8.1 端到端加密](#81-端到端加密)
  - [8.2 資料存儲考量](#82-資料存儲考量)
  - [8.3 合規要求](#83-合規要求)
- [9. 部署與維護](#9-部署與維護)
- [10. 常見問題](#10-常見問題)

---

## 1. WhatsApp 整合概述

### 1.1 為什麼選擇 WhatsApp

WhatsApp 是全球使用者最多的即時通訊軟體，在亞洲、歐洲和拉丁美洲尤其普及。將 OpenClaw 整合到 WhatsApp 有幾個獨特的優勢：

- **最大觸及率**：全球超過 20 億使用者
- **端到端加密**：訊息安全性最高
- **語音訊息原生支援**：使用者習慣發語音訊息
- **跨平台**：iOS、Android、Web 全覆蓋
- **簡訊替代**：在許多國家，WhatsApp 已經取代了簡訊

### 1.2 WhatsApp Business API vs WhatsApp Web

OpenClaw 支援兩種方式整合 WhatsApp：

| 特性 | Business API（Cloud API） | WhatsApp Web（Baileys） |
|------|--------------------------|----------------------|
| 官方支援 | ✅ Meta 官方 | ❌ 非官方 |
| 穩定性 | 高 | 中等 |
| 費用 | 按訊息計費 | 免費 |
| 設定複雜度 | 高（需要商業帳號） | 低（掃 QR Code） |
| 封號風險 | 極低 | 有風險 |
| 功能限制 | 需要模板訊息 | 較少限制 |
| 適合場景 | 正式商業用途 | 個人/實驗用途 |

**建議**：

- 個人使用或實驗：使用 WhatsApp Web 方式（Baileys）
- 正式商業部署：使用 WhatsApp Business API

### 1.3 限制與注意事項

- **WhatsApp 的 24 小時視窗規則**：Business API 模式下，Bot 只能在使用者最後一次傳訊後的 24 小時內免費回應
- **訊息模板**：超過 24 小時視窗後，只能使用預先審核的訊息模板
- **速率限制**：API 有每秒和每日的訊息發送限制
- **非官方方式的風險**：使用 Baileys 可能違反 WhatsApp 的服務條款，有被封號的風險

---

## 2. WhatsApp Business API 配置

### 2.1 Meta Business Suite 設定

1. 前往 [Meta Business Suite](https://business.facebook.com/)
2. 建立或登入商業帳號
3. 進入 **Business Settings**
4. 新增 **WhatsApp Business Account**

### 2.2 建立 WhatsApp Business 帳號

```
Step 1: 在 Meta Developer Portal 建立應用
  → developers.facebook.com
  → 建立新應用
  → 選擇「Business」類型
  → 命名你的應用

Step 2: 新增 WhatsApp 產品
  → 在應用設定中點擊「Add Products」
  → 選擇「WhatsApp」
  → 按照設定精靈完成初始設定

Step 3: 取得臨時 Token（用於測試）
  → WhatsApp → API Setup
  → 複製「Temporary access token」
  → 注意：臨時 Token 24 小時後過期

Step 4: 設定永久 Token
  → Business Settings → System Users
  → 建立系統使用者
  → 分配 WhatsApp Business 權限
  → 生成永久 Token
```

### 2.3 取得 API Token

```bash
# 將 Token 安全存放
echo "WHATSAPP_TOKEN=EAAxxxxxxx..." >> ~/.openclaw/.env
echo "WHATSAPP_PHONE_NUMBER_ID=1234567890" >> ~/.openclaw/.env
echo "WHATSAPP_BUSINESS_ACCOUNT_ID=0987654321" >> ~/.openclaw/.env
```

### 2.4 Webhook 配置

WhatsApp Business API 使用 Webhook 接收訊息。你需要設定一個公開可存取的 Webhook URL：

```bash
# OpenClaw 內建 Webhook 伺服器
openclaw webhook start --port 8443 --path /webhook/whatsapp

# 或使用 ngrok 進行本地測試
ngrok http 8443
# 複製生成的 HTTPS URL
```

在 Meta Developer Portal 中設定 Webhook：

1. 進入 WhatsApp → Configuration
2. 在 Webhook 區塊中填入：
   - **Callback URL**：`https://your-domain.com/webhook/whatsapp`
   - **Verify Token**：你自訂的驗證字串
3. 訂閱 Webhook 欄位：
   - `messages`（必要）
   - `message_status`（建議）

### 2.5 電話號碼驗證

Business API 需要一個專用的電話號碼：

```
1. 購買或使用現有的電話號碼
2. 在 WhatsApp Business 設定中新增電話號碼
3. 通過簡訊或語音通話驗證
4. 等待 Meta 審核（通常 1-3 天）
```

---

## 3. QR Code 配對（WhatsApp Web 方式）

### 3.1 使用 Baileys 函式庫

對於個人使用或實驗，OpenClaw 支援通過 Baileys 函式庫直接連接 WhatsApp Web。這種方式不需要 Business 帳號，只需要一個普通的 WhatsApp 帳號。

```bash
# Baileys 會在 OpenClaw 安裝時自動包含
# 確認依賴已安裝
openclaw whatsapp check-deps
```

### 3.2 QR Code 配對流程

```bash
# 啟動 WhatsApp 配對
openclaw whatsapp pair

# 系統會顯示 QR Code（在終端中）
# ┌───────────────────────────┐
# │ ▄▄▄▄▄ █▀█ █▄▄▀▀█ ▄▄▄▄▄ │
# │ █   █ █▄▀▄█ ▀█▄█ █   █ │
# │ █▄▄▄█ █ ▀▄▀▄▀▀▀█ █▄▄▄█ │
# │ ▄▄▄▄▄▄▄ █ █▄█ █ ▄▄▄  │
# │ ...                      │
# └───────────────────────────┘
# 
# 請用 WhatsApp 手機版掃描此 QR Code
# WhatsApp → 設定 → 已連結的裝置 → 連結裝置

# 配對成功後顯示：
# ✅ WhatsApp paired successfully!
# Connected as: +886 912 345 678
```

### 3.3 Session 持久化

配對成功後，Session 資料需要被持久化，避免每次重啟都需要重新掃碼：

```json
{
  "channels": {
    "whatsapp": {
      "sessionStorage": {
        "type": "file",
        "path": "~/.openclaw/whatsapp-session/"
      }
    }
  }
}
```

Session 文件包含認證憑證，必須安全保管：

```bash
# 設定適當的文件權限
chmod 700 ~/.openclaw/whatsapp-session/
chmod 600 ~/.openclaw/whatsapp-session/*
```

### 3.4 多裝置支援

WhatsApp 的多裝置功能允許最多 4 個裝置同時連線。OpenClaw 作為其中一個裝置連線。即使手機離線，Bot 仍然可以獨立運作（最多 14 天）。

---

## 4. OpenClaw WhatsApp 配置

### 4.1 基礎配置

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "mode": "baileys",
      "allowFrom": {
        "users": ["whatsapp:+886912345678"]
      }
    }
  }
}
```

### 4.2 完整配置範例

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "mode": "baileys",
      
      "allowFrom": {
        "users": [
          "whatsapp:+886912345678",
          "whatsapp:+886923456789"
        ],
        "groups": [
          "group:120363012345678901@g.us"
        ]
      },
      
      "dmPolicy": {
        "enabled": true,
        "allowUnknownUsers": false,
        "greeting": "你好！我是 EasonClaw 🐾 有什麼我能幫你的嗎？",
        "rateLimitPerUser": {
          "messages": 30,
          "period": "1m"
        }
      },
      
      "groupPolicy": {
        "enabled": true,
        "triggerMode": "mention",
        "respondInQuote": true,
        "cooldown": 5
      },
      
      "voice": {
        "enabled": true,
        "autoTranscribe": true,
        "replyWithVoice": false,
        "stt": {
          "provider": "whisper",
          "language": "zh"
        },
        "tts": {
          "provider": "openai-tts",
          "voice": "nova"
        }
      },
      
      "media": {
        "acceptImages": true,
        "acceptDocuments": true,
        "acceptLocations": true,
        "maxFileSize": "10MB"
      },
      
      "sessionStorage": {
        "type": "file",
        "path": "~/.openclaw/whatsapp-session/"
      },
      
      "resilience": {
        "reconnect": {
          "enabled": true,
          "maxRetries": 20,
          "baseDelay": 5000
        }
      }
    }
  }
}
```

### 4.3 配置欄位詳解

| 欄位 | 類型 | 說明 |
|------|------|------|
| `mode` | string | `"baileys"`（WhatsApp Web）或 `"cloud-api"`（Business API） |
| `dmPolicy.greeting` | string | 新對話的問候語 |
| `groupPolicy.triggerMode` | string | `"mention"`: @提及; `"prefix"`: 前綴; `"always"`: 全部 |
| `groupPolicy.respondInQuote` | bool | 以引用回覆方式回應 |
| `voice.autoTranscribe` | bool | 自動將語音訊息轉為文字處理 |
| `voice.replyWithVoice` | bool | 是否用語音訊息回覆 |

---

## 5. 群組設定

### 5.1 群組觸發模式

WhatsApp 群組中，建議使用 `mention` 或 `prefix` 模式：

```json
{
  "groupPolicy": {
    "triggerMode": "mention",
    "mentionName": "EasonClaw"
  }
}
```

在 WhatsApp 群組中 @提及 Bot 的方式是在訊息中包含 Bot 的電話號碼或名稱。

### 5.2 群組管理功能

```json
{
  "groupPolicy": {
    "groupManagement": {
      "welcomeNewMembers": true,
      "welcomeMessage": "歡迎 {name} 加入群組！🎉",
      "summarizeOnRequest": true,
      "moderationEnabled": false
    }
  }
}
```

### 5.3 群組隱私考量

WhatsApp 群組中的所有訊息對所有成員可見。OpenClaw 在群組中的回應也是公開的，因此：

- 不要在群組中回覆敏感資訊
- 引導使用者到私訊處理個人事務
- 遵守群組的隱私規範

---

## 6. 語音訊息支援

### 6.1 接收語音訊息

WhatsApp 是語音訊息使用率最高的平台之一。OpenClaw 能自動處理收到的語音訊息：

```
使用者發送語音訊息 → WhatsApp 傳遞 Opus 音訊
    │
    ▼
OpenClaw 接收 → 下載音訊文件 → 格式轉換（Opus → WAV）
    │
    ▼
STT 轉錄 → 生成文字 → 送入 ReAct 引擎
    │
    ▼
生成回應 → 文字回覆（或語音回覆）
```

### 6.2 發送語音回覆

如果啟用 `replyWithVoice`，OpenClaw 會以語音訊息回覆：

```
ReAct 引擎輸出文字回應
    │
    ▼
TTS 合成語音（使用配置的供應商）
    │
    ▼
格式轉換（WAV → Opus）→ 上傳到 WhatsApp → 發送語音訊息
```

### 6.3 語音處理管線

```python
# WhatsApp 語音訊息處理流程
async def handle_voice_message(message):
    # 1. 下載語音文件
    audio_data = await download_media(message.media_url)
    
    # 2. 轉換格式
    wav_data = convert_opus_to_wav(audio_data)
    
    # 3. STT 轉錄
    transcript = await stt_provider.transcribe(wav_data, language="zh")
    
    # 4. 送入 OpenClaw 核心
    response = await openclaw_core.process(
        text=transcript,
        user_id=message.sender,
        channel="whatsapp",
        metadata={"original_type": "voice"}
    )
    
    # 5. 回覆（文字或語音）
    if config.voice.replyWithVoice:
        audio_response = await tts_provider.synthesize(response.text)
        await send_voice_message(message.sender, audio_response)
    else:
        await send_text_message(message.sender, response.text)
```

---

## 7. 媒體處理

### 7.1 圖片處理

OpenClaw 可以接收和分析 WhatsApp 中的圖片（如果配置了視覺模型）：

```json
{
  "media": {
    "acceptImages": true,
    "imageAnalysis": {
      "enabled": true,
      "provider": "openai-vision",
      "autoDescribe": true
    }
  }
}
```

### 7.2 文件處理

```json
{
  "media": {
    "acceptDocuments": true,
    "allowedTypes": ["pdf", "doc", "docx", "txt", "csv"],
    "maxFileSize": "10MB"
  }
}
```

### 7.3 位置分享

WhatsApp 的位置分享功能可以與天氣查詢等技能結合：

```
使用者分享位置 → OpenClaw 接收經緯度
    │
    ▼
自動觸發相關技能（天氣、附近餐廳等）
```

---

## 8. 安全與隱私

### 8.1 端到端加密

WhatsApp 的端到端加密意味著：

- **Meta/WhatsApp 看不到訊息內容**
- **OpenClaw 能看到**：因為 Bot 是對話的一方
- **本地處理的重要性**：如果使用 Baileys 方式，訊息在本地解密處理

### 8.2 資料存儲考量

```json
{
  "privacy": {
    "messageRetention": "30d",
    "deleteMediaAfterProcessing": true,
    "encryptLocalStorage": true,
    "logLevel": "minimal"
  }
}
```

### 8.3 合規要求

- 告知使用者他們在與 AI 對話
- 不在未經同意的情況下加入群組
- 遵守當地的資料保護法規（GDPR 等）

---

## 9. 部署與維護

```bash
# 啟動 WhatsApp 通道
openclaw start --channel whatsapp

# 監控連線狀態
openclaw channels status

# 查看日誌
openclaw logs --filter channel:whatsapp

# 重新配對（如果 Session 過期）
openclaw whatsapp pair --force
```

**保持連線穩定的建議**：

- 保持手機上的 WhatsApp 帳號活躍
- 不要頻繁重啟服務
- 設定適當的重連策略
- 監控連線狀態，及時處理斷線

---

## 10. 常見問題

**Q: 用 Baileys 會被封號嗎？**

A: 有風險。避免以下行為來降低風險：
- 不要發送垃圾訊息
- 控制發送頻率
- 不要大量新增聯絡人
- 使用專用的電話號碼

**Q: Business API 的費用是多少？**

A: 根據訊息類型和地區不同。前 1000 則服務類訊息免費，超過後按條收費。詳見 Meta 官方定價。

**Q: 語音訊息有長度限制嗎？**

A: WhatsApp 原生支援最長 2 分鐘的語音訊息。OpenClaw 建議限制在 1 分鐘以內，以確保 STT 轉錄的準確度。

**Q: 可以同時在手機和 OpenClaw 上使用同一個 WhatsApp 帳號嗎？**

A: 可以，WhatsApp 的多裝置功能支援最多 4 個裝置同時連線。

**Q: 群組中的所有訊息都會被 AI 處理嗎？**

A: 取決於 `triggerMode` 設定。使用 `mention` 模式時，只有 @提及 Bot 的訊息才會被處理。
