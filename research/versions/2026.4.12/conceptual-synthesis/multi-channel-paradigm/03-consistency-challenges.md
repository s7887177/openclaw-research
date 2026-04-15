# 多通道一致性的挑戰

> **所屬 Topic**：多通道範式（Multi-Channel Paradigm）  
> **層級**：Level 6 — 概念綜合層  
> **前置閱讀**：Level 3 Sessions / Routing / Presence、Level 3 Channel 適配器模式

---

## 摘要

統一多通道帶來的最深層挑戰不是技術性的 API 封裝，而是三個概念層面的一致性問題：人格一致性（同一個 AI 在不同平台是否應該「一樣」？）、跨通道記憶的設計選擇（共享還是隔離？）、以及回應格式的平台適配。本章從設計哲學的角度分析 OpenClaw 如何處理這些張力。

---

## 1. 人格一致性：Agent 在 Discord 和 Telegram 是否應該「一樣」？

### 1.1 問題的本質

這個問題的答案取決於「一樣」的定義：

| 維度 | 應該一致嗎？ | 理由 |
|------|------------|------|
| 核心人格（Core Persona） | ✅ 是 | AI 的性格、價值觀、知識應該跨平台一致 |
| 語氣（Tone） | 🔶 部分 | 在工作 Slack 可以正式些，在朋友 Discord 可以輕鬆些 |
| 回應格式（Format） | ❌ 否 | 應適配平台能力（Markdown vs 純文字） |
| 回應長度（Length） | ❌ 否 | IRC 需要極簡，Discord 可以詳細 |
| 可用功能（Features） | ❌ 否 | 取決於平台能力矩陣 |

### 1.2 OpenClaw 的設計選擇

OpenClaw 透過分層配置實現「統一核心，差異化呈現」：

```yaml
# openclaw.yaml 概念結構

# 第一層：全域 Agent 設定 — 跨所有通道一致
personality: |
  你是一個友善、有耐心的 AI 助手，名叫 Aria。
  你喜歡用類比來解釋複雜概念。
  你說話帶點幽默感，但知道何時該認真。

# 第二層：per-channel 配置覆蓋 — 通道差異化
channels:
  discord:
    personality_append: |
      在 Discord 你可以更隨意一些，使用 emoji。
    model: gpt-4o
    
  slack:
    personality_append: |
      在 Slack 保持專業語氣，簡潔回覆。
    model: gpt-4o-mini  # 工作場景用便宜模型節省成本
    
  twitch:
    personality_append: |
      在 Twitch 聊天室要簡短、有趣、跟上節奏。
      回覆不超過兩句話。
```

這個設計哲學可以類比為：一個人在不同場合的表現——在公司會議上正式專業，在朋友聚會上輕鬆幽默——但**核心性格不變**。

### 1.3 Agent Prompt 適配器

`ChannelPlugin` 介面中的 `agentPrompt` 適配器（`ChannelAgentPromptAdapter`，定義於 `src/channels/plugins/types.core.ts`）允許每個通道在 Agent 的 System Prompt 中注入平台特定的指引。例如：

- Discord 通道可能注入：「你在 Discord 環境中，支援 Markdown 格式和 Embed」
- IRC 通道可能注入：「你在 IRC 環境中，每條訊息限制 350 字元，不支援格式」
- Twitch 通道可能注入：「你在 Twitch 直播聊天中，保持簡短」

這讓 LLM 能夠根據當前平台的上下文自動調整回應風格。

---

## 2. Context 共享 vs 隔離：跨通道記憶的設計選擇

### 2.1 設計光譜

跨通道記憶的設計有一個完整的光譜：

```
完全隔離 ◀────────────────────────────────────▶ 完全共享

每個通道各自       同一使用者的不同通道      所有通道的所有
擁有獨立記憶       共享長期記憶               對話共享上下文
（最安全）         （平衡選擇）               （最無縫）
```

### 2.2 OpenClaw 的 Session 模型

OpenClaw 的 Session 系統（`src/channels/session.ts`）採用的是**以 Session Key 為中心**的模型：

```typescript
// source-repo/src/channels/session.ts:31-74
recordInboundSession(storePath, sessionKey, context, groupResolution)
```

Session Key 的組成通常是 `{channel}:{accountId}:{conversationId}`，這意味著：

- **同一平台的不同對話** → 不同 Session → 隔離的短期上下文
- **不同平台的同一使用者** → 不同 Session → 隔離的短期上下文
- **長期記憶（Memory）** → 由記憶系統管理 → **可以跨 Session 共享**

### 2.3 三層記憶的跨通道行為

| 記憶層級 | 跨通道行為 | 設計理由 |
|---------|-----------|---------|
| **短期記憶（Session Context）** | 隔離 | 每個對話有自己的上下文窗口，避免串話 |
| **中期記憶（Active Memory）** | 部分共享 | 同一 Agent 的活躍記憶可被不同 Session 存取 |
| **長期記憶（Memory Core / LanceDB）** | 共享 | 所有通道寫入同一個記憶庫，搜尋時不限通道 |

這是一個務實的設計：

- 短期隔離確保不會在 Discord 的遊戲討論中突然出現 Slack 工作對話的上下文
- 長期共享確保「我之前跟你提過的那件事」可以跨平台回憶起來

### 2.4 執行緒綁定（Thread Binding）的跨通道意義

`src/channels/thread-bindings-policy.ts` 中的執行緒綁定策略進一步細化了上下文管理：

```typescript
type SessionThreadBindingsConfigShape = {
  enabled?: unknown;
  idleHours?: unknown;      // 預設 24 小時
  maxAgeHours?: unknown;    // 預設 0（無限制）
  spawnSubagentSessions?: unknown;
  spawnAcpSessions?: unknown;
};
```

執行緒綁定允許在支援 Thread 的平台（Discord、Slack、Matrix）上建立**子對話**，每個子對話有獨立的上下文但共享 Agent 的人格和長期記憶。這在概念上類似於「在同一個辦公室裡，你可以跟同事在不同會議室討論不同話題，但你們都是同一個人」。

---

## 3. 回應格式適配：同一內容的多形態呈現

### 3.1 挑戰

Agent 產生的回應是「語義內容」，但不同平台需要不同的「呈現格式」。例如，同一段程式碼說明：

**在 Discord（支援完整 Markdown）：**
```markdown
## 解法

使用遞迴：

\`\`\`python
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
\`\`\`

> 💡 時間複雜度：O(2^n)，可用備忘錄優化
```

**在 IRC（350 字元限制，有限格式）：**
```
fibonacci: if n<=1 return n; else return fib(n-1)+fib(n-2). 時間O(2^n), 可用 memo 優化。
```

**在 WhatsApp（自訂標記）：**
```
*解法*

使用遞迴：
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

💡 時間複雜度：O(2^n)，可用備忘錄優化
```

### 3.2 OpenClaw 的適配策略

OpenClaw 採用三層適配策略：

**第一層：LLM 自適應**
透過 `agentPrompt` 適配器，LLM 在生成回應時就知道目標平台的格式限制。這是最自然的適配——讓 AI 直接用對的格式寫。

**第二層：出站淨化（Sanitize）**
`ChannelOutboundAdapter.sanitizeText()` 在訊息發送前進行平台特定的文字清理：
- 移除不支援的 Markdown 語法
- 轉義平台特殊字元
- 調整連結格式

**第三層：切分與重組（Chunk）**
`chunker` 和 `textChunkLimit` 確保長回應被智慧地切分為符合平台限制的片段，`chunkerMode: "markdown"` 確保切分時不會破壞 Markdown 區塊（如程式碼區塊）。

### 3.3 Markdown 能力宣告

`markdownCapable` 是 `ChannelMeta` 中的關鍵欄位，宣告在以下通道的配置中：

| 平台 | markdownCapable | 來源 |
|------|----------------|------|
| Discord | ✅ | `extensions/discord/package.json:43` |
| Telegram | ✅ | `extensions/telegram/package.json:34` |
| Slack | ✅ | `extensions/slack/package.json:29` |
| Signal | ✅ | `extensions/signal/package.json:24` |
| Google Chat | ✅ | `extensions/googlechat/package.json:41` |
| IRC | ✅ | `extensions/irc/src/channel.ts:60` |

其他平台未宣告 `markdownCapable`，OpenClaw 會據此調整回應格式。

---

## 4. 跨通道的邊界情境

### 4.1 群組 vs 私訊的行為差異

同一個通道（如 Discord）中，群組和私訊的行為也需要差異化：

- **群組**：需要提及偵測（`mentions` 適配器）、考慮是否每條訊息都回應
- **私訊**：可以直接回應所有訊息，語氣更個人化

OpenClaw 的確認反應系統（`AckReactionScope`）精確控制了這個行為，允許配置「只在被提及時才反應」或「群組中所有訊息都反應」等策略。

### 4.2 多帳號 / 多 Agent 場景

一個使用者可能在同一個平台上有多個 OpenClaw Agent（例如：一個工作助手、一個遊戲助手）。Session Key 中的 `accountId` 確保它們的上下文完全隔離——即使在同一個 Discord 伺服器中，不同 Agent 帳號的記憶和對話歷史也不會混淆。

### 4.3 裝置配對與存取控制

某些通道（如 WhatsApp、BlueBubbles）需要與實體裝置配對才能運作。`pairing` 適配器處理這類通道的特殊設定流程，如 QR Code 掃描。這是多通道統一中的例外情境——有些通道天然需要額外的設定步驟。

---

## 引用來源

| 來源 | 路徑 / 說明 |
|------|-------------|
| Session 記錄 | `source-repo/src/channels/session.ts:31-74` |
| 執行緒綁定策略 | `source-repo/src/channels/thread-bindings-policy.ts:1-270` |
| 確認反應邏輯 | `source-repo/src/channels/ack-reactions.ts:1-104` |
| ChannelAgentPromptAdapter | `source-repo/src/channels/plugins/types.core.ts` |
| markdownCapable 宣告 | 見 3.3 節各平台原始碼引用 |
| Level 3 Sessions / Routing / Presence | `research/versions/2026.4.12/cross-cutting-mechanisms/sessions-routing-presence/` |
| Level 3 Channel 適配器模式 | `research/versions/2026.4.12/cross-cutting-mechanisms/channel-adapter-pattern/` |
