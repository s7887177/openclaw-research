# ContextEngine 介面與三大核心操作

## 本章摘要

OpenClaw 的記憶系統建立在一個名為 **ContextEngine** 的可插拔介面之上。這個介面定義了記憶管理的三大核心操作：**ingest**（寫入）、**assemble**（組裝）、**compact**（壓縮），以及數個輔助生命週期方法。透過統一的介面合約（interface contract），OpenClaw 讓不同的記憶後端——從內建的 SQLite 到 LanceDB 向量資料庫再到 QMD 混合搜尋引擎——能夠無縫替換，而上層的 agent 運行時（runtime）完全不需要知道底層用的是哪個引擎。

本章從介面定義出發，逐一拆解每個操作的參數、回傳值與呼叫時機，並追蹤它們在實際運行時的調用路徑。

---

## 1. ContextEngine 介面全貌

ContextEngine 介面定義在 `src/context-engine/types.ts`，是一個 TypeScript interface，包含以下成員：

```typescript
// source-repo/src/context-engine/types.ts:150-281
export interface ContextEngine {
  readonly info: ContextEngineInfo;
  bootstrap?(params: {...}): Promise<BootstrapResult>;
  maintain?(params: {...}): Promise<ContextEngineMaintenanceResult>;
  ingest(params: {...}): Promise<IngestResult>;
  ingestBatch?(params: {...}): Promise<IngestBatchResult>;
  afterTurn?(params: {...}): Promise<void>;
  assemble(params: {...}): Promise<AssembleResult>;
  compact(params: {...}): Promise<CompactResult>;
  prepareSubagentSpawn?(params: {...}): Promise<SubagentSpawnPreparation | undefined>;
  onSubagentEnded?(params: {...}): Promise<void>;
  dispose?(): Promise<void>;
}
```

其中，只有三個方法是**必須實作**的：`ingest`、`assemble`、`compact`（加上 `info` 屬性）。其餘都是可選的進階功能。

### 1.1 info 屬性：引擎身份

```typescript
// source-repo/src/context-engine/types.ts:47-53
export type ContextEngineInfo = {
  id: string;
  name: string;
  version?: string;
  ownsCompaction?: boolean;
};
```

- **id**：引擎的唯一識別碼（例如 `"legacy"`、`"memory-core"`）
- **name**：人類可讀的名稱
- **version**：版本號
- **ownsCompaction**：**關鍵旗標**。當設為 `true` 時，表示引擎自己管理壓縮生命週期；設為 `false` 或省略時，運行時會將壓縮委派給內建的 Pi runtime 演算法（`source-repo/src/context-engine/delegate.ts:33-80`）

這個 `ownsCompaction` 旗標是理解壓縮流程分岔的關鍵——後面會詳細說明。

---

## 2. Ingest：寫入記憶

### 2.1 單筆寫入

```typescript
// source-repo/src/context-engine/types.ts:179-185
ingest(params: {
  sessionId: string;
  sessionKey?: string;
  message: AgentMessage;
  isHeartbeat?: boolean;
}): Promise<IngestResult>;
```

- **sessionId**：UUID 格式的 session 識別碼
- **sessionKey**：正規化的路由鍵（routing key），格式為 `agent:agentId:rest`
- **message**：一條 `AgentMessage`，是 Pi Agent Core 的訊息型別
- **isHeartbeat**：標記此訊息是否屬於心跳運行（heartbeat run），引擎可據此決定是否跳過

回傳值：

```typescript
// source-repo/src/context-engine/types.ts:28-31
export type IngestResult = {
  ingested: boolean;  // 是否成功寫入（false 表示重複或 no-op）
};
```

### 2.2 批次寫入

```typescript
// source-repo/src/context-engine/types.ts:190-196
ingestBatch?(params: {
  sessionId: string;
  sessionKey?: string;
  messages: AgentMessage[];
  isHeartbeat?: boolean;
}): Promise<IngestBatchResult>;
```

`ingestBatch` 是可選方法，允許引擎一次性寫入整個回合（turn）的所有訊息。回傳 `ingestedCount` 表示實際寫入的數量。

```typescript
// source-repo/src/context-engine/types.ts:33-36
export type IngestBatchResult = {
  ingestedCount: number;
};
```

### 2.3 寫入的呼叫時機

寫入操作在每個回合結束後觸發，入口函式是 `finalizeAttemptContextEngineTurn()`。其邏輯如下：

1. 回合完成後，提取本回合新增的訊息（`messagesSnapshot.slice(prePromptMessageCount)`）
2. 若引擎提供 `afterTurn()`，則優先呼叫它（後面詳述）
3. 否則，嘗試 `ingestBatch()`；若不可用，逐一呼叫 `ingest()`
4. 錯誤被記錄但**不會中斷**運行——ingest 失敗不應影響對話

### 2.4 afterTurn：回合後生命週期

```typescript
// source-repo/src/context-engine/types.ts:203-218
afterTurn?(params: {
  sessionId: string;
  sessionKey?: string;
  sessionFile: string;
  messages: AgentMessage[];
  prePromptMessageCount: number;
  autoCompactionSummary?: string;
  isHeartbeat?: boolean;
  tokenBudget?: number;
  runtimeContext?: ContextEngineRuntimeContext;
}): Promise<void>;
```

`afterTurn` 是一個更高層的生命週期鉤子。引擎可以在這裡：
- 持久化規範上下文（canonical context）
- 觸發背景壓縮決策
- 執行統計收集

如果引擎實作了 `afterTurn`，運行時會**優先呼叫它**而非 `ingest/ingestBatch`——引擎完全接管回合後的處理。

---

## 3. Assemble：組裝上下文

```typescript
// source-repo/src/context-engine/types.ts:224-238
assemble(params: {
  sessionId: string;
  sessionKey?: string;
  messages: AgentMessage[];
  tokenBudget?: number;
  availableTools?: Set<string>;
  citationsMode?: MemoryCitationsMode;
  model?: string;
  prompt?: string;
}): Promise<AssembleResult>;
```

### 3.1 參數詳解

| 參數 | 用途 |
|------|------|
| `messages` | 完整的訊息歷史（未經裁剪） |
| `tokenBudget` | token 預算上限，引擎需在此範圍內組裝上下文 |
| `availableTools` | 當前可用的工具名稱集合（`Set<string>`），引擎可據此調整提示引導 |
| `citationsMode` | 記憶引用模式（`MemoryCitationsMode`），控制是否在回覆中引用記憶來源 |
| `model` | 當前使用的模型識別碼（如 `"claude-opus-4"`、`"gpt-4o"`），讓引擎可以按模型調整格式 |
| `prompt` | 本回合的使用者輸入，對**檢索導向引擎**（如 QMD）特別有用 |

### 3.2 回傳值

```typescript
// source-repo/src/context-engine/types.ts:6-13
export type AssembleResult = {
  messages: AgentMessage[];
  estimatedTokens: number;
  systemPromptAddition?: string;
};
```

- **messages**：經過排序、裁剪、適配的訊息陣列，直接餵給模型
- **estimatedTokens**：估算的總 token 數
- **systemPromptAddition**：可選的系統提示附加內容。引擎可以透過這個欄位注入記憶相關的引導（例如「你有以下記憶可參考...」）

### 3.3 組裝的呼叫時機

組裝在每次模型呼叫前觸發，入口是 `assembleAttemptContextEngine()`。流程：

1. 收集完整訊息歷史
2. 計算 token 預算
3. 收集可用工具
4. 呼叫 `contextEngine.assemble()`
5. 將結果中的 `messages` 作為模型輸入
6. 將 `systemPromptAddition` 前置到系統提示

### 3.4 記憶系統提示的建構

對於非 legacy 引擎，`src/context-engine/delegate.ts` 提供了一個輔助函式：

```typescript
// source-repo/src/context-engine/delegate.ts:88-101
export function buildMemorySystemPromptAddition(params: {
  availableTools: Set<string>;
  citationsMode?: MemoryCitationsMode;
}): string | undefined {
  const lines = buildMemoryPromptSection({
    availableTools: params.availableTools,
    citationsMode: params.citationsMode,
  });
  if (lines.length === 0) return undefined;
  const normalized = normalizeStructuredPromptSection(lines.join("\n"));
  return normalized || undefined;
}
```

這讓第三方引擎可以直接使用 OpenClaw 內建的記憶提示格式，而不需要自己重寫。

---

## 4. Compact：壓縮記憶

```typescript
// source-repo/src/context-engine/types.ts:244-258
compact(params: {
  sessionId: string;
  sessionKey?: string;
  sessionFile: string;
  tokenBudget?: number;
  force?: boolean;
  currentTokenCount?: number;
  compactionTarget?: "budget" | "threshold";
  customInstructions?: string;
  runtimeContext?: ContextEngineRuntimeContext;
}): Promise<CompactResult>;
```

### 4.1 參數詳解

| 參數 | 用途 |
|------|------|
| `sessionFile` | 對話記錄檔路徑（JSONL 格式） |
| `tokenBudget` | 壓縮的 token 目標上限 |
| `force` | 強制壓縮，即使未達閾值 |
| `currentTokenCount` | 呼叫者的當前活躍 token 估算 |
| `compactionTarget` | 收斂策略：`"budget"` 以預算為目標，`"threshold"` 以閾值為目標 |
| `customInstructions` | 自訂壓縮指示（如「保留所有關於專案X的細節」） |
| `runtimeContext` | 運行時上下文，包含提示快取遙測（prompt cache telemetry）等 |

### 4.2 回傳值

```typescript
// source-repo/src/context-engine/types.ts:15-26
export type CompactResult = {
  ok: boolean;
  compacted: boolean;
  reason?: string;
  result?: {
    summary?: string;
    firstKeptEntryId?: string;
    tokensBefore: number;
    tokensAfter?: number;
    details?: unknown;
  };
};
```

- **ok**：操作是否成功
- **compacted**：是否實際執行了壓縮（可能因未達閾值而跳過）
- **summary**：壓縮摘要文字
- **firstKeptEntryId**：保留的第一筆訊息 ID，用於追蹤譜系（lineage）
- **tokensBefore / tokensAfter**：壓縮前後的 token 數，用於效果評估

### 4.3 壓縮的兩條路徑

壓縮流程根據 `info.ownsCompaction` 分為兩條路徑：

#### 路徑 A：引擎自有壓縮（ownsCompaction = true）

引擎完全實作自己的壓縮演算法。運行時僅負責：
- 在壓縮前後觸發 hooks
- 記錄壓縮檢查點（checkpoint）

#### 路徑 B：委派到運行時（ownsCompaction = false 或省略）

引擎不擁有壓縮，運行時呼叫 `delegateCompactionToRuntime()`：

```typescript
// source-repo/src/context-engine/delegate.ts:33-80
export async function delegateCompactionToRuntime(
  params: Parameters<ContextEngine["compact"]>[0],
): Promise<CompactResult> {
  const { compactEmbeddedPiSessionDirect } = await loadCompactRuntime();
  // ... 參數映射 ...
  const result = await compactEmbeddedPiSessionDirect({
    ...runtimeContext,
    sessionId: params.sessionId,
    sessionFile: params.sessionFile,
    tokenBudget: params.tokenBudget,
    force: params.force,
    customInstructions: params.customInstructions,
    workspaceDir: typeof runtimeContext.workspaceDir === "string"
      ? runtimeContext.workspaceDir : process.cwd(),
  });
  return { ok: result.ok, compacted: result.compacted, ... };
}
```

這裡使用**動態 import**（`import("../agents/pi-embedded-runner/compact.runtime.js")`）來載入壓縮模組，避免在 bundler 中產生靜態依賴邊。

### 4.4 壓縮的觸發時機

壓縮透過排隊機制（`enqueueCommandInLane()`）觸發，防止死鎖：

1. **自動觸發**：token 使用超過閾值時
2. **溢位觸發**：上下文溢位（overflow）時緊急壓縮
3. **手動觸發**：使用者透過 `/compact` 指令
4. **維護觸發**：定時維護排程

壓縮前會先執行**記憶沖刷**（memory flush），讓 agent 有機會將重要記憶寫入持久儲存（如 `memory/YYYY-MM-DD.md`），然後才進行上下文裁剪。

---

## 5. 輔助生命週期方法

### 5.1 Bootstrap：初始化

```typescript
// source-repo/src/context-engine/types.ts:157-161
bootstrap?(params: {
  sessionId: string;
  sessionKey?: string;
  sessionFile: string;
}): Promise<BootstrapResult>;
```

在 session 首次啟動時呼叫，用於：
- 初始化引擎的內部儲存
- 匯入歷史上下文

```typescript
// source-repo/src/context-engine/types.ts:38-45
export type BootstrapResult = {
  bootstrapped: boolean;
  importedMessages?: number;
  reason?: string;
};
```

### 5.2 Maintain：轉錄維護

```typescript
// source-repo/src/context-engine/types.ts:169-174
maintain?(params: {
  sessionId: string;
  sessionKey?: string;
  sessionFile: string;
  runtimeContext?: ContextEngineRuntimeContext;
}): Promise<ContextEngineMaintenanceResult>;
```

在 bootstrap、成功回合、或壓縮之後呼叫。引擎可以利用 `runtimeContext.rewriteTranscriptEntries()` 來安全地改寫轉錄條目：

```typescript
// source-repo/src/context-engine/types.ts:62-83
export type TranscriptRewriteRequest = {
  replacements: TranscriptRewriteReplacement[];
};

export type TranscriptRewriteResult = {
  changed: boolean;
  bytesFreed: number;
  rewrittenEntries: number;
  reason?: string;
};
```

這是一種**非破壞性**的轉錄修改機制——引擎決定「改什麼」，運行時決定「怎麼改」（透過分支和重新附加，branch-and-reappend）。

### 5.3 子代理生命週期

```typescript
// source-repo/src/context-engine/types.ts:266-270
prepareSubagentSpawn?(params: {
  parentSessionKey: string;
  childSessionKey: string;
  ttlMs?: number;
}): Promise<SubagentSpawnPreparation | undefined>;

// source-repo/src/context-engine/types.ts:275
onSubagentEnded?(params: {
  childSessionKey: string;
  reason: SubagentEndReason;
}): Promise<void>;
```

子代理結束原因（`SubagentEndReason`）包括：`"deleted"` | `"completed"` | `"swept"` | `"released"`（`source-repo/src/context-engine/types.ts:60`）。

---

## 6. 引擎註冊與解析機制

### 6.1 Registry 架構

引擎的註冊中心位於 `src/context-engine/registry.ts`：

```typescript
// source-repo/src/context-engine/registry.ts:10
export type ContextEngineFactory = () => ContextEngine | Promise<ContextEngine>;
```

引擎以工廠模式（factory pattern）註冊，支援非同步建立（例如需要資料庫連線的引擎）。

註冊狀態透過 `resolveGlobalSingleton()` 管理，確保即使在 bundled 的多重副本環境下，仍然共用同一個 Map：

```typescript
// source-repo/src/context-engine/registry.ts:321-326
const contextEngineRegistryState = resolveGlobalSingleton<ContextEngineRegistryState>(
  CONTEXT_ENGINE_REGISTRY_STATE,
  () => ({ engines: new Map() }),
);
```

### 6.2 兩種註冊路徑

**核心路徑**（受信任的內部註冊）：

```typescript
// source-repo/src/context-engine/registry.ts:345-368
export function registerContextEngineForOwner(
  id: string,
  factory: ContextEngineFactory,
  owner: string,
  opts?: RegisterContextEngineForOwnerOptions,
): ContextEngineRegistrationResult {
  // 核心引擎的 slot id 只允許 "core" owner 註冊
  if (id === defaultSlotIdForKey("contextEngine") && normalizedOwner !== CORE_CONTEXT_ENGINE_OWNER) {
    return { ok: false, existingOwner: CORE_CONTEXT_ENGINE_OWNER };
  }
  // 已存在且 owner 不同則拒絕
  if (existing && existing.owner !== normalizedOwner) {
    return { ok: false, existingOwner: existing.owner };
  }
  registry.set(id, { factory, owner: normalizedOwner });
  return { ok: true };
}
```

**公開 SDK 路徑**（第三方插件註冊）：

```typescript
// source-repo/src/context-engine/registry.ts:377-382
export function registerContextEngine(
  id: string,
  factory: ContextEngineFactory,
): ContextEngineRegistrationResult {
  return registerContextEngineForOwner(id, factory, PUBLIC_CONTEXT_ENGINE_OWNER);
}
```

公開路徑刻意限制：不能覆蓋核心 ID，也不能刷新已存在的註冊。

### 6.3 引擎解析

```typescript
// source-repo/src/context-engine/registry.ts:411-427
export async function resolveContextEngine(config?: OpenClawConfig): Promise<ContextEngine> {
  const slotValue = config?.plugins?.slots?.contextEngine;
  const engineId = typeof slotValue === "string" && slotValue.trim()
    ? slotValue.trim()
    : defaultSlotIdForKey("contextEngine");

  const entry = getContextEngineRegistryState().engines.get(engineId);
  if (!entry) {
    throw new Error(`Context engine "${engineId}" is not registered. ...`);
  }

  return wrapContextEngineWithSessionKeyCompat(await entry.factory());
}
```

解析順序：
1. 檢查 `config.plugins.slots.contextEngine` 是否有明確指定
2. 若無，使用預設 slot 值（`"legacy"`）
3. 從 registry 取得工廠函式並建立實例
4. 以 `wrapContextEngineWithSessionKeyCompat()` 包裝，確保向後相容

### 6.4 Legacy 相容性代理

`wrapContextEngineWithSessionKeyCompat()` 是一個 Proxy-based 包裝器（`source-repo/src/context-engine/registry.ts:250-299`），它處理一個微妙的向後相容問題：

新版的 ContextEngine 方法簽章加入了 `sessionKey` 和 `prompt` 參數，但舊版的第三方引擎可能不認識這些參數（用了 Zod 的 `.strict()` 驗證會拋出 `unrecognized_keys` 錯誤）。

Proxy 的策略：
1. 先帶著所有參數呼叫
2. 若拋出「unrecognized key」相關錯誤，學習哪些 key 被拒絕
3. 移除被拒絕的 key 後重試
4. 後續呼叫直接跳過已知被拒絕的 key

這種「嘗試-學習-重試」的模式讓 OpenClaw 能無痛升級介面，不破壞既有引擎。

---

## 7. Prompt Cache 遙測

ContextEngine 還承擔了提示快取（prompt cache）的觀測任務：

```typescript
// source-repo/src/context-engine/types.ts:110-128
export type ContextEnginePromptCacheInfo = {
  retention?: ContextEnginePromptCacheRetention;    // "none"|"short"|"long"|"in_memory"|"24h"
  lastCallUsage?: ContextEnginePromptCacheUsage;    // input/output/cacheRead/cacheWrite
  observation?: ContextEnginePromptCacheObservation; // broke, changes
  lastCacheTouchAt?: number;
  expiresAt?: number;
};
```

快取觀測記錄包含變更代碼（`source-repo/src/context-engine/types.ts:97-103`）：
- `"cacheRetention"` — 快取保留策略變更
- `"model"` — 模型切換
- `"streamStrategy"` — 串流策略變更
- `"systemPrompt"` — 系統提示變更
- `"tools"` — 工具集變更
- `"transport"` — 傳輸方式變更

這些資訊透過 `runtimeContext.promptCache` 傳遞給引擎，讓快取感知引擎（cache-aware engines）能據此調整上下文組裝策略。

---

## 8. 完整記憶生命週期流程

將所有機制串在一起，一個完整的記憶生命週期如下：

```
使用者發送訊息
    │
    ▼
┌─────────────────────────────┐
│  1. assemble()              │  ← 檢索相關記憶，組裝上下文
│     - 語義搜尋（QMD/向量）   │
│     - Token 預算裁剪         │
│     - 工具感知引導           │
│     - systemPromptAddition   │
└─────────────────────────────┘
    │
    ▼
  [模型推理]
    │
    ▼
┌─────────────────────────────┐
│  2. ingest() / ingestBatch()│  ← 將新訊息寫入記憶
│     或 afterTurn()          │
│     - 非阻塞，錯誤不中斷    │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  3. maintain()              │  ← 轉錄維護（可選）
│     - 重寫條目              │
│     - 釋放空間              │
└─────────────────────────────┘
    │
    ▼ （Token 超過閾值時）
┌─────────────────────────────┐
│  4. compact()               │  ← 壓縮記憶
│     - 引擎自有 OR 委派運行時 │
│     - 排隊執行，防止死鎖     │
│     - 產出摘要 + 檢查點      │
└─────────────────────────────┘
    │
    ▼ （Session 結束時）
┌─────────────────────────────┐
│  5. dispose()               │  ← 釋放資源
│     - 關閉 DB 連線          │
│     - 清理暫存              │
└─────────────────────────────┘
```

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/context-engine/types.ts:1-282` | ContextEngine 介面完整定義 |
| `source-repo/src/context-engine/types.ts:6-13` | AssembleResult 型別 |
| `source-repo/src/context-engine/types.ts:15-26` | CompactResult 型別 |
| `source-repo/src/context-engine/types.ts:28-36` | IngestResult / IngestBatchResult 型別 |
| `source-repo/src/context-engine/types.ts:38-53` | BootstrapResult / ContextEngineInfo 型別 |
| `source-repo/src/context-engine/types.ts:55-83` | 子代理與轉錄重寫型別 |
| `source-repo/src/context-engine/types.ts:87-128` | Prompt Cache 相關型別 |
| `source-repo/src/context-engine/types.ts:130-142` | ContextEngineRuntimeContext 型別 |
| `source-repo/src/context-engine/types.ts:150-281` | ContextEngine interface 本體 |
| `source-repo/src/context-engine/delegate.ts:33-80` | delegateCompactionToRuntime() 壓縮委派 |
| `source-repo/src/context-engine/delegate.ts:88-101` | buildMemorySystemPromptAddition() |
| `source-repo/src/context-engine/registry.ts:10` | ContextEngineFactory 型別 |
| `source-repo/src/context-engine/registry.ts:250-299` | Legacy 相容性 Proxy |
| `source-repo/src/context-engine/registry.ts:305-326` | Registry 全域狀態 |
| `source-repo/src/context-engine/registry.ts:345-368` | registerContextEngineForOwner() |
| `source-repo/src/context-engine/registry.ts:377-382` | registerContextEngine() 公開 SDK 入口 |
| `source-repo/src/context-engine/registry.ts:411-427` | resolveContextEngine() 引擎解析 |
