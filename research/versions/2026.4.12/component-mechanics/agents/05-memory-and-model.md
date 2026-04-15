# Agents 子章節 05：記憶與模型

> **引用範圍**：`src/agents/memory-search.ts`, `src/agents/model-selection.ts`

## 1. Active Memory 系統

### 1.1 ResolvedMemorySearchConfig

`memory-search.ts` 定義了完整的記憶體搜尋配置型別，涵蓋向量搜尋、全文搜尋、混合搜尋等所有面向。

```typescript
// source-repo/src/agents/memory-search.ts:16-93
export type ResolvedMemorySearchConfig = {
  enabled: boolean;
  sources: Array<"memory" | "sessions">;
  extraPaths: string[];
  multimodal: MemoryMultimodalSettings;
  provider: string;
  remote?: { baseUrl?; apiKey?; headers?; batch?; };
  experimental: { sessionMemory: boolean };
  fallback: string;
  model: string;
  outputDimensionality?: number;
  local: { modelPath?; modelCacheDir?; };
  store: {
    driver: "sqlite";
    path: string;
    fts: { tokenizer: "unicode61" | "trigram" };
    vector: { enabled: boolean; extensionPath?: string };
  };
  chunking: { tokens: number; overlap: number };
  sync: {
    onSessionStart: boolean;
    onSearch: boolean;
    watch: boolean;
    watchDebounceMs: number;
    intervalMinutes: number;
    sessions: { deltaBytes; deltaMessages; postCompactionForce };
  };
  query: {
    maxResults: number;
    minScore: number;
    hybrid: {
      enabled: boolean;
      vectorWeight: number;
      textWeight: number;
      candidateMultiplier: number;
      mmr: { enabled: boolean; lambda: number };
      temporalDecay: { enabled: boolean; halfLifeDays: number };
    };
  };
  cache: { enabled: boolean; maxEntries?: number };
};
```

> **引用來源**：`source-repo/src/agents/memory-search.ts:16-93`

### 1.2 預設常數

```typescript
// source-repo/src/agents/memory-search.ts:97-113
const DEFAULT_CHUNK_TOKENS = 400;
const DEFAULT_CHUNK_OVERLAP = 80;
const DEFAULT_WATCH_DEBOUNCE_MS = 1500;
const DEFAULT_SESSION_DELTA_BYTES = 100_000;      // 100KB
const DEFAULT_SESSION_DELTA_MESSAGES = 50;
const DEFAULT_MAX_RESULTS = 6;
const DEFAULT_MIN_SCORE = 0.35;
const DEFAULT_HYBRID_ENABLED = true;
const DEFAULT_HYBRID_VECTOR_WEIGHT = 0.7;
const DEFAULT_HYBRID_TEXT_WEIGHT = 0.3;
const DEFAULT_HYBRID_CANDIDATE_MULTIPLIER = 4;
const DEFAULT_MMR_ENABLED = false;
const DEFAULT_MMR_LAMBDA = 0.7;
const DEFAULT_TEMPORAL_DECAY_ENABLED = false;
const DEFAULT_TEMPORAL_DECAY_HALF_LIFE_DAYS = 30;
const DEFAULT_CACHE_ENABLED = true;
const DEFAULT_SOURCES = ["memory"];
```

> **引用來源**：`source-repo/src/agents/memory-search.ts:97-113`

### 1.3 儲存後端

記憶體使用 **SQLite** 作為儲存引擎：

```typescript
store: {
  driver: "sqlite",              // 唯一支援的驅動
  path: string,                  // 資料庫檔案路徑
  fts: {
    tokenizer: "unicode61" | "trigram"  // 全文搜尋分詞器
  },
  vector: {
    enabled: boolean,            // 向量搜尋開關
    extensionPath?: string       // sqlite-vec 擴充路徑
  }
}
```

SQLite 儲存同時支援：
- **FTS5 全文搜尋**：使用 `unicode61`（語言感知）或 `trigram`（N-gram）分詞
- **向量搜尋**：透過 `sqlite-vec` 擴充實現，儲存嵌入向量

### 1.4 混合搜尋策略

預設啟用**混合搜尋**（Hybrid Search），結合向量搜尋與全文搜尋：

```
最終分數 = vectorWeight × 向量相似度 + textWeight × 文字匹配度
         = 0.7 × vector + 0.3 × text
```

進階選項：
- **MMR（最大邊際相關性）**：預設關閉，用於去除重複結果
- **時間衰減**：預設關閉，半衰期 30 天，讓較新的記憶權重更高
- **候選倍數器**：4×，先選出 4 倍候選結果再做精排

### 1.5 分塊策略

```
記憶文件 → 分塊（400 tokens / 塊，80 tokens 重疊）→ 向量化 → 儲存
```

- `tokens=400`：每塊約 400 tokens（約 1,600 字元）
- `overlap=80`：相鄰塊有 80 tokens 的重疊，確保語義不斷裂

### 1.6 同步策略

```typescript
sync: {
  onSessionStart: boolean,       // Session 啟動時同步
  onSearch: boolean,             // 搜尋時同步
  watch: boolean,                // 檔案監視
  watchDebounceMs: 1500,         // 監視防抖
  intervalMinutes: number,       // 定期同步間隔
  sessions: {
    deltaBytes: 100_000,         // 變更量達 100KB 時觸發
    deltaMessages: 50,           // 變更量達 50 條訊息時觸發
    postCompactionForce: boolean // Compaction 後強制同步
  }
}
```

記憶同步有多個觸發點，確保新資訊能及時被索引。

### 1.7 記憶來源

```typescript
const DEFAULT_SOURCES: Array<"memory" | "sessions"> = ["memory"];
```

| 來源 | 說明 |
|------|------|
| `memory` | 明確寫入記憶體的內容（MEMORY.md 等） |
| `sessions` | 對話歷史中的內容（實驗性功能） |

`sessions` 來源需 `experimental.sessionMemory = true` 才啟用。

> **引用來源**：`source-repo/src/agents/memory-search.ts:113-130`

---

## 2. 模型選擇系統

### 2.1 概觀

`model-selection.ts`（883 行）是 OpenClaw 最複雜的模組之一，負責解析、正規化、驗證模型參考。

### 2.2 核心型別

```typescript
// source-repo/src/agents/model-selection.ts:43-48
export type ThinkLevel = "off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "adaptive";

export type ModelAliasIndex = {
  byAlias: Map<string, { alias: string; ref: ModelRef }>;
  byKey: Map<string, string[]>;
};
```

```typescript
// source-repo/src/agents/model-selection-normalize.ts（被 re-export）
export type ModelRef = {
  provider: string;    // 提供者（如 openai, anthropic）
  model: string;       // 模型名稱（如 gpt-4o, claude-sonnet-4）
};
```

> **引用來源**：`source-repo/src/agents/model-selection.ts:43-48`

### 2.3 正規化函式

```typescript
// source-repo/src/agents/model-selection.ts:66-77（re-export）
export {
  findNormalizedProviderKey,
  findNormalizedProviderValue,
  legacyModelKey,
  modelKey,
  normalizeModelRef,
  normalizeProviderId,
  normalizeProviderIdForAuth,
  parseModelRef,
};
```

`parseModelRef()` 將字串解析為 `ModelRef`：
- `"openai/gpt-4o"` → `{ provider: "openai", model: "gpt-4o" }`
- `"gpt-4o"` → `{ provider: defaultProvider, model: "gpt-4o" }`

### 2.4 持久化覆寫解析

```typescript
// source-repo/src/agents/model-selection.ts:79-97
export function resolvePersistedOverrideModelRef(params: {
  defaultProvider: string;
  overrideProvider?: string;
  overrideModel?: string;
}): ModelRef | null
```

此函式處理 Session 層級的模型覆寫。當使用者在對話中切換模型時，覆寫設定會持久化到 Session Store，後續的對話回合會從 Store 讀取。

> **引用來源**：`source-repo/src/agents/model-selection.ts:79-97`

### 2.5 模型解析優先順序

模型選擇遵循嚴格的優先順序：

```
1. Session 持久化覆寫（使用者在對話中手動切換）
2. Agent 個別配置 (agents.list[].model)
3. Agent 全局預設 (agents.defaults.model)
4. 系統預設 (DEFAULT_MODEL / DEFAULT_PROVIDER)
```

```typescript
// source-repo/src/agents/defaults.ts
export const DEFAULT_MODEL = ...;
export const DEFAULT_PROVIDER = ...;
```

### 2.6 Fallback 機制

每個模型配置位置都支援 fallback：

```typescript
// AgentModelConfig（可以是 string 或 object）
model?: string | { primary: string; fallbacks?: string[] }

// 解析流程：
resolveAgentModelPrimaryValue(config)     → 主模型
resolveAgentModelFallbackValues(config)   → fallback 列表
```

Fallback 在主模型不可用時自動切換到備選模型。

### 2.7 Sub-agent 模型解析

```typescript
// source-repo/src/agents/subagent-spawn-plan.ts
resolveSubagentModelAndThinkingPlan(cfg, {
  requesterAgentId,
  targetAgentId,
  modelOverride,
  thinkingOverride,
})
```

Sub-agent 的模型解析有額外的邏輯：
1. 呼叫者明確指定的模型（`params.model`）
2. 父 Agent 的 `subagents.model`
3. 全局 `agents.defaults.subagents.model`
4. 目標 Agent 自身的模型配置
5. 系統預設

### 2.8 模型別名

```typescript
// source-repo/src/config/types.agent-defaults.ts:178
models?: Record<string, AgentModelEntryConfig>;

// AgentModelEntryConfig
{
  alias?: string;
  params?: Record<string, unknown>;
  streaming?: boolean;    // 預設 true，Ollama 可能需設為 false
}
```

模型別名允許用短名稱引用完整的 `provider/model` 字串。`params` 可以傳遞 Provider 特定的 API 參數。

> **引用來源**：`source-repo/src/config/types.agent-defaults.ts:17-28, 178`

### 2.9 安全清理

```typescript
// source-repo/src/agents/model-selection.ts:50-63
function sanitizeModelWarningValue(value: string): string {
  const stripped = value ? stripAnsi(value) : "";
  let controlBoundary = -1;
  for (let index = 0; index < stripped.length; index += 1) {
    const code = stripped.charCodeAt(index);
    if (code <= 0x1f || code === 0x7f) {
      controlBoundary = index;
      break;
    }
  }
  // ...
}
```

模型名稱在日誌輸出前會經過安全清理：
1. 移除 ANSI 控制碼
2. 截斷控制字元之後的內容
3. 套用日誌清理（`sanitizeForLog`）

這防止惡意模型名稱（可能來自不受信任的配置）在終端機中注入控制序列。

> **引用來源**：`source-repo/src/agents/model-selection.ts:50-63`

---

## 3. 常數速查表

### 記憶體系統

| 常數 | 值 | 說明 |
|------|-----|------|
| `DEFAULT_CHUNK_TOKENS` | 400 | 分塊大小 |
| `DEFAULT_CHUNK_OVERLAP` | 80 | 重疊 tokens |
| `DEFAULT_MAX_RESULTS` | 6 | 搜尋結果上限 |
| `DEFAULT_MIN_SCORE` | 0.35 | 最低相關性分數 |
| `DEFAULT_HYBRID_VECTOR_WEIGHT` | 0.7 | 向量搜尋權重 |
| `DEFAULT_HYBRID_TEXT_WEIGHT` | 0.3 | 文字搜尋權重 |
| `DEFAULT_HYBRID_CANDIDATE_MULTIPLIER` | 4 | 候選倍數 |
| `DEFAULT_MMR_LAMBDA` | 0.7 | MMR 多樣性參數 |
| `DEFAULT_TEMPORAL_DECAY_HALF_LIFE_DAYS` | 30 | 時間衰減半衰期 |
| `DEFAULT_WATCH_DEBOUNCE_MS` | 1,500 | 監視防抖 |
| `DEFAULT_SESSION_DELTA_BYTES` | 100,000 | Session 同步閾值 |
| `DEFAULT_SESSION_DELTA_MESSAGES` | 50 | Session 同步閾值 |

### Bootstrap 系統

| 常數 | 值 | 說明 |
|------|-----|------|
| `bootstrapMaxChars` | 20,000 | 單檔最大字元 |
| `bootstrapTotalMaxChars` | 150,000 | 所有檔案總字元上限 |
| `dailyMemoryDays` | 2 | 啟動時載入天數 |
| `maxFileBytes` | 16,384 | 單檔最大位元組 |
| `maxFileChars` | 2,000 | 單檔最大字元 |
| `maxTotalChars` | 4,500 | 啟動上下文總字元上限 |

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| ResolvedMemorySearchConfig | `source-repo/src/agents/memory-search.ts` | 16-93 |
| 記憶體預設常數 | `source-repo/src/agents/memory-search.ts` | 97-113 |
| 記憶來源正規化 | `source-repo/src/agents/memory-search.ts` | 115-130 |
| ThinkLevel | `source-repo/src/agents/model-selection.ts` | 43 |
| ModelAliasIndex | `source-repo/src/agents/model-selection.ts` | 45-48 |
| parseModelRef re-export | `source-repo/src/agents/model-selection.ts` | 66-76 |
| resolvePersistedOverrideModelRef() | `source-repo/src/agents/model-selection.ts` | 79-97 |
| sanitizeModelWarningValue() | `source-repo/src/agents/model-selection.ts` | 50-63 |
| AgentModelEntryConfig | `source-repo/src/config/types.agent-defaults.ts` | 17-23 |
| Bootstrap 限制 | `source-repo/src/config/types.agent-defaults.ts` | 197-207 |
| Startup Context | `source-repo/src/config/types.agent-defaults.ts` | 53-66 |
