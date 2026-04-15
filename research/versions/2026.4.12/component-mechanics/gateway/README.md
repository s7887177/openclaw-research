# Gateway 元件機制：OpenClaw 中央伺服器

> **層級**：Level 2 — Component Mechanics  
> **版本**：2026.4.12  
> **範圍**：`src/gateway/` 目錄（約 284 個檔案）

## 本章摘要

Gateway 是 OpenClaw 系統的**中央通訊樞紐**——它同時扮演 HTTP 伺服器（REST API）、WebSocket 伺服器（即時雙向通訊）、以及 Plugin 宿主（Plugin Host）三重角色。所有外部客戶端（CLI、瀏覽器 Control UI、行動裝置、Channel Adapter）都必須經由 Gateway 與系統互動。

本章從原始碼逐一拆解 Gateway 的內部機制，涵蓋：

| 子章節 | 檔案 | 內容 |
|--------|------|------|
| [01-伺服器啟動與生命週期](./01-server-lifecycle.md) | `server.impl.ts`, `server.ts`, `boot.ts` | 啟動流程、lazy loading、BOOT.md 執行 |
| [02-HTTP 控制平面](./02-http-control-plane.md) | `server-http.ts`, `http-common.ts` | REST 路由、端點、速率限制 |
| [03-WebSocket 協議](./03-websocket-protocol.md) | `protocol/`, `server-methods-list.ts` | 方法清單、事件、schema |
| [04-認證與配對](./04-auth-and-pairing.md) | `auth.ts`, `device-auth.ts`, `startup-auth.ts` | 認證模式、Device Pairing |
| [05-Session 與 Chat 管理](./05-session-chat.md) | `server-chat.ts`, `session-utils.ts` | Session 生命週期、聊天管理 |
| [06-Tool 調用與核准](./06-tool-approval.md) | `tools-invoke-http.ts`, `exec-approval-manager.ts` | 工具執行、核准工作流 |
| [07-Plugin 與配置管理](./07-plugin-config.md) | `server-plugin-bootstrap.ts`, `config-reload.ts` | Plugin 載入、熱重載 |

---

## 目錄結構總覽

Gateway 的程式碼位於 `src/gateway/`，以下為關鍵檔案分類：

### 核心伺服器

| 檔案 | 用途 | 行數 |
|------|------|------|
| `server.impl.ts` | 主要實作，包含 `startGatewayServer()` | ~856 行 |
| `server.ts` | Lazy-loading 包裝器 | 18 行 |
| `boot.ts` | BOOT.md 啟動腳本執行 | 205 行 |
| `server-constants.ts` | 全域常數定義 | 28 行 |
| `events.ts` | 事件類型定義 | 8 行 |

### HTTP 層

| 檔案 | 用途 |
|------|------|
| `server-http.ts` | HTTP 路由註冊與請求處理管線 |
| `http-common.ts` | 共用 HTTP 回應工具 |
| `http-endpoint-helpers.ts` | POST JSON 端點統一處理 |
| `control-ui-routing.ts` | Control UI SPA 路由分類 |
| `control-plane-rate-limit.ts` | 控制平面寫入速率限制 |

### WebSocket 層

| 檔案 | 用途 |
|------|------|
| `server-ws-runtime.ts` | WS 連線管理入口 |
| `server-methods-list.ts` | 134 個 WS 方法 + 25 個事件 |
| `server-methods.ts` | Handler 註冊與授權 |
| `ws-log.ts` | WS 日誌記錄（多層級） |
| `protocol/` | Schema 定義（AJV 驗證） |

### 認證與配對

| 檔案 | 用途 |
|------|------|
| `auth.ts` | 認證邏輯核心 |
| `connection-auth.ts` | 連線級認證解析 |
| `startup-auth.ts` | 啟動時認證引導 |
| `device-auth.ts` | 設備認證 payload 建構 |
| `auth-rate-limit.ts` | 認證速率限制器 |

### Session 管理

| 檔案 | 用途 |
|------|------|
| `server-chat.ts` | 聊天運行佇列 |
| `session-utils.ts` | Session 工具函式 |
| `session-utils.types.ts` | `GatewaySessionRow` 型別 |
| `session-lifecycle-state.ts` | 生命週期狀態追蹤 |
| `session-history-state.ts` | 歷史記錄 SSE 管理 |

### Plugin 與配置

| 檔案 | 用途 |
|------|------|
| `server-plugin-bootstrap.ts` | Plugin 啟動載入 |
| `server-plugins.ts` | Plugin 執行時管理 |
| `server-startup-plugins.ts` | 啟動前 Plugin 準備 |
| `config-reload.ts` | 動態配置重載 |
| `server-maintenance.ts` | 週期維護任務 |

---

## Docker 部署配置

Gateway 透過 Docker Compose 部署，預設配置如下：

```yaml
# source-repo/docker-compose.yml:1-50
services:
  openclaw-gateway:
    image: ${OPENCLAW_IMAGE:-openclaw:local}
    ports:
      - "${OPENCLAW_GATEWAY_PORT:-18789}:18789"   # 主 HTTP/WS 伺服器
      - "${OPENCLAW_BRIDGE_PORT:-18790}:18790"     # Bridge 伺服器
    command: ["node", "dist/index.js", "gateway", "--bind", "${OPENCLAW_GATEWAY_BIND:-lan}", "--port", "18789"]
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:18789/healthz').then(...)"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 20s
```

> **引用來源**：`source-repo/docker-compose.yml:1-50`

**關鍵環境變數**：

| 變數 | 預設值 | 說明 |
|------|--------|------|
| `OPENCLAW_GATEWAY_PORT` | `18789` | Gateway 主埠 |
| `OPENCLAW_BRIDGE_PORT` | `18790` | Bridge 埠 |
| `OPENCLAW_GATEWAY_TOKEN` | （無） | 認證 token |
| `OPENCLAW_GATEWAY_BIND` | `lan` | 綁定模式 |
| `OPENCLAW_CONFIG_DIR` | （必填） | 配置目錄 |
| `OPENCLAW_WORKSPACE_DIR` | （必填） | 工作區目錄 |
| `OPENCLAW_TZ` | `UTC` | 時區 |

---

## 引用來源總表

| 引用 | 檔案路徑 | 行號 |
|------|----------|------|
| GatewayServer 型別 | `source-repo/src/gateway/server.impl.ts` | 147-149 |
| GatewayServerOptions 型別 | `source-repo/src/gateway/server.impl.ts` | 151-204 |
| startGatewayServer() 簽名 | `source-repo/src/gateway/server.impl.ts` | 206-209 |
| Lazy-loading 包裝 | `source-repo/src/gateway/server.ts` | 1-18 |
| MAX_PAYLOAD_BYTES | `source-repo/src/gateway/server-constants.ts` | 3 |
| TICK_INTERVAL_MS | `source-repo/src/gateway/server-constants.ts` | 24 |
| Docker Compose | `source-repo/docker-compose.yml` | 1-50 |
| BASE_METHODS | `source-repo/src/gateway/server-methods-list.ts` | 4-134 |
| GATEWAY_EVENTS | `source-repo/src/gateway/server-methods-list.ts` | 141-166 |
