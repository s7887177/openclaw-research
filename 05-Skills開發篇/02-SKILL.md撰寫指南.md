# SKILL.md 撰寫完全指南

## 目錄

- [1. 引言](#1-引言)
- [2. 從零開始：你的第一個 SKILL.md](#2-從零開始你的第一個-skillmd)
  - [2.1 最小可行技能](#21-最小可行技能)
  - [2.2 建立目錄與文件](#22-建立目錄與文件)
  - [2.3 驗證技能](#23-驗證技能)
- [3. YAML Frontmatter 欄位詳解](#3-yaml-frontmatter-欄位詳解)
  - [3.1 name——技能的身份證號碼](#31-name技能的身份證號碼)
  - [3.2 description——第一印象決定一切](#32-description第一印象決定一切)
  - [3.3 version——語意化版本控制](#33-version語意化版本控制)
  - [3.4 author——作者資訊](#34-author作者資訊)
  - [3.5 tags——標籤與分類](#35-tags標籤與分類)
  - [3.6 metadata.openclaw——系統配置](#36-metadataopenclaw系統配置)
  - [3.7 完整欄位參考表](#37-完整欄位參考表)
- [4. Markdown 正文寫作技巧](#4-markdown-正文寫作技巧)
  - [4.1 Purpose：讓 LLM 知道何時選你](#41-purpose讓-llm-知道何時選你)
  - [4.2 Instructions：精確的操作手冊](#42-instructions精確的操作手冊)
  - [4.3 Constraints：不可踰越的紅線](#43-constraints不可踰越的紅線)
  - [4.4 Examples：最有效的教學方式](#44-examples最有效的教學方式)
- [5. 讓 LLM 正確理解你的工具](#5-讓-llm-正確理解你的工具)
  - [5.1 語意清晰度](#51-語意清晰度)
  - [5.2 參數描述的藝術](#52-參數描述的藝術)
  - [5.3 回傳值的說明](#53-回傳值的說明)
  - [5.4 情境引導](#54-情境引導)
- [6. 安全限制設定](#6-安全限制設定)
  - [6.1 controlPolicy 層級](#61-controlpolicy-層級)
  - [6.2 Constraints 安全規則](#62-constraints-安全規則)
  - [6.3 輸入驗證](#63-輸入驗證)
  - [6.4 輸出過濾](#64-輸出過濾)
- [7. 環境門控（requires）](#7-環境門控requires)
  - [7.1 requires.bins——二進位依賴](#71-requiresbins二進位依賴)
  - [7.2 requires.env——環境變數](#72-requiresenv環境變數)
  - [7.3 requires.skills——技能依賴](#73-requiresskills技能依賴)
  - [7.4 requires.os——作業系統限制](#74-requiresos作業系統限制)
  - [7.5 門控失敗的處理](#75-門控失敗的處理)
- [8. 完整範例集](#8-完整範例集)
  - [8.1 簡單範例：單位換算技能](#81-簡單範例單位換算技能)
  - [8.2 中等範例：待辦事項管理技能](#82-中等範例待辦事項管理技能)
  - [8.3 複雜範例：資料庫查詢助手技能](#83-複雜範例資料庫查詢助手技能)
- [9. 除錯技巧](#9-除錯技巧)
  - [9.1 技能未被發現](#91-技能未被發現)
  - [9.2 技能未被選用](#92-技能未被選用)
  - [9.3 技能被錯誤使用](#93-技能被錯誤使用)
  - [9.4 腳本執行失敗](#94-腳本執行失敗)
  - [9.5 除錯工具](#95-除錯工具)
- [10. 常見錯誤與解決方案](#10-常見錯誤與解決方案)
- [11. SKILL.md 撰寫檢查清單](#11-skillmd-撰寫檢查清單)
- [12. 總結](#12-總結)

---

## 1. 引言

撰寫一份好的 SKILL.md 是 OpenClaw 技能開發中最核心的能力。它不像寫程式那樣有編譯器幫你抓錯——SKILL.md 的品質直接決定了 LLM 能否正確理解和使用你的技能。一份寫得好的 SKILL.md 能讓 LLM 在正確的時機選擇你的技能，用正確的參數呼叫它，並向使用者呈現有意義的結果。而一份寫得差的 SKILL.md 則會導致各種問題：技能被忽略、參數錯誤、誤用場景等等。

本章將從零開始，逐步教你撰寫一個專業級的 SKILL.md。我們會深入每個欄位、每個區段的撰寫要點，提供豐富的範例，並分享在實踐中常見的錯誤和除錯技巧。無論你是第一次接觸技能開發，還是想提升現有技能的品質，這份指南都能幫到你。

---

## 2. 從零開始：你的第一個 SKILL.md

### 2.1 最小可行技能

讓我們從最簡單的技能開始——一個不需要任何腳本、純粹依靠 LLM 推理能力的技能。我們要做一個「表情符號翻譯器」，將使用者的文字翻譯成表情符號表達。

這是**最小可行的** SKILL.md：

```markdown
---
name: emoji-translator
description: 將文字訊息翻譯成表情符號表達方式
version: 1.0.0
author: your-name
tags:
  - fun
  - emoji
---

## Purpose

將使用者的文字訊息翻譯成表情符號。適用於使用者想用有趣的方式表達訊息的場景。

## Instructions

收到使用者的文字後，將每個關鍵概念替換為對應的表情符號。保留原文的意思，但盡量用表情符號表達。

## Constraints

1. 不要完全省略文字，保留必要的連接詞
2. 如果某個概念沒有適當的表情符號，保留原文
3. 只在使用者明確要求時使用此技能

## Examples

**使用者**：把這句話翻譯成表情符號：我今天很開心，因為天氣很好，我和朋友去公園野餐了。
**回應**：👤今天很😊，因為🌤️很好，👤和👫去🏞️🧺了。
```

就這麼簡單！這個 SKILL.md 只有約 30 行，但它包含了技能定義的所有核心要素。

### 2.2 建立目錄與文件

```bash
# 方法一：使用 OpenClaw CLI
openclaw skills create emoji-translator

# 方法二：手動建立
mkdir -p ~/.openclaw/skills/emoji-translator
cat > ~/.openclaw/skills/emoji-translator/SKILL.md << 'EOF'
---
name: emoji-translator
description: 將文字訊息翻譯成表情符號表達方式
version: 1.0.0
author: your-name
tags:
  - fun
  - emoji
metadata:
  openclaw:
    controlPolicy: allow
    requires:
      bins: []
      env: []
    timeout: 10
    category: fun
---

## Purpose

將使用者的文字訊息翻譯成表情符號。

## Instructions

收到使用者的文字後，將每個關鍵概念替換為對應的表情符號。

## Constraints

1. 保留必要的連接詞
2. 沒有適當的表情符號時保留原文
3. 只在使用者明確要求時使用

## Examples

**使用者**：把「我愛吃蘋果」翻譯成表情符號
**回應**：👤❤️🍎
EOF
```

### 2.3 驗證技能

建立完成後，驗證技能是否被正確識別：

```bash
# 重新載入技能
openclaw skills reload

# 列出技能，確認新技能出現
openclaw skills list

# 驗證技能結構
openclaw skills validate emoji-translator

# 輸出範例：
# ✅ SKILL.md found
# ✅ YAML frontmatter valid
# ✅ Required fields present (name, description, version, author)
# ✅ Markdown sections valid (Purpose, Instructions, Constraints, Examples)
# ✅ No script dependencies to check
# ✅ Skill "emoji-translator" is valid and ready to use!

# 測試技能
openclaw chat --test-skill emoji-translator
```

---

## 3. YAML Frontmatter 欄位詳解

YAML Frontmatter 是 SKILL.md 的「元數據層」，提供系統需要的結構化資訊。讓我們逐一深入每個欄位。

### 3.1 name——技能的身份證號碼

`name` 是技能在整個 OpenClaw 生態系統中的唯一識別符。一旦發布到 ClawHub，就不能更改。

**命名規範**：

```yaml
# ✅ 好的命名
name: weather-query
name: github-issue-manager
name: daily-standup-helper
name: code-review-assistant

# ❌ 壞的命名
name: wq              # 太短，無法理解含義
name: Weather Query    # 不能有空格和大寫
name: my_cool_tool     # 不能用底線
name: 123-tool         # 不能以數字開頭
name: -tool            # 不能以連字號開頭
name: tool-            # 不能以連字號結尾
```

**命名策略**：

- **功能導向**：`weather-query`、`file-converter`、`text-summarizer`
- **領域+動作**：`github-issue-create`、`jira-sprint-manage`
- **產品整合**：`slack-notifier`、`discord-bot-manager`

**名稱衝突**：

當多個目錄中有同名技能時，OpenClaw 按優先級選擇：

```
1. ./skills/weather-query/          ← 最高（專案級）
2. ~/.openclaw/skills/weather-query/ ← 高（使用者自定義）
3. ~/.openclaw/community-skills/weather-query/ ← 中（社群）
4. <install>/skills/weather-query/   ← 最低（內建）
```

這個機制允許你用自定義技能覆蓋內建或社群技能的行為，而不需要修改原始技能。

### 3.2 description——第一印象決定一切

`description` 是 LLM 在初始載入時看到的唯一技能描述（Level 1 載入）。它決定了 LLM 是否會進一步載入這個技能的完整定義。因此，它必須在一到兩句話內準確傳達技能的核心功能。

**撰寫原則**：

1. **精確**：描述技能做什麼，不是怎麼做
2. **完整**：涵蓋主要功能，但不要冗長
3. **包含關鍵詞**：使用者可能會用到的自然語言關鍵詞
4. **避免行銷用語**：這不是廣告文案，LLM 不會被「最強大」、「超級好用」打動

```yaml
# ✅ 好的 description
description: 查詢全球城市的即時天氣資訊，包括溫度、濕度、風速和未來7天預報
description: 管理 GitHub Issues 和 Pull Requests，支援建立、搜尋、更新和關閉操作
description: 將文字轉換成各國語言，支援超過100種語言的互譯

# ❌ 壞的 description
description: 天氣工具              # 太模糊
description: 一個非常好用的技能     # 無實質內容
description: >-                    # 太長，太技術化
  使用 OpenWeatherMap API v3.0 的 One Call 端點，
  通過 HTTP GET 請求獲取指定經緯度的天氣數據，
  支援 metric 和 imperial 單位系統...
```

**進階技巧：使用 YAML 多行字串**

當描述較長時，使用 YAML 的折疊字串語法（`>-`）來保持可讀性：

```yaml
description: >-
  管理 GitHub Issues 和 Pull Requests，
  支援建立、搜尋、更新、關閉和標籤管理。
  可同時操作多個 Repository。
```

`>-` 會把多行折疊成一行，並移除末尾換行符。

### 3.3 version——語意化版本控制

OpenClaw Skills 遵循 [Semantic Versioning (SemVer)](https://semver.org/) 規範：

```
主版本.次版本.修訂版本
 MAJOR . MINOR . PATCH
```

- **MAJOR**（主版本）：不相容的 API 變更（如移除功能、改變參數格式）
- **MINOR**（次版本）：向下相容的新功能（如新增參數、新增操作）
- **PATCH**（修訂版本）：向下相容的錯誤修正

```yaml
version: 1.0.0   # 初始穩定版本
version: 1.1.0   # 新增了天氣預報功能
version: 1.1.1   # 修復了中文城市名稱解析錯誤
version: 2.0.0   # 改變了回傳格式（破壞性變更）
```

**預發布版本**：

```yaml
version: 1.0.0-alpha.1   # Alpha 測試版
version: 1.0.0-beta.2    # Beta 測試版
version: 1.0.0-rc.1      # Release Candidate
```

### 3.4 author——作者資訊

```yaml
# 個人作者
author: john-doe

# 組織
author: openclaw-community

# 帶聯絡資訊
author: john-doe <john@example.com>
```

在 ClawHub 上，`author` 會連結到對應的開發者/組織頁面。確保使用與你的 ClawHub 帳號一致的名稱。

### 3.5 tags——標籤與分類

`tags` 用於 ClawHub 的搜尋和分類系統。好的標籤能讓你的技能更容易被發現。

```yaml
tags:
  - weather        # 功能類別
  - utility        # 技能類型
  - api            # 技術特徵
  - openweathermap # 具體服務
  - free           # 價格標記
```

**建議的標籤分類**：

| 類別 | 範例標籤 |
|------|---------|
| 功能 | `weather`, `translation`, `calendar`, `email` |
| 類型 | `utility`, `productivity`, `dev-tools`, `social`, `fun` |
| 技術 | `api`, `database`, `web-scraping`, `automation` |
| 平台 | `github`, `slack`, `discord`, `notion` |
| 語言 | `chinese`, `english`, `multilingual` |
| 成本 | `free`, `freemium`, `paid` |

### 3.6 metadata.openclaw——系統配置

這是 OpenClaw 特有的配置區段，我們來逐一深入每個子欄位：

#### controlPolicy

```yaml
metadata:
  openclaw:
    controlPolicy: allow   # allow | confirm | deny
```

**選擇指南**：

| 策略 | 適用場景 | 範例 |
|------|---------|------|
| `allow` | 只讀、無副作用的操作 | 天氣查詢、單位換算、翻譯 |
| `confirm` | 有副作用但可逆的操作 | 建立文件、發送訊息、修改設定 |
| `deny` | 暫時停用（或高危操作） | 刪除資料、系統管理 |

#### timeout

```yaml
timeout: 30  # 秒
```

超時設定取決於你的技能執行什麼操作：

- **純文字技能（無腳本）**：不需要設定，預設 10 秒
- **本地腳本**：10-30 秒通常足夠
- **API 呼叫**：15-60 秒（取決於 API 回應速度）
- **複雜操作**（如資料分析、文件生成）：60-300 秒

#### retries

```yaml
retries: 2  # 重試次數
```

重試機制在以下情況有用：

- 網路不穩定導致的 API 請求失敗
- 暫時性的外部服務錯誤（5xx）
- 超時（但要注意總時間不要過長）

注意：重試不會對用戶端錯誤（4xx）生效。

#### cache

```yaml
cache: 300  # 快取秒數
```

快取策略：

- `0`：不快取（適用於即時數據，如股價）
- `60`：1 分鐘（適用於頻繁變動的數據）
- `300`：5 分鐘（適用於天氣等中等變動頻率的數據）
- `3600`：1 小時（適用於低變動頻率的數據）
- `86400`：1 天（適用於靜態數據，如匯率）

#### triggers

```yaml
triggers:
  keywords:
    - 天氣
    - 氣溫
    - 下雨
    - 晴天
  patterns:
    - ".*天氣.*如何.*"
    - ".*會不會下雨.*"
    - ".*需要帶傘.*"
```

`triggers` 是一個**輔助**機制，幫助系統更快地匹配使用者意圖。它不是必要的——即使不設定 triggers，LLM 仍然能通過 `description` 和 `Purpose` 來判斷是否使用你的技能。但好的 triggers 可以提高匹配速度和準確度，特別是在技能數量很多的情況下。

**keywords vs patterns**：

- `keywords` 是簡單的關鍵詞匹配（不分大小寫）
- `patterns` 是正則表達式匹配，支援更複雜的模式

### 3.7 完整欄位參考表

| 欄位路徑 | 類型 | 必要 | 預設值 | 說明 |
|----------|------|------|--------|------|
| `name` | string | ✅ | - | 技能唯一名稱（kebab-case） |
| `description` | string | ✅ | - | 一到兩句話的簡短描述 |
| `version` | string | ✅ | - | 語意化版本號（SemVer） |
| `author` | string | ✅ | - | 作者或組織名稱 |
| `tags` | list | ⬜ | `[]` | 標籤列表 |
| `metadata.openclaw.controlPolicy` | string | ⬜ | `confirm` | 安全策略 |
| `metadata.openclaw.requires.bins` | list | ⬜ | `[]` | 需要的二進位工具 |
| `metadata.openclaw.requires.env` | list | ⬜ | `[]` | 需要的環境變數 |
| `metadata.openclaw.requires.skills` | list | ⬜ | `[]` | 依賴的其他技能 |
| `metadata.openclaw.requires.os` | list | ⬜ | `["*"]` | 支援的作業系統 |
| `metadata.openclaw.requires.minVersion` | string | ⬜ | `"0.0.0"` | 最低 OpenClaw 版本 |
| `metadata.openclaw.timeout` | number | ⬜ | `30` | 執行超時（秒） |
| `metadata.openclaw.retries` | number | ⬜ | `0` | 重試次數 |
| `metadata.openclaw.cache` | number | ⬜ | `0` | 快取時間（秒） |
| `metadata.openclaw.category` | string | ⬜ | `general` | 分類 |
| `metadata.openclaw.triggers.keywords` | list | ⬜ | `[]` | 關鍵詞列表 |
| `metadata.openclaw.triggers.patterns` | list | ⬜ | `[]` | 正則表達式列表 |
| `metadata.openclaw.output.format` | string | ⬜ | `text` | 輸出格式 |
| `metadata.openclaw.output.maxLength` | number | ⬜ | `1000` | 最大輸出長度 |

---

## 4. Markdown 正文寫作技巧

Markdown 正文是 SKILL.md 的「靈魂」——它決定了 LLM 如何理解和使用你的技能。寫好正文需要同時為兩種讀者考慮：人類開發者和 AI 模型。

### 4.1 Purpose：讓 LLM 知道何時選你

`## Purpose` 是技能的「自我介紹」。LLM 會用它來判斷：「面對當前使用者的請求，我應該使用這個技能嗎？」

**五要素框架**：

1. **做什麼**（What）：核心功能描述
2. **何時用**（When）：適用場景列表
3. **不做什麼**（What Not）：明確的排除範圍
4. **替代方案**（Alternative）：不適用時應該用什麼
5. **限制**（Limitation）：能力的邊界

```markdown
## Purpose

### 做什麼
這個技能管理你的日常待辦事項清單。支援建立、查詢、更新、
刪除和排序待辦事項。

### 何時用
- 使用者想要記錄一個新的待辦事項
- 使用者想查看今天/本週的待辦清單
- 使用者完成了某個任務，想要標記完成
- 使用者想要重新排列任務優先級

### 不做什麼
- 不處理日曆事件和會議排程（使用 calendar-manager）
- 不發送提醒通知（使用 reminder-service）
- 不與外部專案管理工具同步（如 Jira、Trello）

### 限制
- 最多支援 500 個活躍待辦事項
- 不支援子任務巢狀超過 3 層
- 離線模式下只能讀取，不能修改
```

**常見錯誤**：

```markdown
# ❌ 太模糊
## Purpose
這是一個很有用的工具。

# ❌ 太技術化
## Purpose
使用 SQLite3 資料庫引擎在本地儲存 TODO 記錄，
通過 Python 的 sqlite3 模組執行 CRUD 操作...

# ❌ 缺少排除範圍
## Purpose
管理待辦事項。可以建立、查詢、更新和刪除任務。
（LLM 可能會把日曆、提醒等請求也送到這裡）
```

### 4.2 Instructions：精確的操作手冊

`## Instructions` 是技能的「操作手冊」。它需要精確到 LLM 知道該傳什麼參數、以什麼格式傳、以及如何解讀回傳結果。

**結構化原則**：

1. **操作分組**：按功能將操作分成子區段
2. **參數表格化**：用表格清楚列出每個參數的名稱、類型、必要性和說明
3. **格式範例化**：每個操作都附上 JSON 格式的呼叫範例
4. **回傳說明**：清楚描述回傳值的格式和各欄位含義

```markdown
## Instructions

### 建立待辦事項

呼叫 `scripts/main.py` 執行操作。

**參數**：

| 參數 | 類型 | 必要 | 預設值 | 說明 |
|------|------|------|--------|------|
| `action` | string | ✅ | - | 操作類型，此處為 `"create"` |
| `title` | string | ✅ | - | 待辦事項標題（1-200字） |
| `description` | string | ⬜ | `""` | 詳細描述 |
| `priority` | string | ⬜ | `"medium"` | 優先級：`"high"`, `"medium"`, `"low"` |
| `due_date` | string | ⬜ | `null` | 到期日，格式：YYYY-MM-DD |
| `tags` | list | ⬜ | `[]` | 標籤列表 |

**呼叫格式**：

```json
{
  "action": "create",
  "title": "完成季報",
  "description": "整理第一季度的銷售數據",
  "priority": "high",
  "due_date": "2024-03-31",
  "tags": ["work", "report"]
}
```

**回傳格式**：

```json
{
  "success": true,
  "todo": {
    "id": "td_abc123",
    "title": "完成季報",
    "status": "pending",
    "priority": "high",
    "created_at": "2024-03-15T10:30:00Z"
  }
}
```

向使用者確認待辦事項已建立，包含標題和到期日。
```

**進階技巧：條件邏輯描述**

有時候操作的行為取決於條件，需要清楚描述邏輯：

```markdown
### 查詢待辦事項

如果使用者沒有指定時間範圍，預設顯示「今天」的待辦事項。
如果使用者說「全部」或「所有」，則顯示所有未完成的待辦事項。
如果結果超過 10 筆，先顯示前 10 筆並告知使用者還有更多。

**決策邏輯**：
- "今天有什麼要做的" → action: "list", filter: "today"
- "這週的待辦" → action: "list", filter: "this_week"  
- "所有未完成的任務" → action: "list", filter: "all", status: "pending"
- "高優先的事情" → action: "list", filter: "all", priority: "high"
```

### 4.3 Constraints：不可踰越的紅線

`## Constraints` 定義了技能使用的邊界和規則。好的約束能防止 LLM「過度熱心」或「創造性誤用」你的技能。

**約束分類**：

```markdown
## Constraints

### 安全約束
1. 不要在待辦事項中儲存密碼、API 金鑰或其他敏感資訊
2. 刪除操作必須經過使用者確認
3. 不要批量刪除超過 10 筆待辦事項

### 行為約束
4. 不要在使用者沒有明確要求時主動顯示待辦清單
5. 不要自動將未完成的任務標記為逾期
6. 不要修改其他使用者建立的待辦事項

### 技術約束
7. 標題長度不超過 200 個字元
8. 描述長度不超過 2000 個字元
9. 每個待辦最多 10 個標籤
10. 到期日不能是過去的日期（建立時）

### 品質約束
11. 如果使用者的描述模糊，先確認再建立
12. 建議但不強制使用者設定優先級
13. 重複的標題要提醒使用者
```

**撰寫要點**：

- **具體數字**：「不超過 200 字」比「不要太長」好
- **行為指引**：告訴 LLM 遇到特殊情況該怎麼做
- **正面表述**：「需要使用者確認」比「禁止未經確認的操作」更清晰
- **優先級**：最重要的安全約束放在最前面

### 4.4 Examples：最有效的教學方式

`## Examples` 是 SKILL.md 中最「有力」的區段。LLM 天生是 few-shot learner——給它看幾個範例，它就能學會模式。

**範例的結構**：

每個範例應該包含：

1. **使用者輸入**：模擬使用者的自然語言請求
2. **思考過程**（可選）：展示 LLM 應該如何推理
3. **行動**：實際的技能呼叫
4. **結果**：腳本的回傳值
5. **回應**：LLM 應該如何向使用者呈現結果

```markdown
## Examples

### 範例一：建立簡單的待辦事項

**使用者**：幫我記一下，明天要交報告

**思考**：使用者想建立一個待辦事項，標題是「交報告」，
到期日是明天。

**行動**：
```json
{
  "action": "create",
  "title": "交報告",
  "due_date": "2024-03-16",
  "priority": "high"
}
```

**結果**：
```json
{
  "success": true,
  "todo": { "id": "td_001", "title": "交報告" }
}
```

**回應**：好的，已幫你記下「交報告」，到期日設為明天（3/16）。
因為明天就要交了，我把優先級設為高。

---

### 範例二：模糊請求的處理

**使用者**：我有些事情要做

**思考**：使用者想建立待辦事項，但沒有給出具體內容。
根據約束，我需要先確認。

**回應**：好的，你想記錄什麼待辦事項呢？
可以告訴我具體要做什麼、截止日期和優先程度。

---

### 範例三：查詢和更新

**使用者**：我完成季報了，標記一下

**思考**：使用者完成了一個任務，我需要先找到這個任務，
然後標記為完成。

**行動1**：搜尋相關的待辦事項
```json
{
  "action": "search",
  "query": "季報",
  "status": "pending"
}
```

**結果1**：
```json
{
  "results": [
    { "id": "td_042", "title": "完成季報", "status": "pending" }
  ]
}
```

**行動2**：標記完成
```json
{
  "action": "update",
  "id": "td_042",
  "status": "completed"
}
```

**回應**：已將「完成季報」標記為完成 ✅ 做得好！
```

**範例數量建議**：

- **最少 3 個**：基本用法 + 進階用法 + 邊界/錯誤處理
- **建議 5-7 個**：覆蓋主要使用場景
- **不超過 10 個**：太多會增加上下文負擔

---

## 5. 讓 LLM 正確理解你的工具

### 5.1 語意清晰度

LLM 理解自然語言，但它也會被模糊的描述搞混。語意清晰度的核心原則是：**消除歧義**。

```markdown
# ❌ 有歧義的描述
action: "get" — 獲取資料

# ✅ 無歧義的描述
action: "get" — 根據 ID 獲取單一待辦事項的完整資訊。
  如果 ID 不存在，回傳 { "error": "not_found" }。
```

**消除歧義的技巧**：

1. **定義術語**：如果你的技能使用特定術語（如「項目」可能是 project 也可能是 item），明確定義
2. **區分相似操作**：`search` 和 `list` 有什麼不同？`update` 和 `edit` 有什麼不同？
3. **解釋預設行為**：「如果不指定排序，預設按建立時間從新到舊」
4. **描述邊界情況**：「如果搜尋結果為空，回傳空陣列 `[]`，不是 `null`」

### 5.2 參數描述的藝術

好的參數描述應該讓 LLM 能夠從使用者的自然語言中正確提取參數值。

```markdown
### 參數說明

**location**（必填，string）：
城市名稱或地區名稱。接受以下格式：
- 中文城市名："台北"、"東京"、"紐約"
- 英文城市名："Taipei"、"Tokyo"、"New York"
- 城市+國家："Portland, Oregon" 或 "Portland, Maine"
  （有同名城市時必須包含區域資訊）

如果使用者說「這裡」或「我的位置」，使用最後已知的使用者位置。
如果沒有已知位置，請使用者提供城市名稱。

**date**（選填，string）：
查詢日期。接受以下格式：
- 相對日期："today"、"tomorrow"、"next monday"
- 中文相對日期："今天"、"明天"、"下週一" 
  → 自動轉換為英文格式傳入
- 絕對日期：YYYY-MM-DD 格式，如 "2024-03-15"
- 日期範圍：不支援，如果使用者要查多天，分別呼叫

預設值：如果未指定，使用 "today"
```

### 5.3 回傳值的說明

回傳值的說明決定了 LLM 如何向使用者呈現結果：

```markdown
### 回傳值說明

腳本回傳 JSON 格式的天氣資料：

```json
{
  "location": "台北市",       // 城市全名（可能與輸入不同）
  "date": "2024-03-15",       // 查詢日期
  "current": {
    "temperature": 28,         // 當前溫度（攝氏）
    "feels_like": 31,          // 體感溫度
    "humidity": 75,            // 濕度（百分比）
    "wind_speed": 12,          // 風速（km/h）
    "description": "多雲",     // 天氣描述（中文）
    "icon": "☁️"               // 天氣圖示
  },
  "forecast": [                // 未來預報（最多7天）
    {
      "date": "2024-03-16",
      "temp_high": 30,
      "temp_low": 22,
      "description": "晴",
      "rain_probability": 10   // 降雨機率（百分比）
    }
  ]
}
```

**呈現指引**：
- 優先顯示當前溫度和天氣描述
- 如果濕度 > 80%，提醒使用者較潮濕
- 如果降雨機率 > 50%，建議帶傘
- 使用天氣圖示讓回應更生動
- 如果使用者在規劃行程，主動提供未來幾天的預報
```

### 5.4 情境引導

情境引導是一種高級技巧，幫助 LLM 在不同情境下做出正確的決策：

```markdown
### 使用情境

**情境A：簡單查詢**
使用者只想知道天氣 → 簡短回答，包含溫度和描述即可

**情境B：出行規劃**
使用者在規劃行程 → 提供未來幾天預報，加上穿衣/帶傘建議

**情境C：對話中的附帶查詢**
使用者在聊天中順便問天氣 → 簡短回答，不要中斷對話主題

**情境D：比較查詢**
使用者比較兩個城市的天氣 → 需要呼叫兩次，並對比呈現
```

---

## 6. 安全限制設定

### 6.1 controlPolicy 層級

`controlPolicy` 是第一道安全防線：

```yaml
# 無風險操作——自動執行
controlPolicy: allow

# 中等風險——需要確認
controlPolicy: confirm

# 高風險或暫時停用——拒絕執行
controlPolicy: deny
```

**如何選擇**：

問自己三個問題：

1. 這個操作有副作用嗎？（會改變外部狀態？）
   - 否 → `allow`
   - 是 → 繼續問
2. 副作用可逆嗎？
   - 是 → `confirm`
   - 否 → `confirm` 或 `deny`
3. 副作用影響範圍多大？
   - 只影響使用者自己 → `confirm`
   - 可能影響他人 → `deny`（或加強版 `confirm`）

### 6.2 Constraints 安全規則

在 Constraints 區段中定義安全規則時，要考慮以下面向：

**資料安全**：

```markdown
### 資料安全
- 不要在回應中暴露 API 金鑰或存取令牌
- 不要存取超出技能範圍的文件系統路徑
- 敏感查詢結果要過濾後再回傳
```

**操作安全**：

```markdown
### 操作安全
- 破壞性操作（刪除、覆寫）必須二次確認
- 批量操作限制每次最多 50 筆
- 不允許執行使用者提供的任意程式碼
```

**輸入安全**：

```markdown
### 輸入安全
- 驗證所有輸入參數的類型和範圍
- 過濾潛在的注入攻擊（SQL injection, Command injection）
- 限制字串輸入的最大長度
```

### 6.3 輸入驗證

在 Instructions 中描述輸入驗證規則：

```markdown
### 輸入驗證

在呼叫腳本前，確認以下驗證規則：

| 參數 | 驗證規則 |
|------|---------|
| `title` | 非空字串，長度 1-200 |
| `priority` | 必須是 "high", "medium", "low" 之一 |
| `due_date` | 有效的日期格式，且不早於今天 |
| `tags` | 陣列，每個元素為非空字串，最多 10 個 |

如果驗證失敗，不要嘗試修正——告訴使用者問題所在，
讓使用者提供正確的輸入。
```

### 6.4 輸出過濾

```markdown
### 輸出處理

腳本回傳的結果中可能包含技術性資訊。
向使用者呈現時，請過濾以下內容：

- 隱藏內部 ID（如 "td_abc123"），用位置描述代替
- 隱藏時間戳的時區資訊
- 錯誤訊息要轉換為使用者友善的描述
  - "CONNECTION_TIMEOUT" → "目前服務暫時無法連線，請稍後再試"
  - "RATE_LIMIT_EXCEEDED" → "查詢太頻繁，請等一下再試"
  - "INVALID_LOCATION" → "找不到這個城市，請確認名稱是否正確"
```

---

## 7. 環境門控（requires）

### 7.1 requires.bins——二進位依賴

`requires.bins` 列出技能運作所需的外部二進位工具。OpenClaw 會在系統 PATH 中搜尋這些工具。

```yaml
requires:
  bins:
    - python3      # Python 3 直譯器
    - curl         # HTTP 客戶端
    - jq           # JSON 處理工具
    - ffmpeg       # 影音處理
```

**檢查機制**：

```
OpenClaw 啟動
    │
    ├── which python3  → /usr/bin/python3  ✅
    ├── which curl     → /usr/bin/curl     ✅
    ├── which jq       → /usr/bin/jq       ✅
    └── which ffmpeg   → NOT FOUND         ❌
         │
         ▼
    技能標記為 disabled
    日誌: "Skill 'video-processor' disabled: missing binary 'ffmpeg'"
```

**最佳實踐**：

- 只列出**直接使用**的工具，不要列間接依賴
- 如果工具是可選的（缺少時降級而非失敗），不要放在 `requires.bins`
- 在 SKILL.md 的 Notes 區段中提供安裝指引

```markdown
## Notes

### 安裝依賴

如果缺少必要的工具，以下是安裝方式：

**macOS**：
```bash
brew install python3 curl jq
```

**Ubuntu/Debian**：
```bash
sudo apt-get install python3 curl jq
```

**Windows**：
```powershell
winget install Python.Python.3 curl jqlang.jq
```
```

### 7.2 requires.env——環境變數

`requires.env` 確保技能所需的環境變數已設定。這通常用於 API 金鑰和配置值。

```yaml
requires:
  env:
    - WEATHER_API_KEY       # OpenWeatherMap API 金鑰
    - WEATHER_DEFAULT_CITY  # 預設城市（選填，但列在這裡）
```

**注意**：系統只檢查環境變數**是否存在**（非空），不檢查其值是否有效。如果你的 API 金鑰無效，技能會在執行時才失敗，而不是在啟動時。

**環境變數的設定方式**：

```bash
# 方法一：在 .env 文件中（推薦）
# ~/.openclaw/.env
WEATHER_API_KEY=your-api-key-here

# 方法二：在 shell 環境中
export WEATHER_API_KEY=your-api-key-here

# 方法三：在 openclaw.json 中
# ~/.openclaw/openclaw.json
{
  "env": {
    "WEATHER_API_KEY": "your-api-key-here"
  }
}
```

### 7.3 requires.skills——技能依賴

有些技能需要其他技能才能正常運作：

```yaml
requires:
  skills:
    - location-resolver    # 需要地點解析技能
    - unit-converter       # 需要單位轉換技能
```

**循環依賴檢測**：

OpenClaw 會自動檢測循環依賴。如果技能 A 依賴技能 B，而技能 B 又依賴技能 A，兩個技能都會被標記為 `disabled`。

### 7.4 requires.os——作業系統限制

```yaml
requires:
  os:
    - linux     # Linux
    - darwin    # macOS
    - win32     # Windows
```

如果不指定，預設為 `["*"]`（支援所有作業系統）。

### 7.5 門控失敗的處理

當門控檢查失敗時，OpenClaw 的處理流程：

```
門控檢查失敗
    │
    ├── 記錄日誌（WARNING 級別）
    ├── 標記技能為 disabled
    ├── 在 `openclaw skills list` 中顯示原因
    └── 不影響其他技能的正常運作
```

使用者可以查看門控失敗的原因：

```bash
openclaw skills status my-skill

# 輸出：
# Skill: my-skill
# Status: disabled
# Reason: 
#   - Missing binary: ffmpeg
#   - Missing environment variable: MEDIA_API_KEY
# 
# To fix:
#   1. Install ffmpeg: brew install ffmpeg (macOS) or apt install ffmpeg (Linux)
#   2. Set MEDIA_API_KEY in ~/.openclaw/.env
```

---

## 8. 完整範例集

### 8.1 簡單範例：單位換算技能

這是一個純文字技能（不需要腳本），完全依靠 LLM 的計算能力。

```markdown
---
name: unit-converter
description: 在各種計量單位之間進行換算，支援長度、重量、溫度、面積、體積等
version: 1.0.0
author: openclaw-community
tags:
  - utility
  - math
  - converter
metadata:
  openclaw:
    controlPolicy: allow
    requires:
      bins: []
      env: []
    timeout: 10
    category: utility
    triggers:
      keywords:
        - 換算
        - 轉換
        - 多少
        - 等於
---

## Purpose

在各種計量單位之間進行換算。支援的單位類別：

- **長度**：公尺、公分、英寸、英尺、碼、英里、公里
- **重量**：公斤、克、磅、盎司
- **溫度**：攝氏、華氏、開爾文
- **面積**：平方公尺、坪、畝、公頃
- **體積**：公升、毫升、加侖、盎司
- **時間**：秒、分、時、天
- **數位儲存**：B、KB、MB、GB、TB

不適用於貨幣換算（使用 currency-converter）。

## Instructions

這是一個純推理技能，不需要呼叫外部腳本。

收到換算請求後：
1. 識別源單位和目標單位
2. 使用正確的換算公式計算
3. 回傳結果，保留適當的小數位數

常用換算公式：
- 1 英寸 = 2.54 公分
- 1 磅 = 0.453592 公斤
- °F = °C × 9/5 + 32
- 1 坪 = 3.30579 平方公尺
- 1 加侖 = 3.78541 公升

## Constraints

1. 不支援不同類別之間的換算（如長度→重量）
2. 溫度換算使用標準公式，不要近似
3. 如果使用者沒指定要換成什麼單位，選擇最常用的對應單位
4. 大數字使用千分位格式（如 1,000,000）

## Examples

**使用者**：5 英尺 8 英寸是多高？
**回應**：5 英尺 8 英寸 = 172.72 公分（約 1.73 公尺）。

**使用者**：華氏 98.6 度是攝氏幾度？
**回應**：華氏 98.6°F = 攝氏 37°C。這是人體的正常體溫。

**使用者**：500GB 是多少 MB？
**回應**：500 GB = 512,000 MB（以 1024 進位計算）
或 500,000 MB（以 1000 進位計算）。
一般檔案大小用 1024 進位，硬碟標示用 1000 進位。
```

### 8.2 中等範例：待辦事項管理技能

```markdown
---
name: todo-manager
description: 管理個人待辦事項，支援建立、查詢、更新、完成和刪除，包含優先級和標籤功能
version: 2.0.0
author: openclaw-community
tags:
  - productivity
  - todo
  - task-management
metadata:
  openclaw:
    controlPolicy: confirm
    requires:
      bins:
        - python3
      env: []
    timeout: 15
    cache: 0
    category: productivity
    triggers:
      keywords:
        - 待辦
        - 要做
        - 記一下
        - 完成了
        - 做完了
        - 任務
---

## Purpose

管理使用者的個人待辦事項清單。適用於：

- 快速記錄待辦事項
- 查看今天/本週/所有待辦
- 標記任務完成
- 更新任務資訊
- 按優先級或標籤篩選任務

不適用於：
- 團隊任務管理（使用 project-manager）
- 日曆和會議排程（使用 calendar-manager）
- 長期目標追蹤（使用 goal-tracker）

## Instructions

使用 `scripts/main.py` 執行所有操作。

### create：建立待辦事項

| 參數 | 類型 | 必要 | 說明 |
|------|------|------|------|
| action | string | ✅ | "create" |
| title | string | ✅ | 標題（1-200 字） |
| description | string | ⬜ | 詳細描述 |
| priority | string | ⬜ | "high" / "medium"(預設) / "low" |
| due_date | string | ⬜ | YYYY-MM-DD 格式 |
| tags | list | ⬜ | 標籤列表 |

```json
{
  "action": "create",
  "title": "買牛奶",
  "priority": "low",
  "tags": ["shopping"]
}
```

### list：查詢待辦事項

| 參數 | 類型 | 必要 | 說明 |
|------|------|------|------|
| action | string | ✅ | "list" |
| filter | string | ⬜ | "today" / "this_week" / "all"(預設) |
| status | string | ⬜ | "pending"(預設) / "completed" / "all" |
| priority | string | ⬜ | 按優先級篩選 |
| tags | list | ⬜ | 按標籤篩選 |
| limit | number | ⬜ | 最多回傳筆數（預設 10） |

### update：更新待辦事項

| 參數 | 類型 | 必要 | 說明 |
|------|------|------|------|
| action | string | ✅ | "update" |
| id | string | ✅ | 待辦 ID |
| title | string | ⬜ | 新標題 |
| status | string | ⬜ | "completed" / "pending" |
| priority | string | ⬜ | 新優先級 |

### delete：刪除待辦事項

| 參數 | 類型 | 必要 | 說明 |
|------|------|------|------|
| action | string | ✅ | "delete" |
| id | string | ✅ | 待辦 ID |

**注意**：刪除操作必須在 SKILL.md 呼叫前向使用者確認。

## Constraints

1. **確認建立**：如果使用者的描述很模糊（如「記一下那件事」），先確認具體內容
2. **確認刪除**：刪除前必須告訴使用者要刪除什麼，並等待確認
3. **不主動提醒**：不要在使用者沒有詢問時主動顯示待辦清單
4. **限制數量**：每次查詢最多顯示 10 筆，告知使用者可以要求看更多
5. **不存敏感資訊**：不要將密碼、金鑰等存入待辦事項
6. **重複檢測**：如果標題與現有待辦高度相似，提醒使用者

## Examples

### 快速記錄
**使用者**：記一下明天要打電話給醫生
**行動**：create(title="打電話給醫生", due_date="明天", priority="medium")
**回應**：已記錄：「打電話給醫生」，到期日：明天。需要加其他備註嗎？

### 查看今天的待辦
**使用者**：今天有什麼要做的？
**行動**：list(filter="today", status="pending")
**回應**：今天有 3 件待辦：
1. 🔴 交季報（高優先）
2. 🟡 回覆 John 的信（中優先）
3. 🟢 買牛奶（低優先）

### 標記完成
**使用者**：季報做好了
**行動**：先搜尋 → update(id="...", status="completed")
**回應**：已將「交季報」標記為完成 ✅ 今天還剩 2 件待辦。

### 模糊請求
**使用者**：幫我記一下
**回應**：好的，你想記什麼呢？請告訴我待辦事項的內容。
```

### 8.3 複雜範例：資料庫查詢助手技能

```markdown
---
name: db-query-assistant
description: >-
  安全地查詢和操作資料庫。支援 PostgreSQL 和 MySQL，
  提供自然語言轉 SQL、查詢結果視覺化、和資料庫結構探索功能。
version: 3.0.0
author: openclaw-labs
tags:
  - database
  - sql
  - dev-tools
  - productivity
metadata:
  openclaw:
    controlPolicy: confirm
    requires:
      bins:
        - python3
      env:
        - DATABASE_URL
      skills: []
    timeout: 120
    retries: 1
    cache: 0
    category: dev-tools
    triggers:
      keywords:
        - 資料庫
        - SQL
        - 查詢
        - table
        - database
      patterns:
        - ".*資料庫.*"
        - ".*SELECT.*FROM.*"
---

## Purpose

這個技能提供安全的資料庫操作能力，適用於：

- 用自然語言描述查詢，自動轉換為 SQL
- 探索資料庫結構（表格、欄位、關聯）
- 執行 SELECT 查詢並格式化結果
- 執行安全的 INSERT/UPDATE/DELETE（需確認）
- 匯出查詢結果

不適用於：
- 資料庫管理操作（建表、修改結構、權限管理）
- 大量資料匯入/匯出
- 資料庫備份和還原

## Instructions

使用 `scripts/main.py` 連接資料庫。連接字串從 DATABASE_URL 環境變數讀取。

### 自然語言查詢

```json
{
  "action": "natural_query",
  "question": "上個月有多少新用戶註冊？",
  "context": "users 表有 created_at 欄位"
}
```

回傳：
```json
{
  "sql": "SELECT COUNT(*) as count FROM users WHERE created_at >= '2024-02-01' AND created_at < '2024-03-01'",
  "result": [{ "count": 1523 }],
  "execution_time_ms": 45
}
```

**重要**：生成 SQL 後，先向使用者展示 SQL，確認後再執行。

### 探索結構

```json
{
  "action": "explore",
  "target": "tables" | "columns" | "relations",
  "table": "users"
}
```

### 執行 SQL

```json
{
  "action": "execute",
  "sql": "SELECT * FROM users WHERE status = 'active' LIMIT 10",
  "format": "table" | "json" | "csv"
}
```

### 結果說明

查詢結果回傳 JSON 格式：
```json
{
  "columns": ["id", "name", "email", "created_at"],
  "rows": [...],
  "row_count": 10,
  "total_count": 1523,
  "truncated": true,
  "execution_time_ms": 45
}
```

如果 truncated 為 true，告知使用者只顯示了部分結果。

## Constraints

### 安全約束（最重要！）
1. **只讀優先**：預設只允許 SELECT 查詢
2. **寫入確認**：INSERT/UPDATE/DELETE 必須展示 SQL 並獲得使用者二次確認
3. **禁止 DDL**：不允許 CREATE TABLE, ALTER TABLE, DROP TABLE 等操作
4. **禁止危險操作**：不允許 TRUNCATE, DROP DATABASE 等
5. **注入防護**：所有使用者輸入必須參數化，不要直接拼接 SQL
6. **結果限制**：SELECT 查詢自動加上 LIMIT 100（除非使用者明確要求更多）
7. **敏感欄位**：不要顯示 password, token, secret 等欄位的值

### 行為約束
8. 生成的 SQL 要先展示給使用者，確認後再執行
9. 如果查詢可能很慢（如全表掃描），提醒使用者
10. 結果用表格格式呈現，大量資料建議匯出

## Examples

### 自然語言查詢
**使用者**：上個月有多少訂單？
**思考**：使用者想查詢上個月的訂單數量
**行動**：natural_query("上個月有多少訂單？")
**確認**：我準備執行以下查詢：
```sql
SELECT COUNT(*) as order_count 
FROM orders 
WHERE created_at >= '2024-02-01' 
AND created_at < '2024-03-01'
```
**使用者**：好
**結果**：{ "order_count": 3847 }
**回應**：上個月（2024年2月）共有 3,847 筆訂單。

### 探索結構
**使用者**：資料庫有哪些表格？
**行動**：explore(target="tables")
**回應**：資料庫中有以下表格：
| 表格 | 說明 | 記錄數 |
|------|------|--------|
| users | 使用者 | 15,234 |
| orders | 訂單 | 89,123 |
| products | 商品 | 2,456 |

### 安全攔截
**使用者**：刪掉所有過期的訂單
**回應**：我可以幫你找出過期的訂單，但刪除操作需要你確認。
讓我先查一下有多少筆過期訂單。

**行動**：execute("SELECT COUNT(*) FROM orders WHERE status = 'expired'")
**結果**：共有 234 筆過期訂單。
**回應**：共找到 234 筆過期訂單。確定要刪除這些記錄嗎？
這個操作不可逆。

## Notes

### 支援的資料庫
- PostgreSQL 12+
- MySQL 8.0+

### DATABASE_URL 格式
```
postgresql://user:password@host:5432/dbname
mysql://user:password@host:3306/dbname
```
```

---

## 9. 除錯技巧

### 9.1 技能未被發現

**症狀**：`openclaw skills list` 中看不到你的技能。

**排查步驟**：

```bash
# 1. 檢查文件位置
ls ~/.openclaw/skills/my-skill/SKILL.md

# 2. 檢查文件名（必須是 SKILL.md，大小寫敏感）
ls -la ~/.openclaw/skills/my-skill/

# 3. 強制重新載入
openclaw skills reload --verbose

# 4. 檢查 YAML 語法
openclaw skills validate my-skill
```

**常見原因**：

- 文件名打錯（`skill.md`、`Skill.md` 都不行，必須是 `SKILL.md`）
- YAML frontmatter 語法錯誤（缺少 `---`、縮排不正確）
- 目錄位置錯誤（不在掃描路徑中）
- 文件編碼問題（確保是 UTF-8）

### 9.2 技能未被選用

**症狀**：技能在列表中顯示 `active`，但 LLM 就是不選用它。

**排查步驟**：

```bash
# 1. 查看 LLM 的推理日誌
openclaw chat --debug "你的測試指令"

# 日誌會顯示：
# [ReAct] Available skills: [..., my-skill, ...]
# [ReAct] Thinking: 使用者想要 X，可用的技能有...
# [ReAct] Decision: 使用 other-skill（不是 my-skill）
# [ReAct] Reason: my-skill 的描述不匹配使用者意圖
```

**常見原因與解決方案**：

| 原因 | 解決方案 |
|------|---------|
| `description` 太模糊 | 加入更具體的關鍵詞 |
| `Purpose` 缺少使用場景 | 加入更多「何時用」的描述 |
| 其他技能的描述更匹配 | 差異化你的技能描述 |
| `triggers.keywords` 不完整 | 加入更多使用者可能用的詞 |
| 技能被動態載入策略延遲 | 提高 `description` 的精確度 |

### 9.3 技能被錯誤使用

**症狀**：LLM 選用了你的技能，但參數錯誤或誤解了用途。

**排查步驟**：

```bash
# 檢查 LLM 傳入的參數
openclaw chat --debug --trace-skills "你的測試指令"

# 查看完整的 SKILL.md 被注入的方式
openclaw skills inspect my-skill --show-prompt
```

**常見原因與解決方案**：

| 原因 | 解決方案 |
|------|---------|
| 參數描述不夠精確 | 加入類型、範圍、格式的具體說明 |
| 缺少範例 | 加入更多 Examples |
| Instructions 有歧義 | 重新撰寫，消除歧義 |
| Constraints 不夠具體 | 加入數字限制和具體規則 |

### 9.4 腳本執行失敗

**症狀**：技能被正確選用，但腳本執行返回錯誤。

```bash
# 手動執行腳本測試
cd ~/.openclaw/skills/my-skill
echo '{"action":"test"}' | python3 scripts/main.py

# 檢查腳本權限
ls -la scripts/main.py

# 檢查 Python 依賴
pip3 install -r scripts/requirements.txt

# 查看腳本錯誤日誌
openclaw logs --filter skill:my-skill --level error
```

### 9.5 除錯工具

OpenClaw 提供了一套完整的除錯工具：

```bash
# 驗證技能結構
openclaw skills validate my-skill

# 互動式測試
openclaw skills test my-skill

# 查看技能被注入到提示詞的方式
openclaw skills inspect my-skill --show-prompt

# 模擬技能調用（不實際執行腳本）
openclaw skills dry-run my-skill --input '{"action":"test"}'

# 查看完整的除錯日誌
openclaw logs --verbose --filter skill:my-skill
```

---

## 10. 常見錯誤與解決方案

| # | 錯誤 | 原因 | 解決方案 |
|---|------|------|---------|
| 1 | YAML 解析失敗 | Frontmatter 語法錯誤 | 用 YAML lint 工具檢查；注意縮排用空格不用 Tab |
| 2 | 技能名稱衝突 | 與已有技能同名 | 改名或利用優先級覆蓋機制 |
| 3 | 環境檢查失敗 | 缺少二進位或環境變數 | 安裝工具或設定環境變數 |
| 4 | 技能被忽略 | description 不匹配使用者意圖 | 改善描述和 triggers |
| 5 | 參數傳遞錯誤 | Instructions 描述不精確 | 加入參數表格和 JSON 範例 |
| 6 | 腳本超時 | timeout 設太短或腳本太慢 | 增加 timeout 或優化腳本 |
| 7 | 快取導致舊資料 | cache 值設太高 | 降低 cache 值或設為 0 |
| 8 | 安全策略阻擋 | controlPolicy 設定過嚴 | 根據風險等級調整策略 |
| 9 | 循環依賴 | 技能互相依賴 | 重新設計依賴關係 |
| 10 | 編碼問題 | 文件不是 UTF-8 | 轉換文件編碼 |
| 11 | description 中的 YAML 特殊字元 | 包含冒號、引號等 | 用引號包裹或 `>-` 語法 |
| 12 | tags 格式錯誤 | 不是 YAML 列表格式 | 確保使用 `- tag` 格式 |

**錯誤 #11 的詳細說明**：

YAML 中某些字元有特殊含義，在 description 中使用時需要注意：

```yaml
# ❌ 會出錯
description: 管理 GitHub Issues: 建立、搜尋、更新

# ✅ 正確
description: "管理 GitHub Issues: 建立、搜尋、更新"

# ✅ 也正確
description: >-
  管理 GitHub Issues: 建立、搜尋、更新
```

---

## 11. SKILL.md 撰寫檢查清單

在發布你的技能之前，用以下清單確認品質：

### Frontmatter 檢查

- [ ] `name` 使用 kebab-case，唯一且有描述性
- [ ] `description` 在 1-2 句話內準確描述功能
- [ ] `version` 遵循 SemVer 格式
- [ ] `author` 正確無誤
- [ ] `tags` 包含 3-7 個相關標籤
- [ ] `controlPolicy` 根據風險等級正確設定
- [ ] `requires` 列出所有必要的依賴
- [ ] `timeout` 設定合理
- [ ] YAML 語法正確（通過 `openclaw skills validate`）

### Purpose 檢查

- [ ] 清楚描述技能做什麼
- [ ] 列出適用場景（至少 3 個）
- [ ] 列出不適用場景
- [ ] 指向替代技能（如果有）

### Instructions 檢查

- [ ] 每個操作都有參數表格
- [ ] 每個參數都有類型、必要性和說明
- [ ] 有 JSON 格式的呼叫範例
- [ ] 回傳值格式有說明
- [ ] 呈現指引清晰

### Constraints 檢查

- [ ] 安全約束在最前面
- [ ] 限制有具體數字
- [ ] 邊界情況有處理方式
- [ ] 不過度限制（不影響正常使用）

### Examples 檢查

- [ ] 至少 3 個範例
- [ ] 覆蓋：基本用法、進階用法、錯誤處理
- [ ] 範例格式完整（輸入→思考→行動→結果→回應）
- [ ] 範例中的 JSON 格式正確

### 腳本檢查（如果有）

- [ ] 腳本有執行權限
- [ ] 依賴都已列在 requirements.txt
- [ ] 錯誤處理完善
- [ ] 超時處理存在
- [ ] 安全性已考慮（輸入驗證、注入防護）

---

## 12. 總結

撰寫 SKILL.md 是一門結合技術寫作和 AI 工程的藝術。關鍵要點：

1. **description 是第一印象**——花時間寫好它
2. **Purpose 是導航**——幫助 LLM 做正確的選擇
3. **Instructions 是地圖**——精確到 LLM 不會迷路
4. **Constraints 是護欄**——防止誤用和濫用
5. **Examples 是教材**——最有效的教學方式

記住：**你的 SKILL.md 的讀者是一個非常聰明但沒有常識的實體（LLM）**。它能理解複雜的描述，但不能推斷你沒寫出來的東西。所以，明確、具體、無歧義——這是好的 SKILL.md 的三大原則。

在下一章，我們將進入進階領域——腳本整合、API 對接和複雜的多步驟技能開發。
