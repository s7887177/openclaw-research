# STT 語音轉文字

## 目錄

- [1. STT 基礎概念](#1-stt-基礎概念)
  - [1.1 什麼是 STT](#11-什麼是-stt)
  - [1.2 STT 在 OpenClaw 中的角色](#12-stt-在-openclaw-中的角色)
  - [1.3 轉錄模式](#13-轉錄模式)
- [2. Whisper（OpenAI）](#2-whisperopenai)
  - [2.1 Whisper 模型介紹](#21-whisper-模型介紹)
  - [2.2 API 配置](#22-api-配置)
  - [2.3 進階參數](#23-進階參數)
  - [2.4 本地 Whisper（whisper.cpp）](#24-本地-whisperwhispercpp)
- [3. Deepgram](#3-deepgram)
  - [3.1 Deepgram 介紹](#31-deepgram-介紹)
  - [3.2 配置方式](#32-配置方式)
  - [3.3 串流轉錄](#33-串流轉錄)
  - [3.4 Deepgram 進階功能](#34-deepgram-進階功能)
- [4. Azure Speech Service](#4-azure-speech-service)
  - [4.1 Azure Speech 介紹](#41-azure-speech-介紹)
  - [4.2 配置方式](#42-配置方式)
  - [4.3 中文最佳化](#43-中文最佳化)
- [5. Google Speech-to-Text](#5-google-speech-to-text)
  - [5.1 Google STT 介紹](#51-google-stt-介紹)
  - [5.2 配置方式](#52-配置方式)
- [6. 靜音偵測與 VAD](#6-靜音偵測與-vad)
  - [6.1 什麼是 VAD](#61-什麼是-vad)
  - [6.2 VAD 配置](#62-vad-配置)
  - [6.3 Endpointing 策略](#63-endpointing-策略)
- [7. 音訊前處理](#7-音訊前處理)
  - [7.1 降噪](#71-降噪)
  - [7.2 音量正規化](#72-音量正規化)
  - [7.3 格式轉換](#73-格式轉換)
- [8. 多語言與方言](#8-多語言與方言)
  - [8.1 語言偵測](#81-語言偵測)
  - [8.2 中英混合](#82-中英混合)
  - [8.3 方言支援](#83-方言支援)
- [9. 隱私與安全](#9-隱私與安全)
  - [9.1 音訊資料處理原則](#91-音訊資料處理原則)
  - [9.2 本地處理 vs 雲端處理](#92-本地處理-vs-雲端處理)
  - [9.3 敏感內容過濾](#93-敏感內容過濾)
- [10. 效能調校](#10-效能調校)
  - [10.1 延遲優化](#101-延遲優化)
  - [10.2 準確率優化](#102-準確率優化)
  - [10.3 成本優化](#103-成本優化)
- [11. 常見問題](#11-常見問題)

---

## 1. STT 基礎概念

### 1.1 什麼是 STT

STT（Speech-to-Text），又稱 ASR（Automatic Speech Recognition），是將語音音訊轉換為文字的技術。在 AI 助手的場景中，STT 扮演「耳朵」的角色 — 讓 AI 能「聽懂」使用者說的話。

### 1.2 STT 在 OpenClaw 中的角色

```
                    ┌──────────┐
使用者語音 ────────▶│   STT    │────────▶ 文字
                    │          │
WhatsApp 語音訊息   │ 供應商:   │         送入 ReAct 引擎
Telegram 語音訊息   │ Whisper  │
Discord 語音頻道    │ Deepgram │
電話語音           │ Azure    │
                    └──────────┘
```

STT 的輸出品質直接決定了 AI 的理解品質。如果轉錄有誤，後續的所有處理都會受到影響。

### 1.3 轉錄模式

OpenClaw 支援兩種轉錄模式：

**批次模式（Batch）**：

```
完整音訊 → 一次性轉錄 → 完整文字
適合：語音訊息（WhatsApp、Telegram）
特點：品質最高，但有延遲
```

**串流模式（Streaming）**：

```
音訊片段1 → 部分文字1
音訊片段2 → 部分文字2（可能修正文字1）
音訊片段3 → 部分文字3
...
說完 → 最終確認的完整文字
適合：Talk Mode（Discord 語音頻道）
特點：低延遲，但可能有中間修正
```

---

## 2. Whisper（OpenAI）

### 2.1 Whisper 模型介紹

Whisper 是 OpenAI 開發的通用語音辨識模型，支援 99 種語言，中文辨識品質尤其出色。它有兩種使用方式：

- **OpenAI API**：雲端託管，簡單方便
- **whisper.cpp**：本地運行，免費且注重隱私

### 2.2 API 配置

```json
{
  "voice": {
    "stt": {
      "provider": "whisper",
      "apiKey": "${OPENAI_API_KEY}",
      "model": "whisper-1",
      "language": "zh",
      "responseFormat": "verbose_json",
      "temperature": 0
    }
  }
}
```

使用範例：

```python
# OpenClaw 內部的 Whisper STT 呼叫
import openai

client = openai.OpenAI()

# 轉錄語音檔案
with open("voice_message.ogg", "rb") as audio_file:
    transcript = client.audio.transcriptions.create(
        model="whisper-1",
        file=audio_file,
        language="zh",
        response_format="verbose_json",
        temperature=0
    )

print(transcript.text)
# 輸出: "你好，請問今天天氣如何？"
```

### 2.3 進階參數

```json
{
  "voice": {
    "stt": {
      "provider": "whisper",
      "model": "whisper-1",
      "language": "zh",
      "temperature": 0,
      "prompt": "以下是繁體中文對話",
      "responseFormat": "verbose_json",
      "timestampGranularities": ["word", "segment"]
    }
  }
}
```

| 參數 | 說明 | 建議值 |
|------|------|--------|
| `language` | 指定語言可提高準確率 | `"zh"` |
| `temperature` | 0=最確定，1=最有創意 | `0` |
| `prompt` | 提示詞，幫助模型理解上下文 | 繁體中文提示 |
| `responseFormat` | 回應格式 | `"verbose_json"` |

### 2.4 本地 Whisper（whisper.cpp）

對於注重隱私或預算有限的使用者，可以在本地運行 Whisper：

```bash
# 安裝 whisper.cpp
git clone https://github.com/ggerganov/whisper.cpp
cd whisper.cpp
make

# 下載模型
bash ./models/download-ggml-model.sh large-v3

# 測試轉錄
./main -m models/ggml-large-v3.bin -l zh -f audio.wav
```

OpenClaw 配置：

```json
{
  "voice": {
    "stt": {
      "provider": "whisper-local",
      "modelPath": "~/.openclaw/models/ggml-large-v3.bin",
      "language": "zh",
      "threads": 4,
      "useGPU": true
    }
  }
}
```

本地 Whisper 的模型大小與品質：

| 模型 | 大小 | VRAM 需求 | 相對品質 | 速度 |
|------|------|-----------|----------|------|
| tiny | 75MB | 1GB | ⭐⭐ | 最快 |
| base | 142MB | 1GB | ⭐⭐⭐ | 快 |
| small | 466MB | 2GB | ⭐⭐⭐⭐ | 中 |
| medium | 1.5GB | 5GB | ⭐⭐⭐⭐ | 慢 |
| large-v3 | 3.1GB | 10GB | ⭐⭐⭐⭐⭐ | 最慢 |

---

## 3. Deepgram

### 3.1 Deepgram 介紹

Deepgram 是一個專為開發者設計的 STT 服務，以其超低延遲和串流能力著稱。它是 Talk Mode 的首選 STT 供應商。

### 3.2 配置方式

```json
{
  "voice": {
    "stt": {
      "provider": "deepgram",
      "apiKey": "${DEEPGRAM_API_KEY}",
      "model": "nova-2",
      "language": "zh-TW",
      "smartFormat": true,
      "punctuate": true,
      "diarize": false,
      "alternatives": 1
    }
  }
}
```

### 3.3 串流轉錄

Deepgram 的串流轉錄是其最大優勢，特別適合 Talk Mode：

```python
# Deepgram 串流轉錄範例
from deepgram import DeepgramClient, LiveOptions

deepgram = DeepgramClient(api_key)

connection = deepgram.listen.live.v("1")

options = LiveOptions(
    model="nova-2",
    language="zh-TW",
    smart_format=True,
    interim_results=True,
    utterance_end_ms=1000,
    vad_events=True
)

connection.start(options)

# 即時事件處理
@connection.on("transcript")
def on_transcript(result):
    transcript = result.channel.alternatives[0].transcript
    is_final = result.is_final
    
    if is_final:
        print(f"[最終] {transcript}")
        # 送入 ReAct 引擎
    else:
        print(f"[中間] {transcript}")
        # 可用於顯示即時字幕
```

### 3.4 Deepgram 進階功能

- **Smart Format**：自動添加標點和大小寫
- **Diarization**：說話者辨識（多人對話時區分誰在說話）
- **Utterance End Detection**：偵測使用者說完一句話
- **Custom Vocabulary**：自定義詞彙表，提高專有名詞的辨識率
- **Topic Detection**：自動偵測對話主題

```json
{
  "voice": {
    "stt": {
      "provider": "deepgram",
      "keywords": ["OpenClaw", "EasonClaw", "MCP"],
      "keywordBoost": 1.5
    }
  }
}
```

---

## 4. Azure Speech Service

### 4.1 Azure Speech 介紹

Azure Speech Service 是微軟的語音服務，提供了非常完整的中文支援，包括多種方言。

### 4.2 配置方式

```json
{
  "voice": {
    "stt": {
      "provider": "azure-speech",
      "subscriptionKey": "${AZURE_SPEECH_KEY}",
      "region": "eastasia",
      "language": "zh-TW",
      "outputFormat": "detailed",
      "profanityFilter": "masked"
    }
  }
}
```

### 4.3 中文最佳化

Azure Speech 對中文的支援包括：

| 語言代碼 | 語言 |
|----------|------|
| `zh-TW` | 繁體中文（台灣） |
| `zh-CN` | 簡體中文（中國） |
| `zh-HK` | 粵語（香港） |
| `zh-TW-Hokkien` | 台語/閩南語 |

```json
{
  "voice": {
    "stt": {
      "provider": "azure-speech",
      "language": "zh-TW",
      "customModel": {
        "enabled": false,
        "endpointId": ""
      },
      "phraseList": [
        "OpenClaw",
        "EasonClaw",
        "語音助手"
      ]
    }
  }
}
```

---

## 5. Google Speech-to-Text

### 5.1 Google STT 介紹

Google Speech-to-Text 是 Google Cloud 的語音辨識服務，以其準確率和廣泛的語言支援著稱。

### 5.2 配置方式

```json
{
  "voice": {
    "stt": {
      "provider": "google-stt",
      "credentialsFile": "${GOOGLE_APPLICATION_CREDENTIALS}",
      "model": "latest_long",
      "languageCode": "zh-TW",
      "enableAutomaticPunctuation": true,
      "enableWordTimeOffsets": true,
      "useEnhancedModel": true
    }
  }
}
```

---

## 6. 靜音偵測與 VAD

### 6.1 什麼是 VAD

VAD（Voice Activity Detection）是判斷音訊中是否有人在說話的技術。它在語音系統中扮演關鍵角色：

- **節省資源**：只在有人說話時啟動 STT
- **確定輪次**：判斷使用者何時開始說話、何時說完
- **降低噪音**：過濾無人說話時的背景噪音

### 6.2 VAD 配置

```json
{
  "voice": {
    "vad": {
      "provider": "silero",
      "threshold": 0.5,
      "minSpeechDuration": 250,
      "minSilenceDuration": 500,
      "speechPad": 100,
      "windowSize": 512
    }
  }
}
```

| 參數 | 說明 | 建議值 |
|------|------|--------|
| `threshold` | 語音偵測閾值（0-1） | 0.5 |
| `minSpeechDuration` | 最短語音持續時間（ms） | 250 |
| `minSilenceDuration` | 判定說完的靜音時間（ms） | 500 |
| `speechPad` | 語音前後的填充時間（ms） | 100 |

### 6.3 Endpointing 策略

Endpointing 是判斷使用者是否「說完了」的策略：

```
策略 1: 固定靜音時長
  → 靜音超過 N 毫秒就認為說完了
  → 簡單但不夠智能

策略 2: 語義輔助
  → 結合部分轉錄文字判斷句子是否完整
  → 更智能但延遲稍高

策略 3: 能量下降
  → 監測音量能量的下降趨勢
  → 適合嘈雜環境
```

```json
{
  "voice": {
    "endpointing": {
      "strategy": "hybrid",
      "silenceThreshold": 600,
      "semanticCheck": true,
      "energyDropThreshold": 0.3
    }
  }
}
```

---

## 7. 音訊前處理

### 7.1 降噪

背景噪音會嚴重影響 STT 的準確率。OpenClaw 使用 RNNoise 進行即時降噪：

```json
{
  "voice": {
    "preprocessing": {
      "noiseReduction": {
        "enabled": true,
        "provider": "rnnoise",
        "aggressiveness": 2
      }
    }
  }
}
```

### 7.2 音量正規化

```json
{
  "voice": {
    "preprocessing": {
      "volumeNormalization": {
        "enabled": true,
        "targetLoudness": -23,
        "maxGain": 20
      }
    }
  }
}
```

### 7.3 格式轉換

```json
{
  "voice": {
    "preprocessing": {
      "resample": {
        "targetSampleRate": 16000,
        "targetChannels": 1,
        "targetBitDepth": 16
      }
    }
  }
}
```

---

## 8. 多語言與方言

### 8.1 語言偵測

```json
{
  "voice": {
    "stt": {
      "languageDetection": {
        "enabled": true,
        "candidateLanguages": ["zh", "en", "ja"],
        "fallbackLanguage": "zh"
      }
    }
  }
}
```

### 8.2 中英混合

許多使用者會中英混合說話。OpenClaw 可以設定為中英混合模式：

```json
{
  "voice": {
    "stt": {
      "provider": "whisper",
      "language": "zh",
      "prompt": "以下對話可能包含中文和英文混合的內容。專有名詞如 OpenClaw、GitHub、API 等請保留英文。"
    }
  }
}
```

### 8.3 方言支援

| 方言 | 支援的供應商 | 語言代碼 |
|------|------------|----------|
| 國語/普通話 | 全部 | `zh-TW` / `zh-CN` |
| 粵語 | Azure, Google | `zh-HK` / `yue-Hant-HK` |
| 台語/閩南語 | Azure（有限） | `zh-TW-Hokkien` |
| 客家話 | 有限支援 | — |

---

## 9. 隱私與安全

### 9.1 音訊資料處理原則

```json
{
  "voice": {
    "privacy": {
      "deleteAudioAfterTranscription": true,
      "noAudioLogging": true,
      "onlyStoreTranscript": true,
      "transcriptRetention": "session"
    }
  }
}
```

### 9.2 本地處理 vs 雲端處理

如果隱私是首要考量，使用本地 Whisper：

```json
{
  "voice": {
    "stt": {
      "provider": "whisper-local",
      "modelPath": "~/.openclaw/models/ggml-large-v3.bin"
    },
    "privacy": {
      "forceLocalProcessing": true,
      "blockCloudFallback": true
    }
  }
}
```

### 9.3 敏感內容過濾

```json
{
  "voice": {
    "stt": {
      "contentFiltering": {
        "profanityFilter": true,
        "piiRedaction": {
          "enabled": true,
          "types": ["phone_number", "credit_card", "ssn"]
        }
      }
    }
  }
}
```

---

## 10. 效能調校

### 10.1 延遲優化

```json
{
  "voice": {
    "stt": {
      "performance": {
        "connectionKeepAlive": true,
        "preWarmConnection": true,
        "maxConcurrent": 2,
        "timeout": 30000
      }
    }
  }
}
```

### 10.2 準確率優化

提高 STT 準確率的方法：

1. **指定語言**：不要依賴自動偵測
2. **使用提示詞**：提供上下文提示
3. **音訊前處理**：降噪和正規化
4. **選擇合適的模型**：較大的模型通常更準確
5. **自定義詞彙**：添加專有名詞

### 10.3 成本優化

```
費用最低方案：
  本地 whisper.cpp（免費）→ 需要 GPU

費用次低方案：
  Deepgram Nova-2（$0.0043/min）→ 適合大量使用

費用中等方案：
  Whisper API（$0.006/min）→ 品質最佳的雲端方案
```

---

## 11. 常見問題

**Q: 中文辨識準確率不夠高怎麼辦？**

A: 
1. 確認 `language` 設為 `"zh"`
2. 加入中文提示詞 `prompt`
3. 嘗試使用 Whisper large-v3 模型
4. 啟用音訊前處理（降噪）

**Q: 串流轉錄有亂碼怎麼辦？**

A: 串流模式的中間結果可能不完整，這是正常的。只使用 `is_final: true` 的結果來觸發 AI 處理。

**Q: 語音訊息太長怎麼辦？**

A: Whisper API 限制 25MB 的檔案大小（約 25 分鐘）。超長音訊會自動分段處理。

**Q: 可以同時辨識多種語言嗎？**

A: Whisper 支援自動語言偵測，但指定語言的準確率會更高。建議設定 `candidateLanguages` 來縮小範圍。
