# OpenClaw 語音與媒體系統完整解析

## 本章摘要

OpenClaw 的語音與媒體系統（Voice & Media System）是整個框架中負責「讓 AI 開口說話、聽懂人話」的核心子系統。對於一個 AI 自動化框架而言，語音能力並非錦上添花的附加功能，而是決定使用體驗上限的關鍵因素——它讓 AI 從「只能打字聊天的機器人」升級為「可以自然對話的數位夥伴」。本章將從零開始，帶你完整理解 OpenClaw 如何設計並實現這套語音與媒體系統。

我們將涵蓋四大核心能力：即時語音對話（Realtime Voice）讓使用者能像打電話一樣與 AI 即時交談；即時語音轉文字（Realtime Transcription）負責將人類的語音即時轉換為機器可處理的文字；文字轉語音（Text-to-Speech, TTS）則反過來把 AI 的文字回覆朗讀出來；最後，電話通話整合（Voice Call / Telephony）讓整套系統能接入真實的電話網路，支援撥打和接聽電話。

在深入各子系統之前，讓我們先建立全局觀。整個語音與媒體系統由八個主要元件組成，它們的定位、規模和職責如下表所示：

| 子系統 | 原始碼位置 | 規模 | 核心職責 |
|--------|-----------|------|---------|
| Realtime Voice | `src/realtime-voice/` | 2 檔案，120 行 | 即時雙向語音通話的抽象介面層 |
| Realtime Transcription | `src/realtime-transcription/` | 2 檔案，91 行 | 即時語音辨識的抽象介面層 |
| TTS 核心 | `src/tts/` | 10 檔案，1,254 行 | 文字轉語音的配置管理、指令解析與請求路由 |
| Speech Core 擴充 | `extensions/speech-core/` | 1,391 行 | TTS 的完整實作邏輯，含自動摘要與供應商容錯 |
| ElevenLabs 擴充 | `extensions/elevenlabs/` | 1,235 行 | ElevenLabs 雲端語音合成供應商的實作 |
| Deepgram 擴充 | `extensions/deepgram/` | 282 行 | Deepgram 雲端語音辨識供應商的實作 |
| Talk Voice 擴充 | `extensions/talk-voice/` | 558 行 | 面向使用者的語音角色管理指令 |
| Voice Call 擴充 | `extensions/voice-call/` | 17,260 行 | 完整的電話線路整合（Twilio、Plivo、Telnyx） |

從規模上看，Voice Call 擴充以超過一萬七千行程式碼佔據了絕對主導地位，這並不令人意外——電話整合涉及 Webhook 安全驗證、多供應商適配、即時媒體串流等大量複雜邏輯。相比之下，核心抽象層的 Realtime Voice 和 Realtime Transcription 各自只有不到一百五十行，因為它們只定義介面契約，不包含任何具體實作。

這些子系統共享一個統一的**供應商註冊模式**（Provider Registry Pattern），使得任何第三方開發者都可以透過插件 SDK（Plugin SDK）註冊自己的語音供應商，而無需修改 OpenClaw 的核心程式碼。這種插件化的架構設計，是 OpenClaw 語音系統最值得深入理解的設計哲學。

---

## 一、系統架構總覽

### 1.1 分層架構設計理念

OpenClaw 的語音系統採用經典的**三層架構**（Three-Layer Architecture），將關注點清晰地分離為應用層、核心抽象層和供應商實作層。這種分層不是為了教科書式的美觀，而是基於非常務實的考量：語音技術領域的供應商更迭極為頻繁，新的語音模型和服務幾乎每個月都在出現，如果核心邏輯與特定供應商綁定，每次切換或新增供應商都會是一場痛苦的重構。

```
┌─────────────────────────────────────────────────┐
│            應用層 (Application Layer)              │
│  Talk Mode · Voice Wake · /voice 指令 · 電話通話    │
├─────────────────────────────────────────────────┤
│            核心抽象層 (Core Abstraction)            │
│  src/realtime-voice · src/realtime-transcription   │
│  src/tts (配置、指令、路由)                          │
├─────────────────────────────────────────────────┤
│            供應商實作層 (Provider Layer)             │
│  ElevenLabs · Deepgram · Twilio · Plivo · Telnyx  │
│  sherpa-onnx · 系統 TTS · 本地 MLX                  │
└─────────────────────────────────────────────────┘
```

**應用層**是使用者直接互動的層面，包括 Talk Mode（持續語音對話模式）、Voice Wake（語音喚醒功能）、`/voice` 管理指令，以及電話通話功能。這一層關心的是「使用者體驗」——何時開始聽、何時開始說、如何自然地交替發言。

**核心抽象層**定義了語音相關的所有型別介面與註冊機制，但刻意不包含任何供應商的具體實作程式碼。它就像是一份「契約書」，規定了語音供應商必須實現哪些能力，但不關心供應商內部是怎麼做到的。具體的語音合成演算法、語音辨識模型呼叫、音訊編解碼等工作，全部委託給下方的供應商實作層。

**供應商實作層**由各個擴充套件（Extensions）組成，每個擴充套件封裝了一個或多個特定供應商的完整實作。這層的程式碼量最大、變動最頻繁，但由於與核心層解耦，新增或移除供應商不會影響系統的其他部分。

這種設計的直接好處是：OpenClaw 可以在完全不修改核心程式碼的前提下，透過新增擴充套件來支援任意數量的語音供應商。對於我們研究的 Discord 語音機器人應用場景而言，這意味著我們可以自由選擇最適合的語音供應商，甚至混合使用多個供應商——例如用 ElevenLabs 做語音合成、用 Deepgram 做語音辨識。

### 1.2 統一供應商註冊模式（Shared Provider Registry Pattern）

理解統一供應商註冊模式是掌握整個語音系統運作機制的關鍵。OpenClaw 的設計者做了一個非常聰明的決定：讓三個核心語音子系統——即時語音（Realtime Voice）、即時轉錄（Realtime Transcription）、文字轉語音（TTS）——全部使用**結構完全相同**的供應商發現與管理機制。這不僅降低了學習成本（理解一個就等於理解了三個），也讓插件開發者可以用統一的心智模型來開發不同類型的語音供應商。

這個模式的運作流程分為六個標準步驟：

| 步驟 | 機制 | 詳細說明 |
|------|------|---------|
| 1. 供應商發現 | `resolvePluginCapabilityProviders({ key: "..." })` | 系統啟動時，透過插件 SDK 掃描所有已載入的插件，找出聲明了特定能力鍵的供應商 |
| 2. 對照表建立 | `buildCapabilityProviderMaps()` | 將發現的供應商整理成兩張對照表：正規名稱（canonical）對照表和別名（alias）對照表，支援多種方式引用同一供應商 |
| 3. 識別碼正規化 | `normalize*ProviderId()` | 將使用者輸入的供應商識別碼（可能包含大小寫差異或別名）轉換為統一的內部格式 |
| 4. 供應商列舉 | `list*Providers()` | 列出系統中所有可用的該類型供應商，常用於 UI 顯示和指令補全 |
| 5. 供應商取得 | `get*Provider()` | 根據識別碼取得特定供應商的實例，這是實際使用供應商時的入口 |
| 6. 識別碼標準化 | `canonicalize*ProviderId()` | 將任意形式的識別碼轉為官方標準形式，用於持久化儲存和跨系統識別 |

三個子系統分別使用不同的能力鍵（capability key）來區分供應商類型：

- 即時語音供應商：`"realtimeVoiceProviders"`（`source-repo/src/realtime-voice/provider-registry.ts:17-20`）
- 即時轉錄供應商：`"realtimeTranscriptionProviders"`（`source-repo/src/realtime-transcription/provider-registry.ts:19`）
- 語音合成供應商：`"speechProviders"`（`source-repo/src/tts/provider-registry.ts:17`）

這個模式對插件開發者的好處非常直觀：你只需在自己的擴充套件中實作對應的介面，然後在插件 SDK 的註冊階段以正確的能力鍵進行登記，系統就會在啟動時自動發現你的供應商、為它建立對照條目，並讓使用者可以透過配置檔或指令來選用。整個過程不需要修改 OpenClaw 的任何核心檔案。

---

## 二、即時語音子系統（Realtime Voice）

### 2.1 概念說明

即時語音子系統處理的是**雙向即時語音通話**的場景。所謂「即時」，是指使用者說話的每一個音節都在產生的瞬間被傳送到 AI 端，而 AI 的回覆語音也是邊產生邊播放——這不是「錄完一整段話再送出去等回覆」的請求-回應模式，而是像真人打電話一樣的連續串流（streaming）雙向音訊傳輸。這種即時性對使用者體驗至關重要，因為人類對話中的自然節奏、打斷、接話等行為，都依賴於毫秒級的音訊傳輸延遲。

整個子系統只有兩個檔案、共 120 行程式碼。這個極度精簡的規模並非因為功能簡單，而是因為它採用了純抽象的設計策略——只定義介面契約（type definitions），不包含任何具體實作。真正的即時語音處理邏輯，全部由透過插件系統註冊的供應商來實現。

### 2.2 核心型別定義

#### 角色與狀態

即時語音系統首先定義了三個基礎型別，分別描述供應商的身份識別、對話中的角色，以及連線結束的原因：

```typescript
// source-repo/src/realtime-voice/provider-types.ts:3
export type RealtimeVoiceProviderId = string;

// source-repo/src/realtime-voice/provider-types.ts:5
export type RealtimeVoiceRole = "user" | "assistant";

// source-repo/src/realtime-voice/provider-types.ts:7
export type RealtimeVoiceCloseReason = "completed" | "error";
```

`RealtimeVoiceProviderId` 是一個字串型別的別名，用來識別不同的即時語音供應商。在即時語音的世界中，只有兩個角色：`"user"`（使用者，也就是正在說話的人類）和 `"assistant"`（AI 助手）。這種二元角色模型反映了語音通話場景的本質——它本質上就是兩個參與者之間的一對一對話。連線結束的原因也被簡化為兩種：`"completed"` 表示正常結束（例如使用者掛斷電話），`"error"` 表示因為技術問題而中斷。

#### 工具定義（Tool Definition）

一個令人驚喜的設計是：即時語音不僅支援對話，還支援在通話過程中即時呼叫工具（function calling）。這意味著 AI 在跟你說話的同時，可以去查詢天氣、搜尋資料庫、呼叫外部 API——就像一個真人助理一邊跟你通話，一邊在電腦上幫你查東西：

```typescript
// source-repo/src/realtime-voice/provider-types.ts:9-18
export type RealtimeVoiceTool = {
  type: "function";
  name: string;
  description: string;
  parameters: {
    type: "object";
    properties: Record<string, unknown>;
    required?: string[];
  };
};
```

這個型別的結構完全對齊 OpenAI Function Calling 的規範——`type` 固定為 `"function"`，搭配工具名稱、自然語言描述，以及 JSON Schema 格式的參數定義。這種設計上的一致性讓 OpenClaw 能夠無縫接入支援 Function Calling 的任何大型語言模型。

#### Bridge 回呼（Callbacks）

Bridge（橋接器）是即時語音的核心抽象概念。你可以把它想像成一座連接使用者和 AI 的橋——音訊資料在這座橋上雙向流動。Bridge 的回呼介面定義了系統在語音通話過程中可能收到的所有類型的事件通知：

```typescript
// source-repo/src/realtime-voice/provider-types.ts:27-36
export type RealtimeVoiceBridgeCallbacks = {
  onAudio: (muLaw: Buffer) => void;          // 收到音訊資料
  onClearAudio: () => void;                   // 清除音訊緩衝區
  onMark?: (markName: string) => void;        // 音訊標記點
  onTranscript?: (role: RealtimeVoiceRole, text: string, isFinal: boolean) => void;
  onToolCall?: (event: RealtimeVoiceToolCallEvent) => void;
  onReady?: () => void;                       // 連線就緒
  onError?: (error: Error) => void;           // 發生錯誤
  onClose?: (reason: RealtimeVoiceCloseReason) => void;
};
```

值得注意的設計細節是：`onAudio` 和 `onClearAudio` 是**必要回呼**（型別中沒有 `?` 標記），其餘六個都是選填的。這個選擇傳達了一個重要的設計意圖：音訊串流是語音橋接器最基本、不可或缺的功能——你可以不要文字轉錄、不要工具呼叫、甚至不監聽連線狀態變化，但你一定要能傳送和接收音訊資料。沒有音訊，就沒有語音通話。

`onTranscript` 回呼中的 `isFinal` 參數值得特別關注：它區分了「暫時性轉錄」（partial transcript，使用者還在說話中，文字可能會被修正）和「最終轉錄」（final transcript，這句話已經說完，文字不會再變）。這個區分對於 UI 顯示即時字幕至關重要——暫時性文字通常以較淡的顏色顯示，而最終文字則會被確認並記錄到對話歷史中。

`onClearAudio` 回呼的存在暗示了「打斷」（interruption）機制：當使用者在 AI 說話途中插嘴時，系統需要清除已排隊但尚未播放的音訊資料，才能立即切換到聆聽使用者的模式。

#### Bridge 介面

Bridge 介面定義了系統可以**主動**對語音連線做的所有操作：

```typescript
// source-repo/src/realtime-voice/provider-types.ts:56-66
export type RealtimeVoiceBridge = {
  connect(): Promise<void>;
  sendAudio(audio: Buffer): void;
  setMediaTimestamp(ts: number): void;
  sendUserMessage?(text: string): void;
  triggerGreeting?(instructions?: string): void;
  submitToolResult(callId: string, result: unknown): void;
  acknowledgeMark(): void;
  close(): void;
  isConnected(): boolean;
};
```

這個介面揭示了語音通話的完整生命週期以及系統在每個階段能做的事情：

1. **建立連線** → `connect()` 是非同步的（回傳 Promise），因為建立與遠端語音服務的 WebSocket 連線需要時間
2. **雙向音訊傳輸** → `sendAudio()` 將使用者的麥克風音訊送出去，而 AI 的回覆音訊則透過 `onAudio` 回呼接收回來
3. **媒體同步** → `setMediaTimestamp(ts)` 設定目前的媒體時間戳，用於音訊同步和標記對齊
4. **工具呼叫流程** → AI 透過 `onToolCall` 回呼發起工具呼叫，系統處理完成後透過 `submitToolResult` 將結果回傳
5. **混合模式對話** → `sendUserMessage` 允許在語音通話中插入文字訊息，而 `triggerGreeting` 讓 AI 主動先開口打招呼（例如接起電話時說「你好，請問有什麼我可以幫忙的？」）
6. **標記確認** → `acknowledgeMark()` 用於確認收到音訊標記點，這是音訊串流同步機制的一部分
7. **連線管理** → `close()` 關閉連線、`isConnected()` 檢查目前的連線狀態

### 2.3 供應商註冊表

即時語音的供應商註冊表完全遵循統一供應商註冊模式，提供四個標準函式：

```typescript
// source-repo/src/realtime-voice/provider-registry.ts:10
export function normalizeRealtimeVoiceProviderId(providerId)

// source-repo/src/realtime-voice/provider-registry.ts:30
export function listRealtimeVoiceProviders(cfg?)

// source-repo/src/realtime-voice/provider-registry.ts:34
export function getRealtimeVoiceProvider(providerId, cfg?)

// source-repo/src/realtime-voice/provider-registry.ts:45
export function canonicalizeRealtimeVoiceProviderId(providerId, cfg?)
```

這些函式的內部實作都仰賴 `resolvePluginCapabilityProviders({ key: "realtimeVoiceProviders" })`（`source-repo/src/realtime-voice/provider-registry.ts:17-20`）來發現已註冊的語音供應商。這意味著任何第三方插件只要在註冊時聲明自己提供 `"realtimeVoiceProviders"` 能力，就能被這些函式自動發現和管理。

---

## 三、即時轉錄子系統（Realtime Transcription）

### 3.1 概念說明

即時轉錄子系統負責**將連續的語音串流即時轉換為文字**，也就是語音轉文字（Speech-to-Text, STT）的即時版本。與前面介紹的即時語音子系統相比，即時轉錄的職責範圍更為聚焦——它只需要處理單向的「語音 → 文字」轉換流程，不涉及 AI 回覆的語音合成，也不需要處理工具呼叫。你可以把即時語音想像成「完整的雙向通話」，而即時轉錄則更像是「一個隨時在旁邊做筆記的速記員」。

在實際應用中，即時轉錄最常見的用途包括：在 Talk Mode 中將使用者的口述即時轉為文字輸入、在語音通話中產生即時字幕、以及將會議或對話錄音即時轉為逐字稿。

整個子系統延續了極簡的設計哲學：2 個檔案、91 行程式碼，同樣只定義抽象介面，具體的語音辨識引擎（如 Deepgram、Whisper 等）由供應商實作層負責。

### 3.2 核心型別

供應商識別碼同樣是簡單的字串型別：

```typescript
// source-repo/src/realtime-transcription/provider-types.ts:3
export type RealtimeTranscriptionProviderId = string;
```

#### Session 回呼

即時轉錄使用「Session」（會話）而非即時語音的「Bridge」來命名其核心抽象，這個命名差異反映了兩者的使用模式不同——Bridge 強調的是雙向溝通的橋梁，而 Session 更像是一段單向的工作期間：

```typescript
// source-repo/src/realtime-transcription/provider-types.ts:17-22
export type RealtimeTranscriptionSessionCallbacks = {
  onPartial?: (partial: string) => void;      // 暫時性轉錄結果
  onTranscript?: (transcript: string) => void; // 最終轉錄結果
  onSpeechStart?: () => void;                  // 偵測到語音開始
  onError?: (error: Error) => void;            // 發生錯誤
};
```

與即時語音的八個回呼相比，轉錄只需要四個——而且全部都是選填的。這個設計選擇很有意思：即使你不監聽任何事件，轉錄 Session 仍然可以正常運作（例如僅在內部累積文字，供後續查詢）。但在實務中，你幾乎一定會至少監聽 `onTranscript` 來取得最終的轉錄文字。

`onPartial` 提供邊說邊轉的暫時性結果，這段文字會隨著使用者繼續說話而不斷被修正和覆蓋。`onTranscript` 則在語音辨識引擎偵測到語句結束（例如停頓、語調下降）時回傳確定的完整文字。`onSpeechStart` 在偵測到使用者開始說話時觸發，這對於 UI 來說很有用——可以顯示一個「正在聆聽」的指示器。

#### Session 介面

```typescript
// source-repo/src/realtime-transcription/provider-types.ts:28-33
export type RealtimeTranscriptionSession = {
  connect(): Promise<void>;
  sendAudio(audio: Buffer): void;
  close(): void;
  isConnected(): boolean;
};
```

只有四個方法，代表了轉錄 Session 的極簡生命週期：建立連線、持續送入音訊資料、關閉連線、檢查狀態。與即時語音的 Bridge 介面做對比，你會發現這裡沒有 `sendUserMessage`（不需要文字輸入）、沒有 `submitToolResult`（不涉及工具呼叫）、沒有 `triggerGreeting`（轉錄是被動聆聽，不主動開口）、也沒有 `setMediaTimestamp` 和 `acknowledgeMark`（不需要音訊同步標記）。這些缺失恰好勾勒出轉錄功能的本質：它是一個純粹的感知功能，只負責聽和記錄，不參與互動。

### 3.3 供應商註冊表

```typescript
// source-repo/src/realtime-transcription/provider-registry.ts:10
normalizeRealtimeTranscriptionProviderId()

// source-repo/src/realtime-transcription/provider-registry.ts:32
listRealtimeTranscriptionProviders()

// source-repo/src/realtime-transcription/provider-registry.ts:38
getRealtimeTranscriptionProvider()

// source-repo/src/realtime-transcription/provider-registry.ts:49
canonicalizeRealtimeTranscriptionProviderId()
```

同樣使用統一的插件發現機制來管理供應商：`resolvePluginCapabilityProviders({ key: "realtimeTranscriptionProviders" })`（`source-repo/src/realtime-transcription/provider-registry.ts:19`）。函式命名雖然更長（因為 "RealtimeTranscription" 本身就是個長名稱），但結構和行為與即時語音的註冊表完全一致。

---

## 四、文字轉語音子系統（TTS）

### 4.1 概念說明

文字轉語音（Text-to-Speech, TTS）是語音系統中最複雜的子系統，負責**將 AI 產生的文字回覆轉換成自然的語音**。如果說即時語音處理的是「如何傳輸聲音」，TTS 處理的則是「如何產生聲音」。這個子系統之所以複雜，是因為它需要處理的決策遠比表面看起來的多：什麼時候要把文字轉成語音？文字太長怎麼辦？使用者想要什麼樣的聲音？如果首選的供應商出錯了怎麼備援？

TTS 的程式碼橫跨核心層（`src/tts/`，10 個檔案，共 1,254 行）和擴充層（`extensions/speech-core/`，1,391 行），總計超過 2,600 行。核心層負責配置管理、指令解析和請求路由，而擴充層（Speech Core）則包含實際的合成邏輯、自動摘要和供應商容錯機制。

### 4.2 TTS 自動模式（Auto Mode）

在日常使用中，不是所有的 AI 回覆都需要被轉成語音。例如，你在文字聊天中問了一個問題，你大概不需要 AI 把回答朗讀出來；但如果你正在開車用語音跟 AI 對話，你當然希望每個回覆都有語音。為了應對這種多樣化的需求，OpenClaw 設計了四種自動語音模式：

```typescript
// source-repo/src/tts/tts-auto-mode.ts:4
export const TTS_AUTO_MODES = new Set<TtsAutoMode>(["off", "always", "inbound", "tagged"]);
```

| 模式 | 觸發條件 | 典型使用場景 |
|------|---------|-------------|
| `off` | 永不自動轉語音 | 純文字聊天、開發測試 |
| `always` | 所有 AI 回覆都轉語音 | 語音優先的裝置（智慧音箱、車載系統） |
| `inbound` | 僅當使用者以語音方式發送訊息時，AI 回覆也用語音 | 混合模式（有時打字、有時說話）的手機應用 |
| `tagged` | 僅當 AI 回覆文字中包含特定標記時才轉語音 | 需要精確控制哪些內容要朗讀的進階場景 |

`inbound` 模式特別巧妙——它讓系統自動跟隨使用者的溝通方式，如果你用嘴巴說，AI 就用嘴巴回；如果你用鍵盤打字，AI 就用文字回。這避免了在安靜的辦公室裡突然從喇叭蹦出語音的尷尬場景。

### 4.3 TTS 配置解析

TTS 的配置體系有好幾層優先順序，這反映了 OpenClaw 「讓每一層的設定都能覆蓋上一層」的設計哲學。

首先是 TTS 模式的解析，預設值為 `"final"`（只在最終回覆時合成語音）：

```typescript
// source-repo/src/tts/tts-config.ts:9-11
export function resolveConfiguredTtsMode(cfg: OpenClawConfig): TtsMode {
  return cfg.messages?.tts?.mode ?? "final";
}
```

偏好設定檔路徑的解析遵循三級優先順序（`source-repo/src/tts/tts-config.ts:13-22`）：

1. **函式參數** `prefsPath`（最高優先級）——程式碼中直接指定的路徑
2. **環境變數** `OPENCLAW_TTS_PREFS`——適合在不同環境中切換配置
3. **預設路徑** `~/.openclaw/settings/tts.json`——使用者家目錄下的預設位置

「是否嘗試對訊息進行 TTS 處理」的決策邏輯同樣是多層級的（`source-repo/src/tts/tts-config.ts:45-65`）：

1. 當前 Session 的 `ttsAuto` 設定（最高優先級）
2. 使用者個人偏好設定檔中的設定
3. 全域配置中的 `auto` 欄位
4. 全域配置中的 `enabled` 布林值（最低優先級）

這種層層覆蓋的優先順序設計讓使用者可以在全域層面設定預設行為，然後在個人偏好中覆蓋，再在單次對話中臨時調整——每一層都不會影響其他層的設定。

### 4.4 TTS 指令系統（Directives）

OpenClaw 提供了一套嵌入式指令語法，讓 AI（或技能開發者）可以在回覆文字中直接控制語音合成的各種參數。這套指令系統由 `parseTtsDirectives` 函式負責解析：

```typescript
// source-repo/src/tts/directives.ts:41
export function parseTtsDirectives(text, policy, options?): TtsDirectiveParseResult
```

指令的解析依賴正規表達式來識別文字中的區塊標記：

```typescript
// source-repo/src/tts/directives.ts:56
const blockRegex = /\[\[tts:text\]\]([\s\S]*?)\[\[\/tts:text\]\]/gi;
```

支援的指令類型及其用法：

| 指令語法 | 控制的參數 | 說明 |
|---------|----------|------|
| `[[tts:voice=Rachel]]` | 語音角色 | 切換到指定的語音角色 |
| `[[tts:model=eleven_turbo_v2_5]]` | 合成模型 | 指定要使用的語音模型 |
| `[[tts:stability=0.7]]` | 語音穩定度 | 越高越一致，越低越有變化 |
| `[[tts:similarity=0.8]]` | 相似度增強 | 控制與原始語音的相似程度 |
| `[[tts:speed=1.2]]` | 語速 | 調整朗讀速度 |
| `[[tts:seed=42]]` | 隨機種子 | 固定隨機種子以獲得可重現的結果 |
| `[[tts:language=zh]]` | 語言 | 指定語音合成的語言 |

此外，區塊語法允許指定「只有這段文字需要被朗讀」，其餘部分則保持靜默：

```
[[tts:text]]
只有這段文字會被朗讀出來，其他部分不會。
[[/tts:text]]
```

這套指令系統讓技能（Skills）開發者能夠在回覆中實現精細的語音控制，例如在角色扮演場景中為不同角色使用不同的聲音，或者在教學場景中調慢語速以便使用者跟上。

### 4.5 核心 API 匯出

`src/tts/tts.ts`（34 行）是整個 TTS 子系統的公開入口，它將 `plugin-sdk/tts-runtime.js` 中的實作重新匯出（`source-repo/src/tts/tts.ts`），讓其他子系統可以用統一的路徑來引用這些功能：

| API 函式 | 類別 | 用途說明 |
|----------|------|---------|
| `textToSpeech` | 高階 API | 自動模式判斷 + 長文摘要 + 多供應商容錯，最常用的入口 |
| `synthesizeSpeech` | 低階 API | 直接對指定文字進行語音合成，跳過自動模式邏輯 |
| `textToSpeechTelephony` | 專用 API | 電話通話專用的語音合成，產生電話系統相容的音訊格式 |
| `resolveTtsConfig` | 配置 | 解析並回傳完整的 TTS 配置物件 |
| `resolveTtsAutoMode` | 配置 | 解析目前的自動模式設定 |
| `getTtsProvider` / `setTtsProvider` | 供應商管理 | 取得或切換目前使用的 TTS 供應商 |
| `listSpeechVoices` | 語音管理 | 列出目前供應商下所有可用的語音角色 |
| `maybeApplyTtsToPayload` | 訊息處理 | 根據配置決定是否對訊息附加語音版本 |
| `buildTtsSystemPromptHint` | 提示工程 | 建構系統提示詞中的語音相關提示，告訴 AI 目前啟用了語音 |
| `isSummarizationEnabled` / `setSummarizationEnabled` | 摘要控制 | 查詢或設定長文摘要功能的開關狀態 |

---

## 五、Speech Core 擴充（核心語音實作）

### 5.1 概念說明

`extensions/speech-core/` 是 TTS 子系統的**大腦**——前面介紹的 `src/tts/` 定義了配置和介面，而 Speech Core 則是真正執行語音合成邏輯的地方。主檔案 `src/tts.ts` 以 1,190 行的程式碼量承載了從配置解析、自動模式判斷、長文摘要、供應商選擇、容錯切換到結果回報的完整流程。可以說，所有 `src/tts/` 匯出的高階 API 最終都會走到 Speech Core 裡面的對應實作。

理解 Speech Core 的關鍵在於理解它做了哪些「自動化決策」：它不只是機械地呼叫語音合成 API，而是會根據當前的配置、文字長度、供應商可用性等因素，自動做出一系列智慧判斷。

### 5.2 TTS 結果型別

語音合成的結果使用精細的型別系統來表達。每次合成嘗試都會記錄一個原因代碼（`source-repo/extensions/speech-core/src/tts.ts:76-82`），清楚標明這次嘗試為什麼成功或失敗：

| 原因代碼（`TtsAttemptReasonCode`） | 含義 | 可能的後續動作 |
|----------------------------------|------|--------------|
| `"success"` | 語音合成成功完成 | 回傳音訊檔案 |
| `"no_provider_registered"` | 系統中沒有註冊任何語音合成供應商 | 提示使用者安裝供應商擴充 |
| `"not_configured"` | 供應商已註冊但缺少必要配置（如 API 金鑰） | 提示使用者完成配置 |
| `"unsupported_for_telephony"` | 該供應商不支援電話所需的音訊格式 | 嘗試其他支援電話的供應商 |
| `"timeout"` | 語音合成在指定時間內未完成 | 嘗試更快的供應商或模型 |
| `"provider_error"` | 供應商在合成過程中回傳錯誤 | 嘗試備援供應商 |

完整的 TTS 結果物件（`source-repo/extensions/speech-core/src/tts.ts:92-103`）包含了豐富的除錯和監控資訊，這對於在生產環境中排查語音問題非常有價值：

| 欄位 | 說明 |
|------|------|
| `success` | 布林值，最終是否成功產生了語音 |
| `audioPath` | 產生的音訊檔案路徑（成功時才有） |
| `error` | 錯誤訊息（失敗時才有） |
| `latencyMs` | 合成耗時（毫秒），用於監控效能 |
| `provider` | 最終實際使用的供應商名稱 |
| `fallbackFrom` | 如果發生了容錯切換，記錄原本首選的供應商 |
| `attemptedProviders` | 嘗試過的所有供應商列表 |
| `attempts` | 每次嘗試的詳細記錄陣列，含個別的原因代碼和耗時 |

`fallbackFrom` 和 `attemptedProviders` 這兩個欄位揭示了 Speech Core 內建的**供應商容錯機制**：當首選供應商出錯或逾時時，系統會自動嘗試下一個可用的供應商，而不是直接回報失敗。這種容錯策略對於需要高可用性的應用（如電話客服系統）特別重要。

### 5.3 關鍵 API 實作

以下是 Speech Core 中最重要的函式及其在 `source-repo/extensions/speech-core/src/tts.ts` 中的位置和職責：

| 函式 | 行號 | 類別 | 詳細說明 |
|------|------|------|---------|
| `getResolvedSpeechProviderConfig()` | 301 | 配置 | 解析並回傳目前語音供應商的完整配置，合併各層級的設定 |
| `resolveTtsConfig()` | 313 | 配置 | 解析完整的 TTS 配置，包括模式、自動模式、供應商偏好等 |
| `resolveTtsAutoMode()` | 351 | 配置 | 專門解析自動模式的設定，考慮 session 和偏好的覆蓋 |
| `buildTtsSystemPromptHint()` | 387 | 提示工程 | 建構插入到系統提示詞中的語音提示，讓 AI 知道目前啟用了語音功能 |
| `isTtsEnabled()` | 444 | 狀態查詢 | 綜合判斷 TTS 是否在目前環境下啟用 |
| `textToSpeech()` | 747 | **高階 API** | 最主要的入口，包含自動模式判斷、長文摘要、供應商容錯的完整流程 |
| `synthesizeSpeech()` | 785 | 低階 API | 直接進行語音合成，跳過自動模式和摘要邏輯 |
| `textToSpeechTelephony()` | 889 | 專用 API | 為電話通話場景產生相容的音訊，可能使用不同的編碼格式 |
| `listSpeechVoices()` | 988 | 語音管理 | 列出目前供應商提供的所有語音角色 |
| `maybeApplyTtsToPayload()` | 1018 | 訊息整合 | 根據配置判斷是否需要對訊息加上語音版本 |

其中 `textToSpeech()`（第 747 行）是絕大多數場景下的入口函式。它的內部流程大致是：先判斷自動模式是否允許 TTS → 檢查供應商是否可用 → 如果文字過長就先摘要 → 解析文字中的 TTS 指令 → 嘗試首選供應商進行合成 → 如果失敗則嘗試備援供應商 → 回傳完整的結果物件。

### 5.4 文字摘要機制

Speech Core 內建了一個務實的長文處理策略：當要轉換的文字超過一定長度時，先用語言模型進行摘要，再對摘要文字進行語音合成。

- **觸發閾值**：文字長度超過 4,096 字元（常數名稱：`DEFAULT_MAX_TEXT_LENGTH`）
- **摘要策略**：使用語言模型以溫度（temperature）0.3 進行摘要

溫度 0.3 是一個偏低的值，意味著摘要結果會比較穩定和忠實於原文，而不是天馬行空地改寫。這是合理的選擇——語音摘要的目的是傳達核心資訊，不需要創意發揮。

這個設計反映了一個務實的認知：極少有使用者會想聽 AI 朗讀一篇五千字的長文。摘要後再合成，既降低了語音合成的成本和延遲（供應商通常按字元數收費），也提升了使用者的聆聽體驗。

---

## 六、ElevenLabs 語音合成擴充

### 6.1 概念說明

ElevenLabs 是目前業界最知名的 AI 語音合成服務之一，以其高品質、自然度極高的語音聞名。在 OpenClaw 的生態中，`extensions/elevenlabs/` 是最主要的雲端語音合成供應商實作之一，共 1,235 行程式碼，其中 `speech-provider.ts`（528 行）負責核心的供應商適配邏輯。

作為一個供應商擴充，ElevenLabs 擴充的職責是：接收來自 Speech Core 的語音合成請求，將其轉換為 ElevenLabs API 能理解的格式，呼叫 API 取得音訊，然後將結果回傳給 Speech Core。它本身不做任何「是否要合成」的決策——那些決策已經由上層的 Speech Core 做好了。

### 6.2 預設配置

ElevenLabs 擴充提供了合理的預設配置（`source-repo/extensions/elevenlabs/speech-provider.ts:40-55`），讓使用者只需提供 API 金鑰就能立即開始使用：

| 配置項 | 預設值 | 是否必填 | 說明 |
|--------|--------|---------|------|
| `apiKey` | （無預設） | ✅ 必填 | ElevenLabs 帳號的 API 金鑰 |
| `baseUrl` | `https://api.elevenlabs.io` | 選填 | API 基礎位址，可指向自建代理伺服器 |
| `voiceId` | `pMsXgVXv3BLzUgSXRplE` | 選填 | 預設使用的語音角色識別碼 |
| `modelId` | `eleven_multilingual_v2` | 選填 | 預設使用的語音合成模型 |
| `applyTextNormalization` | — | 選填 | 文字正規化處理：`"auto"`、`"on"` 或 `"off"` |

語音細部設定（Voice Settings）提供了對合成結果更精細的控制：

| 設定 | 作用 | 使用建議 |
|------|------|---------|
| `stability` | 語音穩定度——越高聲音越一致穩定，越低則每次合成會有些微自然變化 | 客服場景建議高穩定度，創意內容可降低 |
| `similarityBoost` | 相似度增強——控制與訓練語音的相似程度 | 需要精確模擬特定聲音時調高 |
| `style` | 風格強度——控制語音的情感表達強度 | 朗讀故事時可調高 |
| `useSpeakerBoost` | 說話者增強——強化說話者特徵，讓聲音更清晰 | 通常建議開啟 |
| `speed` | 語速控制——範圍 0.5 到 2 倍 | 預設 1 倍，電話場景常調為 0.9-1.1 倍 |

### 6.3 支援的語音模型

ElevenLabs 提供了三種不同定位的語音合成模型（`source-repo/extensions/elevenlabs/speech-provider.ts:34-38`），分別針對不同的使用場景：

| 模型 ID | 定位特色 | 適用場景 |
|---------|---------|---------|
| `eleven_multilingual_v2` | 多語言支援，品質最高（預設模型） | 需要支援多種語言的國際化應用 |
| `eleven_turbo_v2_5` | 低延遲快速合成，犧牲少量品質換取速度 | 即時對話、電話通話等對延遲敏感的場景 |
| `eleven_monolingual_v1` | 英語專用，針對英語做了專門優化 | 只需要英語的場景，可能有更好的英語發音 |

在選擇模型時，需要在品質和速度之間做權衡：`eleven_multilingual_v2` 品質最好但延遲較高，`eleven_turbo_v2_5` 速度最快但品質稍遜。對於我們研究的 Discord 語音機器人場景，`eleven_turbo_v2_5` 可能是更好的選擇，因為在即時語音對話中，延遲比完美的音質更重要。

### 6.4 API 端點

主要的語音合成函式 `elevenLabsTTS()` 位於 `source-repo/extensions/elevenlabs/tts.ts:65`，它向 ElevenLabs 的 REST API 發送 HTTP 請求：

```
POST {baseUrl}/v1/text-to-speech/{voiceId}
```

這是一個標準的 RESTful 端點設計——基礎 URL 加上資源路徑（`text-to-speech`）和語音角色 ID。請求體（body）包含要合成的文字和語音設定參數，回應則是一段音訊串流。

---

## 七、Deepgram 語音辨識擴充

### 7.1 概念說明

如果說 ElevenLabs 讓 AI「開口說話」，那 Deepgram 就是讓 AI「聽懂人話」。Deepgram 是 OpenClaw 用於語音辨識（Speech-to-Text, STT）的主要雲端供應商，`extensions/deepgram/` 共 282 行程式碼，核心實作在 `audio.ts`（87 行）。相比 ElevenLabs 擴充的一千多行，Deepgram 擴充的精簡反映了語音辨識 API 在使用上比語音合成要簡單得多——基本上就是把音訊送出去、拿回文字。

### 7.2 核心函式與配置

Deepgram 擴充的核心是一個語音轉文字的函式：

```typescript
// source-repo/extensions/deepgram/audio.ts:30
transcribeDeepgramAudio()
```

這個函式接收音訊資料，透過 HTTP 請求送到 Deepgram 的雲端服務進行辨識，然後回傳辨識出的文字。其配置參數如下：

| 配置項 | 預設值 | 說明 |
|--------|--------|------|
| API 端點 | `POST {baseUrl}/listen` | 語音辨識的 REST 端點 |
| 預設 BaseURL | `https://api.deepgram.com/v1` | Deepgram 的雲端 API 位址 |
| 預設模型 | `nova-3` | Deepgram 最新一代的語音辨識引擎 |

Deepgram 的 `nova-3` 模型是其旗艦級的語音辨識引擎，以高準確率和低延遲著稱。對於中文語音辨識的場景，模型的選擇可能需要根據 Deepgram 官方文件另做調整，但 `nova-3` 在英語和多數主流語言上的表現都相當優異。

值得注意的是，Deepgram 擴充處理的是「非即時」的語音辨識——將一段完整的音訊檔案送出去辨識。「即時」的語音辨識（邊說邊辨識）則由前面介紹的 Realtime Transcription 子系統處理，兩者雖然目標類似但實作方式不同。

---

## 八、Talk Voice 擴充（語音管理指令）

### 8.1 概念說明

`extensions/talk-voice/`（558 行）是面向終端使用者的語音管理工具，提供了簡潔的聊天指令來查詢和切換 AI 的語音角色。如果你把前面介紹的所有子系統想像成一個專業的錄音室，Talk Voice 就是那個簡單易用的控制面板——使用者不需要了解背後的供應商註冊、配置解析和容錯機制，只需要用幾個指令就能挑選和更換 AI 的聲音。

### 8.2 指令一覽

使用者可透過 `/voice` 或 `/talkvoice` 前綴來觸發語音管理指令（`source-repo/extensions/talk-voice/index.ts`）：

| 指令語法 | 說明 | 範例 |
|---------|------|------|
| `/voice list [provider]` | 列出可用的語音角色 | `/voice list elevenlabs` |
| `/voice get` | 查詢目前正在使用的語音角色 | `/voice get` |
| `/voice set <voice_id> [provider]` | 設定新的語音角色 | `/voice set Rachel elevenlabs` |

使用上的重要限制：

- **權限控制**：所有語音管理指令都需要管理員權限（admin-protected）。系統會在執行指令前進行權限檢查，一般使用者無法隨意更改 AI 的語音設定。這在多人共用的團隊環境中很有意義——你不會希望每個成員都能把客服機器人的聲音改成搞笑風格。
- **列表限制**：`/voice list` 最多顯示前 50 個語音角色。這是一個合理的分頁限制，因為像 ElevenLabs 這樣的供應商可能有數百甚至數千個可用語音，一次全部列出既不實用也會造成訊息過長。

---

## 九、Voice Call 擴充（電話通話整合）

### 9.1 概念說明

Voice Call 擴充是整個語音系統中規模最大、複雜度最高的元件，以 **17,260 行**程式碼佔據了語音系統總程式碼量的絕大多數。它的存在讓 OpenClaw 超越了「聊天機器人」的範疇，進入了傳統電話系統（telephony）的世界——使用者可以撥打一個電話號碼，然後直接與 AI 助手進行自然的語音對話。

電話整合之所以如此複雜，是因為它需要同時處理多個截然不同的技術領域：電信供應商的 Webhook 協議（每家都不一樣）、即時媒體串流的音訊編解碼、WebSocket 連線管理、以及嚴格的安全驗證（防止偽造的通話請求導致費用損失或安全漏洞）。相比之下，文字聊天只需要處理 HTTP 請求和 JSON 格式的訊息就夠了。

### 9.2 支援的電話供應商

OpenClaw 支援三家主流的雲端電話服務供應商，每家都有獨立的適配實作：

| 供應商 | 實作檔案 | 程式碼行數 | 市場定位 |
|--------|---------|-----------|---------|
| Twilio | `source-repo/extensions/voice-call/src/providers/twilio.ts` | 790 行 | 全球最大的雲端通訊平台，功能最完整 |
| Plivo | `source-repo/extensions/voice-call/src/providers/plivo.ts` | 601 行 | 成本效益導向的替代方案 |
| Telnyx | `source-repo/extensions/voice-call/src/providers/telnyx.ts` | 357 行 | 新興的全棧電信平台 |

Twilio 的實作最為完整（790 行），這不僅因為 Twilio 本身的 API 功能更豐富，也因為它可能是最多使用者選擇的供應商，因此收到了更多的功能需求和邊界情況處理。Plivo（601 行）作為第二大的實作，功能覆蓋度也相當完整。Telnyx（357 行）則相對精簡，但仍然涵蓋了電話通話的核心流程。

三者之間的程式碼差異主要來自：各供應商的 API 格式和認證機制不同、Webhook 事件結構不同、媒體串流的建立方式不同。但在邏輯層面，它們都實現了相同的通話生命週期：接聽來電 → 透過 Webhook 建立控制通道 → 透過 WebSocket 建立媒體通道 → 雙向音訊交換 → 掛斷並清理資源。

### 9.3 基礎設施元件

除了三個供應商的適配層之外，Voice Call 擴充還包含五個通用的基礎設施元件：

| 元件 | 檔案 | 行數 | 核心職責 |
|------|------|------|---------|
| Webhook 安全 | `source-repo/extensions/voice-call/src/webhook-security.ts` | 994 | 驗證來自電話供應商的每一個 Webhook 請求的真實性和完整性 |
| Webhook 處理 | `source-repo/extensions/voice-call/src/webhook.ts` | 716 | 處理各種 Webhook 事件：來電通知、通話狀態更新、錄音完成等 |
| 媒體串流 | `source-repo/extensions/voice-call/src/media-stream.ts` | 632 | 管理透過 WebSocket 進行的雙向即時音訊串流 |
| 配置管理 | `source-repo/extensions/voice-call/src/config.ts` | 625 | 電話整合的完整配置解析與驗證邏輯 |
| 主插件 | `source-repo/extensions/voice-call/src/index.ts` | 538 | 插件進入點、路由註冊、與 OpenClaw 核心的銜接 |

**Webhook 安全**以 994 行成為最大的單一檔案，這個數字本身就說明了電話系統中安全驗證的複雜性。每個電話供應商都有自己的 Webhook 簽章機制：Twilio 使用 HMAC-SHA1 或 HMAC-SHA256 簽章、Plivo 使用不同的驗證邏輯、Telnyx 也有自己的一套。`webhook-security.ts` 需要針對每個供應商分別實作驗證邏輯，同時還要處理各種邊界情況，例如重放攻擊（replay attack）防護、時間戳過期檢查等。這是整個語音系統中安全最敏感的部分，任何漏洞都可能導致未經授權的通話和相應的費用。

**媒體串流**（`media-stream.ts`）使用 **muLaw 編碼**進行音訊傳輸。muLaw（μ-law）是北美和日本電話系統的標準音訊壓縮演算法，它以對數方式壓縮音訊動態範圍，特別適合人聲頻率範圍的壓縮。音訊透過 WebSocket 協議進行雙向傳輸——WebSocket 的全雙工特性使得發送和接收可以同時進行，不需要像 HTTP 那樣等待請求回應。這個設計與即時語音子系統的 `RealtimeVoiceBridge` 介面中 `onAudio(muLaw: Buffer)` 回呼完全對應，體現了系統在不同層級之間的介面一致性。

---

## 十、Talk Mode 與 Voice Wake

### 10.1 概念說明

Talk Mode（對話模式）和 Voice Wake（語音喚醒）是 OpenClaw 語音系統中面向終端使用者的高階功能，它們構建在前面介紹的所有底層子系統之上，將複雜的語音技術包裝成直覺的使用者體驗。

**Talk Mode** 是一種持續性的語音對話模式——啟用後，AI 會持續聆聽使用者的語音，並以語音方式回覆。你不需要像對講機那樣按住按鈕說話，而是像跟真人聊天一樣自然地說話。AI 會自動判斷你什麼時候說完、什麼時候輪到它回答。

**Voice Wake** 則是語音喚醒機制——類似於 Apple 的「Hey Siri」或 Amazon 的「Alexa」，使用者可以透過說出特定的喚醒詞來啟動 Talk Mode。這消除了在螢幕上找到並點擊啟動按鈕的需求，讓語音互動變得更加無縫和自然。

### 10.2 平台支援矩陣

根據官方說明（`source-repo/README.md:180, 207-208, 259`），Talk Mode 和 Voice Wake 的功能在不同平台上有不同的實現程度：

| 平台 | Talk Mode | Voice Wake | 語音合成方案 | 其他特色 |
|------|-----------|------------|-------------|---------|
| macOS | ✅ 桌面覆蓋層介面 | ✅ 喚醒詞觸發 | MLX 本地語音 + 系統 TTS 備援 | 實驗性本地 MLX 語音供應商 |
| iOS | ✅ 完整支援 | ✅ 完整支援 | 系統 TTS + 雲端供應商 | 相機整合 |
| Android | ✅ 連續語音模式 | — 尚未支援 | ElevenLabs + 系統 TTS 備援 | `talk.speak` 閘道播放 |

幾個值得注意的平台差異：

- **macOS 的 MLX 本地語音**是一個重要的技術亮點。MLX 是 Apple 針對 Apple Silicon 晶片優化的機器學習框架，OpenClaw 利用它在 Mac 上本地執行語音合成模型，完全不需要網路連線。這不僅降低了延遲（不用等待網路往返），也保護了隱私（語音資料不用送到雲端），並且在離線環境中仍然可以使用語音功能。
- **Android 使用 `talk.speak` 閘道**進行語音播放管理，這是一個中間層，負責協調語音合成的輸出和裝置揚聲器的播放，包括處理播放中被打斷的情況。
- **iOS 同時支援 Voice Wake 和相機整合**，這暗示了 OpenClaw 在 iOS 平台上有更豐富的多模態（multimodal）能力。

### 10.3 功能演進歷程

透過追蹤 CHANGELOG（`source-repo/CHANGELOG.md`）中的相關條目，我們可以清楚看到 Talk Mode 從基礎功能逐步演進為成熟特性的過程：

| PR 編號 | 時間軸位置 | 內容描述 | 技術意涵 |
|---------|----------|---------|---------|
| #38445 | `CHANGELOG.md:2105` | macOS Talk Mode：將語音辨識的 `taskHint` 設定為 `.dictation` | 最早期的 Talk Mode 實作，使用 Apple 原生的語音辨識 API。`.dictation` 提示告訴系統這是口述場景而非指令場景，有助於提高連續語音的辨識準確度 |
| #31965 | `CHANGELOG.md:2650` | sherpa-onnx-tts 技能：改為 ESM 執行並新增回歸測試 | 引入了基於 ONNX 的本地 TTS 方案，這是探索邊緣裝置語音合成的早期嘗試 |
| #50849 | `CHANGELOG.md:1485` | Android：將語音合成移至 `talk.speak` 閘道，播放切換為 final-response | 重構了 Android 的語音播放架構，引入中間閘道來統一管理語音輸出 |
| #58490 | `CHANGELOG.md:867` | macOS Voice Wake：新增語音喚醒選項以觸發 Talk Mode | 關鍵里程碑——Voice Wake 功能首次登場，讓使用者可以免手動啟動語音模式 |
| #60306, #61164, #61214 | `CHANGELOG.md:502` | Android：在語音被明確停止時，取消進行中的 `talk.speak` 播放 | 修復了打斷機制的問題，確保使用者說話時 AI 的語音能立即停止 |
| #62459 | `CHANGELOG.md:101` | macOS：首次授予麥克風權限後，自動繼續啟動 Talk Mode | 改善了首次使用體驗——之前使用者授權後還需要手動再次啟動 |
| #63539 | `CHANGELOG.md:13` | macOS：新增實驗性本地 MLX 語音供應商，含明確供應商選擇、本地播放、打斷處理和系統語音備援 | 最新也最重要的更新——引入了本地 MLX 語音合成，代表 OpenClaw 開始認真經營邊緣端的語音能力 |

從這個時間線可以觀察到幾個明確的發展趨勢：

1. **從雲端到本地**：早期完全依賴雲端供應商，後來逐步引入 sherpa-onnx 和 MLX 等本地方案，追求更低延遲和更好的隱私保護。
2. **打斷機制的成熟**：多個 PR 都涉及「打斷」相關的修復和改進，這是自然語音對話中最難處理的技術挑戰之一——AI 說話時使用者突然插嘴，系統需要立即停止 AI 的語音輸出並切換到聆聽模式。
3. **平台特化**：每個平台都有針對性的優化，macOS 利用 MLX 和 Apple 原生 API，Android 則建構了自己的播放閘道架構。

---

## 十一、系統整合視角

在分別理解了各個子系統之後，讓我們退後一步，從整合的角度看看這些元件如何協同工作。這個視角對於理解 OpenClaw 的整體設計哲學尤為重要，也是我們日後進行二次開發（如 Discord 語音機器人）時最需要掌握的知識。

### 11.1 語音通話的完整資料流

當一通電話打進來時，資料在系統中的流動路徑清楚地展示了各子系統之間的協作關係。以下是以 Twilio 為例的完整流程：

```
使用者撥打電話號碼
  ↓
[Twilio 雲端] → 向 OpenClaw 發送 Webhook HTTP 請求
  ↓
webhook-security.ts — 驗證 Twilio 的請求簽章，確認請求來源合法
  ↓
webhook.ts — 解析 Webhook 事件內容，判斷這是一通新來電
  ↓
media-stream.ts — 與 Twilio 建立 WebSocket 連線，開始串流 muLaw 音訊
  ↓
RealtimeVoiceBridge.connect() — 透過即時語音子系統建立與 AI 的語音橋接
  ↓
triggerGreeting() — AI 主動開口打招呼（「你好，請問有什麼我可以幫忙的？」）
  ↓
┌──────────────────────────────────────────────────┐
│  雙向音訊迴圈（持續到掛斷為止）:                       │
│                                                    │
│  使用者說話                                          │
│    → 麥克風音訊 → Twilio → WebSocket → sendAudio()  │
│    → AI 語音模型即時處理                               │
│                                                    │
│  AI 回覆                                            │
│    → onAudio() 回呼 → WebSocket → Twilio → 聽筒播放  │
│                                                    │
│  （期間可能觸發工具呼叫）                               │
│    → onToolCall() → 呼叫外部工具                      │
│    → submitToolResult() → AI 繼續回覆                │
│                                                    │
│  （使用者打斷 AI 說話）                                │
│    → 偵測到使用者語音 → onClearAudio() → 清除待播音訊    │
└──────────────────────────────────────────────────┘
  ↓
使用者掛斷電話
  ↓
RealtimeVoiceBridge.close() — 清理語音資源
  ↓
Webhook 接收通話結束事件 — 記錄通話時長、更新狀態
```

這個流程涵蓋了至少五個子系統的協作：Voice Call 擴充處理電話供應商的介接、媒體串流處理音訊傳輸、即時語音子系統提供橋接抽象、TTS 子系統可能參與語音合成、即時轉錄子系統可能參與語音辨識。

### 11.2 TTS 的決策流程

文字轉語音的流程看似簡單（「把文字變成聲音」），但實際上包含了一系列精細的決策和處理步驟。以下是 `textToSpeech()` 高階 API 的完整決策流程：

```
AI 產生文字回覆
  ↓
shouldAttemptTtsPayload() — 多層級判斷是否需要語音
  ├── 第一優先：檢查當前 session 的 ttsAuto 設定
  ├── 第二優先：檢查使用者個人偏好設定檔
  ├── 第三優先：檢查全域配置的 auto 欄位
  └── 第四優先：檢查全域配置的 enabled 布林值
  ↓
  若決定不需要語音 → 直接回傳文字
  若決定需要語音 ↓
  ↓
文字長度檢查：超過 4,096 字元？
  ├── 是 → 使用語言模型以 temperature=0.3 進行摘要
  │         摘要完成後繼續下面的流程
  └── 否 → 使用原始文字繼續
  ↓
parseTtsDirectives() — 解析文字中的 [[tts:...]] 指令
  （提取語音、模型、速度等覆蓋設定）
  ↓
選擇供應商 → 嘗試主要供應商進行合成
  ├── 成功 → 回傳 TtsResult { success: true, audioPath, latencyMs }
  └── 失敗 → 記錄失敗原因，嘗試備援供應商（fallover）
       ├── 備援成功 → 回傳 TtsResult { success: true, fallbackFrom: "..." }
       └── 所有供應商都失敗 → 回傳 TtsResult { success: false, attempts: [...] }
```

### 11.3 設計模式總結

回顧整個語音與媒體系統，可以歸納出幾個貫穿始終的核心設計模式：

**一、插件化供應商架構**：三個核心子系統都不直接實作任何供應商邏輯，而是透過統一的註冊模式讓外部擴充套件提供實作。這使得新增和替換供應商成為零侵入性的操作。

**二、層級覆蓋的配置體系**：從全域配置到使用者偏好再到 Session 級設定，每一層都可以覆蓋上一層的設定。這種設計在多租戶（multi-tenant）環境中特別有價值——管理員設定全域預設、每個使用者可以有自己的偏好、每次對話可以臨時調整。

**三、優雅降級與容錯**：Speech Core 的多供應商容錯機制、macOS 的「MLX 本地語音 + 系統 TTS 備援」、Android 的「ElevenLabs + 系統 TTS 備援」，都體現了「盡可能提供服務，即使不是最佳品質」的設計理念。

**四、介面一致性**：即時語音的 `onAudio(muLaw: Buffer)` 回呼與電話整合的 muLaw 媒體串流使用相同的資料格式和抽象層級，讓不同來源的音訊可以無縫銜接。

---

## 引用來源

| 編號 | 來源檔案 | 引用範圍 | 主題 |
|------|---------|---------|------|
| 1 | `source-repo/src/realtime-voice/provider-types.ts` | 3-66 | 即時語音型別定義 |
| 2 | `source-repo/src/realtime-voice/provider-registry.ts` | 10-45 | 即時語音供應商註冊 |
| 3 | `source-repo/src/realtime-transcription/provider-types.ts` | 3-33 | 即時轉錄型別定義 |
| 4 | `source-repo/src/realtime-transcription/provider-registry.ts` | 10-49 | 即時轉錄供應商註冊 |
| 5 | `source-repo/src/tts/tts.ts` | 全檔案 | TTS 主要匯出入口 |
| 6 | `source-repo/src/tts/tts-auto-mode.ts` | 4 | TTS 自動模式定義 |
| 7 | `source-repo/src/tts/tts-config.ts` | 9-65 | TTS 配置解析 |
| 8 | `source-repo/src/tts/provider-registry.ts` | 10-45 | TTS 供應商註冊 |
| 9 | `source-repo/src/tts/directives.ts` | 41-56 | TTS 指令解析 |
| 10 | `source-repo/extensions/speech-core/src/tts.ts` | 76-1018 | Speech Core 完整實作 |
| 11 | `source-repo/extensions/elevenlabs/speech-provider.ts` | 34-55 | ElevenLabs 配置與模型 |
| 12 | `source-repo/extensions/elevenlabs/tts.ts` | 65 | ElevenLabs TTS 函式 |
| 13 | `source-repo/extensions/deepgram/audio.ts` | 30 | Deepgram 轉錄函式 |
| 14 | `source-repo/extensions/talk-voice/index.ts` | 全檔案 | Talk Voice 指令 |
| 15 | `source-repo/extensions/voice-call/src/providers/twilio.ts` | 全檔案 | Twilio 電話整合 |
| 16 | `source-repo/extensions/voice-call/src/providers/plivo.ts` | 全檔案 | Plivo 電話整合 |
| 17 | `source-repo/extensions/voice-call/src/providers/telnyx.ts` | 全檔案 | Telnyx 電話整合 |
| 18 | `source-repo/extensions/voice-call/src/webhook-security.ts` | 全檔案 | Webhook 安全驗證 |
| 19 | `source-repo/extensions/voice-call/src/webhook.ts` | 全檔案 | Webhook 處理 |
| 20 | `source-repo/extensions/voice-call/src/media-stream.ts` | 全檔案 | 媒體串流（muLaw） |
| 21 | `source-repo/extensions/voice-call/src/config.ts` | 全檔案 | Voice Call 配置 |
| 22 | `source-repo/extensions/voice-call/src/index.ts` | 全檔案 | Voice Call 主插件 |
| 23 | `source-repo/README.md` | 180, 207-208, 259 | Talk Mode / Voice Wake 說明 |
| 24 | `source-repo/CHANGELOG.md` | 13, 101, 502, 867, 1485, 2105, 2650 | 語音功能演進歷程 |
