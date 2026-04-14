# OpenClaw CLI 完整指令參考

> **摘要**：本文件是 OpenClaw 命令列介面（CLI）的完整參考手冊。所有指令按功能分類，包含語法、參數、範例輸出和常用組合。適合開發者隨時查閱。

---

## 目錄

1. [CLI 概覽](#1-cli-概覽)
2. [Gateway 管理](#2-gateway-管理)
3. [Agent 管理](#3-agent-管理)
4. [Skills 管理](#4-skills-管理)
5. [Config 管理](#5-config-管理)
6. [Security 安全指令](#6-security-安全指令)
7. [Tools 工具管理](#7-tools-工具管理)
8. [Channels 通道管理](#8-channels-通道管理)
9. [Pairing 配對管理](#9-pairing-配對管理)
10. [Memory 記憶管理](#10-memory-記憶管理)
11. [Logs 日誌與除錯](#11-logs-日誌與除錯)
12. [Usage 使用統計](#12-usage-使用統計)
13. [Plugin 插件管理](#13-plugin-插件管理)
14. [環境變數](#14-環境變數)
15. [Cheat Sheet 速查表](#15-cheat-sheet-速查表)

---

## 1. CLI 概覽

### 1.1 安裝

```bash
# 使用 npm（推薦）
npm install -g @openclaw/cli

# 使用 cargo
cargo install openclaw-cli

# 使用 Docker
docker pull openclaw/gateway:latest
```

### 1.2 基本語法

```
openclaw <命令群組> <子命令> [選項] [引數]
```

### 1.3 全域選項

| 選項 | 縮寫 | 說明 | 預設值 |
|------|------|------|--------|
| `--config` | `-c` | 指定配置檔路徑 | `./openclaw.json5` |
| `--verbose` | `-v` | 詳細輸出 | `false` |
| `--quiet` | `-q` | 靜默模式 | `false` |
| `--format` | `-f` | 輸出格式 (`text`, `json`, `yaml`) | `text` |
| `--no-color` | | 停用彩色輸出 | `false` |
| `--help` | `-h` | 顯示說明 | |
| `--version` | `-V` | 顯示版本 | |

### 1.4 命令群組概覽

```
openclaw
├── gateway     # Gateway 啟動、停止、狀態
├── agent       # Agent 管理
├── skill       # Skill 開發與管理
├── config      # 配置檔管理
├── security    # 安全檢查與稽核
├── tool        # 工具管理
├── channel     # 通道管理
├── pair        # 配對管理
├── memory      # 記憶管理
├── log         # 日誌查看
├── usage       # 使用統計
├── plugin      # 插件管理
├── run         # 快速推理（互動模式）
├── init        # 初始化專案
└── version     # 版本資訊
```

---

## 2. Gateway 管理

### 2.1 `openclaw gateway start`

啟動 Gateway 服務。

```bash
openclaw gateway start [選項]
```

| 選項 | 縮寫 | 說明 | 預設值 |
|------|------|------|--------|
| `--port` | `-p` | HTTP 埠號 | `3000` |
| `--host` | | 綁定位址 | `0.0.0.0` |
| `--daemon` | `-d` | 後台模式執行 | `false` |
| `--pid-file` | | PID 檔案路徑 | `./openclaw.pid` |
| `--log-file` | | 日誌檔案路徑 | `stdout` |
| `--env` | `-e` | 環境（development/production） | `development` |

```bash
# 範例：前台啟動
$ openclaw gateway start --port 8080

🚀 OpenClaw Gateway starting...
  ✅ Config loaded: ./openclaw.json5
  ✅ 3 providers configured
  ✅ 2 agents loaded
  ✅ 12 tools available
  ✅ 1 channel connected (Discord)
  🌐 HTTP server listening on http://0.0.0.0:8080
  📡 WebSocket server ready
  🎯 Gateway is running! Press Ctrl+C to stop.

# 範例：後台啟動
$ openclaw gateway start --daemon --log-file ./logs/gateway.log
Gateway started in daemon mode (PID: 12345)
```

### 2.2 `openclaw gateway stop`

停止 Gateway 服務。

```bash
openclaw gateway stop [選項]
```

| 選項 | 說明 |
|------|------|
| `--force` | 強制終止（SIGKILL） |
| `--pid-file` | 指定 PID 檔案 |

```bash
$ openclaw gateway stop
Stopping Gateway (PID: 12345)...
✅ Gateway stopped gracefully.
```

### 2.3 `openclaw gateway status`

查看 Gateway 狀態。

```bash
$ openclaw gateway status

OpenClaw Gateway Status
━━━━━━━━━━━━━━━━━━━━━━━
Status:     🟢 Running
PID:        12345
Uptime:     2h 15m 30s
Port:       8080
Memory:     128 MB
CPU:        2.3%

Providers:
  ✅ openai      (gpt-4, gpt-3.5-turbo)
  ✅ ollama      (llama3)
  ⚠️ anthropic   (rate limited)

Agents:
  ✅ default     (openai/gpt-4)
  ✅ social-bot  (openai/gpt-4)

Channels:
  ✅ Discord     (connected, 3 guilds)
  ⏳ Telegram   (connecting...)

Sessions:
  Active:     5
  Total:      127
```

### 2.4 `openclaw gateway reload`

熱重載配置（不中斷服務）。

```bash
$ openclaw gateway reload

🔄 Reloading configuration...
  ✅ Config reloaded
  ✅ Agent 'social-bot' updated
  ⚠️ New channel 'telegram' requires restart
✅ Reload complete (2 changes applied, 1 pending restart)
```

---

## 3. Agent 管理

### 3.1 `openclaw agent list`

列出所有 Agent。

```bash
$ openclaw agent list

┌──────────────┬────────────────┬─────────┬──────────────┐
│ ID           │ Model          │ Status  │ Sessions     │
├──────────────┼────────────────┼─────────┼──────────────┤
│ default      │ openai/gpt-4   │ 🟢 Active │ 3            │
│ social-bot   │ openai/gpt-4   │ 🟢 Active │ 12           │
│ code-helper  │ ollama/llama3  │ 🔴 Stopped│ 0            │
└──────────────┴────────────────┴─────────┴──────────────┘
```

### 3.2 `openclaw agent info`

查看 Agent 詳細資訊。

```bash
$ openclaw agent info social-bot

Agent: social-bot
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Model:          openai/gpt-4
Temperature:    0.8
Max Tokens:     4096
Persona:        ./personas/social-bot/SOUL.md
User Profile:   ./personas/social-bot/USER.md

Allowed Tools (8):
  ✅ search_web
  ✅ memory_read
  ✅ memory_write
  ✅ calendar_read
  ✅ weather_query
  ✅ remind_set
  ✅ note_create
  ✅ translate

Denied Tools (2):
  ❌ exec
  ❌ file_write

Active Sessions: 12
Total Interactions: 1,847
```

### 3.3 `openclaw agent create`

建立新 Agent。

```bash
openclaw agent create <agent-id> [選項]
```

| 選項 | 說明 |
|------|------|
| `--model` | 模型（格式：provider/model） |
| `--persona` | SOUL.md 路徑 |
| `--template` | 使用模板（general, social, code, assistant） |

```bash
$ openclaw agent create my-bot --model openai/gpt-4 --template social

✅ Agent 'my-bot' created
  📁 Persona: ./personas/my-bot/SOUL.md (from template: social)
  📁 Config: ./agents/my-bot.json5
  💡 Edit ./personas/my-bot/SOUL.md to customize personality
```

### 3.4 `openclaw agent delete`

刪除 Agent。

```bash
$ openclaw agent delete my-bot --confirm

⚠️ This will delete agent 'my-bot' and all associated data.
✅ Agent 'my-bot' deleted.
```

---

## 4. Skills 管理

### 4.1 `openclaw skill list`

```bash
$ openclaw skill list

Built-in Skills:
┌──────────────────┬──────────────┬─────────────────────────────┐
│ Name             │ Type         │ Description                 │
├──────────────────┼──────────────┼─────────────────────────────┤
│ search_web       │ built-in     │ 搜尋網路                    │
│ memory_read      │ built-in     │ 讀取記憶                    │
│ memory_write     │ built-in     │ 寫入記憶                    │
│ exec             │ built-in     │ 執行指令                    │
│ file_read        │ built-in     │ 讀取檔案                    │
│ file_write       │ built-in     │ 寫入檔案                    │
└──────────────────┴──────────────┴─────────────────────────────┘

Custom Skills:
┌──────────────────┬──────────────┬─────────────────────────────┐
│ Name             │ Type         │ Description                 │
├──────────────────┼──────────────┼─────────────────────────────┤
│ weather_query    │ custom       │ 查詢天氣（OpenWeatherMap）  │
│ calendar_read    │ custom       │ 讀取 Google Calendar        │
│ remind_set       │ custom       │ 設定提醒                    │
│ translate        │ custom       │ 翻譯文字                    │
└──────────────────┴──────────────┴─────────────────────────────┘
```

### 4.2 `openclaw skill create`

```bash
openclaw skill create <skill-name> [選項]
```

| 選項 | 說明 |
|------|------|
| `--type` | 類型（function, mcp, http） |
| `--template` | 使用模板 |
| `--dir` | 目標目錄 |

```bash
$ openclaw skill create stock_query --type function --template api-call

✅ Skill 'stock_query' created
  📁 ./skills/stock_query/
  ├── index.ts          # 主要邏輯
  ├── schema.json5      # 參數定義
  └── README.md         # 文件

💡 Edit index.ts to implement your skill logic.
```

### 4.3 `openclaw skill test`

```bash
$ openclaw skill test weather_query --args '{"city": "Taipei"}'

Testing skill: weather_query
  Input: {"city": "Taipei"}
  ⏱ Execution time: 234ms
  ✅ Output: {"temp": 28, "condition": "sunny", "humidity": 65}
```

### 4.4 `openclaw skill install`

```bash
$ openclaw skill install @openclaw/skill-google-calendar

📦 Installing @openclaw/skill-google-calendar@1.2.0...
  ✅ Dependencies resolved
  ✅ Skill installed to ./skills/google-calendar/
  💡 Configure API key in openclaw.json5:
     skills.google-calendar.apiKey = "YOUR_KEY"
```

---

## 5. Config 管理

### 5.1 `openclaw config show`

```bash
$ openclaw config show

# 顯示當前完整配置（已解析）
{
  gateway: {
    port: 8080,
    host: "0.0.0.0"
  },
  providers: {
    openai: {
      type: "openai",
      apiKey: "sk-****...****",  // 自動遮罩
      models: ["gpt-4", "gpt-3.5-turbo"]
    }
  },
  // ...
}
```

### 5.2 `openclaw config validate`

```bash
$ openclaw config validate

Validating ./openclaw.json5...
  ✅ Syntax: Valid JSON5
  ✅ Schema: All required fields present
  ⚠️ Warning: Provider 'anthropic' has no API key configured
  ⚠️ Warning: Agent 'code-helper' references undefined tool 'code_review'
  ✅ Permissions: No conflicts detected

Result: 0 errors, 2 warnings
```

### 5.3 `openclaw config init`

```bash
$ openclaw config init

🧙 OpenClaw Configuration Wizard
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

? Choose a template: (Use arrow keys)
❯ Minimal (single agent, basic config)
  Standard (multi-agent, common skills)
  Advanced (full features, all options)
  Custom (start from scratch)

? Primary AI provider:
❯ OpenAI
  Anthropic
  Ollama (local)
  Custom

? Enter OpenAI API key: sk-****

✅ Configuration created: ./openclaw.json5
✅ Persona created: ./personas/default/SOUL.md
💡 Run 'openclaw gateway start' to begin!
```

### 5.4 `openclaw config diff`

```bash
$ openclaw config diff --from ./openclaw.json5.backup

Changes detected:
  + agents.social-bot (new agent added)
  ~ providers.openai.models: ["gpt-4"] → ["gpt-4", "gpt-4-turbo"]
  - tools.denied: ["exec"] (removed)
```

---

## 6. Security 安全指令

### 6.1 `openclaw security audit`

```bash
$ openclaw security audit

🔒 OpenClaw Security Audit
━━━━━━━━━━━━━━━━━━━━━━━━━━

[CRITICAL] 0 issues
[HIGH]     1 issue
[MEDIUM]   2 issues
[LOW]      3 issues

HIGH:
  ❗ Agent 'code-helper' has 'exec' tool without sandbox
     → Recommendation: Enable sandbox or remove exec permission
     → File: openclaw.json5, line 45

MEDIUM:
  ⚠️ No rate limiting configured for provider 'openai'
     → Recommendation: Set max_requests_per_minute
     → File: openclaw.json5, line 12

  ⚠️ Memory store has no encryption at rest
     → Recommendation: Enable encryption in memory.encryption

LOW:
  💡 No log rotation configured
  💡 Default temperature (1.0) may produce inconsistent results
  💡 No backup schedule configured for memory store

Run 'openclaw security fix --auto' to apply recommended fixes.
```

### 6.2 `openclaw security probe`

```bash
$ openclaw security probe tool exec --agent social-bot --provider openai

Permission Probe: tool 'exec'
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Agent:    social-bot
Provider: openai

Layer 1 (Global Settings):          ✅ PASS
Layer 2 (Provider Settings):        ✅ PASS
Layer 3 (Global Tool List):         ✅ PASS (tool in allowed list)
Layer 4 (Provider Tool List):       ✅ PASS (no restriction)
Layer 5 (Agent Permissions):        ❌ FAIL
  → Agent 'social-bot' denied_tools includes 'exec'

Result: DENIED at Layer 5
```

### 6.3 `openclaw security rotate-keys`

```bash
$ openclaw security rotate-keys

🔑 Key Rotation
━━━━━━━━━━━━━━━

? Rotate API keys for which providers?
  [x] openai
  [ ] anthropic
  [ ] ollama

⚠️ This will:
  1. Generate new session encryption keys
  2. Re-encrypt all stored memories
  3. Update API key references

Proceed? (y/N) y

  ✅ Session keys rotated
  ✅ 847 memory entries re-encrypted
  ✅ Provider keys updated
```

### 6.4 `openclaw security scan-prompts`

```bash
$ openclaw security scan-prompts ./personas/

Scanning prompt files for injection risks...

  ✅ ./personas/default/SOUL.md - Clean
  ⚠️ ./personas/social-bot/SOUL.md - Warning
     Line 23: Contains unescaped user interpolation: {{user.name}}
     → Risk: Prompt injection via username
     → Fix: Use sanitized interpolation: {{user.name | sanitize}}

  ✅ ./personas/code-helper/SOUL.md - Clean

Result: 0 critical, 1 warning
```

---

## 7. Tools 工具管理

### 7.1 `openclaw tool list`

```bash
$ openclaw tool list --agent social-bot

Tools available for agent 'social-bot':
┌──────────────────┬──────────┬──────────┬────────────┐
│ Tool             │ Status   │ Sandbox  │ Approval   │
├──────────────────┼──────────┼──────────┼────────────┤
│ search_web       │ ✅ Allowed│ No       │ No         │
│ memory_read      │ ✅ Allowed│ No       │ No         │
│ memory_write     │ ✅ Allowed│ No       │ No         │
│ weather_query    │ ✅ Allowed│ No       │ No         │
│ calendar_read    │ ✅ Allowed│ No       │ No         │
│ remind_set       │ ✅ Allowed│ No       │ No         │
│ exec             │ ❌ Denied │ -        │ -          │
│ file_write       │ ❌ Denied │ -        │ -          │
│ file_read        │ ⚠️ Limited│ Yes      │ No         │
└──────────────────┴──────────┴──────────┴────────────┘
```

### 7.2 `openclaw tool call`

```bash
$ openclaw tool call search_web --args '{"query": "OpenClaw documentation"}' --agent default

Calling tool: search_web
  Agent: default
  Args: {"query": "OpenClaw documentation"}

Result:
  Status: ✅ Success
  Duration: 1.2s
  Output:
    [
      {"title": "OpenClaw Docs", "url": "https://docs.openclaw.dev", "snippet": "..."},
      {"title": "Getting Started", "url": "https://docs.openclaw.dev/start", "snippet": "..."}
    ]
```

### 7.3 `openclaw tool schema`

```bash
$ openclaw tool schema search_web

Tool: search_web
Description: Search the web for information

Parameters:
  query (string, required): The search query
  limit (number, optional): Maximum results (default: 5, max: 20)
  language (string, optional): Result language (default: "en")

Returns:
  Array of {title, url, snippet}
```

---

## 8. Channels 通道管理

### 8.1 `openclaw channel list`

```bash
$ openclaw channel list

┌────────────┬─────────┬───────────────────────────────┐
│ Channel    │ Status  │ Details                       │
├────────────┼─────────┼───────────────────────────────┤
│ discord    │ 🟢 Online│ 3 guilds, 15 channels         │
│ telegram   │ 🟢 Online│ 2 groups, 5 DMs               │
│ matrix     │ 🔴 Offline│ Not configured                 │
│ api        │ 🟢 Online│ HTTP + WebSocket               │
└────────────┴─────────┴───────────────────────────────┘
```

### 8.2 `openclaw channel connect`

```bash
$ openclaw channel connect discord

Connecting to Discord...
  ✅ Bot token verified
  ✅ Connected to 3 guilds
  ✅ Listening on 15 channels

Discord Bot Info:
  Username: MyClaw#1234
  Guilds: My Server, Test Server, Dev Server
```

### 8.3 `openclaw channel disconnect`

```bash
$ openclaw channel disconnect discord

Disconnecting from Discord...
  ✅ Gracefully disconnected
  ℹ️ Active sessions preserved
```

### 8.4 `openclaw channel test`

```bash
$ openclaw channel test discord --message "Hello, this is a test"

Sending test message to Discord...
  Channel: #general (My Server)
  Message: "Hello, this is a test"
  ✅ Message delivered (latency: 45ms)
```

---

## 9. Pairing 配對管理

### 9.1 `openclaw pair start`

啟動 DM 配對模式。

```bash
$ openclaw pair start

🔗 Pairing Mode Active
━━━━━━━━━━━━━━━━━━━━━━

Your pairing code: CLAW-8F3K-X2M9
Expires in: 5 minutes

Send this code to your bot via any connected channel:
  Discord: DM the bot with "!pair CLAW-8F3K-X2M9"
  Telegram: Send "/pair CLAW-8F3K-X2M9"
  API: POST /api/pair with {"code": "CLAW-8F3K-X2M9"}

Waiting for pairing...
```

### 9.2 `openclaw pair list`

```bash
$ openclaw pair list

Active Pairings:
┌────────────┬──────────────┬───────────┬─────────────────┐
│ User       │ Channel      │ Agent     │ Paired At       │
├────────────┼──────────────┼───────────┼─────────────────┤
│ Eason      │ Discord DM   │ default   │ 2024-01-15 10:30│
│ Eason      │ Telegram     │ social-bot│ 2024-01-15 11:00│
└────────────┴──────────────┴───────────┴─────────────────┘
```

### 9.3 `openclaw pair revoke`

```bash
$ openclaw pair revoke --user Eason --channel telegram

⚠️ Revoke pairing for Eason on Telegram? (y/N) y
✅ Pairing revoked. User will need to re-pair.
```

---

## 10. Memory 記憶管理

### 10.1 `openclaw memory list`

```bash
$ openclaw memory list --agent default --limit 10

Recent Memories (default agent):
┌──────┬──────────────────────────────────┬─────────────────┬───────┐
│ ID   │ Content (preview)                │ Created         │ Score │
├──────┼──────────────────────────────────┼─────────────────┼───────┤
│ m001 │ Eason 喜歡用 TypeScript 寫程式   │ 2024-01-15 10:30│ 0.95  │
│ m002 │ Eason 的生日是 3 月 15 日        │ 2024-01-14 15:00│ 0.88  │
│ m003 │ 最近在研究 AI Agent 架構         │ 2024-01-13 20:00│ 0.82  │
│ ...  │ ...                              │ ...             │ ...   │
└──────┴──────────────────────────────────┴─────────────────┴───────┘

Total: 847 memories
```

### 10.2 `openclaw memory search`

```bash
$ openclaw memory search "TypeScript" --agent default

Search Results (query: "TypeScript"):
  1. [0.95] Eason 喜歡用 TypeScript 寫程式，特別是後端開發
  2. [0.82] 最近在用 TypeScript 開發 OpenClaw 插件
  3. [0.71] 推薦了 Zod 作為 TypeScript 的 schema validation 工具
```

### 10.3 `openclaw memory write`

```bash
$ openclaw memory write --agent default --content "Eason 最近對 Rust 很有興趣"

✅ Memory written (ID: m848)
  Agent: default
  Content: "Eason 最近對 Rust 很有興趣"
  Embedding: Generated (1536 dimensions)
```

### 10.4 `openclaw memory delete`

```bash
$ openclaw memory delete m003 --confirm

⚠️ Delete memory m003? This cannot be undone.
✅ Memory m003 deleted.
```

### 10.5 `openclaw memory export / import`

```bash
# 匯出
$ openclaw memory export --agent default --format json > memories.json
✅ Exported 847 memories to stdout

# 匯入
$ openclaw memory import --agent default --file memories.json
✅ Imported 847 memories (12 duplicates skipped)
```

---

## 11. Logs 日誌與除錯

### 11.1 `openclaw log tail`

```bash
$ openclaw log tail --follow

2024-01-15T10:30:00.000Z [INFO]  inference:start session=abc123 agent=default
2024-01-15T10:30:00.150Z [DEBUG] context:assembly tokens=3200 memories=5
2024-01-15T10:30:01.200Z [INFO]  model:response tokens=150 finish=tool_calls
2024-01-15T10:30:01.250Z [DEBUG] tool:call name=search_web args={"query":"..."}
2024-01-15T10:30:02.500Z [INFO]  tool:result status=success duration=1250ms
2024-01-15T10:30:03.800Z [INFO]  model:response tokens=200 finish=stop
2024-01-15T10:30:03.850Z [INFO]  inference:end total_tokens=350 duration=3850ms
```

### 11.2 `openclaw log search`

```bash
$ openclaw log search --level error --since "1 hour ago"

Error logs (last 1 hour):
  2024-01-15T10:15:00Z [ERROR] provider:openai rate_limit_exceeded
    → Details: 429 Too Many Requests, retry after 30s
  2024-01-15T09:45:00Z [ERROR] tool:exec timeout after 60000ms
    → Details: Command 'npm install' exceeded timeout
```

### 11.3 `openclaw log export`

```bash
$ openclaw log export --since "2024-01-01" --format json > logs.json
✅ Exported 12,450 log entries
```

---

## 12. Usage 使用統計

### 12.1 `openclaw usage show`

```bash
$ openclaw usage show --period today

📊 Usage Report (Today)
━━━━━━━━━━━━━━━━━━━━━━━━━━━

Total Requests:    127
Total Tokens:      45,230
  Prompt:          32,100
  Completion:      13,130

By Provider:
  openai:          120 requests (42,000 tokens)
  ollama:          7 requests (3,230 tokens)

By Agent:
  default:         85 requests
  social-bot:      42 requests

Tool Calls:        89
  search_web:      34
  memory_read:     28
  memory_write:    15
  weather_query:   12
```

### 12.2 `openclaw usage cost`

```bash
$ openclaw usage cost --period month

💰 Cost Estimate (January 2024)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Provider: openai
  gpt-4:          1,200,000 tokens → $36.00
  gpt-3.5-turbo:  500,000 tokens  → $0.75

Provider: ollama (local)
  llama3:         300,000 tokens   → $0.00

Total Estimated Cost: $36.75
Daily Average:        $1.19
```

---

## 13. Plugin 插件管理

### 13.1 `openclaw plugin list`

```bash
$ openclaw plugin list

Installed Plugins:
┌───────────────────────────────┬─────────┬─────────────────────────┐
│ Plugin                        │ Version │ Description             │
├───────────────────────────────┼─────────┼─────────────────────────┤
│ @openclaw/plugin-discord      │ 1.2.0   │ Discord 通道支援        │
│ @openclaw/plugin-telegram     │ 1.1.0   │ Telegram 通道支援       │
│ @openclaw/plugin-memory-pg    │ 0.9.0   │ PostgreSQL 記憶後端     │
│ @openclaw/plugin-tts-azure    │ 1.0.0   │ Azure TTS 語音合成      │
└───────────────────────────────┴─────────┴─────────────────────────┘
```

### 13.2 `openclaw plugin install / uninstall`

```bash
# 安裝
$ openclaw plugin install @openclaw/plugin-matrix
📦 Installing @openclaw/plugin-matrix@1.0.0...
✅ Plugin installed. Configure in openclaw.json5.

# 移除
$ openclaw plugin uninstall @openclaw/plugin-matrix
✅ Plugin uninstalled.
```

---

## 14. 環境變數

| 變數 | 說明 | 預設值 |
|------|------|--------|
| `OPENCLAW_CONFIG` | 配置檔路徑 | `./openclaw.json5` |
| `OPENCLAW_ENV` | 執行環境 | `development` |
| `OPENCLAW_LOG_LEVEL` | 日誌層級 | `info` |
| `OPENCLAW_PORT` | 服務埠號 | `3000` |
| `OPENCLAW_HOST` | 綁定位址 | `0.0.0.0` |
| `OPENCLAW_DATA_DIR` | 資料目錄 | `./data` |
| `OPENCLAW_PLUGIN_DIR` | 插件目錄 | `./plugins` |
| `OPENAI_API_KEY` | OpenAI API 金鑰 | - |
| `ANTHROPIC_API_KEY` | Anthropic API 金鑰 | - |
| `DISCORD_BOT_TOKEN` | Discord Bot Token | - |
| `TELEGRAM_BOT_TOKEN` | Telegram Bot Token | - |

---

## 15. Cheat Sheet 速查表

### 15.1 快速入門

```bash
# 初始化專案
openclaw config init

# 啟動 Gateway
openclaw gateway start

# 互動模式（直接在終端對話）
openclaw run

# 查看狀態
openclaw gateway status
```

### 15.2 日常管理

```bash
# 查看所有 Agent
openclaw agent list

# 查看工具列表
openclaw tool list --agent <agent-id>

# 搜尋記憶
openclaw memory search "關鍵字"

# 查看日誌
openclaw log tail --follow

# 使用統計
openclaw usage show --period today
openclaw usage cost --period month
```

### 15.3 安全與除錯

```bash
# 安全稽核
openclaw security audit

# 權限探查
openclaw security probe tool <tool-name> --agent <agent-id>

# 測試工具
openclaw tool call <tool-name> --args '{"key": "value"}' --agent <agent-id>

# 測試 Skill
openclaw skill test <skill-name> --args '{"key": "value"}'

# 驗證配置
openclaw config validate
```

### 15.4 開發與擴展

```bash
# 建立新 Skill
openclaw skill create <name> --type function

# 安裝插件
openclaw plugin install <package>

# 配對 DM
openclaw pair start

# 匯出記憶
openclaw memory export --agent <agent-id> --format json > backup.json
```

---

## 附錄：指令索引（字母排序）

| 指令 | 說明 |
|------|------|
| `openclaw agent create` | 建立 Agent |
| `openclaw agent delete` | 刪除 Agent |
| `openclaw agent info` | Agent 詳細資訊 |
| `openclaw agent list` | 列出 Agent |
| `openclaw channel connect` | 連接通道 |
| `openclaw channel disconnect` | 斷開通道 |
| `openclaw channel list` | 列出通道 |
| `openclaw channel test` | 測試通道 |
| `openclaw config diff` | 配置差異比較 |
| `openclaw config init` | 初始化配置 |
| `openclaw config show` | 顯示配置 |
| `openclaw config validate` | 驗證配置 |
| `openclaw gateway reload` | 熱重載配置 |
| `openclaw gateway start` | 啟動 Gateway |
| `openclaw gateway status` | 查看狀態 |
| `openclaw gateway stop` | 停止 Gateway |
| `openclaw log export` | 匯出日誌 |
| `openclaw log search` | 搜尋日誌 |
| `openclaw log tail` | 即時日誌 |
| `openclaw memory delete` | 刪除記憶 |
| `openclaw memory export` | 匯出記憶 |
| `openclaw memory import` | 匯入記憶 |
| `openclaw memory list` | 列出記憶 |
| `openclaw memory search` | 搜尋記憶 |
| `openclaw memory write` | 寫入記憶 |
| `openclaw pair list` | 列出配對 |
| `openclaw pair revoke` | 撤銷配對 |
| `openclaw pair start` | 啟動配對 |
| `openclaw plugin install` | 安裝插件 |
| `openclaw plugin list` | 列出插件 |
| `openclaw plugin uninstall` | 移除插件 |
| `openclaw run` | 互動模式 |
| `openclaw security audit` | 安全稽核 |
| `openclaw security probe` | 權限探查 |
| `openclaw security rotate-keys` | 金鑰輪換 |
| `openclaw security scan-prompts` | 掃描提示注入 |
| `openclaw skill create` | 建立 Skill |
| `openclaw skill install` | 安裝 Skill |
| `openclaw skill list` | 列出 Skill |
| `openclaw skill test` | 測試 Skill |
| `openclaw tool call` | 調用工具 |
| `openclaw tool list` | 列出工具 |
| `openclaw tool schema` | 工具 Schema |
| `openclaw usage cost` | 成本估算 |
| `openclaw usage show` | 使用統計 |

---

> **下一章**：[Discord 語音機器人專案](../12-實戰應用篇/01-Discord語音機器人專案.md)
