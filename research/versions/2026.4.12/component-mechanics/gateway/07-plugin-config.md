# Gateway 子章節 07：Plugin 系統與配置管理

> **引用範圍**：`src/gateway/server-plugin-bootstrap.ts`, `server-plugins.ts`, `server-startup-plugins.ts`, `config-reload.ts`, `server-runtime-config.ts`

## 1. Plugin Bootstrap

### 1.1 啟動前準備

```typescript
// source-repo/src/gateway/server-startup-plugins.ts:24-108 (概要)
export async function prepareGatewayPluginBootstrap(params: {
  cfgAtStart: OpenClawConfig;
  startupRuntimeConfig: RuntimeConfig;
  minimalTestGateway: boolean;
  log: SubsystemLogger;
}): Promise<{
  gatewayPluginConfigAtStart: unknown;
  defaultWorkspaceDir: string;
  deferredConfiguredChannelPluginIds: string[];
  startupPluginIds: string[];
  baseMethods: string[];
  pluginRegistry: PluginRegistry;
  baseGatewayMethods: Record<string, GatewayMethodHandler>;
}>
```

此函式在 Gateway 啟動早期執行，負責：
1. 執行 Channel Plugin 啟動維護
2. 執行啟動 Session 遷移
3. 初始化 Sub-agent Registry
4. 解析延遲/啟動 Plugin ID
5. 載入啟動 Plugin 或建立空 Registry

> **引用來源**：`source-repo/src/gateway/server-startup-plugins.ts:24-108`

### 1.2 Plugin 載入

```typescript
// source-repo/src/gateway/server-plugin-bootstrap.ts:62-91 (概要)
export async function prepareGatewayPluginLoad(params: {
  cfg: OpenClawConfig;
  pluginIds: string[];
  channelRegistry: ChannelRegistry;
  log: SubsystemLogger;
}): Promise<PluginLoadResult>
```

載入流程：
1. 套用 Plugin 自動啟用（`applyPluginAutoEnable()`）
2. 安裝 Gateway Plugin 執行時環境
3. 載入 Plugin（含錯誤處理）
4. 準備 Binding Registry
5. 記錄診斷資訊

### 1.3 延遲載入

```typescript
// source-repo/src/gateway/server-plugin-bootstrap.ts:102-112
export async function reloadDeferredGatewayPlugins(params: {
  cfg: OpenClawConfig;
  deferredPluginIds: string[];
  // ...
}): Promise<void>
```

某些 Plugin（如 Channel Plugin）需要非同步初始化，在 Gateway 主體啟動後再載入。

### 1.4 啟動時載入

```typescript
// source-repo/src/gateway/server-plugin-bootstrap.ts:93-100
export async function loadGatewayStartupPlugins(params: {
  cfg: OpenClawConfig;
  startupPluginIds: string[];
  channelRegistryPin: ChannelRegistryPin;
  // ...
}): Promise<PluginLoadResult>
```

> **引用來源**：`source-repo/src/gateway/server-plugin-bootstrap.ts:62-112`

---

## 2. Plugin 執行時管理

### 2.1 Fallback Context

非 WS 路徑（如 Channel Adapter 的 Telegram Webhook）無法取得 WS 連線上下文。`server-plugins.ts` 提供 fallback 機制：

```typescript
// source-repo/src/gateway/server-plugins.ts:53-58 (概要)
export function setFallbackGatewayContextResolver(
  resolver: () => GatewayContext
): void

export function getFallbackGatewayContext(): GatewayContext
```

Fallback context 儲存在全域 singleton 中，使用 Symbol key 避免命名衝突。

> **引用來源**：`source-repo/src/gateway/server-plugins.ts:26-64`

### 2.2 Sub-agent Runtime

Plugin 系統提供 Sub-agent 執行時，允許 Plugin 生成子 Agent：

```typescript
// source-repo/src/gateway/server-plugins.ts (引用)
export function createGatewaySubagentRuntime(params): SubagentRuntime
export function setPluginSubagentOverridePolicies(params): void
```

> **引用來源**：`source-repo/src/gateway/server-plugins.ts`

---

## 3. 動態配置重載

### 3.1 重載設定

```typescript
// source-repo/src/gateway/config-reload.ts:16-24
type GatewayReloadSettings = {
  mode: "off" | "restart" | "hot" | "hybrid";
  debounceMs: number;
};

const DEFAULT_RELOAD_SETTINGS: GatewayReloadSettings = {
  mode: "hybrid",
  debounceMs: 300,
};
```

| 重載模式 | 說明 |
|---------|------|
| `off` | 停用重載 |
| `restart` | 偵測到變更後重啟 Gateway |
| `hot` | 即時套用變更（不重啟） |
| `hybrid` | **預設**：嘗試 hot reload，必要時 restart |

### 3.2 遺失配置重試

```typescript
// source-repo/src/gateway/config-reload.ts:25-26
const MISSING_CONFIG_RETRY_DELAY_MS = 150;  // 150 ms 重試間隔
const MISSING_CONFIG_MAX_RETRIES = 2;       // 最多 2 次重試
```

當配置檔在 file watcher 觸發時暫時不存在（如編輯器原子寫入），系統會重試。

### 3.3 配置差異比較

```typescript
// source-repo/src/gateway/config-reload.ts:28-57
export function diffConfigPaths(
  prev: OpenClawConfig,
  next: OpenClawConfig,
): string[]
```

深度比較兩個配置物件，回傳變更的路徑列表（如 `["agents.defaults.model", "gateway.auth.token"]`）。

### 3.4 重載設定解析

```typescript
// source-repo/src/gateway/config-reload.ts:59-71
export function resolveGatewayReloadSettings(
  cfg: OpenClawConfig
): GatewayReloadSettings
```

從配置中讀取重載模式和防抖間隔。

### 3.5 啟動配置重載器

```typescript
// source-repo/src/gateway/config-reload.ts:77-100+
export function startGatewayConfigReloader(params: {
  configPath: string;
  currentCfg: () => OpenClawConfig;
  onReload: (newCfg: OpenClawConfig, changedPaths: string[]) => Promise<void>;
  onRestart: () => Promise<void>;
  settings: GatewayReloadSettings;
  log: SubsystemLogger;
}): { stop: () => void }
```

啟動 file watcher 監視配置檔案，變更時根據模式決定：
- **hot**: 呼叫 `onReload()` 回調
- **restart**: 呼叫 `onRestart()` 回調
- **hybrid**: 先嘗試 hot，失敗則 restart

> **引用來源**：`source-repo/src/gateway/config-reload.ts:16-100+`

---

## 4. 執行時配置解析

```typescript
// source-repo/src/gateway/server-runtime-config.ts:22-38 (概要)
type GatewayRuntimeConfig = {
  bindHost: string;                         // 解析後的綁定位址
  controlUiEnabled: boolean;                // Control UI 是否啟用
  openAiChatCompletionsEnabled: boolean;    // Chat Completions 端點
  openResponsesEnabled: boolean;            // OpenResponses 端點
  strictTransportSecurityHeader?: string;   // HSTS 標頭
  controlUiBasePath: string;                // Control UI 基本路徑
  controlUiRoot?: string;                   // 自訂 Control UI 根目錄
  resolvedAuth: ResolvedGatewayAuth;        // 解析後的認證配置
  authMode: string;                         // 認證模式列舉
  tailscaleConfig?: TailscaleConfig;        // Tailscale 設定
  tailscaleMode?: string;                   // Tailscale 模式
  hooksConfig?: HooksConfig;                // Webhook 設定
  canvasHostEnabled: boolean;               // Canvas Host 是否啟用
};
```

```typescript
// source-repo/src/gateway/server-runtime-config.ts:40-150+ (概要)
export function resolveGatewayRuntimeConfig(params: {
  cfg: OpenClawConfig;
  opts: GatewayServerOptions;
  authBootstrap: AuthBootstrapResult;
}): GatewayRuntimeConfig
```

驗證邏輯包含：
- 綁定約束：Tailscale 模式需 loopback 綁定
- 認證約束：LAN 綁定不允許無認證
- CORS 約束：Control UI 來源策略
- Tailscale 模式約束

> **引用來源**：`source-repo/src/gateway/server-runtime-config.ts:22-150+`

---

## 5. 排程服務（Cron）

```typescript
// source-repo/src/gateway/server-cron.ts (概要)
export function buildGatewayCronService(params: {
  cfg: OpenClawConfig;
  // ...
}): GatewayCronService
```

Gateway 內建排程引擎，支援定時執行 Agent 任務。相關 WS 方法：

| 方法 | 說明 |
|------|------|
| `cron.list` | 列出排程 |
| `cron.status` | 排程狀態 |
| `cron.add` | 新增排程 |
| `cron.update` | 更新排程 |
| `cron.remove` | 移除排程 |
| `cron.run` | 手動觸發 |
| `cron.runs` | 執行歷史 |

> **引用來源**：`source-repo/src/gateway/server-cron.ts`, `source-repo/src/gateway/server-methods-list.ts:115-121`

---

## 6. Discovery 服務

```typescript
// source-repo/src/gateway/server-discovery.ts (概要)
// source-repo/src/gateway/server-discovery-runtime.ts
```

Gateway 支援透過 mDNS/Bonjour 廣播自身，讓區域網路上的客戶端（如行動 App）自動發現 Gateway。

> **引用來源**：`source-repo/src/gateway/server-discovery.ts`, `source-repo/src/gateway/server-discovery-runtime.ts`

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| prepareGatewayPluginBootstrap | `source-repo/src/gateway/server-startup-plugins.ts` | 24-108 |
| prepareGatewayPluginLoad | `source-repo/src/gateway/server-plugin-bootstrap.ts` | 62-91 |
| reloadDeferredGatewayPlugins | `source-repo/src/gateway/server-plugin-bootstrap.ts` | 102-112 |
| loadGatewayStartupPlugins | `source-repo/src/gateway/server-plugin-bootstrap.ts` | 93-100 |
| Fallback context | `source-repo/src/gateway/server-plugins.ts` | 26-64 |
| GatewayReloadSettings | `source-repo/src/gateway/config-reload.ts` | 16-24 |
| 遺失配置重試常數 | `source-repo/src/gateway/config-reload.ts` | 25-26 |
| diffConfigPaths | `source-repo/src/gateway/config-reload.ts` | 28-57 |
| startGatewayConfigReloader | `source-repo/src/gateway/config-reload.ts` | 77-100+ |
| GatewayRuntimeConfig | `source-repo/src/gateway/server-runtime-config.ts` | 22-38 |
| resolveGatewayRuntimeConfig | `source-repo/src/gateway/server-runtime-config.ts` | 40-150+ |
| Cron WS 方法 | `source-repo/src/gateway/server-methods-list.ts` | 115-121 |
| Discovery | `source-repo/src/gateway/server-discovery.ts` | — |
