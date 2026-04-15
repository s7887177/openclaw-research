# 什麼是 AI Agent？——從第一性原理建立概念基礎

> **摘要**  
> 本章從最基本的定義出發，逐步建立 AI Agent 的完整概念框架。我們將釐清 Agent 與傳統聊天機器人的本質差異、解析 Agent 的核心能力架構（感知、推理、行動、記憶）、介紹 ReAct 範式與工具調用機制、探討多 Agent 協作與人格一致性問題，最後展示 OpenClaw 如何在程式碼層面體現這些概念。本章是整份研究的理論根基——所有後續章節的技術細節，都可以回溯到這裡建立的概念模型。

---

## 1. 定義：什麼是 Agent？

### 1.1 一個最小的定義

在人工智慧的語境中，**Agent**（代理人）有一個簡潔的定義：

> **Agent 是一個能夠感知環境、基於感知做出決策、並採取行動以達成目標的自主系統。**

這個定義有三個關鍵要素：

| 要素 | 含義 | 例子 |
|------|------|------|
| **感知**（Perception） | 從環境中接收資訊 | 讀取使用者訊息、觀察檔案系統、接收 API 回應 |
| **決策**（Decision-Making） | 基於感知到的資訊進行推理 | 分析問題、制定計畫、選擇工具 |
| **行動**（Action） | 對環境產生影響 | 執行命令、修改檔案、發送訊息 |

這個定義源自經典的 AI 理論——Russell & Norvig 在《Artificial Intelligence: A Modern Approach》中的 Rational Agent 概念。但在 LLM 時代，這個定義有了全新的實現方式。

### 1.2 LLM 時代的 Agent

在 2024-2026 年的語境中，當我們說「AI Agent」，通常指的是：

> **以大型語言模型（Large Language Model, LLM）作為核心推理引擎，透過工具調用（Tool Calling）與外部世界互動，並能在多個回合中持續執行任務的自主系統。**

關鍵區別在於：傳統 AI Agent（如機器人控制系統）的推理引擎可能是規則系統或強化學習模型，而現代 AI Agent 的推理引擎是 LLM——這帶來了前所未有的通用性和靈活性。

```
┌──────────────────────────────────────────────────┐
│                   AI Agent                        │
│                                                   │
│   ┌───────────┐   ┌───────────┐   ┌───────────┐ │
│   │   感知     │──▶│   推理     │──▶│   行動     │ │
│   │ Perception │   │ Reasoning │   │  Action   │ │
│   └───────────┘   └───────────┘   └───────────┘ │
│        ▲               │                │        │
│        │          ┌────┴────┐           │        │
│        │          │  LLM    │           │        │
│        │          │ (核心)   │           │        │
│        │          └─────────┘           │        │
│        │                                ▼        │
│   ┌─────────────────────────────────────────┐    │
│   │              環境 (Environment)          │    │
│   │   檔案系統 · API · 資料庫 · 通訊通道     │    │
│   └─────────────────────────────────────────┘    │
│                                                   │
│   ┌─────────────────────────────────────────┐    │
│   │              記憶 (Memory)               │    │
│   │   短期記憶 (Context Window)              │    │
│   │   長期記憶 (Vector Database)             │    │
│   └─────────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘
```

---

## 2. Agent vs Chatbot：本質差異

### 2.1 傳統 Chatbot 模型

一個傳統的聊天機器人（Chatbot）運作方式極其簡單：

```
使用者輸入 ──▶ 模型處理 ──▶ 文字輸出
  (input)      (process)     (output)
```

這是一個**單次映射**（Single-pass Mapping）：輸入進去，文字出來，結束。無論是基於規則的客服機器人，還是使用 ChatGPT API 的簡單應用，本質上都是這個模式。

### 2.2 Agent 模型

Agent 則完全不同。它不是一個單次映射，而是一個**迴圈**（Loop）：

```
         ┌──────────────────────────────────┐
         │                                  │
         ▼                                  │
   ┌───────────┐   ┌───────────┐   ┌──────┴────┐
   │  感知      │──▶│  推理      │──▶│  行動      │
   │ 讀取輸入   │   │ 分析+決策  │   │ 執行工具   │
   └───────────┘   └───────────┘   └───────────┘
                                        │
                                        ▼
                                  ┌───────────┐
                                  │  觀察      │
                                  │ 讀取結果   │
                                  └───────────┘
                                        │
                                  ┌─────┴─────┐
                                  │ 目標達成？  │
                                  └─────┬─────┘
                                   No │    │ Yes
                                      ▼    ▼
                               回到感知   結束
```

這個迴圈稱為 **Perception → Reasoning → Action → Observation** 循環。Agent 會反覆執行這個迴圈，直到它認為任務已經完成。

### 2.3 根本差異對照表

| 維度 | Chatbot | Agent |
|------|---------|-------|
| 執行模式 | 單次 input→output | 多回合迴圈 |
| 與環境互動 | 無（只處理文字） | 有（透過工具） |
| 狀態管理 | 無狀態或簡單歷史 | 完整的記憶系統 |
| 自主性 | 被動回應 | 主動規劃與執行 |
| 錯誤處理 | 回報錯誤 | 嘗試修復並重試 |
| 典型用途 | 問答、翻譯 | 程式碼撰寫、系統管理、自動化 |

### 2.4 一個具體的例子

假設使用者說：「幫我在專案裡找到所有未使用的依賴套件，然後移除它們。」

**Chatbot 的回應**：
> 你可以使用 `depcheck` 工具來找到未使用的依賴。先執行 `npx depcheck`，然後根據輸出結果使用 `npm uninstall` 移除。

**Agent 的行為**：
1. 感知：接收到使用者的請求
2. 推理：我需要先檢查專案結構，然後執行 depcheck
3. 行動：執行 `ls package.json`，確認專案存在
4. 觀察：package.json 存在，這是一個 Node.js 專案
5. 推理：執行 depcheck 來掃描
6. 行動：執行 `npx depcheck`
7. 觀察：找到 3 個未使用的依賴：`lodash`, `moment`, `chalk`
8. 推理：逐一移除這些依賴
9. 行動：執行 `npm uninstall lodash moment chalk`
10. 觀察：移除成功
11. 推理：任務完成，向使用者報告結果
12. 行動：發送完成訊息

這就是 Agent 的本質——它不只是給建議，它**直接做事**。而這正是 OpenClaw 的核心理念：

> *"OpenClaw is the AI that actually does things."*  
> — `source-repo/VISION.md:3`

---

## 3. Agent 的核心能力架構

一個完整的 AI Agent 需要四個核心能力模組，它們共同構成了 Agent 的「認知架構」（Cognitive Architecture）：

```
┌──────────────────────────────────────────────────────────────┐
│                    Agent 認知架構                              │
│                                                               │
│   ┌─────────┐   ┌─────────────┐   ┌─────────┐               │
│   │  感知    │──▶│    推理       │──▶│  行動    │              │
│   │         │   │             │   │         │               │
│   │ 多通道  │   │  LLM Core   │   │ 工具調用 │              │
│   │ 輸入    │   │  規劃引擎    │   │ API 呼叫 │              │
│   │ 解析    │   │  反思能力    │   │ 訊息發送 │              │
│   └─────────┘   └──────┬──────┘   └─────────┘               │
│                        │                                      │
│                   ┌────┴────┐                                 │
│                   │  記憶    │                                 │
│                   │         │                                 │
│                   │ 短期記憶 │  ← Context Window              │
│                   │ 長期記憶 │  ← Vector Database             │
│                   │ 工作記憶 │  ← Session Transcript          │
│                   └─────────┘                                 │
└──────────────────────────────────────────────────────────────┘
```

### 3.1 感知（Perception）

感知是 Agent 從環境中獲取資訊的能力。在 LLM Agent 的語境中，「環境」不只是使用者的文字輸入——它包括多種通道和資料來源。

**OpenClaw 的感知來源**：

| 來源 | 描述 | OpenClaw 實現 |
|------|------|---------------|
| 使用者訊息 | 來自各通道的文字/語音 | 25+ 通道適配器 |
| 工具執行結果 | 命令輸出、API 回應 | Tool Result 回傳 |
| 檔案系統 | 讀取檔案內容 | `read`, `ls`, `grep` 工具 |
| 網頁內容 | 抓取網頁資料 | `web_fetch`, `web_search` 工具 |
| 記憶回想 | 過去的對話與知識 | Context Engine `assemble()` |
| 系統事件 | Cron 觸發、Webhook | Hooks 與 Cron 系統 |
| 媒體 | 圖片、語音、影片 | Media Understanding 模組 |

在 OpenClaw 的架構中，感知能力透過 `RunEmbeddedPiAgentParams` 的參數結構來表達——這個型別定義了 Agent 在一次執行中能接收到的所有輸入：

```typescript
// source-repo/src/agents/pi-embedded-runner/run/params.ts:22-80
export type RunEmbeddedPiAgentParams = {
  sessionId: string;
  sessionKey?: string;
  agentId?: string;
  messageChannel?: string;       // 來源通道
  messageProvider?: string;      // 來源提供者
  prompt: string;                // 使用者提示
  images?: ImageContent[];       // 圖片輸入
  trigger?: EmbeddedRunTrigger;  // 觸發類型：user | cron | heartbeat | ...
  senderId?: string | null;      // 發送者身份
  senderIsOwner?: boolean;       // 是否為擁有者
  // ...
};
```

注意 `trigger` 欄位——它告訴 Agent 這次執行是由使用者觸發（`"user"`）、定時任務觸發（`"cron"`）、心跳觸發（`"heartbeat"`）還是記憶觸發（`"memory"`），展示了 Agent 感知環境的多元方式。

### 3.2 推理（Reasoning）

推理是 Agent 的核心——它基於感知到的資訊進行思考、分析和規劃。在 LLM Agent 中，推理引擎就是 LLM 本身。

**推理的層次**：

1. **反應式推理**（Reactive）：看到輸入，直接回應
2. **規劃式推理**（Planning）：分解任務為步驟，逐步執行
3. **反思式推理**（Reflective）：檢視自己的行為，修正錯誤
4. **元認知推理**（Meta-cognitive）：思考自己的思考過程

OpenClaw 透過 LLM 的 Extended Thinking（擴展思考）功能支援更深層的推理。在 `src/agents/pi-embedded-runner/thinking.ts` 中，系統處理 Claude 模型的 `thinking` 和 `redacted_thinking` 區塊，讓 Agent 能在回應之前進行深度思考。

系統提示詞（System Prompt）是引導推理的關鍵。OpenClaw 的 `buildAgentSystemPrompt()` 函數生成了結構化的提示詞，其中包含了安全規則、工具使用指南和行為準則：

```typescript
// source-repo/src/agents/system-prompt.ts:631-632
const lines = [
  "You are a personal assistant running inside OpenClaw.",
  // ...
];
```

這個看似簡單的身份宣言，定義了 Agent 推理的基本框架——它知道自己是誰、在哪裡運行、能用什麼工具。

### 3.3 行動（Action）

行動是 Agent 對環境產生影響的能力。在 OpenClaw 中，Agent 透過**工具調用**（Tool Calling）來執行行動。

OpenClaw 內建了豐富的工具集（見 `source-repo/src/agents/system-prompt.ts:433-466`）：

| 工具類別 | 工具名稱 | 功能 |
|----------|---------|------|
| **檔案操作** | `read`, `write`, `edit`, `apply_patch` | 讀寫檔案 |
| **搜尋導航** | `grep`, `find`, `ls` | 搜尋與瀏覽 |
| **系統執行** | `exec`, `process` | 執行命令 |
| **網路互動** | `web_search`, `web_fetch` | 網路搜尋 |
| **瀏覽器** | `browser` | 控制瀏覽器 |
| **視覺化** | `canvas` | 展示 Canvas |
| **通訊** | `message` | 發送訊息 |
| **排程** | `cron` | 定時任務 |
| **多 Agent** | `sessions_spawn`, `subagents` | 子 Agent 管理 |
| **媒體** | `image`, `image_generate` | 圖片分析與生成 |

### 3.4 記憶（Memory）

記憶是 Agent 保持上下文連貫性的基礎。沒有記憶，Agent 就只是一個無狀態的函數——每次對話都從零開始。

**記憶的三個層次**：

```
┌─────────────────────────────────────────────────────┐
│                   Agent 記憶系統                      │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  短期記憶 (Short-term Memory)                 │   │
│  │  = LLM 的 Context Window                     │   │
│  │  • 當前對話的最近幾個回合                       │   │
│  │  • 容量有限（4K ~ 200K tokens）                │   │
│  │  • 每次 API 呼叫時直接傳入                     │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  工作記憶 (Working Memory)                    │   │
│  │  = Session Transcript                         │   │
│  │  • 完整的對話歷史（存於磁碟）                   │   │
│  │  • 透過壓縮（Compaction）控制大小              │   │
│  │  • Context Engine 管理組裝與壓縮               │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  長期記憶 (Long-term Memory)                  │   │
│  │  = Vector Database (LanceDB)                  │   │
│  │  • 跨 Session 的知識儲存                       │   │
│  │  • 語義搜尋（Embedding + 向量相似度）           │   │
│  │  • 自動擷取與回想                              │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

OpenClaw 透過 **Context Engine** 介面統一管理記憶系統。這是一個可插拔的抽象層，定義於 `source-repo/src/context-engine/types.ts:150-281`：

```typescript
// source-repo/src/context-engine/types.ts:150-151
export interface ContextEngine {
  readonly info: ContextEngineInfo;
  
  // 初始化
  bootstrap?(params: { sessionId; sessionKey; sessionFile }): Promise<BootstrapResult>;
  
  // 注入新訊息
  ingest(params: { sessionId; message; isHeartbeat? }): Promise<IngestResult>;
  
  // 組裝上下文（核心方法）
  assemble(params: { 
    sessionId; messages; tokenBudget?; 
    availableTools?; prompt?; model? 
  }): Promise<AssembleResult>;
  
  // 壓縮上下文
  compact(params: { sessionId; tokenBudget?; force? }): Promise<CompactResult>;
  
  // 子 Agent 生命週期
  prepareSubagentSpawn?(params: { parentSessionKey; childSessionKey }): Promise<...>;
  onSubagentEnded?(params: { childSessionKey; reason }): Promise<void>;
}
```

`assemble()` 方法是記憶系統的核心——它在每次 Agent 推理前被調用，負責在有限的 Token 預算內組裝最相關的上下文。`compact()` 則在上下文超出預算時進行壓縮，例如生成對話摘要來替代完整歷史。

---

## 4. ReAct 範式：推理與行動的結合

### 4.1 什麼是 ReAct？

**ReAct**（Reasoning + Acting）是 2022 年由 Yao et al. 提出的 Agent 範式。它的核心思想是：讓 LLM 在生成行動之前先進行**顯式推理**，並在觀察行動結果後再次推理。

```
Think: 使用者想要移除未使用的依賴。我需要先找到哪些是未使用的。
Act: exec("npx depcheck")
Observe: Found unused dependencies: lodash, moment
Think: 找到了 2 個未使用的依賴。讓我移除它們。
Act: exec("npm uninstall lodash moment")
Observe: removed 2 packages
Think: 任務完成，向使用者報告。
Act: message("已移除 lodash 和 moment 兩個未使用的依賴。")
```

### 4.2 為什麼 ReAct 是主流？

ReAct 之前，Agent 設計有兩個主要方向：

1. **純推理**（Chain-of-Thought）：LLM 只進行推理，不與外部世界互動
2. **純行動**（Action-only）：LLM 直接選擇行動，不進行顯式推理

ReAct 將兩者結合，獲得了最佳效果：

| 方法 | 推理 | 行動 | 問題 |
|------|------|------|------|
| Chain-of-Thought | ✅ | ❌ | 無法驗證推理，容易幻覺 |
| Action-only | ❌ | ✅ | 缺乏規劃，行為隨機 |
| **ReAct** | ✅ | ✅ | 推理引導行動，行動驗證推理 |

### 4.3 OpenClaw 的 ReAct 實現

OpenClaw 的 ReAct 迴圈實現在 `pi-embedded-runner`（嵌入式 Agent 運行器）中。核心入口是 `runEmbeddedPiAgent()` 函數（`source-repo/src/agents/pi-embedded-runner/run.ts:166`），它負責啟動和管理整個推理-行動迴圈。

迴圈的每一次迭代稱為一個 **Attempt**（嘗試），實現在 `source-repo/src/agents/pi-embedded-runner/run/attempt.ts` 中。一個 Attempt 的流程是：

```
1. 組裝上下文   → ContextEngine.assemble()
2. 建構提示詞   → buildEmbeddedSystemPrompt()
3. 呼叫 LLM    → streamFn() / createAgentSession()
4. 處理工具呼叫  → 執行工具，收集結果
5. 檢查停止條件  → 是否有更多工具呼叫？是否超時？
6. 如有更多工具  → 回到步驟 1（新的 Attempt）
```

外層的 `runEmbeddedPiAgent()` 則處理更高層的控制流程：

- **重試邏輯**：如果 LLM 回傳錯誤（rate limit、context overflow），嘗試 failover 到其他模型
- **壓縮觸發**：如果上下文超出預算，觸發 Context Engine 的 `compact()`
- **身份驗證輪換**：如果 API key 額度用完，切換到下一個 auth profile
- **佇列管理**：透過 `enqueueCommandInLane()` 確保同一 Session 不會並發執行

```typescript
// source-repo/src/agents/pi-embedded-runner/run.ts:214-216
return enqueueSession(() => {
  throwIfAborted();
  return enqueueGlobal(async () => {
    // ... 主要執行邏輯
  });
});
```

這個雙層佇列設計確保了 **Session 層級序列化**（同一個 Session 的請求依序執行）和 **全域層級節流**（防止過多並發佔滿資源）。

---

## 5. Tool Use / Function Calling：Agent 與世界的介面

### 5.1 工具調用的本質

工具調用（Tool Calling / Function Calling）是 Agent 與外部世界互動的主要機制。它的運作方式是：

```
1. LLM 接收到使用者請求和可用工具列表
2. LLM 決定需要使用哪個工具，生成結構化的呼叫參數
3. Agent 框架截取這個呼叫，實際執行工具
4. 工具的執行結果被送回 LLM 作為觀察
5. LLM 基於觀察繼續推理或生成最終回應
```

這個機制讓 LLM 突破了「只能生成文字」的限制，獲得了在真實世界中行動的能力。

### 5.2 OpenClaw 的工具架構

OpenClaw 的工具系統有三個層次：

**第一層：核心工具**（Built-in Tools）

在 `buildAgentSystemPrompt()` 中定義的核心工具（`source-repo/src/agents/system-prompt.ts:433-466`），包括 `read`, `write`, `exec`, `web_search` 等。這些是 Agent 的基本能力。

**第二層：Plugin 工具**

透過 Plugin SDK 提供的擴展工具。Plugin 在 Gateway 進程內以可信任的方式運行（`source-repo/SECURITY.md:120-126`）：

> *"Plugins/extensions are part of OpenClaw's trusted computing base for a gateway. Installing or enabling a plugin grants it the same trust level as local code running on that gateway host."*

**第三層：MCP 工具**

透過 Model Context Protocol（MCP）橋接外部工具伺服器。OpenClaw 使用 `mcporter` 作為 MCP 橋接器（`source-repo/VISION.md:74-80`），保持核心精簡：

> *"This keeps MCP integration flexible and decoupled from core runtime: add or change MCP servers without restarting the gateway."*

### 5.3 工具策略（Tool Policy）

不是所有 Agent 都應該擁有所有工具。OpenClaw 透過 **Tool Policy** 機制控制工具可用性：

- `tools.profile`：預設工具集合（如 `"messaging"` 只允許通訊工具）
- Owner-only 工具：只有擁有者可以觸發的高權限工具
- 動態過濾：根據通道、發送者身份、群組設定動態調整

這確保了在不同信任等級的通道中，Agent 暴露的攻擊面不同。

---

## 6. Multi-Agent Systems：協作與分工

### 6.1 為什麼需要多個 Agent？

單個 Agent 在處理複雜任務時面臨幾個瓶頸：

1. **上下文限制**：一個 Session 的上下文窗口有限
2. **專業化需求**：不同任務需要不同的工具和配置
3. **並行執行**：某些任務可以同時進行
4. **隔離需求**：敏感操作需要獨立的沙箱環境

### 6.2 OpenClaw 的多 Agent 架構

OpenClaw 的多 Agent 系統透過以下機制實現：

**Agent 路由**（Agent Routing）

`resolveAgentRoute()` 函數（`source-repo/src/routing/resolve-route.ts`）負責將進入的訊息路由到正確的 Agent。路由依據包括：

- 通道類型和群組 ID
- 訊息目標和關鍵字
- 帳號綁定和策略設定

**Session Key 體系**

每個 Agent 執行都有一個 Session Key，用於隔離上下文。Session Key 的結構支援多種模式（`source-repo/src/routing/session-key.ts`）：

- 主 Agent Session：`agent:main:channel:userId`
- 子 Agent Session：`subagent:parentKey:childId`
- Cron Session：`cron:agentId:cronId`
- 群組 Session：`group:agentId:groupId`

**子 Agent 生成**

Agent 可以透過 `sessions_spawn` 工具生成子 Agent，實現任務委派：

```
主 Agent ──sessions_spawn──▶ 子 Agent A（程式碼分析）
     │
     └──sessions_spawn──▶ 子 Agent B（文件撰寫）
```

子 Agent 繼承父 Agent 的工具策略，但可以有獨立的沙箱設定。Context Engine 也為此提供了生命週期管理（`source-repo/src/context-engine/types.ts:265-275`）。

**ACP 整合**

Agent Communication Protocol（ACP）讓 OpenClaw 能與外部 Agent 系統（如 Claude Code、Codex）互動，透過 `runtime: "acp"` 模式生成外部 Agent 會話。

---

## 7. Agent 的人格與一致性

### 7.1 人格是什麼？

在 Agent 的語境中，「人格」（Personality）指的是 Agent 的一致行為模式——它的語氣、偏好、專業知識、回應風格。人格的建立主要透過以下機制：

### 7.2 System Prompt 作為人格基礎

OpenClaw 的 System Prompt 建構函數 `buildAgentSystemPrompt()`（`source-repo/src/agents/system-prompt.ts:380-429`）接收大量參數來定義 Agent 的行為：

```typescript
export function buildAgentSystemPrompt(params: {
  workspaceDir: string;
  extraSystemPrompt?: string;      // 操作者自訂的額外提示
  ownerNumbers?: string[];         // 擁有者識別
  toolNames?: string[];            // 可用工具
  contextFiles?: EmbeddedContextFile[];  // 上下文檔案
  skillsPrompt?: string;           // 技能提示
  sandboxInfo?: EmbeddedSandboxInfo;    // 沙箱資訊
  runtimeInfo?: { agentId?; host?; os?; channel?; ... };  // 運行環境
  // ...
});
```

### 7.3 Context Files 作為人格擴展

OpenClaw 支援多種 Context File，按照固定順序注入 System Prompt（`source-repo/src/agents/system-prompt.ts:39-47`）：

```typescript
const CONTEXT_FILE_ORDER = new Map<string, number>([
  ["agents.md", 10],    // Agent 定義
  ["soul.md", 20],      // 人格靈魂
  ["identity.md", 30],  // 身份設定
  ["user.md", 40],      // 使用者偏好
  ["tools.md", 50],     // 工具使用指南
  ["bootstrap.md", 60], // 啟動上下文
  ["memory.md", 70],    // 記憶指南
]);
```

注意 `soul.md` 和 `identity.md` 的存在——這些檔案讓操作者可以定義 Agent 的「靈魂」和「身份」，使 Agent 能在不同對話中保持一致的人格特質。

### 7.4 記憶作為人格延續

人格一致性不只依賴 System Prompt——它還需要記憶的支撐。如果 Agent 忘記了上次對話中建立的關係，它的「人格」就只是表面的角色扮演。

OpenClaw 的記憶系統（Context Engine + LanceDB 向量記憶）確保了 Agent 能：

- 記住使用者的偏好和習慣
- 回想過去的對話內容
- 基於歷史互動調整行為

這讓 Agent 的人格從「一次性角色」升級為「持續性身份」——它不只是在每次對話開始時扮演一個角色，而是真正「記得」你。

---

## 8. OpenClaw 如何體現這些概念：架構對照

讓我們將理論框架與 OpenClaw 的具體實現做一個完整的對照：

```
┌─────────────────────────────────────────────────────────────────┐
│               OpenClaw Agent 認知架構對照                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐  理論概念              OpenClaw 實現         │
│  │     感知         │  ──────────────────────────────────────    │
│  │   Perception     │  多通道輸入             25+ Channel 適配器  │
│  │                  │  媒體理解               Media Understanding │
│  │                  │  系統事件               Hooks + Cron        │
│  └────────┬────────┘                                            │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │     推理         │  LLM 推理               Provider 抽象層     │
│  │   Reasoning      │  擴展思考               Extended Thinking   │
│  │                  │  ReAct 迴圈             pi-embedded-runner  │
│  │                  │  規劃能力               System Prompt 引導  │
│  └────────┬────────┘                                            │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │     行動         │  工具調用               Built-in Tools      │
│  │    Action        │  Plugin 擴展            Plugin SDK          │
│  │                  │  MCP 橋接               mcporter            │
│  │                  │  訊息發送               message 工具        │
│  └────────┬────────┘                                            │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │     記憶         │  短期記憶               Context Window      │
│  │    Memory        │  工作記憶               Session Transcript  │
│  │                  │  長期記憶               LanceDB 向量 DB     │
│  │                  │  記憶管理               Context Engine      │
│  └─────────────────┘                                            │
│                                                                  │
│  ┌─────────────────┐                                            │
│  │   多 Agent       │  路由分派               resolveAgentRoute() │
│  │  Multi-Agent     │  子 Agent 生成          sessions_spawn      │
│  │                  │  外部 Agent             ACP 協定            │
│  └─────────────────┘                                            │
│                                                                  │
│  ┌─────────────────┐                                            │
│  │     人格         │  身份定義               System Prompt       │
│  │  Personality     │  靈魂設定               soul.md / identity.md│
│  │                  │  記憶延續               Context Engine      │
│  └─────────────────┘                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. 總結：Agent 的本質

從第一性原理出發，我們可以將 AI Agent 的本質歸結為一句話：

> **AI Agent 是一個以 LLM 為推理引擎、以工具為行動介面、以記憶為狀態基礎的自主迴圈系統。**

OpenClaw 完整地實現了這個概念模型：
- **感知**透過 25+ 通道和多種媒體格式
- **推理**透過可切換的 LLM Provider（50+ 模型）
- **行動**透過分層的工具系統（核心 + Plugin + MCP）
- **記憶**透過 Context Engine 和 LanceDB 向量資料庫
- **人格**透過 System Prompt 和 Context Files
- **協作**透過 Agent Routing 和子 Agent 機制

理解了這些概念之後，前五層研究中的每一個技術細節——從 Gateway 的 WebSocket 協定到 LanceDB 的向量索引——都可以在這個框架中找到它的位置。

---

## 引用來源

| 來源 | 路徑 | 引用內容 |
|------|------|---------|
| VISION.md | `source-repo/VISION.md:3` | "OpenClaw is the AI that actually does things." |
| VISION.md | `source-repo/VISION.md:11-13` | 演化歷史與核心理念 |
| VISION.md | `source-repo/VISION.md:74-80` | MCP 透過 mcporter 整合 |
| SECURITY.md | `source-repo/SECURITY.md:120-126` | Plugin 信任模型 |
| RunEmbeddedPiAgentParams | `source-repo/src/agents/pi-embedded-runner/run/params.ts:22-80` | Agent 執行參數型別 |
| runEmbeddedPiAgent | `source-repo/src/agents/pi-embedded-runner/run.ts:166-250` | Agent 主執行函數 |
| attempt.ts | `source-repo/src/agents/pi-embedded-runner/run/attempt.ts:1-200` | 單次迴圈嘗試邏輯 |
| ContextEngine | `source-repo/src/context-engine/types.ts:150-281` | 記憶系統插拔介面 |
| buildAgentSystemPrompt | `source-repo/src/agents/system-prompt.ts:380-466` | 系統提示詞建構 |
| CONTEXT_FILE_ORDER | `source-repo/src/agents/system-prompt.ts:39-47` | 人格 Context Files 順序 |
| resolveAgentRoute | `source-repo/src/routing/resolve-route.ts` | Agent 路由解析 |
| session-key.ts | `source-repo/src/routing/session-key.ts` | Session Key 體系 |
