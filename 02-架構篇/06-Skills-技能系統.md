# Skills——OpenClaw 的可擴展技能系統

> **文件版本**：v1.0
> **適用範圍**：OpenClaw Skills 模組設計
> **讀者對象**：技能開發者、外掛作者、AI 工程師、對 Agent 工具系統有興趣的技術人員

---

## 目錄

1. [引言](#引言)
2. [Skills 的設計理念：用 Markdown 定義工具](#skills-的設計理念用-markdown-定義工具)
3. [技能的生命週期](#技能的生命週期)
4. [SKILL.md 的結構](#skillmd-的結構)
5. [技能目錄結構](#技能目錄結構)
6. [內建技能 vs 社群技能 vs 自定義技能](#內建技能-vs-社群技能-vs-自定義技能)
7. [技能的安全門控（Gating）](#技能的安全門控gating)
8. [技能的啟用/停用管理](#技能的啟用停用管理)
9. [ClawHub 市集簡介](#clawhub-市集簡介)
10. [技能開發的基本流程](#技能開發的基本流程)
11. [與 MCP Tools 的關係](#與-mcp-tools-的關係)
12. [總結](#總結)

---

## 引言

一個 AI Agent 的能力邊界，取決於它能「做」什麼。LLM 本身只能生成文字——它不能查詢天氣、不能管理行事曆、不能操作 Git、不能發送郵件。要讓 Agent 具備這些「做」的能力，就需要一個工具系統——讓 Agent 能夠調用外部的工具和服務來完成具體的操作。

OpenClaw 的 Skills（技能）模組就是這個工具系統。它提供了一個標準化、安全、可擴展的框架，讓 Agent 能夠：
- 查詢各種 API（天氣、新聞、股票等）
- 操作本地系統（檔案管理、Shell 命令等）
- 整合第三方服務（GitHub、Google Calendar、Notion 等）
- 執行自定義的腳本和程式

Skills 模組最獨特的設計是「用 Markdown 定義工具」。在其他 Agent 框架中，工具通常透過程式碼（Python 函數、JSON Schema、TypeScript 介面）來定義。OpenClaw 則選擇了一種更人類友好的方式——使用 Markdown 檔案（SKILL.md）來定義技能的元資料、參數、行為和範例。這大幅降低了技能開發的門檻，讓即使不熟悉程式設計的使用者也能理解和修改技能定義。

本文件將全面介紹 Skills 模組的設計理念、技術實作和最佳實踐。無論你是想要使用現有技能、開發自己的技能，還是想要理解 Agent 工具系統的原理，本文件都將提供完整的參考。

---

## Skills 的設計理念：用 Markdown 定義工具

### 為什麼用 Markdown

在設計 Skills 系統時，OpenClaw 面臨一個核心問題：如何讓工具定義既能被機器解析，又能被人類輕鬆理解和修改？

**傳統方式的問題：**

```python
# Python 函數定義方式（LangChain 風格）
@tool
def get_weather(location: str, date: str = "today") -> dict:
    """Get weather forecast for a specific location and date.
    
    Args:
        location: City name or coordinates
        date: Target date (default: today)
    
    Returns:
        Weather data including temperature and conditions
    """
    # 實作程式碼...
```

```json
// JSON Schema 定義方式（OpenAI 風格）
{
  "name": "get_weather",
  "description": "Get weather forecast",
  "parameters": {
    "type": "object",
    "properties": {
      "location": {"type": "string", "description": "City name"},
      "date": {"type": "string", "description": "Target date"}
    },
    "required": ["location"]
  }
}
```

這些方式有各自的問題：
- Python 方式需要懂 Python 才能修改
- JSON 方式冗長且不適合閱讀
- 都不是「自文件化」的——需要額外的文件來解釋如何使用
- 都不方便進行 diff 和 Code Review

**OpenClaw 的 Markdown 方式：**

```markdown
# 天氣查詢技能

## 基本資訊
- **名稱**：weather
- **版本**：1.0.0
- **描述**：查詢指定地點的天氣預報

## 參數
| 參數 | 類型 | 必填 | 預設值 | 說明 |
|------|------|------|-------|------|
| location | string | 是 | - | 城市名稱或座標 |
| date | string | 否 | today | 目標日期 |

## 使用範例
- 「查看台北天氣」→ weather(location="台北")
- 「明天台北天氣如何」→ weather(location="台北", date="明天")

## 回傳格式
```json
{"temp": "25-30°C", "condition": "晴天", "humidity": "65%"}
```

## 注意事項
- 需要網路連線
- 目前支援全球主要城市
- 預報最多支援 7 天
```

### Markdown 方式的優勢

**優勢 1：即看即懂**

任何人打開 SKILL.md，都能立即理解這個技能做什麼、需要什麼參數、回傳什麼結果。不需要懂任何程式語言。

**優勢 2：LLM 天生能理解**

LLM 擅長閱讀 Markdown。將 SKILL.md 的內容直接注入提示詞，LLM 就能理解如何使用這個技能。不需要額外的格式轉換。

**優勢 3：自文件化**

SKILL.md 本身就是技能的完整文件。不需要另外撰寫 README 或 API 文件。技能的定義和文件合為一體。

**優勢 4：版本控制友好**

Markdown 是純文字，Git diff 清晰可讀。團隊協作時，技能的變更可以像程式碼一樣進行 Code Review。

**優勢 5：跨語言**

技能的定義（Markdown）和實作（腳本）是分離的。定義不依賴任何特定的程式語言，而實作腳本可以用任何語言撰寫。

**優勢 6：生態系統友好**

使用者可以在 GitHub 上分享 SKILL.md，其他人可以直接閱讀、理解並使用。比分享一段 Python 程式碼或 JSON Schema 友好得多。

### 設計原則

Skills 系統的設計遵循以下原則：

1. **定義與實作分離**：SKILL.md 定義「什麼」和「為什麼」，腳本實作「怎麼做」
2. **最小驚訝原則**：技能的行為應該符合使用者的直覺預期
3. **安全優先**：每個技能都必須通過安全門控才能執行
4. **可組合性**：技能之間可以互相組合，形成更強大的能力
5. **降級友好**：當技能不可用時，Agent 應該能夠告知使用者而非崩潰

---

## 技能的生命週期

### 完整生命週期

每個技能都經歷以下生命週期：

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ 1. 發現   │────▶│ 2. 註冊   │────▶│ 3. 調用   │────▶│ 4. 回傳   │
│ Discovery │     │ Register │     │ Invoke   │     │ Return   │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
```

### 階段 1：發現（Discovery）

Skills 模組在啟動時和收到刷新指令時，會掃描技能目錄來發現可用的技能：

```
發現流程：

1. 掃描預設技能目錄
   └── ~/.openclaw/skills/

2. 掃描額外的技能目錄（如果配置了）
   └── ~/my-custom-skills/

3. 對每個子目錄：
   ├── 檢查是否存在 SKILL.md
   ├── 解析 SKILL.md 的元資料
   ├── 驗證必要欄位（名稱、描述、參數）
   ├── 檢查腳本檔案是否存在
   └── 記錄發現結果

4. 輸出發現報告：
   ├── ✅ weather：天氣查詢技能 (v1.0.0)
   ├── ✅ calendar：行事曆管理 (v1.2.0)
   ├── ✅ git-helper：Git 操作助手 (v0.9.0)
   ├── ⚠️ mail-sender：缺少 scripts/ 目錄
   └── ❌ broken-skill：SKILL.md 格式錯誤
```

### 階段 2：註冊（Register）

發現的有效技能會被註冊到技能註冊表（Skill Registry）中：

```
┌──────────────────────────────────────────────────────────┐
│                    Skill Registry                         │
├──────────┬──────────┬──────────┬──────────┬──────────────┤
│ Name     │ Version  │ Status   │ Gating   │ Last Used    │
├──────────┼──────────┼──────────┼──────────┼──────────────┤
│ weather  │ 1.0.0    │ enabled  │ none     │ 2025-01-16   │
│ calendar │ 1.2.0    │ enabled  │ confirm  │ 2025-01-16   │
│ git-help │ 0.9.0    │ enabled  │ none     │ 2025-01-15   │
│ shell    │ 1.0.0    │ disabled │ approve  │ never        │
│ file-mgr │ 1.1.0    │ enabled  │ confirm  │ 2025-01-14   │
└──────────┴──────────┴──────────┴──────────┴──────────────┘
```

註冊過程包括：
- 分配唯一的技能 ID
- 設定初始狀態（啟用/停用）
- 配置安全門控等級
- 建立技能的元資料索引
- 生成技能的使用說明（注入提示詞用）

### 階段 3：調用（Invoke）

當 Brain 在 ReAct 迴圈中決定調用某個技能時，Skills 模組執行以下流程：

```
調用流程：

1. 解析調用請求
   ├── 技能名稱：weather
   └── 參數：{"location": "台北", "date": "明天"}

2. 查找技能
   └── 在註冊表中查找 "weather" → 找到

3. 驗證狀態
   ├── 技能是否啟用？ → 是
   └── 技能是否健康？ → 是

4. 安全門控
   ├── 門控等級：none
   └── 不需要額外確認

5. 參數驗證
   ├── location：字串 "台北" → ✅
   └── date：字串 "明天" → ✅

6. 執行腳本
   ├── 執行 scripts/get_weather.sh "台北" "明天"
   ├── 設定環境變數（API Key 等）
   ├── 設定超時限制（30 秒）
   └── 捕獲標準輸出和錯誤

7. 處理結果
   ├── 退出碼：0（成功）
   ├── 標準輸出：{"temp": "18-24°C", "condition": "多雲"}
   └── 返回結構化結果
```

### 階段 4：回傳（Return）

技能執行完成後，結果被格式化並返回給 Brain：

```json
{
  "type": "skill.result",
  "payload": {
    "skill_name": "weather",
    "invocation_id": "inv_abc123",
    "status": "success",
    "result": {
      "temp": "18-24°C",
      "condition": "多雲",
      "rain_chance": "30%",
      "humidity": "70%"
    },
    "execution_time_ms": 1200,
    "metadata": {
      "api_calls": 1,
      "data_source": "OpenWeatherMap"
    }
  }
}
```

Brain 接收到結果後，將其加入 ReAct 迴圈的上下文中，繼續推理過程。

---

## SKILL.md 的結構

### 完整範例

```markdown
---
name: weather
version: 1.0.0
author: OpenClaw Team
license: MIT
tags: [utility, weather, api]
gating: none
enabled: true
dependencies:
  - curl
  - jq
env:
  - WEATHER_API_KEY
---

# 天氣查詢技能

## 描述

查詢指定地點的天氣預報資訊。支援全球主要城市的當日和未來 7 天天氣預報。
資料來源為 OpenWeatherMap API。

## 參數

| 參數 | 類型 | 必填 | 預設值 | 說明 |
|------|------|------|-------|------|
| location | string | 是 | - | 城市名稱（中英文皆可）或經緯度座標 |
| date | string | 否 | "today" | 查詢日期，支援 "today"、"tomorrow"、具體日期（YYYY-MM-DD）|
| unit | string | 否 | "metric" | 溫度單位："metric"（攝氏）或 "imperial"（華氏）|
| detail | boolean | 否 | false | 是否返回詳細資訊（含每小時預報）|

## 使用範例

以下是常見的使用情境和對應的調用方式：

### 基本查詢
- 使用者：「今天台北天氣如何？」
- 調用：`weather(location="台北")`

### 指定日期
- 使用者：「明天東京天氣怎樣？」
- 調用：`weather(location="東京", date="tomorrow")`

### 詳細查詢
- 使用者：「這週末台中天氣預報，我需要詳細資訊」
- 調用：`weather(location="台中", date="2025-01-18", detail=true)`

### 華氏溫度
- 使用者：「紐約天氣？用華氏」
- 調用：`weather(location="New York", unit="imperial")`

## 回傳格式

### 基本回傳
```json
{
  "location": "台北",
  "date": "2025-01-16",
  "temperature": {
    "current": "27°C",
    "high": "30°C",
    "low": "25°C"
  },
  "condition": "晴天",
  "humidity": "65%",
  "wind": "東北風 15km/h",
  "rain_chance": "10%",
  "uv_index": 7,
  "sunrise": "06:35",
  "sunset": "17:28"
}
```

### 詳細回傳（detail=true）
包含以上所有欄位，另外加上每 3 小時的預報資料。

## 錯誤處理

| 錯誤碼 | 說明 | 建議處理 |
|--------|------|---------|
| LOCATION_NOT_FOUND | 無法識別的地點 | 請使用者提供更精確的地名 |
| API_RATE_LIMITED | API 請求頻率超限 | 稍後重試 |
| API_KEY_INVALID | API 金鑰無效或過期 | 請使用者更新 API 金鑰 |
| NETWORK_ERROR | 網路連線失敗 | 檢查網路連線 |

## 配置要求

### 環境變數
- `WEATHER_API_KEY`：OpenWeatherMap API 金鑰（必要）
  - 免費方案：每分鐘 60 次呼叫
  - 取得方式：https://openweathermap.org/api

### 系統依賴
- `curl`：HTTP 請求工具
- `jq`：JSON 處理工具

## 限制
- 免費 API 方案每分鐘最多 60 次查詢
- 預報資料最多 7 天
- 歷史天氣資料不在此技能範圍內
- 某些小城市可能沒有精確資料
```

### YAML Frontmatter 欄位說明

| 欄位 | 必填 | 說明 |
|------|------|------|
| `name` | 是 | 技能的唯一識別名稱（英文、小寫、連字號分隔） |
| `version` | 是 | 語意化版本號（SemVer） |
| `author` | 否 | 技能作者 |
| `license` | 否 | 授權條款 |
| `tags` | 否 | 標籤列表，用於分類和搜尋 |
| `gating` | 否 | 安全門控等級（none / confirm / approve） |
| `enabled` | 否 | 是否預設啟用（預設 true） |
| `dependencies` | 否 | 系統依賴列表 |
| `env` | 否 | 需要的環境變數列表 |
| `timeout` | 否 | 執行超時時間（秒，預設 30） |
| `max_retries` | 否 | 失敗時的最大重試次數（預設 0） |

### Markdown 正文的約定

SKILL.md 的正文使用以下約定的標題結構：

| 標題 | 必填 | 用途 |
|------|------|------|
| `## 描述` | 是 | 技能的詳細描述，會被注入提示詞 |
| `## 參數` | 是 | 參數定義表格 |
| `## 使用範例` | 建議 | 使用範例，幫助 LLM 理解調用方式 |
| `## 回傳格式` | 建議 | 回傳資料的格式說明 |
| `## 錯誤處理` | 建議 | 可能的錯誤和處理建議 |
| `## 配置要求` | 視需求 | 環境變數、依賴等配置需求 |
| `## 限制` | 建議 | 技能的已知限制 |

---

## 技能目錄結構

### 標準目錄結構

每個技能是一個獨立的目錄，位於 `~/.openclaw/skills/` 下：

```
~/.openclaw/skills/weather/
├── SKILL.md                # 技能定義（必要）
├── scripts/                # 執行腳本目錄
│   ├── get_forecast.sh    # 主要腳本
│   ├── parse_response.py  # 輔助腳本
│   └── helpers/
│       └── geocode.sh     # 地理編碼輔助
├── references/             # 參考文件目錄
│   ├── api_docs.md        # API 文件摘錄
│   └── examples.md        # 額外的使用範例
├── tests/                  # 測試目錄（可選）
│   ├── test_basic.sh
│   └── test_error.sh
└── config/                 # 技能特定配置（可選）
    └── default.yaml
```

### 各目錄說明

**SKILL.md（必要）**

技能的核心定義檔案。Skills 模組通過解析這個檔案來理解技能的能力、參數和行為。

**scripts/（必要）**

包含技能的執行腳本。當 Brain 調用技能時，Skills 模組會執行這個目錄中的腳本。

腳本約定：
- 主要腳本的名稱應該反映技能的功能（如 `get_forecast.sh`）
- 腳本可以用任何語言撰寫（Bash、Python、Node.js、Go 等）
- 腳本必須有可執行權限
- 參數通過命令列引數或環境變數傳入
- 結果通過標準輸出（stdout）返回
- 錯誤通過標準錯誤（stderr）和非零退出碼報告

```bash
#!/bin/bash
# scripts/get_forecast.sh

LOCATION="$1"
DATE="${2:-today}"
UNIT="${3:-metric}"

# 調用 API
RESPONSE=$(curl -s "https://api.openweathermap.org/data/2.5/forecast?q=${LOCATION}&units=${UNIT}&appid=${WEATHER_API_KEY}")

# 處理回應
echo "$RESPONSE" | jq '{
  location: .city.name,
  temperature: {
    current: .list[0].main.temp,
    high: .list[0].main.temp_max,
    low: .list[0].main.temp_min
  },
  condition: .list[0].weather[0].description,
  humidity: .list[0].main.humidity,
  wind_speed: .list[0].wind.speed
}'
```

**references/（可選）**

存放技能的參考資料。這些資料不會被自動使用，但可以幫助使用者理解技能的背景知識。Brain 在需要更詳細的資訊時，可以讀取這個目錄。

**tests/（可選）**

技能的測試腳本。開發者可以用這些測試來驗證技能的正確性。

```bash
# tests/test_basic.sh
#!/bin/bash

# 測試基本查詢
RESULT=$(./scripts/get_forecast.sh "台北")
echo "$RESULT" | jq -e '.location' > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✅ 基本查詢測試通過"
else
    echo "❌ 基本查詢測試失敗"
    exit 1
fi
```

**config/（可選）**

技能特定的配置檔案。例如，天氣技能可能有一個配置檔案來儲存偏好的溫度單位或預設地點。

---

## 內建技能 vs 社群技能 vs 自定義技能

### 三種技能類型

OpenClaw 的技能生態系統分為三個層次：

```
┌──────────────────────────────────────────────────┐
│                   技能生態系統                      │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │  自定義技能（User Custom Skills）              │ │
│  │  使用者根據個人需求開發的私有技能               │ │
│  │  優先級最高——會覆蓋同名的社群和內建技能         │ │
│  └──────────────────────────────────────────────┘ │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │  社群技能（Community Skills）                  │ │
│  │  來自 ClawHub 市集或 GitHub 的第三方技能        │ │
│  │  經過社群審核                                  │ │
│  └──────────────────────────────────────────────┘ │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │  內建技能（Built-in Skills）                   │ │
│  │  隨 OpenClaw 安裝，提供基礎能力                │ │
│  │  由核心團隊維護                                │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

### 內建技能

OpenClaw 預裝了一系列常用的基礎技能：

| 技能名稱 | 功能 | 門控等級 |
|---------|------|---------|
| `memory-search` | 搜尋記憶庫 | none |
| `memory-write` | 寫入記憶 | none |
| `web-search` | 網路搜尋 | none |
| `web-fetch` | 擷取網頁內容 | none |
| `file-read` | 讀取本地檔案 | confirm |
| `file-write` | 寫入本地檔案 | confirm |
| `shell-exec` | 執行 Shell 命令 | approve |
| `calendar` | 行事曆管理 | none |
| `reminder` | 設定提醒 | none |
| `calculator` | 數學計算 | none |
| `code-run` | 執行程式碼片段 | approve |
| `git-ops` | Git 操作 | confirm |

### 社群技能

社群技能來自 ClawHub 市集或 GitHub，由第三方開發者貢獻：

```bash
# 從 ClawHub 安裝社群技能
$ openclaw skill install clawhub/stock-ticker
Installing stock-ticker v1.2.0...
✅ Installed to ~/.openclaw/skills/stock-ticker/
⚠️ This skill requires ALPHA_VANTAGE_API_KEY environment variable

# 從 GitHub 安裝
$ openclaw skill install github:user/weather-advanced
```

### 自定義技能

使用者可以根據自己的需求開發私有技能。自定義技能的優先級最高——如果同時存在內建的 `weather` 和自定義的 `weather`，系統會使用自定義版本。

```
技能查找順序（優先級從高到低）：

1. ~/.openclaw/skills/custom/    # 使用者自定義
2. ~/.openclaw/skills/community/ # 社群技能
3. ~/.openclaw/skills/builtin/   # 內建技能
```

---

## 技能的安全門控（Gating）

### 為什麼需要門控

技能系統讓 Agent 具備了「做事」的能力，但這也帶來了安全風險。一個不受控的 Agent 可能會：
- 在不該修改檔案的時候修改了重要檔案
- 執行了具有破壞性的 Shell 命令
- 未經授權就存取了敏感資料
- 向外部服務發送了不當的請求

安全門控（Gating）是 OpenClaw 防止這些風險的核心機制。它在 Agent 調用技能之前插入一個「檢查點」，根據技能的門控等級決定是否需要使用者的確認。

### 門控等級

OpenClaw 定義了三個門控等級：

**等級 1：None（無門控）**

技能可以自由調用，不需要任何確認。適用於唯讀操作和低風險操作。

```
Agent 決定調用 weather(location="台北")
    │
    ▼ 門控等級：none → 直接執行
    │
    ▼
執行成功，返回天氣資料
```

適用場景：天氣查詢、計算器、記憶搜尋、網頁瀏覽

**等級 2：Confirm（確認門控）**

技能調用前，Agent 會在回應中告知使用者即將執行的操作，使用者需要確認後才會執行。

```
Agent 決定調用 file-write(path="/home/user/notes.md", content="...")
    │
    ▼ 門控等級：confirm
    │
    ▼
Agent 回應使用者：
  "我即將將內容寫入 /home/user/notes.md，確認執行嗎？[確認/取消]"
    │
    ├── 使用者回覆「確認」→ 執行寫入
    └── 使用者回覆「取消」→ 取消操作
```

適用場景：檔案寫入、Git 操作、行事曆修改

**等級 3：Approve（審批門控）**

最嚴格的門控等級。技能調用需要使用者明確審批，且會顯示完整的操作詳情供審查。

```
Agent 決定調用 shell-exec(command="rm -rf ./old-data/")
    │
    ▼ 門控等級：approve
    │
    ▼
Agent 回應使用者：
  "⚠️ 需要審批的操作：
   技能：shell-exec
   命令：rm -rf ./old-data/
   影響：將永久刪除 ./old-data/ 目錄及其所有內容
   風險等級：高
   
   請審批：[批准/拒絕/修改]"
    │
    ├── 使用者回覆「批准」→ 執行
    ├── 使用者回覆「拒絕」→ 取消
    └── 使用者回覆「修改」→ 使用者提供修改後的命令
```

適用場景：Shell 命令、程式碼執行、敏感資料操作

### 門控配置

使用者可以自定義每個技能的門控等級：

```yaml
# ~/.openclaw/config/skills.yaml

gating:
  # 覆蓋特定技能的門控等級
  overrides:
    file-read: "none"        # 將檔案讀取降級為無門控
    web-search: "confirm"    # 將網頁搜尋提升為確認門控
    
  # 預設門控等級（對未指定的技能）
  default: "confirm"
  
  # 自動確認設定
  auto_confirm:
    enabled: false           # 是否允許自動確認
    trusted_skills:          # 信任的技能列表（自動確認）
      - "memory-search"
      - "calculator"
      - "weather"
    max_auto_confirms: 5     # 單次 ReAct 迴圈中最多自動確認 5 次
```

### 門控的使用者體驗

好的門控設計應該在安全和便利之間取得平衡。以下是一些使用者體驗的優化：

**快捷確認**：在 Telegram 中，確認操作可以用 inline keyboard 按鈕來完成，使用者只需點一下。

**批次確認**：如果一次 ReAct 迴圈中有多個需要確認的操作，可以一次性展示所有操作，讓使用者一次確認。

**信任記憶**：如果使用者對某個技能連續確認了多次，系統可以建議將其加入信任列表。

---

## 技能的啟用/停用管理

### 管理介面

使用者可以透過命令列管理技能的啟用狀態：

```bash
# 列出所有技能
$ openclaw skill list
┌────────────────┬─────────┬──────────┬──────────┐
│ Name           │ Version │ Status   │ Gating   │
├────────────────┼─────────┼──────────┼──────────┤
│ weather        │ 1.0.0   │ enabled  │ none     │
│ calendar       │ 1.2.0   │ enabled  │ confirm  │
│ git-helper     │ 0.9.0   │ enabled  │ confirm  │
│ shell-exec     │ 1.0.0   │ disabled │ approve  │
│ stock-ticker   │ 1.2.0   │ enabled  │ none     │
└────────────────┴─────────┴──────────┴──────────┘

# 停用技能
$ openclaw skill disable shell-exec
✅ shell-exec 已停用

# 啟用技能
$ openclaw skill enable shell-exec
✅ shell-exec 已啟用（門控等級：approve）

# 查看技能詳情
$ openclaw skill info weather
Name:        weather
Version:     1.0.0
Author:      OpenClaw Team
Status:      enabled
Gating:      none
Description: 查詢指定地點的天氣預報
Parameters:  location (required), date, unit, detail
Last Used:   2025-01-16 10:00
Usage Count: 42
```

### 停用的影響

當技能被停用時：
- Brain 的提示詞中不會包含該技能的描述
- LLM 不會生成該技能的調用請求
- 即使 LLM 嘗試調用（幻覺），Skills 模組也會拒絕

### 動態啟停

技能可以根據條件動態啟用/停用：

```yaml
# 動態啟停規則
dynamic_skills:
  # 只在工作時間啟用 Git 技能
  git-helper:
    schedule:
      enabled_hours: "09:00-18:00"
      enabled_days: ["mon", "tue", "wed", "thu", "fri"]
  
  # 只有特定頻道啟用 Shell 技能
  shell-exec:
    channels: ["cli"]  # 只在 CLI 頻道啟用
```

---

## ClawHub 市集簡介

### 什麼是 ClawHub

ClawHub 是 OpenClaw 的官方技能市集，類似於 VS Code 的 Extension Marketplace 或 npm registry。它提供了一個集中的平台，供開發者分享和發現技能。

### 市集功能

```
┌──────────────────────────────────────────────────────┐
│                    ClawHub 市集                        │
│                                                        │
│  🔍 搜尋技能                                          │
│  ┌────────────────────────────────────────────────┐   │
│  │ 搜尋: weather                                   │   │
│  └────────────────────────────────────────────────┘   │
│                                                        │
│  📦 熱門技能                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │ 天氣查詢  │ │ GitHub   │ │ Notion   │             │
│  │ ⭐ 4.8   │ │ ⭐ 4.9   │ │ ⭐ 4.7   │             │
│  │ ↓ 12.3K  │ │ ↓ 8.5K   │ │ ↓ 6.2K   │             │
│  └──────────┘ └──────────┘ └──────────┘             │
│                                                        │
│  🏷️ 分類                                             │
│  [工具] [開發] [生產力] [社交] [媒體] [智慧家居]       │
│                                                        │
│  📊 統計                                              │
│  - 總技能數：487                                       │
│  - 本月新增：23                                        │
│  - 活躍開發者：156                                     │
└──────────────────────────────────────────────────────┘
```

### 市集操作

```bash
# 搜尋技能
$ openclaw skill search "stock"
Found 3 skills:
  1. clawhub/stock-ticker (v1.2.0) - ⭐ 4.5 - 即時股票行情查詢
  2. clawhub/stock-portfolio (v0.8.0) - ⭐ 4.2 - 投資組合追蹤
  3. clawhub/crypto-price (v1.0.0) - ⭐ 4.6 - 加密貨幣價格查詢

# 查看技能詳情
$ openclaw skill search-info clawhub/stock-ticker
Name:        stock-ticker
Version:     1.2.0
Author:      @finance-tools
Stars:       ⭐ 4.5 (128 ratings)
Downloads:   12,345
Description: 查詢即時股票行情，支援台股、美股、港股
Requires:    ALPHA_VANTAGE_API_KEY
Gating:      none
Last Update: 2025-01-10

# 安裝技能
$ openclaw skill install clawhub/stock-ticker
Installing stock-ticker v1.2.0...
✅ Installed successfully
⚠️ Remember to set ALPHA_VANTAGE_API_KEY in your environment

# 更新技能
$ openclaw skill update stock-ticker
Updating stock-ticker: 1.2.0 → 1.3.0
✅ Updated successfully

# 卸載技能
$ openclaw skill uninstall stock-ticker
Uninstalling stock-ticker...
✅ Uninstalled successfully
```

### 安全審核

ClawHub 上的技能在發布前會經過以下審核：

1. **自動化掃描**：檢查腳本中是否有明顯的惡意程式碼
2. **依賴審查**：確認所有依賴都是安全的
3. **門控建議**：根據技能的操作類型建議適當的門控等級
4. **社群審查**：由社群志願者進行程式碼審查
5. **使用者評分**：安裝後的使用者可以評分和留下評價

---

## 技能開發的基本流程

### 步驟概覽

```
1. 確定需求 → 2. 建立目錄 → 3. 撰寫 SKILL.md → 4. 實作腳本
→ 5. 測試 → 6. 安裝 → 7. 使用 → 8.（可選）發布到 ClawHub
```

### 步驟 1：確定需求

明確技能要解決的問題：
- 技能的功能是什麼？
- 需要什麼輸入參數？
- 回傳什麼結果？
- 需要什麼外部依賴？
- 安全風險等級如何？

### 步驟 2：建立目錄

```bash
# 建立技能目錄
mkdir -p ~/.openclaw/skills/my-skill/scripts
mkdir -p ~/.openclaw/skills/my-skill/references
mkdir -p ~/.openclaw/skills/my-skill/tests
```

### 步驟 3：撰寫 SKILL.md

使用前面描述的 SKILL.md 格式撰寫技能定義。重點是：
- 描述要清晰，LLM 會直接閱讀
- 參數要完整，包含類型、必填性和說明
- 使用範例要豐富，幫助 LLM 理解調用方式
- 錯誤處理要明確

### 步驟 4：實作腳本

```bash
#!/bin/bash
# ~/.openclaw/skills/my-skill/scripts/main.sh

# 接收參數
PARAM1="$1"
PARAM2="${2:-default_value}"

# 參數驗證
if [ -z "$PARAM1" ]; then
    echo '{"error": "MISSING_PARAM", "message": "param1 is required"}' >&2
    exit 1
fi

# 執行操作
RESULT=$(some_command "$PARAM1" "$PARAM2")

# 返回結果（JSON 格式）
echo "{\"status\": \"success\", \"data\": \"$RESULT\"}"
```

### 步驟 5：測試

```bash
# 手動測試
$ cd ~/.openclaw/skills/my-skill
$ ./scripts/main.sh "test-param"
{"status": "success", "data": "test-result"}

# 使用測試腳本
$ bash tests/test_basic.sh
✅ 基本功能測試通過
✅ 錯誤處理測試通過
✅ 參數驗證測試通過
```

### 步驟 6：安裝和使用

```bash
# 重新載入技能列表
$ openclaw skill reload
Scanning skills directory...
✅ Found new skill: my-skill (v1.0.0)

# 在對話中使用
User: 幫我使用 my-skill 做某件事
Agent: 好的，我來調用 my-skill... [結果]
```

### 步驟 7：發布（可選）

```bash
# 發布到 ClawHub
$ openclaw skill publish my-skill
Validating SKILL.md... ✅
Checking scripts... ✅
Running security scan... ✅
Uploading to ClawHub...
✅ my-skill v1.0.0 published successfully!
View at: https://clawhub.openclaw.dev/skills/my-skill
```

---

## 與 MCP Tools 的關係

### 什麼是 MCP

MCP（Model Context Protocol）是 Anthropic 提出的一個開放標準，旨在為 LLM 應用提供統一的工具和上下文整合協議。MCP 定義了：
- 工具（Tools）：LLM 可以調用的外部功能
- 資源（Resources）：LLM 可以讀取的外部資料源
- 提示（Prompts）：預定義的提示詞模板

### OpenClaw Skills 與 MCP Tools 的比較

| 維度 | OpenClaw Skills | MCP Tools |
|------|----------------|-----------|
| **定義方式** | Markdown (SKILL.md) | JSON Schema |
| **協議** | WebSocket + OpenClaw 訊息 | MCP 協議（JSON-RPC） |
| **執行方式** | Shell 腳本 | MCP Server |
| **安全門控** | 內建（none/confirm/approve） | 由客戶端實作 |
| **市集** | ClawHub | 各 MCP Server 自行分發 |
| **可讀性** | 高（Markdown） | 中（JSON Schema） |
| **生態系統** | OpenClaw 專屬 | 跨應用（Claude、VS Code 等） |

### 整合方案

OpenClaw 支援將 MCP 工具作為 Skills 使用。透過一個特殊的「MCP 橋接技能」，OpenClaw 可以連接到任何 MCP Server，並將其提供的工具暴露為 OpenClaw Skills：

```
┌──────────────────────────────────────────────────────┐
│                    OpenClaw                            │
│                                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ weather  │  │ calendar │  │  MCP Bridge       │   │
│  │ (native) │  │ (native) │  │  (技能)            │   │
│  └──────────┘  └──────────┘  └────────┬─────────┘   │
│                                        │              │
└────────────────────────────────────────│──────────────┘
                                         │
                              ┌──────────▼──────────┐
                              │   MCP Protocol       │
                              └──────────┬──────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                     │
              ┌─────▼─────┐      ┌──────▼──────┐     ┌──────▼──────┐
              │ MCP Server│      │ MCP Server  │     │ MCP Server  │
              │ (GitHub)  │      │ (Notion)    │     │ (Slack)     │
              └───────────┘      └─────────────┘     └─────────────┘
```

### MCP 整合配置

```yaml
# ~/.openclaw/config/mcp.yaml

mcp_servers:
  - name: "github"
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_TOKEN: "${GITHUB_TOKEN}"
    auto_import: true  # 自動將 MCP 工具匯入為 Skills
    gating: "confirm"   # 所有匯入的工具使用 confirm 門控
    
  - name: "filesystem"
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/docs"]
    auto_import: true
    gating: "confirm"
```

### 何時用 Skills，何時用 MCP

**用 OpenClaw Skills：**
- 簡單的腳本型工具
- 不需要長期運行的服務
- OpenClaw 專屬的功能
- 需要利用 OpenClaw 的記憶和上下文

**用 MCP Tools：**
- 已有 MCP Server 可用的服務
- 需要與其他支援 MCP 的應用共享的工具
- 複雜的、需要持久連線的服務整合
- 社群提供的豐富 MCP Server 生態

**最佳實踐：兩者並用**

在實際使用中，最佳策略是兩者並用：
- 用原生 Skills 實作 OpenClaw 專屬的功能（記憶管理、提醒等）
- 用 MCP 橋接整合第三方服務（GitHub、Notion、Slack 等）
- 對於兩者都能實現的功能，根據複雜度選擇——簡單的用 Skills，複雜的用 MCP

---

## 總結

Skills 模組是 OpenClaw 的「工具箱」，它讓 Agent 從一個只能對話的語言模型，變成了一個能夠真正「做事」的 AI 助理。

### 核心設計要點

1. **Markdown 定義**：使用 SKILL.md 定義技能，人機皆可讀
2. **完整生命週期**：發現 → 註冊 → 調用 → 回傳的標準化流程
3. **安全門控**：none / confirm / approve 三級門控，平衡安全與便利
4. **三層生態**：內建技能 + 社群技能 + 自定義技能
5. **ClawHub 市集**：集中的技能發現和分享平台
6. **MCP 整合**：支援 MCP 協議，接入更廣泛的工具生態
7. **低門檻開發**：任何人都能用 Markdown + Shell 腳本開發新技能

### 延伸閱讀

- [01-系統架構總覽.md](./01-系統架構總覽.md)——回顧 Skills 在整體架構中的位置
- [02-Gateway-中央編排器.md](./02-Gateway-中央編排器.md)——了解 Gateway 如何路由技能調用
- [03-Brain-推理引擎.md](./03-Brain-推理引擎.md)——了解 Brain 在 ReAct 迴圈中如何決定調用技能
- [04-Memory-記憶系統.md](./04-Memory-記憶系統.md)——了解技能如何與記憶系統互動

---

> **上一篇**：[05-Heartbeat-自主排程.md](./05-Heartbeat-自主排程.md)
> **下一篇**：（架構篇完結）
