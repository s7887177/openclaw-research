# OpenClaw Skills 概念與結構——用 Markdown 定義 AI 的能力

## 目錄

- [1. 引言：為什麼用 Markdown 定義能力？](#1-引言為什麼用-markdown-定義能力)
- [2. Skills 設計哲學](#2-skills-設計哲學)
  - [2.1 從程式碼到文件：範式轉移](#21-從程式碼到文件範式轉移)
  - [2.2 聲明式 vs 命令式](#22-聲明式-vs-命令式)
  - [2.3 人類可讀即機器可讀](#23-人類可讀即機器可讀)
  - [2.4 Markdown 的獨特優勢](#24-markdown-的獨特優勢)
- [3. 技能的生命週期](#3-技能的生命週期)
  - [3.1 發現（Discovery）](#31-發現discovery)
  - [3.2 註冊（Registration）](#32-註冊registration)
  - [3.3 調用（Invocation）](#33-調用invocation)
  - [3.4 回傳（Response）](#34-回傳response)
  - [3.5 生命週期完整流程圖](#35-生命週期完整流程圖)
- [4. 目錄結構詳解](#4-目錄結構詳解)
  - [4.1 根目錄：SKILL.md](#41-根目錄skillmd)
  - [4.2 scripts/ 目錄](#42-scripts-目錄)
  - [4.3 references/ 目錄](#43-references-目錄)
  - [4.4 tests/ 目錄](#44-tests-目錄)
  - [4.5 assets/ 目錄](#45-assets-目錄)
  - [4.6 完整目錄範例](#46-完整目錄範例)
- [5. YAML Frontmatter 格式](#5-yaml-frontmatter-格式)
  - [5.1 基本欄位](#51-基本欄位)
  - [5.2 metadata.openclaw 欄位](#52-metadataopenclaw-欄位)
  - [5.3 requires 環境需求](#53-requires-環境需求)
  - [5.4 完整 Frontmatter 範例](#54-完整-frontmatter-範例)
- [6. Markdown 正文格式](#6-markdown-正文格式)
  - [6.1 Purpose 區段](#61-purpose-區段)
  - [6.2 Instructions 區段](#62-instructions-區段)
  - [6.3 Constraints 區段](#63-constraints-區段)
  - [6.4 Examples 區段](#64-examples-區段)
  - [6.5 可選區段](#65-可選區段)
- [7. 技能分類與區別](#7-技能分類與區別)
  - [7.1 內建技能（Built-in Skills）](#71-內建技能built-in-skills)
  - [7.2 社群技能（Community Skills）](#72-社群技能community-skills)
  - [7.3 自定義技能（Custom Skills）](#73-自定義技能custom-skills)
  - [7.4 三者比較表](#74-三者比較表)
- [8. ClawHub 市集](#8-clawhub-市集)
  - [8.1 市集概覽](#81-市集概覽)
  - [8.2 運作方式](#82-運作方式)
  - [8.3 搜尋與安裝](#83-搜尋與安裝)
  - [8.4 品質保證機制](#84-品質保證機制)
  - [8.5 商業模式](#85-商業模式)
- [9. 完整的 SKILL.md 範例](#9-完整的-skillmd-範例)
  - [9.1 簡單範例：天氣查詢技能](#91-簡單範例天氣查詢技能)
  - [9.2 中等範例：Markdown 筆記管理技能](#92-中等範例markdown-筆記管理技能)
  - [9.3 複雜範例：GitHub 專案管理技能](#93-複雜範例github-專案管理技能)
- [10. 設計模式與最佳實踐](#10-設計模式與最佳實踐)
- [11. 總結與展望](#11-總結與展望)

---

## 1. 引言：為什麼用 Markdown 定義能力？

在傳統的軟體開發中，當我們想要為一個系統新增能力時，通常的做法是撰寫程式碼——定義函式、類別、API 端點。這種方法在傳統軟體工程中行之有效，但在 AI 智能體（Agent）的時代，這個範式正在被重新思考。

OpenClaw 做出了一個看似大膽、實則深思熟慮的設計決策：**用 Markdown 文件來定義 AI 的能力**。這不是偷懶，也不是簡化，而是一種對 LLM 本質的深刻理解所帶來的範式轉移。

想像一下這個場景：你想讓你的 AI 助手具備「查詢天氣」的能力。在傳統框架中，你可能需要：

1. 撰寫一個 Python 函式定義
2. 定義 JSON Schema 描述參數
3. 撰寫路由邏輯
4. 處理序列化/反序列化
5. 編寫測試程式

而在 OpenClaw 中，你只需要：

1. 建立一個 `SKILL.md` 文件
2. 用自然語言描述這個技能做什麼、怎麼用
3. 如果需要外部呼叫，放一個腳本在 `scripts/` 目錄

這就是 Skills 系統的核心理念——**讓定義能力的門檻低到任何人都能參與，同時保持足夠的表達力來處理複雜場景**。

本章將深入探討 Skills 系統的設計哲學、完整的生命週期、目錄結構規範、文件格式，以及生態系統的運作方式。無論你是想使用現有技能、修改技能，還是從零開始建立自己的技能，這裡都是你的起點。

---

## 2. Skills 設計哲學

### 2.1 從程式碼到文件：範式轉移

OpenClaw 的 Skills 設計哲學根植於一個核心觀察：**大型語言模型（LLM）的原生語言是自然語言，不是程式碼**。

在 Function Calling 框架中（如 OpenAI 的 functions API），我們用 JSON Schema 來描述工具：

```json
{
  "name": "get_weather",
  "description": "Get current weather for a location",
  "parameters": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "City name"
      }
    }
  }
}
```

這種方式有幾個問題：

- **描述空間有限**：JSON Schema 的 `description` 欄位通常只有一行，很難傳達複雜的使用情境
- **缺乏範例**：LLM 是 few-shot learner，沒有範例就容易誤用工具
- **約束難以表達**：「不要在使用者沒有明確要求時主動查天氣」這種約束，在 JSON Schema 中無處可放
- **維護困難**：JSON 的嚴格語法讓非程式設計師望而卻步

OpenClaw 的解法是：既然 LLM 理解自然語言最好，那就用自然語言來描述工具。而 Markdown 是結構化自然語言的最佳載體——它既有足夠的結構（標題、列表、程式碼區塊），又保持了人類的可讀性。

### 2.2 聲明式 vs 命令式

Skills 系統採用**聲明式（Declarative）**設計：

- **聲明式**（OpenClaw Skills）：「這個技能**是什麼**、**能做什麼**、**應該在何時使用**」
- **命令式**（傳統框架）：「收到請求後，先做 A，再做 B，然後回傳 C」

聲明式的好處在於，它把「如何執行」的決策權交給了 LLM 的推理引擎（ReAct Loop）。LLM 會根據上下文、使用者意圖、以及技能的描述，自主決定：

1. 是否需要使用這個技能
2. 使用哪些參數
3. 如何組合多個技能的結果
4. 如何呈現結果給使用者

這種設計讓技能的組合變得極為靈活。你不需要預先定義「查天氣然後推薦穿搭」的工作流程——只要有「天氣查詢」和「穿搭推薦」兩個技能，LLM 自己就能把它們串起來。

### 2.3 人類可讀即機器可讀

OpenClaw Skills 的一個核心設計原則是：**如果一個人類開發者能讀懂這份 SKILL.md 並正確使用這個工具，那麼 LLM 也能**。

這意味著：

- 寫 SKILL.md 就像寫技術文件一樣
- 你的讀者同時是人類和 AI
- 越清晰的描述，AI 越不容易犯錯
- 範例不只是給人看的，也是給 AI 做 few-shot learning 的

這個原則帶來了一個意外的好處：**Skills 的文件化是自動的**。SKILL.md 本身就是這個技能的完整文件。你不需要額外維護一份 README 或 API 文件——技能定義本身就是文件。

### 2.4 Markdown 的獨特優勢

為什麼選擇 Markdown 而不是其他格式？

| 特性 | Markdown | JSON/YAML | XML | 自然語言純文字 |
|------|----------|-----------|-----|--------------|
| 人類可讀性 | ★★★★★ | ★★★☆☆ | ★★☆☆☆ | ★★★★★ |
| 結構化程度 | ★★★★☆ | ★★★★★ | ★★★★★ | ★☆☆☆☆ |
| LLM 理解度 | ★★★★★ | ★★★★☆ | ★★★☆☆ | ★★★★☆ |
| 版本控制友好 | ★★★★★ | ★★★★☆ | ★★★☆☆ | ★★★★★ |
| 學習曲線 | ★★★★★ | ★★★☆☆ | ★★☆☆☆ | ★★★★★ |
| 內嵌程式碼 | ★★★★★ | ★★☆☆☆ | ★★☆☆☆ | ★☆☆☆☆ |

Markdown 在各項指標上都有極佳的平衡。特別是它的「內嵌程式碼區塊」能力——你可以在一份 Markdown 文件中同時包含自然語言描述和結構化的程式碼範例，這對於 LLM 理解技能的使用方式至關重要。

此外，Markdown 是軟體開發社群最廣泛使用的文件格式。GitHub、GitLab、各種文件框架都原生支援 Markdown。這意味著 Skills 可以無縫整合到現有的開發工作流程中。

---

## 3. 技能的生命週期

每個技能在 OpenClaw 系統中都經歷四個階段的生命週期：**發現（Discovery）→ 註冊（Registration）→ 調用（Invocation）→ 回傳（Response）**。理解這個生命週期對於有效開發和除錯技能至關重要。

### 3.1 發現（Discovery）

當 OpenClaw 啟動時（或當你執行 `openclaw skills reload` 時），系統會掃描以下目錄來發現技能：

```
# 掃描順序（優先級由高到低）
1. ~/.openclaw/skills/           # 使用者自定義技能
2. ./skills/                     # 專案級技能
3. ~/.openclaw/community-skills/ # 已安裝的社群技能
4. <openclaw-install>/skills/    # 內建技能
```

發現過程的具體步驟：

1. **目錄遍歷**：遞迴掃描上述目錄
2. **文件識別**：尋找名為 `SKILL.md` 的文件
3. **Frontmatter 解析**：讀取每個 SKILL.md 的 YAML frontmatter
4. **環境檢查**：驗證 `requires` 中指定的二進位工具和環境變數是否可用
5. **衝突偵測**：如果多個目錄有同名技能，高優先級目錄的版本會覆蓋低優先級的
6. **相依性檢查**：確認技能所需的其他技能是否已發現

```bash
# 查看已發現的技能
openclaw skills list

# 輸出範例：
# ┌─────────────────┬──────────┬────────────┬──────────┐
# │ Name            │ Version  │ Source     │ Status   │
# ├─────────────────┼──────────┼────────────┼──────────┤
# │ web-search      │ 1.2.0    │ built-in   │ active   │
# │ file-manager    │ 1.0.0    │ built-in   │ active   │
# │ weather         │ 0.3.1    │ community  │ active   │
# │ my-custom-skill │ 0.1.0    │ custom     │ active   │
# │ docker-deploy   │ 1.1.0    │ community  │ disabled │
# └─────────────────┴──────────┴────────────┴──────────┘
```

注意 `docker-deploy` 技能顯示 `disabled`——這通常意味著它的環境需求（例如 `docker` 二進位）未被滿足。

### 3.2 註冊（Registration）

發現階段完成後，所有活躍的技能會被「註冊」到 ReAct 推理引擎中。註冊過程如下：

1. **SKILL.md 完整讀取**：系統讀取 SKILL.md 的完整內容（Frontmatter + 正文）
2. **系統提示詞注入**：技能的描述被壓縮並注入到 LLM 的系統提示詞（System Prompt）中
3. **工具描述生成**：如果技能包含腳本，系統會自動生成對應的 Function Calling 工具描述
4. **上下文窗口管理**：如果技能過多超出上下文窗口，系統會使用動態載入策略

動態載入策略是一個重要的設計決策。當技能數量過多時，不可能把所有技能的完整描述都塞進系統提示詞。OpenClaw 使用以下策略：

```
Level 1（始終載入）：技能名稱 + 一句話描述
Level 2（按需載入）：完整的 Purpose 和 Instructions
Level 3（執行時載入）：Examples 和 Constraints
```

當 LLM 的 ReAct 推理引擎認為某個技能可能與當前任務相關時，會從 Level 1 升級到 Level 2；當決定使用該技能時，會載入到 Level 3。這個機制確保了即使安裝了數百個技能，系統仍然能高效運作。

### 3.3 調用（Invocation）

當使用者發出請求，ReAct 推理引擎判斷需要使用某個技能時，調用過程如下：

1. **意圖匹配**：LLM 根據使用者意圖和技能描述，選擇最適合的技能
2. **參數生成**：LLM 根據技能的 Instructions 和 Examples，生成調用參數
3. **約束檢查**：系統檢查 Constraints 中定義的限制條件
4. **安全門控**：ControlPolicy 層級檢查（允許/需確認/拒絕）
5. **執行分派**：
   - 如果技能純文字（無腳本）：LLM 直接根據 Instructions 生成回應
   - 如果技能有腳本：系統執行 `scripts/` 中對應的腳本
6. **超時控制**：如果執行超過 `metadata.openclaw.timeout` 設定，自動中止

純文字技能和腳本技能的區別非常重要：

**純文字技能**是最簡單的形式，完全依賴 LLM 的推理能力。例如，一個「翻譯技能」可能只需要告訴 LLM「你現在是一個翻譯員，請將使用者的輸入翻譯成指定語言」，不需要任何外部呼叫。

**腳本技能**則涉及外部系統互動。例如，「天氣查詢技能」需要呼叫外部 API，這時就需要一個腳本來處理 HTTP 請求和回應解析。

```
調用流程示意：

使用者: "台北今天天氣如何？"
     │
     ▼
ReAct 推理引擎
     │
     ├── 思考: 使用者想知道天氣
     ├── 行動: 選擇 weather 技能
     ├── 參數: { location: "台北", date: "today" }
     │
     ▼
安全門控 ── 通過 ──▶ 執行腳本: scripts/get_weather.py
     │                       │
     │                       ▼
     │                API 回應: { temp: 28, ... }
     │                       │
     ▼                       ▼
  回傳結果給 ReAct 引擎
     │
     ▼
LLM 整合結果: "台北今天晴天，氣溫 28°C..."
```

### 3.4 回傳（Response）

技能執行完成後，結果會經過以下處理：

1. **結果接收**：腳本的 stdout 輸出或 LLM 的生成文字被收集
2. **格式化**：根據技能定義的輸出格式進行整理
3. **上下文注入**：結果被注入回 ReAct Loop 的觀察（Observation）環節
4. **連鎖判斷**：LLM 判斷是否需要進一步操作（呼叫其他技能）
5. **最終回應**：所有必要的技能調用完成後，LLM 生成最終的使用者回應

回傳階段一個關鍵的設計是**連鎖判斷**。LLM 不一定在調用完一個技能後就直接回應使用者——它可能決定需要更多資訊。例如：

```
使用者: "我明天要去台北出差，幫我準備一下"
     │
     ▼
思考 → 行動: weather("台北", "明天")
觀察 → 明天台北有雨，20°C
     │
     ▼
思考 → 行動: calendar("明天", "查詢")
觀察 → 明天有三個會議
     │
     ▼
思考 → 行動: transport("current_location", "台北", "明天")
觀察 → 高鐵 06:30 出發，08:30 到達
     │
     ▼
最終回應: "明天台北有雨，記得帶傘。你有三個會議，
          建議搭 06:30 的高鐵..."
```

### 3.5 生命週期完整流程圖

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenClaw Skills 生命週期                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │ 發現      │───▶│ 註冊      │───▶│ 調用      │───▶│ 回傳      │  │
│  │ Discovery │    │ Register │    │ Invoke   │    │ Response │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│       │               │               │               │        │
│  ┌────┴────┐    ┌────┴────┐    ┌────┴────┐    ┌────┴────┐     │
│  │掃描目錄  │    │解析文件  │    │意圖匹配  │    │結果收集  │     │
│  │環境檢查  │    │注入提示詞│    │參數生成  │    │格式化    │     │
│  │衝突偵測  │    │生成工具  │    │安全門控  │    │連鎖判斷  │     │
│  │依賴檢查  │    │動態載入  │    │執行分派  │    │最終回應  │     │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 目錄結構詳解

每個 OpenClaw 技能都是一個獨立的目錄，有明確的結構規範。理解這個結構是開發技能的基礎。

### 4.1 根目錄：SKILL.md

`SKILL.md` 是每個技能**唯一必要**的文件。它就像一個 npm 套件的 `package.json` 和 `README.md` 的結合——既包含元數據（YAML Frontmatter），也包含完整的技能描述（Markdown 正文）。

```
my-skill/
└── SKILL.md    ← 唯一必要的文件
```

如果你的技能是純文字技能（不需要外部呼叫），只需要這一個文件就夠了。例如，一個「程式碼審查」技能可能只需要告訴 LLM 如何進行程式碼審查的準則和步驟，完全不需要任何腳本。

### 4.2 scripts/ 目錄

`scripts/` 目錄包含技能需要執行的腳本文件。支援多種語言：

```
my-skill/
├── SKILL.md
└── scripts/
    ├── main.py          # Python 腳本
    ├── helper.sh        # Bash 腳本
    ├── processor.ts     # TypeScript 腳本
    └── utils/
        └── common.py    # 工具模組
```

腳本的執行遵循以下規則：

- **入口點**：預設為 `scripts/main.py`、`scripts/main.sh` 或 `scripts/main.ts`
- **參數傳遞**：通過環境變數或 JSON stdin 傳入
- **結果回傳**：通過 stdout 輸出（JSON 或純文字格式）
- **錯誤處理**：非零退出碼表示執行失敗，stderr 會被記錄
- **權限**：腳本需有執行權限（`chmod +x`）
- **沙盒**：腳本在受限環境中執行，依安全策略而定

### 4.3 references/ 目錄

`references/` 目錄存放技能的參考資料，這些資料會在 LLM 推理時被作為額外上下文載入：

```
my-skill/
├── SKILL.md
└── references/
    ├── api-docs.md       # API 文件摘要
    ├── examples.json     # 額外的使用範例
    └── domain-knowledge/ # 領域知識
        ├── glossary.md   # 術語表
        └── rules.md      # 業務規則
```

`references/` 中的文件**不是在啟動時載入的**——它們是按需載入的。當 LLM 決定使用某個技能時，如果需要更多上下文，它可以請求載入 references 中的特定文件。這個設計節省了寶貴的上下文窗口空間。

### 4.4 tests/ 目錄

`tests/` 目錄包含技能的測試案例：

```
my-skill/
├── SKILL.md
└── tests/
    ├── test_basic.yaml    # 基本功能測試
    ├── test_edge.yaml     # 邊界情況測試
    └── test_security.yaml # 安全性測試
```

測試文件使用 YAML 格式定義測試案例：

```yaml
# tests/test_basic.yaml
tests:
  - name: "基本天氣查詢"
    input:
      user_message: "台北天氣如何？"
    expected:
      skill_used: "weather"
      parameters:
        location: "台北"
      output_contains:
        - "溫度"
        - "台北"

  - name: "缺少地點應該詢問"
    input:
      user_message: "天氣如何？"
    expected:
      should_ask_user: true
      ask_contains: "哪個城市"
```

### 4.5 assets/ 目錄

`assets/` 目錄存放技能使用的靜態資源：

```
my-skill/
├── SKILL.md
└── assets/
    ├── icon.png           # 技能圖示（用於 ClawHub）
    ├── screenshot.png     # 使用截圖
    └── templates/
        └── report.html    # HTML 模板
```

### 4.6 完整目錄範例

以下是一個功能完整的技能目錄結構：

```
weather-skill/
├── SKILL.md                    # 技能定義（必要）
├── scripts/
│   ├── main.py                 # 主要腳本
│   ├── requirements.txt        # Python 依賴
│   └── utils/
│       ├── __init__.py
│       ├── api_client.py       # API 客戶端
│       └── formatters.py       # 格式化工具
├── references/
│   ├── weather-api-docs.md     # API 文件
│   └── weather-codes.json      # 天氣代碼對照表
├── tests/
│   ├── test_basic.yaml         # 基本測試
│   ├── test_edge_cases.yaml    # 邊界測試
│   └── fixtures/
│       └── sample_response.json # 模擬 API 回應
├── assets/
│   ├── icon.png                # 技能圖示
│   └── screenshots/
│       ├── basic-usage.png     # 基本使用截圖
│       └── advanced-usage.png  # 進階使用截圖
├── LICENSE                     # 授權條款
├── CHANGELOG.md                # 變更記錄
└── README.md                   # 附加說明（選擇性）
```

---

## 5. YAML Frontmatter 格式

SKILL.md 的 YAML Frontmatter 是技能的「身份證」，包含所有結構化的元數據。它被夾在兩組三個破折號 (`---`) 之間，位於文件的最頂端。

### 5.1 基本欄位

```yaml
---
name: weather-query
description: 查詢全球各城市的即時天氣資訊，包括溫度、濕度、風速和天氣預報
version: 1.2.0
author: openclaw-community
tags:
  - weather
  - utility
  - api
---
```

各欄位說明：

| 欄位 | 類型 | 必要 | 說明 |
|------|------|------|------|
| `name` | string | ✅ | 技能的唯一識別名稱，使用 kebab-case |
| `description` | string | ✅ | 一到兩句話的簡短描述，這是 LLM 在 Level 1 載入時看到的唯一描述 |
| `version` | string | ✅ | 遵循 Semantic Versioning（主版本.次版本.修訂版本） |
| `author` | string | ✅ | 作者或組織名稱 |
| `tags` | list | 建議 | 標籤列表，用於 ClawHub 搜尋和分類 |

**`name` 的命名規範**：

- 只能包含小寫英文字母、數字和連字號（`-`）
- 不能以數字或連字號開頭
- 建議長度在 3 到 50 個字元之間
- 應該具有描述性：`weather-query` 好，`wq` 不好
- 全域唯一（在 ClawHub 中不能重複）

**`description` 的撰寫要點**：

- 這是 LLM 決定是否使用你的技能的第一印象
- 要精確——描述技能**做什麼**，不是**怎麼做**
- 包含關鍵詞，讓 LLM 能正確匹配使用者意圖
- 避免過於技術化的描述

### 5.2 metadata.openclaw 欄位

`metadata.openclaw` 是 OpenClaw 特有的配置區段，定義技能在系統中的行為：

```yaml
metadata:
  openclaw:
    # 安全策略
    controlPolicy: confirm    # allow | confirm | deny
    
    # 環境需求
    requires:
      bins:
        - python3
        - curl
      env:
        - WEATHER_API_KEY
      skills:
        - location-resolver
    
    # 執行配置
    timeout: 30               # 超時秒數
    retries: 2                # 重試次數
    cache: 300                # 快取秒數（0 = 不快取）
    
    # 分類
    category: utility         # utility | productivity | social | dev-tools | ...
    
    # 觸發設定
    triggers:
      keywords:
        - 天氣
        - 氣溫
        - 下雨
      patterns:
        - ".*天氣.*"
        - ".*會不會下雨.*"
    
    # 輸出配置
    output:
      format: text            # text | json | markdown
      maxLength: 500          # 最大輸出字數
```

各欄位詳解：

**`controlPolicy`**：決定技能被調用時的安全等級

- `allow`：直接執行，不需使用者確認（適用於低風險操作）
- `confirm`：執行前需要使用者確認（預設值，適用於中等風險操作）
- `deny`：停用此技能（用於暫時關閉某個技能）

**`requires`**：環境門控，技能運作所需的外部條件

- `bins`：所需的可執行文件列表，OpenClaw 會在 PATH 中搜尋
- `env`：所需的環境變數列表，缺少時技能會被標記為 `disabled`
- `skills`：所依賴的其他技能列表

**`timeout`**：腳本執行的最長時間（秒），超時後自動中止並回報錯誤

**`retries`**：腳本執行失敗時的自動重試次數

**`cache`**：相同參數的快取有效時間（秒），避免重複呼叫外部 API

**`triggers`**：幫助 LLM 更精確地匹配使用者意圖的觸發條件

- `keywords`：關鍵詞列表，使用者訊息中包含這些詞時提高匹配機率
- `patterns`：正則表達式列表，進行更精確的模式匹配

### 5.3 requires 環境需求

`requires` 是一個非常重要的機制，它確保技能只在環境條件滿足時才會被啟用。這避免了使用者嘗試使用無法執行的技能時遇到的混亂錯誤。

```yaml
requires:
  # 二進位工具
  bins:
    - python3          # Python 3 直譯器
    - ffmpeg           # 影音處理工具
    - git              # Git 版本控制
  
  # 環境變數
  env:
    - OPENAI_API_KEY   # OpenAI API 金鑰
    - DATABASE_URL     # 資料庫連線字串
  
  # 依賴的其他技能
  skills:
    - file-manager     # 文件管理技能
    - web-search       # 網頁搜尋技能
  
  # 作業系統限制
  os:
    - linux
    - darwin           # macOS
  
  # 最低版本要求
  minVersion: "0.5.0"  # 最低 OpenClaw 版本
```

環境檢查的過程是這樣的：

```
startup → 檢查 bins → 檢查 env → 檢查 skills → 檢查 os
    │         │           │            │            │
    │     缺 python3   缺 API_KEY   缺 web-search   OS 不符
    │         │           │            │            │
    │         ▼           ▼            ▼            ▼
    │      warn       warn          warn         warn
    │         │           │            │            │
    │         └───────────┴────────────┴────────────┘
    │                          │
    │                          ▼
    │                   技能標記為 disabled
    │                   記錄原因到日誌
    ▼
  全部通過 → 技能標記為 active
```

### 5.4 完整 Frontmatter 範例

以下是一個完整的 YAML Frontmatter 範例：

```yaml
---
name: github-issue-manager
description: >-
  管理 GitHub Issues：建立、搜尋、更新、關閉 issue，
  以及管理標籤和指派人員。支援多個 repository 的操作。
version: 2.1.0
author: openclaw-community
tags:
  - github
  - issue-tracking
  - dev-tools
  - productivity
metadata:
  openclaw:
    controlPolicy: confirm
    requires:
      bins:
        - gh
      env:
        - GITHUB_TOKEN
      skills: []
      os:
        - linux
        - darwin
        - win32
    timeout: 60
    retries: 1
    cache: 0
    category: dev-tools
    triggers:
      keywords:
        - issue
        - bug
        - feature request
        - github
      patterns:
        - ".*issue.*"
        - ".*建立.*bug.*"
    output:
      format: markdown
      maxLength: 2000
---
```

---

## 6. Markdown 正文格式

YAML Frontmatter 之後就是 Markdown 正文——這是技能描述的主體，也是 LLM 理解和使用技能的主要依據。正文有四個核心區段和若干可選區段。

### 6.1 Purpose 區段

`## Purpose` 告訴 LLM 這個技能**存在的目的**和**適用場景**。這是最重要的區段之一，因為它直接影響 LLM 是否會選擇使用這個技能。

```markdown
## Purpose

這個技能讓你能夠查詢全球任何城市的即時天氣資訊。它適用於以下場景：

- 使用者想知道某個城市的當前天氣狀況
- 使用者在計劃旅行，需要知道目的地的天氣預報
- 使用者想了解是否需要帶雨具或特定衣物
- 使用者需要天氣數據來做決策（如戶外活動安排）

這個技能**不適用於**：

- 歷史天氣數據查詢（超過 7 天前的）
- 天氣警報或災害預警（請使用 emergency-alerts 技能）
- 非地球的天氣（如火星天氣）
```

撰寫要點：

- **正面描述**：先說能做什麼
- **負面描述**：再說不能做什麼，避免 LLM 誤用
- **場景化**：用具體的使用場景幫助 LLM 理解
- **交叉參考**：指向更適合的技能，幫助 LLM 做正確的選擇

### 6.2 Instructions 區段

`## Instructions` 告訴 LLM **如何使用**這個技能。這裡描述操作步驟、參數說明、以及各種使用方式。

```markdown
## Instructions

### 基本查詢

要查詢一個城市的天氣，請提供以下資訊：

- **location**（必填）：城市名稱，支援中文或英文。例如："台北"、"Tokyo"、"New York"
- **date**（選填）：查詢日期，預設為今天。支援 "today"、"tomorrow"，或具體日期如 "2024-03-15"
- **units**（選填）：溫度單位，"celsius"（預設）或 "fahrenheit"

### 執行腳本

呼叫 `scripts/main.py`，傳入參數格式：

```json
{
  "location": "台北",
  "date": "today",
  "units": "celsius"
}
```

### 回應格式

腳本會回傳以下 JSON：

```json
{
  "location": "台北市",
  "date": "2024-03-15",
  "temperature": 28,
  "humidity": 75,
  "description": "多雲",
  "forecast": [
    { "date": "2024-03-16", "temp_high": 30, "temp_low": 22, "desc": "晴" },
    { "date": "2024-03-17", "temp_high": 25, "temp_low": 18, "desc": "雨" }
  ]
}
```

請用友善的方式向使用者呈現這些資訊。如果使用者在規劃行程，
主動提供未來幾天的預報。
```

### 6.3 Constraints 區段

`## Constraints` 定義技能的**限制和安全規則**。這個區段對於防止技能被濫用至關重要。

```markdown
## Constraints

1. **頻率限制**：天氣 API 有每分鐘 60 次的請求限制。如果使用者短時間內重複查詢相同城市，使用快取結果。

2. **地點驗證**：如果使用者提供的地點名稱模糊（如 "Springfield"，美國有多個同名城市），主動詢問使用者要查詢的具體地點。

3. **日期範圍**：只支援今天到未來 7 天的預報。如果使用者詢問更遠的日期，禮貌地告知限制。

4. **不要主動查詢**：除非使用者明確要求天氣資訊，否則不要主動使用此技能。不要在使用者提到某個城市時就自動查天氣。

5. **錯誤處理**：如果 API 回傳錯誤，告訴使用者「目前無法取得天氣資訊」，不要猜測天氣。

6. **隱私**：不要記錄或分享使用者查詢的地點資訊。
```

### 6.4 Examples 區段

`## Examples` 提供具體的使用範例，這對 LLM 的 few-shot learning 至關重要。好的範例能大幅提高技能被正確使用的機率。

```markdown
## Examples

### 範例一：基本天氣查詢

**使用者**：台北今天天氣怎麼樣？

**思考**：使用者想知道台北的即時天氣。

**行動**：呼叫 weather 技能
```json
{ "location": "台北", "date": "today", "units": "celsius" }
```

**結果**：
```json
{ "temperature": 28, "humidity": 75, "description": "多雲" }
```

**回應**：台北今天多雲，氣溫 28°C，濕度 75%。建議帶把傘以防午後雷陣雨。

### 範例二：旅行天氣查詢

**使用者**：我下週末要去東京，天氣如何？

**思考**：使用者要去東京旅行，需要知道未來幾天的天氣預報。

**行動**：呼叫 weather 技能
```json
{ "location": "Tokyo", "date": "2024-03-22", "units": "celsius" }
```

**回應**：東京下週末（3/22-3/23）預計天氣晴朗，氣溫約 15-20°C。
建議帶一件輕薄外套，早晚溫差較大。

### 範例三：模糊地點處理

**使用者**：Portland 天氣如何？

**回應**：美國有兩個主要的 Portland——俄勒岡州和緬因州。
你想查哪一個呢？
```

### 6.5 可選區段

除了四個核心區段外，你還可以加入以下可選區段：

**`## Notes`**：額外的注意事項或開發者備忘錄

```markdown
## Notes

- 天氣 API 使用的是 OpenWeatherMap，免費方案每天有 1000 次請求限制
- 中文城市名稱的轉換使用內建的城市名稱映射表
- 如果遇到 API 限速，系統會自動降級使用快取數據
```

**`## Changelog`**：技能的更新記錄

```markdown
## Changelog

### v1.2.0 (2024-03-01)
- 新增 7 天天氣預報功能
- 支援華氏溫度

### v1.1.0 (2024-02-15)
- 修復中文城市名稱識別問題

### v1.0.0 (2024-02-01)
- 初始發布
```

---

## 7. 技能分類與區別

OpenClaw 的技能生態系統分為三大類別，每種類別有不同的特性、管理方式和適用場景。

### 7.1 內建技能（Built-in Skills）

內建技能是隨 OpenClaw 安裝一起提供的基礎技能，它們提供了最核心的功能。

**特性**：

- 隨 OpenClaw 核心安裝自動部署
- 經過完整測試和安全審核
- 版本與 OpenClaw 主版本同步
- 不可修改（但可以用自定義技能覆蓋）

**常見的內建技能**：

| 技能名稱 | 功能描述 |
|----------|----------|
| `web-search` | 網頁搜尋 |
| `file-manager` | 文件讀寫操作 |
| `calculator` | 數學計算 |
| `code-runner` | 程式碼執行 |
| `reminder` | 提醒與排程 |
| `knowledge-base` | 知識庫查詢 |

**目錄位置**：`<openclaw-install>/skills/`

### 7.2 社群技能（Community Skills）

社群技能是由 OpenClaw 社群貢獻並通過 ClawHub 市集發布的技能。

**特性**：

- 通過 ClawHub 市集安裝
- 由社群維護，品質和更新頻率不一
- 經過基本的安全審核
- 可以通過 `openclaw skills install` 命令安裝

**安裝方式**：

```bash
# 搜尋技能
openclaw skills search weather

# 安裝技能
openclaw skills install weather-advanced

# 更新技能
openclaw skills update weather-advanced

# 移除技能
openclaw skills remove weather-advanced
```

**目錄位置**：`~/.openclaw/community-skills/`

### 7.3 自定義技能（Custom Skills）

自定義技能是使用者自己開發的技能，完全由使用者控制。

**特性**：

- 使用者完全控制
- 不需要發布到 ClawHub
- 優先級最高（會覆蓋同名的內建/社群技能）
- 適合個人化需求和企業內部使用

**建立方式**：

```bash
# 使用樣板建立新技能
openclaw skills create my-custom-skill

# 或手動建立
mkdir -p ~/.openclaw/skills/my-custom-skill
touch ~/.openclaw/skills/my-custom-skill/SKILL.md
```

**目錄位置**：

- 使用者級：`~/.openclaw/skills/`
- 專案級：`./skills/`

### 7.4 三者比較表

| 特性 | 內建技能 | 社群技能 | 自定義技能 |
|------|---------|---------|-----------|
| 安裝方式 | 自動隨核心安裝 | `openclaw skills install` | 手動建立 |
| 更新方式 | 隨核心更新 | `openclaw skills update` | 手動更新 |
| 品質保證 | OpenClaw 團隊 | 社群審核 | 使用者自行負責 |
| 安全審核 | 完整 | 基本 | 無 |
| 優先級 | 最低 | 中 | 最高 |
| 可修改 | 否 | 可 fork | 完全控制 |
| 目錄 | `<install>/skills/` | `~/.openclaw/community-skills/` | `~/.openclaw/skills/` 或 `./skills/` |
| 適用場景 | 基礎功能 | 常見需求 | 個人化/企業 |

---

## 8. ClawHub 市集

### 8.1 市集概覽

ClawHub 是 OpenClaw 生態系統的技能市集，類似於 npm（Node.js）、PyPI（Python）或 VS Code Extensions Marketplace。它是一個集中式的平台，讓開發者可以分享、發現和安裝 OpenClaw 技能。

ClawHub 的核心理念是「**讓 AI 能力可以像積木一樣被組合**」。每個技能都是一塊積木，透過 ClawHub，使用者可以輕鬆地為自己的 AI 助手添加各種能力。

### 8.2 運作方式

ClawHub 的運作流程如下：

```
開發者                        ClawHub                      使用者
  │                            │                            │
  │  1. 開發技能               │                            │
  │  2. 打包 & 提交            │                            │
  │ ──────────────────────────▶│                            │
  │                            │  3. 自動化審核              │
  │                            │  4. 人工審核（可選）        │
  │                            │  5. 發布上架                │
  │                            │──────────────────────────▶│
  │                            │                            │  6. 搜尋 & 發現
  │                            │                            │  7. 安裝
  │                            │◀──────────────────────────│
  │                            │                            │  8. 使用
  │  9. 收到反饋               │                            │
  │ ◀──────────────────────────│                            │
  │  10. 更新版本              │                            │
  │ ──────────────────────────▶│                            │
  │                            │  11. 推送更新               │
  │                            │──────────────────────────▶│
```

**技能包格式**：

技能以 `.claw` 格式打包（本質上是一個結構化的 tar.gz）：

```bash
# 打包技能
openclaw skills pack ./my-skill

# 輸出：my-skill-1.0.0.claw
```

`.claw` 文件的結構：

```
my-skill-1.0.0.claw
├── manifest.json     # 打包元數據
├── SKILL.md          # 技能定義
├── scripts/          # 腳本（如果有）
├── references/       # 參考資料（如果有）
├── tests/            # 測試（如果有）
├── assets/           # 資源（如果有）
├── LICENSE           # 授權條款
└── checksums.sha256  # 完整性校驗
```

### 8.3 搜尋與安裝

```bash
# 搜尋技能
openclaw skills search "weather"
# 結果：
# weather-basic      v1.0.0  ⭐ 4.5  ↓ 12.3k  基本天氣查詢
# weather-advanced   v2.1.0  ⭐ 4.8  ↓ 45.6k  進階天氣資訊與預報
# weather-alerts     v1.2.0  ⭐ 4.2  ↓ 5.8k   天氣警報通知

# 按標籤搜尋
openclaw skills search --tag productivity

# 查看技能詳情
openclaw skills info weather-advanced

# 安裝技能
openclaw skills install weather-advanced

# 安裝特定版本
openclaw skills install weather-advanced@2.0.0

# 批量安裝（使用 skills.yaml）
openclaw skills install --from skills.yaml
```

### 8.4 品質保證機制

ClawHub 對提交的技能進行多層品質保證：

**自動化檢查**：

1. **結構驗證**：檢查 SKILL.md 格式是否正確、必要欄位是否完整
2. **安全掃描**：檢查腳本中是否有危險操作（如刪除系統文件、讀取敏感目錄）
3. **依賴檢查**：確認聲明的依賴是否存在且版本相容
4. **測試執行**：運行 `tests/` 目錄中的測試案例
5. **大小限制**：技能包不得超過 10MB

**人工審核**（針對高風險技能）：

- 包含系統級操作的技能
- 需要敏感權限的技能
- 被標記或投訴的技能

**社群反饋**：

- 星級評分（1-5 星）
- 使用者評論
- 下載次數和趨勢
- 問題回報追蹤

### 8.5 商業模式

ClawHub 採用開放市集模式：

- **免費技能**：大多數社群技能免費提供
- **付費技能**：開發者可以設定價格（ClawHub 收取一定比例的分潤）
- **捐贈**：使用者可以對喜歡的免費技能進行捐贈
- **企業方案**：企業可以建立私有 ClawHub 實例，管理內部技能

---

## 9. 完整的 SKILL.md 範例

### 9.1 簡單範例：天氣查詢技能

```markdown
---
name: weather-query
description: 查詢全球城市的即時天氣資訊，包括溫度、濕度和天氣預報
version: 1.0.0
author: openclaw-community
tags:
  - weather
  - utility
metadata:
  openclaw:
    controlPolicy: allow
    requires:
      bins:
        - python3
      env:
        - WEATHER_API_KEY
    timeout: 15
    cache: 300
    category: utility
    triggers:
      keywords:
        - 天氣
        - 氣溫
        - 下雨
---

## Purpose

查詢全球城市的即時天氣資訊。適用於使用者想知道某個城市的天氣狀況，
或是在規劃行程時需要了解天氣預報的場景。

不適用於歷史天氣查詢（7天以上）或災害預警。

## Instructions

使用 `scripts/main.py` 查詢天氣 API。

**參數**：
- `location`（必填）：城市名稱，中英文皆可
- `date`（選填）：日期，預設今天，支援 "today"、"tomorrow" 或 YYYY-MM-DD
- `units`（選填）：溫度單位，"celsius"（預設）或 "fahrenheit"

**呼叫方式**：
```json
{ "location": "台北", "date": "today", "units": "celsius" }
```

**回傳格式**：
```json
{
  "location": "台北市",
  "temperature": 28,
  "humidity": 75,
  "description": "多雲",
  "forecast": [...]
}
```

## Constraints

1. 不要在使用者沒有明確要求時主動查天氣
2. 地點模糊時請使用者確認
3. 只支援未來 7 天的預報
4. API 錯誤時說明無法取得，不要猜測

## Examples

### 基本查詢
**使用者**：台北天氣？
**行動**：weather-query(location="台北")
**回應**：台北目前多雲，28°C，濕度 75%。
```

### 9.2 中等範例：Markdown 筆記管理技能

```markdown
---
name: note-manager
description: 管理 Markdown 格式的個人筆記，支援建立、搜尋、編輯、標籤管理和匯出功能
version: 1.5.0
author: openclaw-community
tags:
  - notes
  - productivity
  - markdown
metadata:
  openclaw:
    controlPolicy: confirm
    requires:
      bins:
        - python3
      env: []
    timeout: 30
    cache: 0
    category: productivity
    triggers:
      keywords:
        - 筆記
        - 記錄
        - 備忘
        - 寫下
---

## Purpose

這個技能管理使用者的 Markdown 筆記庫。適用於：

- 快速建立筆記、靈感記錄、待辦清單
- 搜尋過去的筆記
- 編輯和更新現有筆記
- 用標籤整理筆記
- 匯出筆記為 PDF 或 HTML

筆記儲存在 `~/.openclaw/data/notes/` 目錄中。

## Instructions

### 建立筆記
```json
{
  "action": "create",
  "title": "筆記標題",
  "content": "筆記內容（Markdown 格式）",
  "tags": ["標籤1", "標籤2"]
}
```

### 搜尋筆記
```json
{
  "action": "search",
  "query": "搜尋關鍵字",
  "tags": ["可選標籤過濾"],
  "date_range": {
    "from": "2024-01-01",
    "to": "2024-03-15"
  }
}
```

### 編輯筆記
```json
{
  "action": "edit",
  "note_id": "筆記ID",
  "content": "更新後的內容",
  "append": false
}
```

### 列出筆記
```json
{
  "action": "list",
  "limit": 10,
  "sort": "updated_at",
  "tags": ["可選標籤過濾"]
}
```

## Constraints

1. 建立筆記前確認標題，避免重複
2. 刪除筆記時必須經過使用者確認
3. 筆記內容不超過 10,000 字
4. 敏感資訊（密碼、金鑰）不應存入筆記
5. 搜尋結果最多回傳 20 筆

## Examples

### 快速記錄
**使用者**：幫我記一下，明天下午三點跟 John 開會
**行動**：note-manager(action="create", title="會議提醒", 
  content="明天下午三點跟 John 開會", tags=["meeting"])
**回應**：已建立筆記「會議提醒」，標籤：meeting。

### 搜尋筆記
**使用者**：上週我記了什麼關於專案的東西？
**行動**：note-manager(action="search", query="專案", 
  date_range={"from": "上週一", "to": "今天"})
**回應**：找到 3 筆相關筆記：
1. 「專案進度更新」- 3 天前
2. 「專案架構討論」- 5 天前
3. 「專案需求整理」- 6 天前
要看哪一篇的內容嗎？
```

### 9.3 複雜範例：GitHub 專案管理技能

```markdown
---
name: github-project-manager
description: >-
  全方位的 GitHub 專案管理工具。管理 Issues、Pull Requests、
  Projects、Actions，以及 Repository 設定。支援多 repo 操作。
version: 2.0.0
author: openclaw-labs
tags:
  - github
  - dev-tools
  - project-management
  - automation
metadata:
  openclaw:
    controlPolicy: confirm
    requires:
      bins:
        - gh
        - git
      env:
        - GITHUB_TOKEN
      skills:
        - file-manager
    timeout: 120
    retries: 2
    cache: 60
    category: dev-tools
    triggers:
      keywords:
        - github
        - issue
        - PR
        - pull request
        - repository
---

## Purpose

這是一個全方位的 GitHub 專案管理技能，適用於：

- 管理 Issues（建立、查詢、更新、關閉、標籤管理）
- 管理 Pull Requests（查看、審核、合併）
- 查看 CI/CD 狀態（GitHub Actions）
- Repository 管理（分支、標籤、設定）
- 專案看板管理（GitHub Projects）

此技能使用 GitHub CLI（`gh`）作為底層工具，因此需要安裝 
`gh` 並完成認證。

不適用於：GitLab、Bitbucket 或其他非 GitHub 平台。

## Instructions

### Issue 管理

#### 建立 Issue
```json
{
  "action": "issue.create",
  "repo": "owner/repo",
  "title": "Issue 標題",
  "body": "Issue 內容（支援 Markdown）",
  "labels": ["bug", "priority:high"],
  "assignees": ["username"],
  "milestone": "v2.0"
}
```

#### 搜尋 Issues
```json
{
  "action": "issue.search",
  "repo": "owner/repo",
  "query": "搜尋關鍵字",
  "state": "open",
  "labels": ["bug"],
  "sort": "updated",
  "limit": 10
}
```

#### 更新 Issue
```json
{
  "action": "issue.update",
  "repo": "owner/repo",
  "number": 42,
  "state": "closed",
  "comment": "已修復於 PR #55"
}
```

### Pull Request 管理

#### 查看 PR
```json
{
  "action": "pr.get",
  "repo": "owner/repo",
  "number": 55
}
```

#### 審核 PR
```json
{
  "action": "pr.review",
  "repo": "owner/repo",
  "number": 55,
  "event": "APPROVE",
  "body": "LGTM! 程式碼品質很好。"
}
```

### GitHub Actions

#### 查看工作流程狀態
```json
{
  "action": "actions.status",
  "repo": "owner/repo",
  "workflow": "ci.yml"
}
```

### Repository 管理

#### 建立分支
```json
{
  "action": "repo.branch",
  "repo": "owner/repo",
  "name": "feature/new-feature",
  "from": "main"
}
```

## Constraints

1. 所有修改操作（建立、更新、刪除、合併）需要使用者確認
2. 不要自動合併 Pull Request，除非使用者明確要求
3. 操作預設在使用者上次提到的 repository 上執行
4. 如果未指定 repo，詢問使用者
5. 敏感操作（刪除 repo、force push）需要二次確認
6. 遵守 GitHub API 速率限制

## Examples

### 建立 Bug Report
**使用者**：在 my-project 建一個 bug，登入頁面會崩潰
**行動**：github-project-manager(action="issue.create",
  repo="me/my-project", title="登入頁面崩潰",
  body="使用者回報登入頁面會崩潰...", labels=["bug"])
**確認**：即將在 me/my-project 建立 Issue「登入頁面崩潰」，確認嗎？
**使用者**：確認
**回應**：已建立 Issue #123「登入頁面崩潰」。

### 查看專案狀態
**使用者**：my-project 現在有多少 open issues？
**行動**：github-project-manager(action="issue.search",
  repo="me/my-project", state="open")
**回應**：my-project 目前有 15 個 open issues：
- 5 個標記為 bug
- 7 個標記為 feature
- 3 個標記為 documentation

## Notes

- 使用 `gh auth status` 確認認證狀態
- 支援 GitHub Enterprise（配置 GH_HOST 環境變數）
- 大量操作建議使用 GitHub Actions 自動化
```

---

## 10. 設計模式與最佳實踐

在開發 OpenClaw Skills 時，以下設計模式和最佳實踐能幫助你建立高品質、易維護的技能：

**1. 單一職責原則**

每個技能應該專注於做好一件事。與其建立一個「萬能」技能，不如建立多個專精的技能，讓 ReAct 引擎來組合它們。

```
❌ all-in-one-tool（天氣 + 翻譯 + 計算 + 筆記）
✅ weather-query + translator + calculator + note-manager
```

**2. 漸進式揭露**

在 SKILL.md 中，從最常用的功能開始描述，逐漸深入到進階功能。這與 LLM 的動態載入策略配合——常用功能在 Level 2 就被載入，進階功能只在需要時載入。

**3. 防禦式描述**

總是描述技能「不應該做什麼」，不只是「應該做什麼」。LLM 有時會「過度熱心」，明確的約束能防止誤用。

**4. 豐富的範例**

至少提供 3 個範例：一個基本用法、一個進階用法、一個邊界/錯誤處理情況。範例是 LLM few-shot learning 的關鍵。

**5. 優雅降級**

當外部依賴不可用時，技能應該提供有意義的錯誤訊息，而不是無聲失敗。在 SKILL.md 的 Instructions 中描述降級策略。

**6. 版本相容性**

使用 Semantic Versioning，在 CHANGELOG 中清楚記錄破壞性變更。使用 `metadata.openclaw.minVersion` 確保與 OpenClaw 核心的相容性。

**7. 測試覆蓋**

為每個技能編寫測試案例（`tests/` 目錄），覆蓋：正常流程、邊界情況、錯誤處理、安全約束。

---

## 11. 總結與展望

OpenClaw 的 Skills 系統代表了 AI 能力定義的一個新方向——**用文件取代程式碼，用自然語言取代形式化描述**。這個設計哲學建立在對 LLM 本質的深刻理解之上：LLM 最擅長理解的就是自然語言。

通過本章，我們了解了：

- **設計哲學**：為什麼用 Markdown 定義 AI 能力是合理的
- **生命週期**：技能如何被發現、註冊、調用和回傳
- **目錄結構**：一個技能包含哪些文件和目錄
- **文件格式**：YAML Frontmatter 和 Markdown 正文的寫法
- **技能分類**：內建、社群、自定義三種技能的區別
- **ClawHub 市集**：生態系統的運作方式

在下一章，我們將深入 SKILL.md 的撰寫技巧，從零開始教你寫出一個高品質的技能定義文件。

> **關鍵要點**：Skills 系統的精髓不在於技術複雜度，而在於**設計簡潔性**。一個好的 SKILL.md 就像一份好的工作說明書——清晰、完整、無歧義。如果你能寫好一份技術文件，你就能建立一個優秀的 OpenClaw 技能。
