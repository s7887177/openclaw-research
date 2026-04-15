# Gateway 子章節 01：伺服器啟動與生命週期

> **引用範圍**：`src/gateway/server.impl.ts`, `server.ts`, `boot.ts`, `server-startup*.ts`

## 1. Lazy Loading 入口：server.ts

Gateway 的公開入口是 `server.ts`，它**不直接載入**實作邏輯，而是透過動態 `import()` 實現 lazy loading（延遲載入）：

```typescript
// source-repo/src/gateway/server.ts:3-12
async function loadServerImpl() {
  return await import("./server.impl.js");
}

export async function startGatewayServer(
  ...args: Parameters<typeof import("./server.impl.js").startGatewayServer>
): ReturnType<typeof import("./server.impl.js").startGatewayServer> {
  const mod = await loadServerImpl();
  return await mod.startGatewayServer(...args);
}
```

**設計目的**：`server.impl.ts` 有大量依賴（856 行程式碼、數十個 import），延遲載入可讓不需要 Gateway 功能的 CLI 子命令（如 `openclaw version`）快速啟動。

> **引用來源**：`source-repo/src/gateway/server.ts:1-18`

---

## 2. 主函式：startGatewayServer()

這是 Gateway 的**核心啟動函式**，負責建立整個伺服器基礎設施：

```typescript
// source-repo/src/gateway/server.impl.ts:206-209
export async function startGatewayServer(
  port = 18789,
  opts: GatewayServerOptions = {},
): Promise<GatewayServer>
```

### 2.1 GatewayServer 回傳型別

```typescript
// source-repo/src/gateway/server.impl.ts:147-149
export type GatewayServer = {
  close: (opts?: { reason?: string; restartExpectedMs?: number | null }) => Promise<void>;
};
```

回傳的物件只暴露一個 `close()` 方法，支援優雅關閉（Graceful Shutdown），可附帶關閉原因與重啟預期時間。

### 2.2 GatewayServerOptions 配置

```typescript
// source-repo/src/gateway/server.impl.ts:151-204
export type GatewayServerOptions = {
  bind?: "loopback" | "lan" | "tailnet" | "auto";  // 綁定位址策略
  host?: string;                                     // 進階位址覆寫
  controlUiEnabled?: boolean;                        // 是否啟用 Control UI
  openAiChatCompletionsEnabled?: boolean;            // /v1/chat/completions 端點
  openResponsesEnabled?: boolean;                    // /v1/responses 端點
  auth?: GatewayAuthConfig;                          // 認證配置覆寫
  tailscale?: GatewayTailscaleConfig;                // Tailscale 配置覆寫
  allowCanvasHostInTests?: boolean;                  // 測試用 Canvas Host
  wizardRunner?: (...) => Promise<void>;             // 自定義 Setup Wizard
  startupStartedAt?: number;                         // 啟動時戳
};
```

`bind` 參數控制伺服器在哪個網路介面上監聽：

| bind 值 | 綁定位址 | 適用場景 |
|---------|----------|---------|
| `loopback` | `127.0.0.1` | 本機開發 |
| `lan` | `0.0.0.0` | Docker / LAN 部署 |
| `tailnet` | Tailscale IPv4 (100.64.0.0/10) | Tailscale 私有網路 |
| `auto` | 優先 loopback，否則 LAN | 自動偵測 |

> **引用來源**：`source-repo/src/gateway/server.impl.ts:151-204`

---

## 3. 啟動流程詳解

`startGatewayServer()` 的內部流程可分為 **10 個階段**：

### 階段 1：載入配置快照（Line 224-227）

```typescript
// source-repo/src/gateway/server.impl.ts:224-227
const configSnapshot = await loadGatewayStartupConfigSnapshot({
  minimalTestGateway,
  log,
});
```

呼叫 `loadGatewayStartupConfigSnapshot()` 從磁碟讀取 `openclaw.yml` 配置，建立不可變快照。

### 階段 2：準備認證引導（Line 247-252）

```typescript
// source-repo/src/gateway/server.impl.ts:247-252
const authBootstrap = await prepareGatewayStartupConfig({
  configSnapshot,
  authOverride: opts.auth,
  tailscaleOverride: opts.tailscale,
  activateRuntimeSecrets,
});
```

如果配置中沒有 token，此階段會**自動產生新的 token** 並嘗試寫入配置檔。

### 階段 3：解析執行時配置（Line ~318）

呼叫 `resolveGatewayRuntimeConfig()` 驗證並解析：
- 綁定位址約束（loopback vs LAN vs Tailscale）
- 認證需求（LAN 上不允許無認證）
- Control UI 來源策略（CORS）
- Tailscale 模式約束

### 階段 4：建立速率限制器（Line ~379）

```typescript
// source-repo/src/gateway/server.impl.ts:140-144
const browserRateLimiter = createAuthRateLimiter({
  ...rateLimitConfig,
  exemptLoopback: false,  // 瀏覽器不免除 loopback 限制
});
```

建立兩個獨立的速率限制器：
- **rateLimiter**：一般認證，loopback 免除
- **browserRateLimiter**：瀏覽器認證，不免除 loopback

### 階段 5：建立 Gateway 執行時狀態（Line ~438）

建立核心資料結構，包含 HTTP 伺服器、WS 伺服器、連線集合（clients Set）、前認證連線預算（preauth connection budget）。

### 階段 6：啟動早期執行時（Line ~548）

呼叫 `startGatewayEarlyRuntime()` 初始化：
- 健康監測（Health Monitor）
- 訊息去重（Dedupe）
- 維護計時器（Tick: 30s, Health: 60s）
- mDNS/Bonjour 探索
- 事件訂閱

### 階段 7：附加 WebSocket 處理器（Line ~710）

呼叫 `attachGatewayWsHandlers()` 將 WS 連線處理器綁定到 HTTP 伺服器的 upgrade 事件。

### 階段 8：啟動後附加執行時（Line ~737）

呼叫 `startGatewayPostAttachRuntime()` 執行：
- 更新可用性檢查
- Tailscale 配置（如啟用）
- 記錄就緒時戳

### 階段 9：載入 Plugin（Line ~737+）

呼叫 `reloadDeferredGatewayPlugins()` 載入延遲初始化的 Plugin（如 Channel Plugin）。

### 階段 10：啟動配置重載器（Line ~772）

呼叫 `startManagedGatewayConfigReloader()` 監視配置檔變更，支援 hot reload。

---

## 4. BOOT.md 啟動腳本

Gateway 支援在啟動時執行一個特殊的 `BOOT.md` 檔案：

```typescript
// source-repo/src/gateway/boot.ts:139-204
export async function runBootOnce(params: {
  cfg: OpenClawConfig;
  storePath: string;
  workspace: string;
  sessionKey: string;
  log: SubsystemLogger;
}): Promise<BootRunResult>
```

**執行邏輯**：
1. 從 workspace 目錄讀取 `BOOT.md`（`loadBootFile()`, Line 57-75）
2. 如果檔案存在且非空，將內容作為 Agent Prompt 執行
3. 執行前快照 Session 狀態（`snapshotMainSessionMapping()`, Line 77-113）
4. 執行失敗時回復 Session 狀態（`restoreMainSessionMapping()`, Line 115-137）

```typescript
// source-repo/src/gateway/boot.ts:38-41
type BootRunResult =
  | { status: "skipped"; reason: "missing" | "empty" }
  | { status: "ran" }
  | { status: "failed"; reason: string };
```

> **引用來源**：`source-repo/src/gateway/boot.ts:38-41, 57-75, 139-204`

---

## 5. 全域常數

Gateway 定義了一組關鍵常數控制系統行為上限：

```typescript
// source-repo/src/gateway/server-constants.ts:1-28
export const MAX_PAYLOAD_BYTES = 25 * 1024 * 1024;           // 25 MB — WS 最大酬載
export const MAX_BUFFERED_BYTES = 50 * 1024 * 1024;          // 50 MB — 每連線發送緩衝
export const MAX_PREAUTH_PAYLOAD_BYTES = 64 * 1024;          // 64 KB — 認證前酬載上限

export const TICK_INTERVAL_MS = 30_000;                       // 30 秒 — keepalive 間隔
export const HEALTH_REFRESH_INTERVAL_MS = 60_000;             // 60 秒 — 健康刷新間隔
export const DEDUPE_TTL_MS = 5 * 60_000;                      // 5 分鐘 — 去重 TTL
export const DEDUPE_MAX = 1000;                                // 最大去重條目數
```

另有一個動態常數控制聊天歷史大小：

```typescript
// source-repo/src/gateway/server-constants.ts:7
const DEFAULT_MAX_CHAT_HISTORY_MESSAGES_BYTES = 6 * 1024 * 1024; // 6 MB
```

> **引用來源**：`source-repo/src/gateway/server-constants.ts:1-28`

---

## 6. 維護計時器

Gateway 在啟動後建立多個週期性維護任務：

| 任務 | 間隔 | 來源 |
|------|------|------|
| Tick（keepalive 廣播） | 30 秒 | `server-maintenance.ts` |
| Health Refresh（健康快照） | 60 秒 | `server-maintenance.ts` |
| Dedupe Cleanup（去重清理） | 5 分鐘 | `server-maintenance.ts` |
| Media Cleanup（可選） | 配置決定 | `server-maintenance.ts` |

> **引用來源**：`source-repo/src/gateway/server-maintenance.ts:17-50+`, `source-repo/src/gateway/server-constants.ts:24-27`

---

## 7. 伺服器關閉流程

`GatewayServer.close()` 觸發優雅關閉：

1. 廣播 `shutdown` 事件給所有 WS 客戶端
2. 停止配置重載器
3. 停止維護計時器
4. 關閉 Plugin 執行時
5. 關閉 MCP Loopback Server
6. 關閉 HTTP/WS 伺服器
7. 清理診斷心跳

呼叫 `runGatewayClosePrelude()` 和 `createGatewayCloseHandler()` 協調整個關閉流程。

> **引用來源**：`source-repo/src/gateway/server-close.ts`, `source-repo/src/gateway/server.impl.ts:45`

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| Lazy loading 入口 | `source-repo/src/gateway/server.ts` | 1-18 |
| startGatewayServer 簽名 | `source-repo/src/gateway/server.impl.ts` | 206-209 |
| GatewayServer 型別 | `source-repo/src/gateway/server.impl.ts` | 147-149 |
| GatewayServerOptions | `source-repo/src/gateway/server.impl.ts` | 151-204 |
| 配置快照載入 | `source-repo/src/gateway/server.impl.ts` | 224-227 |
| 認證引導 | `source-repo/src/gateway/server.impl.ts` | 247-252 |
| Browser rate limiter | `source-repo/src/gateway/server.impl.ts` | 140-144 |
| 全域常數 | `source-repo/src/gateway/server-constants.ts` | 1-28 |
| BOOT.md 執行 | `source-repo/src/gateway/boot.ts` | 38-41, 139-204 |
| 維護計時器 | `source-repo/src/gateway/server-maintenance.ts` | 17-50+ |
| 關閉處理 | `source-repo/src/gateway/server-close.ts` | — |
