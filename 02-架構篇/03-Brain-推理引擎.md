# Brain——OpenClaw 的推理引擎

> **文件版本**：v1.0
> **適用範圍**：OpenClaw Brain 模組設計
> **讀者對象**：AI 工程師、後端開發者、提示工程師、對 LLM 整合有興趣的技術人員

---

## 目錄

1. [引言](#引言)
2. [LLM 作為「大腦」的概念](#llm-作為大腦的概念)
3. [提示組裝流程](#提示組裝流程)
4. [上下文窗口管理](#上下文窗口管理)
5. [ReAct 代理迴圈的運作](#react-代理迴圈的運作)
6. [模型提供者的抽象層](#模型提供者的抽象層)
7. [支援的模型](#支援的模型)
8. [模型選擇策略與備援](#模型選擇策略與備援)
9. [會話管理](#會話管理)
10. [串流回應的實作](#串流回應的實作)
11. [溫度、最大 Token 等參數調校](#溫度最大-token-等參數調校)
12. [總結](#總結)

---

## 引言

在 OpenClaw 的五大核心模組中，Brain 是最核心也最複雜的一個。如果 Gateway 是系統的「神經中樞」負責訊號傳遞，那麼 Brain 就是真正的「大腦」負責思考和決策。Brain 模組封裝了與大型語言模型（Large Language Model，LLM）的所有互動邏輯，從提示詞的組裝到回應的解析，從上下文管理到工具調用，都在 Brain 的職責範圍內。

Brain 模組的設計面臨一個獨特的挑戰：它需要在保持高度抽象性（支援多個不同的 LLM 提供者）的同時，提供精細的控制能力（允許使用者調校推理參數、自定義提示詞模板、配置備援策略）。這種平衡反映在 Brain 的分層架構中：頂層提供統一的推理介面，中間層處理提示詞組裝和上下文管理，底層則是可替換的模型提供者適配器。

本文件將從概念層面到實作細節，全面介紹 Brain 模組的設計。無論你是想要理解 LLM 在 Agent 架構中的角色，還是想要優化自己的 OpenClaw 配置，本文件都將提供有價值的參考。

值得強調的是，Brain 模組的設計哲學是「LLM 不可靠，但可以被有效利用」。LLM 可能會產生幻覺、回答不一致、或誤解指令。Brain 模組的工作就是透過精心設計的提示詞、嚴格的回應解析、和穩健的錯誤處理，將 LLM 的原始能力轉化為可靠的 Agent 行為。

---

## LLM 作為「大腦」的概念

### 從工具到器官

傳統的軟體架構將外部 API 視為「工具」——調用它、獲取結果、處理結果。但在 AI Agent 架構中，LLM 的角色更接近人體的「器官」。它不只是被調用，而是持續地參與系統的決策過程。就像人的大腦不是一個外部顧問，而是思考和行動的核心一樣，LLM 在 OpenClaw 中扮演的是「大腦」的角色。

這個類比帶來了幾個重要的設計啟示：

**持續性（Continuity）**：人的大腦不會在每次思考後「重啟」。同樣，Brain 模組維護著對話的上下文和記憶，確保 LLM 的每次推理都建立在之前的基礎上。

**整合性（Integration）**：人的大腦整合了來自五感的資訊。同樣，Brain 模組整合了使用者輸入、記憶上下文、技能結果等多種資訊源，形成完整的推理輸入。

**決策性（Decision-making）**：人的大腦不只是回答問題，還要決定行動。同樣，Brain 模組不只生成文字回應，還要決定是否需要調用工具、查詢記憶、或請求更多資訊。

**可塑性（Plasticity）**：人的大腦可以學習和適應。雖然 LLM 本身不會在推理時更新權重，但 Brain 模組透過記憶系統實現了一種「學習」——將過去的經驗轉化為未來推理的上下文。

### Brain 的職責邊界

明確 Brain 的職責邊界對於理解系統架構至關重要：

**Brain 負責的事情：**
- 組裝提示詞（將系統提示、記憶、歷史、輸入組合起來）
- 管理上下文窗口（確保不超過模型限制）
- 執行 ReAct 代理迴圈（思考→行動→觀察→思考→...）
- 解析 LLM 回應（提取文字回應和工具調用）
- 管理與 LLM 提供者的通訊
- 處理串流回應
- 實施備援策略

**Brain 不負責的事情：**
- 訊息路由（Gateway 的職責）
- 記憶儲存和檢索的底層實作（Memory 的職責）
- 技能腳本的實際執行（Skills 的職責）
- 定時任務的排程（Heartbeat 的職責）
- 頻道特定的訊息格式轉換（適配器的職責）

### 架構分層

Brain 模組內部採用分層架構：

```
┌─────────────────────────────────────────────────┐
│                   Brain Module                    │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │         Public Interface Layer                │ │
│  │  brain.process(input) → response             │ │
│  │  brain.stream(input) → stream<token>         │ │
│  └─────────────────────┬───────────────────────┘ │
│                        │                          │
│  ┌─────────────────────▼───────────────────────┐ │
│  │         Orchestration Layer                   │ │
│  │  Prompt Assembly │ ReAct Loop │ Session Mgr  │ │
│  └─────────────────────┬───────────────────────┘ │
│                        │                          │
│  ┌─────────────────────▼───────────────────────┐ │
│  │         Context Management Layer              │ │
│  │  Token Counting │ Window Mgmt │ Memory Query │ │
│  └─────────────────────┬───────────────────────┘ │
│                        │                          │
│  ┌─────────────────────▼───────────────────────┐ │
│  │         Provider Abstraction Layer            │ │
│  │  Anthropic │ OpenAI │ Google │ Ollama        │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

---

## 提示組裝流程

### 提示詞的結構

Brain 模組發送給 LLM 的完整提示詞由多個組件組合而成。每個組件都有特定的用途和優先級：

```
完整提示詞結構：

┌──────────────────────────────────────────┐
│ 1. 系統提示（System Prompt）              │  ← 最高優先級
│    定義 Agent 的身份、行為準則、能力範圍    │     永遠存在
├──────────────────────────────────────────┤
│ 2. 人格設定（Personality）                │  ← 高優先級
│    定義語氣、偏好、互動風格                │     幾乎永遠存在
├──────────────────────────────────────────┤
│ 3. 長期記憶摘要（Long-term Memory）       │  ← 中高優先級
│    來自 MEMORY.md 的核心記憶               │     根據空間裁剪
├──────────────────────────────────────────┤
│ 4. 今日筆記（Today's Notes）              │  ← 中優先級
│    來自當日 daily note 的相關內容          │     根據空間裁剪
├──────────────────────────────────────────┤
│ 5. 語意搜尋結果（Semantic Search Results）│  ← 中優先級
│    與當前輸入語意相關的記憶片段             │     根據相關度排序
├──────────────────────────────────────────┤
│ 6. 可用技能列表（Available Skills）       │  ← 中低優先級
│    當前啟用的技能及其用法說明               │     根據空間裁剪
├──────────────────────────────────────────┤
│ 7. 對話歷史（Conversation History）       │  ← 低優先級
│    當前會話的對話記錄                      │     最早的訊息先被裁剪
├──────────────────────────────────────────┤
│ 8. 使用者輸入（User Input）               │  ← 最高優先級
│    使用者當前的訊息                        │     永遠存在
└──────────────────────────────────────────┘
```

### 組裝流程

提示詞的組裝是一個有序的過程，需要在有限的上下文窗口內盡可能地包含最相關的資訊：

```
組裝流程：

步驟 1：計算固定組件的 Token 數
  ├── 系統提示：~500 tokens
  ├── 人格設定：~300 tokens
  ├── 使用者輸入：variable
  └── 保留回應空間：~2000 tokens
  
步驟 2：計算可用空間
  └── 可用空間 = 模型上下文窗口 - 固定組件 - 回應保留

步驟 3：按優先級填充可變組件
  ├── 1. 長期記憶摘要（分配 20% 可用空間）
  ├── 2. 今日筆記（分配 15% 可用空間）
  ├── 3. 語意搜尋結果（分配 15% 可用空間）
  ├── 4. 可用技能列表（分配 10% 可用空間）
  └── 5. 對話歷史（分配剩餘空間）

步驟 4：裁剪超出預算的組件
  └── 從最低優先級開始裁剪

步驟 5：組裝最終提示詞
  └── 按照定義的順序拼接所有組件
```

### 系統提示範例

系統提示（System Prompt）定義了 Agent 的核心行為：

```markdown
你是 OpenClaw，一個運行在使用者本機的個人 AI 助理。

## 核心原則
1. 你以使用者的利益為最高優先
2. 你誠實透明，不知道的事情會直接說不知道
3. 你尊重使用者的隱私，不會主動收集或分享敏感資訊
4. 你的所有操作都是可審計的

## 能力範圍
- 你可以進行自然語言對話
- 你可以調用已啟用的技能來執行操作
- 你可以查詢和更新記憶
- 你可以根據排程執行自動任務

## 行為準則
- 回答問題時，先思考需要什麼資訊，再決定是否需要調用工具
- 如果需要調用工具，使用提供的工具定義格式
- 如果不確定使用者的意圖，主動詢問
- 保持回應簡潔但完整

## 工具調用格式
當你需要調用工具時，使用以下格式：
<tool_use>
<name>工具名稱</name>
<parameters>
{"param1": "value1", "param2": "value2"}
</parameters>
</tool_use>

## 當前時間
{current_datetime}

## 當前使用者
{user_name}
```

### 記憶注入

提示詞組裝的一個關鍵步驟是記憶注入——將相關的記憶片段嵌入提示詞中：

```markdown
## 你的長期記憶
以下是你關於使用者的重要記憶：

- 使用者名稱：Eason
- 職業：軟體工程師，專注於分散式系統
- 偏好的程式語言：Go、TypeScript
- 工作時間：通常 09:00-18:00
- 使用者目前正在進行的專案：OpenClaw 研究

## 今日相關筆記
- 10:00 使用者討論了 Gateway 的路由機制設計
- 14:00 使用者詢問了 WebSocket 的心跳配置
- 今天的情緒：積極、專注

## 與當前問題相關的記憶
- [2025-01-14] 使用者曾經詢問過 LLM 串流回應的最佳實踐
- [2025-01-10] 使用者表達過對 Ollama 本地模型的興趣
```

### 提示詞模板系統

OpenClaw 支援自定義提示詞模板，允許使用者根據自己的需求調整提示詞的結構和內容：

```yaml
# ~/.openclaw/config/prompts/system.yaml

template: |
  你是 {{agent_name}}，{{agent_description}}。
  
  ## 核心原則
  {{core_principles}}
  
  ## 可用技能
  {{#each available_skills}}
  - **{{this.name}}**：{{this.description}}
    用法：{{this.usage}}
  {{/each}}
  
  ## 當前上下文
  - 時間：{{current_datetime}}
  - 使用者：{{user_name}}
  - 頻道：{{channel_name}}

variables:
  agent_name: "OpenClaw"
  agent_description: "一個運行在使用者本機的個人 AI 助理"
  core_principles: |
    1. 以使用者利益為最高優先
    2. 誠實透明
    3. 尊重隱私
```

---

## 上下文窗口管理

### 上下文窗口的概念

上下文窗口（Context Window）是 LLM 在單次推理中能夠處理的最大 token 數量。不同的模型有不同的上下文窗口大小：

| 模型 | 上下文窗口 | 建議的輸入上限 |
|------|----------|-------------|
| Claude 3.5 Sonnet | 200K tokens | ~150K tokens |
| Claude 3 Haiku | 200K tokens | ~150K tokens |
| GPT-4o | 128K tokens | ~96K tokens |
| GPT-4o-mini | 128K tokens | ~96K tokens |
| Gemini 1.5 Pro | 2M tokens | ~1.5M tokens |
| Ollama (Llama 3) | 8K tokens | ~6K tokens |
| Ollama (Mistral) | 32K tokens | ~24K tokens |

「建議的輸入上限」是考慮了保留回應空間後的值。通常建議將輸入控制在上下文窗口的 75% 以內，留出 25% 用於模型生成回應。

### Token 預算分配

Brain 模組使用「Token 預算」機制來管理上下文窗口。每個提示詞組件都有一個 token 預算，組裝過程中會嚴格遵守預算限制：

```yaml
# ~/.openclaw/config/brain.yaml

context_management:
  # 回應保留的 token 數
  response_reserve: 4096
  
  # 各組件的 token 預算（佔可用空間的百分比）
  budget:
    system_prompt: 10%      # 系統提示
    personality: 5%          # 人格設定
    long_term_memory: 15%   # 長期記憶
    daily_notes: 10%        # 今日筆記
    semantic_search: 15%    # 語意搜尋結果
    available_skills: 10%   # 可用技能列表
    conversation_history: 35%  # 對話歷史
```

### 溢出處理策略

當所有組件的 token 總和超過上下文窗口時，Brain 使用以下策略進行裁剪：

**策略 1：優先級裁剪（Priority Trimming）**

從最低優先級的組件開始裁剪。通常對話歷史是第一個被裁剪的對象——最早的訊息首先被移除。

```
裁剪順序（從先到後）：
1. 對話歷史中最早的訊息
2. 語意搜尋結果中相關度最低的片段
3. 今日筆記中最不相關的條目
4. 可用技能列表中使用頻率最低的技能
5. 長期記憶中最不相關的段落
   ↑
   系統提示和使用者輸入永遠不被裁剪
```

**策略 2：摘要壓縮（Summary Compression）**

當對話歷史太長時，Brain 可以將較早的對話摘要化，而非直接移除：

```
原始對話歷史：
[msg1] User: 今天天氣如何？
[msg2] AI: 今天台北天氣晴朗，氣溫 25-30°C...
[msg3] User: 那明天呢？
[msg4] AI: 明天預計有雨...
[msg5] User: 幫我查看行程
[msg6] AI: 您今天有以下行程...
[msg7] User: 第三個會議改到下午
[msg8] AI: 已將「產品會議」從上午改到下午...

壓縮後：
[summary] 之前的對話摘要：使用者查詢了今明兩天的天氣（今天晴朗25-30°C，明天有雨），
         然後查看了今日行程並將「產品會議」改到下午。
[msg7] User: 第三個會議改到下午
[msg8] AI: 已將「產品會議」從上午改到下午...
```

**策略 3：滑動窗口（Sliding Window）**

對於簡單的場景，Brain 可以使用滑動窗口策略——只保留最近 N 輪對話：

```yaml
context_management:
  sliding_window:
    enabled: true
    max_turns: 20  # 保留最近 20 輪對話
```

### Token 計數

精確的 token 計數對於上下文管理至關重要。不同的模型使用不同的 tokenizer，Brain 模組為每個支援的模型提供了對應的 token 計數器：

```
Token 計數示意：

文字："今天天氣如何？"

Claude tokenizer: ["今天", "天氣", "如何", "？"]    → 4 tokens
GPT tokenizer:    ["今天", "天", "氣", "如何", "？"] → 5 tokens
Llama tokenizer:  ["今", "天", "天氣", "如何", "？"] → 5 tokens
```

由於不同 tokenizer 的差異，Brain 模組會根據當前配置的模型選擇正確的 tokenizer 來計算 token 數。對於精確度要求不高的場景（如估算可用空間），也可以使用近似計算（例如中文約 1-2 tokens/字）。

---

## ReAct 代理迴圈的運作

### ReAct 模式簡介

ReAct（Reasoning + Acting）是一種讓 LLM 能夠交替進行推理和行動的代理模式。這是 OpenClaw 實現「Agent」能力的核心機制——讓 LLM 不僅能回答問題，還能執行操作。

ReAct 的核心思想是：

1. **推理（Reasoning）**：LLM 分析當前狀態，決定下一步行動
2. **行動（Acting）**：執行 LLM 決定的操作（調用工具、查詢資料等）
3. **觀察（Observation）**：獲取行動的結果
4. **重複**：基於觀察結果，進行新一輪的推理

```
ReAct 迴圈：

      ┌──────────────┐
      │   推理         │
      │  (Reasoning)  │
      └──────┬───────┘
             │
     需要工具？ │
      ├── 否 ──┤── 是 ──┐
      │        │         │
      ▼        │         ▼
  ┌──────┐    │    ┌──────────┐
  │ 回應  │    │    │   行動    │
  │(Done) │    │    │ (Acting) │
  └──────┘    │    └─────┬────┘
              │          │
              │          ▼
              │    ┌──────────┐
              │    │   觀察    │
              │    │(Observe) │
              │    └─────┬────┘
              │          │
              └──────────┘
                回到推理
```

### 實際執行流程

讓我們透過一個具體的範例來理解 ReAct 迴圈的實際運作：

**使用者輸入**：「幫我查看明天台北的天氣，然後加到我的行事曆提醒」

**第一輪迴圈——推理+行動（查天氣）：**

```
Brain 發送給 LLM 的提示詞包含：
- 系統提示（含可用工具列表）
- 記憶上下文
- 使用者輸入

LLM 回應：
"使用者想要查看明天台北的天氣，並加到行事曆。
我需要先查詢天氣資訊，然後再建立行事曆提醒。
讓我先調用天氣查詢工具。"

<tool_use>
<name>weather.get_forecast</name>
<parameters>
{"location": "台北", "date": "2025-01-17"}
</parameters>
</tool_use>
```

**Brain 解析到工具調用** → 將請求轉發給 Skills 模組 → Skills 執行天氣查詢腳本 → 獲取結果

**第二輪迴圈——觀察+推理+行動（加行事曆）：**

```
Brain 將工具結果加入上下文，再次呼叫 LLM：

上下文現在包含：
- 之前的所有內容
- 工具調用結果：{"temp": "18-24°C", "condition": "多雲", "rain_chance": "30%"}

LLM 回應：
"天氣查詢結果顯示明天台北多雲，18-24°C，降雨機率 30%。
現在我需要將這個資訊加到行事曆提醒中。"

<tool_use>
<name>calendar.create_reminder</name>
<parameters>
{
  "title": "明天天氣提醒：台北多雲 18-24°C",
  "date": "2025-01-17",
  "time": "07:00",
  "note": "降雨機率 30%，建議攜帶雨傘"
}
</parameters>
</tool_use>
```

**Brain 再次解析工具調用** → 轉發給 Skills → 執行行事曆操作 → 獲取結果

**第三輪迴圈——觀察+最終回應：**

```
Brain 將第二次工具結果加入上下文，再次呼叫 LLM：

上下文包含：
- 所有之前的內容
- 行事曆建立結果：{"status": "created", "reminder_id": "rem_789"}

LLM 回應：
"明天台北的天氣預報是多雲，氣溫 18-24°C，降雨機率約 30%。
建議攜帶雨傘以防萬一。

我已經幫你在明天早上 7 點設定了天氣提醒，
內容包含了天氣資訊和建議。"
```

**沒有更多工具調用** → Brain 返回最終回應給使用者

### 迴圈控制

為了防止 ReAct 迴圈無限運行（例如 LLM 不斷調用工具而不生成最終回應），Brain 實施了多重控制機制：

**最大迭代次數**

```yaml
react:
  max_iterations: 10  # 最多 10 輪迴圈
```

如果達到最大迭代次數，Brain 會在最後一次呼叫 LLM 時，在提示詞中加入強制完成指令：

```
[系統通知] 你已經執行了多次工具調用。請基於目前已獲得的資訊，
直接生成最終回應給使用者。不要再調用任何工具。
```

**超時控制**

```yaml
react:
  total_timeout: 120000  # 整個 ReAct 迴圈最多 120 秒
  per_step_timeout: 30000  # 每個步驟最多 30 秒
```

**Token 預算控制**

隨著每輪迴圈，上下文中的工具調用和結果會累積 token。Brain 會監控累積的 token 數，當接近上下文窗口限制時，壓縮或裁剪較早的工具交互記錄。

**工具調用限制**

```yaml
react:
  max_tool_calls_per_step: 3  # 每輪迴圈最多調用 3 個工具
  max_total_tool_calls: 20     # 整個迴圈最多調用 20 次工具
```

### 錯誤恢復

ReAct 迴圈中的錯誤處理尤為重要，因為工具調用可能失敗：

```
工具調用失敗時的處理：

1. 將錯誤資訊作為觀察結果返回給 LLM
2. LLM 會看到類似以下的內容：

   工具調用結果：
   錯誤：weather.get_forecast 調用失敗
   原因：外部 API 暫時不可用
   建議：可以嘗試其他方式或告知使用者

3. LLM 基於錯誤資訊決定下一步：
   - 嘗試替代工具
   - 使用已有的資訊生成部分回應
   - 告知使用者工具暫時不可用
```

---

## 模型提供者的抽象層

### 抽象層的設計

Brain 模組的一個核心設計是「模型提供者抽象層」（Provider Abstraction Layer）。這個抽象層為不同的 LLM 服務提供了統一的介面，使得上層的推理邏輯不需要關心底層使用的是哪個模型。

```
┌─────────────────────────────────────────────────────┐
│            Orchestration Layer                        │
│     (Prompt Assembly, ReAct Loop, Session Mgr)       │
│                                                       │
│     使用統一的 Provider Interface：                    │
│     provider.complete(prompt, options) → response     │
│     provider.stream(prompt, options) → stream         │
└─────────────────────────┬───────────────────────────┘
                          │
                ┌─────────▼──────────┐
                │  Provider Interface │
                │                    │
                │  complete()        │
                │  stream()          │
                │  count_tokens()    │
                │  get_model_info()  │
                └─────────┬──────────┘
                          │
        ┌─────────┬───────┼───────┬──────────┐
        │         │       │       │          │
   ┌────▼────┐ ┌─▼──┐ ┌──▼───┐ ┌▼──────┐ ┌─▼───────┐
   │Anthropic│ │Open│ │Google│ │Ollama │ │Custom  │
   │Provider │ │AI  │ │      │ │       │ │Provider│
   │         │ │Prov│ │Prov  │ │Prov   │ │        │
   └─────────┘ └────┘ └──────┘ └───────┘ └────────┘
```

### Provider Interface

每個模型提供者都必須實作以下介面：

```typescript
interface LLMProvider {
  // 基本資訊
  name: string;
  models: ModelInfo[];
  
  // 核心方法
  complete(request: CompletionRequest): Promise<CompletionResponse>;
  stream(request: CompletionRequest): AsyncIterable<StreamChunk>;
  
  // 輔助方法
  countTokens(text: string, model: string): number;
  getModelInfo(model: string): ModelInfo;
  
  // 健康檢查
  healthCheck(): Promise<HealthStatus>;
}

interface CompletionRequest {
  model: string;
  messages: Message[];
  temperature?: number;
  max_tokens?: number;
  top_p?: number;
  stop_sequences?: string[];
  tools?: ToolDefinition[];
}

interface CompletionResponse {
  content: string;
  tool_calls?: ToolCall[];
  usage: {
    input_tokens: number;
    output_tokens: number;
    total_tokens: number;
  };
  finish_reason: "stop" | "tool_use" | "length" | "error";
  model: string;
  latency_ms: number;
}

interface StreamChunk {
  type: "text" | "tool_use_start" | "tool_use_end" | "done";
  content?: string;
  tool_call?: Partial<ToolCall>;
}
```

### 提供者間的差異處理

不同的 LLM 提供者在 API 設計上存在差異，抽象層負責處理這些差異：

**訊息格式差異**

```
Anthropic (Claude):
{
  "role": "user",
  "content": [
    {"type": "text", "text": "..."}
  ]
}

OpenAI (GPT):
{
  "role": "user",
  "content": "..."
}

Google (Gemini):
{
  "role": "user",
  "parts": [
    {"text": "..."}
  ]
}
```

抽象層將 OpenClaw 的內部訊息格式轉換為每個提供者要求的格式。

**工具調用格式差異**

不同提供者的工具調用格式也不同。抽象層負責在 OpenClaw 的標準工具格式和各提供者的專有格式之間進行轉換。

**串流格式差異**

串流回應的事件格式在不同提供者之間也有差異。抽象層將所有格式統一為 OpenClaw 的 `StreamChunk` 格式。

---

## 支援的模型

### Anthropic Claude

Anthropic 的 Claude 系列是 OpenClaw 的預設推薦模型，提供了優秀的推理能力和工具使用能力：

```yaml
# Claude 配置
provider:
  type: "anthropic"
  api_key: "${ANTHROPIC_API_KEY}"
  models:
    default: "claude-sonnet-4-20250514"
    available:
      - name: "claude-sonnet-4-20250514"
        context_window: 200000
        max_output: 8192
        cost_per_1k_input: 0.003
        cost_per_1k_output: 0.015
        strengths: ["推理", "工具使用", "長上下文"]
      - name: "claude-3-5-haiku-20241022"
        context_window: 200000
        max_output: 8192
        cost_per_1k_input: 0.001
        cost_per_1k_output: 0.005
        strengths: ["速度", "成本效益"]
```

**Claude 的優勢：**
- 出色的指令遵循能力，適合結構化的工具調用
- 200K token 的大上下文窗口，可以容納大量記憶
- 支援原生的工具使用（Tool Use）API
- 回應風格自然，適合作為個人助理

### OpenAI GPT

OpenAI 的 GPT 系列提供了成熟穩定的 API 和廣泛的生態系統：

```yaml
# GPT 配置
provider:
  type: "openai"
  api_key: "${OPENAI_API_KEY}"
  models:
    default: "gpt-4o"
    available:
      - name: "gpt-4o"
        context_window: 128000
        max_output: 16384
        cost_per_1k_input: 0.005
        cost_per_1k_output: 0.015
        strengths: ["多模態", "程式碼", "通用性"]
      - name: "gpt-4o-mini"
        context_window: 128000
        max_output: 16384
        cost_per_1k_input: 0.00015
        cost_per_1k_output: 0.0006
        strengths: ["成本效益", "速度"]
```

**GPT 的優勢：**
- 成熟穩定的 API，文件和社群支援豐富
- 支援 Function Calling，與工具調用整合良好
- GPT-4o-mini 提供極佳的性價比
- 多模態支援（可以處理圖片輸入）

### Google Gemini

Google 的 Gemini 系列以超大的上下文窗口著稱：

```yaml
# Gemini 配置
provider:
  type: "google"
  api_key: "${GOOGLE_API_KEY}"
  models:
    default: "gemini-1.5-pro"
    available:
      - name: "gemini-1.5-pro"
        context_window: 2000000
        max_output: 8192
        cost_per_1k_input: 0.00125
        cost_per_1k_output: 0.005
        strengths: ["超長上下文", "多模態", "成本"]
      - name: "gemini-1.5-flash"
        context_window: 1000000
        max_output: 8192
        cost_per_1k_input: 0.000075
        cost_per_1k_output: 0.0003
        strengths: ["速度", "成本效益", "長上下文"]
```

**Gemini 的優勢：**
- 2M token 的超大上下文窗口，可以一次性處理大量資訊
- 極具競爭力的定價
- Gemini Flash 提供超快的回應速度
- 原生多模態支援

### 本地 Ollama

Ollama 允許在本地運行開源模型，完全不需要網路連線：

```yaml
# Ollama 配置
provider:
  type: "ollama"
  base_url: "http://localhost:11434"
  models:
    default: "llama3.1:8b"
    available:
      - name: "llama3.1:8b"
        context_window: 8192
        max_output: 2048
        cost: 0  # 本地運行，無費用
        strengths: ["隱私", "離線", "無費用"]
        requirements: "8GB RAM"
      - name: "mistral:7b"
        context_window: 32768
        max_output: 4096
        cost: 0
        strengths: ["隱私", "離線", "較大上下文"]
        requirements: "8GB RAM"
      - name: "qwen2.5:14b"
        context_window: 32768
        max_output: 4096
        cost: 0
        strengths: ["中文能力", "隱私", "離線"]
        requirements: "16GB RAM"
```

**Ollama 的優勢：**
- 完全本地運行，零資料外洩風險
- 無使用費用（只有硬體成本）
- 離線可用
- 開源模型，完全透明

**Ollama 的限制：**
- 需要足夠的 GPU/RAM 資源
- 推理速度通常慢於雲端服務
- 上下文窗口較小
- 模型能力通常不如頂級雲端模型

---

## 模型選擇策略與備援

### 選擇策略

Brain 模組支援多種模型選擇策略，使用者可以根據需求配置：

**策略 1：固定模型（Fixed Model）**

始終使用同一個模型，最簡單的配置：

```yaml
model_selection:
  strategy: "fixed"
  model: "claude-sonnet-4-20250514"
```

**策略 2：任務路由（Task-based Routing）**

根據任務類型選擇不同的模型：

```yaml
model_selection:
  strategy: "task_based"
  rules:
    - task: "simple_chat"
      model: "claude-3-5-haiku-20241022"  # 簡單聊天用較便宜的模型
    - task: "complex_reasoning"
      model: "claude-sonnet-4-20250514"        # 複雜推理用強模型
    - task: "code_generation"
      model: "gpt-4o"                    # 程式碼生成用 GPT
    - task: "long_context"
      model: "gemini-1.5-pro"            # 長上下文用 Gemini
  default: "claude-3-5-haiku-20241022"
```

**策略 3：成本優化（Cost Optimization）**

優先使用便宜的模型，只在需要時升級：

```yaml
model_selection:
  strategy: "cost_optimized"
  primary: "gpt-4o-mini"           # 預設使用便宜模型
  escalation_trigger:
    - "tool_use_required"          # 需要工具調用時升級
    - "complex_query_detected"     # 偵測到複雜查詢時升級
  escalation_model: "gpt-4o"
```

### 備援機制（Fallback）

當主要模型不可用時（API 錯誤、速率限制、網路問題），Brain 會自動切換到備援模型：

```yaml
fallback:
  enabled: true
  chain:
    - provider: "anthropic"
      model: "claude-sonnet-4-20250514"
    - provider: "openai"
      model: "gpt-4o"
    - provider: "google"
      model: "gemini-1.5-pro"
    - provider: "ollama"
      model: "llama3.1:8b"     # 最後的備援：本地模型
  
  # 備援觸發條件
  triggers:
    - error_code: 429          # Rate limited
    - error_code: 500          # Server error
    - error_code: 503          # Service unavailable
    - timeout: 30000           # 30 秒超時
```

備援流程：

```
嘗試 Claude ──失敗──▶ 嘗試 GPT ──失敗──▶ 嘗試 Gemini ──失敗──▶ 嘗試 Ollama
    │                   │                    │                     │
    │成功                │成功                │成功                  │成功
    ▼                   ▼                    ▼                     ▼
  回應                 回應                  回應                   回應
                                                                   │
                                                              ──失敗──▶ 返回降級回應
                                                                       "目前無法處理"
```

### 模型健康監控

Brain 模組持續監控各模型提供者的健康狀態：

```
┌──────────────────────────────────────────────┐
│           Model Health Dashboard              │
├──────────┬──────────┬───────┬───────┬────────┤
│ Provider │ Model    │Status │Latency│Error % │
├──────────┼──────────┼───────┼───────┼────────┤
│ Anthropic│ Claude   │  ✅   │ 1.2s  │  0.1%  │
│ OpenAI   │ GPT-4o   │  ✅   │ 0.8s  │  0.3%  │
│ Google   │ Gemini   │  ⚠️   │ 3.5s  │  2.1%  │
│ Ollama   │ Llama3   │  ✅   │ 5.0s  │  0.0%  │
└──────────┴──────────┴───────┴───────┴────────┘
```

---

## 會話管理

### 會話的概念

在 OpenClaw 中，「會話」（Session）是使用者與 Agent 之間的一段連續互動。每個會話有獨立的對話歷史、上下文和狀態。會話的管理對於提供連貫的對話體驗至關重要。

### 會話生命週期

```
建立 ──▶ 活躍 ──▶ 閒置 ──▶ 過期 ──▶ 歸檔
          │         │
          │         │ 使用者回來
          │         │
          │         └──▶ 活躍（恢復）
          │
          │ 使用者明確結束
          │
          └──▶ 結束 ──▶ 歸檔
```

**建立（Created）**：當使用者發送第一條訊息時，自動建立新會話。

**活躍（Active）**：會話正在進行中，使用者和 Agent 持續互動。

**閒置（Idle）**：使用者停止互動一段時間（預設 30 分鐘）。會話狀態被保留在記憶體中。

**過期（Expired）**：閒置超過配置的閾值（預設 24 小時）。會話上下文被寫入檔案並從記憶體中釋放。

**歸檔（Archived）**：會話記錄被壓縮並歸檔到記憶系統中。

### 會話狀態管理

每個會話維護以下狀態：

```json
{
  "session_id": "sess_20250116_001",
  "channel": "telegram",
  "user_id": "user_eason",
  "state": "active",
  "created_at": "2025-01-16T10:00:00Z",
  "last_activity": "2025-01-16T10:30:00Z",
  "message_count": 15,
  "total_tokens_used": 45000,
  "conversation_history": [...],
  "react_state": null,
  "metadata": {
    "topic": "技術討論",
    "mood": "積極"
  }
}
```

### 跨頻道會話

OpenClaw 支援跨頻道的會話連續性。例如，使用者在 Telegram 上開始的對話，可以在 CLI 上繼續：

```
Telegram:
User: 幫我分析這段程式碼的問題
AI: 好的，我看到幾個潛在問題...

（使用者切換到 CLI）

CLI:
User: 繼續剛才的程式碼分析
AI: 接續剛才的分析，第三個問題是...  ← 無縫繼續
```

這是透過使用者 ID（而非頻道 ID）來關聯會話實現的。只要是同一個使用者，無論從哪個頻道發送訊息，Brain 都能找到並恢復最近的活躍會話。

---

## 串流回應的實作

### 為什麼需要串流

LLM 生成回應通常需要數秒到數十秒的時間。如果等到整個回應生成完畢再一次性返回，使用者體驗會很差——使用者會看到長時間的空白，然後突然出現一大段文字。

串流回應（Streaming Response）解決了這個問題，讓使用者能夠即時看到 LLM 正在「打字」的效果：

```
非串流模式：
User: 幫我寫一首詩
[等待 8 秒...]
AI: 春風拂面桃花開，
    碧水東流燕歸來。
    山間雲霧輕如紗，
    一壺清茶品人生。

串流模式：
User: 幫我寫一首詩
AI: 春                          ← 0.5 秒
AI: 春風拂面                    ← 1.0 秒
AI: 春風拂面桃花開，            ← 1.5 秒
AI: 春風拂面桃花開，碧水        ← 2.0 秒
...（持續更新直到完成）
```

### 串流架構

```
LLM API ──stream──▶ Brain ──stream──▶ Gateway ──stream──▶ Adapter ──▶ User
                     │                  │
                     │ 解析工具調用      │ 路由
                     │ 累積完整回應      │ 格式轉換
```

### 串流與 ReAct 的協調

在 ReAct 迴圈中，串流需要特殊處理。LLM 的串流回應可能包含文字和工具調用的混合：

```
LLM 串流輸出：
[text] "讓我幫你查看明天的天氣"
[text] "。\n\n"
[tool_start] weather.get_forecast
[tool_params] {"location": "台北"}
[tool_end]

Brain 的處理：
1. 文字部分即時推送給使用者
2. 工具調用部分不推送，而是暫存
3. 串流結束後，執行工具調用
4. 獲得工具結果後，開始新一輪串流
```

### 串流的配置

```yaml
streaming:
  enabled: true
  
  # 緩衝策略
  buffer:
    # 按句子緩衝：累積到完整句子後再推送
    strategy: "sentence"
    # 也可以選擇 "token"（逐 token）或 "word"（逐詞）
    
  # 推送間隔（毫秒）
  min_push_interval: 50
  
  # 串流超時（秒）
  stream_timeout: 120
  
  # 是否在串流期間顯示思考過程
  show_thinking: false
```

---

## 溫度、最大 Token 等參數調校

### 參數總覽

LLM 的推理行為可以透過多個參數來調校。Brain 模組允許使用者在全域和會話層級設定這些參數：

| 參數 | 範圍 | 預設值 | 影響 |
|------|------|-------|------|
| `temperature` | 0.0 - 2.0 | 0.7 | 回應的隨機性/創造性 |
| `max_tokens` | 1 - 模型上限 | 4096 | 回應的最大長度 |
| `top_p` | 0.0 - 1.0 | 0.9 | 核心取樣的機率閾值 |
| `top_k` | 1 - 100 | 40 | 取樣時考慮的最高機率 token 數 |
| `frequency_penalty` | -2.0 - 2.0 | 0.0 | 對已出現 token 的懲罰 |
| `presence_penalty` | -2.0 - 2.0 | 0.0 | 對新話題的鼓勵 |
| `stop_sequences` | 字串陣列 | [] | 停止生成的觸發序列 |

### 溫度（Temperature）

溫度是最常調校的參數，它控制了 LLM 輸出的隨機性：

```
溫度 0.0（確定性）：
  - 每次都選擇機率最高的 token
  - 回應高度一致和可預測
  - 適合事實性查詢、程式碼生成、格式化輸出

溫度 0.7（平衡）：
  - 在高機率 token 中進行適度隨機選擇
  - 回應自然但不過於離散
  - 適合一般對話、解釋性回答

溫度 1.0+（高創造性）：
  - 大幅增加低機率 token 被選中的可能
  - 回應更有創造性但可能不夠準確
  - 適合創意寫作、腦力激盪
```

**OpenClaw 的建議配置：**

```yaml
# 不同場景的溫度設定
temperature_presets:
  factual: 0.0        # 事實查詢
  coding: 0.2          # 程式碼相關
  conversation: 0.7    # 一般對話（預設）
  creative: 1.0        # 創意任務
  brainstorm: 1.2      # 腦力激盪
```

### 最大 Token（Max Tokens）

最大 token 限制了 LLM 單次回應的長度：

```yaml
max_tokens_presets:
  brief: 256          # 簡短回答
  normal: 4096        # 一般回答（預設）
  detailed: 8192      # 詳細回答
  long_form: 16384    # 長篇內容生成
```

設定過小會導致回應被截斷，設定過大則浪費資源（token 計費）且可能導致冗長回應。

### 參數配置方式

使用者可以在多個層級配置這些參數：

**全域配置**（適用於所有對話）：
```yaml
# ~/.openclaw/config/brain.yaml
default_params:
  temperature: 0.7
  max_tokens: 4096
  top_p: 0.9
```

**頻道配置**（適用於特定頻道）：
```yaml
channel_params:
  telegram:
    max_tokens: 2048  # Telegram 訊息有長度限制
  cli:
    max_tokens: 8192  # CLI 可以顯示更長的內容
```

**會話內臨時調整**：
```
User: /config temperature 0.2
AI: 已將溫度設定為 0.2（低隨機性模式）

User: 幫我寫一段排序演算法
AI: [使用低溫度生成更確定性的程式碼]
```

### 參數調校建議

根據不同的使用場景，以下是 OpenClaw 的參數調校建議：

**場景 1：個人助理（日常使用）**
```yaml
temperature: 0.7
max_tokens: 4096
top_p: 0.9
# 平衡的設定，適合多數場景
```

**場景 2：技術諮詢（高精確度）**
```yaml
temperature: 0.2
max_tokens: 4096
top_p: 0.95
# 低溫度確保技術回答的準確性
```

**場景 3：創意寫作**
```yaml
temperature: 1.0
max_tokens: 8192
top_p: 0.85
frequency_penalty: 0.5  # 避免重複
# 高溫度鼓勵創造性
```

**場景 4：工具調用（ReAct 模式）**
```yaml
temperature: 0.0
max_tokens: 4096
# 零溫度確保工具調用的格式正確
```

---

## 總結

Brain 是 OpenClaw 的智慧核心，它將 LLM 的原始能力轉化為可靠、可控的 Agent 行為。本文件詳細介紹了 Brain 模組的各個設計面向：

### 核心設計要點

1. **提示詞組裝**：多組件、有優先級的提示詞結構，確保最相關的資訊被包含在有限的上下文窗口中
2. **上下文管理**：Token 預算制度、溢出裁剪策略、摘要壓縮，精細管理上下文窗口
3. **ReAct 迴圈**：推理→行動→觀察的迭代模式，讓 Agent 能夠使用工具完成複雜任務
4. **Provider 抽象**：統一的介面支援多個 LLM 提供者，一行配置即可切換模型
5. **備援機制**：多級備援鏈，確保即使主要模型不可用，系統仍能提供服務
6. **串流回應**：即時推送生成中的回應，提供良好的使用者體驗
7. **參數調校**：多層級的參數配置，從全域到會話層級的精細控制

### 延伸閱讀

- [01-系統架構總覽.md](./01-系統架構總覽.md)——回顧 Brain 在整體架構中的位置
- [04-Memory-記憶系統.md](./04-Memory-記憶系統.md)——了解 Brain 如何利用記憶上下文
- [05-Heartbeat-自主排程.md](./05-Heartbeat-自主排程.md)——了解 Brain 如何處理 Heartbeat 觸發的任務
- [06-Skills-技能系統.md](./06-Skills-技能系統.md)——了解 Brain 在 ReAct 迴圈中如何調用技能

---

> **上一篇**：[02-Gateway-中央編排器.md](./02-Gateway-中央編排器.md)
> **下一篇**：[04-Memory-記憶系統.md](./04-Memory-記憶系統.md)
