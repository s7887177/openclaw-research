# ReAct 模式原理——推理與行動的統一

> **摘要**：本文深入解析 ReAct（Reasoning + Acting）模式的理論基礎與實作範式。從傳統 Chain-of-Thought 的侷限出發，剖析 ReAct 論文的核心思想、Thought-Action-Observation 循環的數學形式化、與其他方法的比較、工具調用標準化、錯誤恢復策略、多步推理挑戰，以及未來發展方向。

---

## 目錄

1. [導論：AI 不只是思考](#1-導論ai-不只是思考)
2. [傳統 Chain-of-Thought 的侷限](#2-傳統-chain-of-thought-的侷限)
3. [ReAct 論文的核心思想](#3-react-論文的核心思想)
4. [Thought → Action → Observation 循環](#4-thought--action--observation-循環)
5. [接地（Grounding）——為什麼行動比只是思考重要](#5-接地grounding為什麼行動比只是思考重要)
6. [ReAct vs CoT vs Act-only 詳細比較](#6-react-vs-cot-vs-act-only-詳細比較)
7. [ReAct 的實作範式](#7-react-的實作範式)
8. [工具調用的標準化](#8-工具調用的標準化)
9. [錯誤恢復與重試策略](#9-錯誤恢復與重試策略)
10. [多步推理的挑戰](#10-多步推理的挑戰)
11. [ReAct 的局限與未來發展](#11-react-的局限與未來發展)
12. [與人類思考方式的類比](#12-與人類思考方式的類比)

---

## 1. 導論：AI 不只是思考

### 1.1 從 GPT 到 Agent

大型語言模型（LLM）的發展經歷了幾個重要階段：

```
階段 1：生成式模型（GPT-2/3 時期）
  輸入文字 → 生成文字
  純粹的文字接龍，沒有推理能力

階段 2：指令跟隨（InstructGPT、ChatGPT）
  指令 → 遵循指令生成回覆
  能理解意圖，但仍然只能「說」不能「做」

階段 3：推理（Chain-of-Thought）
  問題 → 一步步推理 → 答案
  能進行複雜推理，但答案可能是錯的（幻覺）

階段 4：Agent（ReAct 等）
  任務 → 推理 → 採取行動 → 觀察結果 → 繼續推理
  能與外部世界互動，用真實資訊修正推理
```

### 1.2 Agent 的核心差異

```
傳統 LLM（只思考）：
  問：2024 年台北的人口是多少？
  答：台北的人口大約是 260 萬。（可能已經過時或不準確）

Agent（思考 + 行動）：
  想：使用者問台北人口，我需要查詢最新資料。
  做：search_web("2024 台北市人口數")
  看：搜尋結果顯示 2024 年台北市人口約 249 萬。
  想：找到了最新數據。
  答：根據最新資料，2024 年台北市人口約 249 萬人。
```

---

## 2. 傳統 Chain-of-Thought 的侷限

### 2.1 Chain-of-Thought（CoT）回顧

Chain-of-Thought 是 Wei et al. (2022) 提出的方法，通過讓模型「一步步思考」來提升推理能力。

```
CoT 範例：

問：小明有 15 顆蘋果，給了小華 6 顆，又從媽媽那裡得到 4 顆。
    小明現在有幾顆蘋果？

CoT 推理：
  步驟 1：小明一開始有 15 顆蘋果。
  步驟 2：給了小華 6 顆，所以 15 - 6 = 9 顆。
  步驟 3：從媽媽那裡得到 4 顆，所以 9 + 4 = 13 顆。
  答案：小明現在有 13 顆蘋果。
```

### 2.2 CoT 的侷限

#### 2.2.1 只能用模型已知的知識

```
問：OpenClaw 的最新版本號是什麼？

CoT 推理：
  步驟 1：我需要知道 OpenClaw 的最新版本。
  步驟 2：根據我的訓練資料...
  步驟 3：我不確定最新版本，可能是 0.x...
  → ⚠️ 無法獲取最新資訊，只能猜測
```

#### 2.2.2 推理可能基於錯誤的前提

```
問：計算從台北到巴黎的最佳機票價格。

CoT 推理：
  步驟 1：台北到巴黎的機票通常需要...
  步驟 2：根據我記得的價格，大約是...
  步驟 3：考慮到季節因素...
  → ⚠️ 使用過時的價格資訊，結果不準確
```

#### 2.2.3 無法驗證中間步驟

```
問：Python 3.12 是否支援 match 語句？

CoT 推理：
  步驟 1：match 語句是在 Python 3.10 引入的。
  步驟 2：Python 3.12 是更新的版本。
  步驟 3：所以 Python 3.12 應該支援 match。
  → 雖然結論正確，但推理過程無法被驗證
  → 如果前提錯誤（如 match 不是在 3.10 引入的），整個推理就錯了
```

#### 2.2.4 幻覺問題

```
CoT 的致命缺陷——模型可能自信地推理出完全錯誤的結論：

問：解釋 OpenClaw 的 ResonanceEngine 模組。

CoT 推理：
  步驟 1：OpenClaw 的 ResonanceEngine 是核心模組...
  步驟 2：它負責處理情感共鳴分析...
  步驟 3：使用 BERT 模型進行語意分析...
  → ⚠️ ResonanceEngine 可能根本不存在！
  → 模型「幻覺」出了一個聽起來合理但完全虛構的解釋
```

### 2.3 CoT 侷限的根本原因

```
CoT 的根本問題：

1. 封閉世界假設
   → CoT 只能使用模型訓練時的知識
   → 無法獲取新資訊

2. 無反饋推理
   → 每一步推理都基於前一步的結果
   → 但中間結果無法被驗證
   → 錯誤會累積和放大

3. 只能「思考」不能「行動」
   → 遇到需要查資料的問題只能猜
   → 遇到需要計算的問題只能心算
   → 遇到需要操作的問題只能建議

這就是 ReAct 要解決的問題。
```

---

## 3. ReAct 論文的核心思想

### 3.1 論文資訊

```
標題：ReAct: Synergizing Reasoning and Acting in Language Models
作者：Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak Shafran,
      Karthik Narasimhan, Yuan Cao
機構：Princeton University, Google Brain
發表：ICLR 2023
連結：https://arxiv.org/abs/2210.03629
```

### 3.2 核心洞察

ReAct 的核心洞察可以用一句話概括：

> **推理（Reasoning）和行動（Acting）應該交織進行，互相強化。**

```
人類解決問題的方式：

  想：這道數學題需要用到畢氏定理...
  做：在紙上畫出直角三角形
  看：直角邊分別是 3 和 4
  想：所以斜邊 = √(9+16) = √25 = 5
  做：用計算機驗算 √25
  看：確認是 5
  答：斜邊是 5

人類自然地在「想」和「做」之間切換。
ReAct 讓 AI 也能這樣做。
```

### 3.3 三個觀察

論文的三個關鍵觀察：

**觀察 1：只推理（CoT）會導致幻覺**

```
只推理的 Agent：
  問：Aside from Apple Remote, what other Apple device can control the program Frontrow was designed for?
  想：Frontrow 是一個 Apple 程式...
  想：Apple 的裝置包括 iPhone、iPod...
  答：iPod 可以控制 Frontrow。
  → ❌ 錯誤！（正確答案需要查詢）
```

**觀察 2：只行動（Act-only）缺乏計劃能力**

```
只行動的 Agent：
  問：同上
  做：search("Frontrow Apple")
  看：一堆不相關的結果
  做：search("Frontrow program")
  看：還是不太相關
  做：search("Apple device control Frontrow")
  看：終於找到相關資訊
  → 花了 3 次搜尋才找到，效率低
  → 因為沒有「想」的步驟來指導搜尋策略
```

**觀察 3：推理+行動互相強化**

```
ReAct Agent：
  想：我需要找出 Frontrow 是什麼程式，以及除了 Apple Remote 外
      什麼裝置可以控制它。先搜尋 Frontrow。
  做：search("Front Row Apple software")
  看：Front Row 是 Apple 的媒體中心軟體，可以用 Apple Remote
      或任何支援紅外線的裝置控制。
  想：搜尋結果提到了 Apple Remote，但我需要「其他」裝置。
      讓我找找還有什麼可以控制 Front Row。
  做：search("control Front Row without Apple Remote")
  看：iPhone 和 iPod Touch 的 Remote app 也可以控制 Front Row。
  想：找到了！除了 Apple Remote，iPhone 和 iPod Touch 的 Remote
      app 也可以控制 Front Row。
  答：iPhone 和 iPod Touch（透過 Remote app）可以控制 Front Row。
  → ✅ 正確且高效
```

### 3.4 形式化定義

論文將 ReAct 定義為一個在「推理-行動空間」中的決策過程：

```
傳統 RL Agent：
  狀態 → 行動 → 狀態 → 行動 → ...
  (s₁) → (a₁) → (s₂) → (a₂) → ...

CoT Agent：
  狀態 → 思考 → 思考 → ... → 答案
  (s₁) → (t₁) → (t₂) → ... → (answer)

ReAct Agent：
  狀態 → 思考 → 行動 → 觀察 → 思考 → 行動 → 觀察 → ...
  (s₁) → (t₁) → (a₁) → (o₁) → (t₂) → (a₂) → (o₂) → ...

其中：
  s = 狀態（state）= 使用者輸入 + 歷史上下文
  t = 思考（thought）= 模型的內部推理（自然語言）
  a = 行動（action）= 調用工具或 API
  o = 觀察（observation）= 工具返回的結果
```

---

## 4. Thought → Action → Observation 循環

### 4.1 數學形式化

ReAct 的核心是一個迭代過程。在每個時間步 i：

```
上下文 Cᵢ = (s, t₁, a₁, o₁, t₂, a₂, o₂, ..., tᵢ₋₁, aᵢ₋₁, oᵢ₋₁)

思考生成：tᵢ = LLM(Cᵢ, prompt_think)
  → 模型基於當前上下文，生成推理文字

行動生成：aᵢ = LLM(Cᵢ + tᵢ, prompt_act)
  → 模型基於上下文和當前思考，選擇行動

觀察獲取：oᵢ = Environment(aᵢ)
  → 執行行動，從環境中獲取觀察結果

終止條件：
  如果 tᵢ 包含最終答案 → 結束
  如果 i > max_iterations → 強制結束
  否則 → 繼續下一輪 (i+1)
```

### 4.2 Thought（思考）的角色

```
思考在 ReAct 中扮演多重角色：

1. 分解任務
   思考：「這個問題需要分兩步解決：先找到 X，再用 X 查詢 Y。」

2. 選擇行動
   思考：「我需要搜尋最新資料，所以應該使用 search_web 工具。」

3. 整合資訊
   思考：「前面的搜尋告訴我 A=3，這次搜尋告訴我 B=5，所以 A+B=8。」

4. 處理錯誤
   思考：「上一次搜尋沒有找到結果，我應該換一個關鍵字試試。」

5. 判斷是否完成
   思考：「我已經收集到所有需要的資訊，可以給出最終答案了。」
```

### 4.3 Action（行動）的類型

```
行動類型在 ReAct 原始論文中主要有三種：

1. Search[query]  → 搜尋 Wikipedia
2. Lookup[term]   → 在當前頁面中查找特定術語
3. Finish[answer] → 結束並回答

在 OpenClaw 等現代 Agent 框架中，行動種類更多：

工具調用類：
  search_web(query)        → 網路搜尋
  file_read(path)          → 讀取檔案
  file_write(path, content)→ 寫入檔案
  exec(command)            → 執行命令
  memory_read(key)         → 讀取記憶
  memory_write(key, value) → 寫入記憶

通訊類：
  discord_send(channel, message) → 發送 Discord 訊息
  telegram_send(chat, message)   → 發送 Telegram 訊息

特殊類：
  finish(answer)           → 結束並回答
  ask_user(question)       → 向使用者提問
```

### 4.4 Observation（觀察）的處理

```
觀察是連接 AI 推理和真實世界的橋樑：

行動：search_web("台北天氣")
觀察：「台北今天晴天，氣溫 28°C，相對濕度 65%。」

觀察的特性：
1. 來自真實環境 → 不是模型生成的 → 是「接地」的資訊
2. 可能包含噪音 → 搜尋結果可能包含不相關資訊
3. 可能失敗 → 工具可能返回錯誤
4. 可能很長 → 需要摘要或截斷

處理策略：
  成功的觀察 → 加入上下文，繼續推理
  失敗的觀察 → 生成新的思考，嘗試不同行動
  過長的觀察 → 由框架截斷或摘要
```

### 4.5 循環的視覺化

```
                    ┌──────────┐
                    │  使用者   │
                    │  輸入    │
                    └────┬─────┘
                         │
                         ▼
                  ┌──────────────┐
              ┌──▶│   思考       │
              │   │  (Thought)   │──────── 是否有最終答案？
              │   └──────┬───────┘              │
              │          │                      │ 是
              │          │ 否                    ▼
              │          ▼                 ┌──────────┐
              │   ┌──────────────┐         │  輸出    │
              │   │   行動       │         │  答案    │
              │   │  (Action)    │         └──────────┘
              │   └──────┬───────┘
              │          │
              │          ▼
              │   ┌──────────────┐
              │   │   觀察       │
              │   │(Observation) │
              │   └──────┬───────┘
              │          │
              └──────────┘
                循環繼續
```

---

## 5. 接地（Grounding）——為什麼行動比只是思考重要

### 5.1 什麼是接地？

在認知科學和 AI 中，「接地」（Grounding）指的是**將抽象符號與真實世界的經驗連接起來**。

```
未接地的知識：
  「台北的人口是 X 萬」
  → X 是模型訓練時的資料，可能已經過時
  → 這個知識是「浮在空中」的——與現實脫節

接地的知識：
  search_web("2024 台北市人口")
  → 返回：249 萬
  → 這個知識來自真實的資料來源
  → 是「接地」的——與現實連接
```

### 5.2 Grounding 解決的問題

```
問題 1：知識過時
  CoT：基於訓練資料推理（可能是 2 年前的）
  ReAct：搜尋最新資料 → 使用最新資訊

問題 2：事實錯誤
  CoT：模型可能記錯或混淆事實
  ReAct：查詢可靠來源 → 驗證事實

問題 3：推理錯誤
  CoT：計算可能出錯
  ReAct：使用計算工具 → 得到精確結果

問題 4：幻覺
  CoT：模型可能生成看似合理但虛構的內容
  ReAct：每一步都可以用外部資料驗證
```

### 5.3 接地的層次

```
接地程度：

  完全不接地 ──────────────────────── 完全接地
      │                                  │
  純 CoT                            每步都驗證
  所有推理都                        每個事實都
  來自模型內部                      通過外部查詢確認

  在這個光譜上，ReAct 處於中間偏右的位置：
  → 不是每一步都查詢（那樣太慢）
  → 但在關鍵步驟進行查詢和驗證
  → 讓「思考」指導何時需要「行動」
```

---

## 6. ReAct vs CoT vs Act-only 詳細比較

### 6.1 比較表

| 特性 | CoT (只思考) | Act-only (只行動) | ReAct (思考+行動) |
|------|-------------|-------------------|-------------------|
| 推理能力 | ✅ 強 | ❌ 弱 | ✅ 強 |
| 外部資訊 | ❌ 無 | ✅ 有 | ✅ 有 |
| 幻覺風險 | ❌ 高 | ⚠️ 中 | ✅ 低 |
| 效率 | ✅ 高（無工具調用） | ⚠️ 低（盲目嘗試） | ✅ 中高（有計劃） |
| 可解釋性 | ✅ 高（看到推理） | ❌ 低（只看到動作） | ✅ 最高（看到推理和動作） |
| 適用場景 | 純邏輯推理 | 簡單的查詢任務 | 複雜的多步驟任務 |
| 錯誤恢復 | ❌ 差 | ⚠️ 隨機重試 | ✅ 有策略的重試 |
| Token 消耗 | ⚠️ 中 | ⚠️ 中 | ❌ 高 |

### 6.2 同一問題的三種方法比較

```
問題：「蘋果公司在 2023 年的營收是多少？和 2022 年比增長了多少？」

═══ CoT (只思考) ═══

想：蘋果公司 2023 年的營收...根據我的知識，大約是 3940 億美元。
想：2022 年的營收大約是 3943 億美元。
想：所以增長率 = (3940 - 3943) / 3943 ≈ -0.08%，略有下降。
答：蘋果公司 2023 年營收約 3940 億美元，較 2022 年略有下降。

→ ⚠️ 數字可能不準確，但推理邏輯正確

═══ Act-only (只行動) ═══

做：search("Apple revenue 2023")
看：Apple 2023 fiscal year revenue was $383.3 billion.
做：search("Apple revenue 2022")
看：Apple 2022 fiscal year revenue was $394.3 billion.
做：calculate("(383.3 - 394.3) / 394.3 * 100")
看：-2.79%
答：蘋果 2023 營收 3833 億，比 2022 年下降 2.79%。

→ 找到了正確數據，但搜尋策略是隨機的，沒有解釋「為什麼這樣搜」

═══ ReAct (思考+行動) ═══

想：使用者問的是蘋果公司的營收比較。我需要先查詢 2023 年的數據。
做：search("Apple Inc fiscal year 2023 annual revenue")
看：Apple's FY2023 revenue was $383.3 billion.
想：找到了 2023 年的數據。現在需要 2022 年的數據來計算增長率。
做：search("Apple Inc fiscal year 2022 annual revenue")
看：Apple's FY2022 revenue was $394.3 billion.
想：現在有兩年的數據了。讓我計算增長率：
    增長率 = (383.3 - 394.3) / 394.3 × 100 = -2.79%
    所以是下降了 2.79%。
答：蘋果公司 2023 財年營收為 3833 億美元，較 2022 財年的 3943 億美元
    下降了約 2.79%。

→ ✅ 數據準確、推理清晰、過程可追蹤
```

### 6.3 實驗結果

根據 ReAct 論文，在 HotpotQA（多跳問答）和 FEVER（事實驗證）任務上：

```
HotpotQA 準確率：
  Standard (無 CoT)  : 28.7%
  CoT (只思考)       : 29.4%
  Act-only (只行動)  : 25.7%
  ReAct (思考+行動)  : 27.4% (但幻覺率大幅降低)
  ReAct + CoT 混合   : 35.1% ← 最佳

FEVER 準確率：
  Standard           : 57.1%
  CoT                : 56.3%
  Act-only           : 58.9%
  ReAct              : 60.9%
  ReAct + CoT 混合   : 64.6% ← 最佳

關鍵發現：
  ReAct 的純準確率不一定最高，
  但它的幻覺率最低、可解釋性最好、
  並且和 CoT 結合後效果最佳。
```

---

## 7. ReAct 的實作範式

### 7.1 基本實作（Python 虛擬碼）

```python
class ReActAgent:
    def __init__(self, llm, tools, max_iterations=10):
        self.llm = llm          # 語言模型
        self.tools = tools      # 可用工具字典
        self.max_iterations = max_iterations

    def run(self, user_input: str) -> str:
        """執行 ReAct 循環"""
        # 初始化上下文
        context = [
            {"role": "system", "content": self.build_system_prompt()},
            {"role": "user", "content": user_input}
        ]

        for i in range(self.max_iterations):
            # 步驟 1：讓 LLM 思考並決定行動
            response = self.llm.generate(context)

            # 步驟 2：解析回應
            thought, action = self.parse_response(response)

            # 記錄思考
            context.append({
                "role": "assistant",
                "content": f"Thought: {thought}"
            })

            # 步驟 3：檢查是否完成
            if action.type == "finish":
                return action.answer

            # 步驟 4：執行行動
            try:
                observation = self.execute_action(action)
            except ToolError as e:
                observation = f"Error: {e}"

            # 步驟 5：將觀察加入上下文
            context.append({
                "role": "tool",
                "content": f"Observation: {observation}"
            })

        # 超過最大迭代次數
        return "抱歉，我無法在規定的步驟內完成這個任務。"

    def build_system_prompt(self) -> str:
        """建構系統提示"""
        tool_descriptions = "\n".join(
            f"- {name}: {tool.description}"
            for name, tool in self.tools.items()
        )
        return f"""你是一個 ReAct Agent。你需要交替進行思考和行動來解決問題。

可用工具：
{tool_descriptions}

格式：
Thought: [你的推理]
Action: [工具名稱]([參數])

或者如果你已經知道答案：
Thought: [你的推理]
Action: finish([最終答案])

每次只執行一個行動。"""

    def parse_response(self, response: str):
        """解析 LLM 的回應為思考和行動"""
        thought_match = re.search(r'Thought: (.+?)(?=Action:)', response, re.DOTALL)
        action_match = re.search(r'Action: (.+?)(?:\((.+?)\))?$', response)

        thought = thought_match.group(1).strip() if thought_match else ""
        action_name = action_match.group(1).strip()
        action_args = action_match.group(2).strip() if action_match.group(2) else ""

        return thought, Action(type=action_name, args=action_args)

    def execute_action(self, action) -> str:
        """執行行動並返回觀察"""
        if action.type not in self.tools:
            return f"Error: Unknown tool '{action.type}'"

        tool = self.tools[action.type]

        # 權限檢查（OpenClaw 的九層模型會在這裡介入）
        if not self.check_permissions(action):
            return f"Error: Permission denied for tool '{action.type}'"

        return tool.execute(action.args)
```

### 7.2 進階實作——帶有記憶和上下文管理

```python
class AdvancedReActAgent:
    def __init__(self, llm, tools, memory, config):
        self.llm = llm
        self.tools = tools
        self.memory = memory
        self.config = config

    async def run(self, user_input: str, session_id: str) -> AsyncIterator[str]:
        """異步執行 ReAct 循環，支持串流輸出"""

        # 上下文組裝（Context Assembly）
        context = await self.assemble_context(user_input, session_id)

        for i in range(self.config.max_iterations):
            # 生成思考和行動（串流）
            async for token in self.llm.stream_generate(context):
                yield token  # 串流輸出

            # 解析完整回應
            response = self.collect_streamed_response()
            thought, action = self.parse_response(response)

            # 觸發 hooks
            await self.hooks.post_inference(thought, action)

            if action.type == "finish":
                # 持久化：保存到記憶
                await self.memory.save_interaction(session_id, user_input, action.answer)
                return

            # 執行行動（帶權限檢查和沙箱）
            await self.hooks.pre_tool(action)
            observation = await self.execute_with_safety(action, session_id)
            await self.hooks.post_tool(action, observation)

            # 更新上下文
            context = self.update_context(context, thought, action, observation)

            # Token 壓縮（如果上下文太長）
            if self.estimate_tokens(context) > self.config.max_context_tokens:
                context = await self.compress_context(context)

    async def assemble_context(self, user_input, session_id):
        """組裝完整上下文"""
        return {
            "system": self.build_system_prompt(),
            "persona": await self.load_persona(),           # SOUL.md
            "user_profile": await self.load_user_profile(), # USER.md
            "memory": await self.memory.recall(user_input), # 相關記憶
            "history": await self.get_session_history(session_id),
            "user_input": user_input
        }
```

---

## 8. 工具調用的標準化

### 8.1 Function Calling（OpenAI 風格）

```python
# OpenAI Function Calling 格式
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_web",
            "description": "搜尋網路以獲取最新資訊",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "搜尋查詢關鍵字"
                    }
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "memory_read",
            "description": "從記憶中讀取之前儲存的資訊",
            "parameters": {
                "type": "object",
                "properties": {
                    "key": {
                        "type": "string",
                        "description": "記憶的鍵值"
                    }
                },
                "required": ["key"]
            }
        }
    }
]

# API 調用
response = openai.chat.completions.create(
    model="gpt-4",
    messages=messages,
    tools=tools,
    tool_choice="auto"  # 讓模型自己決定是否使用工具
)
```

### 8.2 Tool Use（Anthropic 風格）

```python
# Anthropic Tool Use 格式
response = anthropic.messages.create(
    model="claude-3-opus-20240229",
    max_tokens=4096,
    tools=[
        {
            "name": "search_web",
            "description": "搜尋網路以獲取最新資訊",
            "input_schema": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "搜尋查詢"
                    }
                },
                "required": ["query"]
            }
        }
    ],
    messages=messages
)
```

### 8.3 MCP（Model Context Protocol）

```
MCP 是 Anthropic 提出的開放標準，
用於標準化 AI 模型與外部工具的互動：

特點：
1. 統一的工具描述格式
2. 支援多種傳輸方式（stdio、HTTP、WebSocket）
3. 支援資源（Resources）和提示（Prompts）
4. 開源標準，多家支援

OpenClaw 支援 MCP，這意味著：
→ 任何 MCP 相容的工具都可以被 OpenClaw 使用
→ OpenClaw 也可以作為 MCP Server 供其他系統調用
```

---

## 9. 錯誤恢復與重試策略

### 9.1 錯誤類型

```
ReAct 循環中可能遇到的錯誤：

1. 工具不存在
   → Agent 嘗試調用一個不存在的工具
   → 解決：告知模型可用工具列表

2. 參數錯誤
   → Agent 提供了錯誤的參數格式
   → 解決：返回錯誤訊息，讓模型修正

3. 權限被拒
   → 工具調用被權限系統拒絕
   → 解決：告知模型該工具不可用

4. 工具執行失敗
   → 網路錯誤、API 限額等
   → 解決：重試或使用替代工具

5. 超時
   → 工具執行時間過長
   → 解決：終止並告知模型

6. 幻覺工具調用
   → 模型生成了不符合格式的工具調用
   → 解決：解析錯誤時重新生成
```

### 9.2 重試策略

```python
class RetryStrategy:
    def __init__(self, max_retries=3, backoff_factor=2):
        self.max_retries = max_retries
        self.backoff_factor = backoff_factor

    async def execute_with_retry(self, action, context):
        for attempt in range(self.max_retries):
            try:
                result = await self.execute_action(action)
                return result
            except TransientError as e:
                # 暫時性錯誤（網路、超時）→ 重試
                wait_time = self.backoff_factor ** attempt
                await asyncio.sleep(wait_time)
                continue
            except PermissionError as e:
                # 權限錯誤 → 不重試，告知模型
                return f"Permission denied: {e}"
            except ToolNotFoundError as e:
                # 工具不存在 → 不重試，告知模型
                return f"Tool not found: {e}"
            except Exception as e:
                # 其他錯誤 → 記錄並返回錯誤
                logger.error(f"Tool error: {e}")
                return f"Error: {e}"

        return "Error: Max retries exceeded"
```

### 9.3 錯誤恢復範例

```
場景：搜尋 API 暫時不可用

循環 1：
  想：需要搜尋台北天氣。
  做：search_web("台北天氣")
  看：Error: Search API temporarily unavailable

循環 2：
  想：搜尋 API 暫時不可用，讓我再試一次。
  做：search_web("台北天氣")
  看：Error: Search API temporarily unavailable

循環 3：
  想：搜尋 API 持續不可用。我可以嘗試用其他方式獲取資訊，
      或者根據我的知識提供一個大致的回答。
  做：finish("抱歉，我目前無法查詢即時天氣資訊。
      建議你查看中央氣象局網站或使用天氣 App 獲取最新資訊。")

→ Agent 在多次失敗後，選擇了優雅的降級策略
```

---

## 10. 多步推理的挑戰

### 10.1 幻覺在長鏈推理中的累積

```
隨著推理步驟增加，幻覺風險也增加：

步驟 1：正確率 95%
步驟 2：正確率 95% × 95% = 90.25%
步驟 3：正確率 95% × 95% × 95% = 85.74%
...
步驟 10：正確率 95%¹⁰ = 59.87%

即使每一步有 95% 的正確率，10 步之後整體正確率只有 60%！

ReAct 如何緩解：
→ 在關鍵步驟使用工具驗證
→ 如果中間結果可疑，重新搜尋驗證
→ 減少不必要的推理步驟
```

### 10.2 工具失敗的級聯效應

```
步驟 1：搜尋 X → 成功
步驟 2：用 X 的結果搜尋 Y → 成功
步驟 3：用 Y 的結果計算 Z → 失敗（API 錯誤）
步驟 4：??? → 無法繼續（因為沒有 Z）

解決策略：
1. 重試步驟 3
2. 使用替代工具計算 Z
3. 回溯到步驟 2，嘗試不同路徑
4. 告知使用者部分結果
```

### 10.3 無限循環問題

```
危險場景：Agent 陷入循環

循環 1：
  想：我需要找到 X。
  做：search("X")
  看：沒有找到 X。

循環 2：
  想：上次沒找到，讓我再搜尋一次。
  做：search("X")
  看：沒有找到 X。

循環 3：
  想：還是沒找到，再試一次...
  → ♾️ 無限循環！

解決方案：
1. 設定最大迭代次數 (max_iterations)
2. 偵測重複行動
3. 在系統提示中指導：「如果同一搜尋連續失敗兩次，嘗試不同策略」
4. 計時器強制終止
```

### 10.4 上下文窗口溢出

```
長對話或多步推理會消耗大量 token：

系統提示：  ~500 tokens
使用者輸入：~100 tokens
循環 1：    ~200 tokens (想+做+看)
循環 2：    ~300 tokens
循環 3：    ~400 tokens
...
循環 10：   ~500 tokens
累計：      ~3500+ tokens（還沒算觀察結果的長度）

如果觀察結果很長（如搜尋返回大量文字）：
→ 上下文可能超過模型的窗口限制

解決策略：
1. 摘要舊的循環（只保留關鍵資訊）
2. 截斷長的觀察結果
3. 使用更長上下文的模型
4. 限制每次觀察的最大長度
```

---

## 11. ReAct 的局限與未來發展

### 11.1 當前局限

```
1. 效率問題
   → 每次行動都需要一次 LLM 調用
   → 多步推理需要多次 API 調用
   → 成本和延遲都較高

2. 計劃能力有限
   → ReAct 是逐步推理的（greedy）
   → 沒有全域計劃能力
   → 可能陷入局部最優

3. 工具依賴
   → 如果可用工具有限，ReAct 的優勢就小
   → 工具的品質直接影響結果

4. 上下文限制
   → 長對話會超出上下文窗口
   → 需要壓縮或截斷

5. 單一路徑
   → ReAct 每次只探索一條路徑
   → 如果選擇了錯誤的行動，需要回溯
```

### 11.2 Tree of Thoughts（思考樹）

```
ReAct: 線性路徑
  想 → 做 → 看 → 想 → 做 → 看 → 答

Tree of Thoughts: 分支探索
         想
        / | \
       /  |  \
      做  做  做
      |   |   |
      看  看  看
      |   |   |
      想  想  想
      |       |
     最佳    次佳
      |
      答

特點：
→ 同時探索多條路徑
→ 可以比較不同路徑的結果
→ 選擇最佳路徑
→ 但成本更高
```

### 11.3 Planning（計劃）

```
ReAct: 逐步決策
  步驟 1 → 步驟 2 → 步驟 3 → ...
  （每一步才決定下一步做什麼）

Planning: 先計劃再執行
  計劃階段：分析任務 → 制定計劃 → 分解子任務
  執行階段：按計劃逐步執行
  調整階段：根據結果調整計劃

範例：
  任務：「幫我整理一份 2023 年科技新聞摘要」

  ReAct 方式：
    想：搜尋 2023 科技新聞
    做：search(...)
    想：找到了一些，再搜尋更多
    做：search(...)
    ...

  Planning 方式：
    計劃：
      1. 搜尋 2023 年 Q1 科技新聞
      2. 搜尋 2023 年 Q2 科技新聞
      3. 搜尋 2023 年 Q3 科技新聞
      4. 搜尋 2023 年 Q4 科技新聞
      5. 整合所有搜尋結果
      6. 按重要性排序
      7. 撰寫摘要
    執行：按計劃逐步完成
```

### 11.4 Multi-Agent（多代理）

```
單一 ReAct Agent 的限制：一個模型做所有事

Multi-Agent 方式：
  ┌───────────┐
  │ 規劃 Agent │ → 制定計劃
  └─────┬─────┘
        │
  ┌─────▼─────┐
  │ 搜尋 Agent │ → 負責收集資訊
  └─────┬─────┘
        │
  ┌─────▼─────┐
  │ 分析 Agent │ → 負責分析資料
  └─────┬─────┘
        │
  ┌─────▼─────┐
  │ 撰寫 Agent │ → 負責生成最終回答
  └─────┬─────┘
        │
       答案

每個 Agent 可以使用不同的模型和工具，
專注於自己擅長的任務。
```

---

## 12. 與人類思考方式的類比

### 12.1 人類的自然思考模式

```
人類解決問題時，自然地在「想」和「做」之間切換：

場景：修理壞掉的水龍頭

  想：水龍頭漏水，可能是墊圈壞了。
  做：拿出工具箱，檢查水龍頭。
  看：確實是墊圈磨損了。
  想：需要一個新墊圈。我記得五金行在街角。
  做：去五金行買墊圈。
  看：買到了合適的墊圈。
  想：現在可以更換了。先關水閥。
  做：關閉水閥，拆下舊墊圈，裝上新的。
  看：水龍頭不漏了！
  想：修好了。

這就是 ReAct 模式——
人類不會先在腦中完整模擬修理過程（純 CoT），
也不會不思考就亂動手（純 Act），
而是邊想邊做。
```

### 12.2 認知科學的支持

```
ReAct 模式與認知科學中的多個理論呼應：

1. 體化認知（Embodied Cognition）
   → 認知不只在大腦中發生，而是與身體和環境互動的結果
   → ReAct 讓 AI 通過「行動」與環境互動

2. 情境認知（Situated Cognition）
   → 知識在特定情境中才有意義
   → ReAct 通過觀察獲取情境資訊

3. 分散式認知（Distributed Cognition）
   → 認知分散在人、工具和環境之間
   → ReAct 將推理分散在 AI 模型和外部工具之間

4. 探索式搜尋（Heuristic Search）
   → 人類通過經驗法則和試錯來解決問題
   → ReAct 的思考步驟就是啟發式指導
```

### 12.3 AI Agent 的思考方式演化

```
GPT-1/2: 鸚鵡學舌
  → 重複訓練資料中的模式
  → 沒有「思考」能力

GPT-3: 開始推理
  → Few-shot 學習
  → 初步的邏輯推理

CoT: 顯式推理
  → 一步步思考
  → 但封閉在模型內部

ReAct: 推理+行動
  → 與外部世界互動
  → 用真實資訊修正推理

未來: ???
  → 長期規劃
  → 多模態感知
  → 自主學習
  → 與人類協作

每一步都讓 AI 更接近人類的思考方式。
ReAct 是其中關鍵的一步——
它讓 AI 從「象牙塔中的思考者」
變成「腳踏實地的行動者」。
```

---

## 總結

ReAct 模式的核心價值在於：

1. **統一推理和行動**：不再是「只想不做」或「只做不想」
2. **接地**：用真實世界的資訊修正推理
3. **可解釋**：每一步推理和行動都是透明的
4. **容錯**：可以從錯誤中恢復
5. **靈活**：適用於各種任務

OpenClaw 作為一個 AI Agent 框架，其推理引擎的核心就是 ReAct 模式。下一章將深入解析 OpenClaw 如何實作這個推理迴圈。

---

> **參考文獻**：
> - Yao, S., et al. (2023). "ReAct: Synergizing Reasoning and Acting in Language Models." ICLR 2023.
> - Wei, J., et al. (2022). "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models." NeurIPS 2022.
> - Yao, S., et al. (2023). "Tree of Thoughts: Deliberate Problem Solving with Large Language Models."
> - Schick, T., et al. (2023). "Toolformer: Language Models Can Teach Themselves to Use Tools."
> - Nakano, R., et al. (2021). "WebGPT: Browser-assisted question-answering with human feedback."
