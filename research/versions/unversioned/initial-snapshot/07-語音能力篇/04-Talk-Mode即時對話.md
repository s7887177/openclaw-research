# Talk Mode 即時對話

## 目錄

- [1. Talk Mode 概述](#1-talk-mode-概述)
  - [1.1 什麼是 Talk Mode](#11-什麼是-talk-mode)
  - [1.2 Talk Mode vs 語音訊息](#12-talk-mode-vs-語音訊息)
  - [1.3 使用場景](#13-使用場景)
- [2. Talk Mode 架構](#2-talk-mode-架構)
  - [2.1 核心引擎](#21-核心引擎)
  - [2.2 處理管線](#22-處理管線)
  - [2.3 狀態機](#23-狀態機)
  - [2.4 延遲預算分配](#24-延遲預算分配)
- [3. Barge-In 打斷機制](#3-barge-in-打斷機制)
  - [3.1 什麼是 Barge-In](#31-什麼是-barge-in)
  - [3.2 Barge-In 策略](#32-barge-in-策略)
  - [3.3 Polite Barge-In（禮貌打斷）](#33-polite-barge-in禮貌打斷)
  - [3.4 配置方式](#34-配置方式)
- [4. Wake Words 喚醒詞](#4-wake-words-喚醒詞)
  - [4.1 喚醒詞的作用](#41-喚醒詞的作用)
  - [4.2 配置方式](#42-配置方式)
  - [4.3 自定義喚醒詞](#43-自定義喚醒詞)
- [5. VAD 與 Endpointing](#5-vad-與-endpointing)
  - [5.1 VAD 在 Talk Mode 中的角色](#51-vad-在-talk-mode-中的角色)
  - [5.2 Endpointing 策略](#52-endpointing-策略)
  - [5.3 微調建議](#53-微調建議)
- [6. Kiwi Voice 引擎](#6-kiwi-voice-引擎)
  - [6.1 什麼是 Kiwi Voice](#61-什麼是-kiwi-voice)
  - [6.2 Kiwi Voice 配置](#62-kiwi-voice-配置)
  - [6.3 與標準管線的差異](#63-與標準管線的差異)
- [7. 通道整合](#7-通道整合)
  - [7.1 Discord 語音頻道](#71-discord-語音頻道)
  - [7.2 Twilio 電話整合](#72-twilio-電話整合)
  - [7.3 WebRTC 整合](#73-webrtc-整合)
- [8. 回聲消除與音訊處理](#8-回聲消除與音訊處理)
  - [8.1 回聲消除的重要性](#81-回聲消除的重要性)
  - [8.2 技術方案](#82-技術方案)
  - [8.3 配置方式](#83-配置方式)
- [9. Talk Mode 完整配置](#9-talk-mode-完整配置)
  - [9.1 基礎配置](#91-基礎配置)
  - [9.2 完整配置範例](#92-完整配置範例)
  - [9.3 配置欄位詳解](#93-配置欄位詳解)
- [10. 效能優化](#10-效能優化)
  - [10.1 延遲優化策略](#101-延遲優化策略)
  - [10.2 品質優化策略](#102-品質優化策略)
  - [10.3 資源管理](#103-資源管理)
- [11. 疑難排解](#11-疑難排解)

---

## 1. Talk Mode 概述

### 1.1 什麼是 Talk Mode

Talk Mode 是 OpenClaw 最進階的語音功能 — 即時雙向語音對話。它不是「錄音 → 發送 → 等待」的異步模式，而是像打電話一樣，使用者說話的同時 AI 就在聽，AI 回應時使用者也能即時聽到，甚至可以打斷 AI。

```
傳統語音訊息：
  使用者：[錄音 5 秒] → [發送] → [等待 3-8 秒] → [聽回覆]
  延遲體驗：約 10-15 秒一個來回

Talk Mode：
  使用者：邊說 → AI 邊聽 → 使用者說完 → [1-2 秒] → AI 邊說回覆
  延遲體驗：約 1-2 秒，像真人對話
```

### 1.2 Talk Mode vs 語音訊息

| 特性 | Talk Mode | 語音訊息 |
|------|----------|----------|
| 延遲 | 1-2 秒 | 5-15 秒 |
| 互動方式 | 即時雙向 | 單向異步 |
| 打斷支援 | ✅ | ❌ |
| 音訊品質 | 即時壓縮（較低） | 完整音訊（較高） |
| 適合場景 | 即時對話、閒聊 | 詳細問題、長回覆 |
| 通道支援 | Discord 語音、Twilio | WhatsApp、Telegram |
| 資源消耗 | 持續佔用 | 按需使用 |

### 1.3 使用場景

- **Discord 語音頻道**：在語音頻道中和 AI 即時對話
- **電話整合（Twilio）**：透過電話與 AI 通話
- **本地 Talk Mode**：在電腦上使用麥克風和揚聲器
- **車載助手**：開車時免持語音互動
- **智慧家居**：類似 Alexa/Google Home 的體驗

---

## 2. Talk Mode 架構

### 2.1 核心引擎

```
┌────────────────────────────────────────────────────┐
│                Talk Mode Engine                      │
│                                                      │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐   │
│  │ Audio  │─▶│  VAD   │─▶│  STT   │─▶│ ReAct  │   │
│  │ Input  │  │        │  │ Stream │  │ Engine │   │
│  └────────┘  └────────┘  └────────┘  └────────┘   │
│       ▲                                    │        │
│       │         ┌──────────────┐           │        │
│       │         │  Turn Mgmt   │           │        │
│       │         │              │           │        │
│       │         │ • 誰在說話？  │           ▼        │
│       │         │ • 何時回應？  │     ┌────────┐    │
│       │         │ • 是否打斷？  │     │  TTS   │    │
│       │         └──────────────┘     │ Stream │    │
│       │                              └────────┘    │
│       │                                    │        │
│  ┌────────┐       ┌──────────┐             │        │
│  │ Echo   │◀──────│  Audio   │◀────────────┘        │
│  │ Cancel │       │  Output  │                       │
│  └────────┘       └──────────┘                       │
└────────────────────────────────────────────────────┘
```

### 2.2 處理管線

完整的 Talk Mode 處理流程：

```
1. 音訊輸入（麥克風/Discord/電話）
   │
2. 回聲消除（如果 AI 正在說話）
   │
3. VAD 偵測（有人在說話嗎？）
   │
4. 串流 STT 轉錄（邊聽邊轉文字）
   │
5. Endpointing（使用者說完了嗎？）
   │
6. 送入 ReAct 引擎（串流輸入）
   │
7. LLM 生成回應（串流輸出）
   │
8. 串流 TTS 合成（邊生成邊說）
   │
9. 音訊輸出（揚聲器/Discord/電話）
   │
10. 同時監聽 Barge-In（使用者打斷了嗎？）
```

### 2.3 狀態機

Talk Mode 的狀態管理：

```
┌─────────┐    語音偵測    ┌──────────┐
│ IDLE    │─────────────▶│ LISTENING │
│ 待命    │               │ 聽取中    │
└─────────┘               └──────────┘
     ▲                         │
     │                    說完/靜音
     │                         │
     │                         ▼
     │                    ┌──────────┐
     │  回應完畢           │ THINKING │
     ├────────────────────│ 思考中    │
     │                    └──────────┘
     │                         │
     │                    生成開始
     │                         │
     │                         ▼
     │                    ┌──────────┐
     │  被打斷             │ SPEAKING │
     ├────────────────────│ 回應中    │
     │                    └──────────┘
     │                         │
     │                    說完
     │                         │
     └─────────────────────────┘
```

### 2.4 延遲預算分配

要達到流暢的對話體驗，總延遲需要控制在 1.5 秒以內：

```
延遲預算（目標 1.5 秒）：

  Endpointing 確認    ≈ 300ms
  + STT 最終結果      ≈ 200ms
  + LLM 首 Token      ≈ 500ms
  + TTS 首字節        ≈ 300ms
  + 網路/處理開銷     ≈ 200ms
  ─────────────────────────
  總計                ≈ 1500ms ✅
```

---

## 3. Barge-In 打斷機制

### 3.1 什麼是 Barge-In

Barge-In 是指使用者在 AI 正在說話時打斷它。這是自然對話的重要特徵 — 人類對話中經常互相打斷。

```
沒有 Barge-In：
  AI: "讓我詳細解釋一下這個概念。首先，我們需要..."
  使用者：（等 AI 說完才能說話）😤

有 Barge-In：
  AI: "讓我詳細解釋一下這個概..."
  使用者：「我已經知道了，直接告訴我怎麼做。」
  AI：（停止說話，開始聽）→ "好的，你可以這樣做..."
```

### 3.2 Barge-In 策略

```
策略 1: 立即停止（Immediate）
  使用者一開口 → AI 立刻停止說話
  優點：反應最快
  缺點：誤判率高（咳嗽、嘆氣也會觸發）

策略 2: 確認後停止（Confirmed）
  使用者開口 → 等待 300ms 確認是真的在說話 → AI 停止
  優點：減少誤判
  缺點：有短暫的重疊

策略 3: 禮貌打斷（Polite）← 推薦
  使用者開口 → AI 降低音量 → 確認使用者在說話 → AI 停止
  優點：最自然的對話體驗
  缺點：實作較複雜
```

### 3.3 Polite Barge-In（禮貌打斷）

OpenClaw 的 Polite Barge-In 機制模仿了人類對話中的禮貌打斷行為：

```
Step 1: 偵測到使用者可能在說話
  → AI 將音量降低到 30%（而非直接停止）

Step 2: 確認使用者確實在說話（300ms 後）
  → AI 完全停止說話
  → 開始聽取使用者的話

Step 3: 如果是誤判（使用者只是咳嗽）
  → AI 恢復原音量繼續說話
  → 使用者感受不到中斷
```

### 3.4 配置方式

```json
{
  "talkMode": {
    "bargeIn": {
      "enabled": true,
      "strategy": "polite",
      "sensitivity": 0.7,
      "confirmationDelay": 300,
      "volumeReduction": 0.3,
      "resumeIfFalseAlarm": true
    }
  }
}
```

---

## 4. Wake Words 喚醒詞

### 4.1 喚醒詞的作用

在群組語音場景中（如 Discord 語音頻道），AI 不應該對所有說話都回應。喚醒詞讓使用者可以通過特定的詞語來「喚醒」AI：

```
使用者 A: "今天的會議幾點？"        → AI 不回應（不是對 AI 說的）
使用者 B: "三點開始。"              → AI 不回應
使用者 A: "EasonClaw，幫我設提醒"   → AI 回應（偵測到喚醒詞）
```

### 4.2 配置方式

```json
{
  "talkMode": {
    "wakeWord": {
      "enabled": true,
      "words": ["EasonClaw", "小易", "Hey Claw"],
      "provider": "porcupine",
      "sensitivity": 0.7,
      "listenTimeout": 10000,
      "confirmSound": true
    }
  }
}
```

| 參數 | 說明 | 建議值 |
|------|------|--------|
| `words` | 喚醒詞列表 | 自定義 |
| `provider` | 喚醒詞偵測引擎 | `"porcupine"` |
| `sensitivity` | 靈敏度（0-1） | 0.7 |
| `listenTimeout` | 喚醒後的聆聽時長（ms） | 10000 |
| `confirmSound` | 偵測到喚醒詞時播放確認音 | true |

### 4.3 自定義喚醒詞

使用 Picovoice Porcupine 建立自定義喚醒詞：

```bash
# 1. 在 Picovoice Console 建立自定義喚醒詞
# 2. 下載模型文件
# 3. 放入 OpenClaw 配置目錄

# 目錄結構
~/.openclaw/
  wake-words/
    easonclaw.ppn    # 自定義喚醒詞模型
```

```json
{
  "talkMode": {
    "wakeWord": {
      "customModelPath": "~/.openclaw/wake-words/easonclaw.ppn"
    }
  }
}
```

---

## 5. VAD 與 Endpointing

### 5.1 VAD 在 Talk Mode 中的角色

VAD 在 Talk Mode 中比在批次轉錄中更為關鍵，它需要即時判斷：

```
持續音訊流
    │
    ▼
┌──────────────┐
│     VAD      │
├──────────────┤
│ 靜音 → IDLE  │
│ 語音 → START │
│ 靜音 → END?  │ ← 使用者是在停頓還是說完了？
│ 語音 → 繼續  │
└──────────────┘
```

### 5.2 Endpointing 策略

判斷使用者是否「說完了」是 Talk Mode 最困難的問題之一。太早判斷會打斷使用者，太晚判斷會增加延遲：

```json
{
  "talkMode": {
    "endpointing": {
      "strategy": "hybrid",
      
      "silenceBased": {
        "duration": 700,
        "energyThreshold": 0.02
      },
      
      "semanticBased": {
        "enabled": true,
        "checkPartialTranscript": true,
        "completenesScore": 0.8
      },
      
      "prosodyBased": {
        "enabled": true,
        "pitchDropThreshold": 0.3,
        "energyDropThreshold": 0.4
      }
    }
  }
}
```

**三種策略**：

1. **靜音基礎**：靜音超過 700ms → 認為說完
2. **語義基礎**：分析部分轉錄文字，判斷句子是否完整
3. **韻律基礎**：語調下降 + 音量減小 → 認為說完

### 5.3 微調建議

```
太常被打斷（AI 太早回應）：
  → 增加 silenceBased.duration（如 700 → 1000）
  → 降低 semanticBased.completenessScore

AI 回應太慢（等太久才說話）：
  → 減少 silenceBased.duration（如 700 → 500）
  → 啟用 prosodyBased 偵測
```

---

## 6. Kiwi Voice 引擎

### 6.1 什麼是 Kiwi Voice

Kiwi Voice 是 OpenClaw 內建的端到端語音對話引擎，整合了 VAD + STT + LLM + TTS 的完整管線，針對低延遲對話進行了最佳化。

### 6.2 Kiwi Voice 配置

```json
{
  "talkMode": {
    "engine": "kiwi",
    
    "kiwi": {
      "stt": {
        "provider": "deepgram",
        "model": "nova-2"
      },
      "llm": {
        "provider": "openai",
        "model": "gpt-4o",
        "maxTokens": 150,
        "temperature": 0.7,
        "systemPrompt": "你是 EasonClaw，一個友善的中文語音助手。回覆請簡短，每次不超過 2-3 句話。"
      },
      "tts": {
        "provider": "openai-tts",
        "model": "tts-1",
        "voice": "nova"
      },
      
      "optimization": {
        "prefetchEnabled": true,
        "parallelPipeline": true,
        "shortResponseBias": true,
        "maxResponseLength": 100
      }
    }
  }
}
```

### 6.3 與標準管線的差異

```
標準管線：
  Audio → STT → 完成 → ReAct → 完成 → TTS → 完成 → Play
  延遲：2-4 秒

Kiwi Voice 管線：
  Audio → STT（串流）→ ReAct（串流輸入/輸出）→ TTS（串流）→ Play
  所有階段並行，管線式處理
  延遲：0.8-1.5 秒
```

---

## 7. 通道整合

### 7.1 Discord 語音頻道

Discord 是 Talk Mode 最主要的使用場景：

```json
{
  "channels": {
    "discord": {
      "voice": {
        "enabled": true,
        "talkMode": true,
        "autoJoin": false,
        "joinCommand": "/voice join",
        "leaveCommand": "/voice leave",
        
        "audioSettings": {
          "inputSampleRate": 48000,
          "outputSampleRate": 48000,
          "channels": 2,
          "frameSize": 960
        }
      }
    }
  }
}
```

使用流程：

```
1. 使用者在 Discord 中使用 /voice join
2. Bot 加入使用者所在的語音頻道
3. Bot 開始監聽語音（使用喚醒詞或 @提及）
4. 即時語音對話開始
5. 使用 /voice leave 結束
```

### 7.2 Twilio 電話整合

Twilio 讓 OpenClaw 能接聽和撥打真正的電話：

```json
{
  "channels": {
    "twilio": {
      "enabled": true,
      "accountSid": "${TWILIO_ACCOUNT_SID}",
      "authToken": "${TWILIO_AUTH_TOKEN}",
      "phoneNumber": "+18001234567",
      
      "talkMode": {
        "enabled": true,
        "greeting": "你好，這裡是 EasonClaw 助理服務。有什麼我能幫你的嗎？",
        "endCall": {
          "keyword": "掛斷",
          "timeout": 30
        }
      },
      
      "webhookUrl": "https://your-domain.com/webhook/twilio"
    }
  }
}
```

### 7.3 WebRTC 整合

對於網頁應用，可以使用 WebRTC 實現瀏覽器中的 Talk Mode：

```json
{
  "channels": {
    "webrtc": {
      "enabled": true,
      "signalingServer": "wss://your-domain.com/webrtc",
      "iceServers": [
        { "urls": "stun:stun.l.google.com:19302" }
      ],
      "talkMode": {
        "enabled": true
      }
    }
  }
}
```

---

## 8. 回聲消除與音訊處理

### 8.1 回聲消除的重要性

在 Talk Mode 中，AI 的語音輸出會被麥克風收回，形成回聲迴路：

```
❌ 沒有回聲消除：
  AI 說話 → 揚聲器播放 → 麥克風收音 → STT 轉錄 AI 的聲音
  → AI 以為使用者在說話 → 再次回應 → 無限迴圈 😱

✅ 有回聲消除：
  AI 說話 → 揚聲器播放 → 麥克風收音 → 回聲消除器去除 AI 聲音
  → 只保留使用者的聲音 → 正常處理
```

### 8.2 技術方案

```
方案 1: 硬體回聲消除
  → 使用耳機（最簡單有效）
  → 使用有 AEC 的麥克風

方案 2: 軟體回聲消除
  → WebRTC AEC3 演算法
  → SpeexDSP AEC

方案 3: 暫停監聽
  → AI 說話時暫停 STT（最簡單但不支援 Barge-In）
```

### 8.3 配置方式

```json
{
  "talkMode": {
    "echoCancellation": {
      "enabled": true,
      "provider": "webrtc-aec3",
      "tailLength": 128,
      "suppressionLevel": "moderate"
    }
  }
}
```

---

## 9. Talk Mode 完整配置

### 9.1 基礎配置

```json
{
  "talkMode": {
    "enabled": true,
    "engine": "kiwi",
    "stt": { "provider": "deepgram" },
    "tts": { "provider": "openai-tts", "voice": "nova" }
  }
}
```

### 9.2 完整配置範例

```json
{
  "talkMode": {
    "enabled": true,
    "engine": "kiwi",
    
    "stt": {
      "provider": "deepgram",
      "model": "nova-2",
      "language": "zh-TW",
      "interimResults": true,
      "utteranceEndMs": 1000,
      "vadEvents": true
    },
    
    "tts": {
      "provider": "openai-tts",
      "model": "tts-1",
      "voice": "nova",
      "speed": 1.05,
      "format": "opus"
    },
    
    "llm": {
      "model": "gpt-4o",
      "maxTokens": 150,
      "temperature": 0.7,
      "systemPrompt": "你是 EasonClaw，一個友善的中文語音助手。回覆請簡短自然，像跟朋友聊天一樣。"
    },
    
    "vad": {
      "provider": "silero",
      "threshold": 0.5,
      "minSpeechDuration": 250,
      "minSilenceDuration": 500
    },
    
    "endpointing": {
      "strategy": "hybrid",
      "silenceThreshold": 700,
      "semanticCheck": true
    },
    
    "bargeIn": {
      "enabled": true,
      "strategy": "polite",
      "sensitivity": 0.7,
      "confirmationDelay": 300
    },
    
    "wakeWord": {
      "enabled": false
    },
    
    "echoCancellation": {
      "enabled": true,
      "provider": "webrtc-aec3"
    },
    
    "performance": {
      "prefetchEnabled": true,
      "maxConcurrentStreams": 1,
      "audioBufferSize": 4096
    },
    
    "safety": {
      "maxSessionDuration": 3600,
      "idleTimeout": 300,
      "maxDailyMinutes": 120
    }
  }
}
```

### 9.3 配置欄位詳解

| 欄位 | 說明 | 建議值 |
|------|------|--------|
| `engine` | Talk Mode 引擎 | `"kiwi"` |
| `bargeIn.strategy` | 打斷策略 | `"polite"` |
| `endpointing.strategy` | 端點偵測策略 | `"hybrid"` |
| `safety.maxSessionDuration` | 最長對話時間（秒） | `3600` |
| `safety.idleTimeout` | 閒置超時（秒） | `300` |

---

## 10. 效能優化

### 10.1 延遲優化策略

```
1. 選擇低延遲的供應商組合：
   STT: Deepgram Nova-2（~200ms）
   LLM: GPT-4o（~500ms first token）
   TTS: OpenAI tts-1（~300ms）

2. 啟用管線並行：
   LLM 邊生成 → TTS 邊合成

3. 縮短回應長度：
   maxTokens: 150（約 2-3 句中文）

4. 預熱連接：
   啟動時建立到所有供應商的持久連接

5. 使用 Opus 編碼：
   低延遲，低頻寬
```

### 10.2 品質優化策略

- 使用適合中文的 STT 和 TTS 供應商
- 啟用音訊前處理（降噪、正規化）
- 針對語音場景調整 System Prompt
- 定期測試和調校 Endpointing 參數

### 10.3 資源管理

```json
{
  "talkMode": {
    "resources": {
      "cpuLimit": "200%",
      "memoryLimit": "512MB",
      "networkBandwidth": "1Mbps",
      "gpuEnabled": false
    }
  }
}
```

---

## 11. 疑難排解

**Q: AI 經常打斷我（太早回應）**

A: 增加 `endpointing.silenceThreshold`（如 700 → 1000ms），或降低 VAD 靈敏度。

**Q: AI 回應太慢**

A: 
1. 檢查網路延遲
2. 使用更快的 STT/TTS 供應商
3. 減少 `llm.maxTokens`
4. 啟用 `prefetchEnabled`

**Q: 有回聲問題**

A: 
1. 使用耳機（最簡單有效）
2. 啟用 `echoCancellation`
3. 確認麥克風不在靠近揚聲器的位置

**Q: 在 Discord 語音頻道中 Bot 不說話**

A: 
1. 確認 Bot 有語音頻道的權限（Connect + Speak）
2. 使用 `/voice join` 讓 Bot 加入頻道
3. 檢查 TTS 供應商的 API Key 是否有效
4. 查看日誌：`openclaw logs --filter talkmode`

**Q: 語音辨識不準確**

A: 
1. 確認 STT 語言設定為 `"zh-TW"`
2. 啟用降噪
3. 確認麥克風品質
4. 嘗試不同的 STT 供應商
