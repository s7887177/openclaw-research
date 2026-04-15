# 語音通道的新維度與未來趨勢

> **所屬 Topic**：多通道範式（Multi-Channel Paradigm）  
> **層級**：Level 6 — 概念綜合層  
> **前置閱讀**：Level 2 Voice & Media、Level 3 Provider 抽象層

---

## 摘要

語音通道代表了多通道範式從文字到多模態（Multimodal）的質變。本章分析 OpenClaw 如何透過 STT（Speech-to-Text）+ TTS（Text-to-Speech）+ Realtime Voice Bridge 三層架構，將語音能力無縫整合進現有的通道體系。同時探討 Talk Mode / Wake Mode 等創新互動模式，以及 AR/VR、多模態等未來通道趨勢。

---

## 1. 從文字到語音的跨越

### 1.1 為什麼語音是不同的維度？

文字通道和語音通道的根本差異不只是「輸入/輸出格式不同」——它們代表完全不同的互動範式：

| 維度 | 文字通道 | 語音通道 |
|------|---------|---------|
| **互動節奏** | 非同步，使用者可以慢慢打字 | 即時，沉默就是尷尬 |
| **回應延遲容忍度** | 秒到分鐘級 | 毫秒到秒級 |
| **多人場景** | 每個人輪流發訊息 | 多人同時說話 |
| **情感頻寬** | 低（文字 + emoji） | 高（語調、節奏、音量） |
| **資訊密度** | 高（可以用格式化、連結、圖片） | 低（只有聲音） |
| **回溯能力** | 強（可以捲動歷史） | 弱（說過就過了，除非錄音） |

這些差異意味著語音通道不能簡單地「把文字轉成語音」——它需要根本不同的互動設計。

### 1.2 OpenClaw 的語音架構三層模型

OpenClaw 的語音能力由三個互補的層級組成：

```
第三層：即時語音橋接（Realtime Voice Bridge）
        ├── 全雙工語音通話
        ├── 即時轉寫 + 即時合成
        └── 適用於 Discord 語音頻道、電話呼叫

第二層：TTS / STT 引擎層
        ├── TTS：SpeechProviderPlugin（文字 → 語音）
        ├── STT：RealtimeTranscriptionProviderPlugin（語音 → 文字）
        └── 多 Provider 支援（ElevenLabs, Deepgram, Microsoft, OpenAI）

第一層：通道語音傳輸層
        ├── Discord @discordjs/voice
        ├── QQ Bot SILK Codec
        └── Voice Call Extension（電話）
```

### 1.3 SpeechProviderPlugin 介面

TTS 的核心是 `SpeechProviderPlugin`（`src/tts/provider-types.ts`）：

```typescript
type SpeechProviderPlugin = {
  synthesizeText(req: SpeechSynthesisRequest): Promise<SpeechSynthesisResult>;
  listVoices(req: SpeechListVoicesRequest): Promise<SpeechVoiceOption[]>;
};

type SpeechSynthesisResult = {
  audioBuffer: Buffer;        // 音訊資料
  outputFormat: string;       // 格式（mp3, opus, etc.）
  fileExtension: string;      // 副檔名
  voiceCompatible: boolean;   // 是否適合作為語音訊息
};
```

多個 Provider 實作了這個介面：
- **ElevenLabs**（`extensions/elevenlabs/`）：高品質、擬真語音，透過 `api.registerSpeechProvider()` 註冊
- **Microsoft Speech**（`extensions/microsoft/`）：Azure Cognitive Services
- **OpenAI TTS**（透過 OpenAI API）
- **Sherpa-ONNX**（`skills/sherpa-onnx-tts/`）：本地離線 TTS，不需要雲端 API

### 1.4 Realtime Voice Bridge

即時語音橋接（`src/realtime-voice/provider-types.ts`）是最複雜的語音元件，支援全雙工（Full-duplex）語音通訊：

```typescript
type RealtimeVoiceBridge = {
  connect(): Promise<void>;
  sendAudio(audio: Buffer): void;           // 發送使用者語音
  setMediaTimestamp(ts: number): void;
  sendUserMessage?(text: string): void;     // 發送文字（混合模式）
  triggerGreeting?(instructions?: string): void;  // 觸發問候
  submitToolResult(callId: string, result: unknown): void;  // 工具結果
  acknowledgeMark(): void;
  close(): void;
  isConnected(): boolean;
};

type RealtimeVoiceBridgeCallbacks = {
  onAudio: (muLaw: Buffer) => void;        // 收到 AI 語音
  onClearAudio: () => void;                 // 清除播放佇列
  onTranscript?: (role, text, isFinal) => void;  // 轉寫結果
  onToolCall?: (event) => void;             // 工具呼叫
  onReady?: () => void;
  onError?: (error: Error) => void;
  onClose?: (reason) => void;
};
```

這個橋接支援：
- **即時音訊串流**：µ-Law 編碼的音訊雙向傳輸
- **即時轉寫**：語音轉文字，區分 `user` 和 `assistant` 角色
- **工具呼叫**：在語音對話中觸發工具（如搜尋、執行程式碼）
- **打斷處理**：`onClearAudio` 允許使用者打斷 AI 的發言

---

## 2. Talk Mode 與 Wake Mode

### 2.1 Talk Mode

Talk Mode 是 OpenClaw 的語音互動模式，由 `extensions/talk-voice/` 管理。它將語音能力整合到通道體驗中：

- **啟動方式**：透過指令或語音觸發
- **運作原理**：持續監聽麥克風 → STT 轉寫 → Agent 處理 → TTS 回應 → 播放
- **Voice Picker**：允許使用者從多個 TTS Provider 中選擇偏好的聲音

### 2.2 Wake Mode 概念

Wake Mode 是一種被動監聽模式：

- AI 在語音頻道中「待命」
- 偵測到喚醒詞（Wake Word）或被提及時才回應
- 沒有被觸發時，AI 會聽但不說話
- 這特別適合 Discord 語音頻道的多人場景

> 💡 Wake Mode 的概念與本研究「應用目標二：Discord 語音社交機器人」直接相關。機器人應該有禮貌地聆聽，懂得在適當時機才插話，而不是每句話都搶著回應。

---

## 3. 語音通道與文字通道的整合

### 3.1 語音作為通道的擴展，而非替代

OpenClaw 的語音能力不是獨立的系統，而是現有通道的擴展。例如在 Discord 中：

```
Discord 伺服器
├── #general（文字頻道）── 文字通道 Plugin 處理
├── #voice-lounge（語音頻道）── 語音通道擴展處理
│   ├── STT：將成員的語音轉為文字
│   ├── Agent：處理文字（與文字通道共用同一個 Agent 核心）
│   └── TTS：將 Agent 回應轉為語音播放
└── 兩者共享同一個 Agent 人格與長期記憶
```

### 3.2 語音訊息（Voice Note）

除了即時語音通話，OpenClaw 也支援語音訊息——使用者錄製語音、AI 回覆語音。`SpeechSynthesisTarget` 型別區分了兩種場景：

```typescript
type SpeechSynthesisTarget = "audio-file" | "voice-note";
```

- `"audio-file"`：生成音訊檔案（適合長回應）
- `"voice-note"`：生成語音訊息（適合短回覆，模擬人類語音訊息）

---

## 4. 未來趨勢

### 4.1 AR/VR 通道

隨著 Apple Vision Pro、Meta Quest 等設備的普及，AR/VR 將成為新的通訊通道。在 OpenClaw 的架構下，AR/VR 通道意味著：

- **空間互動**：AI 人格可以有視覺化的虛擬形象
- **手勢輸入**：除了語音和文字，還可以透過手勢互動
- **環境感知**：AI 可以「看到」使用者周圍的環境並做出回應

在 OpenClaw 的 `ChannelPlugin` 架構中，這只需要：
1. 新增一個 AR/VR channel extension
2. 宣告適當的 `capabilities`（如 `media`, 語音，可能是新的 `spatial` 能力）
3. 實作對應的 `outbound` 和 `gateway` 適配器

### 4.2 多模態（Multimodal）整合

當前的通道大多是「文字」或「文字+語音」。未來的趨勢是真正的多模態——同一個互動中同時包含：

- **文字**：打字詢問
- **語音**：口語對話
- **影像**：分享螢幕截圖或即時影像
- **手勢/表情**：透過攝影機捕捉的非語言訊號

OpenClaw 的架構已經有部分基礎：
- `media-understanding-core`：影像/影片理解
- `realtime-transcription`：即時語音轉寫
- `video-generation-core`：影片生成

缺少的是一個統一的「多模態 Session」概念——能在同一個對話中同時處理多種模態的輸入。

### 4.3 Agent-to-Agent 通道

隨著 ACP（Agent Communication Protocol，`source-repo/src/acp/`）的發展，未來的「通道」可能不只連接人和 AI，還連接 AI 和 AI。想像一個場景：

- 你的 OpenClaw Agent 透過 ACP 與你朋友的 OpenClaw Agent 對話
- 兩個 Agent 代表各自的使用者協調行程
- 最終結果再透過各自的通道回報給各自的使用者

這是多通道範式的終極延伸——通道不只連接人與機器，還連接機器與機器。

### 4.4 物聯網（IoT）通道

OpenClaw 的 `extensions/phone-control/` 和 `skills/openhue/`（Philips Hue 燈光控制）已經暗示了 IoT 整合的可能性。未來可能出現：

- 智慧音箱作為語音通道
- 車載系統作為語音+視覺通道
- 智慧手錶作為簡訊通道

---

## 5. 多通道範式的終極願景

回到開頭的問題：為什麼需要多通道？

最終答案是：**因為人類的注意力是碎片化的，而 AI 助手應該能在每個碎片中與你同在。**

OpenClaw 的多通道範式不是技術炫耀，而是對一個簡單事實的回應——人們不會為了跟 AI 說話而專門打開一個 App。AI 需要存在於你已經在用的地方。不管是 Discord 的遊戲頻道、Slack 的工作空間、WhatsApp 的家庭群組、還是通勤路上的語音通話——它都應該在那裡，記得你，懂你，幫你。

這就是「一個人格，多個觸點」的真正意義。

---

## 引用來源

| 來源 | 路徑 / 說明 |
|------|-------------|
| SpeechProviderPlugin 型別 | `source-repo/src/tts/provider-types.ts` |
| RealtimeVoiceBridge 型別 | `source-repo/src/realtime-voice/provider-types.ts` |
| ElevenLabs 註冊 | `source-repo/extensions/elevenlabs/index.ts:1-10` |
| Deepgram 註冊 | `source-repo/extensions/deepgram/index.ts:1-10` |
| Talk Voice Extension | `source-repo/extensions/talk-voice/index.ts` |
| Speech Core | `source-repo/extensions/speech-core/src/tts.ts` |
| Sherpa-ONNX TTS Skill | `source-repo/skills/sherpa-onnx-tts/` |
| Voice Call Extension | `source-repo/extensions/voice-call/` |
| ACP 子系統 | `source-repo/src/acp/` |
| Phone Control | `source-repo/extensions/phone-control/` |
| Level 2 Voice & Media | `research/versions/2026.4.12/component-mechanics/voice-media/` |
| Level 3 Provider 抽象層 | `research/versions/2026.4.12/cross-cutting-mechanisms/provider-abstraction/` |
