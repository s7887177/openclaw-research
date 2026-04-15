# 競品概覽：AI Agent 框架生態

> **所屬 Topic**：OpenClaw vs 替代方案（OpenClaw vs Alternatives）  
> **層級**：Level 6 — 概念綜合層  
> **前置閱讀**：Level 4 架構總覽

---

## 摘要

AI Agent 框架生態正處於爆發期，不同框架的設計定位差異巨大——從開發者工具鏈到自主 Agent，從多 Agent 協作到低程式碼工作流。本章逐一介紹 9 類主要框架的核心定位、設計哲學與目標使用者，為後續的系統性比較建立基礎。

---

## 1. 框架分類地圖

在深入每個框架之前，先建立一個宏觀的分類地圖：

```
                         AI Agent 框架生態
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   開發者工具導向          產品導向             自動化導向
        │                     │                     │
   ┌────┴────┐          ┌────┴────┐          ┌────┴────┐
   │         │          │         │          │         │
LangChain  CrewAI    OpenClaw  Botpress    n8n      Dify
LangGraph  MetaGPT   AutoGPT   Rasa       Make     Flowise
```

| 導向 | 核心問題 | 目標使用者 |
|------|---------|-----------|
| 開發者工具（Developer Toolkit） | 「如何建構 AI Agent？」 | 軟體工程師 |
| 產品（Product） | 「如何使用 AI Agent？」 | 終端使用者 / 非技術人員 |
| 自動化（Automation） | 「如何讓 AI 自動做事？」 | 業務人員 / 技術人員 |

---

## 2. 開發者工具導向

### 2.1 LangChain / LangGraph

**核心定位**：AI 應用開發的瑞士刀。

LangChain 是目前最流行的 LLM 應用開發框架，提供一套標準化的抽象（Abstractions）來串接 LLM、工具、記憶、檢索等元件。LangGraph 是其子專案，專注於有向無環圖（DAG）形式的 Agent 工作流。

**設計哲學**：「給你一盒樂高，你自己組裝。」LangChain 不是一個產品，而是一個工具箱。它假設你是一個開發者，你知道你要建什麼，它幫你省去重複造輪子的時間。

**典型使用方式**：
```python
from langchain.agents import create_react_agent
agent = create_react_agent(llm, tools, prompt)
# 你需要自己處理：部署、通道整合、記憶持久化、安全...
```

**優勢**：
- 極高的彈性——幾乎可以建構任何 LLM 應用
- 龐大的整合生態（700+ 整合）
- 活躍的社群和文件
- LangGraph 提供了有向圖（Graph）Agent 的建構方式，適合複雜的多步驟推理

**限制**：
- 不提供開箱即用的產品體驗——安裝完 LangChain，你還是什麼都沒有
- 需要大量自行開發才能做出完整的 Agent（路由、記憶持久化、部署、安全）
- 不原生支援任何通訊通道，若要連 Discord 或 Telegram 都需自己寫整合碼
- 純開發者工具，非技術人員完全無法使用
- 抽象層過多，有時反而增加複雜度（「LangChain tax」）

### 2.2 AutoGPT / AgentGPT

**核心定位**：自主任務執行 Agent。

AutoGPT 是最早的「自主 Agent」嘗試之一——給它一個高層目標，它自動分解任務、規劃步驟、使用工具執行。AgentGPT 是其瀏覽器版本，提供 Web UI。

**設計哲學**：「給我一個目標，我自己想辦法完成。」AutoGPT 追求的是完全自主的任務執行，減少人類干預。

**典型使用方式**：
```
目標：研究 2024 年最受歡迎的 JavaScript 框架，寫一份報告
→ AutoGPT 自動：搜尋網路 → 整理資料 → 撰寫報告 → 存檔
```

**優勢**：
- 令人印象深刻的自主能力展示
- 自動任務分解與規劃
- 支援 Plugin 擴展

**限制**：
- 可靠性不足——經常陷入迴圈或偏離目標，需要人類監督
- Token 消耗極高（大量的自我對話、重複的目標確認）
- 不適合持續性的「助手」場景——它是任務導向的，完成就結束
- 基本沒有通道整合（通常只有 Web UI 或 CLI）
- 安全隱憂：自主 Agent 如果失控可能造成意想不到的後果（如刪除檔案、發送不當訊息）

### 2.3 CrewAI

**核心定位**：多 Agent 角色扮演與協作框架。

CrewAI 的核心概念是「Crew」（團隊），每個 Agent 有不同的角色（Role）、目標（Goal）、背景故事（Backstory），多個 Agent 協作完成複雜任務。

**設計哲學**：「一個人不夠，組一個團隊。」CrewAI 認為複雜任務需要多個專業化的 Agent 協作，就像人類團隊一樣。

**典型使用方式**：
```python
researcher = Agent(role="Senior Researcher", goal="Research topic", ...)
writer = Agent(role="Tech Writer", goal="Write article", ...)
crew = Crew(agents=[researcher, writer], tasks=[research_task, write_task])
result = crew.kickoff()
```

**優勢**：
- 直覺的多 Agent 協作模型
- 角色分工明確，易於理解
- 任務委派（Delegation）機制

**限制**：
- 面向任務執行，不是持續性助手——每個「Crew」執行一次就結束
- 不支援通道整合——結果通常透過 Python 輸出
- 多 Agent 的 Token 開銷更大——每個 Agent 都帶自己的 System Prompt
- 主要是 Python 生態，不適合 TypeScript/Node.js 使用者
- 角色設定需要精心設計，否則 Agent 之間的協作效果不佳

### 2.4 MetaGPT

**核心定位**：模擬軟體公司的多 Agent 框架。

MetaGPT 的獨特之處在於它模擬了一個完整的軟體開發團隊——產品經理、架構師、工程師、QA——每個角色由不同的 Agent 扮演，遵循標準化的軟體開發流程。

**設計哲學**：「用 AI 模擬整個開發團隊。」MetaGPT 嘗試將人類軟體工程的最佳實踐（PRD、設計文件、程式碼審查）編碼進 Agent 的行為中。

**優勢**：
- 有結構化的開發流程（PRD → 設計 → 實作 → 測試）
- 生成的文檔品質較高
- 學術創新性強

**限制**：
- 高度特化——主要用於軟體開發
- 不是通用 AI 助手
- 無通道整合
- 研究性質較重，生產可用性有限

---

## 3. 產品導向

### 3.1 OpenClaw

**核心定位**：個人 AI 助手的本地優先多通道閘道。

OpenClaw 的獨特之處在於它同時是：
1. 一個通道閘道（Channel Gateway）——統一 25+ 通訊平台
2. 一個 Agent 執行環境——提供 ReAct 推理迴圈
3. 一個 Plugin 生態——111 extensions + 53 skills
4. 一個記憶系統——短/中/長期記憶
5. 一個安全框架——個人操作者信任模型

**設計哲學**：「你的 AI，在你的電腦上，連接你的所有通訊平台。」OpenClaw 是 Local-first 的——Agent 運行在你自己的機器上，資料不上雲。

**詳細分析見後續章節。**

### 3.2 Botpress / Rasa

**核心定位**：傳統聊天機器人開發平台。

Botpress 和 Rasa 是 LLM 時代之前的聊天機器人框架，最初設計用於意圖識別（Intent Recognition）和對話流程管理（Dialog Management）。它們後來加入了 LLM 整合，但核心架構仍然帶有「基於規則的對話流」的痕跡。

**Botpress 設計哲學**：「視覺化拖拉對話流程。」Botpress 提供圖形化的流程編輯器，適合非技術人員設計聊天機器人。

**Rasa 設計哲學**：「用 ML 理解使用者意圖。」Rasa 強調自訂 NLU（Natural Language Understanding）模型，適合需要高度客製化的企業場景。

| 特性 | Botpress | Rasa |
|------|----------|------|
| 介面 | 視覺化 Studio | YAML + Python |
| NLU | 內建（LLM-based） | 自訂模型 |
| 通道 | Web, Messenger, WhatsApp, Telegram（有限） | Web, Messenger, Slack, Telegram |
| 部署 | Cloud + Self-hosted | Self-hosted |
| LLM 整合 | 是（後來加入） | 是（後來加入） |
| 開源 | 是 | 是 |

**優勢**：
- 成熟的對話管理（Dialog Management）
- 企業級多租戶支援
- 視覺化流程設計（Botpress）
- 自訂 NLU 模型（Rasa）

**限制**：
- 架構相對過時——基於意圖/槽位（Intent/Slot）而非 LLM-native
- 通道支援有限（通常 4-6 個，遠少於 OpenClaw 的 21+）
- 不是個人助手——面向企業客服場景
- 缺少記憶系統、語音通道
- Rasa 的學習曲線陡峭，需要 ML 工程能力

值得一提的是，Botpress 在 2024-2025 年做了大幅轉型，從傳統的意圖識別轉向 LLM-native 架構。但其核心仍然是「為企業客服而建」——多租戶、分析儀表板、知識庫管理。這與 OpenClaw 的「個人助手」定位根本不同。

---

## 4. 自動化導向

### 4.1 Dify / Flowise

**核心定位**：低程式碼 AI 工作流平台。

Dify 和 Flowise 提供拖拉式（Drag-and-drop）的 AI 工作流編輯器，讓非開發者也能建構 LLM 應用。它們的核心是「節點」（Node）—— 每個節點代表一個步驟（LLM 呼叫、工具調用、條件判斷），節點之間用線連接形成工作流。

**Dify 設計哲學**：「人人都能建構 AI 應用。」Dify 定位為 LLM-Ops 平台，涵蓋從 Prompt 工程到部署監控的全流程。

**Flowise 設計哲學**：「LangChain 的視覺化版本。」Flowise 直接將 LangChain 的元件封裝為可視覺化節點。

**限制**：
- 低門檻——非工程師也能建構 AI 工作流
- 視覺化流程設計
- RAG（Retrieval-Augmented Generation）整合良好
- 可部署為 API
- Dify 的知識庫功能完善，支援多種文件格式的導入和向量化

**限制**：
- 不是 Agent——是工作流，每次執行遵循預定義路徑
- 無通道整合（通常只有 API endpoint + 內建 Web Chat）
- 無個人助手體驗——你得主動去 Web 介面跟它對話
- 缺少記憶系統（或非常基礎，限於單一對話內）
- 無語音能力
- 工作流一旦設計好，修改的彈性有限——不如 ReAct 模式靈活

**Dify vs Flowise 的差異**：Dify 定位更高端（LLMOps 平台），提供使用監控、A/B 測試、標註管理等企業功能。Flowise 更輕量，直接將 LangChain 節點視覺化，適合快速原型。但兩者都不提供通道整合——你建出來的「應用」需要嵌入到你自己的前端中。

### 4.2 n8n / Make (Integromat)

**核心定位**：通用自動化平台（加上 AI 整合）。

n8n 和 Make 原本是工作流自動化工具（類似 Zapier），後來加入了 AI 節點（LLM 呼叫、向量搜尋、Agent 節點）。它們的核心價值是**連接不同的 SaaS 服務**——「當 A 發生時，做 B」。

**設計哲學**：「連接一切，自動化一切。」

**與 AI Agent 的關係**：n8n/Make 不是 AI Agent 框架——它們是自動化平台，但可以在自動化流程中嵌入 LLM 呼叫。這讓它們成為一種「低程式碼 Agent」的可能性，但其核心是事件驅動的自動化，而非持續性的 AI 助手。

**優勢**：
- 龐大的 SaaS 整合（n8n: 400+, Make: 1500+）
- 成熟的工作流引擎（排程、重試、錯誤處理）
- 自託管能力（n8n）
- 可以用 AI 增強現有自動化
- 非 AI 場景也非常有用——可以自動化任何 SaaS 之間的資料流

**限制**：
- 不是真正的 AI Agent——沒有持續性對話、沒有記憶
- AI 只是工作流中的一個節點，不能自主決策
- 無通道整合（只能觸發/回應 webhook）
- 無語音能力
- 學習曲線（複雜工作流可以非常龐大）

**n8n 與 OpenClaw 的互補性**：值得特別注意的是，n8n 和 OpenClaw 不是競爭關係，而是天然互補。n8n 擅長「事件驅動的自動化流程」（如：新的 GitHub Issue → 分類 → 指派 → 通知），OpenClaw 擅長「對話式的 AI 互動」。兩者可以結合：n8n 觸發 OpenClaw 的工具呼叫，或 OpenClaw 的 Webhook Extension 接收 n8n 的通知。

---

## 5. 定位總結

| 框架 | 一句話定位 | 核心價值 | 目標使用者 |
|------|-----------|---------|-----------|
| **LangChain** | LLM 應用開發工具箱 | 靈活的元件組合 | 軟體工程師 |
| **AutoGPT** | 自主任務執行 Agent | 自動目標達成 | 技術愛好者 |
| **CrewAI** | 多 Agent 協作框架 | 角色分工協作 | AI 工程師 |
| **MetaGPT** | 軟體開發多 Agent | 模擬開發團隊 | AI 研究者/開發者 |
| **Botpress** | 視覺化聊天機器人 | 低門檻對話設計 | 產品/客服團隊 |
| **Rasa** | 企業級對話 AI | 自訂 NLU 模型 | ML 工程師 |
| **Dify** | 低程式碼 AI 工作流 | 人人建構 AI 應用 | 非技術人員/產品團隊 |
| **Flowise** | LangChain 視覺化 | 拖拉式 LLM 流程 | 低程式碼開發者 |
| **n8n** | AI 增強的自動化 | 連接 SaaS 服務 | DevOps/自動化工程師 |
| **Make** | AI 增強的自動化 | 連接 SaaS 服務 | 業務自動化人員 |
| **OpenClaw** | 個人 AI 助手閘道 | 多通道 + 本地優先 | 技術使用者（個人用） |

---

## 引用來源

| 來源 | 說明 |
|------|------|
| LangChain 官方文件 | https://python.langchain.com/docs/ |
| AutoGPT GitHub | https://github.com/Significant-Gravitas/AutoGPT |
| CrewAI 官方文件 | https://docs.crewai.com/ |
| MetaGPT GitHub | https://github.com/geekan/MetaGPT |
| Botpress 官方文件 | https://botpress.com/docs |
| Rasa 官方文件 | https://rasa.com/docs/ |
| Dify 官方文件 | https://docs.dify.ai/ |
| Flowise GitHub | https://github.com/FlowiseAI/Flowise |
| n8n 官方文件 | https://docs.n8n.io/ |
| Make 官方文件 | https://www.make.com/en/help |
| OpenClaw 原始碼 | `source-repo/` — OpenClaw v2026.4.12 |
| Level 4 架構總覽 | `research/versions/2026.4.12/system-behavior/architecture-overview/` |
