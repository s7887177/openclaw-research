# Docker 安裝指南——推薦的生產環境部署方式

> **文件版本**：v1.0
> **適用 OpenClaw 版本**：0.9.x — 1.x
> **最後更新**：2025-07
> **前置條件**：請先完成 [系統需求與前置條件](01-系統需求與前置條件.md) 中的所有檢查

Docker 是部署 OpenClaw 的**推薦方式**，特別適合生產環境。透過 Docker 容器化部署，您可以獲得一致的運行環境、簡便的升級流程、以及可靠的備份還原機制。本指南將帶您從零開始，逐步完成 OpenClaw 的 Docker 部署。

---

## 目錄

1. [為什麼推薦 Docker](#1-為什麼推薦-docker)
2. [Docker 安裝（如果尚未安裝）](#2-docker-安裝如果尚未安裝)
3. [克隆 OpenClaw 倉庫](#3-克隆-openclaw-倉庫)
4. [切換到最新穩定版本](#4-切換到最新穩定版本)
5. [環境變數配置（.env 文件完整說明）](#5-環境變數配置env-文件完整說明)
6. [docker-compose.yml 詳解](#6-docker-composeyml-詳解)
7. [使用官方 setup.sh 腳本](#7-使用官方-setupsh-腳本)
8. [手動 Docker Compose 配置](#8-手動-docker-compose-配置)
9. [首次啟動與驗證](#9-首次啟動與驗證)
10. [日誌查看與監控](#10-日誌查看與監控)
11. [升級流程](#11-升級流程)
12. [備份與還原](#12-備份與還原)
13. [常見問題排解](#13-常見問題排解)
14. [進階：Nginx SSL 反向代理配置](#14-進階nginx-ssl-反向代理配置)

---

## 1. 為什麼推薦 Docker

### 1.1 Docker 部署的優勢

在所有支援的安裝方式中，Docker 部署提供了以下無可比擬的優勢：

| 優勢 | 說明 |
|------|------|
| **環境一致性** | 開發、測試、生產環境完全相同，消除「在我機器上能跑」的問題 |
| **依賴隔離** | 所有依賴都封裝在容器內，不會與宿主機系統衝突 |
| **簡便升級** | 只需拉取新映像、重啟容器即可完成升級 |
| **輕鬆回滾** | 保留舊映像，升級失敗時可立即回滾 |
| **資料持久化** | 使用 Docker Volume 保障資料安全 |
| **資源管理** | 精確控制 CPU、記憶體使用量 |
| **多實例部署** | 輕鬆在同一台機器上運行多個 OpenClaw 實例 |
| **安全隔離** | 容器沙箱提供額外的安全層級 |
| **日誌管理** | 統一的日誌收集和輪替機制 |
| **健康檢查** | 內建健康檢查，自動重啟異常容器 |

### 1.2 Docker vs npm 直接安裝

| 面向 | Docker 部署 | npm 直接安裝 |
|------|-------------|-------------|
| 安裝難度 | ⭐⭐ 中等 | ⭐ 簡單 |
| 生產環境適用 | ✅ 強烈推薦 | ⚠️ 可行但不推薦 |
| 環境隔離 | ✅ 完全隔離 | ❌ 與系統共享 |
| 升級便利性 | ✅ 映像更換 | ⚠️ 需手動處理依賴 |
| 資源控制 | ✅ 精確控制 | ❌ 需額外工具 |
| 多實例 | ✅ 原生支援 | ⚠️ 需手動管理 |
| 調試便利性 | ⚠️ 需額外步驟 | ✅ 直接調試 |
| 啟動速度 | ⚠️ 稍慢（容器啟動） | ✅ 直接啟動 |

### 1.3 適用場景

**Docker 部署適合以下場景：**

- 生產環境部署
- 需要長期穩定運行的服務
- 團隊協作（統一開發環境）
- CI/CD 整合
- 需要多個通道或多實例的部署
- 對安全性有較高要求的場景

**不適合 Docker 的場景（請參考 npm 安裝指南）：**

- 快速原型驗證
- 本地開發調試（頻繁修改程式碼）
- 資源極度受限的環境（如 512 MB RAM）

---

## 2. Docker 安裝（如果尚未安裝）

如果您已經安裝了 Docker 和 Docker Compose v2，請跳到第 3 節。

### 2.1 Linux 安裝

#### Ubuntu/Debian

```bash
# 步驟 1：移除舊版本 Docker
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null || true

# 步驟 2：安裝必要的前置套件
sudo apt update
sudo apt install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release

# 步驟 3：添加 Docker 官方 GPG 金鑰
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 步驟 4：添加 Docker 的 APT 倉庫
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 步驟 5：安裝 Docker Engine
sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

# 步驟 6：啟動 Docker 並設為開機自啟
sudo systemctl start docker
sudo systemctl enable docker

# 步驟 7：將當前使用者加入 docker 群組
sudo usermod -aG docker $USER

# ⚠️ 重要：需要重新登入或執行以下命令使群組生效
newgrp docker

# 步驟 8：驗證安裝
echo "=== Docker 版本 ==="
docker --version

echo "=== Docker Compose 版本 ==="
docker compose version

echo "=== 測試 Docker ==="
docker run --rm hello-world
```

#### CentOS/Rocky Linux/AlmaLinux

```bash
# 步驟 1：移除舊版本
sudo dnf remove -y docker docker-client docker-client-latest \
  docker-common docker-latest docker-latest-logrotate \
  docker-logrotate docker-engine 2>/dev/null || true

# 步驟 2：安裝 dnf 工具和前置套件
sudo dnf install -y dnf-plugins-core

# 步驟 3：添加 Docker 倉庫
sudo dnf config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

# 步驟 4：安裝 Docker
sudo dnf install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

# 步驟 5：啟動並設為自啟
sudo systemctl start docker
sudo systemctl enable docker

# 步驟 6：設定使用者群組
sudo usermod -aG docker $USER
newgrp docker

# 步驟 7：驗證
docker --version
docker compose version
docker run --rm hello-world
```

### 2.2 macOS 安裝

```bash
# 方法 1：使用 Homebrew 安裝 Docker Desktop（推薦）
brew install --cask docker

# 安裝完成後，從「應用程式」中啟動 Docker Desktop
# 首次啟動需要授予系統權限

# 等待 Docker Desktop 完全啟動後驗證
docker --version
docker compose version
```

```
macOS Docker Desktop 建議設定：
┌─────────────────────────────────────┐
│ Docker Desktop > Settings           │
│                                     │
│ General:                            │
│   ☑ Start Docker Desktop on login   │
│   ☑ Use the WSL 2 based engine     │
│                                     │
│ Resources:                          │
│   CPUs: 4                          │
│   Memory: 4 GB                      │
│   Swap: 1 GB                        │
│   Disk image size: 40 GB            │
│                                     │
│ Docker Engine:                      │
│   (使用預設設定)                     │
└─────────────────────────────────────┘
```

### 2.3 Windows (WSL2) 安裝

```powershell
# 在 PowerShell 中（以系統管理員身分）
# 步驟 1：確認 WSL2 已安裝
wsl --list --verbose

# 步驟 2：安裝 Docker Desktop for Windows
# 從 https://www.docker.com/products/docker-desktop/ 下載
# 或使用 winget
winget install Docker.DockerDesktop

# 步驟 3：在 Docker Desktop 設定中啟用 WSL2 整合
# Settings > General > ☑ Use the WSL 2 based engine
# Settings > Resources > WSL Integration > 啟用您的 Linux 發行版

# 步驟 4：在 WSL2 中驗證
wsl
docker --version
docker compose version
```

### 2.4 Docker 安裝後配置

無論在哪個平台，安裝完 Docker 後建議進行以下配置：

```bash
# 配置 Docker daemon（Linux）
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  },
  "storage-driver": "overlay2",
  "live-restore": true,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  }
}
EOF

# 重啟 Docker
sudo systemctl daemon-reload
sudo systemctl restart docker

# 驗證 daemon 配置
docker info | grep -E "(Storage Driver|Logging Driver|Live Restore)"
```

---

## 3. 克隆 OpenClaw 倉庫

### 3.1 克隆倉庫

```bash
# 選擇安裝目錄
# 生產環境建議使用 /opt 或 /srv 目錄
sudo mkdir -p /opt/openclaw
sudo chown $USER:$USER /opt/openclaw

# 克隆倉庫
cd /opt/openclaw
git clone https://github.com/open-claw/open-claw.git .

# 或者，如果您想將安裝在使用者目錄
cd ~
git clone https://github.com/open-claw/open-claw.git openclaw
cd openclaw
```

### 3.2 倉庫結構概覽

```
openclaw/
├── docker-compose.yml      # Docker Compose 主配置
├── docker-compose.dev.yml  # 開發環境覆蓋配置
├── Dockerfile              # 主 Dockerfile
├── Dockerfile.sandbox      # 沙箱 Dockerfile
├── setup.sh                # 官方安裝腳本
├── .env.example            # 環境變數範本
├── openclaw.json5.example  # OpenClaw 配置範本
├── package.json            # Node.js 專案配置
├── src/                    # 原始碼
├── skills/                 # 內建技能
├── docs/                   # 文件
├── scripts/                # 輔助腳本
│   ├── backup.sh           # 備份腳本
│   ├── restore.sh          # 還原腳本
│   └── health-check.sh     # 健康檢查腳本
└── data/                   # 資料目錄（執行後產生）
    ├── db/                 # 資料庫檔案
    ├── logs/               # 日誌檔案
    ├── media/              # 媒體檔案
    └── skills/             # 使用者技能
```

---

## 4. 切換到最新穩定版本

### 4.1 查看可用版本

```bash
# 列出所有版本標籤（按版本號排序）
git tag --sort=-version:refname | head -20

# 輸出範例：
# v1.2.3
# v1.2.2
# v1.2.1
# v1.1.0
# v1.0.0
# v0.9.5
# ...

# 查看最新穩定版本
git tag --sort=-version:refname | grep -v "alpha\|beta\|rc" | head -1
```

### 4.2 切換版本

```bash
# 取得最新標籤
LATEST_TAG=$(git tag --sort=-version:refname | grep -v "alpha\|beta\|rc" | head -1)
echo "最新穩定版本: $LATEST_TAG"

# 切換到最新穩定版本
git checkout "$LATEST_TAG"

# 確認目前版本
git describe --tags
```

### 4.3 版本選擇策略

| 版本類型 | 格式 | 適用場景 | 穩定性 |
|----------|------|----------|--------|
| **穩定版（Release）** | `v1.2.3` | 生產環境 | ⭐⭐⭐⭐⭐ |
| **候選版（RC）** | `v1.3.0-rc.1` | 預發布測試 | ⭐⭐⭐⭐ |
| **Beta 版** | `v1.3.0-beta.1` | 功能測試 | ⭐⭐⭐ |
| **Alpha 版** | `v1.3.0-alpha.1` | 早期測試 | ⭐⭐ |
| **main 分支** | `main` | 開發環境 | ⭐ |

```bash
# 如果您想使用 main 分支（最新開發版）
git checkout main
git pull origin main

# 如果您想使用特定版本
git checkout v1.2.3
```

---

## 5. 環境變數配置（.env 文件完整說明）

`.env` 文件是 Docker 部署的核心配置。它包含了 OpenClaw 運行所需的所有環境變數。

### 5.1 建立 .env 文件

```bash
# 從範本複製
cp .env.example .env

# 設定安全權限（僅擁有者可讀寫）
chmod 600 .env

# 使用您偏好的編輯器編輯
nano .env
# 或
vim .env
# 或
code .env  # 使用 VS Code
```

### 5.2 .env 文件完整參數說明

```bash
# ============================================================
# OpenClaw Docker 環境變數配置
# ============================================================
# 本文件包含 OpenClaw Docker 部署所需的所有環境變數。
# 請根據您的需求修改以下配置。
# ⚠️ 請勿將此文件提交到版本控制系統中！
# ============================================================

# -----------------------------------------------------------
# 1. 基本設定
# -----------------------------------------------------------

# OpenClaw 實例名稱（僅用於日誌和識別）
OPENCLAW_NAME=my-openclaw-bot

# 運行模式: production | development | test
NODE_ENV=production

# 時區設定（影響排程任務和日誌時間戳）
TZ=Asia/Taipei

# 語言設定
LANG=zh-TW

# -----------------------------------------------------------
# 2. AI 模型 API Keys
# -----------------------------------------------------------

# OpenAI API Key（GPT-4o、GPT-4o-mini 等）
# 取得方式: https://platform.openai.com/api-keys
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Anthropic API Key（Claude 系列模型）
# 取得方式: https://console.anthropic.com/settings/keys
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Google AI API Key（Gemini 系列模型）
# 取得方式: https://aistudio.google.com/
GOOGLE_AI_API_KEY=AIzaSyxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Groq API Key（高速推理）
# 取得方式: https://console.groq.com/keys
GROQ_API_KEY=gsk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Together AI API Key
# 取得方式: https://api.together.ai/settings/api-keys
TOGETHER_API_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# OpenRouter API Key（統一多模型存取）
# 取得方式: https://openrouter.ai/keys
OPENROUTER_API_KEY=sk-or-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 本地模型 Ollama 端點（如果使用本地模型）
OLLAMA_BASE_URL=http://host.docker.internal:11434

# -----------------------------------------------------------
# 3. 通訊通道配置
# -----------------------------------------------------------

# -- WhatsApp --
# WhatsApp Business API 配置
# 取得方式: https://developers.facebook.com/docs/whatsapp
WHATSAPP_ACCESS_TOKEN=EAAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
WHATSAPP_PHONE_NUMBER_ID=123456789012345
WHATSAPP_BUSINESS_ACCOUNT_ID=123456789012345
WHATSAPP_VERIFY_TOKEN=your-custom-verify-token
WHATSAPP_APP_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# -- Telegram --
# Telegram Bot 配置
# 取得方式: 與 @BotFather 對話
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGhIJKlmNoPQRsTUVwxYZ

# -- Discord --
# Discord Bot 配置
# 取得方式: https://discord.com/developers/applications
DISCORD_BOT_TOKEN=MTIzNDU2Nzg5MDEyMzQ1Njc4OQ.xxxxxx.xxxxxxxxxxxxxxxxxxxxxxxxxxxx
DISCORD_APPLICATION_ID=123456789012345678

# -- Slack --
# Slack Bot 配置
# 取得方式: https://api.slack.com/apps
SLACK_BOT_TOKEN=xoxb-xxxxxxxxxxxx-xxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxx
SLACK_APP_TOKEN=xapp-1-xxxxxxxxxxxx-xxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxx
SLACK_SIGNING_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# -----------------------------------------------------------
# 4. Gateway 配置
# -----------------------------------------------------------

# Gateway HTTP 端口（容器內部）
GATEWAY_PORT=3000

# 外部存取基礎 URL（用於 Webhook 回呼）
# ⚠️ 必須是可從外部存取的 HTTPS URL
GATEWAY_BASE_URL=https://openclaw.yourdomain.com

# Gateway 管理介面密碼
GATEWAY_AUTH_TOKEN=your-secure-admin-password-here

# 控制台 UI
CONTROL_UI_ENABLED=true

# 更新通道: stable | beta | nightly
UPDATE_CHANNEL=stable

# -----------------------------------------------------------
# 5. 沙箱配置
# -----------------------------------------------------------

# 啟用沙箱（允許 AI 執行程式碼）
SANDBOX_ENABLED=true

# 沙箱類型: docker | local
SANDBOX_TYPE=docker

# 沙箱逾時（秒）
SANDBOX_TIMEOUT=60

# 沙箱記憶體限制（MB）
SANDBOX_MEMORY_LIMIT=512

# 沙箱 CPU 限制（核心數）
SANDBOX_CPU_LIMIT=1

# -----------------------------------------------------------
# 6. 資料庫與儲存
# -----------------------------------------------------------

# 資料儲存目錄（Docker volume 映射的宿主機路徑）
DATA_DIR=./data

# 日誌目錄
LOG_DIR=./data/logs

# 資料庫類型: sqlite | postgres
DB_TYPE=sqlite

# PostgreSQL 配置（僅在 DB_TYPE=postgres 時使用）
# DB_HOST=localhost
# DB_PORT=5432
# DB_NAME=openclaw
# DB_USER=openclaw
# DB_PASSWORD=your-secure-db-password

# -----------------------------------------------------------
# 7. 安全設定
# -----------------------------------------------------------

# 加密金鑰（用於加密敏感資料）
# ⚠️ 首次設定後請勿變更，否則已加密資料將無法解密
# 產生方式: openssl rand -hex 32
ENCRYPTION_KEY=your-64-character-hex-encryption-key-here

# JWT 秘鑰（用於 API 認證）
# 產生方式: openssl rand -hex 32
JWT_SECRET=your-64-character-hex-jwt-secret-here

# CORS 允許的來源（逗號分隔）
CORS_ORIGINS=https://openclaw.yourdomain.com

# -----------------------------------------------------------
# 8. 日誌配置
# -----------------------------------------------------------

# 日誌等級: debug | info | warn | error
LOG_LEVEL=info

# 日誌格式: json | text
LOG_FORMAT=json

# 日誌檔案大小上限（MB）
LOG_MAX_SIZE=10

# 日誌檔案保留數量
LOG_MAX_FILES=5

# -----------------------------------------------------------
# 9. 進階設定
# -----------------------------------------------------------

# Web 搜尋 API（選配）
# Tavily API Key（推薦）
TAVILY_API_KEY=tvly-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 或 SerpAPI Key
SERPAPI_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 記憶體配置
MEMORY_TOKEN_LIMIT=128000
CONTEXT_RETENTION_MESSAGES=50

# Heartbeat 間隔（秒）
HEARTBEAT_INTERVAL=30

# 圖片最大尺寸（像素）
IMAGE_MAX_DIMENSION_PX=1024
```

### 5.3 快速生成安全金鑰

```bash
# 生成 ENCRYPTION_KEY
echo "ENCRYPTION_KEY=$(openssl rand -hex 32)"

# 生成 JWT_SECRET
echo "JWT_SECRET=$(openssl rand -hex 32)"

# 生成 GATEWAY_AUTH_TOKEN
echo "GATEWAY_AUTH_TOKEN=$(openssl rand -base64 24)"

# 生成 WHATSAPP_VERIFY_TOKEN
echo "WHATSAPP_VERIFY_TOKEN=$(openssl rand -hex 16)"

# 一次生成所有金鑰並寫入 .env（追加模式）
cat >> .env << EOF

# === 自動生成的安全金鑰 ===
ENCRYPTION_KEY=$(openssl rand -hex 32)
JWT_SECRET=$(openssl rand -hex 32)
GATEWAY_AUTH_TOKEN=$(openssl rand -base64 24)
EOF
```

### 5.4 環境變數驗證

```bash
# 檢查 .env 文件是否包含所有必要的變數
required_vars=(
  "OPENAI_API_KEY"
  "GATEWAY_PORT"
  "GATEWAY_BASE_URL"
  "ENCRYPTION_KEY"
  "JWT_SECRET"
)

echo "=== 環境變數檢查 ==="
for var in "${required_vars[@]}"; do
  if grep -q "^${var}=" .env && ! grep -q "^${var}=$" .env; then
    echo "✅ $var 已設定"
  else
    echo "❌ $var 未設定或為空"
  fi
done

# 檢查至少有一個 AI API Key
ai_keys=("OPENAI_API_KEY" "ANTHROPIC_API_KEY" "GOOGLE_AI_API_KEY" "OLLAMA_BASE_URL")
has_ai_key=false
for key in "${ai_keys[@]}"; do
  if grep -q "^${key}=.\+" .env; then
    has_ai_key=true
    echo "✅ AI Key 已設定: $key"
  fi
done
if ! $has_ai_key; then
  echo "❌ 未設定任何 AI API Key"
fi
```

---

## 6. docker-compose.yml 詳解

### 6.1 官方 docker-compose.yml 結構

以下是 OpenClaw 官方 `docker-compose.yml` 的完整內容和逐行註解：

```yaml
# OpenClaw Docker Compose 配置
# 版本: 適用於 Docker Compose v2
# 文件: docker-compose.yml

services:
  # ============================================================
  # OpenClaw 核心服務
  # ============================================================
  openclaw:
    # 使用官方映像（推薦）
    image: ghcr.io/open-claw/open-claw:latest
    # 或使用本地構建
    # build:
    #   context: .
    #   dockerfile: Dockerfile

    # 容器名稱
    container_name: openclaw

    # 重啟策略: 除非手動停止，否則自動重啟
    restart: unless-stopped

    # 環境變數文件
    env_file:
      - .env

    # 端口映射: 宿主機端口:容器端口
    ports:
      - "${GATEWAY_PORT:-3000}:3000"

    # 資料卷映射
    volumes:
      # 持久化資料目錄
      - openclaw-data:/app/data
      # 配置文件（可選，如果使用 openclaw.json5）
      - ./openclaw.json5:/app/openclaw.json5:ro
      # 自訂技能目錄
      - ./skills:/app/custom-skills:ro
      # 日誌目錄
      - openclaw-logs:/app/data/logs

    # 健康檢查
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

    # 資源限制
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2G
        reservations:
          cpus: "0.5"
          memory: 512M

    # 網路
    networks:
      - openclaw-network

    # 依賴服務
    depends_on:
      sandbox:
        condition: service_healthy
        required: false

    # 日誌配置
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

    # 安全選項
    security_opt:
      - no-new-privileges:true

  # ============================================================
  # 沙箱服務（可選）
  # ============================================================
  sandbox:
    image: ghcr.io/open-claw/open-claw-sandbox:latest
    # build:
    #   context: .
    #   dockerfile: Dockerfile.sandbox

    container_name: openclaw-sandbox

    restart: unless-stopped

    # 沙箱不需要對外開放端口
    # 透過 Docker 內部網路與核心服務通信
    expose:
      - "8080"

    environment:
      - SANDBOX_TIMEOUT=${SANDBOX_TIMEOUT:-60}
      - SANDBOX_MEMORY_LIMIT=${SANDBOX_MEMORY_LIMIT:-512}

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s

    deploy:
      resources:
        limits:
          cpus: "${SANDBOX_CPU_LIMIT:-1}"
          memory: ${SANDBOX_MEMORY_LIMIT:-512}M
        reservations:
          cpus: "0.25"
          memory: 256M

    networks:
      - openclaw-network

    # 沙箱安全加固
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    read_only: true
    tmpfs:
      - /tmp:size=100M
      - /app/workspace:size=200M

    logging:
      driver: json-file
      options:
        max-size: "5m"
        max-file: "3"

# ============================================================
# 資料卷定義
# ============================================================
volumes:
  openclaw-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ${DATA_DIR:-./data}/db

  openclaw-logs:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ${DATA_DIR:-./data}/logs

# ============================================================
# 網路定義
# ============================================================
networks:
  openclaw-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

### 6.2 各區塊詳解

#### services 區塊

每個 `service` 代表一個 Docker 容器。OpenClaw 標準部署包含兩個服務：

| 服務名稱 | 用途 | 必要性 |
|----------|------|--------|
| `openclaw` | 核心服務，處理所有 AI 對話和通道訊息 | 必要 |
| `sandbox` | 程式碼沙箱，用於安全執行 AI 生成的程式碼 | 選用 |

#### volumes 區塊

Docker Volume 用於持久化資料。即使容器被刪除重建，Volume 中的資料依然保留。

```bash
# 查看所有 OpenClaw 相關的 Volume
docker volume ls | grep openclaw

# 查看特定 Volume 的詳細資訊
docker volume inspect openclaw-data

# Volume 資料存放位置（Linux）
# 預設: /var/lib/docker/volumes/openclaw-data/_data
```

#### networks 區塊

自訂網路確保容器間的安全通信，並與宿主機其他服務隔離。

```bash
# 查看網路
docker network ls | grep openclaw

# 檢查網路中的容器
docker network inspect openclaw-network
```

### 6.3 開發環境覆蓋配置

```yaml
# docker-compose.dev.yml
# 開發環境覆蓋配置，疊加在主 docker-compose.yml 之上使用

services:
  openclaw:
    # 使用本地構建而非官方映像
    build:
      context: .
      dockerfile: Dockerfile
      target: development

    # 掛載原始碼（實現熱重載）
    volumes:
      - ./src:/app/src
      - ./package.json:/app/package.json
      - ./openclaw.json5:/app/openclaw.json5

    # 開發環境變數
    environment:
      - NODE_ENV=development
      - LOG_LEVEL=debug

    # 開發端口
    ports:
      - "3000:3000"
      - "9229:9229"   # Node.js 調試端口

    # 取消資源限制
    deploy:
      resources:
        limits:
          cpus: "4.0"
          memory: 4G
```

使用方式：

```bash
# 使用開發環境配置啟動
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

---

## 7. 使用官方 setup.sh 腳本

OpenClaw 提供了一個互動式安裝腳本，可以自動完成大部分配置工作。

### 7.1 執行安裝腳本

```bash
# 確保在 OpenClaw 目錄下
cd /opt/openclaw

# 賦予執行權限
chmod +x setup.sh

# 執行安裝腳本
./setup.sh
```

### 7.2 安裝腳本互動流程

腳本會引導您完成以下步驟：

```
┌──────────────────────────────────────────┐
│     🐾 OpenClaw Setup Wizard v1.2.3     │
│                                          │
│  Welcome! This wizard will help you      │
│  configure OpenClaw for first use.       │
└──────────────────────────────────────────┘

Step 1/8: Bot Identity
  ? Bot name: [My OpenClaw Bot]
  ? Bot emoji: [🐾]

Step 2/8: AI Model Provider
  ? Select primary AI provider:
    ❯ OpenAI (GPT-4o, GPT-4o-mini)
      Anthropic (Claude Sonnet, Haiku)
      Google (Gemini 2.0 Flash, Pro)
      Ollama (Local models)
      OpenRouter (Multiple providers)

  ? Enter your OpenAI API Key: sk-proj-xxxxx
  ? Test API Key? [Y/n]: Y
    ✅ API Key verified successfully!

Step 3/8: Communication Channels
  ? Select channels to enable: (space to select)
    ❯ ◉ Telegram
      ◯ WhatsApp
      ◉ Discord
      ◯ Slack

  ? Enter Telegram Bot Token: 1234567890:ABCdef...
    ✅ Telegram Bot verified: @MyOpenClawBot

  ? Enter Discord Bot Token: MTIzNDU2...
    ✅ Discord Bot verified: MyOpenClaw#1234

Step 4/8: Gateway Configuration
  ? Gateway port [3000]:
  ? External URL (for webhooks): https://openclaw.example.com
  ? Enable Control UI? [Y/n]: Y
  ? Set admin password: ********

Step 5/8: Sandbox Configuration
  ? Enable code sandbox? [Y/n]: Y
  ? Sandbox timeout (seconds) [60]:
  ? Sandbox memory limit (MB) [512]:

Step 6/8: Security
  ✅ Encryption key generated
  ✅ JWT secret generated

Step 7/8: Data Storage
  ? Data directory [./data]:
  ✅ Data directories created

Step 8/8: Review & Deploy
  ┌─────────────────────────────────┐
  │ Configuration Summary           │
  ├─────────────────────────────────┤
  │ Bot Name: My OpenClaw Bot       │
  │ AI Provider: OpenAI (GPT-4o)   │
  │ Channels: Telegram, Discord     │
  │ Sandbox: Enabled                │
  │ Gateway: :3000                  │
  │ URL: openclaw.example.com       │
  └─────────────────────────────────┘

  ? Deploy now? [Y/n]: Y

  ⏳ Pulling Docker images...
  ⏳ Starting services...
  ✅ OpenClaw is running!

  🌐 Control UI: https://openclaw.example.com
  📋 View logs: docker compose logs -f
```

### 7.3 setup.sh 命令列參數

```bash
# 查看所有選項
./setup.sh --help

# 靜默安裝（使用 .env 中的配置）
./setup.sh --silent

# 僅生成配置（不啟動）
./setup.sh --config-only

# 指定 compose 檔案
./setup.sh --compose-file docker-compose.yml

# 跳過 API Key 驗證
./setup.sh --skip-verify

# 指定資料目錄
./setup.sh --data-dir /data/openclaw

# 完整範例：靜默安裝，指定配置
./setup.sh --silent --data-dir /data/openclaw --config-only
```

---

## 8. 手動 Docker Compose 配置

如果您偏好手動配置而非使用安裝腳本，請按照以下步驟操作。

### 8.1 建立資料目錄

```bash
# 建立所有必要的資料目錄
mkdir -p data/{db,logs,media,skills,backups}

# 設定權限
chmod -R 755 data/
chmod 700 data/db  # 資料庫目錄限制存取

# 確認目錄結構
tree data/ -L 1
# data/
# ├── backups/
# ├── db/
# ├── logs/
# ├── media/
# └── skills/
```

### 8.2 建立 openclaw.json5 配置檔

```bash
# 從範本複製
cp openclaw.json5.example openclaw.json5

# 或建立最小配置（詳見「配置檔完整參考」）
cat > openclaw.json5 << 'JSON5'
{
  // OpenClaw 基本配置
  identity: {
    name: "我的 OpenClaw",
    theme: "friendly",
    emoji: "🐾",
  },

  agents: {
    model: "openai/gpt-4o-mini",
    fallbacks: ["anthropic/claude-sonnet-4-20250514", "openai/gpt-4o"],
  },

  channels: {
    telegram: {
      enabled: true,
    },
  },

  gateway: {
    port: 3000,
    controlUi: {
      enabled: true,
    },
  },
}
JSON5
```

### 8.3 拉取 Docker 映像

```bash
# 拉取所有需要的映像
docker compose pull

# 或分別拉取
docker pull ghcr.io/open-claw/open-claw:latest
docker pull ghcr.io/open-claw/open-claw-sandbox:latest

# 查看已下載的映像
docker images | grep open-claw
```

### 8.4 自訂 docker-compose.yml

如果您需要自訂配置，建議使用 `docker-compose.override.yml`（會自動與主文件合併）：

```yaml
# docker-compose.override.yml
# 此文件會自動與 docker-compose.yml 合併

services:
  openclaw:
    # 自訂環境變數
    environment:
      - LOG_LEVEL=debug

    # 額外的端口映射
    ports:
      - "3000:3000"
      - "3001:3001"  # 額外的 API 端口

    # 額外的 volume 掛載
    volumes:
      - /path/to/custom/skills:/app/custom-skills:ro

    # 自訂資源限制
    deploy:
      resources:
        limits:
          cpus: "4.0"
          memory: 4G

  # 添加 PostgreSQL 資料庫
  postgres:
    image: postgres:16-alpine
    container_name: openclaw-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: openclaw
      POSTGRES_USER: openclaw
      POSTGRES_PASSWORD: ${DB_PASSWORD:-changeme}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - openclaw-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U openclaw"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres-data:
```

### 8.5 不使用沙箱的精簡配置

如果您不需要沙箱功能，可以使用更精簡的配置：

```yaml
# docker-compose.minimal.yml
services:
  openclaw:
    image: ghcr.io/open-claw/open-claw:latest
    container_name: openclaw
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "3000:3000"
    volumes:
      - ./data:/app/data
      - ./openclaw.json5:/app/openclaw.json5:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

啟動方式：

```bash
docker compose -f docker-compose.minimal.yml up -d
```

---

## 9. 首次啟動與驗證

### 9.1 啟動服務

```bash
# 啟動所有服務（背景執行）
docker compose up -d

# 觀察啟動過程（前景執行，Ctrl+C 退出觀察但不停止服務）
docker compose up -d && docker compose logs -f

# 僅啟動核心服務（不啟動沙箱）
docker compose up -d openclaw
```

### 9.2 確認服務狀態

```bash
# 查看所有服務狀態
docker compose ps

# 期望輸出：
# NAME                STATUS          PORTS
# openclaw            Up (healthy)    0.0.0.0:3000->3000/tcp
# openclaw-sandbox    Up (healthy)    8080/tcp

# 查看容器詳細資訊
docker inspect openclaw --format '{{.State.Status}} - {{.State.Health.Status}}'
# 期望輸出: running - healthy

# 查看資源使用狀況
docker stats --no-stream openclaw openclaw-sandbox
```

### 9.3 健康檢查

```bash
# 方法 1：使用 curl 檢查健康端點
curl -s http://localhost:3000/health | jq .

# 期望回應：
# {
#   "status": "ok",
#   "version": "1.2.3",
#   "uptime": 45,
#   "services": {
#     "ai": "connected",
#     "channels": {
#       "telegram": "connected",
#       "discord": "connected"
#     },
#     "sandbox": "available",
#     "database": "connected"
#   }
# }

# 方法 2：使用 OpenClaw 內建的 doctor 命令
docker compose exec openclaw openclaw doctor

# 期望輸出：
# 🔍 Running diagnostics...
#
# ✅ Node.js v24.x.x
# ✅ Configuration valid
# ✅ OpenAI API: Connected (GPT-4o available)
# ✅ Telegram: Connected (@MyBot)
# ✅ Discord: Connected (MyBot#1234)
# ✅ Sandbox: Available
# ✅ Database: Connected (SQLite)
# ✅ Disk space: 15.2 GB available
# ✅ Memory: 1.2 GB / 2.0 GB used
#
# 🎉 All checks passed!
```

### 9.4 測試通道連線

#### 測試 Telegram

```bash
# 確認 Telegram Webhook 已設定
curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getWebhookInfo" | jq .

# 手動設定 Webhook（如果自動設定失敗）
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setWebhook" \
  -H "Content-Type: application/json" \
  -d "{
    \"url\": \"${GATEWAY_BASE_URL}/webhook/telegram\",
    \"allowed_updates\": [\"message\", \"callback_query\", \"inline_query\"]
  }" | jq .
```

#### 測試 Discord

```bash
# Discord 使用 WebSocket 而非 Webhook
# 確認 Bot 已連線：在 Discord 伺服器中查看 Bot 是否上線
# 或查看日誌中的連線訊息
docker compose logs openclaw | grep -i "discord.*connect"
```

### 9.5 首次對話測試

在 Telegram 或其他已設定的通道中：

```
使用者: 你好，你是誰？
Bot:    你好！我是 OpenClaw，一個 AI 助手。🐾 有什麼我可以幫助你的嗎？

使用者: 1+1等於多少？
Bot:    1 + 1 = 2

使用者: /help
Bot:    以下是可用的指令：
        /help - 顯示幫助資訊
        /reset - 重置對話
        /model - 查看/切換模型
        /status - 查看系統狀態
```

---

## 10. 日誌查看與監控

### 10.1 查看日誌

```bash
# 即時查看所有服務的日誌
docker compose logs -f

# 即時查看特定服務的日誌
docker compose logs -f openclaw
docker compose logs -f sandbox

# 查看最近 100 行日誌
docker compose logs --tail 100 openclaw

# 查看特定時間範圍的日誌
docker compose logs --since "2025-01-15T00:00:00" --until "2025-01-15T23:59:59" openclaw

# 查看日誌並過濾關鍵字
docker compose logs openclaw | grep -i "error"
docker compose logs openclaw | grep -i "webhook"

# 僅查看錯誤和警告
docker compose logs openclaw 2>&1 | grep -E "(ERROR|WARN)"
```

### 10.2 日誌檔案位置

```bash
# Docker 容器日誌位置（JSON 格式）
# Linux: /var/lib/docker/containers/<container-id>/<container-id>-json.log
docker inspect openclaw --format '{{.LogPath}}'

# OpenClaw 應用程式日誌（映射到宿主機）
ls -la data/logs/

# 日誌檔案結構
# data/logs/
# ├── openclaw.log          # 主日誌
# ├── openclaw-error.log    # 錯誤日誌
# ├── access.log            # HTTP 存取日誌
# ├── webhook.log           # Webhook 日誌
# └── sandbox.log           # 沙箱日誌
```

### 10.3 日誌等級說明

| 等級 | 說明 | 使用場景 |
|------|------|----------|
| `debug` | 詳細的調試資訊 | 開發和排錯 |
| `info` | 一般運行資訊 | 生產環境（預設） |
| `warn` | 警告訊息 | 需要注意的問題 |
| `error` | 錯誤訊息 | 需要修復的問題 |

```bash
# 變更日誌等級（不重啟）
docker compose exec openclaw openclaw config set logging.level debug

# 或修改 .env 並重啟
# LOG_LEVEL=debug
docker compose restart openclaw
```

### 10.4 日誌輪替配置

```bash
# Docker daemon 層級的日誌輪替（/etc/docker/daemon.json）
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",    # 每個日誌檔案最大 10 MB
    "max-file": "5"       # 保留 5 個日誌檔案
  }
}
```

### 10.5 進階監控

#### 使用 Docker 內建監控

```bash
# 即時資源監控
docker stats openclaw openclaw-sandbox

# 輸出範例：
# CONTAINER     CPU %   MEM USAGE / LIMIT   MEM %   NET I/O         BLOCK I/O
# openclaw      2.34%   345.2MiB / 2GiB     16.86%  1.23MB/456KB    12.3MB/1.2MB
# openclaw-sb   0.12%   128.5MiB / 512MiB   25.10%  234KB/56KB      1.2MB/256KB

# 查看容器事件
docker events --filter container=openclaw --since "1h"

# 查看容器行程
docker top openclaw
```

#### 使用 Prometheus + Grafana（進階）

```yaml
# 在 docker-compose.override.yml 中添加監控服務
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: openclaw-prometheus
    restart: unless-stopped
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - openclaw-network

  grafana:
    image: grafana/grafana:latest
    container_name: openclaw-grafana
    restart: unless-stopped
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3100:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - openclaw-network

volumes:
  prometheus-data:
  grafana-data:
```

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'openclaw'
    static_configs:
      - targets: ['openclaw:3000']
    metrics_path: '/metrics'
```

---

## 11. 升級流程

### 11.1 標準升級流程

```bash
# 步驟 1：備份（重要！）
./scripts/backup.sh
# 或手動備份
docker compose exec openclaw openclaw backup create
cp -r data/ data.backup.$(date +%Y%m%d)/

# 步驟 2：查看最新版本和變更日誌
git fetch --tags
LATEST=$(git tag --sort=-version:refname | grep -v "alpha\|beta\|rc" | head -1)
echo "最新版本: $LATEST"
echo "當前版本: $(git describe --tags)"

# 查看版本差異
git log $(git describe --tags)..$LATEST --oneline

# 步驟 3：停止服務
docker compose down

# 步驟 4：更新原始碼
git checkout main
git pull origin main
git checkout $LATEST

# 步驟 5：拉取新映像
docker compose pull

# 步驟 6：啟動新版本
docker compose up -d

# 步驟 7：驗證
docker compose ps
curl -s http://localhost:3000/health | jq .
docker compose exec openclaw openclaw doctor

# 步驟 8：清理舊映像（可選）
docker image prune -f
```

### 11.2 自動升級腳本

```bash
#!/usr/bin/env bash
# upgrade.sh — OpenClaw 自動升級腳本

set -euo pipefail

INSTALL_DIR="/opt/openclaw"
BACKUP_DIR="${INSTALL_DIR}/data/backups"

cd "$INSTALL_DIR"

echo "🔍 檢查更新..."

# 取得最新版本
git fetch --tags --quiet
CURRENT=$(git describe --tags 2>/dev/null || echo "unknown")
LATEST=$(git tag --sort=-version:refname | grep -v "alpha\|beta\|rc" | head -1)

if [ "$CURRENT" = "$LATEST" ]; then
  echo "✅ 已是最新版本 ($CURRENT)"
  exit 0
fi

echo "📦 發現新版本: $CURRENT → $LATEST"

# 備份
echo "💾 建立備份..."
mkdir -p "$BACKUP_DIR"
BACKUP_NAME="backup-${CURRENT}-$(date +%Y%m%d-%H%M%S)"
cp -r data/ "${BACKUP_DIR}/${BACKUP_NAME}/"
echo "   備份位置: ${BACKUP_DIR}/${BACKUP_NAME}/"

# 停止服務
echo "⏹ 停止服務..."
docker compose down

# 更新
echo "📥 更新到 $LATEST..."
git checkout "$LATEST"

# 拉取映像
echo "🐳 拉取新映像..."
docker compose pull

# 啟動
echo "▶️ 啟動服務..."
docker compose up -d

# 等待健康檢查
echo "⏳ 等待服務就緒..."
for i in $(seq 1 30); do
  if curl -sf http://localhost:3000/health > /dev/null 2>&1; then
    echo "✅ 升級成功！版本: $LATEST"
    exit 0
  fi
  sleep 2
done

echo "❌ 服務啟動逾時，嘗試回滾..."
docker compose down
git checkout "$CURRENT"
docker compose pull
docker compose up -d
echo "⚠️ 已回滾到 $CURRENT"
exit 1
```

### 11.3 零停機升級（進階）

對於需要零停機的生產環境，可使用藍綠部署策略：

```bash
# 藍綠部署範例（需要 Nginx 作為前端）

# 步驟 1：啟動新版本在不同端口
GATEWAY_PORT=3001 docker compose -p openclaw-green up -d

# 步驟 2：等待新版本就緒
until curl -sf http://localhost:3001/health; do sleep 2; done

# 步驟 3：切換 Nginx 上游（使用 Nginx 配置重載）
sudo sed -i 's/proxy_pass http:\/\/127.0.0.1:3000/proxy_pass http:\/\/127.0.0.1:3001/' \
  /etc/nginx/sites-available/openclaw
sudo nginx -s reload

# 步驟 4：停止舊版本
docker compose -p openclaw-blue down

# 步驟 5：將新版本改回標準端口
docker compose -p openclaw-green down
GATEWAY_PORT=3000 docker compose up -d
sudo sed -i 's/proxy_pass http:\/\/127.0.0.1:3001/proxy_pass http:\/\/127.0.0.1:3000/' \
  /etc/nginx/sites-available/openclaw
sudo nginx -s reload
```

---

## 12. 備份與還原

### 12.1 備份策略

| 資料類型 | 位置 | 備份頻率 | 重要性 |
|----------|------|----------|--------|
| **配置檔** | `.env`, `openclaw.json5` | 每次修改後 | ⭐⭐⭐⭐⭐ |
| **資料庫** | `data/db/` | 每日 | ⭐⭐⭐⭐⭐ |
| **對話記錄** | `data/db/conversations.db` | 每日 | ⭐⭐⭐⭐ |
| **自訂技能** | `skills/` | 每次修改後 | ⭐⭐⭐⭐ |
| **媒體檔案** | `data/media/` | 每週 | ⭐⭐⭐ |
| **日誌檔案** | `data/logs/` | 不必要 | ⭐ |

### 12.2 手動備份

```bash
# 完整備份
backup_name="openclaw-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "data/backups/${backup_name}"

# 備份資料庫（建議先停止服務以確保一致性）
docker compose stop openclaw
cp -r data/db/ "data/backups/${backup_name}/db/"
docker compose start openclaw

# 備份配置
cp .env "data/backups/${backup_name}/"
cp openclaw.json5 "data/backups/${backup_name}/"
cp docker-compose.yml "data/backups/${backup_name}/"
cp -r skills/ "data/backups/${backup_name}/skills/" 2>/dev/null || true

# 壓縮備份
tar -czf "data/backups/${backup_name}.tar.gz" \
  -C "data/backups/" "${backup_name}/"
rm -rf "data/backups/${backup_name}/"

echo "✅ 備份完成: data/backups/${backup_name}.tar.gz"
echo "   大小: $(du -h "data/backups/${backup_name}.tar.gz" | cut -f1)"
```

### 12.3 使用 OpenClaw 內建備份命令

```bash
# 建立備份
docker compose exec openclaw openclaw backup create

# 建立具名備份
docker compose exec openclaw openclaw backup create --name "pre-upgrade-v1.2.3"

# 列出所有備份
docker compose exec openclaw openclaw backup list

# 輸出：
# ID          NAME                    DATE                 SIZE
# bk_abc123   auto-daily-20250115     2025-01-15 03:00    2.3 MB
# bk_def456   pre-upgrade-v1.2.3      2025-01-15 14:30    2.1 MB
# bk_ghi789   auto-daily-20250114     2025-01-14 03:00    2.2 MB
```

### 12.4 自動備份腳本

```bash
#!/usr/bin/env bash
# auto-backup.sh — OpenClaw 自動備份腳本

set -euo pipefail

INSTALL_DIR="/opt/openclaw"
BACKUP_DIR="${INSTALL_DIR}/data/backups"
RETENTION_DAYS=30  # 保留 30 天的備份

cd "$INSTALL_DIR"

# 建立備份
echo "$(date '+%Y-%m-%d %H:%M:%S') - 開始備份..."

backup_name="auto-$(date +%Y%m%d-%H%M%S)"
mkdir -p "${BACKUP_DIR}"

# 使用 SQLite 的線上備份功能（不需要停止服務）
docker compose exec -T openclaw sqlite3 /app/data/db/openclaw.db ".backup '/app/data/backups/${backup_name}.db'"

# 備份配置
tar -czf "${BACKUP_DIR}/${backup_name}.tar.gz" \
  --exclude='data/backups' \
  --exclude='data/logs' \
  .env \
  openclaw.json5 \
  data/db/ \
  skills/ 2>/dev/null || true

echo "$(date '+%Y-%m-%d %H:%M:%S') - 備份完成: ${backup_name}"

# 清理過期備份
find "${BACKUP_DIR}" -name "auto-*" -mtime "+${RETENTION_DAYS}" -delete
echo "$(date '+%Y-%m-%d %H:%M:%S') - 已清理 ${RETENTION_DAYS} 天前的備份"

# 顯示備份狀態
echo "目前備份數量: $(ls -1 "${BACKUP_DIR}"/auto-* 2>/dev/null | wc -l)"
echo "備份總大小: $(du -sh "${BACKUP_DIR}" | cut -f1)"
```

設定每日自動備份（cron）：

```bash
# 每天凌晨 3 點執行備份
echo "0 3 * * * /opt/openclaw/scripts/auto-backup.sh >> /opt/openclaw/data/logs/backup.log 2>&1" | \
  crontab -
```

### 12.5 還原備份

```bash
# 方法 1：使用 OpenClaw 內建還原命令
docker compose exec openclaw openclaw backup restore bk_abc123

# 方法 2：手動還原
# 步驟 1：停止服務
docker compose down

# 步驟 2：解壓備份
tar -xzf data/backups/auto-20250115-030000.tar.gz -C /opt/openclaw/

# 步驟 3：還原資料庫
cp data/backups/auto-20250115-030000.db data/db/openclaw.db

# 步驟 4：重新啟動
docker compose up -d

# 步驟 5：驗證
curl -s http://localhost:3000/health | jq .
```

### 12.6 異地備份（推薦）

```bash
# 備份到 S3
aws s3 sync data/backups/ s3://my-openclaw-backups/

# 備份到 Google Cloud Storage
gsutil rsync -r data/backups/ gs://my-openclaw-backups/

# 備份到另一台伺服器
rsync -avz --progress data/backups/ user@backup-server:/backups/openclaw/

# 使用 rclone（支援多種雲端儲存）
rclone sync data/backups/ remote:openclaw-backups/
```

---

## 13. 常見問題排解

### 13.1 容器無法啟動

#### 問題：容器不斷重啟

```bash
# 查看容器狀態
docker compose ps
# 如果 STATUS 顯示 "Restarting"

# 查看容器日誌
docker compose logs --tail 50 openclaw

# 常見原因和解決方案：

# 原因 1：端口被佔用
sudo ss -tlnp | grep :3000
# 解決：停止佔用端口的程序，或在 .env 中變更 GATEWAY_PORT

# 原因 2：.env 文件格式錯誤
# 檢查是否有多餘的空格或特殊字符
cat -A .env | head -20
# 解決：確保每行格式為 KEY=VALUE，沒有多餘空格

# 原因 3：Volume 權限問題
ls -la data/
# 解決：修正權限
sudo chown -R 1000:1000 data/
```

#### 問題：健康檢查失敗

```bash
# 手動執行健康檢查
docker compose exec openclaw curl -f http://localhost:3000/health

# 如果返回錯誤，查看詳細日誌
docker compose logs --tail 100 openclaw | grep -E "(ERROR|health)"

# 增加健康檢查的等待時間
# 在 docker-compose.override.yml 中：
# healthcheck:
#   start_period: 120s  # 增加到 120 秒
```

### 13.2 API 連線問題

```bash
# 問題：AI 模型 API 無法連線

# 步驟 1：從容器內測試連線
docker compose exec openclaw curl -s https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY" | head -5

# 步驟 2：檢查 DNS 解析
docker compose exec openclaw nslookup api.openai.com

# 步驟 3：檢查防火牆
sudo iptables -L -n | grep -E "(DROP|REJECT)"

# 步驟 4：如果在企業網路環境中，可能需要設定代理
# 在 .env 中添加：
# HTTP_PROXY=http://proxy.company.com:8080
# HTTPS_PROXY=http://proxy.company.com:8080
# NO_PROXY=localhost,127.0.0.1
```

### 13.3 Webhook 問題

```bash
# 問題：Telegram/WhatsApp 無法接收訊息

# 步驟 1：確認外部 URL 可存取
curl -I https://openclaw.yourdomain.com/health

# 步驟 2：確認 SSL 憑證有效
openssl s_client -connect openclaw.yourdomain.com:443 </dev/null 2>/dev/null | \
  openssl x509 -noout -dates

# 步驟 3：確認 Webhook 已設定（以 Telegram 為例）
source .env
curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getWebhookInfo" | jq .

# 步驟 4：手動重設 Webhook
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setWebhook" \
  -d "url=https://openclaw.yourdomain.com/webhook/telegram" | jq .

# 步驟 5：檢查 Nginx 反向代理日誌
sudo tail -20 /var/log/nginx/error.log
sudo tail -20 /var/log/nginx/access.log | grep webhook
```

### 13.4 記憶體不足

```bash
# 問題：容器被 OOM Killer 終止

# 查看 OOM 事件
docker inspect openclaw | jq '.[0].State.OOMKilled'
dmesg | grep -i "oom\|out of memory" | tail -5

# 解決方案 1：增加容器記憶體限制
# 在 docker-compose.override.yml 中：
# deploy:
#   resources:
#     limits:
#       memory: 4G

# 解決方案 2：增加系統 swap
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 13.5 磁碟空間不足

```bash
# 查看 Docker 空間使用
docker system df

# 清理未使用的資源
docker system prune -a --volumes

# ⚠️ 上面的命令會清理所有未使用的映像和 volume
# 更安全的清理方式：

# 只清理懸垂映像
docker image prune -f

# 清理構建快取
docker builder prune -f

# 清理日誌
truncate -s 0 data/logs/*.log
```

### 13.6 Docker Compose 版本問題

```bash
# 問題：docker compose 命令不存在

# 確認 Docker Compose v2 是否安裝
docker compose version

# 如果返回錯誤，手動安裝 Compose 插件
DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins
curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o $DOCKER_CONFIG/cli-plugins/docker-compose
chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose

# 再次確認
docker compose version
```

### 13.7 快速診斷腳本

```bash
#!/usr/bin/env bash
# diagnose.sh — OpenClaw 快速診斷腳本

echo "=== OpenClaw 診斷報告 ==="
echo "時間: $(date)"
echo ""

echo "--- Docker 狀態 ---"
docker --version
docker compose version
docker compose ps
echo ""

echo "--- 容器資源使用 ---"
docker stats --no-stream openclaw openclaw-sandbox 2>/dev/null
echo ""

echo "--- 健康檢查 ---"
curl -s http://localhost:3000/health | jq . 2>/dev/null || echo "健康端點無回應"
echo ""

echo "--- 最近的錯誤日誌 ---"
docker compose logs --tail 20 openclaw 2>&1 | grep -i error || echo "無錯誤"
echo ""

echo "--- 磁碟空間 ---"
df -h / | tail -1
docker system df
echo ""

echo "--- 網路連線 ---"
for host in api.openai.com api.anthropic.com; do
  if curl -s --connect-timeout 5 -o /dev/null "https://$host"; then
    echo "✅ $host"
  else
    echo "❌ $host"
  fi
done
```

---

## 14. 進階：Nginx SSL 反向代理配置

### 14.1 為什麼需要反向代理

在生產環境中，直接暴露 OpenClaw 的 3000 端口是不安全的。使用 Nginx 反向代理可以提供：

- **SSL/TLS 終止**：在 Nginx 層處理 HTTPS 加密
- **請求限流**：防止 DDoS 攻擊和過度使用
- **安全標頭**：自動添加安全相關的 HTTP 標頭
- **存取控制**：IP 白名單、基本認證等
- **靜態檔案快取**：減少後端負載
- **日誌管理**：統一的存取日誌格式

### 14.2 使用 Docker Compose 部署 Nginx

```yaml
# 在 docker-compose.override.yml 中添加 Nginx 服務
services:
  nginx:
    image: nginx:alpine
    container_name: openclaw-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./nginx/logs:/var/log/nginx
      - certbot-webroot:/var/www/certbot:ro
      - certbot-certs:/etc/letsencrypt:ro
    depends_on:
      - openclaw
    networks:
      - openclaw-network

  # Certbot 用於自動續期 SSL 憑證
  certbot:
    image: certbot/certbot
    container_name: openclaw-certbot
    volumes:
      - certbot-webroot:/var/www/certbot
      - certbot-certs:/etc/letsencrypt
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  certbot-webroot:
  certbot-certs:
```

### 14.3 Nginx 配置詳解

```bash
# 建立 Nginx 配置目錄
mkdir -p nginx/{conf.d,ssl,logs}
```

```nginx
# nginx/conf.d/openclaw.conf
# OpenClaw Nginx 反向代理完整配置

# 請求限流區域定義
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/m;
limit_req_zone $binary_remote_addr zone=webhook:10m rate=60r/m;
limit_req_zone $binary_remote_addr zone=general:10m rate=120r/m;

# 上游定義
upstream openclaw_backend {
    server openclaw:3000;
    keepalive 32;
}

# HTTP → HTTPS 重導向
server {
    listen 80;
    listen [::]:80;
    server_name openclaw.yourdomain.com;

    # Let's Encrypt ACME 挑戰
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # 其他所有請求重導向到 HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS 主配置
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name openclaw.yourdomain.com;

    # ============================
    # SSL 配置
    # ============================
    ssl_certificate /etc/letsencrypt/live/openclaw.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/openclaw.yourdomain.com/privkey.pem;

    # SSL 安全設定
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # ============================
    # 安全標頭
    # ============================
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self';" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

    # ============================
    # 一般設定
    # ============================
    client_max_body_size 50M;
    client_body_timeout 60s;
    client_header_timeout 60s;

    # Gzip 壓縮
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # ============================
    # Webhook 端點（高優先級）
    # ============================
    location /webhook/ {
        limit_req zone=webhook burst=20 nodelay;

        proxy_pass http://openclaw_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Webhook 需要較長的逾時
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;

        # 不快取 Webhook 請求
        proxy_no_cache 1;
        proxy_cache_bypass 1;
    }

    # ============================
    # API 端點
    # ============================
    location /api/ {
        limit_req zone=api burst=10 nodelay;

        proxy_pass http://openclaw_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 120s;
    }

    # ============================
    # 控制台 UI
    # ============================
    location /ui/ {
        limit_req zone=general burst=30 nodelay;

        proxy_pass http://openclaw_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 支援（控制台即時更新）
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # ============================
    # 健康檢查端點（不限流）
    # ============================
    location /health {
        proxy_pass http://openclaw_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;

        access_log off;
    }

    # ============================
    # 預設位置
    # ============================
    location / {
        limit_req zone=general burst=30 nodelay;

        proxy_pass http://openclaw_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 120s;
    }

    # ============================
    # 禁止存取的路徑
    # ============================
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

### 14.4 取得 SSL 憑證（Let's Encrypt）

```bash
# 首次取得憑證（Nginx 尚未啟動前）
# 使用 standalone 模式
docker run --rm \
  -p 80:80 \
  -v certbot-certs:/etc/letsencrypt \
  -v certbot-webroot:/var/www/certbot \
  certbot/certbot certonly \
  --standalone \
  --email your@email.com \
  --agree-tos \
  --no-eff-email \
  -d openclaw.yourdomain.com

# 或者，如果 Nginx 已經在運行
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path /var/www/certbot \
  --email your@email.com \
  --agree-tos \
  --no-eff-email \
  -d openclaw.yourdomain.com
```

### 14.5 替代方案：使用 Caddy（更簡單）

如果您覺得 Nginx 配置太複雜，可以使用 Caddy。Caddy 會自動處理 SSL 憑證。

```yaml
# docker-compose.override.yml 使用 Caddy
services:
  caddy:
    image: caddy:2-alpine
    container_name: openclaw-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config
    depends_on:
      - openclaw
    networks:
      - openclaw-network

volumes:
  caddy-data:
  caddy-config:
```

```
# Caddyfile
openclaw.yourdomain.com {
    reverse_proxy openclaw:3000

    # 請求限流
    rate_limit {
        zone dynamic {
            key {remote_host}
            events 60
            window 1m
        }
    }

    # 安全標頭
    header {
        X-Frame-Options "SAMEORIGIN"
        X-Content-Type-Options "nosniff"
        X-XSS-Protection "1; mode=block"
        Strict-Transport-Security "max-age=63072000"
        Referrer-Policy "strict-origin-when-cross-origin"
    }

    # 日誌
    log {
        output file /var/log/caddy/access.log
        format json
    }
}
```

### 14.6 驗證反向代理設定

```bash
# 測試 HTTP 到 HTTPS 重導向
curl -I http://openclaw.yourdomain.com
# 期望: 301 Moved Permanently → https://...

# 測試 HTTPS
curl -I https://openclaw.yourdomain.com
# 期望: 200 OK

# 測試安全標頭
curl -I https://openclaw.yourdomain.com 2>/dev/null | grep -E "^(X-|Strict|Content-Security|Referrer)"

# 測試 SSL 評分
# 前往 https://www.ssllabs.com/ssltest/ 輸入您的域名
# 目標評分: A 或 A+

# 測試 Webhook 端點
curl -X POST https://openclaw.yourdomain.com/webhook/telegram \
  -H "Content-Type: application/json" \
  -d '{"test": true}'
```

---

## 下一步

完成 Docker 部署後，建議：

1. 閱讀 **[配置檔完整參考](04-配置檔完整參考.md)** 以了解所有配置選項
2. 閱讀 **記憶與人格篇** 自訂您的 Bot 人格
3. 閱讀 **Skills 開發篇** 為 Bot 添加自訂技能
4. 設定 **自動備份** 和 **監控告警**

---

> **📝 附註**：本指南以 Docker Compose v2 為基礎撰寫。如果您使用的是 Docker Swarm 或 Kubernetes，部署方式將有所不同，請參考進階部署文件。
