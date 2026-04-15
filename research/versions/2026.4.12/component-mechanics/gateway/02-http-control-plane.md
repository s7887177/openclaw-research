# Gateway 子章節 02：HTTP 控制平面

> **引用範圍**：`src/gateway/server-http.ts`, `http-common.ts`, `http-endpoint-helpers.ts`, `control-ui-routing.ts`, `control-plane-rate-limit.ts`

## 1. HTTP 伺服器建立

Gateway 的 HTTP 伺服器**直接使用 Node.js 原生 `http`/`https` 模組**，不依賴 Express 或 Fastify 等框架。核心建立函式為：

```typescript
// source-repo/src/gateway/server-http.ts:826+ (概要)
export function createGatewayHttpServer(params: {
  canvasHost: CanvasHostServer | null;
  clients: Set<GatewayClient>;
  resolvedAuth: ResolvedGatewayAuth;
  getResolvedAuth: () => ResolvedGatewayAuth;
  rateLimiter: AuthRateLimiter;
  browserRateLimiter: AuthRateLimiter;
  // ... 更多參數
}): http.Server
```

回傳標準 Node.js HTTP Server 物件，所有路由邏輯在 request handler 內以**順序 if/else 管線**實作。

> **引用來源**：`source-repo/src/gateway/server-http.ts:826-1111`

---

## 2. 請求處理管線

HTTP 請求按照以下**固定順序**通過處理階段（Stage Pipeline）：

```
請求進入
  │
  ├─ 1. 安全標頭（Security Headers）
  ├─ 2. Hooks 處理器（Webhooks）
  ├─ 3. /v1/models — 模型列表
  ├─ 4. /v1/embeddings — 嵌入向量
  ├─ 5. /tools/invoke — 工具調用
  ├─ 6. /sessions/:id/kill — Session 終止
  ├─ 7. /sessions/:id/history — Session 歷史
  ├─ 8. /v1/responses — OpenResponses API
  ├─ 9. /v1/chat/completions — OpenAI 相容端點
  ├─ 10. Canvas 認證 & A2UI 路徑
  ├─ 11. Plugin 路由（動態註冊）
  ├─ 12. Control UI 端點（SPA）
  └─ 13. Gateway 探測端點（/health, /healthz, /ready, /readyz）
```

> **引用來源**：`source-repo/src/gateway/server-http.ts:899-1095`

**設計重點**：Plugin 路由插入在 Control UI 之前，確保 Plugin 可以擴充 HTTP 端點而不被 SPA catch-all 攔截。

---

## 3. 主要 REST 端點

### 3.1 模型端點

| 路徑 | 方法 | 說明 |
|------|------|------|
| `GET /v1/models` | GET | 列出可用 LLM 模型 |
| `POST /v1/embeddings` | POST | 產生嵌入向量 |

這些端點由 `models-http.ts` 和 `embeddings-http.ts` 處理。

> **引用來源**：`source-repo/src/gateway/models-http.ts`, `source-repo/src/gateway/embeddings-http.ts`

### 3.2 工具調用端點

```
POST /tools/invoke
```

處理器：`handleToolsInvokeHttpRequest()`，允許透過 HTTP 直接調用 Agent 工具。

> **引用來源**：`source-repo/src/gateway/tools-invoke-http.ts:130-150+`

### 3.3 Session 管理端點

| 路徑 | 方法 | 說明 |
|------|------|------|
| `POST /sessions/:id/kill` | POST | 終止指定 Session |
| `GET /sessions/:id/history` | GET | 取得 Session 聊天歷史（SSE） |

> **引用來源**：`source-repo/src/gateway/session-kill-http.ts`, `source-repo/src/gateway/sessions-history-http.ts`

### 3.4 OpenAI 相容端點

| 路徑 | 方法 | 說明 | 預設 |
|------|------|------|------|
| `POST /v1/chat/completions` | POST | OpenAI Chat Completions 相容 API | **停用** |
| `POST /v1/responses` | POST | OpenResponses API | **停用** |

兩個端點預設停用，需在配置中明確啟用。

> **引用來源**：`source-repo/src/gateway/server.impl.ts:171-179`（GatewayServerOptions 中的說明）

### 3.5 健康檢查端點

| 路徑 | 說明 |
|------|------|
| `GET /health` | 健康狀態 |
| `GET /healthz` | Kubernetes-style 健康檢查 |
| `GET /ready` | 就緒狀態 |
| `GET /readyz` | Kubernetes-style 就緒檢查 |

Docker Compose 中的 healthcheck 使用 `/healthz`：

```yaml
# source-repo/docker-compose.yml:40-46
healthcheck:
  test: ["CMD", "node", "-e",
    "fetch('http://127.0.0.1:18789/healthz').then((r)=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
```

> **引用來源**：`source-repo/docker-compose.yml:40-46`

---

## 4. 安全標頭

所有 HTTP 回應都會套用基線安全標頭：

```typescript
// source-repo/src/gateway/http-common.ts:11-22
export function setDefaultSecurityHeaders(res: http.ServerResponse): void {
  // X-Content-Type-Options: nosniff
  // X-Frame-Options: DENY
  // Strict-Transport-Security: (如配置)
  // Content-Security-Policy: (如配置)
}
```

> **引用來源**：`source-repo/src/gateway/http-common.ts:11-22`

---

## 5. 共用 HTTP 工具函式

`http-common.ts` 提供一組標準化的回應工具：

| 函式 | 用途 | 行號 |
|------|------|------|
| `sendJson(res, statusCode, body)` | 發送 JSON 回應 | 24-28 |
| `sendText(res, statusCode, text)` | 發送純文字回應 | 30-34 |
| `sendMethodNotAllowed(res)` | 405 回應 | 36-39 |
| `sendUnauthorized(res, reason?)` | 401 回應 | 41-45 |
| `sendRateLimited(res, retryAfterMs)` | 429 回應（含 Retry-After） | 47-57 |
| `sendGatewayAuthFailure(res, result)` | 認證失敗路由 | 59-65 |
| `sendInvalidRequest(res, reason?)` | 400 回應 | 67-71 |
| `readJsonBodyOrError(req, res, maxBytes)` | 讀取並驗證 JSON body | 73-96 |
| `setSseHeaders(res)` | SSE 串流標頭 | 102-108 |
| `watchClientDisconnect(req, callback)` | 監視客戶端斷線 | 110-140 |

> **引用來源**：`source-repo/src/gateway/http-common.ts:24-140`

---

## 6. POST JSON 端點統一處理

`handleGatewayPostJsonEndpoint()` 提供 POST JSON 端點的標準化驗證管線：

```typescript
// source-repo/src/gateway/http-endpoint-helpers.ts:12-77
export async function handleGatewayPostJsonEndpoint(params: {
  req: http.IncomingMessage;
  res: http.ServerResponse;
  pathname: string;
  expectedPath: string;
  resolvedAuth: ResolvedGatewayAuth;
  operatorScopes?: string[];
  maxBodyBytes?: number;
}): Promise<{ body: unknown; requestAuth: GatewayRequestAuth } | null>
```

驗證流程：
1. 路徑比對（pathname vs expectedPath）
2. HTTP 方法驗證（僅允許 POST）
3. 認證驗證
4. Operator Scope 驗證
5. 讀取並驗證 JSON body（含大小限制）

> **引用來源**：`source-repo/src/gateway/http-endpoint-helpers.ts:12-77`

---

## 7. Control UI 路由

Control UI 是 Gateway 內建的瀏覽器管理介面，其路由邏輯由 `classifyControlUiRequest()` 負責：

```typescript
// source-repo/src/gateway/control-ui-routing.ts:11-51
export function classifyControlUiRequest(params: {
  pathname: string;
  basePath: string;
  // ...
}): ControlUiRequestClassification
```

分類結果包含：
- **SPA 頁面**：轉向 `index.html`（SPA catch-all）
- **靜態資源**：直接回傳檔案
- **重定向**：路徑正規化重定向
- **略過**：非 Control UI 請求

> **引用來源**：`source-repo/src/gateway/control-ui-routing.ts:11-51`

---

## 8. 控制平面速率限制

控制平面的**寫入操作**（`config.apply`, `config.patch`, `update.run`）有獨立的速率限制：

```typescript
// source-repo/src/gateway/control-plane-rate-limit.ts:3-7
export const CONTROL_PLANE_RATE_LIMIT_MAX_REQUESTS = 3;       // 每視窗最多 3 次
export const CONTROL_PLANE_RATE_LIMIT_WINDOW_MS = 60_000;     // 60 秒視窗
export const CONTROL_PLANE_BUCKET_MAX_STALE_MS = 5 * 60_000;  // 5 分鐘過期
export const CONTROL_PLANE_BUCKET_MAX_ENTRIES = 10_000;        // 最大桶數（DoS 保護）
```

三個核心函式：

| 函式 | 用途 | 行號 |
|------|------|------|
| `resolveControlPlaneRateLimitKey()` | 從裝置/IP 衍生限速 key | 24-35 |
| `consumeControlPlaneWriteBudget()` | 消耗寫入預算 | 37-94 |
| `pruneStaleControlPlaneBuckets()` | 清理過期桶 | 101-110 |

> **引用來源**：`source-repo/src/gateway/control-plane-rate-limit.ts:3-110`

另外，HTTP Hook 端點有獨立的認證失敗限制：

```typescript
// source-repo/src/gateway/server-http.ts:77-78
const HOOK_AUTH_FAILURE_LIMIT = 20;          // 20 次失敗
const HOOK_AUTH_FAILURE_WINDOW_MS = 60_000;  // 1 分鐘視窗
```

> **引用來源**：`source-repo/src/gateway/server-http.ts:77-78`

---

## 9. WebSocket Upgrade 處理

HTTP 伺服器同時負責 WebSocket upgrade：

```typescript
// source-repo/src/gateway/server-http.ts:1113-1235
export function attachGatewayUpgradeHandler(params: {
  httpServer: http.Server;
  wss: WebSocketServer;
  clients: Set<GatewayClient>;
  preauthConnectionBudget: PreauthConnectionBudget;
  resolvedAuth: ResolvedGatewayAuth;
  // ...
}): void
```

Upgrade 處理包含：
1. 前認證連線預算（preauth connection budget）檢查
2. Origin 驗證
3. 認證驗證
4. WebSocket 協議升級

> **引用來源**：`source-repo/src/gateway/server-http.ts:1113-1235`

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| HTTP 伺服器建立 | `source-repo/src/gateway/server-http.ts` | 826-1111 |
| 請求處理管線 | `source-repo/src/gateway/server-http.ts` | 899-1095 |
| 安全標頭 | `source-repo/src/gateway/http-common.ts` | 11-22 |
| 共用 HTTP 工具 | `source-repo/src/gateway/http-common.ts` | 24-140 |
| POST JSON 端點 | `source-repo/src/gateway/http-endpoint-helpers.ts` | 12-77 |
| Control UI 路由 | `source-repo/src/gateway/control-ui-routing.ts` | 11-51 |
| 控制平面速率限制常數 | `source-repo/src/gateway/control-plane-rate-limit.ts` | 3-7 |
| 速率限制函式 | `source-repo/src/gateway/control-plane-rate-limit.ts` | 24-110 |
| Hook 認證限制 | `source-repo/src/gateway/server-http.ts` | 77-78 |
| WebSocket Upgrade | `source-repo/src/gateway/server-http.ts` | 1113-1235 |
| Docker healthcheck | `source-repo/docker-compose.yml` | 40-46 |
