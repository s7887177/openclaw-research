# 進階 Skills 開發——腳本整合與複雜技能

## 目錄

- [1. 引言](#1-引言)
- [2. scripts/ 目錄深入解析](#2-scripts-目錄深入解析)
  - [2.1 Python 腳本開發](#21-python-腳本開發)
  - [2.2 Bash 腳本開發](#22-bash-腳本開發)
  - [2.3 TypeScript 腳本開發](#23-typescript-腳本開發)
  - [2.4 語言選擇指南](#24-語言選擇指南)
- [3. 在 SKILL.md 中引用腳本](#3-在-skillmd-中引用腳本)
  - [3.1 入口點配置](#31-入口點配置)
  - [3.2 參數傳遞機制](#32-參數傳遞機制)
  - [3.3 結果回傳協議](#33-結果回傳協議)
- [4. 環境變數管理](#4-環境變數管理)
  - [4.1 系統環境變數](#41-系統環境變數)
  - [4.2 技能專用環境變數](#42-技能專用環境變數)
  - [4.3 安全地管理密鑰](#43-安全地管理密鑰)
- [5. API 整合技能開發](#5-api-整合技能開發)
  - [5.1 REST API 整合](#51-rest-api-整合)
  - [5.2 GraphQL API 整合](#52-graphql-api-整合)
  - [5.3 認證機制處理](#53-認證機制處理)
  - [5.4 速率限制與重試](#54-速率限制與重試)
- [6. 瀏覽器自動化技能](#6-瀏覽器自動化技能)
  - [6.1 使用 Playwright](#61-使用-playwright)
  - [6.2 網頁擷取最佳實踐](#62-網頁擷取最佳實踐)
  - [6.3 安全考量](#63-安全考量)
- [7. 多步驟工作流技能](#7-多步驟工作流技能)
  - [7.1 工作流設計模式](#71-工作流設計模式)
  - [7.2 狀態管理](#72-狀態管理)
  - [7.3 錯誤恢復](#73-錯誤恢復)
- [8. 技能間的協作](#8-技能間的協作)
  - [8.1 技能鏈結](#81-技能鏈結)
  - [8.2 資料傳遞](#82-資料傳遞)
  - [8.3 衝突處理](#83-衝突處理)
- [9. 錯誤處理策略](#9-錯誤處理策略)
  - [9.1 錯誤分類](#91-錯誤分類)
  - [9.2 優雅降級](#92-優雅降級)
  - [9.3 錯誤回報格式](#93-錯誤回報格式)
- [10. 效能優化](#10-效能優化)
  - [10.1 快取策略](#101-快取策略)
  - [10.2 並行執行](#102-並行執行)
  - [10.3 延遲載入](#103-延遲載入)
- [11. 安全考量](#11-安全考量)
  - [11.1 輸入消毒](#111-輸入消毒)
  - [11.2 沙盒執行](#112-沙盒執行)
  - [11.3 敏感資料處理](#113-敏感資料處理)
- [12. 實戰案例：完整的 API 整合技能](#12-實戰案例完整的-api-整合技能)
- [13. 總結](#13-總結)

---

## 1. 引言

在前兩章中，我們學會了 Skills 的基礎概念和 SKILL.md 的撰寫技巧。這些知識足以讓你建立純文字技能——完全依靠 LLM 推理能力的技能。然而，當你需要與外部世界互動時——呼叫 API、查詢資料庫、處理文件、自動化瀏覽器操作——你就需要**腳本整合**。

本章將深入探討 OpenClaw 技能的進階開發技巧。我們將學習如何編寫和整合腳本、如何管理環境變數和密鑰、如何設計多步驟工作流、如何讓多個技能協作，以及如何處理錯誤、優化效能和確保安全性。

---

## 2. scripts/ 目錄深入解析

### 2.1 Python 腳本開發

Python 是 OpenClaw 技能腳本的**首選語言**，因為它有豐富的第三方函式庫、簡潔的語法和廣泛的社群支援。

**基本結構**：

```python
#!/usr/bin/env python3
"""
天氣查詢技能腳本
從 OpenWeatherMap API 獲取天氣資訊
"""

import sys
import json
import os
import urllib.request
import urllib.error

def get_weather(location: str, date: str = "today", units: str = "celsius") -> dict:
    """查詢指定城市的天氣"""
    api_key = os.environ.get("WEATHER_API_KEY")
    if not api_key:
        return {"error": "WEATHER_API_KEY not set", "code": "ENV_MISSING"}
    
    unit_map = {"celsius": "metric", "fahrenheit": "imperial"}
    api_units = unit_map.get(units, "metric")
    
    url = (
        f"https://api.openweathermap.org/data/2.5/weather"
        f"?q={location}&appid={api_key}&units={api_units}&lang=zh_tw"
    )
    
    try:
        req = urllib.request.Request(url)
        with urllib.request.urlopen(req, timeout=10) as response:
            data = json.loads(response.read().decode())
        
        return {
            "location": data["name"],
            "temperature": data["main"]["temp"],
            "feels_like": data["main"]["feels_like"],
            "humidity": data["main"]["humidity"],
            "description": data["weather"][0]["description"],
            "wind_speed": data["wind"]["speed"],
        }
    except urllib.error.HTTPError as e:
        if e.code == 404:
            return {"error": f"City '{location}' not found", "code": "NOT_FOUND"}
        return {"error": f"API error: {e.code}", "code": "API_ERROR"}
    except urllib.error.URLError:
        return {"error": "Network error", "code": "NETWORK_ERROR"}
    except Exception as e:
        return {"error": str(e), "code": "UNKNOWN_ERROR"}

def main():
    """主入口點：從 stdin 讀取 JSON 參數，執行操作，輸出結果到 stdout"""
    try:
        input_data = json.loads(sys.stdin.read())
    except json.JSONDecodeError:
        print(json.dumps({"error": "Invalid JSON input", "code": "INVALID_INPUT"}))
        sys.exit(1)
    
    location = input_data.get("location")
    if not location:
        print(json.dumps({"error": "Missing 'location' parameter", "code": "MISSING_PARAM"}))
        sys.exit(1)
    
    result = get_weather(
        location=location,
        date=input_data.get("date", "today"),
        units=input_data.get("units", "celsius"),
    )
    
    print(json.dumps(result, ensure_ascii=False))
    
    if "error" in result:
        sys.exit(1)

if __name__ == "__main__":
    main()
```

**Python 腳本最佳實踐**：

1. **使用 shebang**：`#!/usr/bin/env python3`，確保跨平台相容
2. **依賴管理**：建立 `scripts/requirements.txt`
3. **錯誤處理**：永遠不要讓腳本崩潰——捕獲所有異常並回傳 JSON 錯誤
4. **退出碼**：成功回傳 0，失敗回傳非零
5. **標準輸出**：只在 stdout 輸出 JSON 結果，除錯訊息送到 stderr
6. **超時處理**：設定網路請求的超時時間

**requirements.txt 範例**：

```
requests>=2.28.0
python-dotenv>=1.0.0
pydantic>=2.0.0
```

安裝依賴：

```bash
cd ~/.openclaw/skills/my-skill
pip3 install -r scripts/requirements.txt
```

### 2.2 Bash 腳本開發

Bash 腳本適合簡單的系統操作和命令行工具串接。

```bash
#!/usr/bin/env bash
set -euo pipefail

# 從 stdin 讀取 JSON 參數
INPUT=$(cat)

# 使用 jq 解析參數
ACTION=$(echo "$INPUT" | jq -r '.action')
TARGET=$(echo "$INPUT" | jq -r '.target // ""')

case "$ACTION" in
    "disk_usage")
        RESULT=$(df -h --output=source,size,used,avail,pcent / | tail -1)
        echo "{\"disk_usage\": \"$RESULT\"}"
        ;;
    "memory")
        TOTAL=$(free -m | awk '/^Mem:/{print $2}')
        USED=$(free -m | awk '/^Mem:/{print $3}')
        AVAILABLE=$(free -m | awk '/^Mem:/{print $7}')
        echo "{\"total_mb\": $TOTAL, \"used_mb\": $USED, \"available_mb\": $AVAILABLE}"
        ;;
    "process_list")
        PROCS=$(ps aux --sort=-%mem | head -11 | tail -10 | \
            awk '{printf "{\"pid\":\"%s\",\"cpu\":\"%s\",\"mem\":\"%s\",\"cmd\":\"%s\"},", $2, $3, $4, $11}')
        echo "{\"processes\": [${PROCS%,}]}"
        ;;
    *)
        echo "{\"error\": \"Unknown action: $ACTION\", \"code\": \"UNKNOWN_ACTION\"}" >&2
        exit 1
        ;;
esac
```

**Bash 腳本注意事項**：

- 使用 `set -euo pipefail` 確保錯誤處理
- 使用 `jq` 解析和生成 JSON
- 避免複雜邏輯——如果邏輯複雜，改用 Python
- 注意跨平台差異（macOS vs Linux 的命令選項可能不同）

### 2.3 TypeScript 腳本開發

TypeScript 腳本適合需要型別安全和 Node.js 生態的場景。

```typescript
#!/usr/bin/env npx ts-node
/**
 * 股票價格查詢技能腳本
 */

import * as readline from 'readline';

interface StockInput {
  action: string;
  symbol: string;
  period?: string;
}

interface StockResult {
  symbol: string;
  price: number;
  change: number;
  changePercent: number;
  volume: number;
  timestamp: string;
}

async function getStockPrice(symbol: string): Promise<StockResult> {
  const apiKey = process.env.STOCK_API_KEY;
  if (!apiKey) {
    throw new Error('STOCK_API_KEY not set');
  }

  const url = `https://api.example.com/stock/${symbol}?apikey=${apiKey}`;
  const response = await fetch(url);
  
  if (!response.ok) {
    throw new Error(`API returned ${response.status}`);
  }

  const data = await response.json();
  
  return {
    symbol: data.symbol,
    price: data.price,
    change: data.change,
    changePercent: data.changePercent,
    volume: data.volume,
    timestamp: new Date().toISOString(),
  };
}

async function main() {
  const rl = readline.createInterface({ input: process.stdin });
  let inputData = '';
  
  for await (const line of rl) {
    inputData += line;
  }
  
  try {
    const input: StockInput = JSON.parse(inputData);
    const result = await getStockPrice(input.symbol);
    console.log(JSON.stringify(result));
  } catch (error: any) {
    console.log(JSON.stringify({
      error: error.message,
      code: 'EXECUTION_ERROR'
    }));
    process.exit(1);
  }
}

main();
```

### 2.4 語言選擇指南

| 場景 | 推薦語言 | 原因 |
|------|---------|------|
| API 呼叫 | Python | requests 庫最方便 |
| 系統操作 | Bash | 直接呼叫系統命令 |
| Web 自動化 | Python / TypeScript | Playwright 支援 |
| 資料處理 | Python | pandas, numpy 生態 |
| 文件操作 | Python / Bash | 都很適合 |
| 即時通訊 | TypeScript | WebSocket 原生支援 |
| 簡單工具 | Bash | 快速實現 |
| 複雜邏輯 | Python / TypeScript | 型別安全和可維護性 |

---

## 3. 在 SKILL.md 中引用腳本

### 3.1 入口點配置

SKILL.md 中引用腳本的方式非常直覺——在 Instructions 區段中描述如何呼叫腳本：

```markdown
## Instructions

### 執行方式

本技能使用 `scripts/main.py` 作為入口腳本。

呼叫方式：將 JSON 格式的參數通過 stdin 傳入腳本，
腳本會將結果以 JSON 格式輸出到 stdout。
```

**多入口點設計**：

對於複雜技能，你可以有多個腳本入口點：

```markdown
## Instructions

本技能有多個操作模組：

- **查詢操作**：使用 `scripts/query.py`
- **寫入操作**：使用 `scripts/write.py`
- **分析操作**：使用 `scripts/analyze.py`

每個腳本都接受 JSON 輸入並回傳 JSON 輸出。
```

### 3.2 參數傳遞機制

OpenClaw 支援兩種參數傳遞方式：

**方式一：stdin JSON（推薦）**

```markdown
## Instructions

將參數以 JSON 格式傳入 `scripts/main.py` 的 stdin：

```json
{
  "action": "search",
  "query": "example",
  "limit": 10
}
```
```

對應的 Python 腳本讀取方式：

```python
import sys
import json

input_data = json.loads(sys.stdin.read())
action = input_data["action"]
query = input_data.get("query", "")
```

**方式二：環境變數**

```markdown
## Instructions

參數通過環境變數傳遞：

- `SKILL_ACTION`：操作類型
- `SKILL_QUERY`：搜尋關鍵字
- `SKILL_LIMIT`：結果限制數量
```

對應的腳本：

```python
import os

action = os.environ.get("SKILL_ACTION")
query = os.environ.get("SKILL_QUERY", "")
limit = int(os.environ.get("SKILL_LIMIT", "10"))
```

### 3.3 結果回傳協議

腳本的結果通過 stdout 回傳，必須是有效的 JSON 格式：

**成功回傳**：

```json
{
  "success": true,
  "data": {
    "results": [...],
    "count": 10,
    "page": 1
  }
}
```

**錯誤回傳**：

```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "No results found for query 'xyz'",
    "details": "The search returned 0 results. Try a different query."
  }
}
```

在 SKILL.md 中描述回傳格式，讓 LLM 知道如何解讀：

```markdown
### 回傳格式

成功時：
```json
{
  "success": true,
  "data": { ... }
}
```

如果 `success` 為 `false`，告訴使用者 `error.message` 的內容。
不要顯示 `error.code` 或技術細節。
```

---

## 4. 環境變數管理

### 4.1 系統環境變數

OpenClaw 在執行技能腳本時，會自動注入以下系統環境變數：

| 環境變數 | 說明 |
|----------|------|
| `OPENCLAW_SKILL_NAME` | 當前技能名稱 |
| `OPENCLAW_SKILL_DIR` | 技能目錄的絕對路徑 |
| `OPENCLAW_DATA_DIR` | 技能數據目錄 |
| `OPENCLAW_VERSION` | OpenClaw 版本 |
| `OPENCLAW_USER_ID` | 當前使用者 ID |
| `OPENCLAW_CHANNEL` | 當前通道（discord, telegram 等） |
| `OPENCLAW_LOCALE` | 使用者語言偏好 |

在腳本中使用：

```python
import os

skill_name = os.environ.get("OPENCLAW_SKILL_NAME", "unknown")
data_dir = os.environ.get("OPENCLAW_DATA_DIR", "/data")
channel = os.environ.get("OPENCLAW_CHANNEL", "cli")
```

### 4.2 技能專用環境變數

你可以在 `requires.env` 中聲明技能專用的環境變數：

```yaml
requires:
  env:
    - WEATHER_API_KEY
    - WEATHER_DEFAULT_CITY
    - WEATHER_UNITS
```

**設定方式**：

```bash
# 方式一：.env 文件（推薦）
cat >> ~/.openclaw/.env << 'EOF'
WEATHER_API_KEY=abc123def456
WEATHER_DEFAULT_CITY=Taipei
WEATHER_UNITS=metric
EOF

# 方式二：openclaw.json
cat ~/.openclaw/openclaw.json
{
  "env": {
    "WEATHER_API_KEY": "abc123def456"
  }
}

# 方式三：技能級 .env（只對該技能有效）
cat > ~/.openclaw/skills/weather-query/.env << 'EOF'
WEATHER_API_KEY=abc123def456
EOF
```

**優先級**（由高到低）：

1. 技能目錄中的 `.env`
2. `openclaw.json` 中的 `env` 區段
3. `~/.openclaw/.env`
4. 系統環境變數

### 4.3 安全地管理密鑰

**絕對不要做的事**：

```python
# ❌ 絕對不要把密鑰寫在腳本中
API_KEY = "sk-1234567890abcdef"

# ❌ 絕對不要在錯誤訊息中暴露密鑰
print(f"API call failed with key: {api_key}")

# ❌ 絕對不要把密鑰記錄到日誌
logging.info(f"Using API key: {api_key}")
```

**應該這樣做**：

```python
import os

api_key = os.environ.get("API_KEY")
if not api_key:
    print(json.dumps({
        "error": "API_KEY environment variable is not set",
        "code": "ENV_MISSING"
    }))
    sys.exit(1)

# 在日誌中只顯示部分密鑰
masked_key = f"{api_key[:4]}...{api_key[-4:]}"
print(f"Using API key: {masked_key}", file=sys.stderr)
```

---

## 5. API 整合技能開發

### 5.1 REST API 整合

這是最常見的整合模式。以下是一個完整的 REST API 整合範例：

```python
#!/usr/bin/env python3
"""REST API 整合範例：新聞搜尋"""

import sys
import json
import os
from urllib.request import Request, urlopen
from urllib.error import HTTPError, URLError
from urllib.parse import urlencode

class NewsAPI:
    BASE_URL = "https://newsapi.org/v2"
    
    def __init__(self):
        self.api_key = os.environ.get("NEWS_API_KEY")
        if not self.api_key:
            raise EnvironmentError("NEWS_API_KEY not set")
    
    def _request(self, endpoint: str, params: dict) -> dict:
        params["apiKey"] = self.api_key
        url = f"{self.BASE_URL}/{endpoint}?{urlencode(params)}"
        
        req = Request(url, headers={"User-Agent": "OpenClaw-Skill/1.0"})
        
        try:
            with urlopen(req, timeout=15) as response:
                return json.loads(response.read().decode())
        except HTTPError as e:
            if e.code == 401:
                return {"error": "Invalid API key", "code": "AUTH_ERROR"}
            elif e.code == 429:
                return {"error": "Rate limit exceeded", "code": "RATE_LIMIT"}
            else:
                return {"error": f"HTTP {e.code}", "code": "API_ERROR"}
        except URLError:
            return {"error": "Network error", "code": "NETWORK_ERROR"}
    
    def search(self, query: str, language: str = "zh", page_size: int = 5) -> dict:
        return self._request("everything", {
            "q": query,
            "language": language,
            "pageSize": page_size,
            "sortBy": "publishedAt"
        })
    
    def top_headlines(self, country: str = "tw", category: str = None) -> dict:
        params = {"country": country, "pageSize": 5}
        if category:
            params["category"] = category
        return self._request("top-headlines", params)

def main():
    try:
        input_data = json.loads(sys.stdin.read())
    except json.JSONDecodeError:
        print(json.dumps({"error": "Invalid JSON input"}))
        sys.exit(1)
    
    try:
        api = NewsAPI()
    except EnvironmentError as e:
        print(json.dumps({"error": str(e), "code": "ENV_MISSING"}))
        sys.exit(1)
    
    action = input_data.get("action", "search")
    
    if action == "search":
        result = api.search(
            query=input_data.get("query", ""),
            language=input_data.get("language", "zh"),
            page_size=input_data.get("limit", 5)
        )
    elif action == "headlines":
        result = api.top_headlines(
            country=input_data.get("country", "tw"),
            category=input_data.get("category")
        )
    else:
        result = {"error": f"Unknown action: {action}", "code": "INVALID_ACTION"}
    
    print(json.dumps(result, ensure_ascii=False))

if __name__ == "__main__":
    main()
```

### 5.2 GraphQL API 整合

```python
#!/usr/bin/env python3
"""GraphQL API 整合範例：GitHub"""

import sys
import json
import os
from urllib.request import Request, urlopen

GITHUB_GRAPHQL_URL = "https://api.github.com/graphql"

def graphql_query(query: str, variables: dict = None) -> dict:
    token = os.environ.get("GITHUB_TOKEN")
    if not token:
        return {"error": "GITHUB_TOKEN not set", "code": "ENV_MISSING"}
    
    payload = json.dumps({"query": query, "variables": variables or {}}).encode()
    
    req = Request(GITHUB_GRAPHQL_URL, data=payload, headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    })
    
    with urlopen(req, timeout=30) as response:
        return json.loads(response.read().decode())

def get_repo_issues(owner: str, repo: str, first: int = 10) -> dict:
    query = """
    query($owner: String!, $repo: String!, $first: Int!) {
      repository(owner: $owner, name: $repo) {
        issues(first: $first, orderBy: {field: UPDATED_AT, direction: DESC}) {
          nodes {
            number
            title
            state
            createdAt
            author { login }
            labels(first: 5) { nodes { name } }
          }
          totalCount
        }
      }
    }
    """
    return graphql_query(query, {
        "owner": owner, "repo": repo, "first": first
    })

def main():
    input_data = json.loads(sys.stdin.read())
    action = input_data.get("action")
    
    if action == "issues":
        result = get_repo_issues(
            owner=input_data["owner"],
            repo=input_data["repo"],
            first=input_data.get("limit", 10)
        )
    else:
        result = {"error": f"Unknown action: {action}"}
    
    print(json.dumps(result, ensure_ascii=False))

if __name__ == "__main__":
    main()
```

### 5.3 認證機制處理

不同的 API 使用不同的認證方式，腳本需要正確處理：

```python
class AuthHandler:
    """統一的認證處理器"""
    
    @staticmethod
    def api_key(key_name: str, header_name: str = "X-API-Key"):
        """API Key 認證"""
        key = os.environ.get(key_name)
        return {header_name: key}
    
    @staticmethod
    def bearer_token(token_name: str):
        """Bearer Token 認證"""
        token = os.environ.get(token_name)
        return {"Authorization": f"Bearer {token}"}
    
    @staticmethod
    def basic_auth(user_env: str, pass_env: str):
        """Basic Auth 認證"""
        import base64
        user = os.environ.get(user_env, "")
        password = os.environ.get(pass_env, "")
        credentials = base64.b64encode(f"{user}:{password}".encode()).decode()
        return {"Authorization": f"Basic {credentials}"}
```

### 5.4 速率限制與重試

```python
import time

class RateLimiter:
    """簡易速率限制器"""
    
    def __init__(self, max_calls: int, period: int):
        self.max_calls = max_calls
        self.period = period
        self.calls = []
    
    def wait_if_needed(self):
        now = time.time()
        self.calls = [t for t in self.calls if now - t < self.period]
        
        if len(self.calls) >= self.max_calls:
            sleep_time = self.period - (now - self.calls[0])
            if sleep_time > 0:
                time.sleep(sleep_time)
        
        self.calls.append(time.time())

def retry_with_backoff(func, max_retries=3, base_delay=1):
    """指數退避重試"""
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)
            print(f"Retry {attempt + 1}/{max_retries} after {delay}s: {e}", 
                  file=sys.stderr)
            time.sleep(delay)
```

---

## 6. 瀏覽器自動化技能

### 6.1 使用 Playwright

Playwright 是 OpenClaw 推薦的瀏覽器自動化工具：

```python
#!/usr/bin/env python3
"""瀏覽器自動化技能：網頁截圖"""

import sys
import json
import base64
import asyncio
from playwright.async_api import async_playwright

async def take_screenshot(url: str, full_page: bool = False) -> dict:
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page(viewport={"width": 1280, "height": 720})
        
        try:
            await page.goto(url, wait_until="networkidle", timeout=30000)
            screenshot = await page.screenshot(full_page=full_page)
            title = await page.title()
            
            return {
                "success": True,
                "title": title,
                "url": url,
                "screenshot_base64": base64.b64encode(screenshot).decode(),
            }
        except Exception as e:
            return {"success": False, "error": str(e)}
        finally:
            await browser.close()

async def extract_text(url: str) -> dict:
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()
        
        try:
            await page.goto(url, wait_until="networkidle", timeout=30000)
            text = await page.inner_text("body")
            title = await page.title()
            
            return {
                "success": True,
                "title": title,
                "text": text[:5000],
                "truncated": len(text) > 5000,
            }
        except Exception as e:
            return {"success": False, "error": str(e)}
        finally:
            await browser.close()

def main():
    input_data = json.loads(sys.stdin.read())
    action = input_data.get("action", "screenshot")
    url = input_data.get("url")
    
    if not url:
        print(json.dumps({"error": "Missing URL"}))
        sys.exit(1)
    
    if action == "screenshot":
        result = asyncio.run(take_screenshot(url, input_data.get("full_page", False)))
    elif action == "extract":
        result = asyncio.run(extract_text(url))
    else:
        result = {"error": f"Unknown action: {action}"}
    
    print(json.dumps(result, ensure_ascii=False))

if __name__ == "__main__":
    main()
```

### 6.2 網頁擷取最佳實踐

```python
async def scrape_with_best_practices(url: str) -> dict:
    async with async_playwright() as p:
        browser = await p.chromium.launch(
            headless=True,
            args=[
                "--disable-gpu",
                "--no-sandbox",
                "--disable-dev-shm-usage",
            ]
        )
        
        context = await browser.new_context(
            user_agent="OpenClaw-Bot/1.0 (compatible; +https://openclaw.dev/bot)",
            viewport={"width": 1280, "height": 720},
            locale="zh-TW",
        )
        
        page = await context.new_page()
        
        # 阻擋不必要的資源
        await page.route("**/*.{png,jpg,jpeg,gif,svg,woff,woff2}", 
                         lambda route: route.abort())
        
        try:
            response = await page.goto(url, wait_until="domcontentloaded", timeout=15000)
            
            if response.status >= 400:
                return {"error": f"HTTP {response.status}", "code": "HTTP_ERROR"}
            
            # 等待主要內容載入
            await page.wait_for_selector("main, article, .content", timeout=5000)
            
            # 提取結構化內容
            content = await page.evaluate("""
                () => {
                    const article = document.querySelector('main, article, .content');
                    return article ? article.innerText : document.body.innerText;
                }
            """)
            
            return {"success": True, "content": content[:10000]}
        
        except Exception as e:
            return {"success": False, "error": str(e)}
        finally:
            await browser.close()
```

### 6.3 安全考量

瀏覽器自動化技能有額外的安全風險：

```markdown
## Constraints

### 瀏覽器安全
1. 只存取使用者明確提供的 URL
2. 不允許存取 localhost 或內部網路地址
3. 不允許存取 file:// 協議
4. 不允許執行使用者提供的 JavaScript
5. 設定頁面載入超時（最多 30 秒）
6. 阻擋自動下載
7. 不儲存瀏覽器 cookies 或歷史記錄
```

在腳本中實現 URL 驗證：

```python
from urllib.parse import urlparse
import ipaddress

def validate_url(url: str) -> bool:
    parsed = urlparse(url)
    
    if parsed.scheme not in ("http", "https"):
        return False
    
    hostname = parsed.hostname
    if not hostname:
        return False
    
    # 阻擋本地地址
    blocked = ["localhost", "127.0.0.1", "0.0.0.0", "::1"]
    if hostname in blocked:
        return False
    
    try:
        ip = ipaddress.ip_address(hostname)
        if ip.is_private or ip.is_loopback or ip.is_reserved:
            return False
    except ValueError:
        pass  # 不是 IP 地址，繼續
    
    return True
```

---

## 7. 多步驟工作流技能

### 7.1 工作流設計模式

多步驟工作流讓一個技能能夠完成複雜的任務。在 SKILL.md 中，你可以描述工作流的步驟，讓 LLM 按順序（或條件性地）執行。

**模式一：順序工作流**

```markdown
## Instructions

### 部署工作流

執行部署需要依序完成以下步驟：

1. **檢查**：呼叫 `scripts/check.py` 確認程式碼通過所有測試
2. **建置**：呼叫 `scripts/build.py` 建置生產版本
3. **部署**：呼叫 `scripts/deploy.py` 推送到伺服器
4. **驗證**：呼叫 `scripts/verify.py` 確認部署成功

每個步驟必須在前一步成功後才能執行。
如果任何步驟失敗，停止工作流並回報錯誤。
```

**模式二：條件工作流**

```markdown
### 智慧推薦工作流

根據使用者需求，選擇不同的推薦路徑：

- 如果使用者想要餐廳推薦：
  1. 查詢使用者位置（location-resolver 技能）
  2. 查詢天氣（weather-query 技能）
  3. 根據天氣推薦室內/室外餐廳
  
- 如果使用者想要活動推薦：
  1. 查詢日曆（calendar-manager 技能）
  2. 查詢天氣
  3. 根據空閒時間和天氣推薦活動
```

**模式三：迭代工作流**

```markdown
### 資料收集工作流

收集多頁資料的步驟：

1. 發起第一頁查詢
2. 解析結果，檢查是否有下一頁
3. 如果有下一頁且未達到限制，繼續查詢下一頁
4. 最多查詢 5 頁
5. 匯總所有結果並回傳
```

### 7.2 狀態管理

多步驟工作流需要在步驟之間維持狀態。OpenClaw 提供了兩種機制：

**方式一：腳本內部狀態**

對於單個腳本處理所有步驟的情況，用腳本內的變數管理狀態。

```python
class WorkflowState:
    def __init__(self):
        self.steps_completed = []
        self.data = {}
        self.errors = []
    
    def add_step(self, name: str, result: dict):
        self.steps_completed.append(name)
        self.data[name] = result
    
    def add_error(self, step: str, error: str):
        self.errors.append({"step": step, "error": error})
    
    def to_dict(self):
        return {
            "completed": self.steps_completed,
            "data": self.data,
            "errors": self.errors,
            "success": len(self.errors) == 0
        }
```

**方式二：文件系統狀態**

對於跨次呼叫的狀態保持：

```python
import os
import json

STATE_DIR = os.environ.get("OPENCLAW_DATA_DIR", ".")
STATE_FILE = os.path.join(STATE_DIR, "workflow_state.json")

def save_state(state: dict):
    with open(STATE_FILE, "w") as f:
        json.dump(state, f)

def load_state() -> dict:
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE) as f:
            return json.load(f)
    return {}
```

### 7.3 錯誤恢復

多步驟工作流中，錯誤恢復策略至關重要：

```markdown
### 錯誤恢復策略

如果工作流在某個步驟失敗：

1. **記錄已完成的步驟**
2. **不要自動重試整個工作流**
3. **告訴使用者哪個步驟失敗了**
4. **提供選項**：
   - 重試失敗的步驟
   - 跳過失敗的步驟
   - 取消整個工作流
5. **如果是中間步驟失敗，保留已完成步驟的結果**
```

---

## 8. 技能間的協作

### 8.1 技能鏈結

OpenClaw 的 ReAct 引擎天然支持技能鏈結——一個技能的輸出可以成為另一個技能的輸入。你不需要在代碼中手動實現鏈結；只需要在 SKILL.md 中描述協作方式，LLM 會自動編排。

```markdown
## Instructions

### 與其他技能的協作

本技能可以與以下技能協作：

- **location-resolver**：如果使用者提供的是模糊地點
  （如「附近」、「這裡」），先使用 location-resolver 
  解析出具體的城市名稱，再傳入本技能。

- **unit-converter**：本技能回傳攝氏溫度。如果使用者
  偏好華氏溫度，可以使用 unit-converter 進行轉換。

- **calendar-manager**：如果使用者在規劃行程，
  可以結合日曆資訊和天氣預報，提供更完整的建議。
```

### 8.2 資料傳遞

技能間的資料傳遞由 ReAct 引擎管理。在 SKILL.md 中，你只需要描述輸入和輸出的格式，讓 LLM 負責資料的轉接。

```markdown
### 輸入格式

本技能期望接收以下格式的地點資訊：

```json
{
  "location": "台北市",        // 城市名稱
  "latitude": 25.0330,         // 緯度（選填）
  "longitude": 121.5654        // 經度（選填）
}
```

如果有精確的經緯度（例如從 location-resolver 獲得），
會使用經緯度進行查詢以獲得更準確的結果。
如果只有城市名稱，則使用城市名稱查詢。
```

### 8.3 衝突處理

當多個技能可能回應相同的使用者請求時，需要在 SKILL.md 中明確區分：

```markdown
## Purpose

...

### 與相似技能的區別

- **weather-basic**：只提供當前天氣，沒有預報功能。
  如果使用者只需要「現在」的天氣，weather-basic 更快。
  
- **weather-alerts**：專注於天氣警報和災害預警。
  如果使用者問「會不會有颱風」，應使用 weather-alerts。

本技能（weather-advanced）的定位是**全面的天氣資訊**，
包括當前天氣、未來預報和體感建議。
```

---

## 9. 錯誤處理策略

### 9.1 錯誤分類

```python
class SkillError:
    """技能錯誤分類"""
    
    # 使用者輸入錯誤（可以請使用者修正）
    INVALID_INPUT = "INVALID_INPUT"
    MISSING_PARAM = "MISSING_PARAM"
    
    # 環境錯誤（需要管理員處理）
    ENV_MISSING = "ENV_MISSING"
    BINARY_MISSING = "BINARY_MISSING"
    
    # 外部服務錯誤（重試可能解決）
    API_ERROR = "API_ERROR"
    NETWORK_ERROR = "NETWORK_ERROR"
    RATE_LIMIT = "RATE_LIMIT"
    TIMEOUT = "TIMEOUT"
    
    # 內部錯誤（需要開發者修復）
    INTERNAL_ERROR = "INTERNAL_ERROR"
    PARSE_ERROR = "PARSE_ERROR"
```

### 9.2 優雅降級

在 SKILL.md 中描述降級策略：

```markdown
### 降級策略

如果主要的天氣 API（OpenWeatherMap）不可用：

1. 嘗試備用 API（WeatherAPI.com）
2. 如果備用也不可用，回傳最後的快取資料（如果有）
3. 如果沒有快取，告訴使用者目前無法取得天氣資訊
4. 建議使用者稍後再試，或直接搜尋天氣網站
```

### 9.3 錯誤回報格式

統一的錯誤回報格式讓 LLM 能正確處理和呈現錯誤：

```python
def error_response(code: str, message: str, user_message: str = None) -> dict:
    return {
        "success": False,
        "error": {
            "code": code,
            "message": message,
            "user_message": user_message or message,
        }
    }

# 使用範例
print(json.dumps(error_response(
    code="NOT_FOUND",
    message="City 'Taipe' not found in API database",
    user_message="找不到 'Taipe' 這個城市，你是不是要查 '台北'（Taipei）？"
)))
```

---

## 10. 效能優化

### 10.1 快取策略

```python
import json
import os
import time
import hashlib

CACHE_DIR = os.path.join(
    os.environ.get("OPENCLAW_DATA_DIR", "."), "cache"
)

def get_cache(key: str, ttl: int = 300) -> dict | None:
    cache_file = os.path.join(CACHE_DIR, f"{hashlib.md5(key.encode()).hexdigest()}.json")
    
    if os.path.exists(cache_file):
        with open(cache_file) as f:
            data = json.load(f)
        if time.time() - data["timestamp"] < ttl:
            return data["result"]
    return None

def set_cache(key: str, result: dict):
    os.makedirs(CACHE_DIR, exist_ok=True)
    cache_file = os.path.join(CACHE_DIR, f"{hashlib.md5(key.encode()).hexdigest()}.json")
    
    with open(cache_file, "w") as f:
        json.dump({"timestamp": time.time(), "result": result}, f)
```

### 10.2 並行執行

```python
import asyncio

async def parallel_queries(cities: list) -> list:
    """並行查詢多個城市的天氣"""
    tasks = [get_weather_async(city) for city in cities]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    return [
        r if not isinstance(r, Exception) else {"error": str(r)}
        for r in results
    ]
```

### 10.3 延遲載入

```python
# 延遲載入重量級依賴
_playwright = None

def get_playwright():
    global _playwright
    if _playwright is None:
        from playwright.sync_api import sync_playwright
        _playwright = sync_playwright().start()
    return _playwright
```

---

## 11. 安全考量

### 11.1 輸入消毒

```python
import re
import shlex

def sanitize_input(value: str, max_length: int = 1000) -> str:
    if len(value) > max_length:
        value = value[:max_length]
    # 移除控制字元
    value = re.sub(r'[\x00-\x1f\x7f-\x9f]', '', value)
    return value

def sanitize_sql_input(value: str) -> str:
    """防止 SQL 注入——但更好的方式是使用參數化查詢"""
    dangerous = ["'", '"', ";", "--", "/*", "*/", "\\"]
    for char in dangerous:
        value = value.replace(char, "")
    return value

def sanitize_shell_input(value: str) -> str:
    """安全地處理 shell 輸入"""
    return shlex.quote(value)
```

### 11.2 沙盒執行

OpenClaw 為技能腳本提供了多層沙盒保護：

```markdown
## Notes

### 安全等級

本技能在以下安全約束下執行：

- **文件系統**：只能讀寫 `OPENCLAW_DATA_DIR` 指定的目錄
- **網路**：只允許存取白名單中的域名
- **系統呼叫**：禁止 fork, exec 等危險系統呼叫
- **資源限制**：記憶體上限 512MB，CPU 時間上限 30 秒
```

### 11.3 敏感資料處理

```python
def mask_sensitive(data: dict, sensitive_keys: set = None) -> dict:
    """遮蔽敏感欄位"""
    if sensitive_keys is None:
        sensitive_keys = {"password", "token", "secret", "api_key", "credential"}
    
    masked = {}
    for key, value in data.items():
        if any(s in key.lower() for s in sensitive_keys):
            masked[key] = "***REDACTED***"
        elif isinstance(value, dict):
            masked[key] = mask_sensitive(value, sensitive_keys)
        else:
            masked[key] = value
    return masked
```

---

## 12. 實戰案例：完整的 API 整合技能

讓我們把所有學到的內容整合成一個完整的實戰案例——一個「Notion 頁面管理」技能。

**目錄結構**：

```
notion-manager/
├── SKILL.md
├── scripts/
│   ├── main.py
│   ├── notion_client.py
│   ├── formatters.py
│   └── requirements.txt
├── references/
│   └── notion-api-reference.md
├── tests/
│   ├── test_basic.yaml
│   └── fixtures/
│       └── sample_page.json
└── assets/
    └── icon.png
```

**SKILL.md**：

```markdown
---
name: notion-manager
description: >-
  管理 Notion 工作區中的頁面、資料庫和區塊。支援建立、查詢、
  更新和組織 Notion 內容。
version: 1.0.0
author: openclaw-community
tags:
  - notion
  - productivity
  - note-taking
metadata:
  openclaw:
    controlPolicy: confirm
    requires:
      bins:
        - python3
      env:
        - NOTION_API_KEY
    timeout: 30
    retries: 1
    cache: 60
    category: productivity
---

## Purpose

管理 Notion 工作區中的內容。適用於：
- 在 Notion 中建立新頁面或資料庫項目
- 搜尋 Notion 中的內容
- 更新現有頁面的屬性或內容
- 查看資料庫的資料

不適用於 Notion 工作區管理（權限、成員等）。

## Instructions

使用 `scripts/main.py` 操作 Notion API。

### 搜尋內容
```json
{ "action": "search", "query": "會議紀錄", "limit": 5 }
```

### 建立頁面
```json
{
  "action": "create_page",
  "parent_id": "database-or-page-id",
  "title": "新頁面標題",
  "content": "頁面內容（Markdown 格式）"
}
```

### 更新頁面
```json
{
  "action": "update_page",
  "page_id": "page-id",
  "properties": { "Status": "Done" }
}
```

## Constraints

1. 寫入操作需使用者確認
2. 不刪除頁面（只歸檔）
3. 內容長度限制 10,000 字

## Examples

**使用者**：在 Notion 搜尋上週的會議紀錄
**行動**：search(query="會議紀錄")
**回應**：找到 3 筆會議紀錄：
1. 「週會 3/15」- 2 天前
2. 「產品討論 3/13」- 4 天前
3. 「團隊同步 3/11」- 6 天前
```

---

## 13. 總結

進階 Skills 開發的核心要點：

1. **選擇合適的腳本語言**——Python 最通用，Bash 最簡單，TypeScript 適合 Web 相關
2. **標準化的輸入輸出**——stdin JSON 輸入，stdout JSON 輸出
3. **安全地管理密鑰**——使用環境變數，永不硬編碼
4. **完善的錯誤處理**——分類錯誤，統一格式，優雅降級
5. **效能優化**——合理使用快取、並行和延遲載入
6. **安全至上**——輸入消毒、沙盒執行、敏感資料遮蔽

掌握了這些技巧，你就能建立能夠與任何外部服務互動的強大技能。在下一章，我們將學習如何將你的技能發布到 ClawHub，讓全世界的 OpenClaw 使用者都能使用。
