# Control UI 架構：Web 元件、WebSocket Gateway 與裝置認證

> **本章摘要**：OpenClaw 的 Control UI 是一個以 Vite + Lit 建構的單頁網頁應用程式。它透過 WebSocket 與 Gateway 即時通訊，使用 Ed25519 裝置認證，支援 13 種語言。本章解析其技術架構、元件層級、Gateway 通訊協定和認證機制。

---

## 1. 技術堆疊

### 1.1 核心依賴

| 技術 | 版本 | 用途 |
|------|------|------|
| **Lit** | ^3.3.2 | Web Components 框架 |
| **Vite** | — | 建構工具 |
| **marked** | ^18.0.0 | Markdown 渲染 |
| **DOMPurify** | ^3.3.3 | HTML 淨化（安全） |
| **@noble/ed25519** | 3.0.1 | Ed25519 簽章（裝置認證） |
| **@create-markdown/preview** | ^2.0.3 | Markdown 預覽 |

> **來源**：`source-repo/ui/package.json` 的 `dependencies`

### 1.2 專案身份

```json
{
  "name": "openclaw-control-ui",
  "private": true,
  "type": "module"
}
```

Control UI 是私有套件（`private: true`），不對外發布——它是 OpenClaw 核心的內嵌組件。

> **來源**：`source-repo/ui/package.json:1-4`

---

## 2. 目錄結構

### 2.1 頂層結構

```
ui/
├── package.json
├── src/
│   ├── i18n/                    # 國際化
│   │   ├── index.ts
│   │   ├── locales/             # 13 種語言
│   │   │   ├── en.ts, de.ts, es.ts, fr.ts, id.ts
│   │   │   ├── ja-JP.ts, ko.ts, pl.ts, pt-BR.ts
│   │   │   ├── tr.ts, uk.ts, zh-CN.ts, zh-TW.ts
│   │   └── .i18n/               # 翻譯詞彙表和元資料
│   └── ui/
│       ├── app.ts               # 主應用 LitElement（812 行）
│       ├── gateway.ts           # WebSocket 客戶端（643 行）
│       ├── app-gateway.ts       # 應用層 Gateway 處理（555 行）
│       ├── app-render.ts        # 渲染邏輯（2,050 行）
│       ├── app-settings.ts      # 設定管理（707 行）
│       ├── app-lifecycle.ts     # 生命週期管理
│       ├── navigation.ts        # 路由導航（209 行）
│       ├── device-auth.ts       # 裝置認證
│       ├── session-key.ts       # Session Key 管理
│       ├── views/               # 頁面視圖元件
│       ├── controllers/         # 狀態控制器
│       └── components/          # 共用元件
```

> **來源**：`source-repo/ui/src/ui/` 目錄結構

### 2.2 視圖（views/）

views/ 目錄包含所有頁面級元件，覆蓋 OpenClaw 的完整功能面：

| 檔案 | 功能 |
|------|------|
| `chat.ts` | 主聊天介面 |
| `channels.ts` | 通道管理（Discord/Slack/Telegram/WhatsApp/Signal/iMessage/Nostr/GoogleChat） |
| `channels.discord.ts` ~ `channels.whatsapp.ts` | 各通道專屬配置 |
| `config-form.ts` | 通用配置表單 |
| `config.ts` | 系統配置頁 |
| `agents.ts` | 代理管理 |
| `sessions.ts` | Session 瀏覽 |
| `skills.ts` | 技能管理 |
| `cron.ts` | 排程任務 |
| `dreaming.ts` | 記憶整合（dreaming）視圖 |
| `overview.ts` | 總覽儀表板 |
| `usage.ts` | 使用量/成本追蹤 |
| `logs.ts` | 系統日誌 |
| `nodes.ts` | 節點（裝置）管理 |
| `debug.ts` | 除錯工具 |

> **來源**：`source-repo/ui/src/ui/views/` 目錄列表

### 2.3 控制器（controllers/）

controllers/ 實作狀態管理，每個功能區域一個控制器：

| 控制器 | 職責 |
|--------|------|
| `chat.ts` | 聊天訊息狀態 |
| `agents.ts` | 代理狀態 |
| `channels.ts` | 通道配置狀態 |
| `config.ts` | 系統配置狀態 |
| `cron.ts` | 排程狀態 |
| `dreaming.ts` | Dreaming 狀態 |
| `sessions.ts` | Session 狀態 |
| `skills.ts` | 技能狀態 |
| `usage.ts` | 使用量狀態 |
| `logs.ts` | 日誌狀態 |
| `models.ts` | 模型列表狀態 |
| `health.ts` | 健康檢查狀態 |
| `devices.ts` | 裝置狀態 |
| `presence.ts` | 在線狀態 |
| `exec-approval.ts` | 執行審批狀態 |
| `agent-identity.ts` | 代理身份狀態 |
| `assistant-identity.ts` | 助手身份狀態 |

> **來源**：`source-repo/ui/src/ui/controllers/` 目錄列表

---

## 3. 主應用元件 (app.ts)

### 3.1 結構

`app.ts`（812 行）是整個 Control UI 的根元件，繼承自 `LitElement`：

```typescript
@customElement("...")
export class OpenClawApp extends LitElement {
  @state() ...  // 所有應用狀態
}
```

### 3.2 模組化拆分

app.ts 本身不包含業務邏輯——它透過 import 委派到專門模組：

```typescript
// 通道操作
import { handleChannelConfigReload, handleChannelConfigSave, ... } from "./app-channels.ts";
// 聊天操作
import { handleAbortChat, handleSendChat, removeQueuedMessage } from "./app-chat.ts";
// Gateway 連線
import { connectGateway } from "./app-gateway.ts";
// 設定管理
import { applySettings, loadCron, setTab, setTheme } from "./app-settings.ts";
// 渲染
import { renderApp } from "./app-render.ts";
// 捲動控制
import { exportLogs, handleChatScroll, handleLogsScroll } from "./app-scroll.ts";
// 生命週期
import { handleConnected, handleDisconnected, handleFirstUpdated } from "./app-lifecycle.ts";
```

> **來源**：`source-repo/ui/src/ui/app.ts:1-60`

### 3.3 渲染邏輯

`app-render.ts`（2,050 行）是最大的 UI 檔案，負責將應用狀態渲染為 HTML。這反映了 Control UI 的「薄框架」哲學——Lit 提供 reactive rendering，但大部分邏輯在純 TypeScript 函數中。

---

## 4. Gateway WebSocket 通訊

### 4.1 概觀

Control UI 透過 WebSocket 與 OpenClaw Gateway 進行所有通訊。`gateway.ts`（643 行）實作了完整的 WebSocket 客戶端。

### 4.2 幀類型

Gateway 通訊使用兩種幀格式：

**事件幀（Gateway → UI）**：
```typescript
export type GatewayEventFrame = {
  type: "event";
  event: string;
  payload?: unknown;
  seq?: number;
  stateVersion?: { presence: number; health: number };
};
```

**回應幀（Gateway → UI）**：
```typescript
export type GatewayResponseFrame = {
  type: "res";
  id: string;       // 對應請求 ID
  ok: boolean;
  payload?: unknown;
  error?: {
    code: string;
    message: string;
    details?: unknown;
    retryable?: boolean;
    retryAfterMs?: number;
  };
};
```

> **來源**：`source-repo/ui/src/ui/gateway.ts:17-38`

### 4.3 連線流程

```
UI 開啟 → 建立 WebSocket → 發送 Hello（含認證）→ 收到 Hello OK（含快照）→ 持續雙向通訊
```

**app-gateway.ts** 處理應用層的 Gateway 邏輯：
- `connectGateway()`：建立連線
- `handleGatewayEvent()`：處理 Gateway 事件
- `applySnapshot()`：從 Hello OK 回應中套用初始狀態快照

> **來源**：`source-repo/ui/src/ui/app-gateway.ts:206, 321, 535`

### 4.4 錯誤處理

```typescript
export class GatewayRequestError extends Error {
  // 包含 code、message、details、retryable、retryAfterMs
}

export function isNonRecoverableAuthError(error: GatewayErrorInfo | undefined): boolean {
  // 判斷是否為不可恢復的認證錯誤
}
```

> **來源**：`source-repo/ui/src/ui/gateway.ts:47, 77`

---

## 5. 裝置認證（Device Auth）

### 5.1 Ed25519 簽章

Control UI 使用 Ed25519 數位簽章進行裝置認證：

```typescript
import { signDevicePayload } from "./device-identity.ts";
import { buildDeviceAuthPayload } from "../../../src/gateway/device-auth.js";
```

認證流程：
1. 首次開啟時，`loadOrCreateDeviceIdentity()` 產生 Ed25519 金鑰對
2. 連線時，`signDevicePayload()` 用私鑰簽署 payload
3. Gateway 用公鑰驗證身份

> **來源**：`source-repo/ui/src/ui/gateway.ts:14-15`

### 5.2 Token 儲存

裝置認證 token 儲存在 localStorage：

```typescript
const STORAGE_KEY = "openclaw.device.auth.v1";

// 資料結構
interface DeviceAuthStore {
  version: 1;
  deviceId: string;
  tokens: Record<string, DeviceAuthEntry>;
}
```

操作函數：
- `loadDeviceAuthToken()`：讀取 token
- `storeDeviceAuthToken()`：儲存 token
- `clearDeviceAuthToken()`：清除 token

> **來源**：`source-repo/ui/src/ui/device-auth.ts:10-50`

### 5.3 連線認證參數

```typescript
export type GatewayConnectAuth = {
  mode: string;
  // ...認證相關欄位
};

export type GatewayConnectDevice = {
  // 裝置資訊
};

export type GatewayConnectParams = {
  // 連線參數：包含 auth + device + client info
};
```

> **來源**：`source-repo/ui/src/ui/gateway.ts:154-188`

---

## 6. Session Key 管理

### 6.1 格式

Session key 遵循 `agent:{agentId}:{rest}` 的格式：

```typescript
export type ParsedAgentSessionKey = {
  agentId: string;  // 預設 "main"
  rest: string;
};

// 解析邏輯
export function parseAgentSessionKey(sessionKey: string): ParsedAgentSessionKey | null {
  const parts = raw.split(":").filter(Boolean);
  if (parts.length < 3 || parts[0] !== "agent") return null;
  return { agentId: parts[1], rest: parts.slice(2).join(":") };
}
```

預設值：
- `DEFAULT_AGENT_ID = "main"`
- `DEFAULT_MAIN_KEY = "main"`

> **來源**：`source-repo/ui/src/ui/session-key.ts:7-40`

---

## 7. 國際化（i18n）

### 7.1 支援的語言

Control UI 支援 13 種語言（TypeScript 模組格式）：

| 語言代碼 | 語言 |
|----------|------|
| `en` | 英語 |
| `de` | 德語 |
| `es` | 西班牙語 |
| `fr` | 法語 |
| `id` | 印尼語 |
| `ja-JP` | 日語 |
| `ko` | 韓語 |
| `pl` | 波蘭語 |
| `pt-BR` | 巴西葡萄牙語 |
| `tr` | 土耳其語 |
| `uk` | 烏克蘭語 |
| `zh-CN` | 簡體中文 |
| `zh-TW` | 繁體中文 |

> **來源**：`source-repo/ui/src/i18n/locales/` 目錄列表

### 7.2 翻譯基礎設施

- 每個語言一個 `.ts` 模組（非 JSON）
- `.i18n/` 目錄包含每種語言的 `meta.json` 和 `glossary.{lang}.json`
- 翻譯詞彙表確保術語一致性

---

## 8. 模型選擇狀態

`chat-model-select-state.ts` 管理 AI 模型的選擇邏輯：

```typescript
// 使用者可在 UI 中切換不同的 AI 模型
// 支援 model ref 格式：provider/model（如 "anthropic/claude-opus-4.5"）
```

`chat-model-ref.ts` 定義了模型引用的解析和格式化邏輯。

> **來源**：`source-repo/ui/src/ui/chat-model-select-state.ts`、`source-repo/ui/src/ui/chat-model-ref.ts`

---

## 9. 架構模式總結

### 9.1 設計原則

1. **Web Components**：使用 Lit 的 `@customElement` 和 reactive properties
2. **函數式分離**：app.ts 是薄殼，邏輯分散在 app-*.ts 模組中
3. **控制器模式**：每個功能區一個 controller，管理狀態和 Gateway 互動
4. **即時通訊**：所有資料透過 WebSocket 即時同步，無 REST API
5. **安全至上**：Ed25519 裝置認證、DOMPurify HTML 淨化

### 9.2 資料流

```
用戶操作 → LitElement event → app-*.ts handler → Gateway WebSocket request
     ↓
Gateway response/event → app-gateway.ts → controller update → LitElement re-render
```

---

## 引用來源

| 來源路徑 | 引用內容 |
|----------|----------|
| `source-repo/ui/package.json` | 依賴和套件資訊 |
| `source-repo/ui/src/ui/app.ts:1-60` | 主應用元件和模組導入 |
| `source-repo/ui/src/ui/gateway.ts:17-38` | 幀類型定義 |
| `source-repo/ui/src/ui/gateway.ts:47, 77` | 錯誤處理 |
| `source-repo/ui/src/ui/gateway.ts:14-15` | 裝置認證導入 |
| `source-repo/ui/src/ui/gateway.ts:154-188` | 連線參數類型 |
| `source-repo/ui/src/ui/app-gateway.ts:206, 321, 535` | 應用層 Gateway 函數 |
| `source-repo/ui/src/ui/device-auth.ts:10-50` | Token 儲存 |
| `source-repo/ui/src/ui/session-key.ts:7-40` | Session Key 格式 |
| `source-repo/ui/src/i18n/locales/` | 13 種語言模組 |
| `source-repo/ui/src/ui/views/` | 頁面視圖列表 |
| `source-repo/ui/src/ui/controllers/` | 控制器列表 |
