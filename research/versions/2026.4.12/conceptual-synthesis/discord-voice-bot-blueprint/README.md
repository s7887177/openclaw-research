# Discord 語音社交機器人藍圖

> **「我想做一個真的能交到朋友的 AI。」** —— Eason

## 本章摘要

這份藍圖是整個 OpenClaw 研究專案中最核心的應用設計文件。它將 Eason 的夢想——一個能加入 Discord 語音頻道、有禮貌地聆聽、在適當時機插話、記住每個人、並以「交朋友」為目標的 AI——拆解為可執行的工程方案。

本藍圖基於對 OpenClaw v2026.4.12 原始碼的深度分析，涵蓋從語音輸入管線到社交智慧引擎的完整技術棧。每個設計決策都有 OpenClaw 現有程式碼的支撐，每個缺口都有明確的開發路徑。

## 願景

想像一個 Discord 語音頻道：四五個朋友在閒聊，AI 也在裡面。它不是那種「Hey Siri」才會回應的機器人——它就像一個安靜但熱情的新朋友：

- 🎧 **聆聽**：持續接收頻道中每個人的聲音
- 🧠 **理解**：即時轉文字，知道「誰」在說「什麼」
- 💾 **記憶**：記住每個人的名字、喜好、上次聊了什麼
- 🤔 **判斷**：知道什麼時候該說話、什麼時候該安靜
- 🗣️ **發言**：用自然的語音回應，有自己的語調和個性
- 🤝 **社交**：有禮貌、有幽默感、能建立長期關係

## 文件結構

| 檔案 | 內容 | 字數 |
|------|------|------|
| [01-願景與架構設計.md](./01-願景與架構設計.md) | 完整使用場景、需求拆解、系統架構、五大管線設計 | ~7,000 |
| [02-能力評估與技術深潛.md](./02-能力評估與技術深潛.md) | OpenClaw 已有 vs 需開發能力、關鍵程式碼分析、技術瓶頸 | ~5,500 |
| [03-實作路線圖與風險評估.md](./03-實作路線圖與風險評估.md) | 五階段路線圖、成本估算、風險矩陣、Agent 人格設計 | ~5,000 |

## 核心架構總覽

```
┌─────────────────────────────────────────────────────────────────┐
│                    Discord Voice Channel                        │
│  👤 Alice  👤 Bob  👤 Carol  🤖 ClawBot                        │
└────────┬────────────────────────────────────────────┬───────────┘
         │ Opus audio streams (per-user)              │ Opus playback
         ▼                                            ▲
┌─────────────────────┐                    ┌─────────────────────┐
│  Listening Pipeline  │                    │  Speaking Pipeline   │
│  ┌───────────────┐  │                    │  ┌───────────────┐  │
│  │ Opus Decode   │  │                    │  │ TTS Synthesis  │  │
│  │ (@discordjs/  │  │                    │  │ (OpenAI /      │  │
│  │  opus)        │  │                    │  │  ElevenLabs)   │  │
│  ├───────────────┤  │                    │  ├───────────────┤  │
│  │ WAV Build     │  │                    │  │ Audio Resource │  │
│  │ (48kHz/16bit) │  │                    │  │ → AudioPlayer  │  │
│  ├───────────────┤  │                    │  └───────────────┘  │
│  │ STT Transcribe│  │                    └─────────────────────┘
│  │ (Deepgram /   │  │                              ▲
│  │  Whisper)     │  │                              │
│  ├───────────────┤  │                    ┌─────────────────────┐
│  │ Speaker ID    │  │                    │  Response Generator  │
│  │ (userId map)  │  │                    │  ┌───────────────┐  │
│  └───────────────┘  │                    │  │ Agent Command  │  │
└──────────┬──────────┘                    │  │ (agentCommand  │  │
           │ transcript + speakerId        │  │  FromIngress)  │  │
           ▼                               │  └───────────────┘  │
┌─────────────────────────────────┐        └──────────▲──────────┘
│    Context Tracking Engine       │                   │
│  ┌────────────┐ ┌────────────┐  │        ┌─────────────────────┐
│  │ Room State │ │ Topic      │  │        │ Interjection Engine  │
│  │ (who's     │ │ Detection  │  │        │  ┌───────────────┐  │
│  │  here)     │ │ (rolling)  │  │───────▶│  │ Rule Triggers │  │
│  ├────────────┤ ├────────────┤  │        │  │ + LLM Judge   │  │
│  │ Dialogue   │ │ Emotion    │  │        │  │ + Confidence  │  │
│  │ History    │ │ Signals    │  │        │  │   Threshold   │  │
│  └────────────┘ └────────────┘  │        │  └───────────────┘  │
└──────────┬──────────────────────┘        └─────────────────────┘
           │                                          ▲
           ▼                                          │
┌─────────────────────────────────────────────────────┘
│         Memory & Personality System
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  │ Per-Person   │  │ Active       │  │ Agent        │
│  │ Memory       │  │ Memory       │  │ Personality  │
│  │ (LanceDB)    │  │ (Recall/     │  │ (System      │
│  │              │  │  Capture)    │  │  Prompt)     │
│  └──────────────┘  └──────────────┘  └──────────────┘
└──────────────────────────────────────────────────────
```

## 關鍵技術發現

經過深度原始碼分析，以下是最重要的發現：

### ✅ OpenClaw 已經擁有的（直接可用）

1. **Discord Voice Manager**（`extensions/discord/src/voice/manager.ts`）：完整的語音頻道連接、音訊捕獲、Opus 解碼、STT 轉寫、TTS 回放管線
2. **Per-user 音訊串流**：Discord.js `connection.receiver.subscribe(userId)` 已實現分軌接收
3. **Speaker Identity 解析**：`resolveSpeakerContext()` 已能將 userId 映射為暱稱/用戶名
4. **TTS 多 Provider**：OpenAI、ElevenLabs、sherpa-onnx、Microsoft 等
5. **LanceDB 向量記憶**：嵌入、儲存、語義搜尋
6. **Active Memory**：自動回憶/捕獲循環
7. **Auto-Join**：`voice.autoJoin` 設定自動加入語音頻道

### 🔧 需要擴展的

1. **目前是「被動模式」**：收到語音 → 轉文字 → 回應。需要新增「主動插話」邏輯
2. **Session 是 per-channel 而非 per-user**：語音頻道共用一個 session，需要在記憶層做 per-person 隔離
3. **沒有「房間狀態」追蹤**：需要新增 Context Tracking Engine
4. **沒有插話決策**：需要新增 Interjection Decision Engine

### ❌ 需要新開發的

1. **插話決策引擎**：核心創新——何時該說話
2. **房間狀態管理器**：誰在場、話題追蹤、對話滾動 window
3. **Per-person 記憶隔離層**：基於現有 LanceDB，加上 person-scoped 查詢
4. **社交行為控制器**：頻率控制、禮貌策略、關係追蹤

## 延伸閱讀

- [AI Agent 基礎概念](../../ai-agent-foundations/) — 理解 Agent 架構
- [OpenClaw 系統架構](../../architecture/) — 核心架構分析
- [記憶系統深度研究](../../deep-dive/memory-system/) — 記憶系統詳解
- [語音能力研究](../../deep-dive/voice-capabilities/) — 語音管線詳解
- [Discord 整合研究](../../deep-dive/discord-integration/) — Discord 擴充詳解

---

*這份藍圖不是紙上談兵。每一個設計決策都有 OpenClaw 原始碼的支撐，每一個「需要開發」的能力都有明確的技術路徑。Eason 的夢想，從這裡開始落地。*
