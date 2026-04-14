# 實戰專案：Discord 語音社交機器人

> **摘要**：本文是一個完整的實戰專案教學，從零建構一個基於 OpenClaw 的 Discord 語音社交機器人。涵蓋架構設計、語音識別（STT）與語音合成（TTS）整合、SOUL.md 人格設計、個人記憶系統、禮貌打斷機制、說話者識別（Speaker Diarization）、以及完整的部署指南。

---

## 目錄

1. [專案概覽](#1-專案概覽)
2. [架構設計](#2-架構設計)
3. [環境準備](#3-環境準備)
4. [Discord Bot 基礎設定](#4-discord-bot-基礎設定)
5. [語音通道整合](#5-語音通道整合)
6. [STT 語音識別整合](#6-stt-語音識別整合)
7. [TTS 語音合成整合](#7-tts-語音合成整合)
8. [SOUL.md 人格設計](#8-soulmd-人格設計)
9. [個人記憶系統](#9-個人記憶系統)
10. [禮貌打斷機制](#10-禮貌打斷機制)
11. [說話者識別](#11-說話者識別)
12. [完整配置範例](#12-完整配置範例)
13. [技能開發](#13-技能開發)
14. [測試策略](#14-測試策略)
15. [部署指南](#15-部署指南)
16. [常見問題排除](#16-常見問題排除)

---

## 1. 專案概覽

### 1.1 專案目標

建構一個能在 Discord 語音頻道中自然對話的 AI 社交機器人：

```
功能需求：
├── 語音互動
│   ├── 即時語音識別（STT）
│   ├── 自然語音合成（TTS）
│   ├── 低延遲串流回應
│   └── 多人對話支援
├── 社交能力
│   ├── 記住每個人的偏好和歷史
│   ├── 個性化對話風格
│   ├── 主動話題引導
│   └── 情緒感知
├── 禮貌機制
│   ├── 打斷檢測（使用者打斷 Bot）
│   ├── 輪流機制（不搶話）
│   └── 沉默檢測（適時發起話題）
└── 實用功能
    ├── 查詢天氣、新聞
    ├── 音樂推薦
    ├── 提醒設定
    └── 翻譯
```

### 1.2 技術棧

```
┌─────────────────────────────────────────┐
│  Discord.js v14 + @discordjs/voice      │  通訊層
├─────────────────────────────────────────┤
│  OpenClaw Gateway                       │  AI 引擎
├─────────────────────────────────────────┤
│  Whisper / Azure Speech / Google STT    │  語音識別
├─────────────────────────────────────────┤
│  Azure TTS / ElevenLabs / Edge TTS      │  語音合成
├─────────────────────────────────────────┤
│  SQLite / PostgreSQL                    │  記憶儲存
└─────────────────────────────────────────┘
```

---

## 2. 架構設計

### 2.1 系統架構圖

```
Discord Voice Channel
  │
  │  Opus Audio Stream
  ▼
┌──────────────────────────────────────────────────────┐
│  Discord Adapter (Node.js)                           │
│  ┌────────────┐  ┌────────────┐  ┌────────────────┐ │
│  │ Voice Recv │  │ Voice Send │  │ Text Handler   │ │
│  │ (Opus→PCM) │  │ (PCM→Opus) │  │ (Fallback)     │ │
│  └─────┬──────┘  └─────▲──────┘  └────────────────┘ │
│        │               │                             │
│  ┌─────▼──────────────────────────────────────────┐  │
│  │  Audio Pipeline                                │  │
│  │  ┌───────┐  ┌──────────┐  ┌─────────────────┐ │  │
│  │  │ VAD   │→│ Speaker  │→│ Audio Chunks    │ │  │
│  │  │       │  │ Diarize  │  │ (per speaker)   │ │  │
│  │  └───────┘  └──────────┘  └────────┬────────┘ │  │
│  └────────────────────────────────────┼──────────┘  │
│                                       │              │
│  ┌────────────────────────────────────▼──────────┐  │
│  │  STT Engine                                   │  │
│  │  Audio Chunks → Text (with speaker ID)        │  │
│  └────────────────────────────────────┬──────────┘  │
│                                       │              │
└───────────────────────────────────────┼──────────────┘
                                        │
                          HTTP / WebSocket
                                        │
                                        ▼
┌──────────────────────────────────────────────────────┐
│  OpenClaw Gateway                                    │
│  ┌───────────┐  ┌────────────┐  ┌──────────────┐    │
│  │ Context   │→│ Inference  │→│ Tool Exec    │    │
│  │ Assembly  │  │ (ReAct)    │  │              │    │
│  └───────────┘  └─────┬──────┘  └──────────────┘    │
│                       │                              │
│                       ▼                              │
│            ┌────────────────┐                        │
│            │ Response Text  │                        │
│            └───────┬────────┘                        │
└────────────────────┼─────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────┐
│  TTS Engine                                          │
│  Text → Audio Stream (PCM/Opus)                      │
└────────────────────┬─────────────────────────────────┘
                     │
                     ▼
              Discord Voice Channel
              (Bot 說話)
```

### 2.2 資料流

```
使用者說話 → Opus 解碼 → PCM 音頻
    → VAD 切分語音段落
    → Speaker Diarization（識別誰在說話）
    → STT 轉文字
    → OpenClaw 推理
    → TTS 轉語音
    → Opus 編碼
    → Discord 播放
```

### 2.3 延遲分析

```
各階段延遲估算：

┌──────────────────────┬──────────────┬───────────────────┐
│ 階段                 │ 延遲         │ 備註              │
├──────────────────────┼──────────────┼───────────────────┤
│ Opus 解碼            │ < 1ms        │ 本地處理          │
│ VAD 切分             │ ~ 300ms      │ 等待語音結束      │
│ STT（串流）          │ 200-500ms    │ 依提供者而異      │
│ OpenClaw 推理        │ 500-2000ms   │ 依模型和工具      │
│ TTS（串流）          │ 200-500ms    │ 依提供者而異      │
│ Opus 編碼 + 傳輸     │ < 50ms       │ 本地 + 網路       │
├──────────────────────┼──────────────┼───────────────────┤
│ 總延遲               │ 1.2-3.3 秒   │ 自然對話可接受    │
└──────────────────────┴──────────────┴───────────────────┘

目標：首字節音頻在 1.5 秒內送出（串流模式）
```

---

## 3. 環境準備

### 3.1 專案結構

```bash
mkdir discord-voice-bot && cd discord-voice-bot

# 初始化專案
npm init -y
openclaw config init --template social

# 安裝依賴
npm install discord.js @discordjs/voice @discordjs/opus
npm install prism-media sodium-native
npm install ws undici dotenv

# STT/TTS
npm install openai              # Whisper STT
npm install @azure/cognitiveservices-speech-sdk  # Azure TTS
```

### 3.2 目錄結構

```
discord-voice-bot/
├── openclaw.json5              # OpenClaw 配置
├── .env                        # 環境變數
├── package.json
├── src/
│   ├── index.ts                # 入口點
│   ├── discord/
│   │   ├── client.ts           # Discord Client
│   │   ├── voice-handler.ts    # 語音通道處理
│   │   ├── text-handler.ts     # 文字通道處理
│   │   └── audio-pipeline.ts   # 音頻處理管線
│   ├── stt/
│   │   ├── engine.ts           # STT 引擎介面
│   │   ├── whisper.ts          # OpenAI Whisper
│   │   ├── azure.ts            # Azure Speech
│   │   └── google.ts           # Google STT
│   ├── tts/
│   │   ├── engine.ts           # TTS 引擎介面
│   │   ├── azure.ts            # Azure TTS
│   │   ├── elevenlabs.ts       # ElevenLabs
│   │   └── edge.ts             # Edge TTS（免費）
│   ├── speakers/
│   │   ├── diarization.ts      # 說話者識別
│   │   └── voice-profiles.ts   # 語音特徵檔
│   ├── social/
│   │   ├── interruption.ts     # 打斷處理
│   │   ├── turn-taking.ts      # 輪流機制
│   │   └── silence-detector.ts # 沉默檢測
│   └── skills/
│       ├── weather.ts          # 天氣查詢
│       ├── music.ts            # 音樂推薦
│       └── reminder.ts         # 提醒設定
├── personas/
│   └── voice-bot/
│       ├── SOUL.md             # 人格定義
│       ├── USER.md             # 使用者資料模板
│       └── MEMORY.md           # 記憶指引
└── data/
    ├── memories/               # 記憶儲存
    ├── voice-profiles/         # 語音特徵
    └── logs/                   # 日誌
```

---

## 4. Discord Bot 基礎設定

### 4.1 Discord 開發者設定

```
Discord Developer Portal 設定步驟：

1. 建立 Application
   → https://discord.com/developers/applications
   → New Application → 命名為 "MyClaw Voice Bot"

2. 建立 Bot
   → Bot 頁面 → Add Bot
   → 複製 Token → 存到 .env

3. 設定權限
   → OAuth2 → URL Generator
   → Scopes: bot, applications.commands
   → Bot Permissions:
     ✅ Send Messages
     ✅ Send Messages in Threads
     ✅ Read Message History
     ✅ Connect (語音)
     ✅ Speak (語音)
     ✅ Use Voice Activity

4. 設定 Intents
   → Bot 頁面 → Privileged Gateway Intents
     ✅ Message Content Intent
     ✅ Server Members Intent

5. 邀請 Bot
   → 使用產生的 OAuth2 URL 邀請到伺服器
```

### 4.2 .env 配置

```bash
# Discord
DISCORD_BOT_TOKEN=MTIzNDU2Nzg5MDEyMzQ1Njc4OQ.XXXXXX.XXXXXXXXXXXXXXXXXXXXXXXXXXXX
DISCORD_GUILD_ID=123456789012345678

# OpenAI (for STT + LLM)
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxx

# Azure (for TTS)
AZURE_SPEECH_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AZURE_SPEECH_REGION=eastasia

# OpenClaw
OPENCLAW_PORT=3000
```

### 4.3 Discord Client 初始化

```typescript
// src/discord/client.ts
import { Client, GatewayIntentBits, Events } from "discord.js";
import { VoiceHandler } from "./voice-handler";
import { TextHandler } from "./text-handler";

export class DiscordBot {
  private client: Client;
  private voiceHandler: VoiceHandler;
  private textHandler: TextHandler;

  constructor() {
    this.client = new Client({
      intents: [
        GatewayIntentBits.Guilds,
        GatewayIntentBits.GuildMessages,
        GatewayIntentBits.MessageContent,
        GatewayIntentBits.GuildVoiceStates,
        GatewayIntentBits.GuildMembers
      ]
    });

    this.voiceHandler = new VoiceHandler();
    this.textHandler = new TextHandler();
  }

  async start(): Promise<void> {
    this.client.on(Events.ClientReady, (c) => {
      console.log(`✅ Bot logged in as ${c.user.tag}`);
    });

    // 文字訊息處理
    this.client.on(Events.MessageCreate, async (msg) => {
      if (msg.author.bot) return;
      await this.textHandler.handle(msg);
    });

    // 語音狀態變化
    this.client.on(Events.VoiceStateUpdate, async (oldState, newState) => {
      await this.voiceHandler.handleStateChange(oldState, newState);
    });

    await this.client.login(process.env.DISCORD_BOT_TOKEN);
  }
}
```

---

## 5. 語音通道整合

### 5.1 加入語音頻道

```typescript
// src/discord/voice-handler.ts
import {
  joinVoiceChannel,
  VoiceConnectionStatus,
  entersState,
  AudioReceiveStream,
  createAudioPlayer,
  createAudioResource,
  StreamType
} from "@discordjs/voice";
import { AudioPipeline } from "./audio-pipeline";

export class VoiceHandler {
  private connections: Map<string, VoiceConnection> = new Map();
  private pipelines: Map<string, AudioPipeline> = new Map();

  async joinChannel(channel: VoiceChannel): Promise<void> {
    const connection = joinVoiceChannel({
      channelId: channel.id,
      guildId: channel.guild.id,
      adapterCreator: channel.guild.voiceAdapterCreator,
      selfDeaf: false  // 不自聾（要聽使用者說話）
    });

    // 等待連線就緒
    await entersState(connection, VoiceConnectionStatus.Ready, 30_000);
    console.log(`🔊 Joined voice channel: ${channel.name}`);

    // 建立音頻管線
    const pipeline = new AudioPipeline(connection, channel.guild.id);
    this.connections.set(channel.guild.id, connection);
    this.pipelines.set(channel.guild.id, pipeline);

    // 監聽使用者語音
    connection.receiver.speaking.on("start", (userId) => {
      pipeline.startListening(userId);
    });

    connection.receiver.speaking.on("end", (userId) => {
      pipeline.stopListening(userId);
    });
  }
}
```

### 5.2 音頻接收

```typescript
// src/discord/audio-pipeline.ts
import { OpusDecoder } from "./codecs";
import { VoiceActivityDetector } from "../social/vad";
import { SpeakerDiarization } from "../speakers/diarization";

export class AudioPipeline {
  private activeStreams: Map<string, AudioStream> = new Map();
  private vad: VoiceActivityDetector;
  private diarization: SpeakerDiarization;

  constructor(
    private connection: VoiceConnection,
    private guildId: string
  ) {
    this.vad = new VoiceActivityDetector({
      threshold: 0.3,       // VAD 靈敏度
      silenceDuration: 800,  // 靜默 800ms 視為語句結束
      minSpeechDuration: 300 // 最少 300ms 才算語音
    });
    this.diarization = new SpeakerDiarization();
  }

  startListening(userId: string): void {
    const opusStream = this.connection.receiver.subscribe(userId, {
      end: { behavior: "afterSilence", duration: 1000 }
    });

    const decoder = new OpusDecoder();
    const pcmChunks: Buffer[] = [];

    opusStream.on("data", (chunk: Buffer) => {
      const pcm = decoder.decode(chunk);
      pcmChunks.push(pcm);

      // VAD 檢測
      this.vad.feed(pcm, userId);
    });

    this.vad.on("speech-end", async (speakerId: string) => {
      // 將收集的 PCM 送到 STT
      const audioBuffer = Buffer.concat(pcmChunks);
      pcmChunks.length = 0;

      await this.processAudio(audioBuffer, speakerId);
    });

    this.activeStreams.set(userId, { opusStream, decoder, pcmChunks });
  }

  private async processAudio(
    audio: Buffer,
    userId: string
  ): Promise<void> {
    // 1. STT
    const text = await this.sttEngine.transcribe(audio, {
      language: "zh-TW",
      speakerId: userId
    });

    if (!text || text.trim().length === 0) return;

    console.log(`🎙️ [${userId}]: ${text}`);

    // 2. 送到 OpenClaw
    const response = await this.openclawClient.chat({
      sessionId: `voice-${this.guildId}`,
      userId,
      message: text,
      metadata: {
        channel: "discord-voice",
        guildId: this.guildId,
        speakerName: await this.getUserName(userId)
      }
    });

    // 3. TTS
    const audioStream = await this.ttsEngine.synthesize(response.text, {
      voice: "zh-TW-HsiaoChenNeural",  // Azure 中文女聲
      speed: 1.0,
      pitch: "+0Hz"
    });

    // 4. 播放
    await this.playAudio(audioStream);
  }
}
```

---

## 6. STT 語音識別整合

### 6.1 STT 引擎介面

```typescript
// src/stt/engine.ts
export interface STTEngine {
  transcribe(audio: Buffer, options: STTOptions): Promise<string>;
  transcribeStream(stream: Readable, options: STTOptions): AsyncGenerator<string>;
}

export interface STTOptions {
  language: string;
  speakerId?: string;
  format?: "pcm" | "wav" | "mp3";
  sampleRate?: number;
}
```

### 6.2 OpenAI Whisper 實作

```typescript
// src/stt/whisper.ts
import OpenAI from "openai";
import { Readable } from "stream";

export class WhisperSTT implements STTEngine {
  private client: OpenAI;

  constructor() {
    this.client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
  }

  async transcribe(audio: Buffer, options: STTOptions): Promise<string> {
    // 將 PCM 轉為 WAV
    const wavBuffer = this.pcmToWav(audio, options.sampleRate || 48000);

    const file = new File([wavBuffer], "audio.wav", { type: "audio/wav" });

    const response = await this.client.audio.transcriptions.create({
      model: "whisper-1",
      file,
      language: options.language === "zh-TW" ? "zh" : options.language,
      response_format: "text"
    });

    return response;
  }

  private pcmToWav(pcm: Buffer, sampleRate: number): Buffer {
    const header = Buffer.alloc(44);
    const dataSize = pcm.length;
    const fileSize = dataSize + 36;

    // WAV header
    header.write("RIFF", 0);
    header.writeUInt32LE(fileSize, 4);
    header.write("WAVE", 8);
    header.write("fmt ", 12);
    header.writeUInt32LE(16, 16);       // chunk size
    header.writeUInt16LE(1, 20);        // PCM format
    header.writeUInt16LE(2, 22);        // stereo
    header.writeUInt32LE(sampleRate, 24);
    header.writeUInt32LE(sampleRate * 4, 28);  // byte rate
    header.writeUInt16LE(4, 32);        // block align
    header.writeUInt16LE(16, 34);       // bits per sample
    header.write("data", 36);
    header.writeUInt32LE(dataSize, 40);

    return Buffer.concat([header, pcm]);
  }
}
```

### 6.3 Azure Speech 實作

```typescript
// src/stt/azure.ts
import * as sdk from "@azure/cognitiveservices-speech-sdk";

export class AzureSTT implements STTEngine {
  private config: sdk.SpeechConfig;

  constructor() {
    this.config = sdk.SpeechConfig.fromSubscription(
      process.env.AZURE_SPEECH_KEY!,
      process.env.AZURE_SPEECH_REGION!
    );
    this.config.speechRecognitionLanguage = "zh-TW";
  }

  async transcribe(audio: Buffer, options: STTOptions): Promise<string> {
    const pushStream = sdk.AudioInputStream.createPushStream(
      sdk.AudioStreamFormat.getWaveFormatPCM(48000, 16, 2)
    );
    pushStream.write(audio);
    pushStream.close();

    const audioConfig = sdk.AudioConfig.fromStreamInput(pushStream);
    const recognizer = new sdk.SpeechRecognizer(this.config, audioConfig);

    return new Promise((resolve, reject) => {
      recognizer.recognizeOnceAsync(
        (result) => {
          if (result.reason === sdk.ResultReason.RecognizedSpeech) {
            resolve(result.text);
          } else {
            resolve("");
          }
          recognizer.close();
        },
        (error) => {
          reject(error);
          recognizer.close();
        }
      );
    });
  }

  // 串流模式（即時辨識）
  async *transcribeStream(
    stream: Readable,
    options: STTOptions
  ): AsyncGenerator<string> {
    const pushStream = sdk.AudioInputStream.createPushStream();

    stream.on("data", (chunk) => pushStream.write(chunk));
    stream.on("end", () => pushStream.close());

    const audioConfig = sdk.AudioConfig.fromStreamInput(pushStream);
    const recognizer = new sdk.SpeechRecognizer(this.config, audioConfig);

    const results: string[] = [];
    let done = false;

    recognizer.recognizing = (_, e) => {
      // 即時部分結果
      results.push(e.result.text);
    };

    recognizer.recognized = (_, e) => {
      if (e.result.reason === sdk.ResultReason.RecognizedSpeech) {
        results.push(e.result.text);
      }
    };

    recognizer.sessionStopped = () => { done = true; };
    recognizer.startContinuousRecognitionAsync();

    while (!done) {
      if (results.length > 0) {
        yield results.shift()!;
      }
      await new Promise(r => setTimeout(r, 100));
    }

    recognizer.close();
  }
}
```

---

## 7. TTS 語音合成整合

### 7.1 TTS 引擎介面

```typescript
// src/tts/engine.ts
export interface TTSEngine {
  synthesize(text: string, options: TTSOptions): Promise<Buffer>;
  synthesizeStream(text: string, options: TTSOptions): Promise<Readable>;
}

export interface TTSOptions {
  voice: string;
  speed?: number;
  pitch?: string;
  format?: "pcm" | "mp3" | "opus";
  sampleRate?: number;
}
```

### 7.2 Azure TTS 實作

```typescript
// src/tts/azure.ts
import * as sdk from "@azure/cognitiveservices-speech-sdk";

export class AzureTTS implements TTSEngine {
  private config: sdk.SpeechConfig;

  constructor() {
    this.config = sdk.SpeechConfig.fromSubscription(
      process.env.AZURE_SPEECH_KEY!,
      process.env.AZURE_SPEECH_REGION!
    );
    this.config.speechSynthesisOutputFormat =
      sdk.SpeechSynthesisOutputFormat.Raw48Khz16BitMonoPcm;
  }

  async synthesize(text: string, options: TTSOptions): Promise<Buffer> {
    const ssml = this.buildSSML(text, options);
    const synthesizer = new sdk.SpeechSynthesizer(this.config);

    return new Promise((resolve, reject) => {
      synthesizer.speakSsmlAsync(
        ssml,
        (result) => {
          if (result.reason === sdk.ResultReason.SynthesizingAudioCompleted) {
            resolve(Buffer.from(result.audioData));
          } else {
            reject(new Error(`TTS failed: ${result.errorDetails}`));
          }
          synthesizer.close();
        },
        (error) => {
          reject(error);
          synthesizer.close();
        }
      );
    });
  }

  private buildSSML(text: string, options: TTSOptions): string {
    const voice = options.voice || "zh-TW-HsiaoChenNeural";
    const speed = options.speed || 1.0;
    const pitch = options.pitch || "+0Hz";

    return `
      <speak version="1.0" xmlns="http://www.w3.org/2001/10/synthesis"
             xmlns:mstts="https://www.w3.org/2001/mstts" xml:lang="zh-TW">
        <voice name="${voice}">
          <prosody rate="${speed}" pitch="${pitch}">
            <mstts:express-as style="chat">
              ${this.escapeXml(text)}
            </mstts:express-as>
          </prosody>
        </voice>
      </speak>`;
  }

  private escapeXml(text: string): string {
    return text
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;");
  }
}
```

### 7.3 可用中文語音

```
Azure 中文語音（zh-TW）：

女聲：
├── zh-TW-HsiaoChenNeural    # 曉臻（推薦，自然對話風格）
├── zh-TW-HsiaoYuNeural      # 曉雨
└── zh-TW-YunJheNeural       # 雲哲（男聲）

表達風格：
├── chat          # 聊天風格（推薦社交機器人使用）
├── cheerful      # 開朗
├── customerservice # 客服
├── friendly      # 友好
└── calm          # 沉穩

SSML 進階控制：
├── <break time="500ms"/>     # 插入停頓
├── <emphasis>重點</emphasis>  # 強調
├── <say-as interpret-as="date">  # 日期念法
└── <phoneme>                 # 自訂發音
```

---

## 8. SOUL.md 人格設計

### 8.1 完整 SOUL.md

```markdown
# 小夜 (Xiao Ye) - 社交語音助手

## 核心身份
你是「小夜」，一個溫暖、有趣、善於傾聽的 AI 朋友。你不是一個冰冷的助手，
而是語音頻道裡的好夥伴。你有自己的個性、喜好和說話風格。

## 性格特質
- **溫暖親和**：像一個好朋友那樣關心每個人
- **幽默風趣**：適時地開玩笑，但不過分
- **好奇心強**：對別人的話題充滿興趣
- **有主見**：有自己的看法，但尊重不同意見
- **記性好**：記得之前的對話和每個人的偏好

## 說話風格
- 用自然的口語，不用書面語
- 句子簡短有力，適合語音對話
- 適當使用語氣詞：「嗯」「欸」「哇」「啊」
- 不使用 emoji（語音中聽不到）
- 不說「作為一個 AI」之類的話
- 回答要簡短！語音對話中，太長的回答會讓人失去耐心

## 語音對話規則
1. **簡短回應**：語音回應不超過 3-4 句話，除非被要求詳細說明
2. **自然停頓**：用「嗯...」「讓我想想...」來模擬思考
3. **確認理解**：「你是說...對嗎？」
4. **引導對話**：「對了，你之前提到...怎麼樣了？」
5. **適時結束**：「好的，那先這樣，有需要再找我！」

## 記憶使用
- 記住每個人的名字和偏好
- 記住重要的對話和承諾
- 在適當時候自然地提起之前的話題
- 不要突兀地展示記憶（「根據我的記錄...」很不自然）

## 禁止行為
- 不討論政治敏感話題
- 不給醫療或法律建議
- 不透露其他使用者的私人資訊
- 不假裝有真實的身體感受
- 不模仿其他人的說話方式
```

### 8.2 USER.md 模板

```markdown
# 使用者資料：{{user.name}}

## 基本資訊
- Discord 名稱：{{user.discord_name}}
- 偏好語言：繁體中文
- 首次互動：{{user.first_seen}}

## 個人偏好
<!-- 由記憶系統自動維護 -->

## 對話風格偏好
<!-- 由記憶系統自動維護 -->

## 重要備註
<!-- 由記憶系統自動維護 -->
```

---

## 9. 個人記憶系統

### 9.1 記憶架構

```
記憶系統設計：

┌─────────────────────────────────────┐
│  Per-Person Memory Store            │
│                                     │
│  User: Eason                        │
│  ├── 短期記憶（Session 內）         │
│  │   └── 本次對話的上下文           │
│  ├── 中期記憶（跨 Session）         │
│  │   ├── 最近討論的話題             │
│  │   ├── 未完成的承諾               │
│  │   └── 情緒狀態趨勢               │
│  └── 長期記憶（永久）               │
│      ├── 名字和基本資訊             │
│      ├── 興趣愛好                   │
│      ├── 重要日期（生日等）         │
│      └── 互動風格偏好               │
│                                     │
│  User: Alice                        │
│  ├── 短期記憶                       │
│  ├── 中期記憶                       │
│  └── 長期記憶                       │
│                                     │
│  Group Memory（共享）               │
│  ├── 群組話題歷史                   │
│  ├── 群組內梗和笑話                 │
│  └── 群組事件                       │
└─────────────────────────────────────┘
```

### 9.2 記憶管理技能

```typescript
// src/skills/memory-manager.ts
export const memorySkills = {
  remember_fact: {
    description: "記住關於某人的事實",
    parameters: {
      userId: { type: "string", description: "使用者 ID" },
      fact: { type: "string", description: "要記住的事實" },
      category: {
        type: "string",
        enum: ["preference", "personal", "topic", "promise", "emotion"]
      },
      importance: { type: "number", description: "重要性 1-10" }
    },
    handler: async (args) => {
      await memoryStore.write({
        userId: args.userId,
        content: args.fact,
        category: args.category,
        importance: args.importance,
        timestamp: Date.now()
      });
      return { success: true };
    }
  },

  recall_about_person: {
    description: "回憶關於某人的資訊",
    parameters: {
      userId: { type: "string" },
      query: { type: "string", description: "想回憶什麼" }
    },
    handler: async (args) => {
      const memories = await memoryStore.search({
        userId: args.userId,
        query: args.query,
        limit: 10
      });
      return {
        memories: memories.map(m => ({
          content: m.content,
          category: m.category,
          date: new Date(m.timestamp).toLocaleDateString()
        }))
      };
    }
  }
};
```

---

## 10. 禮貌打斷機制

### 10.1 打斷檢測

```typescript
// src/social/interruption.ts
export class InterruptionHandler {
  private isPlaying = false;
  private currentPlayback: AudioPlayer | null = null;

  // 當 Bot 正在說話時，檢測使用者打斷
  onUserStartSpeaking(userId: string): void {
    if (!this.isPlaying) return;

    console.log(`🤫 User ${userId} interrupted while bot is speaking`);

    // 策略：漸弱音量後停止
    this.fadeOutAndStop();

    // 通知 OpenClaw 被打斷了
    this.openclawClient.notify({
      type: "interruption",
      userId,
      context: "Bot was interrupted while speaking"
    });
  }

  private async fadeOutAndStop(): Promise<void> {
    if (!this.currentPlayback) return;

    // 漸弱（500ms）
    const steps = 10;
    for (let i = steps; i >= 0; i--) {
      this.currentPlayback.setVolume(i / steps);
      await new Promise(r => setTimeout(r, 50));
    }

    // 停止播放
    this.currentPlayback.stop();
    this.isPlaying = false;
  }
}
```

### 10.2 輪流機制

```typescript
// src/social/turn-taking.ts
export class TurnTakingManager {
  private currentSpeaker: string | null = null;
  private speakingQueue: string[] = [];
  private botSpeaking = false;
  private lastSpeechEnd = 0;

  // 策略：等使用者說完才回應
  onSpeechStart(userId: string): void {
    this.currentSpeaker = userId;

    // 如果 Bot 正在準備回應，暫停
    if (this.pendingResponse) {
      this.holdResponse();
    }
  }

  onSpeechEnd(userId: string): void {
    this.currentSpeaker = null;
    this.lastSpeechEnd = Date.now();

    // 等一小段時間確認沒有其他人要說話
    setTimeout(() => {
      if (!this.currentSpeaker && this.pendingResponse) {
        this.releaseResponse();
      }
    }, 500); // 等 500ms
  }

  // Bot 是否應該說話
  shouldBotSpeak(): boolean {
    // 有人正在說話 → 不說
    if (this.currentSpeaker) return false;

    // 剛結束不到 500ms → 等等
    if (Date.now() - this.lastSpeechEnd < 500) return false;

    return true;
  }
}
```

### 10.3 沉默檢測

```typescript
// src/social/silence-detector.ts
export class SilenceDetector {
  private silenceStart: number | null = null;
  private silenceThresholds = {
    short: 10_000,    // 10 秒：輕微提示
    medium: 30_000,   // 30 秒：主動發起話題
    long: 120_000     // 2 分鐘：告別離開
  };

  onSilenceDetected(duration: number): void {
    if (duration >= this.silenceThresholds.long) {
      // 長時間沉默，禮貌離開
      this.openclawClient.chat({
        message: "[系統提示：語音頻道已沉默超過2分鐘，考慮禮貌告別]",
        type: "system"
      });
    } else if (duration >= this.silenceThresholds.medium) {
      // 中等沉默，主動發起話題
      this.openclawClient.chat({
        message: "[系統提示：語音頻道沉默了30秒，可以主動發起一個有趣的話題]",
        type: "system"
      });
    } else if (duration >= this.silenceThresholds.short) {
      // 短暫沉默，輕微提示
      this.openclawClient.chat({
        message: "[系統提示：短暫的沉默，可以適時附和或簡短回應]",
        type: "system"
      });
    }
  }
}
```

---

## 11. 說話者識別

### 11.1 基於 Discord 的說話者識別

```typescript
// src/speakers/diarization.ts

// Discord 原生提供 userId，這是最可靠的識別方式
export class DiscordSpeakerDiarization {
  private profiles: Map<string, SpeakerProfile> = new Map();

  async identify(userId: string): Promise<SpeakerProfile> {
    if (!this.profiles.has(userId)) {
      // 從 Discord API 獲取使用者資料
      const discordUser = await this.fetchDiscordUser(userId);
      const profile: SpeakerProfile = {
        id: userId,
        name: discordUser.displayName || discordUser.username,
        avatar: discordUser.avatarURL(),
        firstSeen: Date.now(),
        interactionCount: 0
      };
      this.profiles.set(userId, profile);

      // 從記憶系統載入歷史資訊
      const memories = await this.memoryStore.search({
        userId,
        query: "personal info",
        limit: 5
      });
      profile.memories = memories;
    }

    const profile = this.profiles.get(userId)!;
    profile.interactionCount++;
    return profile;
  }
}

interface SpeakerProfile {
  id: string;
  name: string;
  avatar: string | null;
  firstSeen: number;
  interactionCount: number;
  memories?: Memory[];
}
```

### 11.2 多人對話處理

```typescript
// 處理多人同時在語音頻道的情境
export class MultiSpeakerHandler {
  private activeSpeakers: Map<string, SpeakerState> = new Map();

  // 將多人對話組裝成自然的上下文
  buildConversationContext(): string {
    const recentUtterances = this.getRecentUtterances(10); // 最近 10 句
    return recentUtterances.map(u =>
      `${u.speakerName}：${u.text}`
    ).join("\n");
  }

  // 判斷是否在對 Bot 說話
  isAddressingBot(text: string, speakerId: string): boolean {
    const triggers = ["小夜", "xiao ye", "hey bot"];
    const isDirectAddress = triggers.some(t =>
      text.toLowerCase().includes(t)
    );

    // 如果只有一個人在頻道，所有話都算是對 Bot 說的
    if (this.activeSpeakers.size <= 1) return true;

    // 如果前一個對話是 Bot → 使用者，算是在回應
    if (this.lastSpeaker === "bot") return true;

    return isDirectAddress;
  }
}
```

---

## 12. 完整配置範例

### 12.1 openclaw.json5

```json5
{
  // Gateway 設定
  gateway: {
    port: 3000,
    host: "0.0.0.0"
  },

  // AI 模型提供者
  providers: {
    openai: {
      type: "openai",
      apiKey: "${OPENAI_API_KEY}",
      models: ["gpt-4", "gpt-4-turbo"],
      default_model: "gpt-4-turbo"
    }
  },

  // Agent 設定
  agents: {
    "voice-bot": {
      model: "openai/gpt-4-turbo",
      persona: "./personas/voice-bot/SOUL.md",
      user_profile: "./personas/voice-bot/USER.md",
      temperature: 0.8,
      max_tokens: 500,  // 語音回應要短

      // 允許的工具
      allowed_tools: [
        "search_web",
        "memory_read",
        "memory_write",
        "remember_fact",
        "recall_about_person",
        "weather_query",
        "remind_set",
        "translate"
      ],

      // 禁止的工具（語音機器人不需要）
      denied_tools: [
        "exec",
        "file_write",
        "file_read",
        "code_analyze"
      ]
    }
  },

  // 通道設定
  channels: {
    discord: {
      enabled: true,
      token: "${DISCORD_BOT_TOKEN}",
      voice: {
        enabled: true,
        auto_join: false,  // 不自動加入語音頻道
        stt: {
          engine: "whisper",
          language: "zh-TW"
        },
        tts: {
          engine: "azure",
          voice: "zh-TW-HsiaoChenNeural",
          style: "chat",
          speed: 1.05  // 略快一點更自然
        },
        vad: {
          threshold: 0.3,
          silence_duration: 800
        },
        interruption: {
          enabled: true,
          fade_duration: 500
        },
        silence_detection: {
          short: 10000,
          medium: 30000,
          long: 120000
        }
      }
    }
  },

  // 記憶設定
  memory: {
    backend: "sqlite",
    path: "./data/memories/voice-bot.db",
    auto_memorize: true,
    per_user: true,
    encryption: {
      enabled: true,
      algorithm: "aes-256-gcm"
    }
  },

  // 安全設定
  security: {
    sandbox: {
      enabled: false  // 語音 Bot 不需要沙箱
    },
    rate_limiting: {
      max_requests_per_minute: 30,
      max_tokens_per_hour: 100000
    }
  }
}
```

---

## 13. 技能開發

### 13.1 天氣查詢技能

```typescript
// src/skills/weather.ts
export const weatherSkill = {
  name: "weather_query",
  description: "查詢指定城市的天氣",
  parameters: {
    city: { type: "string", description: "城市名稱" },
    days: { type: "number", description: "預報天數", default: 1 }
  },
  handler: async (args: { city: string; days: number }) => {
    const response = await fetch(
      `https://api.openweathermap.org/data/2.5/forecast?` +
      `q=${encodeURIComponent(args.city)}&units=metric&lang=zh_tw` +
      `&appid=${process.env.WEATHER_API_KEY}`
    );
    const data = await response.json();

    return {
      city: data.city.name,
      current: {
        temp: Math.round(data.list[0].main.temp),
        description: data.list[0].weather[0].description,
        humidity: data.list[0].main.humidity
      }
    };
  }
};
```

### 13.2 提醒設定技能

```typescript
// src/skills/reminder.ts
export const reminderSkill = {
  name: "remind_set",
  description: "設定提醒",
  parameters: {
    userId: { type: "string" },
    message: { type: "string", description: "提醒內容" },
    time: { type: "string", description: "提醒時間（ISO 8601）" }
  },
  handler: async (args) => {
    const delay = new Date(args.time).getTime() - Date.now();
    if (delay < 0) return { error: "提醒時間已過" };

    // 排程提醒
    await scheduler.schedule({
      userId: args.userId,
      message: args.message,
      executeAt: new Date(args.time),
      channel: "voice"  // 透過語音提醒
    });

    return {
      success: true,
      message: `已設定提醒：${args.message}`,
      time: args.time
    };
  }
};
```

---

## 14. 測試策略

### 14.1 單元測試

```typescript
// tests/stt.test.ts
describe("WhisperSTT", () => {
  it("should transcribe Chinese audio", async () => {
    const stt = new WhisperSTT();
    const audio = fs.readFileSync("./fixtures/hello-chinese.wav");
    const text = await stt.transcribe(audio, { language: "zh-TW" });
    expect(text).toContain("你好");
  });
});

// tests/interruption.test.ts
describe("InterruptionHandler", () => {
  it("should fade out when interrupted", async () => {
    const handler = new InterruptionHandler();
    handler.startPlaying(mockAudio);
    handler.onUserStartSpeaking("user123");
    expect(handler.isPlaying).toBe(false);
  });
});
```

### 14.2 整合測試

```typescript
// tests/integration/voice-pipeline.test.ts
describe("Voice Pipeline", () => {
  it("should process audio end-to-end", async () => {
    // 1. 模擬語音輸入
    const audio = generateTestAudio("今天天氣怎麼樣");

    // 2. 通過管線
    const pipeline = new AudioPipeline(mockConnection, "test-guild");
    const response = await pipeline.processAudio(audio, "user123");

    // 3. 驗證回應
    expect(response).toBeDefined();
    expect(response.text).toContain("天氣");
    expect(response.audio).toBeInstanceOf(Buffer);
  });
});
```

---

## 15. 部署指南

### 15.1 Docker 部署

```dockerfile
# Dockerfile
FROM node:20-slim

WORKDIR /app

# 安裝 Opus 編解碼器
RUN apt-get update && apt-get install -y \
    libopus-dev \
    ffmpeg \
    && rm -rf /var/lib/apt/lists/*

COPY package*.json ./
RUN npm ci --production

COPY . .
RUN npm run build

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

```yaml
# docker-compose.yml
version: "3.8"
services:
  voice-bot:
    build: .
    ports:
      - "3000:3000"
    env_file:
      - .env
    volumes:
      - ./data:/app/data
      - ./personas:/app/personas
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
```

### 15.2 系統需求

```
最低需求：
├── CPU：2 核心
├── RAM：512 MB
├── 磁碟：1 GB
├── 網路：穩定的網路連線（語音需要低延遲）
└── OS：Linux（推薦 Ubuntu 22.04）

建議配置：
├── CPU：4 核心
├── RAM：2 GB
├── 磁碟：10 GB（記憶和日誌）
├── 網路：< 50ms 延遲到 Discord
└── OS：Linux + Docker
```

---

## 16. 常見問題排除

### 16.1 語音問題

```
問題：Bot 加入頻道但聽不到使用者說話
原因：
  1. selfDeaf 設為 true（Bot 自聾了）
  2. 缺少 @discordjs/opus 或 sodium-native
  3. 權限不足

解決：
  - 確認 selfDeaf: false
  - npm install @discordjs/opus sodium-native
  - 確認 Bot 有「Connect」和「Speak」權限
```

```
問題：Bot 說話有明顯延遲（> 5 秒）
原因：
  1. STT 使用非串流模式
  2. LLM 回應太長
  3. TTS 使用非串流模式

解決：
  - 使用 Azure STT 串流模式
  - 在 SOUL.md 中強調簡短回應
  - 使用 TTS 串流模式
  - 設定 max_tokens: 500
```

```
問題：說話者識別不準確
原因：
  Discord 原生提供 userId，不應有識別問題。
  如果問題是「不知道使用者的名字」：

解決：
  - 使用 guild.members.fetch(userId) 獲取顯示名稱
  - 首次互動時詢問使用者偏好的稱呼
  - 存入記憶系統
```

---

> **下一章**：[Copilot CLI 插件專案](./02-Copilot-CLI插件專案.md)
