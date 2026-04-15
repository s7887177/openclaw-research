# 多記憶後端、QMD 格式與 Session 檔案管理

## 本章摘要

OpenClaw 的記憶系統不僅有統一的 ContextEngine 介面，還提供了多種可切換的記憶後端實作。本章深入探討三大記憶後端（memory-core SQLite、memory-lancedb 向量資料庫、QMD 混合搜尋引擎）、Active Memory 子代理機制、向量嵌入（embeddings）的角色、以及 session 檔案的持久化與歸檔管理。

---

## 1. 向量嵌入（Vector Embeddings）的角色

在記憶檢索中，**向量嵌入**是實現語義搜尋的核心技術。OpenClaw 透過 `/v1/embeddings` HTTP 端點提供 OpenAI 相容的嵌入 API。

### 1.1 嵌入端點

```typescript
// source-repo/src/gateway/embeddings-http.ts
// HTTP 端點: /v1/embeddings (OpenAI-compatible)
```

端點接受的請求格式：
- **model**：可以是明確的 `"provider/model"` 格式（如 `"openai/text-embedding-3-small"`），也可以是 `"openclaw"` 或 `"openclaw/<agentId>"`
- **input**：字串或字串陣列
- **encoding_format**：`float` 或 `base64`
- **dimensions**：可選，覆蓋模型預設維度

### 1.2 Provider 選擇邏輯

嵌入的 provider 解析（`resolveEmbeddingsTarget`，`source-repo/src/gateway/embeddings-http.ts:172`）：

1. 優先使用請求中明確指定的 provider
2. 其次使用設定中的 `memory.provider`（預設 `"openai"`）
3. 若設定允許自動選擇，則嘗試可用的 provider
4. Provider 不可用時回退到次選

### 1.3 嵌入在記憶系統中的位置

嵌入是記憶寫入和檢索之間的「翻譯層」：

```
寫入時：文字 → 嵌入模型 → 向量 → 儲存到向量 DB
檢索時：查詢 → 嵌入模型 → 向量 → 相似度搜尋 → 取回相關記憶
```

Memory Host SDK 提供了多個 provider 實作（`source-repo/packages/memory-host-sdk/src/host/`）：
- `embeddings-openai.ts` — OpenAI 嵌入
- `embeddings-voyage.ts` — Voyage AI 嵌入
- `embeddings-mistral.ts` — Mistral 嵌入
- `embeddings-lmstudio.ts` — LM Studio 本地嵌入
- `embeddings-bedrock.ts` — AWS Bedrock 嵌入

---

## 2. Memory-Core：內建 SQLite 記憶後端

**位置**：`source-repo/extensions/memory-core/`

Memory-core 是 OpenClaw 的預設記憶後端，使用檔案系統加 SQLite 作為儲存。

### 2.1 儲存結構

- 記憶儲存在 `memory/YYYY-MM-DD.md` 檔案中
- 使用日期分片，每天一個檔案
- 格式為 Markdown，人類可讀

### 2.2 核心功能

Memory-core 提供：
- **`memory_search`** 工具：讓 agent 搜尋記憶
- **`memory_get`** 工具：取得特定記憶條目
- **CLI 重建索引**：命令列工具重建搜尋索引
- **`buildPromptSection()`**：建構記憶相關的系統提示段落
- **`buildMemoryFlushPlan()`**：建構壓縮前的記憶沖刷計畫

### 2.3 記憶沖刷計畫（Memory Flush Plan）

壓縮前的記憶沖刷是 memory-core 的特色機制（`source-repo/extensions/memory-core/src/flush-plan.ts`）。

設定預設值：
- `softThresholdTokens`：4000 tokens
- `forceFlushTranscriptBytes`：2MB
- `relativePath`：`memory/{YYYY-MM-DD}.md`

沖刷時會注入安全提示：
- 「僅將持久記憶寫入 `memory/YYYY-MM-DD.md`」
- 「若檔案已存在，僅 APPEND（不覆寫）」
- 「`MEMORY.md`、`DREAMS.md`、`SOUL.md`、`TOOLS.md`、`AGENTS.md` 為唯讀」
- 「使用標準 `YYYY-MM-DD.md` 格式（不要加時間戳變體）」

```typescript
// source-repo/extensions/memory-core/src/flush-plan.ts:95-139
// buildMemoryFlushPlan() 將 YYYY-MM-DD 替換為實際日期
// 附加當前時間行
// 提供靜默回覆 token 選項
```

---

## 3. Memory-LanceDB：向量資料庫後端

**位置**：`source-repo/extensions/memory-lancedb/`

LanceDB 後端使用本地向量資料庫，支援高效的語義搜尋。

### 3.1 資料模型

```typescript
// source-repo/extensions/memory-lancedb/index.ts
// MemoryEntry 型別
MemoryEntry = {
  id: string;
  text: string;
  vector: number[];     // 嵌入向量
  importance: number;   // 重要性分數
  category: string;     // 分類標籤
  createdAt: number;    // 建立時間戳
}

// 搜尋結果
MemorySearchResult = {
  entry: MemoryEntry;
  score: number;        // 相似度分數
}
```

### 3.2 MemoryDB 類別

```typescript
// source-repo/extensions/memory-lancedb/index.ts:51-93
// MemoryDB 類別
// - 延遲初始化（lazy initialization）
// - 自動建立 / 連接 LanceDB table
// - 本地檔案系統儲存
```

LanceDB 的優勢在於純向量搜尋，適合需要高語義匹配精度的場景。但它不像 QMD 那樣結合了 BM25 關鍵字搜尋。

---

## 4. Memory-Wiki：Obsidian Vault 整合

**位置**：`source-repo/extensions/memory-wiki/`

Memory-wiki 將 Obsidian vault 作為記憶來源，讓 agent 能存取使用者的個人知識庫。這是一個較特殊的後端，定位為「外部知識整合」而非「對話記憶儲存」。

---

## 5. QMD（Query Markdown Document）格式

### 5.1 概述

QMD 是 OpenClaw 整合的本地優先搜尋側車（sidecar），結合三種搜尋技術：

1. **BM25**：傳統的關鍵字搜尋（精確匹配）
2. **向量搜尋**：語義相似度搜尋
3. **重排序（Reranking）+ 查詢擴展（Query Expansion）**：提升結果品質

**文件位置**：`source-repo/docs/concepts/memory-qmd.md`

QMD 的運行時環境是 Bun + node-llama-cpp，首次使用時會自動下載 GGUF 模型（約 2GB）。

### 5.2 搜尋模式

QMD 提供三種搜尋模式：

| 模式 | 說明 | 速度 | 品質 |
|------|------|------|------|
| `"search"` | BM25 + 向量混合（預設） | 快 | 好 |
| `"vsearch"` | 純向量搜尋 | 快 | 中 |
| `"query"` | 進階查詢擴展 + 重排序 | 慢 | 最高 |

### 5.3 集合（Collections）

QMD 管理多個搜尋集合（`source-repo/extensions/memory-core/src/memory/qmd-manager.ts`）：

1. **預設工作空間集合**：追蹤 `MEMORY.md` + `memory/` 目錄樹
   - 位置：`~/.openclaw/agents/<agentId>/qmd/`
2. **Session 集合**：清洗後的 User/Assistant 回合
   - 位置：`~/.openclaw/agents/<id>/qmd/sessions/`
   - 啟用條件：`memory.qmd.sessions.enabled: true`
3. **額外路徑集合**：使用者指定的額外索引目錄
   - 結果前綴：`qmd/<collection>/<path>`

### 5.4 設定範例

```json
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "searchMode": "search",
      "paths": [
        { "name": "docs", "path": "~/notes", "pattern": "**/*.md" }
      ],
      "sessions": { "enabled": true },
      "scope": {
        "default": "deny",
        "rules": [
          { "action": "allow", "match": { "chatType": "direct" } }
        ]
      }
    }
  }
}
```

### 5.5 查詢結果型別

```typescript
// source-repo/packages/memory-host-sdk/src/host/qmd-query-parser.ts:7-16
QmdQueryResult = {
  docid?: string;     // 文件 ID
  score?: number;     // 相關性分數
  collection?: string; // 所屬集合
  file?: string;      // 來源檔案
  snippet?: string;   // 摘要片段
  body?: string;      // 完整內容
  startLine?: number; // 起始行號
  endLine?: number;   // 結束行號
}
```

### 5.6 環境變數

```bash
QMD_EMBED_MODEL="hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf"
QMD_RERANK_MODEL="/path/to/reranker.gguf"
QMD_GENERATE_MODEL="/path/to/generator.gguf"
```

超時設定：`memory.qmd.limits.timeoutMs`（預設 4000ms）

若 QMD 不可用，系統回退到內建的 SQLite 引擎。

### 5.7 Memory Host SDK 的 QMD 支援

SDK 提供完整的 QMD 整合工具（`source-repo/src/memory-host-sdk/engine-qmd.ts:1-22`）：

- 查詢擴展函式
- Session 檔案管理
- QMD 查詢解析與結果型別
- QMD 範圍推導（根據 channel 和 chat type）
- QMD 行程生成（process spawning）工具

---

## 6. Active Memory 插件

### 6.1 概述

**位置**：`source-repo/extensions/active-memory/index.ts`
**插件 ID**：`"active-memory"`

Active Memory 是一個特殊的記憶機制：它在每次回合的提示（prompt）之前，啟動一個**專用子代理（sub-agent）**來搜尋和回憶相關記憶。結果會被注入到主 agent 的上下文中。

### 6.2 設定結構

```typescript
// source-repo/extensions/active-memory/index.ts:57-123
{
  enabled?: boolean;
  agents?: string[];                    // 限制適用的 agent ID 列表
  model?: string;                       // 子代理使用的模型
  modelFallback?: string;               // 回退模型
  thinking?: ActiveMemoryThinkingLevel; // 思考層級
  promptStyle?: "balanced" | "strict" | "contextual"
    | "recall-heavy" | "precision-heavy" | "preference-only";
  timeoutMs?: number;       // 預設 15000ms
  queryMode?: "message" | "recent" | "full"; // 預設 "recent"
  maxSummaryChars?: number; // 預設 220
  persistTranscripts?: boolean;
  transcriptDir?: string;   // 預設 "active-memory"
  qmd?: {
    searchMode?: "inherit" | "search" | "vsearch" | "query";
  };
}
```

### 6.3 子代理運作機制

Active Memory 的核心是一個 `beforePrompt` 回調：

```typescript
// 透過 api.registerBeforePrompt() 註冊
// 在每次回合的提示之前執行
```

運作流程：

1. **觸發**：主 agent 準備送出提示前，`beforePrompt` 回調被觸發
2. **建立子代理 Session**：Session ID 格式為 `"active-memory-{timestamp}-{uuid}"`（`source-repo/extensions/active-memory/index.ts:630+`）
3. **搜尋記憶**：子代理使用 QMD 或 SQLite 搜尋相關記憶
4. **結果注入**：搜尋結果被注入到主 agent 的上下文
5. **快取**：結果會被快取（TTL: 15000ms，最多 1000 條目）

### 6.4 記憶提取模式

Active Memory 使用特殊標記來識別記憶內容（`source-repo/extensions/active-memory/index.ts:48-55`）：

- `🧩 active memory:` — 一般記憶
- `🔎 active memory debug:` — 除錯記憶
- `🧠 memory search:` — 記憶搜尋結果

### 6.5 Prompt Style 策略

六種提示風格控制子代理的搜尋行為：

| 風格 | 描述 |
|------|------|
| `"balanced"` | 平衡模式，兼顧精確度和召回率 |
| `"strict"` | 嚴格模式，只回傳高度相關的記憶 |
| `"contextual"` | 上下文感知模式，考慮對話脈絡 |
| `"recall-heavy"` | 偏重召回率，盡量找到相關記憶 |
| `"precision-heavy"` | 偏重精確度，減少雜訊 |
| `"preference-only"` | 僅回傳使用者偏好相關的記憶 |

### 6.6 Session Toggle 機制

Active Memory 使用 session-specific 的切換檔案（`session-toggles.json`）來記錄每個 session 的啟用狀態，允許在 session 層級開關記憶回調。

---

## 7. Session 檔案管理

### 7.1 轉錄檔案系統

**位置**：`source-repo/src/gateway/session-transcript-files.fs.ts`

Session 的對話記錄以 JSONL（JSON Lines）格式儲存在磁碟上。管理這些檔案涉及幾個核心操作：

#### 解析候選路徑

```typescript
// source-repo/src/gateway/session-transcript-files.fs.ts:70-124
// resolveSessionTranscriptCandidates()
// 解析可能的轉錄檔路徑
// 檢查：sessionFile, storePath, agentId, legacy ~/.openclaw/sessions
```

系統會檢查多達 7 個以上的候選路徑，包括：
- 明確指定的 `sessionFile` 路徑
- 標準的 sessions 目錄下的轉錄路徑
- Legacy 路徑 `~/.openclaw/sessions/`

#### 歸檔操作

```typescript
// source-repo/src/gateway/session-transcript-files.fs.ts:126-131
// archiveFileOnDisk()
// 重新命名為：{path}.{reason}.{timestamp}
// reason: "reset" | "deleted"
```

```typescript
// source-repo/src/gateway/session-transcript-files.fs.ts:133-146
// archiveSessionTranscripts()
// 歸檔 session 的所有轉錄檔
// 可選 restrictToStoreDir 旗標
```

#### 清理過期歸檔

```typescript
// source-repo/src/gateway/session-transcript-files.fs.ts:232-269
// cleanupArchivedSessionTranscripts()
// 按年齡非同步清理舊歸檔
// 回傳：{ removed, scanned }
```

### 7.2 轉錄鍵解析

**位置**：`source-repo/src/gateway/session-transcript-key.ts`

```typescript
// source-repo/src/gateway/session-transcript-key.ts:15
// TRANSCRIPT_SESSION_KEY_CACHE - 反向查找快取（最多 256 條，FIFO 淘汰）
```

```typescript
// source-repo/src/gateway/session-transcript-key.ts:60-157
// resolveSessionKeyForTranscriptFile()
// 透過轉錄檔路徑反向查找 session key
// - 比對正規化路徑
// - 按 sessionId 分組
// - 回傳最新的（按 updatedAt 排序）
```

### 7.3 Session ID 格式

Session ID 使用標準 UUID 格式：

```
[0-9a-f]{8}-[0-9a-f]{4}-[1-8][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}
```

轉錄檔名模式：
- `{topicIndex}-topic-{sessionId}.jsonl`
- `{timestamp}_{sessionId}.jsonl`

### 7.4 Session 歸檔運行時

**位置**：`source-repo/src/gateway/session-archive.runtime.ts`

歸檔運行時處理 session 結束時的清理工作：

```typescript
// source-repo/src/gateway/session-archive.fs.ts:22-31
// classifySessionTranscriptCandidate() → "current" | "stale" | "custom"
```

```typescript
// source-repo/src/gateway/session-archive.fs.ts:33-55
// extractGeneratedTranscriptSessionId() - 從檔名提取 UUID
```

### 7.5 穩定的 Session 結束轉錄

```typescript
// source-repo/src/gateway/session-transcript-files.fs.ts:193-230
// resolveStableSessionEndTranscript()
// 決定 session 結束時應使用哪個轉錄檔
```

---

## 8. 記憶後端的切換機制

### 8.1 設定方式

記憶後端的選擇透過設定檔控制：

```json
{
  "memory": {
    "backend": "qmd",   // 或 "sqlite", "lancedb" 等
    "provider": "openai" // 嵌入 provider
  }
}
```

### 8.2 後端解析

後端解析透過 `resolveMemoryBackendConfig()` 函式進行，它會：
1. 讀取 `memory.backend` 設定
2. 載入對應的記憶引擎工廠
3. 初始化引擎實例

### 8.3 ContextEngine 作為切換點

如前章所述，`resolveContextEngine()` 透過 `plugins.slots.contextEngine` 設定來選擇引擎。每個記憶後端需要：

1. 實作 `ContextEngine` 介面（至少 `ingest`、`assemble`、`compact`）
2. 透過 `registerContextEngine()` 或 `registerContextEngineForOwner()` 註冊工廠
3. 設定使用者在 `plugins.slots.contextEngine` 中指定引擎 ID

### 8.4 Memory Host SDK

**位置**：`source-repo/packages/memory-host-sdk/` 和 `source-repo/src/memory-host-sdk/`

SDK 提供記憶後端開發者所需的基礎建設：

核心匯出（`source-repo/src/memory-host-sdk/engine.ts:1-8`）：
- `engine-foundation.js` — 核心合約
- `engine-storage.js` — 儲存後端抽象
- `engine-embeddings.js` — 嵌入整合
- `engine-qmd.js` — QMD 支援

Host 子目錄提供具體的 provider 實作：
- `embeddings-*.ts` — 各家嵌入 provider
- `qmd-*.ts` — QMD 整合工具
- `session-files.ts` — Session 輔助函式
- `batch-runner.ts` — 批次執行器
- `remote-http.ts` — 遠端 HTTP 後端

---

## 9. 記憶壓縮檢查點

### 9.1 壓縮檢查點的概念

壓縮不是一次性的操作——OpenClaw 會在每次壓縮時建立**檢查點（checkpoint）**，記錄壓縮前後的狀態。

```typescript
// source-repo/src/gateway/session-compaction-checkpoints.ts:17
// MAX_COMPACTION_CHECKPOINTS_PER_SESSION = 25（最多保留 25 個檢查點）
```

### 9.2 檢查點原因分類

```typescript
// source-repo/src/gateway/session-compaction-checkpoints.ts:40-54
// resolveSessionCompactionCheckpointReason() 映射：
// trigger="manual"    → "manual"
// timedOut=true       → "timeout-retry"
// trigger="overflow"  → "overflow-retry"
// default             → "auto-threshold"
```

### 9.3 快照擷取

```typescript
// source-repo/src/gateway/session-compaction-checkpoints.ts:56-105
// captureCompactionCheckpointSnapshot()
// - 複製轉錄檔（檔名包含 UUID）
// - 擷取 SessionManager.getLeafId()
// - 回傳含 sessionId, sessionFile, leafId 的快照
```

### 9.4 檢查點持久化

```typescript
// source-repo/src/gateway/session-compaction-checkpoints.ts:120-188
// persistSessionCompactionCheckpoint()
// - 建立 SessionCompactionCheckpoint
// - 儲存 checkpointId, reason, token 數, 摘要
// - 裁剪至最後 25 個檢查點
```

---

## 10. 跨元件的記憶互動流程

將所有元件串在一起，記憶系統的完整跨元件互動如下：

```
使用者訊息 → Gateway
    │
    ├── Routing (src/routing/) → 決定 agentId + sessionKey
    │
    ├── Session 建立/恢復 (src/sessions/)
    │   └── 載入 session transcript (JSONL)
    │
    ├── [Active Memory beforePrompt]
    │   └── 子代理搜尋 (extensions/active-memory/)
    │       ├── QMD 語義搜尋 (packages/memory-host-sdk/)
    │       └── 結果注入主 agent 上下文
    │
    ├── ContextEngine.assemble()
    │   ├── 語義搜尋（QMD 或向量 DB）
    │   ├── Token 預算裁剪
    │   └── 回傳 messages + systemPromptAddition
    │
    ├── [LLM 推理] → 使用 /v1/embeddings (src/gateway/embeddings-http.ts)
    │
    ├── ContextEngine.ingest() / ingestBatch()
    │   └── 寫入 SQLite / LanceDB / QMD 集合
    │
    ├── ContextEngine.maintain() [可選]
    │   └── 轉錄維護 / 重寫
    │
    ├── [若 token 超過閾值]
    │   ├── Memory Flush Plan (extensions/memory-core/src/flush-plan.ts)
    │   │   └── 寫入 memory/YYYY-MM-DD.md
    │   └── ContextEngine.compact()
    │       ├── 自有壓縮 OR 委派到 Pi runtime
    │       └── 建立壓縮檢查點
    │
    └── [Session 結束]
        ├── ContextEngine.dispose()
        ├── archiveSessionTranscripts()
        └── cleanupArchivedSessionTranscripts()
```

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/gateway/embeddings-http.ts` | 向量嵌入 HTTP 端點 |
| `source-repo/extensions/memory-core/index.ts` | Memory-Core SQLite 後端 |
| `source-repo/extensions/memory-core/src/flush-plan.ts` | 記憶沖刷計畫 |
| `source-repo/extensions/memory-lancedb/index.ts:51-93` | LanceDB 向量記憶後端 |
| `source-repo/extensions/memory-wiki/` | Obsidian vault 整合 |
| `source-repo/extensions/active-memory/index.ts:48-55` | Active Memory 記憶提取模式 |
| `source-repo/extensions/active-memory/index.ts:57-123` | Active Memory 設定結構 |
| `source-repo/extensions/active-memory/index.ts:630+` | 子代理 Session ID 格式 |
| `source-repo/packages/memory-host-sdk/src/host/qmd-query-parser.ts:7-16` | QMD 查詢結果型別 |
| `source-repo/packages/memory-host-sdk/src/host/` | 嵌入 provider 實作 |
| `source-repo/src/memory-host-sdk/engine.ts:1-8` | Memory Host SDK 核心匯出 |
| `source-repo/src/memory-host-sdk/engine-qmd.ts:1-22` | QMD SDK 匯出 |
| `source-repo/docs/concepts/memory-qmd.md` | QMD 概念文件 |
| `source-repo/src/gateway/session-transcript-files.fs.ts:70-269` | Session 檔案管理 |
| `source-repo/src/gateway/session-transcript-key.ts:15-157` | 轉錄鍵解析 |
| `source-repo/src/gateway/session-archive.fs.ts:22-55` | Session 歸檔分類 |
| `source-repo/src/gateway/session-compaction-checkpoints.ts:17-188` | 壓縮檢查點管理 |
