# npm 安裝指南——開發者快速上手

> **文件版本**：v1.0
> **適用 OpenClaw 版本**：0.9.x — 1.x
> **最後更新**：2025-07
> **前置條件**：請先完成 [系統需求與前置條件](01-系統需求與前置條件.md) 中的所有檢查

npm 安裝方式是 OpenClaw 最快的上手途徑。只需一行指令即可完成安裝，非常適合開發者快速體驗和本地開發。本指南將詳細介紹從安裝到日常使用的完整流程。

---

## 目錄

1. [npm 安裝的適用場景](#1-npm-安裝的適用場景)
2. [全域安裝 OpenClaw](#2-全域安裝-openclaw)
3. [初始化與啟動精靈](#3-初始化與啟動精靈)
4. [配置檔案位置（~/.openclaw/）](#4-配置檔案位置openclaw)
5. [手動啟動與常駐程式](#5-手動啟動與常駐程式)
6. [openclaw onboard --install-daemon 說明](#6-openclaw-onboard---install-daemon-說明)
7. [開發模式運行](#7-開發模式運行)
8. [從 npm 切換到 Docker](#8-從-npm-切換到-docker)
9. [常見問題排解](#9-常見問題排解)

---

## 1. npm 安裝的適用場景

### 1.1 何時選擇 npm 安裝

npm 安裝方式並非萬能，它在特定場景下才是最佳選擇。在決定使用 npm 安裝之前，請先了解它的優勢和限制。

#### 推薦使用 npm 安裝的場景

| 場景 | 說明 |
|------|------|
| **快速原型驗證** | 想快速體驗 OpenClaw 的功能，5 分鐘內完成安裝 |
| **本地開發** | 開發自訂技能（Skills）或外掛（Plugins） |
| **單人使用** | 個人助手，不需要高可用性 |
| **CI/CD 整合** | 在 CI 環境中測試 OpenClaw 相關功能 |
| **學習與實驗** | 學習 OpenClaw 的架構和功能 |
| **輕量環境** | 在資源受限的環境中運行（如 512 MB RAM） |
| **腳本整合** | 將 OpenClaw 作為腳本或工具鏈的一部分 |

#### 不建議使用 npm 安裝的場景

| 場景 | 原因 | 替代方案 |
|------|------|----------|
| **生產環境** | 缺少容器隔離和資源管理 | 使用 Docker |
| **多實例部署** | npm 全域安裝只有一個實例 | 使用 Docker |
| **需要沙箱** | npm 安裝的沙箱功能受限 | 使用 Docker |
| **團隊共用** | 難以標準化環境 | 使用 Docker |
| **高可用需求** | 缺少自動重啟和健康檢查 | 使用 Docker + 系統服務 |

### 1.2 npm vs Docker 詳細比較

| 面向 | npm 安裝 | Docker 安裝 |
|------|----------|-------------|
| **安裝時間** | ~2 分鐘 | ~5-10 分鐘 |
| **安裝指令** | `npm i -g openclaw` | `docker compose up -d` |
| **磁碟空間** | ~200 MB | ~1-2 GB |
| **記憶體佔用** | ~200-400 MB | ~400-800 MB |
| **環境隔離** | ❌ 與系統共享 | ✅ 完全隔離 |
| **升級方式** | `npm update -g openclaw` | `docker compose pull && up -d` |
| **回滾能力** | ⚠️ 有限（npm 可指定版本） | ✅ 映像標籤回滾 |
| **日誌管理** | 需手動配置 | 內建日誌輪替 |
| **程序管理** | 需 systemd/pm2 | Docker 自動重啟 |
| **沙箱** | ⚠️ 僅 local 模式 | ✅ Docker 沙箱 |
| **多實例** | ❌ 困難 | ✅ 簡單 |
| **開發調試** | ✅ 直接調試 | ⚠️ 需額外步驟 |
| **適合場景** | 開發、學習、個人使用 | 生產、團隊、正式部署 |

### 1.3 npm 安裝的架構圖

```
npm 全域安裝架構：

 ┌─────────────────────────────────────────────┐
 │               宿主機作業系統                   │
 │                                              │
 │  ┌────────────────────────────────────────┐ │
 │  │          Node.js 執行環境               │ │
 │  │                                        │ │
 │  │  ┌──────────────────────────────────┐  │ │
 │  │  │        OpenClaw 核心程式           │  │ │
 │  │  │                                  │  │ │
 │  │  │  ┌──────┐ ┌──────┐ ┌──────┐    │  │ │
 │  │  │  │Gateway│ │Agents│ │Memory│    │  │ │
 │  │  │  └──────┘ └──────┘ └──────┘    │  │ │
 │  │  │  ┌──────┐ ┌──────┐ ┌──────┐    │  │ │
 │  │  │  │Skills│ │Sandbox│ │ Tools│    │  │ │
 │  │  │  └──────┘ └──────┘ └──────┘    │  │ │
 │  │  └──────────────────────────────────┘  │ │
 │  └────────────────────────────────────────┘ │
 │                                              │
 │  設定目錄: ~/.openclaw/                       │
 │  資料目錄: ~/.openclaw/data/                  │
 │  日誌目錄: ~/.openclaw/logs/                  │
 └─────────────────────────────────────────────┘
```

---

## 2. 全域安裝 OpenClaw

### 2.1 前置確認

在安裝之前，請確認您的環境已準備就緒：

```bash
# 確認 Node.js 版本（需要 22+，建議 24+）
node --version
# 期望: v22.x.x 或 v24.x.x

# 確認 npm 版本（需要 9+）
npm --version
# 期望: 9.x.x 或 10.x.x

# 確認網路連線
curl -s https://registry.npmjs.org/openclaw | head -1
# 期望: 返回 JSON 資料
```

### 2.2 安裝指令

```bash
# 方法 1：標準全域安裝（推薦）
npm install -g openclaw

# 方法 2：指定版本安裝
npm install -g openclaw@1.2.3

# 方法 3：安裝最新 beta 版（測試用）
npm install -g openclaw@beta

# 方法 4：安裝最新開發版（不建議）
npm install -g openclaw@next
```

### 2.3 安裝過程詳解

```bash
$ npm install -g openclaw

# 安裝過程輸出範例：
npm warn deprecated some-old-package@1.0.0: This package is no longer maintained
added 342 packages in 45s

# 安裝完成後的全域套件位置
npm root -g
# Linux/macOS: /usr/local/lib/node_modules（系統安裝）
# 或: ~/.npm-global/lib/node_modules（使用者安裝）
# nvm: ~/.nvm/versions/node/v24.x.x/lib/node_modules

# 確認安裝位置
which openclaw
# 期望: /usr/local/bin/openclaw 或類似路徑
```

### 2.4 處理權限問題

如果遇到 `EACCES` 權限錯誤，有以下解決方案：

```bash
# 方案 A：使用 nvm（最推薦）
# nvm 安裝的 Node.js 不需要 sudo
nvm install 24
npm install -g openclaw

# 方案 B：設定 npm 全域目錄
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
npm install -g openclaw

# 方案 C：使用 sudo（不推薦，可能造成後續權限問題）
sudo npm install -g openclaw

# 方案 D：修改 npm 全域目錄權限（不推薦）
sudo chown -R $(whoami) $(npm config get prefix)/{lib/node_modules,bin,share}
npm install -g openclaw
```

### 2.5 驗證安裝

```bash
# 確認 openclaw 指令可用
openclaw --version
# 期望輸出: openclaw v1.2.3

# 查看可用指令
openclaw --help

# 輸出範例:
# 🐾 OpenClaw CLI v1.2.3
#
# Usage: openclaw <command> [options]
#
# Commands:
#   openclaw start              Start OpenClaw service
#   openclaw stop               Stop OpenClaw service
#   openclaw status             Show service status
#   openclaw onboard            Interactive setup wizard
#   openclaw config             Configuration management
#   openclaw doctor             Run diagnostics
#   openclaw logs               View logs
#   openclaw backup             Backup management
#   openclaw skill              Skill management
#   openclaw channel            Channel management
#   openclaw model              Model management
#   openclaw update             Update OpenClaw
#
# Options:
#   --version, -v    Show version
#   --help, -h       Show help
#   --verbose        Verbose output
#   --config <path>  Custom config file path
```

### 2.6 一鍵安裝腳本

如果您想要更自動化的安裝流程：

```bash
#!/usr/bin/env bash
# install-openclaw.sh — OpenClaw npm 一鍵安裝腳本

set -euo pipefail

echo "🐾 OpenClaw npm 安裝程式"
echo "========================"

# 檢查 Node.js
if ! command -v node &>/dev/null; then
  echo "❌ Node.js 未安裝"
  echo "請先安裝 Node.js 22+（建議使用 nvm）："
  echo "  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash"
  echo "  nvm install 24"
  exit 1
fi

NODE_MAJOR=$(node --version | sed 's/v//' | cut -d. -f1)
if [ "$NODE_MAJOR" -lt 22 ]; then
  echo "❌ Node.js 版本太低: $(node --version)"
  echo "需要 Node.js 22+，請升級"
  exit 1
fi
echo "✅ Node.js $(node --version)"

# 檢查 npm
NPM_MAJOR=$(npm --version | cut -d. -f1)
if [ "$NPM_MAJOR" -lt 9 ]; then
  echo "⚠️ npm 版本較低，正在升級..."
  npm install -g npm@latest
fi
echo "✅ npm $(npm --version)"

# 安裝 OpenClaw
echo ""
echo "📦 安裝 OpenClaw..."
npm install -g openclaw

# 驗證
echo ""
echo "✅ 安裝完成！"
openclaw --version
echo ""
echo "接下來執行初始化精靈："
echo "  openclaw onboard"
```

---

## 3. 初始化與啟動精靈

### 3.1 啟動 onboard 精靈

安裝完成後，使用 `openclaw onboard` 進行互動式初始化設定。這是設定 OpenClaw 的最簡單方式。

```bash
# 啟動互動式設定精靈
openclaw onboard
```

### 3.2 Onboard 精靈完整流程

```
$ openclaw onboard

  🐾 Welcome to OpenClaw!
  Let's set up your AI assistant.

  ┌──────────────────────────────────┐
  │  OpenClaw Onboard Wizard v1.2.3 │
  └──────────────────────────────────┘

═══════════════════════════════════════
  Step 1 of 7: 🏷️  Bot Identity
═══════════════════════════════════════

  ? What's your bot's name? (My OpenClaw)
  > 小爪助手

  ? Choose a theme for your bot:
    ❯ friendly    — 友善且樂於助人
      professional — 專業且精確
      casual      — 隨性且有趣
      custom      — 自訂主題

  ? Choose an emoji for your bot: (🐾)
  > 🐾

═══════════════════════════════════════
  Step 2 of 7: 🤖 AI Model
═══════════════════════════════════════

  ? Select your primary AI provider:
    ❯ OpenAI (GPT-4o, GPT-4o-mini)
      Anthropic (Claude Sonnet, Haiku)
      Google (Gemini 2.0 Flash, Pro)
      Ollama (Local models - no API key needed)
      OpenRouter (Multiple providers)
      Skip for now

  ? Enter your OpenAI API Key:
  > sk-proj-xxxxxxxxxxxxxxxxxxxx

  ? Testing API Key...
    ✅ Connected! Available models: gpt-4o, gpt-4o-mini, gpt-o3-mini

  ? Select default model:
    ❯ gpt-4o-mini  ($0.15/1M input, $0.60/1M output) — Best value
      gpt-4o       ($2.50/1M input, $10.00/1M output) — Best quality
      gpt-o3-mini  ($1.10/1M input, $4.40/1M output) — Best reasoning

  ? Add a fallback model? (Y/n)
  > Y

  ? Enter Anthropic API Key for fallback:
  > sk-ant-xxxxxxxxxxxxxxxxxxxx
    ✅ Fallback model: claude-sonnet-4-20250514

═══════════════════════════════════════
  Step 3 of 7: 💬 Communication Channel
═══════════════════════════════════════

  ? Select your primary channel:
    ❯ Telegram    — Easy setup, great for personal use
      WhatsApp    — Business API required
      Discord     — Perfect for communities
      Slack       — Ideal for teams
      HTTP API    — REST API only, no messaging app
      Skip for now

  ? Enter your Telegram Bot Token:
    (Get one from @BotFather on Telegram)
  > 1234567890:ABCdefGhIJKlmNoPQRsTUVwxYZ

  ? Testing Telegram connection...
    ✅ Connected! Bot: @MyOpenClawBot

  ? Set up Webhook automatically? (Y/n)
  > Y

  ? Enter your public URL for webhooks:
    (Must be HTTPS, e.g., https://bot.example.com)
  > https://openclaw.example.com

    ✅ Webhook configured: https://openclaw.example.com/webhook/telegram

═══════════════════════════════════════
  Step 4 of 7: 🔐 Security
═══════════════════════════════════════

  ? Set a password for the Control UI:
  > ********

  ? Enable sandbox (code execution)? (y/N)
  > n
    ℹ️  Sandbox disabled. You can enable it later in openclaw.json5.

  ✅ Encryption key generated
  ✅ JWT secret generated

═══════════════════════════════════════
  Step 5 of 7: 🌐 Gateway
═══════════════════════════════════════

  ? Gateway port: (3000)
  > 3000

  ? Enable Control UI? (Y/n)
  > Y

    ✅ Control UI will be available at http://localhost:3000/ui

═══════════════════════════════════════
  Step 6 of 7: 🔍 Web Search (Optional)
═══════════════════════════════════════

  ? Enable web search capability? (y/N)
  > y

  ? Select search provider:
    ❯ Tavily (Recommended, AI-optimized)
      SerpAPI
      Skip

  ? Enter Tavily API Key:
  > tvly-xxxxxxxxxxxxxxxxxxxx
    ✅ Web search enabled

═══════════════════════════════════════
  Step 7 of 7: 📋 Review & Apply
═══════════════════════════════════════

  Configuration Summary:
  ┌─────────────────────────────────────────┐
  │ Name:      小爪助手 🐾                    │
  │ Theme:     friendly                      │
  │ Model:     openai/gpt-4o-mini            │
  │ Fallback:  anthropic/claude-sonnet-4-20250514  │
  │ Channel:   Telegram (@MyOpenClawBot)     │
  │ Gateway:   :3000                         │
  │ Sandbox:   Disabled                      │
  │ Search:    Tavily                        │
  │ Config:    ~/.openclaw/openclaw.json5    │
  └─────────────────────────────────────────┘

  ? Apply configuration and start? (Y/n)
  > Y

  ⏳ Writing configuration...
  ✅ Configuration saved to ~/.openclaw/openclaw.json5

  ⏳ Starting OpenClaw...
  ✅ OpenClaw is running!

  ┌─────────────────────────────────────────┐
  │  🎉 Setup Complete!                     │
  │                                          │
  │  Control UI: http://localhost:3000/ui    │
  │  Telegram:   @MyOpenClawBot             │
  │                                          │
  │  Try sending a message to your bot!     │
  │                                          │
  │  Useful commands:                        │
  │    openclaw status  — Check status       │
  │    openclaw logs    — View logs          │
  │    openclaw stop    — Stop service       │
  │    openclaw doctor  — Run diagnostics    │
  └─────────────────────────────────────────┘
```

### 3.3 onboard 命令列參數

```bash
# 查看所有 onboard 選項
openclaw onboard --help

# 靜默模式（使用已有配置或預設值）
openclaw onboard --silent

# 指定配置檔路徑
openclaw onboard --config /path/to/openclaw.json5

# 僅設定特定部分
openclaw onboard --only model     # 僅設定模型
openclaw onboard --only channel   # 僅設定通道
openclaw onboard --only identity  # 僅設定身分

# 重置配置並重新設定
openclaw onboard --reset

# 跳過啟動
openclaw onboard --no-start

# 安裝為系統服務（daemon）
openclaw onboard --install-daemon

# 組合使用
openclaw onboard --install-daemon --no-start
```

### 3.4 非互動式初始化

如果您在 CI/CD 環境或腳本中使用，可以使用非互動式方式初始化：

```bash
# 使用環境變數進行初始化
export OPENAI_API_KEY="sk-proj-xxxxx"
export TELEGRAM_BOT_TOKEN="1234567890:ABCdef..."
export GATEWAY_BASE_URL="https://openclaw.example.com"

openclaw onboard --silent \
  --name "CI Bot" \
  --model "openai/gpt-4o-mini" \
  --channel telegram \
  --no-start

# 或直接建立配置檔
openclaw config init \
  --name "CI Bot" \
  --model "openai/gpt-4o-mini"
```

---

## 4. 配置檔案位置（~/.openclaw/）

### 4.1 目錄結構

OpenClaw 的 npm 安裝將所有配置和資料存放在使用者的家目錄下：

```
~/.openclaw/                    # OpenClaw 主目錄
├── openclaw.json5              # 主配置檔
├── .env                        # 環境變數（API Keys 等）
├── data/                       # 資料目錄
│   ├── db/                     # 資料庫
│   │   ├── openclaw.db         # 主資料庫（SQLite）
│   │   ├── conversations.db    # 對話記錄
│   │   └── memory.db           # 記憶資料庫
│   ├── media/                  # 媒體檔案（圖片、語音等）
│   │   ├── received/           # 接收的檔案
│   │   └── generated/          # AI 生成的檔案
│   └── skills/                 # 使用者安裝的技能
│       ├── my-custom-skill/
│       └── another-skill/
├── logs/                       # 日誌目錄
│   ├── openclaw.log            # 主日誌
│   ├── openclaw-error.log      # 錯誤日誌
│   ├── access.log              # HTTP 存取日誌
│   └── webhook.log             # Webhook 日誌
├── backups/                    # 備份目錄
│   ├── auto-20250115-030000/
│   └── manual-pre-upgrade/
├── cache/                      # 快取目錄
│   ├── models/                 # 模型快取
│   └── web/                    # 網頁搜尋快取
├── plugins/                    # 外掛目錄
└── tmp/                        # 臨時檔案
```

### 4.2 各檔案與目錄說明

| 路徑 | 說明 | 可刪除 | 備份重要性 |
|------|------|--------|-----------|
| `openclaw.json5` | 主配置檔，定義所有行為 | ❌ | ⭐⭐⭐⭐⭐ |
| `.env` | 環境變數，存放 API Keys | ❌ | ⭐⭐⭐⭐⭐ |
| `data/db/` | 資料庫，包含對話和記憶 | ❌ | ⭐⭐⭐⭐⭐ |
| `data/media/` | 使用者上傳和 AI 生成的媒體 | ⚠️ | ⭐⭐⭐ |
| `data/skills/` | 自訂技能 | ❌ | ⭐⭐⭐⭐ |
| `logs/` | 運行日誌 | ✅ | ⭐ |
| `backups/` | 備份檔案 | ⚠️ | ⭐⭐ |
| `cache/` | 快取檔案 | ✅ | ⭐ |
| `tmp/` | 臨時檔案 | ✅ | — |

### 4.3 自訂配置目錄

如果您不想使用預設的 `~/.openclaw/` 目錄，可以自訂：

```bash
# 方法 1：使用環境變數
export OPENCLAW_HOME=/opt/openclaw
openclaw start

# 方法 2：使用命令列參數
openclaw start --home /opt/openclaw

# 方法 3：使用 --config 指定配置檔
openclaw start --config /etc/openclaw/openclaw.json5

# 方法 4：在 .bashrc 中永久設定
echo 'export OPENCLAW_HOME=/opt/openclaw' >> ~/.bashrc
source ~/.bashrc
```

### 4.4 檔案權限建議

```bash
# 設定正確的檔案權限
chmod 700 ~/.openclaw/                    # 目錄僅擁有者存取
chmod 600 ~/.openclaw/openclaw.json5      # 配置檔僅擁有者讀寫
chmod 600 ~/.openclaw/.env                # 環境變數僅擁有者讀寫
chmod 700 ~/.openclaw/data/               # 資料目錄
chmod 700 ~/.openclaw/data/db/            # 資料庫目錄
chmod 755 ~/.openclaw/logs/               # 日誌目錄可讀

# 確認權限
ls -la ~/.openclaw/
```

### 4.5 多使用者環境

如果多個使用者需要在同一台機器上運行各自的 OpenClaw 實例：

```bash
# 每個使用者的 OpenClaw 完全獨立
# 使用者 A: ~/.openclaw/
# 使用者 B: ~/.openclaw/
# 使用者 C: ~/.openclaw/

# 每個使用者都有自己的：
# - 配置檔
# - 資料庫
# - 日誌
# - 端口號（需要不同）

# 使用者 A 的配置
# ~/.openclaw/openclaw.json5 中：
# gateway: { port: 3000 }

# 使用者 B 的配置
# ~/.openclaw/openclaw.json5 中：
# gateway: { port: 3001 }

# 使用者 C 的配置
# ~/.openclaw/openclaw.json5 中：
# gateway: { port: 3002 }
```

---

## 5. 手動啟動與常駐程式

### 5.1 基本啟動

```bash
# 前景啟動（適合調試，Ctrl+C 停止）
openclaw start

# 背景啟動
openclaw start --daemon
# 或
openclaw start -d

# 指定端口啟動
openclaw start --port 3001

# 使用特定配置啟動
openclaw start --config /path/to/openclaw.json5

# 啟動時指定日誌等級
openclaw start --log-level debug

# 組合參數
openclaw start -d --port 3001 --log-level info
```

### 5.2 服務管理指令

```bash
# 查看服務狀態
openclaw status

# 輸出範例：
# 🐾 OpenClaw Status
# ─────────────────────
# Status:    Running ✅
# PID:       12345
# Uptime:    2h 15m 33s
# Port:      3000
# Version:   1.2.3
# Model:     openai/gpt-4o-mini
# Channels:  telegram (connected)
# Memory:    345 MB
# CPU:       1.2%

# 停止服務
openclaw stop

# 重啟服務
openclaw restart

# 強制停止（如果正常停止無效）
openclaw stop --force
```

### 5.3 使用 systemd 管理（Linux 推薦）

對於需要開機自啟和自動重啟的場景，建議使用 systemd：

```bash
# 建立 systemd 服務檔案
sudo tee /etc/systemd/system/openclaw.service << 'EOF'
[Unit]
Description=OpenClaw AI Assistant
Documentation=https://github.com/open-claw/open-claw
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=openclaw
Group=openclaw
WorkingDirectory=/home/openclaw
Environment=NODE_ENV=production
Environment=OPENCLAW_HOME=/home/openclaw/.openclaw
ExecStart=/usr/local/bin/openclaw start
ExecStop=/usr/local/bin/openclaw stop
ExecReload=/usr/local/bin/openclaw restart

# 自動重啟設定
Restart=on-failure
RestartSec=10
StartLimitIntervalSec=60
StartLimitBurst=3

# 資源限制
MemoryMax=2G
CPUQuota=200%

# 安全加固
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/openclaw/.openclaw
PrivateTmp=true

# 日誌設定
StandardOutput=journal
StandardError=journal
SyslogIdentifier=openclaw

[Install]
WantedBy=multi-user.target
EOF

# 重新載入 systemd 配置
sudo systemctl daemon-reload

# 啟動服務
sudo systemctl start openclaw

# 設為開機自啟
sudo systemctl enable openclaw

# 查看狀態
sudo systemctl status openclaw

# 查看日誌
sudo journalctl -u openclaw -f

# 重啟服務
sudo systemctl restart openclaw

# 停止服務
sudo systemctl stop openclaw
```

#### systemd 服務注意事項

```bash
# 如果使用 nvm 安裝的 Node.js，需要指定完整路徑
# 查找 openclaw 的完整路徑
which openclaw
# 例如: /home/user/.nvm/versions/node/v24.0.0/bin/openclaw

# 修改 ExecStart 為完整路徑
# ExecStart=/home/user/.nvm/versions/node/v24.0.0/bin/openclaw start

# 或者，在 Environment 中設定 PATH
# Environment=PATH=/home/user/.nvm/versions/node/v24.0.0/bin:/usr/bin:/bin
```

### 5.4 使用 PM2 管理（替代方案）

PM2 是 Node.js 專用的程序管理器，提供了更豐富的功能：

```bash
# 安裝 PM2
npm install -g pm2

# 建立 PM2 配置檔
cat > ~/.openclaw/ecosystem.config.js << 'EOF'
module.exports = {
  apps: [{
    name: 'openclaw',
    script: 'openclaw',
    args: 'start',
    cwd: process.env.HOME,
    env: {
      NODE_ENV: 'production',
      OPENCLAW_HOME: `${process.env.HOME}/.openclaw`,
    },

    // 自動重啟設定
    autorestart: true,
    watch: false,
    max_memory_restart: '1G',
    restart_delay: 5000,

    // 日誌設定
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
    error_file: `${process.env.HOME}/.openclaw/logs/pm2-error.log`,
    out_file: `${process.env.HOME}/.openclaw/logs/pm2-out.log`,
    merge_logs: true,
    log_type: 'json',

    // 叢集模式（如需多實例）
    instances: 1,
    exec_mode: 'fork',

    // 環境變數
    env_production: {
      NODE_ENV: 'production',
      LOG_LEVEL: 'info',
    },
    env_development: {
      NODE_ENV: 'development',
      LOG_LEVEL: 'debug',
    },
  }],
};
EOF

# 使用 PM2 啟動
pm2 start ~/.openclaw/ecosystem.config.js --env production

# PM2 常用指令
pm2 status              # 查看狀態
pm2 logs openclaw       # 查看日誌
pm2 restart openclaw    # 重啟
pm2 stop openclaw       # 停止
pm2 delete openclaw     # 刪除程序
pm2 monit               # 即時監控

# 設定開機自啟
pm2 startup
pm2 save

# 查看詳細資訊
pm2 describe openclaw
```

### 5.5 使用 screen/tmux（臨時方案）

在沒有 systemd 或 PM2 的環境中，可以使用 screen 或 tmux：

```bash
# 使用 tmux
tmux new-session -d -s openclaw 'openclaw start'

# 附加到工作階段
tmux attach-session -t openclaw

# 分離工作階段（Ctrl+B, D）

# 使用 screen
screen -dmS openclaw openclaw start

# 附加到 screen
screen -r openclaw

# 分離（Ctrl+A, D）
```

---

## 6. openclaw onboard --install-daemon 說明

### 6.1 功能概述

`openclaw onboard --install-daemon` 是一個特殊的 onboard 模式，它不僅會完成標準的初始化設定，還會自動安裝系統服務（daemon），使 OpenClaw 能夠在背景持續運行，並在系統重啟後自動啟動。

### 6.2 執行流程

```bash
$ openclaw onboard --install-daemon

  🐾 OpenClaw Daemon Installer
  ═══════════════════════════════

  Detecting system service manager...
  ✅ Found: systemd

  Step 1: Running standard onboard...
  (... 標準 onboard 流程 ...)

  Step 2: Installing daemon...
  ✅ Created service file: /etc/systemd/system/openclaw.service
  ✅ Daemon reloaded
  ✅ Service enabled for auto-start
  ✅ Service started

  Step 3: Verification...
  ✅ Service status: active (running)
  ✅ Health check: OK

  ┌──────────────────────────────────────────────┐
  │  🎉 OpenClaw daemon installed successfully!  │
  │                                              │
  │  Service commands:                           │
  │    systemctl status openclaw                 │
  │    systemctl restart openclaw                │
  │    systemctl stop openclaw                   │
  │    journalctl -u openclaw -f                 │
  │                                              │
  │  The service will auto-start on boot.        │
  └──────────────────────────────────────────────┘
```

### 6.3 支援的服務管理器

| 作業系統 | 服務管理器 | 自動偵測 |
|----------|-----------|----------|
| Ubuntu/Debian/CentOS | systemd | ✅ |
| macOS | launchd | ✅ |
| Alpine Linux | OpenRC | ✅ |
| 其他 | PM2 (fallback) | ✅ |

### 6.4 macOS launchd 配置

在 macOS 上，`--install-daemon` 會建立 launchd plist 檔案：

```xml
<!-- ~/Library/LaunchAgents/com.openclaw.agent.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openclaw.agent</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/openclaw</string>
        <string>start</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/Users/username/.openclaw/logs/launchd-out.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/username/.openclaw/logs/launchd-error.log</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>NODE_ENV</key>
        <string>production</string>
        <key>OPENCLAW_HOME</key>
        <string>/Users/username/.openclaw</string>
    </dict>

    <key>ThrottleInterval</key>
    <integer>10</integer>

    <key>ProcessType</key>
    <string>Background</string>
</dict>
</plist>
```

```bash
# macOS 手動管理 daemon
# 載入服務
launchctl load ~/Library/LaunchAgents/com.openclaw.agent.plist

# 卸載服務
launchctl unload ~/Library/LaunchAgents/com.openclaw.agent.plist

# 查看狀態
launchctl list | grep openclaw

# 啟動
launchctl start com.openclaw.agent

# 停止
launchctl stop com.openclaw.agent
```

### 6.5 解除安裝 Daemon

```bash
# 使用 OpenClaw 指令解除安裝
openclaw daemon uninstall

# 或手動解除安裝

# Linux (systemd)
sudo systemctl stop openclaw
sudo systemctl disable openclaw
sudo rm /etc/systemd/system/openclaw.service
sudo systemctl daemon-reload

# macOS (launchd)
launchctl unload ~/Library/LaunchAgents/com.openclaw.agent.plist
rm ~/Library/LaunchAgents/com.openclaw.agent.plist
```

### 6.6 Daemon 狀態監控

```bash
# 查看 daemon 狀態
openclaw daemon status

# 輸出：
# 🐾 OpenClaw Daemon Status
# ──────────────────────────
# Service Manager: systemd
# Service Name:    openclaw
# Status:          active (running)
# PID:             12345
# Auto-start:      enabled
# Uptime:          3 days, 14 hours
# Restarts:        0
# Memory:          345 MB
# CPU:             0.8%

# 查看 daemon 日誌
openclaw daemon logs
openclaw daemon logs --tail 100
openclaw daemon logs --follow
```

---

## 7. 開發模式運行

### 7.1 啟動開發模式

開發模式提供了更詳細的日誌輸出、熱重載和調試功能。

```bash
# 方法 1：使用 --dev 旗標
openclaw start --dev

# 方法 2：設定環境變數
NODE_ENV=development openclaw start

# 方法 3：使用 openclaw dev 指令（如果可用）
openclaw dev
```

### 7.2 開發模式特性

| 特性 | 生產模式 | 開發模式 |
|------|----------|----------|
| 日誌等級 | info | debug |
| 日誌格式 | JSON | 彩色文字 |
| 熱重載 | ❌ | ✅ |
| 原始碼映射 | ❌ | ✅ |
| 錯誤堆疊 | 精簡 | 完整 |
| 效能分析 | ❌ | 可選 |
| API 速率限制 | ✅ | 寬鬆 |
| 控制台 UI | 基本 | 完整（含調試面板） |

### 7.3 開發模式配置

```json5
// ~/.openclaw/openclaw.json5 — 開發模式配置
{
  // 在 gateway 區塊啟用開發功能
  gateway: {
    port: 3000,
    controlUi: {
      enabled: true,
      // 開發模式下啟用調試面板
      debug: true,
    },
  },

  // 啟用詳細日誌
  logging: {
    level: "debug",
    format: "text",  // 更適合終端閱讀的格式
    // 記錄所有 API 請求和回應
    includeApiPayloads: true,
  },

  // 啟用 mock 模式（不實際呼叫 API，節省費用）
  agents: {
    // 使用便宜的模型進行開發
    model: "openai/gpt-4o-mini",
  },
}
```

### 7.4 從原始碼運行（貢獻者）

如果您想修改 OpenClaw 的核心程式碼：

```bash
# 步驟 1：克隆原始碼
git clone https://github.com/open-claw/open-claw.git
cd open-claw

# 步驟 2：安裝依賴
npm install

# 步驟 3：建立開發環境配置
cp .env.example .env
cp openclaw.json5.example openclaw.json5
# 編輯 .env 和 openclaw.json5

# 步驟 4：以開發模式啟動
npm run dev

# 步驟 5：開發模式會啟用檔案監控
# 修改 src/ 下的檔案會自動重啟

# 其他開發指令
npm run build         # 構建
npm run test          # 測試
npm run lint          # 程式碼檢查
npm run type-check    # TypeScript 型別檢查
```

### 7.5 調試

```bash
# 使用 Node.js 內建調試器
openclaw start --inspect

# 使用特定調試端口
openclaw start --inspect=9229

# 啟動時暫停（等待調試器連接）
openclaw start --inspect-brk

# 在 VS Code 中調試
# .vscode/launch.json
```

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to OpenClaw",
      "port": 9229,
      "restart": true,
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Launch OpenClaw (Dev)",
      "runtimeExecutable": "openclaw",
      "runtimeArgs": ["start", "--dev"],
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}
```

### 7.6 開發模式實用技巧

```bash
# 技巧 1：使用 mock 模型避免 API 費用
openclaw start --dev --mock-model

# 技巧 2：僅啟動特定通道
openclaw start --dev --channel telegram

# 技巧 3：使用本地 Ollama 避免 API 費用
# 在 openclaw.json5 中設定模型為 ollama/llama3.1:8b

# 技巧 4：監控 API 使用量
openclaw usage
# 輸出：
# Model              Requests   Input Tokens   Output Tokens   Cost
# gpt-4o-mini       45         12,340         5,678          $0.05
# claude-sonnet-4   12         8,901          3,456          $0.08
# Total:            57         21,241         9,134          $0.13

# 技巧 5：重置對話（清除記憶）
openclaw memory clear
openclaw memory clear --channel telegram --user 123456

# 技巧 6：匯出對話記錄
openclaw export conversations --format json --output ./conversations.json

# 技巧 7：測試特定技能
openclaw skill test my-skill --input "測試輸入"
```

---

## 8. 從 npm 切換到 Docker

當您的 OpenClaw 從開發階段進入生產階段，建議從 npm 安裝切換到 Docker 部署。以下是完整的遷移流程。

### 8.1 遷移前準備

```bash
# 步驟 1：確認當前 npm 安裝的版本
openclaw --version

# 步驟 2：建立完整備份
openclaw backup create --name "pre-docker-migration"

# 步驟 3：匯出配置
cp ~/.openclaw/openclaw.json5 ./migration/
cp ~/.openclaw/.env ./migration/

# 步驟 4：匯出資料庫
cp -r ~/.openclaw/data/db/ ./migration/db/

# 步驟 5：匯出自訂技能
cp -r ~/.openclaw/data/skills/ ./migration/skills/ 2>/dev/null || true
```

### 8.2 遷移步驟

```bash
# 步驟 1：停止 npm 版本的 OpenClaw
openclaw stop
# 如果使用 systemd
sudo systemctl stop openclaw
sudo systemctl disable openclaw

# 步驟 2：準備 Docker 環境
mkdir -p /opt/openclaw
cd /opt/openclaw

# 步驟 3：克隆倉庫
git clone https://github.com/open-claw/open-claw.git .
git checkout $(git tag --sort=-version:refname | grep -v "alpha\|beta\|rc" | head -1)

# 步驟 4：遷移配置檔
# 將 npm 版本的 .env 轉換為 Docker 版本
cp ~/migration/.env .env
# 檢查並調整 Docker 特定的環境變數
# 例如：OLLAMA_BASE_URL 需要改為 http://host.docker.internal:11434

# 步驟 5：遷移 openclaw.json5
cp ~/migration/openclaw.json5 ./openclaw.json5

# 步驟 6：遷移資料庫
mkdir -p data/db
cp ~/migration/db/* data/db/

# 步驟 7：遷移自訂技能
mkdir -p skills
cp -r ~/migration/skills/* skills/ 2>/dev/null || true

# 步驟 8：建立其他必要目錄
mkdir -p data/{logs,media,backups}

# 步驟 9：啟動 Docker 版本
docker compose up -d

# 步驟 10：驗證
docker compose ps
curl -s http://localhost:3000/health | jq .
docker compose exec openclaw openclaw doctor
```

### 8.3 遷移後驗證清單

```bash
# 驗證清單
echo "=== 遷移驗證 ==="

# 1. 服務狀態
echo "1. 服務狀態:"
docker compose ps

# 2. 健康檢查
echo "2. 健康檢查:"
curl -s http://localhost:3000/health | jq .

# 3. 通道連線
echo "3. 通道連線:"
docker compose logs --tail 10 openclaw | grep -i "channel\|connect"

# 4. 資料庫完整性
echo "4. 資料庫:"
docker compose exec openclaw openclaw db check

# 5. 對話記錄
echo "5. 對話記錄:"
docker compose exec openclaw openclaw conversations count

# 6. 測試對話
echo "6. 請在通訊平台上發送測試訊息"
```

### 8.4 清理 npm 安裝

```bash
# 確認 Docker 版本正常運行後，可以清理 npm 安裝

# 解除安裝全域套件
npm uninstall -g openclaw

# 移除 systemd 服務（如果有）
sudo rm -f /etc/systemd/system/openclaw.service
sudo systemctl daemon-reload

# 備份後刪除 npm 版本的資料
# ⚠️ 請確認 Docker 版本已成功遷移所有資料後再執行！
# tar -czf ~/openclaw-npm-backup.tar.gz ~/.openclaw/
# rm -rf ~/.openclaw/
```

### 8.5 遷移常見問題

| 問題 | 原因 | 解決方案 |
|------|------|----------|
| Ollama 無法連線 | Docker 容器無法存取 localhost | 使用 `host.docker.internal` 或 Docker 網路 |
| 資料庫損壞 | 遷移過程中檔案不完整 | 使用 `sqlite3` 進行完整性檢查和修復 |
| Webhook 失效 | URL 或端口變更 | 重新設定 Webhook URL |
| 權限錯誤 | Docker 容器使用者不同 | 調整 data/ 目錄的擁有者 |
| 自訂技能失效 | 路徑不同 | 更新 openclaw.json5 中的技能路徑 |

```bash
# 修復 Ollama 連線（Docker 環境）
# 在 .env 中修改：
# OLLAMA_BASE_URL=http://host.docker.internal:11434

# 或如果 Ollama 也在 Docker 中：
# OLLAMA_BASE_URL=http://ollama:11434

# 修復資料庫
docker compose exec openclaw sqlite3 /app/data/db/openclaw.db "PRAGMA integrity_check;"

# 修復權限
sudo chown -R 1000:1000 data/
```

---

## 9. 常見問題排解

### 9.1 安裝問題

#### 問題：npm install -g 失敗

```bash
# 症狀：EACCES 權限錯誤
# npm ERR! Error: EACCES: permission denied

# 解決方案 1（推薦）：使用 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 24
npm install -g openclaw

# 解決方案 2：設定 npm 全域目錄
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
npm install -g openclaw
```

#### 問題：安裝後 openclaw 指令找不到

```bash
# 症狀：command not found: openclaw

# 確認 npm 全域 bin 目錄
npm bin -g
# 例如: /home/user/.npm-global/bin

# 確認 PATH 包含此目錄
echo $PATH | tr ':' '\n' | grep npm

# 如果不包含，添加到 PATH
echo 'export PATH="$(npm bin -g):$PATH"' >> ~/.bashrc
source ~/.bashrc

# 確認
which openclaw
```

#### 問題：Node.js 版本不滿足

```bash
# 症狀：Error: OpenClaw requires Node.js 22+. Current: v18.x.x

# 使用 nvm 升級
nvm install 24
nvm alias default 24

# 確認
node --version
# 重新安裝
npm install -g openclaw
```

#### 問題：原生依賴編譯失敗

```bash
# 症狀：gyp ERR! build error / node-pre-gyp ERR!

# 確認已安裝編譯工具
# Ubuntu/Debian
sudo apt install -y build-essential python3

# CentOS/Rocky Linux
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y python3

# macOS
xcode-select --install

# 清理快取後重試
npm cache clean --force
npm install -g openclaw
```

### 9.2 啟動問題

#### 問題：端口被佔用

```bash
# 症狀：Error: listen EADDRINUSE: address already in use :::3000

# 查找佔用端口的程序
sudo ss -tlnp | grep :3000
# 或
sudo lsof -i :3000

# 解決方案 1：停止佔用端口的程序
# kill <PID>

# 解決方案 2：使用不同的端口
openclaw start --port 3001

# 解決方案 3：在配置中變更端口
# openclaw.json5:
# gateway: { port: 3001 }
```

#### 問題：API Key 無效

```bash
# 症狀：Error: 401 Unauthorized — Invalid API key

# 檢查 API Key
openclaw config validate

# 測試 API Key
# OpenAI
curl -s https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY" | jq '.error // "OK"'

# Anthropic
curl -s https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-sonnet-4-20250514","max_tokens":1,"messages":[{"role":"user","content":"hi"}]}' | jq '.error // "OK"'

# 更新 API Key
openclaw config set models.providers.openai.apiKey "sk-proj-new-key"
openclaw restart
```

#### 問題：配置檔語法錯誤

```bash
# 症狀：Error: Failed to parse openclaw.json5

# 驗證配置檔語法
openclaw config validate

# 常見錯誤：
# 1. 缺少逗號
# 2. 多餘的逗號（JSON5 允許尾隨逗號，通常不是問題）
# 3. 未閉合的括號
# 4. 不正確的引號

# 使用 jq 或 json5 工具檢查
npx json5 ~/.openclaw/openclaw.json5

# 恢復預設配置
openclaw config reset
```

### 9.3 運行時問題

#### 問題：記憶體持續增長

```bash
# 症狀：記憶體使用量不斷增加，最終 OOM

# 查看記憶體使用
openclaw status --detail

# 解決方案 1：設定記憶體限制
# 在 openclaw.json5 中：
# memory: {
#   tokenLimit: 64000,  # 減少 token 限制
#   contextRetention: 20,  # 減少保留的對話輪數
# }

# 解決方案 2：定期重啟
# 使用 cron 定期重啟
echo "0 4 * * * /usr/local/bin/openclaw restart" | crontab -

# 解決方案 3：使用 PM2 的記憶體限制
# pm2 start openclaw --max-memory-restart 1G

# 解決方案 4：分析記憶體洩漏
openclaw start --dev --heapsnapshot
# 使用 Chrome DevTools 分析 heap snapshot
```

#### 問題：回應速度慢

```bash
# 症狀：Bot 回應延遲很高

# 步驟 1：檢查 API 延遲
openclaw diagnostics --network

# 步驟 2：檢查模型選擇
# 較大的模型（如 gpt-4o）比較慢
# 考慮使用 gpt-4o-mini 或 claude-haiku

# 步驟 3：檢查 token 使用量
openclaw usage --last 24h

# 步驟 4：減少上下文長度
# memory: { contextRetention: 10 }

# 步驟 5：啟用串流回應
# agents: { streaming: true }
```

#### 問題：Webhook 接收不到訊息

```bash
# 步驟 1：確認外部 URL 可存取
curl -I https://openclaw.yourdomain.com/health

# 步驟 2：確認 SSL 憑證有效
openssl s_client -connect openclaw.yourdomain.com:443 </dev/null 2>/dev/null | \
  grep "Verify return code"

# 步驟 3：查看 webhook 日誌
openclaw logs --filter webhook

# 步驟 4：測試本地接收
curl -X POST http://localhost:3000/webhook/telegram \
  -H "Content-Type: application/json" \
  -d '{"update_id": 1, "message": {"text": "test"}}'

# 步驟 5：重新設定 webhook
openclaw channel telegram setup-webhook
```

### 9.4 升級問題

#### 問題：升級後功能異常

```bash
# 步驟 1：查看版本變更日誌
npm info openclaw changelog

# 步驟 2：清除快取
openclaw cache clear

# 步驟 3：重新初始化（保留資料）
openclaw config migrate

# 步驟 4：回滾到上一個版本
npm install -g openclaw@1.2.2  # 指定上一個版本

# 步驟 5：執行完整診斷
openclaw doctor --verbose
```

### 9.5 完整解除安裝

```bash
#!/usr/bin/env bash
# uninstall-openclaw.sh — 完整解除安裝 OpenClaw

echo "⚠️ 即將完整解除安裝 OpenClaw"
echo "這將刪除所有配置、資料和備份！"
read -p "確定要繼續嗎？(y/N) " confirm

if [[ "$confirm" != "y" && "$confirm" != "Y" ]]; then
  echo "已取消"
  exit 0
fi

# 停止服務
openclaw stop 2>/dev/null || true

# 解除安裝 daemon
openclaw daemon uninstall 2>/dev/null || true

# 移除 systemd 服務
sudo systemctl stop openclaw 2>/dev/null || true
sudo systemctl disable openclaw 2>/dev/null || true
sudo rm -f /etc/systemd/system/openclaw.service
sudo systemctl daemon-reload 2>/dev/null || true

# 移除 launchd 服務 (macOS)
launchctl unload ~/Library/LaunchAgents/com.openclaw.agent.plist 2>/dev/null || true
rm -f ~/Library/LaunchAgents/com.openclaw.agent.plist

# 移除 PM2 程序
pm2 delete openclaw 2>/dev/null || true

# 解除安裝 npm 套件
npm uninstall -g openclaw

# 建立最終備份（可選）
if [[ -d ~/.openclaw ]]; then
  echo "建立最終備份..."
  tar -czf ~/openclaw-final-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/
  echo "備份已保存: ~/openclaw-final-backup-$(date +%Y%m%d).tar.gz"

  # 刪除資料目錄
  rm -rf ~/.openclaw/
fi

echo "✅ OpenClaw 已完全解除安裝"
```

### 9.6 取得幫助

如果以上方法都無法解決您的問題：

```bash
# 收集診斷資訊
openclaw doctor --verbose > diagnostic-report.txt 2>&1

# 收集日誌
openclaw logs --tail 200 >> diagnostic-report.txt

# 收集系統資訊
echo "=== System Info ===" >> diagnostic-report.txt
uname -a >> diagnostic-report.txt
node --version >> diagnostic-report.txt
npm --version >> diagnostic-report.txt
openclaw --version >> diagnostic-report.txt

# 到 GitHub 提交 Issue
# https://github.com/open-claw/open-claw/issues/new
# 附上 diagnostic-report.txt（請先移除敏感資訊如 API Keys）
```

---

## 下一步

完成 npm 安裝後，建議：

1. 閱讀 **[配置檔完整參考](04-配置檔完整參考.md)** 深入了解所有配置選項
2. 閱讀 **記憶與人格篇** 打造獨特的 Bot 人格
3. 閱讀 **Skills 開發篇** 學習如何為 Bot 添加自訂技能
4. 如需部署到生產環境，請參考 **[Docker 安裝指南](02-Docker安裝指南.md)**

---

> **📝 附註**：npm 安裝方式最適合個人開發和學習。如果您計畫將 OpenClaw 部署到生產環境並需要長期穩定運行，強烈建議遷移到 Docker 部署方式。
