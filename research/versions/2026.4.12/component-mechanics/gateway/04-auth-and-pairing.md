# Gateway 子章節 04：認證與配對

> **引用範圍**：`src/gateway/auth.ts`, `startup-auth.ts`, `connection-auth.ts`, `device-auth.ts`, `auth-rate-limit.ts`, `src/pairing/`

## 1. 認證模式

Gateway 支援 4 種認證模式（Authentication Mode），由 `ResolvedGatewayAuth` 型別描述：

```typescript
// source-repo/src/gateway/auth.ts:37-44
export type ResolvedGatewayAuth = {
  mode: "none" | "token" | "password" | "trusted-proxy";
  modeSource: "override" | "config" | "password" | "token" | "default";
  token?: string;
  password?: string;
  allowTailscale?: boolean;
  trustedProxy?: { header: string; allowedIps: string[] };
};
```

| 模式 | 說明 | 典型場景 |
|------|------|---------|
| `none` | 無認證 | 僅限 loopback 開發 |
| `token` | Bearer Token 認證 | 生產環境預設 |
| `password` | 密碼認證 | 簡易部署 |
| `trusted-proxy` | 信任反向代理標頭 | 企業部署 |

**模式來源（modeSource）**追蹤認證模式的決定來源，用於診斷：`override`（程式覆寫）、`config`（配置檔）、`default`（預設值）。

> **引用來源**：`source-repo/src/gateway/auth.ts:37-44`

---

## 2. 認證結果

每次認證嘗試都回傳 `GatewayAuthResult`：

```typescript
// source-repo/src/gateway/auth.ts:51-67
export type GatewayAuthResult = {
  ok: boolean;           // 認證是否成功
  method?: string;       // 使用的認證方法
  user?: string;         // 已認證的使用者
  reason?: string;       // 失敗原因
  rateLimited?: boolean; // 是否被速率限制
  retryAfterMs?: number; // 速率限制重試間隔
};
```

> **引用來源**：`source-repo/src/gateway/auth.ts:51-67`

---

## 3. 認證參數

授權 Gateway 連線需要的完整參數集：

```typescript
// source-repo/src/gateway/auth.ts:76-102 (摘要)
type AuthorizeGatewayConnectParams = {
  auth: ResolvedGatewayAuth;           // 認證配置
  connectAuth: { token?: string; password?: string };  // 客戶端提供的憑證
  req: http.IncomingMessage;           // HTTP 請求
  trustedProxies?: string[];           // 信任的代理 IP
  tailscaleWhois?: TailscaleWhois;     // Tailscale 身份
  authSurface: "http" | "ws-control-ui";  // 認證表面
  rateLimiter: AuthRateLimiter;        // 速率限制器
  clientIp?: string;                   // 客戶端 IP
  rateLimitScope?: string;             // 限速範圍
  allowRealIpFallback?: boolean;       // 允許 X-Real-IP 回退
  browserOriginPolicy?: BrowserOriginPolicy;  // 瀏覽器來源策略
};
```

> **引用來源**：`source-repo/src/gateway/auth.ts:76-102`

---

## 4. 啟動認證引導

Gateway 啟動時，`startup-auth.ts` 確保系統有有效的認證配置：

```typescript
// source-repo/src/gateway/startup-auth.ts:160-265 (摘要)
export async function ensureGatewayStartupAuth(params: {
  cfg: OpenClawConfig;
  authOverride?: GatewayAuthConfig;
  // ...
}): Promise<{
  resolvedAuth: ResolvedGatewayAuth;
  generatedToken?: string;
  persistedGeneratedToken?: boolean;
}>
```

**引導邏輯**：
1. 如果已有 token/password → 直接使用
2. 如果沒有 → **自動產生隨機 token**（Line 224）
3. 嘗試將產生的 token 寫入配置檔持久化（Line 240-244）
4. 驗證 token 不是已知的弱憑證

### 4.1 已知弱憑證

```typescript
// source-repo/src/gateway/startup-auth.ts:34-40
const KNOWN_WEAK_GATEWAY_TOKENS = new Set([
  "change-me-to-a-long-random-token",
]);
const KNOWN_WEAK_GATEWAY_PASSWORDS = new Set([
  "change-me-to-a-strong-password",
]);
```

`assertGatewayAuthNotKnownWeak()`（Line 267-290）會在啟動時檢查並警告使用者。

> **引用來源**：`source-repo/src/gateway/startup-auth.ts:34-40, 160-265, 267-290`

### 4.2 配置合併

```typescript
// source-repo/src/gateway/startup-auth.ts:42-69
export function mergeGatewayAuthConfig(
  base: GatewayAuthConfig | undefined,
  override: GatewayAuthConfig | undefined,
): GatewayAuthConfig | undefined
```

支援逐層合併 base 與 override 認證配置。

```typescript
// source-repo/src/gateway/startup-auth.ts:71-86
export function mergeGatewayTailscaleConfig(
  base: GatewayTailscaleConfig | undefined,
  override: GatewayTailscaleConfig | undefined,
): GatewayTailscaleConfig | undefined
```

> **引用來源**：`source-repo/src/gateway/startup-auth.ts:42-86`

### 4.3 Hooks Token 分離

```typescript
// source-repo/src/gateway/startup-auth.ts:292-314
export function assertHooksTokenSeparateFromGatewayAuth(params: {
  hooksToken?: string;
  gatewayToken?: string;
  gatewayPassword?: string;
}): void
```

確保 Hooks 端點使用的 token 與 Gateway 主 token 不同，防止憑證重用攻擊。

> **引用來源**：`source-repo/src/gateway/startup-auth.ts:292-314`

---

## 5. 連線認證解析

連線級認證由 `connection-auth.ts` 處理：

```typescript
// source-repo/src/gateway/connection-auth.ts:46-59
export async function resolveGatewayConnectionAuth(
  params: ConnectionAuthParams
): Promise<ResolvedGatewayAuth>

export function resolveGatewayConnectionAuthFromConfig(
  cfg: OpenClawConfig
): ResolvedGatewayAuth
```

提供非同步（從外部源取得）和同步（從配置取得）兩種解析路徑。

> **引用來源**：`source-repo/src/gateway/connection-auth.ts:46-59`

---

## 6. 設備認證（Device Auth）

行動裝置和遠端節點使用結構化的認證 payload：

### v2 格式

```typescript
// source-repo/src/gateway/device-auth.ts:20-34
export function buildDeviceAuthPayload(params: {
  deviceId: string;
  clientId: string;
  clientMode: string;
  role: string;
  scopes: string;
  signedAtMs: number;
  token: string;
  nonce: string;
}): string
// 格式: "v2|deviceId|clientId|clientMode|role|scopes|signedAtMs|token|nonce"
```

### v3 格式（增加平台資訊）

```typescript
// source-repo/src/gateway/device-auth.ts:36-54
export function buildDeviceAuthPayloadV3(params: {
  // ...v2 所有參數 +
  platform: string;
  deviceFamily: string;
}): string
// 格式: "v3|deviceId|clientId|clientMode|role|scopes|signedAtMs|token|nonce|platform|deviceFamily"
```

v3 增加了 `platform`（平台，如 "ios", "android"）和 `deviceFamily`（裝置族，如 "iphone", "pixel"）欄位。

> **引用來源**：`source-repo/src/gateway/device-auth.ts:20-54`

---

## 7. 認證速率限制

```typescript
// source-repo/src/gateway/auth-rate-limit.ts (概要)
export function createAuthRateLimiter(config: {
  maxAttempts: number;
  windowMs: number;
  exemptLoopback: boolean;
}): AuthRateLimiter
```

Gateway 建立**兩個**獨立的速率限制器：

| 限制器 | exemptLoopback | 用途 |
|--------|---------------|------|
| `rateLimiter` | `true` | 一般 WS/HTTP 認證 |
| `browserRateLimiter` | `false` | 瀏覽器 Control UI 認證 |

瀏覽器限制器不免除 loopback，因為瀏覽器可能被利用進行 CSRF 攻擊。

> **引用來源**：`source-repo/src/gateway/server.impl.ts:130-144`, `source-repo/src/gateway/auth-rate-limit.ts`

---

## 8. 配對系統（Pairing）

配對系統位於 `src/pairing/` 目錄，管理節點和裝置的信任建立：

| 檔案 | 用途 |
|------|------|
| `pairing-challenge.ts` | 產生配對挑戰碼 |
| `pairing-messages.ts` | 配對協議訊息 |
| `pairing-store.ts` | 配對狀態持久化 |
| `pairing-labels.ts` | 配對顯示標籤 |
| `setup-code.ts` | Setup Code 產生 |

**配對流程概要**：

```
1. 新裝置 → 發送 node.pair.request
2. Gateway → 產生配對挑戰 + Setup Code
3. 操作者 → 在 Control UI 看到配對請求
4. 操作者 → 核准 (node.pair.approve) 或拒絕 (node.pair.reject)
5. 核准後 → 裝置獲得 device token
6. 後續連線 → 使用 device-auth payload 認證
```

相關 WS 事件：
- `node.pair.requested` → 新配對請求推送
- `node.pair.resolved` → 配對結果推送
- `device.pair.requested` → 裝置配對請求推送
- `device.pair.resolved` → 裝置配對結果推送

> **引用來源**：`source-repo/src/pairing/` 目錄, `source-repo/src/gateway/server-methods-list.ts:93-103, 155-159`

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| ResolvedGatewayAuth 型別 | `source-repo/src/gateway/auth.ts` | 37-44 |
| GatewayAuthResult 型別 | `source-repo/src/gateway/auth.ts` | 51-67 |
| AuthorizeGatewayConnectParams | `source-repo/src/gateway/auth.ts` | 76-102 |
| ensureGatewayStartupAuth() | `source-repo/src/gateway/startup-auth.ts` | 160-265 |
| 已知弱憑證 | `source-repo/src/gateway/startup-auth.ts` | 34-40 |
| mergeGatewayAuthConfig() | `source-repo/src/gateway/startup-auth.ts` | 42-69 |
| assertHooksTokenSeparate | `source-repo/src/gateway/startup-auth.ts` | 292-314 |
| 連線認證解析 | `source-repo/src/gateway/connection-auth.ts` | 46-59 |
| Device Auth v2 payload | `source-repo/src/gateway/device-auth.ts` | 20-34 |
| Device Auth v3 payload | `source-repo/src/gateway/device-auth.ts` | 36-54 |
| 雙速率限制器 | `source-repo/src/gateway/server.impl.ts` | 130-144 |
| 配對系統 | `source-repo/src/pairing/` | — |
| 配對 WS 方法 | `source-repo/src/gateway/server-methods-list.ts` | 93-103 |
