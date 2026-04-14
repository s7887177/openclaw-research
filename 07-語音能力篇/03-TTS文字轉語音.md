# TTS 文字轉語音

## 目錄

- [1. TTS 基礎概念](#1-tts-基礎概念)
  - [1.1 什麼是 TTS](#11-什麼是-tts)
  - [1.2 TTS 在 OpenClaw 中的角色](#12-tts-在-openclaw-中的角色)
  - [1.3 現代 TTS 的演進](#13-現代-tts-的演進)
- [2. ElevenLabs](#2-elevenlabs)
  - [2.1 ElevenLabs 介紹](#21-elevenlabs-介紹)
  - [2.2 配置方式](#22-配置方式)
  - [2.3 語音選擇](#23-語音選擇)
  - [2.4 語音克隆](#24-語音克隆)
  - [2.5 串流合成](#25-串流合成)
- [3. OpenAI TTS](#3-openai-tts)
  - [3.1 OpenAI TTS 介紹](#31-openai-tts-介紹)
  - [3.2 配置方式](#32-配置方式)
  - [3.3 語音選擇](#33-語音選擇)
  - [3.4 高品質 vs 低延遲](#34-高品質-vs-低延遲)
- [4. Edge TTS](#4-edge-tts)
  - [4.1 Edge TTS 介紹](#41-edge-tts-介紹)
  - [4.2 配置方式](#42-配置方式)
  - [4.3 中文語音推薦](#43-中文語音推薦)
  - [4.4 SSML 支援](#44-ssml-支援)
- [5. MiniMax TTS](#5-minimax-tts)
  - [5.1 MiniMax 介紹](#51-minimax-介紹)
  - [5.2 配置方式](#52-配置方式)
  - [5.3 中文特化優勢](#53-中文特化優勢)
- [6. Azure Speech TTS](#6-azure-speech-tts)
  - [6.1 Azure TTS 介紹](#61-azure-tts-介紹)
  - [6.2 配置方式](#62-配置方式)
  - [6.3 SSML 進階用法](#63-ssml-進階用法)
- [7. [[tts]] 標記系統](#7-tts-標記系統)
  - [7.1 什麼是 [[tts]] 標記](#71-什麼是-tts-標記)
  - [7.2 使用方式](#72-使用方式)
  - [7.3 進階標記](#73-進階標記)
  - [7.4 條件語音](#74-條件語音)
- [8. 語音自定義](#8-語音自定義)
  - [8.1 語音克隆](#81-語音克隆)
  - [8.2 語音混合](#82-語音混合)
  - [8.3 情緒控制](#83-情緒控制)
- [9. 效能與成本](#9-效能與成本)
  - [9.1 延遲對比](#91-延遲對比)
  - [9.2 成本對比](#92-成本對比)
  - [9.3 快取策略](#93-快取策略)
- [10. 最佳實踐](#10-最佳實踐)

---

## 1. TTS 基礎概念

### 1.1 什麼是 TTS

TTS（Text-to-Speech）是將文字轉換為自然語音的技術。在 AI 助手的場景中，TTS 讓 AI 能夠「說話」，而不是只透過文字回覆。

### 1.2 TTS 在 OpenClaw 中的角色

```
ReAct 引擎輸出文字回應
    │
    ▼
┌──────────────────┐
│    TTS Engine     │
│                   │
│  文字 ─▶ 音訊     │
│                   │
│  ElevenLabs      │
│  OpenAI TTS      │
│  Edge TTS        │
│  MiniMax         │
│  Azure Speech    │
└──────────────────┘
    │
    ▼
音訊輸出到各通道
├── Discord 語音頻道（串流播放）
├── WhatsApp 語音訊息
├── Telegram 語音訊息
└── 本地揚聲器（Talk Mode）
```

### 1.3 現代 TTS 的演進

```
第一代：拼接式 TTS（1990s）
  → 聽起來很機械，像電腦在念稿

第二代：參數式 TTS（2010s）
  → 更流暢但仍有不自然感

第三代：神經網路 TTS（2020s）
  → 幾乎與真人無法區分
  → 支援情緒、語調控制
  → 支援語音克隆
  → 現在 OpenClaw 使用的技術
```

---

## 2. ElevenLabs

### 2.1 ElevenLabs 介紹

ElevenLabs 是目前語音合成品質最高的供應商之一，以其接近真人的語音品質和語音克隆能力著稱。

### 2.2 配置方式

```json
{
  "voice": {
    "tts": {
      "provider": "elevenlabs",
      "apiKey": "${ELEVENLABS_API_KEY}",
      "voiceId": "21m00Tcm4TlvDq8ikWAM",
      "modelId": "eleven_multilingual_v2",
      "stability": 0.5,
      "similarityBoost": 0.75,
      "style": 0,
      "outputFormat": "mp3_44100_128"
    }
  }
}
```

### 2.3 語音選擇

ElevenLabs 提供數千種預製語音，以及自定義語音克隆。推薦的中文語音：

```json
{
  "voices": {
    "rachel": {
      "id": "21m00Tcm4TlvDq8ikWAM",
      "description": "年輕女性，溫和清晰"
    },
    "adam": {
      "id": "pNInz6obpgDQGcFmaJgB",
      "description": "年輕男性，穩重自信"
    },
    "bella": {
      "id": "EXAVITQu4vr4xnSDxMaL",
      "description": "成熟女性，優雅沉穩"
    }
  }
}
```

### 2.4 語音克隆

ElevenLabs 的語音克隆功能可以用幾分鐘的語音樣本創建自定義語音：

```
Step 1: 準備語音樣本
  → 至少 1 分鐘的清晰語音
  → 建議 3-5 分鐘以獲得更好效果
  → 無背景噪音

Step 2: 上傳到 ElevenLabs
  → 在 ElevenLabs 平台建立新語音
  → 上傳語音樣本
  → 平台自動處理和建立語音模型

Step 3: 取得 Voice ID
  → 建立完成後獲得唯一的 Voice ID
  → 在 OpenClaw 配置中使用此 ID
```

### 2.5 串流合成

ElevenLabs 支援串流合成，適合 Talk Mode：

```python
# 串流 TTS 範例
async def stream_tts(text):
    async for chunk in elevenlabs.generate_stream(
        text=text,
        voice="Rachel",
        model="eleven_multilingual_v2",
        stream=True
    ):
        yield chunk  # 即時播放音訊片段
```

---

## 3. OpenAI TTS

### 3.1 OpenAI TTS 介紹

OpenAI TTS 提供了高品質的語音合成，以其簡單的 API 和合理的價格著稱。有兩個模型：

- `tts-1`：低延遲，適合即時對話
- `tts-1-hd`：高品質，適合預錄內容

### 3.2 配置方式

```json
{
  "voice": {
    "tts": {
      "provider": "openai-tts",
      "apiKey": "${OPENAI_API_KEY}",
      "model": "tts-1",
      "voice": "nova",
      "speed": 1.0,
      "responseFormat": "mp3"
    }
  }
}
```

### 3.3 語音選擇

OpenAI TTS 提供 6 種語音：

| 語音 | 特性 | 中文表現 |
|------|------|----------|
| `alloy` | 中性，平衡 | ⭐⭐⭐ |
| `echo` | 男性，沉穩 | ⭐⭐⭐ |
| `fable` | 男性，敘事 | ⭐⭐⭐ |
| `onyx` | 男性，低沉 | ⭐⭐⭐ |
| `nova` | 女性，活潑 | ⭐⭐⭐⭐ |
| `shimmer` | 女性，柔和 | ⭐⭐⭐⭐ |

**中文推薦**：`nova` 或 `shimmer`

### 3.4 高品質 vs 低延遲

```json
{
  "tts": {
    "talkMode": {
      "model": "tts-1",
      "comment": "低延遲，適合即時對話"
    },
    "messageReply": {
      "model": "tts-1-hd",
      "comment": "高品質，適合語音訊息回覆"
    }
  }
}
```

---

## 4. Edge TTS

### 4.1 Edge TTS 介紹

Edge TTS 是微軟 Edge 瀏覽器使用的 TTS 服務，最大的優勢是**完全免費**且品質相當不錯。對於預算有限但需要語音功能的場景，Edge TTS 是最佳選擇。

### 4.2 配置方式

```json
{
  "voice": {
    "tts": {
      "provider": "edge-tts",
      "voice": "zh-TW-HsiaoChenNeural",
      "rate": "+0%",
      "volume": "+0%",
      "pitch": "+0Hz"
    }
  }
}
```

注意：Edge TTS 不需要 API Key！

### 4.3 中文語音推薦

Edge TTS 提供多種高品質的中文語音：

| 語音 ID | 語言 | 性別 | 特點 |
|---------|------|------|------|
| `zh-TW-HsiaoChenNeural` | 繁體中文 | 女 | 自然流暢，推薦 |
| `zh-TW-YunJheNeural` | 繁體中文 | 男 | 沉穩清晰 |
| `zh-TW-HsiaoYuNeural` | 繁體中文 | 女 | 年輕活潑 |
| `zh-CN-XiaoxiaoNeural` | 簡體中文 | 女 | 非常自然 |
| `zh-CN-YunxiNeural` | 簡體中文 | 男 | 年輕親切 |
| `zh-HK-HiuMaanNeural` | 粵語 | 女 | 標準粵語 |

### 4.4 SSML 支援

Edge TTS 支援 SSML（Speech Synthesis Markup Language），可以精細控制語音：

```xml
<speak version="1.0" xmlns="http://www.w3.org/2001/10/synthesis" xml:lang="zh-TW">
  <voice name="zh-TW-HsiaoChenNeural">
    <prosody rate="+10%" pitch="+5%">
      你好！我是 EasonClaw。
    </prosody>
    <break time="500ms"/>
    <prosody rate="-10%" volume="soft">
      很高興認識你。
    </prosody>
  </voice>
</speak>
```

---

## 5. MiniMax TTS

### 5.1 MiniMax 介紹

MiniMax 是一家中國 AI 公司，其 TTS 服務在中文語音合成方面達到了頂尖水準。

### 5.2 配置方式

```json
{
  "voice": {
    "tts": {
      "provider": "minimax",
      "apiKey": "${MINIMAX_API_KEY}",
      "groupId": "${MINIMAX_GROUP_ID}",
      "model": "speech-01",
      "voiceId": "female-tianmei",
      "speed": 1.0,
      "vol": 1.0,
      "pitch": 0,
      "audioSampleRate": 24000,
      "bitrate": 128000
    }
  }
}
```

### 5.3 中文特化優勢

MiniMax 的中文 TTS 優勢：

- **聲音自然度極高**：接近真人的中文發音
- **情緒表達豐富**：支援多種情緒風格
- **多種角色語音**：提供不同年齡、性別的角色語音
- **語音克隆**：支援客製化語音
- **低延遲串流**：適合即時對話

---

## 6. Azure Speech TTS

### 6.1 Azure TTS 介紹

Azure Speech 的 TTS 是所有供應商中中文語音品質最高、功能最豐富的選擇。它提供 400+ 種語音，包括多種中文方言。

### 6.2 配置方式

```json
{
  "voice": {
    "tts": {
      "provider": "azure-speech",
      "subscriptionKey": "${AZURE_SPEECH_KEY}",
      "region": "eastasia",
      "voiceName": "zh-TW-HsiaoChenNeural",
      "outputFormat": "audio-24khz-48kbitrate-mono-mp3",
      "prosody": {
        "rate": "0%",
        "pitch": "0%",
        "volume": "100%"
      }
    }
  }
}
```

### 6.3 SSML 進階用法

Azure Speech 擁有最完整的 SSML 支援：

```xml
<speak version="1.0" xmlns="http://www.w3.org/2001/10/synthesis"
       xmlns:mstts="http://www.w3.org/2001/mstts" xml:lang="zh-TW">
  <voice name="zh-TW-HsiaoChenNeural">
    
    <!-- 風格控制 -->
    <mstts:express-as style="cheerful" styledegree="2">
      太好了！你的問題已經解決了！
    </mstts:express-as>
    
    <break time="300ms"/>
    
    <!-- 語調控制 -->
    <prosody rate="-10%" pitch="+5%">
      接下來我來說明一下步驟。
    </prosody>
    
    <!-- 靜音 -->
    <break time="500ms"/>
    
    <!-- 讀音指定 -->
    <phoneme alphabet="x-microsoft-pinyin" ph="kai1 yuan2">
      OpenClaw 是一個開源專案。
    </phoneme>
    
  </voice>
</speak>
```

Azure 語音風格：

| 風格 | 說明 |
|------|------|
| `cheerful` | 開心愉快 |
| `sad` | 悲傷低沉 |
| `angry` | 生氣憤怒 |
| `fearful` | 恐懼害怕 |
| `friendly` | 友善親切 |
| `serious` | 嚴肅認真 |
| `newscast` | 新聞播報 |

---

## 7. [[tts]] 標記系統

### 7.1 什麼是 [[tts]] 標記

OpenClaw 的 `[[tts]]` 標記是一個獨特的功能，讓你在 SKILL.md 或 System Prompt 中直接控制 TTS 行為。當 AI 的回應中包含 `[[tts]]` 標記時，該段文字會被特殊處理。

### 7.2 使用方式

在 SKILL.md 中：

```markdown
# 天氣查詢技能

## 回覆模板

查詢到天氣資訊後，請用以下格式回覆：

[[tts voice="cheerful"]]
今天 {city} 的天氣是 {weather}，溫度 {temperature} 度。
[[/tts]]

如果是壞天氣，使用擔心的語氣：

[[tts voice="concerned"]]
提醒你，今天 {city} 有 {weather}，記得帶傘喔。
[[/tts]]
```

### 7.3 進階標記

```markdown
<!-- 指定語音供應商和語音 -->
[[tts provider="elevenlabs" voice="Rachel"]]
這段話用 ElevenLabs 的 Rachel 語音
[[/tts]]

<!-- 控制語速和音調 -->
[[tts speed="1.2" pitch="+5%"]]
這段話稍微快一點、音調高一點
[[/tts]]

<!-- 插入停頓 -->
歡迎使用 EasonClaw！[[pause 500ms]] 有什麼我能幫你的嗎？

<!-- 指定不朗讀的部分 -->
[[no-tts]]
這段技術資訊只顯示文字，不會被朗讀：
- API endpoint: https://api.example.com
- Token: Bearer xxx
[[/no-tts]]

<!-- 拼音提示（處理多音字） -->
今天有一場重[[pinyin zhòng]]要的會議。
```

### 7.4 條件語音

根據上下文自動選擇語音風格：

```markdown
## 回覆規則

回覆時根據內容類型選擇語氣：
- 好消息 → [[tts style="cheerful"]]
- 壞消息 → [[tts style="empathetic"]]
- 技術說明 → [[tts style="professional"]]
- 日常閒聊 → [[tts style="friendly"]]
```

---

## 8. 語音自定義

### 8.1 語音克隆

使用 ElevenLabs 克隆自己的語音：

```json
{
  "voice": {
    "tts": {
      "provider": "elevenlabs",
      "customVoice": {
        "enabled": true,
        "voiceId": "your-cloned-voice-id",
        "name": "EasonVoice"
      }
    }
  }
}
```

### 8.2 語音混合

某些供應商支援語音混合，創造獨特的語音：

```json
{
  "voice": {
    "tts": {
      "voiceMix": {
        "enabled": true,
        "voices": [
          { "id": "voice1", "weight": 0.7 },
          { "id": "voice2", "weight": 0.3 }
        ]
      }
    }
  }
}
```

### 8.3 情緒控制

```json
{
  "voice": {
    "tts": {
      "emotionDetection": {
        "enabled": true,
        "autoAdapt": true,
        "mapping": {
          "happy": { "style": "cheerful", "speed": 1.1 },
          "sad": { "style": "empathetic", "speed": 0.9 },
          "excited": { "style": "enthusiastic", "speed": 1.2 },
          "calm": { "style": "gentle", "speed": 0.95 }
        }
      }
    }
  }
}
```

---

## 9. 效能與成本

### 9.1 延遲對比

| 供應商 | 首字節延遲 | 完整合成（100字） | 串流支援 |
|--------|-----------|-----------------|----------|
| ElevenLabs | ~300ms | ~1s | ✅ |
| OpenAI TTS | ~400ms | ~1.5s | ✅ |
| Edge TTS | ~500ms | ~2s | ✅ |
| MiniMax | ~300ms | ~1s | ✅ |
| Azure Speech | ~200ms | ~0.8s | ✅ |

### 9.2 成本對比

| 供應商 | 費用（每 1000 字元） | 免費額度 |
|--------|---------------------|----------|
| ElevenLabs | ~$0.30 | 10,000 字元/月 |
| OpenAI TTS | ~$0.015 | 無 |
| Edge TTS | 免費 | 無限 |
| MiniMax | ~$0.01 | 有（有限） |
| Azure Speech | ~$0.016 | 500K 字元/月 |

### 9.3 快取策略

對於重複的問候語或固定回覆，可以快取 TTS 結果：

```json
{
  "voice": {
    "tts": {
      "cache": {
        "enabled": true,
        "maxSize": "100MB",
        "maxAge": "7d",
        "cacheablePatterns": [
          "你好！*",
          "感謝你的*",
          "抱歉，*"
        ]
      }
    }
  }
}
```

---

## 10. 最佳實踐

**1. 選擇合適的供應商**

```
免費方案    → Edge TTS
高性價比    → OpenAI TTS
最佳中文    → MiniMax 或 Azure Speech
語音克隆    → ElevenLabs
即時對話    → OpenAI TTS（tts-1）或 ElevenLabs
```

**2. 為不同場景配置不同的 TTS**

```json
{
  "voice": {
    "tts": {
      "default": { "provider": "edge-tts" },
      "talkMode": { "provider": "openai-tts", "model": "tts-1" },
      "highQuality": { "provider": "elevenlabs" }
    }
  }
}
```

**3. 文字前處理**

在送入 TTS 之前，對文字進行處理：
- 移除 Markdown 格式標記
- 展開縮寫（API → A P I）
- 處理數字（2024 → 二零二四）
- 處理 URL（移除或簡化）
- 處理表情符號

**4. 控制回覆長度**

語音回覆應比文字回覆更簡短。建議在 System Prompt 中加入：

```
當用語音回覆時，請保持簡潔，每次回覆不超過 100 個字。
```
