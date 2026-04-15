# Hooks 事件系統（Event-Driven Hook System）

> **引用範圍**：`src/hooks/` 全部 28+ 個檔案
> **字數目標**：8,000–12,000 字

OpenClaw 的 Hook 系統是一個**事件驅動的擴充架構**，讓內部模組與外部 plugin 能在系統生命週期的關鍵節點插入自訂邏輯。它橫跨整個 OpenClaw 核心——從閘道啟動、Agent 引導、到訊息收發與會話修補，每一個重要動作都會觸發對應的 hook 事件。本章將從公開 API 門面開始，逐層深入到事件型別系統、全域註冊表、優先序策略、動態載入器、plugin hook 探索、前言解析、以及訊息 hook 映射器。

---

## 1. 核心架構

### 1.1 公開 API 門面（hooks.ts，15 行）

`hooks.ts` 是整個 hook 系統的公開入口，它的角色極為單純——將內部實作重新匯出（re-export）為更簡潔的名稱：

```typescript
// source-repo/src/hooks/hooks.ts（完整）
export {
  registerInternalHook   as registerHook,
  unregisterInternalHook as unregisterHook,
  clearInternalHooks     as clearHooks,
  getRegisteredEventKeys as getRegisteredHookEventKeys,
  triggerInternalHook    as triggerHook,
  createInternalHookEvent as createHookEvent,
} from "./internal-hooks";
```

這個模式有兩個好處：

1. **命名簡化**：外部使用者只需 `import { registerHook } from "hooks"`，不必處理 `Internal` 前綴
2. **內部自由度**：內部實作可以自由重構，只要這個門面不變，外部介面就穩定

從這 15 行出發，所有的 hook 功能都由 `internal-hooks.ts` 提供。

### 1.2 架構全景

```
                   hooks.ts（門面）
                       │
              ┌────────┴────────┐
              │                 │
      internal-hooks.ts    internal-hook-types.ts
      (註冊表 + 觸發器)     (型別定義)
              │
    ┌─────────┼─────────┬────────────┐
    │         │         │            │
 loader.ts  policy.ts  plugin-    frontmatter.ts
 (載入器)   (策略)     hooks.ts   (前言解析)
                      (plugin 探索)
              │
   message-hook-mappers.ts
   (訊息映射器)
```

---

## 2. 事件型別系統

### 2.1 基礎型別（internal-hook-types.ts，18 行）

```typescript
// source-repo/src/hooks/internal-hook-types.ts（概要）

// 事件類型：五大類
type InternalHookEventType = "command" | "session" | "agent" | "gateway" | "message";

// 事件介面
interface InternalHookEvent {
  type: InternalHookEventType; // 事件大類
  action: string;              // 具體動作（如 "bootstrap", "received"）
  sessionKey: string;          // 對話金鑰
  context: Record<string, unknown>; // 自訂上下文
  timestamp: number;           // 時間戳
  messages: unknown[];         // 相關訊息
}

// 處理器簽名
type InternalHookHandler = (event: InternalHookEvent) => Promise<void> | void;
```

五大事件類型構成了 OpenClaw 的事件分類學：

| 類型 | 說明 | 典型場景 |
|------|------|---------|
| `command` | CLI 指令相關 | 指令執行前後 |
| `session` | 對話 session 相關 | session 建立、修補 |
| `agent` | Agent 生命週期 | Agent 引導（bootstrap）|
| `gateway` | 閘道相關 | 閘道啟動 |
| `message` | 訊息處理 | 收到/發送/轉錄/預處理 |

### 2.2 具體事件型別（internal-hooks.ts，第 23–177 行）

`internal-hooks.ts` 定義了七個具體事件型別，每個都對應一個明確的系統動作：

#### AgentBootstrapHookEvent

```typescript
// source-repo/src/hooks/internal-hooks.ts:23+（概要）
interface AgentBootstrapHookEvent extends InternalHookEvent {
  type: "agent";
  action: "bootstrap";
  context: {
    workspaceDir: string;      // 工作區目錄
    bootstrapFiles: string[];  // 引導檔案清單
    cfg: OpenClawConfig;       // 完整配置
    sessionKey: string;        // 對話金鑰
  };
}
```

**觸發時機**：Agent 初次啟動時，在載入工具與模型之前。plugin 可以在此注入額外的引導檔案或修改配置。

#### GatewayStartupHookEvent

```typescript
// source-repo/src/hooks/internal-hooks.ts（概要）
interface GatewayStartupHookEvent extends InternalHookEvent {
  type: "gateway";
  action: "startup";
  context: {
    cfg: OpenClawConfig;       // 配置
    deps: GatewayDeps;         // 閘道依賴
    workspaceDir: string;      // 工作區目錄
  };
}
```

**觸發時機**：閘道完成初始化並準備接受連線時。適合用來初始化外部服務連接或設定全域狀態。

#### MessageReceivedHookEvent

```typescript
// source-repo/src/hooks/internal-hooks.ts（概要）
interface MessageReceivedHookEvent extends InternalHookEvent {
  type: "message";
  action: "received";
  context: {
    from: string;              // 發送者
    content: string;           // 訊息內容
    timestamp: number;         // 時間戳
    channelId: string;         // 頻道 ID
    accountId: string;         // 帳號 ID
    conversationId: string;    // 對話 ID
    messageId: string;         // 訊息 ID
  };
}
```

**觸發時機**：收到外部訊息時（來自 WhatsApp、Discord 等頻道）。

#### MessageSentHookEvent

```typescript
// source-repo/src/hooks/internal-hooks.ts（概要）
interface MessageSentHookEvent extends InternalHookEvent {
  type: "message";
  action: "sent";
  context: {
    to: string;                // 接收者
    content: string;           // 訊息內容
    success: boolean;          // 是否發送成功
    error?: string;            // 失敗原因
    channelId: string;         // 頻道 ID
    isGroup: boolean;          // 是否為群組訊息
    groupId?: string;          // 群組 ID
  };
}
```

**觸發時機**：訊息發送完成後（無論成功或失敗）。

#### MessageTranscribedHookEvent

```typescript
// source-repo/src/hooks/internal-hooks.ts（概要）
interface MessageTranscribedHookEvent extends InternalHookEvent {
  type: "message";
  action: "transcribed";
  context: {
    transcript: string;        // 轉錄文字
    body: string;              // 原始訊息本體
    bodyForAgent: string;      // 給 Agent 的版本
    mediaPath: string;         // 媒體檔案路徑
    mediaType: string;         // 媒體類型
  };
}
```

**觸發時機**：語音訊息完成 STT（Speech-to-Text）轉錄後。

#### MessagePreprocessedHookEvent

```typescript
// source-repo/src/hooks/internal-hooks.ts（概要）
interface MessagePreprocessedHookEvent extends InternalHookEvent {
  type: "message";
  action: "preprocessed";
  context: {
    transcript: string;        // 處理後的文字
    isGroup: boolean;          // 是否為群組訊息
    groupId?: string;          // 群組 ID
  };
}
```

**觸發時機**：訊息預處理完成後（文字清理、指令偵測等步驟之後）。

#### SessionPatchHookEvent

```typescript
// source-repo/src/hooks/internal-hooks.ts（概要）
interface SessionPatchHookEvent extends InternalHookEvent {
  type: "session";
  action: "patch";
  context: {
    sessionEntry: SessionEntry; // 原始 session 資料
    patch: SessionPatch;        // 修補內容
    cfg: OpenClawConfig;        // 配置
  };
}
```

**觸發時機**：session 資料被修改時（例如更新標題、標籤、或設定）。

### 2.3 事件觀察：四個訊息事件的流水線

四個 `message` 類型的事件形成了一條處理流水線：

```
外部訊息 ──→ received ──→ transcribed ──→ preprocessed ──→ [Agent 處理] ──→ sent
                          (僅語音訊息)
```

Plugin 可以在流水線的任何節點插入邏輯：
- **received**：過濾垃圾訊息、記錄接收日誌
- **transcribed**：修改轉錄結果、標記語言
- **preprocessed**：注入額外上下文、分流路由
- **sent**：追蹤送達率、觸發後續動作

---

## 3. 全域註冊表（Global Registry）

### 3.1 Singleton 策略（internal-hooks.ts，第 188–306 行）

hook 系統的核心是一個**全域單例**（global singleton）的處理器註冊表：

```typescript
// source-repo/src/hooks/internal-hooks.ts:188+（概要）
const HANDLERS_KEY = Symbol.for("openclaw.internalHookHandlers");
const INTERNAL_HOOKS_ENABLED_KEY = Symbol.for("openclaw.internalHooksEnabled");

// 取得或建立全域 Map
function getHandlersMap(): Map<string, InternalHookHandler[]> {
  const g = globalThis as any;
  if (!g[HANDLERS_KEY]) {
    g[HANDLERS_KEY] = new Map();
  }
  return g[HANDLERS_KEY];
}
```

使用 `Symbol.for()` 而非普通 `Symbol()` 是關鍵設計。`Symbol.for("openclaw.internalHookHandlers")` 在整個 JavaScript 執行環境（runtime）中返回**同一個** Symbol，即使程式碼被打包器（bundler）分割成多個 chunk。這解決了一個現實問題：

> 當 OpenClaw 的不同模組被分別打包時，每個 chunk 都會有自己的模組作用域（module scope）。如果用普通的模組級變數，不同 chunk 會得到不同的 Map，導致 hook 註冊失效。

### 3.2 核心 API

#### registerInternalHook()

```typescript
// source-repo/src/hooks/internal-hooks.ts:220-225（概要）
function registerInternalHook(eventKey: string, handler: InternalHookHandler) {
  const handlers = getHandlersMap();
  if (!handlers.has(eventKey)) {
    handlers.set(eventKey, []);
  }
  handlers.get(eventKey)!.push(handler);
}
```

`eventKey` 支援兩種格式：
- `"message"` — 監聽所有 message 類型的事件
- `"message:received"` — 只監聽 message 類型且 action 為 received 的事件

#### unregisterInternalHook()

```typescript
// source-repo/src/hooks/internal-hooks.ts:233-248（概要）
function unregisterInternalHook(eventKey: string, handler: InternalHookHandler) {
  const handlers = getHandlersMap();
  const list = handlers.get(eventKey);
  if (!list) return;

  const idx = list.indexOf(handler);
  if (idx !== -1) {
    list.splice(idx, 1);
  }
  // 清除空陣列，保持 Map 整潔
  if (list.length === 0) {
    handlers.delete(eventKey);
  }
}
```

注意 `indexOf(handler)` 使用的是**參考相等**（reference equality），這表示解除註冊時必須傳入與註冊時相同的函式參考。

#### clearInternalHooks()

```typescript
// source-repo/src/hooks/internal-hooks.ts:253-255
function clearInternalHooks() {
  getHandlersMap().clear();
}
```

清除所有 handler。主要用於測試環境和 hook 重新載入。

#### triggerInternalHook()

```typescript
// source-repo/src/hooks/internal-hooks.ts:286-306（概要）
async function triggerInternalHook(event: InternalHookEvent) {
  // 1. 檢查是否啟用
  if (!isInternalHooksEnabled()) return;

  // 2. 檢查是否有監聽者
  if (!hasInternalHookListeners(event.type, event.action)) return;

  const handlers = getHandlersMap();

  // 3. 收集 type 級處理器
  const typeHandlers = handlers.get(event.type) ?? [];

  // 4. 收集 type:action 級處理器
  const actionHandlers = handlers.get(`${event.type}:${event.action}`) ?? [];

  // 5. 按註冊順序執行所有處理器
  for (const handler of [...typeHandlers, ...actionHandlers]) {
    try {
      await handler(event);
    } catch (err) {
      // 捕獲錯誤，記錄但不中斷
      console.error(formatErrorMessage(err));
    }
  }
}
```

觸發流程的三個關鍵設計決策：

1. **雙層匹配**：先找 type 級 handler（如 `"message"`），再找 type:action 級 handler（如 `"message:received"`），兩者都會被執行
2. **順序執行**：handler 按註冊順序依序執行，非並行。這確保了可預測的行為順序
3. **錯誤隔離**：單一 handler 的錯誤不會影響其他 handler 的執行，錯誤會被記錄但吞下（swallow）

#### createInternalHookEvent()

```typescript
// source-repo/src/hooks/internal-hooks.ts:316-330（概要）
function createInternalHookEvent(
  type: InternalHookEventType,
  action: string,
  context: Record<string, unknown>,
  sessionKey = ""
): InternalHookEvent {
  return {
    type,
    action,
    sessionKey,
    context,
    timestamp: Date.now(),
    messages: [],
  };
}
```

工廠函式自動填入 `timestamp` 和空的 `messages` 陣列。

### 3.3 型別守衛（Type Guards，第 364–457 行）

為了在 handler 中安全地存取具體事件型別的 context 欄位，系統提供了一組型別守衛函式：

```typescript
// source-repo/src/hooks/internal-hooks.ts:364-457（概要）
function isAgentBootstrapEvent(e: InternalHookEvent): e is AgentBootstrapHookEvent {
  return e.type === "agent" && e.action === "bootstrap";
}

function isGatewayStartupEvent(e: InternalHookEvent): e is GatewayStartupHookEvent {
  return e.type === "gateway" && e.action === "startup";
}

function isMessageReceivedEvent(e: InternalHookEvent): e is MessageReceivedHookEvent {
  return e.type === "message" && e.action === "received";
}

function isMessageSentEvent(e: InternalHookEvent): e is MessageSentHookEvent {
  return e.type === "message" && e.action === "sent";
}

function isMessageTranscribedEvent(e: InternalHookEvent): e is MessageTranscribedHookEvent {
  return e.type === "message" && e.action === "transcribed";
}

function isMessagePreprocessedEvent(e: InternalHookEvent): e is MessagePreprocessedHookEvent {
  return e.type === "message" && e.action === "preprocessed";
}

function isSessionPatchEvent(e: InternalHookEvent): e is SessionPatchHookEvent {
  return e.type === "session" && e.action === "patch";
}
```

使用範例：

```typescript
registerHook("message", async (event) => {
  if (isMessageReceivedEvent(event)) {
    // TypeScript 現在知道 event.context 有 from, content 等欄位
    console.log(`收到來自 ${event.context.from} 的訊息`);
  }
});
```

---

## 4. Hook 型別與來源

### 4.1 Hook 資料結構（types.ts，68 行）

```typescript
// source-repo/src/hooks/types.ts:35-43（概要）
interface Hook {
  name: string;            // hook 名稱（唯一識別）
  description: string;     // 說明
  source: HookSource;      // 來源
  pluginId?: string;       // 若來自 plugin，其 ID
  filePath: string;        // 檔案路徑
  baseDir: string;         // 基礎目錄
  handlerPath: string;     // 處理器路徑（可能不同於 filePath）
}
```

### 4.2 Hook 來源分類

```typescript
// source-repo/src/hooks/types.ts（概要）
type HookSource =
  | "openclaw-bundled"     // 隨 OpenClaw 打包的內建 hook
  | "openclaw-managed"     // OpenClaw 管理目錄中的 hook
  | "openclaw-workspace"   // 工作區（專案目錄）中的 hook
  | "openclaw-plugin";     // 來自 plugin 的 hook
```

四種來源對應了 hook 的四個可能位置：

| 來源 | 位置 | 信任等級 |
|------|------|---------|
| `openclaw-bundled` | OpenClaw 套件內 | 最高——隨版本發布，經過審查 |
| `openclaw-plugin` | Plugin 目錄 | 高——使用者主動安裝的 plugin |
| `openclaw-managed` | `~/.openclaw/hooks/` | 中——使用者手動放置的 hook |
| `openclaw-workspace` | 專案目錄 | 最低——需要明確啟用 |

### 4.3 HookEntry

```typescript
// source-repo/src/hooks/types.ts:48-53（概要）
interface HookEntry {
  hook: Hook;                          // hook 本體
  frontmatter: Record<string, unknown>; // YAML 前言
  metadata: OpenClawHookMetadata;       // 結構化元資料
  invocationPolicy: HookInvocationPolicy; // 呼叫策略
}
```

### 4.4 Hook 元資料

```typescript
// source-repo/src/hooks/types.ts:10-27（概要）
interface OpenClawHookMetadata {
  events: string[];           // 監聽的事件清單（如 ["message:received"]）
  export: string;             // 匯出的處理器名稱（預設 "default"）
  os: string[];               // 支援的作業系統（如 ["linux", "darwin"]）
  requires: {
    bins: string[];           // 必要的系統二進位檔
    anyBins: string[];        // 至少需要其中一個
    env: string[];            // 必要的環境變數
    config: string[];         // 必要的配置項
  };
  install: string[];          // 安裝指令（如 ["npm install"]）
}
```

`requires` 欄位用於在載入前進行**前置條件檢查**：如果某個 hook 需要 `ffmpeg` 但系統未安裝，載入器會跳過它而不是在執行時才失敗。

### 4.5 呼叫策略

```typescript
// source-repo/src/hooks/types.ts（概要）
interface HookInvocationPolicy {
  enabled: boolean;  // 是否啟用
}
```

目前呼叫策略只有 `enabled` 一個欄位，但結構上預留了未來擴充的空間（例如限流、延遲執行等）。

---

## 5. 優先序與覆蓋策略（Policy）

### 5.1 來源策略定義（policy.ts，144 行）

```typescript
// source-repo/src/hooks/policy.ts:26-55（概要）
const HOOK_SOURCE_POLICIES = {
  "openclaw-bundled": {
    precedence: 10,          // 最低優先序
    defaultEnabled: true,    // 預設啟用
    canBeOverriddenBy: ["openclaw-managed", "openclaw-plugin"],
  },
  "openclaw-plugin": {
    precedence: 20,
    defaultEnabled: true,
    canBeOverriddenBy: ["openclaw-managed"],
  },
  "openclaw-managed": {
    precedence: 30,
    defaultEnabled: true,
    canBeOverriddenBy: ["openclaw-managed", "openclaw-plugin", "openclaw-bundled"],
  },
  "openclaw-workspace": {
    precedence: 40,          // 最高優先序
    defaultEnabled: false,   // 預設不啟用（需明確 opt-in）
    canBeOverriddenBy: ["openclaw-workspace"],
  },
};
```

優先序數字越大越優先。這形成了一個有趣的層級：

```
workspace (40) > managed (30) > plugin (20) > bundled (10)
```

但有兩個重要的限制：

1. **workspace hook 預設不啟用**：因為工作區是最不受信任的來源（可能來自 git clone），必須明確在配置中啟用
2. **覆蓋是有方向性的**：bundled hook 可以被 managed 和 plugin 覆蓋，但 workspace hook 只能被其他 workspace hook 覆蓋

### 5.2 啟用狀態解析

```typescript
// source-repo/src/hooks/policy.ts:76-98（概要）
function resolveHookEnableState(entry: HookEntry, cfg: OpenClawConfig): boolean {
  // Plugin hook 永遠啟用（啟用邏輯由 plugin 系統管理）
  if (entry.hook.source === "openclaw-plugin") return true;

  // 檢查配置中是否明確禁用
  if (cfg.hooks?.disabled?.includes(entry.hook.name)) return false;

  // Workspace hook 需要明確啟用
  if (entry.hook.source === "openclaw-workspace") {
    return cfg.hooks?.enabled?.includes(entry.hook.name) ?? false;
  }

  // 其他來源使用預設值
  return entry.invocationPolicy.enabled;
}
```

解析的優先順序：plugin 強制啟用 → 配置明確禁用 → workspace 需 opt-in → 使用預設。

### 5.3 Hook 條目解析

```typescript
// source-repo/src/hooks/policy.ts:109-143（概要）
function resolveHookEntries(entries: HookEntry[]): HookEntry[] {
  // 1. 按優先序排列（高 → 低）
  const sorted = entries.sort(
    (a, b) => getPrecedence(b.hook.source) - getPrecedence(a.hook.source)
  );

  // 2. 合併同名 hook，保留最高優先序
  const merged = new Map<string, HookEntry>();
  for (const entry of sorted) {
    const key = entry.hook.name;
    if (!merged.has(key)) {
      merged.set(key, entry);
    } else {
      // 檢查是否允許覆蓋
      const existing = merged.get(key)!;
      if (canOverrideHook(entry.hook.source, existing.hook.source)) {
        merged.set(key, entry);
      }
    }
  }

  return [...merged.values()];
}
```

當多個同名 hook 來自不同來源時，`canOverrideHook()` 決定誰勝出。

---

## 6. 載入器（Loader）

### 6.1 主載入函式（loader.ts，~270 行）

```typescript
// source-repo/src/hooks/loader.ts:79+（概要）
async function loadInternalHooks(
  cfg: OpenClawConfig,
  workspaceDir: string,
  opts?: { skipBundled?: boolean }
) {
  // 1. 重置：解除所有已註冊的 hook
  resetLoadedHooks();

  // 2. 從目錄探索載入 hook
  const workspaceEntries = await loadWorkspaceHookEntries(workspaceDir);

  // 3. 載入舊版配置處理器
  const legacyEntries = getLegacyInternalHookHandlers(cfg);

  // 4. 合併所有條目
  const allEntries = [...workspaceEntries, ...legacyEntries];

  // 5. 逐一處理
  for (const entry of allEntries) {
    // 5a. 前置條件檢查
    if (!shouldIncludeHook(entry)) continue;

    // 5b. 開啟安全邊界檔案
    const boundary = await openBoundaryFile(entry.hook.filePath);

    // 5c. 動態匯入模組
    const importUrl = buildImportUrl(entry);
    const mod = await import(importUrl);

    // 5d. 解析匯出的處理器函式
    const handler = resolveFunctionModuleExport(mod, entry.metadata.export);

    // 5e. 為每個監聽事件註冊
    for (const eventKey of entry.metadata.events) {
      registerInternalHook(eventKey, handler);
    }

    // 5f. 追蹤註冊，以便後續卸載
    trackRegistration(entry, handler);
  }
}
```

### 6.2 模組匯入與快取破壞

```typescript
// source-repo/src/hooks/loader.ts（概要）
function buildImportUrl(entry: HookEntry): string {
  const base = pathToFileURL(entry.hook.handlerPath).href;

  // 打包的 hook 不需要快取破壞（內容不會變）
  if (entry.hook.source === "openclaw-bundled") {
    return base;
  }

  // 可變 hook 使用 mtime + size 作為快取破壞參數
  const stat = fs.statSync(entry.hook.filePath);
  return `${base}?mtime=${stat.mtimeMs}&size=${stat.size}`;
}
```

這個設計解決了 Node.js 的模組快取問題：當使用者修改了 hook 檔案後，如果不加查詢參數，`import()` 會返回快取的舊版本。透過附加 `mtime`（修改時間）和 `size`（檔案大小）作為查詢參數，JavaScript 引擎會將其視為不同的模組 URL，從而強制重新載入。

打包的（bundled）hook 不需要這個機制，因為它們隨版本發布，在程式執行期間不會改變。

### 6.3 安全警告

```typescript
// source-repo/src/hooks/loader.ts（概要）
function maybeWarnTrustedHookSource(entry: HookEntry) {
  if (
    entry.hook.source === "openclaw-workspace" ||
    entry.hook.source === "openclaw-managed"
  ) {
    console.warn(
      `⚠️  載入受信任的本地程式碼：${entry.hook.filePath}`
    );
  }
}
```

workspace 和 managed 的 hook 會顯示警告，提醒使用者這些是**本地程式碼**，將在 OpenClaw 的程序中執行，擁有完整的系統權限。

---

## 7. 插件 Hook 探索（plugin-hooks.ts，98 行）

```typescript
// source-repo/src/hooks/plugin-hooks.ts:21-98（概要）
async function resolvePluginHookDirs(): Promise<PluginHookDirEntry[]> {
  // 1. 載入 plugin manifest 登錄表
  const registry = await loadPluginManifestRegistry();

  const dirs: PluginHookDirEntry[] = [];

  for (const [pluginId, manifest] of registry) {
    // 2. 檢查 plugin 是否啟用
    if (!isPluginActivated(pluginId)) continue;

    // 3. 檢查 memory slot 決策（僅一個 memory plugin 可啟用）
    if (!checkMemorySlotDecision(pluginId)) continue;

    // 4. 解析每個 hook 路徑
    for (const hookPath of manifest.hooks ?? []) {
      const resolved = path.resolve(manifest.baseDir, hookPath);

      // 4a. 檢查路徑存在
      if (!fs.existsSync(resolved)) continue;

      // 4b. 安全檢查：路徑必須在 plugin 目錄內
      if (!isPathInsideWithRealpath(resolved, manifest.baseDir)) continue;

      dirs.push({ dir: resolved, pluginId });
    }
  }

  // 5. 去重（同一路徑只保留一次）
  return deduplicateByPath(dirs);
}
```

安全檢查 `isPathInsideWithRealpath()` 解析符號連結（symlink）後再驗證路徑，防止 plugin 透過 symlink 逃逸到 plugin 目錄外的檔案系統。

Memory slot 決策的概念：OpenClaw 只允許**一個** memory plugin 同時啟用（避免衝突），因此在探索 hook 時也需要尊重這個限制。

---

## 8. 前言解析（Frontmatter）

### 8.1 YAML 前言格式（frontmatter.ts，83 行）

hook 檔案可以在開頭包含 YAML 前言（frontmatter），定義元資料：

```typescript
// 範例 hook 檔案
// ---
// events:
//   - message:received
//   - message:sent
// export: handleMessage
// os: [linux, darwin]
// requires:
//   bins: [ffmpeg]
// install:
//   - npm install some-dep
// enabled: true
// hookKey: my-custom-hook
// ---
```

### 8.2 解析函式

```typescript
// source-repo/src/hooks/frontmatter.ts（概要）

// 解析 YAML 前言
function parseFrontmatter(content: string): Record<string, unknown> {
  // 找到 --- ... --- 區塊，解析 YAML
}

// 提取 OpenClaw 元資料
function resolveOpenClawMetadata(fm: Record<string, unknown>): OpenClawHookMetadata {
  return {
    events: fm.events ?? [],
    export: fm.export ?? "default",
    os: fm.os ?? [],
    requires: {
      bins: fm.requires?.bins ?? [],
      anyBins: fm.requires?.anyBins ?? [],
      env: fm.requires?.env ?? [],
      config: fm.requires?.config ?? [],
    },
    install: fm.install ?? [],
  };
}

// 解析呼叫策略
function resolveHookInvocationPolicy(fm: Record<string, unknown>): HookInvocationPolicy {
  return { enabled: fm.enabled ?? true };
}

// 解析 hook 金鑰
function resolveHookKey(fm: Record<string, unknown>, hookName: string): string {
  return fm.hookKey ?? hookName;
}
```

`export` 欄位允許 hook 檔案匯出多個函式，並指定哪一個作為 handler。預設是 `"default"`（即 `export default`）。

---

## 9. 訊息 Hook 映射器（message-hook-mappers.ts，397 行）

### 9.1 標準化上下文（Canonical Context）

為了讓不同頻道（WhatsApp、Discord、Telegram 等）的訊息格式統一，映射器定義了「標準化上下文」（canonical context）：

```typescript
// source-repo/src/hooks/message-hook-mappers.ts（概要）

// 入站訊息標準化上下文
interface CanonicalInboundMessageHookContext {
  from: string;
  content: string;
  channelId: string;
  isGroup: boolean;
  groupId?: string;
  accountId: string;
  conversationId: string;
  messageId: string;
  timestamp: number;
}

// 發送訊息標準化上下文
interface CanonicalSentMessageHookContext {
  to: string;
  content: string;
  success: boolean;
  error?: string;
  channelId: string;
  isGroup: boolean;
  groupId?: string;
}
```

### 9.2 映射流程

映射器將內部的 `FinalizedMsgContext`（由各頻道 adapter 產生）轉換為標準化上下文，再從標準化上下文產生兩個方向的 hook context：

```
FinalizedMsgContext     （頻道特定格式）
        │
        ▼
CanonicalContext        （標準化中間格式）
        │
    ┌───┴───┐
    ▼       ▼
Plugin    Internal      （分別給 plugin hook 和內部 hook）
Context   Context
```

這三層轉換的好處：

1. **頻道無關**：新頻道只需實作到 `FinalizedMsgContext → Canonical` 的映射
2. **版本穩定**：plugin 依賴的 context 格式不會因內部重構而改變
3. **職責清晰**：plugin context 可以包含額外的 plugin 專用欄位

### 9.3 特殊處理

映射器還處理了幾個特殊情境：

- **語音訊息的 transcript**：將 STT 結果併入 context
- **媒體路徑**：將暫存的媒體檔案路徑傳入 hook，讓 plugin 可以存取原始媒體
- **對話解析**：根據 sessionKey 解析出完整的對話資訊（頻道、帳號、群組等）

---

## 10. 引用來源

| 引用代碼 | 檔案路徑 | 行號範圍 | 說明 |
|----------|----------|----------|------|
| HK-API-1 | `source-repo/src/hooks/hooks.ts` | 1–15 | 公開 API 門面 |
| HK-TYP-1 | `source-repo/src/hooks/internal-hook-types.ts` | 1–18 | 基礎事件型別 |
| HK-INT-1 | `source-repo/src/hooks/internal-hooks.ts` | 23–177 | 具體事件型別定義 |
| HK-INT-2 | `source-repo/src/hooks/internal-hooks.ts` | 188–306 | 全域註冊表核心 API |
| HK-INT-3 | `source-repo/src/hooks/internal-hooks.ts` | 220–225 | `registerInternalHook()` |
| HK-INT-4 | `source-repo/src/hooks/internal-hooks.ts` | 233–248 | `unregisterInternalHook()` |
| HK-INT-5 | `source-repo/src/hooks/internal-hooks.ts` | 253–255 | `clearInternalHooks()` |
| HK-INT-6 | `source-repo/src/hooks/internal-hooks.ts` | 286–306 | `triggerInternalHook()` |
| HK-INT-7 | `source-repo/src/hooks/internal-hooks.ts` | 316–330 | `createInternalHookEvent()` |
| HK-INT-8 | `source-repo/src/hooks/internal-hooks.ts` | 364–457 | 型別守衛函式 |
| HK-TPE-1 | `source-repo/src/hooks/types.ts` | 10–27 | `OpenClawHookMetadata` |
| HK-TPE-2 | `source-repo/src/hooks/types.ts` | 35–43 | `Hook` 介面 |
| HK-TPE-3 | `source-repo/src/hooks/types.ts` | 48–53 | `HookEntry` 介面 |
| HK-POL-1 | `source-repo/src/hooks/policy.ts` | 26–55 | `HOOK_SOURCE_POLICIES` |
| HK-POL-2 | `source-repo/src/hooks/policy.ts` | 76–98 | `resolveHookEnableState()` |
| HK-POL-3 | `source-repo/src/hooks/policy.ts` | 109–143 | `resolveHookEntries()` |
| HK-LDR-1 | `source-repo/src/hooks/loader.ts` | 79+ | `loadInternalHooks()` |
| HK-PLG-1 | `source-repo/src/hooks/plugin-hooks.ts` | 21–98 | `resolvePluginHookDirs()` |
| HK-FMT-1 | `source-repo/src/hooks/frontmatter.ts` | 1–83 | 前言解析函式 |
| HK-MAP-1 | `source-repo/src/hooks/message-hook-mappers.ts` | 1–397 | 訊息 hook 映射器 |
