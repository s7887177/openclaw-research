# OpenClaw 記憶系統（Memory System）完整解析

## 本章摘要

OpenClaw 的記憶系統（Memory System）是整個框架中最關鍵的基礎設施之一。如果說大型語言模型（Large Language Model, LLM）是 AI Agent 的「大腦」，那麼記憶系統就是它的「海馬迴」——負責將短暫的對話資訊轉化為持久的知識，讓 Agent 能在不同時間點、不同對話脈絡中，始終如一地理解和記住與使用者互動的歷史。

沒有記憶系統的 AI Agent，就像一個每天醒來都失去前一天所有記憶的人。你今天告訴它你的名字，明天它就忘了；你昨天教它你偏好的程式語言，今天它又會從頭問起。這種體驗對使用者來說是極為糟糕的。OpenClaw 的記憶系統正是為了解決這個根本問題而設計的。

本章將從零開始，假設你對 OpenClaw 完全沒有任何了解，逐步拆解記憶系統的每一個組成部分。我們會涵蓋以下六大主題：

1. **Context Engine 框架**：記憶系統的核心抽象層，定義了記憶如何被攝入（Ingest）、組裝（Assemble）、壓縮（Compact）與維護（Maintain）。這是整個記憶系統的骨架，所有其他組件都建構在這個抽象層之上。
2. **Context Engine 註冊機制**：插件如何透過工廠模式（Factory Pattern）註冊自訂的 Context Engine。這個機制讓 OpenClaw 的記憶系統具備高度的可擴展性——你可以替換內建的記憶引擎，換成自己開發的版本。
3. **Memory Host SDK**：為記憶系統提供嵌入向量（Embedding）生成、QMD 文件處理、批次運算等底層能力。這是記憶系統與外部 AI 服務之間的橋樑。
4. **Memory Core 擴充套件**：OpenClaw 內建的檔案式記憶搜尋引擎，包含混合搜尋（Hybrid Search）演算法、最大邊際相關性（Maximal Marginal Relevance, MMR）排序、以及獨特的夢境系統（Dreaming System）。
5. **Memory LanceDB 擴充套件**：基於 LanceDB 向量資料庫的替代記憶儲存方案，適合需要高效能向量搜尋的場景。
6. **Active Memory 擴充套件**：最為先進的主動式記憶召回機制。它不等你查詢，而是在每次對話之前自動派出一個子代理（Sub-agent），幫你找出可能相關的過往記憶。

讀完本章後，你將完整理解 OpenClaw 如何讓一個 AI Agent「記住事情」，以及這套記憶系統在架構設計上的哲學思考——為什麼要分這麼多層？為什麼要用插件化架構？為什麼需要「夢境」？

---

## 1. 什麼是記憶系統？為什麼 AI Agent 需要它？

在深入技術細節之前，讓我們先建立對「AI Agent 記憶」這個概念的直覺理解。

### 1.1 語言模型的根本限制

大型語言模型本質上是一個「無狀態」（Stateless）的函式：給定一段輸入文字，它產出一段輸出文字，然後就忘記一切。每一次 API 呼叫都是獨立的。模型本身不會記得上一次呼叫的內容。

那我們平常使用 ChatGPT 時，為什麼感覺它「記得」我們剛才說的話呢？答案是：應用程式把你之前的所有對話訊息都塞回了下一次 API 呼叫的輸入中。但這帶來一個嚴重的問題——上下文視窗（Context Window）是有限的。即使是最新的模型，上下文視窗也只有幾十萬個 Token，而且 Token 是要花錢的。你不可能把過去一年的所有對話都塞進去。

### 1.2 三種記憶的需求

一個成熟的 AI Agent 需要至少三種層次的記憶能力：

- **短期記憶**（Short-term Memory）：記住當前這輪對話中的上下文。這是最基本的需求，通常透過直接保留對話歷史來實現。但即使是短期記憶，當對話過長時也需要壓縮（Compaction）策略。
- **長期記憶**（Long-term Memory）：記住跨對話、跨時段的重要資訊。例如使用者的偏好、曾經做過的決定、重要的事實等。長期記憶需要持久化儲存，並且需要有效的檢索機制。
- **主動召回**（Active Recall）：在適當時機自動想起相關記憶，而非被動地等待使用者查詢。就像人類在聊天時會自然地聯想到相關的過往經歷一樣，AI Agent 也應該能在對話中主動喚起相關的記憶。

OpenClaw 的記憶系統透過一個名為 **Context Engine**（上下文引擎）的核心抽象介面，將上述所有層次的記憶能力統一在一個可插拔的框架之下。接下來，讓我們深入這個框架的每一個組件。

---

## 2. Context Engine 框架：記憶系統的核心骨架

Context Engine 是 OpenClaw 記憶系統的核心抽象層。你可以把它想像成一個「記憶管理員」——它不關心記憶具體儲存在哪裡（檔案系統？向量資料庫？雲端服務？），也不關心記憶是用什麼演算法檢索的。它只定義了一套標準的操作介面：如何放入記憶、如何取出記憶、如何壓縮記憶。任何實作了這套介面的模組，都可以作為 OpenClaw 的記憶引擎來使用。

這種抽象設計帶來的好處是巨大的：你可以在不修改框架核心程式碼的情況下，完全替換記憶系統的底層實作。想用 Redis 儲存記憶？寫一個 Context Engine 插件。想用 PostgreSQL 搭配 pgvector 做向量搜尋？同樣寫一個插件就好。OpenClaw 自帶了多種實作（Memory Core、Memory LanceDB、Active Memory），但你完全可以用自己的版本替代它們。

### 2.1 核心型別定義

Context Engine 的所有型別定義集中在 `source-repo/src/context-engine/types.ts`（共 281 行）。這個檔案是理解整個記憶系統的起點，其中定義的每一個型別都對應著記憶生命週期中的一個關鍵環節。

#### AssembleResult — 組裝結果

「組裝」（Assemble）是記憶系統中最核心的操作之一。每當 Agent 需要回覆使用者時，它必須將各種來源的記憶——短期對話歷史、長期記憶片段、系統指令——組合成一份完整的提示（Prompt），送入語言模型。這個組裝過程的產出就是 `AssembleResult`：

```typescript
// source-repo/src/context-engine/types.ts:6-13
export type AssembleResult = {
  messages: AgentMessage[];        // 組裝後的訊息陣列
  estimatedTokens: number;         // 預估的 Token 數量
  systemPromptAddition?: string;   // 額外附加到系統提示的內容
};
```

這裡有三個欄位，每一個都有其設計考量。`messages` 是最終要傳給語言模型的訊息陣列，包含了經過篩選和排序的對話歷史與記憶內容。`estimatedTokens` 讓呼叫方知道這次組裝大約消耗了多少 Token 預算（Token Budget），這對於控制上下文視窗的使用至關重要——如果組裝後的 Token 數超出了模型的限制，系統就需要觸發壓縮。`systemPromptAddition` 則允許 Context Engine 在系統提示中動態注入額外內容，例如從長期記憶中提取出的使用者偏好資訊。

#### CompactResult — 壓縮結果

壓縮（Compaction）是記憶系統中極為重要的機制。想像一下：你和 AI Agent 聊了一整天，產生了數百輪對話。如果全部保留，Token 數會遠超模型的上下文視窗限制。壓縮的作用就是將較早的對話內容摘要化——保留關鍵資訊，但用更少的 Token 來表達。當對話累積到超過 Token 預算時，Context Engine 會自動觸發壓縮操作，其結果以 `CompactResult` 型別回傳：

```typescript
// source-repo/src/context-engine/types.ts:15-26
export type CompactResult = {
  ok: boolean;                     // 操作是否成功
  compacted: boolean;              // 是否實際執行了壓縮
  reason?: string;                 // 未壓縮的原因說明
  result?: {
    summary?: string;              // 壓縮後的摘要內容
    firstKeptEntryId?: string;     // 第一筆被保留的記錄 ID
    tokensBefore: number;          // 壓縮前的 Token 數
    tokensAfter?: number;          // 壓縮後的 Token 數
    details?: unknown;             // 額外細節（引擎自定義）
  };
};
```

值得注意的是 `compacted` 與 `ok` 的區分——操作可能成功（`ok: true`）但實際上並未壓縮（`compacted: false`），例如當前的 Token 使用量尚在預算之內，不需要壓縮。`reason` 欄位在這種情況下會解釋為什麼沒有壓縮。而 `tokensBefore` 與 `tokensAfter` 的對比讓我們能量化壓縮效率——例如從 50,000 Token 壓縮到 15,000 Token，壓縮率為 70%。

#### 其他基礎型別

除了上述兩個主要結果型別之外，Context Engine 還定義了一系列輔助型別，每個型別都服務於記憶生命週期中的特定環節：

| 型別 | 來源 | 用途 |
|------|------|------|
| `IngestResult` | `source-repo/src/context-engine/types.ts:28` | 單筆記憶攝入結果，包含 `ingested: boolean`，表示這筆資料是否成功被記憶系統接收 |
| `IngestBatchResult` | `source-repo/src/context-engine/types.ts:29` | 批次攝入結果，包含 `ingestedCount: number`，回報成功攝入的記錄數量 |
| `BootstrapResult` | `source-repo/src/context-engine/types.ts:38-45` | 引擎初始化結果，含 `importedMessages` 匯入訊息數與 `reason` 原因說明 |
| `ContextEngineInfo` | `source-repo/src/context-engine/types.ts:47-53` | 引擎元資料：`id` 識別碼、`name` 名稱、`version` 版本、`ownsCompaction` 是否自行管理壓縮 |
| `SubagentSpawnPreparation` | `source-repo/src/context-engine/types.ts:55-58` | 子代理生成前的準備工作，包含一個 `rollback` 回滾函式，在子代理生成失敗時可以還原狀態 |
| `SubagentEndReason` | `source-repo/src/context-engine/types.ts:60` | 子代理結束原因的聯合型別：`"deleted"`（被刪除）、`"completed"`（正常完成）、`"swept"`（被清掃）、`"released"`（被釋放） |

其中 `ContextEngineInfo` 的 `ownsCompaction` 欄位特別值得注意。如果一個 Context Engine 宣告自己擁有壓縮權（`ownsCompaction: true`），那麼框架就不會嘗試對它進行外部壓縮操作，而是完全由引擎自己決定何時、如何壓縮。這種設計允許高度客製化的壓縮策略。

### 2.2 ContextEngine 介面：記憶的完整生命週期

`ContextEngine` 介面（`source-repo/src/context-engine/types.ts:150-281`）是整個記憶系統最重要的型別定義。它描繪了一個記憶引擎從初始化到銷毀的完整生命週期——每一個方法都對應著記憶管理中的一個關鍵操作。理解這個介面，就等於理解了 OpenClaw 記憶系統的核心設計意圖。

| 方法 | 行號 | 必要性 | 說明 |
|------|------|--------|------|
| `info` | 152 | 必要（屬性） | 引擎的元資料，型別為 `ContextEngineInfo`，包含 ID、名稱、版本等識別資訊 |
| `bootstrap()` | 157 | 可選 | 引擎的初始化啟動方法，用於匯入既有的對話訊息或進行首次設定 |
| `maintain()` | 169 | 可選 | 定期維護任務的執行入口，例如重建搜尋索引、清理過期記憶等 |
| `ingest()` | 179 | **必要** | 攝入單筆新記憶或對話訊息，這是記憶寫入的基本單位 |
| `ingestBatch()` | 190 | 可選 | 批次攝入多筆記憶，當有大量資料需要一次性寫入時使用，效率高於多次呼叫 `ingest` |
| `afterTurn()` | 203 | 可選 | 每輪對話結束後自動觸發的回呼（Callback），可用於更新統計、觸發背景任務等 |
| `assemble()` | 224 | **必要** | 將記憶組裝成模型可用的訊息格式，是記憶讀取的核心入口 |
| `compact()` | 244 | **必要** | 壓縮過長的對話歷史，將舊訊息摘要化以釋放 Token 空間 |
| `prepareSubagentSpawn()` | 266 | 可選 | 為子代理（Sub-agent）的生成做前置準備工作，例如快照當前狀態 |
| `onSubagentEnded()` | 275 | 可選 | 當子代理結束時收到通知的回呼，可用於清理資源或合併子代理的記憶 |
| `dispose()` | 280 | 可選 | 引擎銷毀時的清理方法，用於釋放資料庫連線等資源 |

從這張表可以清楚看到，只有三個方法是必要的：`ingest`、`assemble`、`compact`。這三個方法構成了記憶系統的核心循環——**攝入 → 組裝 → 壓縮**——也就是「記住新東西 → 在需要時回想起來 → 當記得太多時精簡」。這個循環對應著人類記憶的基本運作方式：編碼（Encoding）→ 提取（Retrieval）→ 遺忘與鞏固（Forgetting & Consolidation）。

而可選方法則提供了更豐富的生命週期鉤子。例如 `bootstrap` 讓引擎可以在首次啟動時匯入大量既有資料；`maintain` 讓引擎可以定期進行索引優化；`afterTurn` 讓引擎可以在每輪對話後執行一些後處理邏輯；`prepareSubagentSpawn` 和 `onSubagentEnded` 則支援了多代理（Multi-agent）場景下的記憶管理。

#### assemble 方法的參數細節

`assemble` 方法（`source-repo/src/context-engine/types.ts:224-238`）是記憶系統中最常被呼叫的方法，它負責在每次使用者發送訊息時，從記憶中提取最相關的內容，並組裝成完整的上下文。讓我們仔細檢視它的參數設計：

| 參數 | 型別 | 說明 |
|------|------|------|
| `sessionId` | `string` | 當前會話的唯一識別碼，用於區分不同對話 |
| `sessionKey` | `string` | 會話存取金鑰，用於權限驗證與資料隔離 |
| `messages` | `AgentMessage[]` | 目前已有的訊息陣列，作為組裝的基礎 |
| `tokenBudget` | `number` | Token 預算上限，告訴引擎最多可以使用多少 Token |
| `availableTools?` | `Set<string>` | 可用工具名稱集合，讓引擎知道 Agent 能使用哪些工具，以便在提示中提供相應的引導 |
| `citationsMode?` | `MemoryCitationsMode` | 記憶引用模式，控制是否在回覆中標注記憶來源 |
| `model?` | `string` | 當前使用的模型識別碼，因為不同模型的 Token 計算方式可能不同 |
| `prompt?` | `string` | 使用者當前的提示文字，這是語義檢索（Semantic Retrieval）的關鍵入口——引擎可以根據這段文字去長期記憶中搜尋最相關的記憶片段 |

這些參數的設計展現了 OpenClaw 記憶系統的精巧之處。特別是 `prompt` 參數——它允許 Context Engine 根據使用者當前的問題來動態檢索最相關的長期記憶。比如使用者問「你還記得我上次說的那個專案嗎？」，引擎就可以用這段話作為查詢，去向量資料庫中搜尋語義上最接近的記憶條目。而 `availableTools` 參數則讓引擎可以在提示中加入工具使用的指引——如果 Agent 目前可以使用 `memory_search` 工具，引擎可能會在系統提示中加入「你可以使用記憶搜尋工具來查找更多相關資訊」之類的引導。

#### compact 方法的參數細節

`compact` 方法（`source-repo/src/context-engine/types.ts:244-258`）是記憶系統的「瘦身師」，負責在對話過長時進行壓縮。它的參數提供了豐富的控制選項，讓呼叫方可以精細地調整壓縮行為：

| 參數 | 型別 | 說明 |
|------|------|------|
| `sessionId` | `string` | 當前會話的唯一識別碼 |
| `sessionKey` | `string` | 會話存取金鑰 |
| `sessionFile` | `string` | 會話檔案的路徑，指向對話歷史的持久化儲存位置 |
| `tokenBudget` | `number` | Token 預算上限，壓縮後的結果不應超過此值 |
| `force?` | `boolean` | 是否強制執行壓縮，即使 Token 數尚未超標也執行 |
| `currentTokenCount?` | `number` | 當前實際的 Token 使用量，幫助引擎判斷壓縮的急迫性 |
| `compactionTarget?` | `number` | 壓縮目標 Token 數，指定壓縮後希望達到的精確 Token 數量 |
| `customInstructions?` | `string` | 自訂壓縮指令，例如「特別保留使用者的偏好設定」 |
| `runtimeContext?` | `ContextEngineRuntimeContext` | 執行環境上下文，提供壓縮過程中可能需要的環境資訊 |

`force` 參數的存在反映了一個重要的使用場景：有時候你希望在對話結束時主動觸發一次壓縮，將對話精華寫入長期記憶，即使當前的 Token 使用量還在預算之內。而 `compactionTarget` 則提供了精細的控制粒度——你可以告訴引擎「我希望壓縮後剩下大約 8,000 Token」，而不是只說「請壓縮到預算以內」。`customInstructions` 更是讓使用者可以用自然語言指導壓縮過程，例如「保留所有關於專案時程的討論」或「摘要時偏重技術細節而非閒聊」。

---

## 3. Context Engine 註冊機制：插件化的記憶引擎

OpenClaw 不會把記憶引擎的實作硬編碼在框架核心中。取而代之的是一套靈活的註冊與解析機制，讓任何插件都可以提供自己的 Context Engine 實作。這套機制的所有邏輯定義在 `source-repo/src/context-engine/registry.ts`（共 427 行）。

### 3.1 工廠模式與註冊

Context Engine 使用工廠模式（Factory Pattern）進行註冊。工廠模式的核心思想是：不直接建立物件，而是註冊一個「能建立物件的函式」，等到真正需要時才呼叫它。這在記憶系統中特別有意義——建立一個 Context Engine 可能涉及連接資料庫、載入模型、初始化索引等昂貴的操作，如果在系統啟動時就全部建立，會嚴重拖慢啟動速度。

```typescript
// source-repo/src/context-engine/registry.ts:10-11
export type ContextEngineFactory = () => ContextEngine | Promise<ContextEngine>;
export type ContextEngineRegistrationResult = { ok: true } | { ok: false; existingOwner: string };
```

工廠函式（Factory）是一個無參數函式，回傳一個 `ContextEngine` 實例或 Promise。這種延遲初始化（Lazy Initialization）設計意味著引擎只在實際被使用時才會被建立，避免了不必要的資源消耗。

### 3.2 兩種註冊方式

OpenClaw 提供了兩種 Context Engine 的註冊方式，分別針對不同的使用場景。

**帶擁有者的註冊**（內部使用）——這是更底層的 API，允許指定一個「擁有者」（Owner）字串，用於追蹤是誰註冊了這個引擎：

```typescript
// source-repo/src/context-engine/registry.ts:345-368
export function registerContextEngineForOwner(
  id: string,
  factory: ContextEngineFactory,
  owner: string,
  opts?: { allowSameOwnerRefresh?: boolean }
): ContextEngineRegistrationResult
```

注意 `opts` 參數中的 `allowSameOwnerRefresh` 選項——它允許同一個擁有者重新註冊同名的引擎，這在熱重載（Hot Reload）場景中非常有用。

**公開 SDK 註冊**（插件使用）——這是插件開發者會使用的簡化版 API：

```typescript
// source-repo/src/context-engine/registry.ts:377-382
export function registerContextEngine(
  id: string,
  factory: ContextEngineFactory
): ContextEngineRegistrationResult {
  return registerContextEngineForOwner(id, factory, PUBLIC_CONTEXT_ENGINE_OWNER);
}
```

公開版本的 `registerContextEngine` 是插件開發者會使用的 API，它內部委託給帶擁有者版本，並使用 `PUBLIC_CONTEXT_ENGINE_OWNER` 作為預設擁有者。擁有者機制的存在是為了安全性——它確保不同來源的引擎註冊不會互相覆蓋。如果插件 A 已經註冊了一個名為 `"my-engine"` 的引擎，插件 B 嘗試用相同的名稱再註冊一個，系統會回傳 `{ ok: false, existingOwner: "plugin-a" }`，拒絕覆蓋。

### 3.3 引擎解析（Resolution）：從配置到實例

當 OpenClaw 的對話系統實際需要使用 Context Engine 時，它會呼叫 `resolveContextEngine` 函式來取得引擎實例。這個解析過程是記憶系統啟動的關鍵步驟：

```typescript
// source-repo/src/context-engine/registry.ts:411-427
export async function resolveContextEngine(
  config?: OpenClawConfig
): Promise<ContextEngine> {
  const slotValue = config?.plugins?.slots?.contextEngine;
  const engineId = typeof slotValue === "string" && slotValue.trim()
    ? slotValue.trim()
    : defaultSlotIdForKey("contextEngine");
  const entry = getContextEngineRegistryState().engines.get(engineId);
  if (!entry) throw new Error(`Context engine "${engineId}" is not registered...`);
  return wrapContextEngineWithSessionKeyCompat(await entry.factory());
}
```

讓我們逐步拆解這個函式的邏輯，因為它展示了 OpenClaw 插件槽位（Plugin Slot）系統的運作方式：

1. **讀取配置**：首先從 `OpenClawConfig` 的 `plugins.slots.contextEngine` 欄位中讀取使用者指定的 Context Engine ID
2. **回退到預設值**：如果使用者沒有設定（或設定了空字串），就使用 `defaultSlotIdForKey("contextEngine")` 取得系統預設的引擎 ID
3. **查找工廠函式**：在全域註冊表中查找對應 ID 的工廠函式
4. **錯誤處理**：如果找不到對應的引擎，拋出明確的錯誤訊息
5. **建立與包裝**：呼叫工廠函式建立引擎實例，然後用 `wrapContextEngineWithSessionKeyCompat()` 包裝以確保向後相容性

這個流程的關鍵設計在於第 1-2 步：使用者可以透過配置檔完全控制要使用哪個記憶引擎，而不需要修改任何程式碼。這讓記憶引擎的切換變得像改一行設定一樣簡單。

### 3.4 向後相容性層：平滑升級的保障

在快速演進的開源專案中，API 的向後相容性是一個永恆的挑戰。OpenClaw 透過相容性包裝層來解決這個問題，確保使用舊版 API 開發的 Context Engine 插件不會因為框架升級而失效：

```
SESSION_KEY_COMPAT_METHODS: bootstrap, maintain, ingest, ingestBatch,
                            afterTurn, assemble, compact
```

`wrapContextEngineWithSessionKeyCompat()`（`source-repo/src/context-engine/registry.ts:17-36`）會對上述七個方法進行透明包裝，確保 Session Key 的傳遞方式與舊版 API 保持一致。這意味著即使框架內部重構了 Session Key 的處理邏輯，舊版的第三方引擎插件仍然可以正常運作，不需要任何修改。這種設計體現了 OpenClaw 對生態系統穩定性的重視。

### 3.5 其他輔助功能

除了核心的註冊與解析機制之外，`registry.ts` 還提供了兩個實用的查詢函式，方便開發者在開發和除錯時使用：

| 函式 | 行號 | 說明 |
|------|------|------|
| `getContextEngineFactory(id)` | `registry.ts:387-389` | 根據 ID 取得對應的工廠函式，如果 ID 不存在則回傳 `undefined` |
| `listContextEngineIds()` | `registry.ts:394-396` | 列出所有已註冊的引擎 ID 陣列，常用於除錯或建構引擎選擇器 UI |

### 3.6 Context Engine 目錄結構總覽

整個 Context Engine 框架由七個檔案組成，每個檔案都有明確的單一職責。這種模組化設計讓程式碼的可維護性大大提升——如果你只想了解註冊機制，只需要閱讀 `registry.ts`；如果你想了解向後相容如何實現，只需要閱讀 `legacy.ts`：

| 檔案 | 行數 | 職責 |
|------|------|------|
| `types.ts` | 281 | 核心介面與型別定義——定義了 `ContextEngine` 介面及所有相關型別 |
| `registry.ts` | 427 | 註冊表與解析邏輯——管理引擎的註冊、查找與實例化 |
| `index.ts` | 26 | 公開匯出（Public Exports）——統一的模組入口，控制哪些 API 對外可見 |
| `delegate.ts` | 101 | `delegateCompactionToRuntime()`——將壓縮操作委託給執行環境處理，適用於引擎本身不想處理壓縮的場景 |
| `init.ts` | 23 | `ensureContextEnginesInitialized()`——確保所有引擎已經完成初始化，通常在系統啟動時呼叫 |
| `legacy.ts` | 87 | `LegacyContextEngine`——舊版引擎的適配器實作，將舊版 API 映射到新版介面 |
| `legacy.registration.ts` | 8 | `registerLegacyContextEngine()`——用於註冊舊版格式的引擎，僅八行程式碼，足見其簡潔 |

總計不到 1,000 行程式碼，卻建構出一個高度靈活、可擴展、且向後相容的記憶引擎框架。這種精煉的設計值得仔細學習。

---

## 4. Memory Host SDK：記憶系統的底層基礎能力

如果說 Context Engine 框架定義了「記憶應該如何被管理」，那麼 Memory Host SDK（`source-repo/packages/memory-host-sdk/src/`）就是提供「記憶管理需要的底層工具」。它包含嵌入向量生成、文件處理、批次運算等核心能力，是連接記憶系統與外部 AI 服務的橋樑。

你可以把它想像成一個工具箱：Context Engine 是使用工具箱的工匠，而 Memory Host SDK 則是工具箱本身，裡面裝著各種螺絲起子、扳手和電鑽。

### 4.1 模組匯出結構

SDK 透過三個主要模組來組織其功能，每個模組聚焦於一個特定的能力領域：

| 模組檔案 | 匯出來源 | 職責 |
|----------|----------|------|
| `engine.ts` | engine-foundation、engine-storage、engine-embeddings、engine-qmd | 引擎核心能力的統一入口——把基礎設施、儲存、嵌入向量和 QMD 處理的所有功能匯集在一處 |
| `query.ts` | `extractKeywords`、`isQueryStopWordToken` | 查詢預處理工具——從使用者查詢中提取關鍵字並過濾停用詞（Stop Words），提升搜尋精確度 |
| `runtime.ts` | runtime-core、runtime-cli、runtime-files | 執行環境抽象——封裝了核心運行時、命令列介面和檔案系統操作的底層能力 |

### 4.2 嵌入向量提供者（Embedding Providers）

嵌入向量（Embedding）是語義搜尋的基礎技術。簡單來說，嵌入向量就是把一段文字轉換成一串數字（向量），讓語義相近的文字在向量空間中也彼此靠近。例如「我喜歡寫 Python」和「我偏好使用 Python 程式語言」這兩句話雖然用詞不同，但它們的嵌入向量會非常接近。有了嵌入向量，我們就可以做語義搜尋——不是比對關鍵字，而是比對「意思」。

OpenClaw 支援多種嵌入向量提供者，全部定義在 `source-repo/packages/memory-host-sdk/src/host/` 目錄下。這種多提供者架構意味著使用者可以根據需求、成本和隱私要求選擇最適合的方案：

| 檔案 | 行數 | 匯出函式 | 部署方式 | 說明 |
|------|------|----------|----------|------|
| `embeddings-openai.ts` | 58 | `createOpenAiEmbeddingProvider`、`DEFAULT_OPENAI_EMBEDDING_MODEL` | 雲端 | OpenAI 嵌入服務，最成熟穩定的選擇 |
| `embeddings-voyage.ts` | 82 | `createVoyageEmbeddingProvider`、`DEFAULT_VOYAGE_EMBEDDING_MODEL` | 雲端 | Voyage AI 嵌入服務，專為搜尋最佳化 |
| `embeddings-mistral.ts` | 51 | `createMistralEmbeddingProvider`、`DEFAULT_MISTRAL_EMBEDDING_MODEL` | 雲端 | Mistral AI 嵌入服務，歐洲替代方案 |
| `embeddings-bedrock.ts` | 398 | `createBedrockEmbeddingProvider` | 雲端 | AWS Bedrock 嵌入，企業級整合 |
| `embeddings-gemini.ts` | 336 | `createGeminiEmbeddingProvider` | 雲端 | Google Gemini 嵌入服務 |
| `embeddings-ollama.ts` | 5 | `createOllamaEmbeddingProvider` | 本地 | Ollama 本地嵌入，完全隱私 |
| `embeddings-lmstudio.ts` | 1 | `createLmstudioEmbeddingProvider` | 本地 | LM Studio 本地嵌入，圖形介面操作 |
| `embeddings-debug.ts` | 13 | Debug provider | 開發 | 除錯用的模擬提供者 |

從程式碼行數可以看出一些有趣的資訊：Bedrock（398 行）和 Gemini（336 行）的實作程式碼遠多於其他提供者，這可能是因為 AWS 和 Google 的 API 需要更複雜的認證流程和錯誤處理。而 Ollama（5 行）和 LM Studio（1 行）的極短實作暗示它們可能是基於某個通用的本地嵌入提供者介面進行簡單適配。

**雲端 vs 本地的選擇**：如果你處理的是敏感資料（例如醫療記錄或金融資訊），選擇 Ollama 或 LM Studio 這樣的本地方案可以確保資料永遠不會離開你的機器。但如果你追求最高品質的嵌入向量，OpenAI 或 Voyage 的雲端服務通常會有更好的表現。

### 4.3 QMD 處理

QMD 是 OpenClaw 記憶系統中使用的一種文件處理格式與工具鏈。QMD 處理模組負責與 QMD 二進位工具的互動、解析查詢結果、以及管理記憶的作用域（Scope）。這三個檔案各自負責 QMD 處理流程中的一個環節：

| 檔案 | 行數 | 關鍵匯出 | 職責 |
|------|------|----------|------|
| `qmd-process.ts` | 184 | `checkQmdBinaryAvailability`、`resolveCliSpawnInvocation`、`runCliCommand` | 負責與 QMD 二進位工具的互動——檢查工具是否可用、決定如何啟動 CLI 子程序、以及執行具體的 CLI 命令 |
| `qmd-query-parser.ts` | 153 | `parseQmdQueryJson`、`QmdQueryResult` 型別 | 負責解析 QMD 查詢回傳的 JSON 結果，將原始資料轉換為結構化的 `QmdQueryResult` 物件 |
| `qmd-scope.ts` | 110 | `deriveQmdScopeChannel`、`deriveQmdScopeChatType`、`isQmdScopeAllowed` | 負責記憶的作用域（Scope）控制，決定特定記憶在特定通道或聊天類型中是否可見 |

QMD Scope 模組特別值得深入理解。在多通道（Multi-channel）部署場景中——例如一個 Agent 同時服務於 Discord 群組、私人訊息和 Slack 頻道——不同通道的記憶可能需要隔離。你不會希望 Agent 在 Discord 群組中突然提起它在私人訊息中得知的資訊。`deriveQmdScopeChannel` 和 `deriveQmdScopeChatType` 正是負責計算這種記憶可見性規則的函式，而 `isQmdScopeAllowed` 則是最終的判斷函式——給定一筆記憶和一個上下文，它告訴你這筆記憶是否應該被包含在結果中。

### 4.4 其他 Host 模組

除了嵌入向量和 QMD 處理之外，Memory Host SDK 還包含幾個重要的輔助模組：

| 檔案 | 行數 | 職責 |
|------|------|------|
| `batch-runner.ts` | 64 | 協調批次嵌入運算（Batch Embedding Operations）——當需要一次性為大量文字生成嵌入向量時，這個模組負責分批處理、控制並發度、以及彙總結果 |
| `session-files.ts` | 161 | 會話檔案管理——提供 `buildSessionEntry`（建構會話條目）、`listSessionFilesForAgent`（列出某個 Agent 的所有會話檔案）、`sessionPathForFile`（計算特定檔案的會話路徑）三個核心函式 |
| `remote-http.ts` | 40 | 嵌入向量遠端 HTTP 客戶端——允許將嵌入向量的生成委託給遠端服務，適用於分散式部署場景 |

---

## 5. Memory Core 擴充套件：內建的智慧記憶引擎

Memory Core（`source-repo/extensions/memory-core/index.ts`）是 OpenClaw 自帶的記憶擴充套件，也是大多數使用者會使用的預設記憶方案。它提供了一整套基於檔案系統的記憶搜尋工具，並且包含了一些非常獨特的設計——尤其是夢境系統（Dreaming System）。

### 5.1 插件註冊：一次性注入所有記憶能力

Memory Core 使用 OpenClaw 的標準插件格式 `definePluginEntry` 進行註冊。讓我們仔細看看它在 `register` 函式中一口氣註冊了哪些能力：

```typescript
// source-repo/extensions/memory-core/index.ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "memory-core",
  name: "Memory (Core)",
  description: "File-backed memory search tools and CLI",
  kind: "memory",
  register(api) {
    registerBuiltInMemoryEmbeddingProviders(api);    // 行 30：註冊內建嵌入提供者
    registerShortTermPromotionDreaming(api);          // 行 31：註冊短期記憶提升夢境
    registerDreamingCommand(api);                     // 行 32：註冊夢境命令
    api.registerMemoryCapability({                    // 行 33：註冊記憶能力
      promptBuilder: buildPromptSection,
      flushPlanResolver: buildMemoryFlushPlan,
      runtime: memoryRuntime,
      publicArtifacts: { listArtifacts: listMemoryCorePublicArtifacts },
    });
    api.registerTool(createMemorySearchTool, {        // 行 42：註冊 memory_search 工具
      names: ["memory_search"]
    });
    api.registerTool(createMemoryGetTool, {           // 行 51：註冊 memory_get 工具
      names: ["memory_get"]
    });
    api.registerCli(registerMemoryCli, {              // 行 60：註冊 memory CLI
      descriptors: [{ name: "memory", ... }]
    });
  },
});
```

### 5.2 註冊能力詳解

Memory Core 在 `register` 函式中一次註冊了多項能力，涵蓋了從嵌入向量生成到命令列介面的完整功能鏈：

| 註冊項目 | 行號 | 說明 |
|----------|------|------|
| `registerBuiltInMemoryEmbeddingProviders` | 30 | 將 SDK 中支援的所有嵌入向量提供者（OpenAI、Voyage、Mistral 等）統一註冊到插件系統中 |
| `registerShortTermPromotionDreaming` | 31 | 註冊短期記憶提升夢境機制——負責將重要的短期記憶「提升」為長期記憶 |
| `registerDreamingCommand` | 32 | 註冊 `/dreaming` CLI 命令，讓使用者可以手動觸發或檢視夢境過程 |
| `registerMemoryCapability` | 33 | 註冊核心記憶能力，包含四個關鍵組件：提示建構器（promptBuilder）、刷新計畫解析器（flushPlanResolver）、記憶執行環境（runtime）、以及公開產物（publicArtifacts） |
| `memory_search` 工具 | 42 | 註冊 `memory_search` 工具，讓 Agent 可以在對話中搜尋長期記憶資料庫 |
| `memory_get` 工具 | 51 | 註冊 `memory_get` 工具，讓 Agent 可以根據 ID 取得特定的記憶條目 |
| `memory` CLI | 60 | 註冊命令列記憶管理介面，供開發者和管理員直接操作記憶資料 |

`registerMemoryCapability` 中的四個組件值得特別解釋：`promptBuilder`（由 `buildPromptSection` 實作）負責將相關記憶格式化為系統提示的一部分；`flushPlanResolver`（由 `buildMemoryFlushPlan` 實作）負責決定何時以及如何將暫存的短期記憶寫入長期儲存；`runtime` 提供記憶操作的執行環境；`publicArtifacts` 則提供列出所有公開記憶產物的能力。

### 5.3 內部模組結構與搜尋演算法

Memory Core 的內部實作分佈在多個模組中，每個模組都負責記憶管理流程中的一個特定環節。理解這些模組的分工，有助於我們掌握記憶搜尋從查詢到結果的完整流程：

| 模組路徑 | 職責 |
|----------|------|
| `src/memory/manager.ts` | `MemoryIndexManager` 類別——這是記憶索引管理的核心。它繼承自 `MemoryManagerEmbeddingOps`（提供嵌入向量操作能力），並實作 `MemorySearchManager` 介面（提供搜尋能力）。這種雙重繼承的設計讓 Manager 同時具備了寫入和讀取記憶的能力 |
| `src/memory/hybrid.ts` | 混合搜尋（Hybrid Search）實作——結合全文檢索（FTS, Full-Text Search）與向量搜尋（Vector Search）兩種方式。全文檢索擅長精確的關鍵字匹配，向量搜尋則擅長語義層面的模糊匹配。兩者結合可以在精確性和召回率之間取得最佳平衡 |
| `src/memory/mmr.ts` | 最大邊際相關性（Maximal Marginal Relevance, MMR）演算法實作——這是一個經典的資訊檢索演算法，用於在搜尋結果中平衡「相關性」和「多樣性」。如果只按相關性排序，前十筆結果可能都在說同一件事；MMR 演算法會確保結果中涵蓋不同面向的記憶 |
| `src/tools.ts` | `createMemorySearchTool()` 與 `createMemoryGetTool()` 的具體實作——定義了這兩個工具的輸入參數、執行邏輯和輸出格式 |
| `src/prompt-section.ts` | `buildPromptSection()`——負責將記憶內容構建為系統提示（System Prompt）的一部分。這個函式決定了記憶以什麼樣的格式和措辭呈現給語言模型 |
| `src/flush-plan.ts` | `buildMemoryFlushPlan()`——制定記憶刷新計畫，決定哪些暫存在記憶體中的短期記憶應該被持久化到磁碟上，以及如何組織這些持久化的資料 |

### 5.4 夢境系統（Dreaming System）：模擬人類記憶鞏固

夢境系統是 Memory Core 中最獨特、也最令人驚艷的設計。它的靈感來自神經科學——人類在睡眠時，大腦並沒有閒著，而是在進行記憶的鞏固和整理。海馬迴會將白天形成的短期記憶「回放」給大腦皮層，將重要的資訊轉化為長期記憶。OpenClaw 的夢境系統正是模擬了這個過程。

夢境系統由四個模組組成，對應著不同的處理階段和修復機制：

| 模組 | 職責 |
|------|------|
| `src/dreaming.ts` | 夢境系統的核心邏輯與調度器——決定何時開始夢境、選擇哪個階段、以及如何處理夢境產生的結果 |
| `src/dreaming-command.ts` | CLI 命令入口——讓使用者可以透過命令列手動觸發夢境過程或查看夢境狀態 |
| `src/dreaming-phases.ts` | 三個夢境階段的具體實作：**Light Sleep**（淺睡期）——快速掃描最近的記憶，識別重要的資訊片段；**Deep Sleep**（深睡期）——深度處理和整合記憶，建立記憶之間的關聯；**REM**（快速動眼期）——進行高層次的敘事整理和記憶修剪 |
| `src/dreaming-repair.ts` | 夢境修復機制——處理中斷或損壞的夢境過程。例如如果系統在深睡期意外關閉，修復模組可以在下次啟動時恢復或重新執行未完成的夢境 |

根據 CHANGELOG 的記錄，夢境系統經過了多次迭代和改進，這些改動記錄讓我們能看到系統的演進軌跡：

- 夢境系統支援**心跳事件**（Heartbeat Events），讓外部系統可以監控夢境過程是否正常運行（`source-repo/CHANGELOG:39-40`）
- **淺睡信心度**（Light-sleep Confidence）機制讓系統可以評估淺睡期記憶篩選的可靠度（`source-repo/CHANGELOG:39-40`）
- **敘事清理**（Narrative Cleanup）功能用於去除記憶中冗餘或矛盾的敘述片段（`source-repo/CHANGELOG:39-40`）
- 短期召回支援從**巢狀每日筆記**（Nested Daily Notes）中提取記憶，這意味著夢境系統可以理解結構化的筆記檔案（`source-repo/CHANGELOG:42`）
- 夢境系統可以與**外部記憶槽**（External Memory Slot）並行運作（`source-repo/CHANGELOG:57`），這意味著即使使用者安裝了第三方的記憶插件，Memory Core 的夢境功能仍然可以獨立運行，為其他記憶系統提供鞏固服務

---

## 6. Memory LanceDB 擴充套件：向量資料庫驅動的記憶儲存

Memory LanceDB（`source-repo/extensions/memory-lancedb/index.ts`）提供了一個基於 LanceDB 向量資料庫的替代記憶儲存方案。相較於 Memory Core 的檔案式儲存，LanceDB 方案在處理大量記憶條目時可能有更好的查詢效能，尤其是在向量近似最近鄰（Approximate Nearest Neighbor, ANN）搜尋方面。

### 6.1 記憶條目結構：每筆記憶的完整面貌

在 Memory LanceDB 中，每一筆記憶都以 `MemoryEntry` 型別儲存。這個型別的欄位設計揭示了 OpenClaw 對「記憶」這個概念的建模方式：

```typescript
// source-repo/extensions/memory-lancedb/index.ts:29-36
type MemoryEntry = {
  id: string;            // 記憶的唯一識別碼
  text: string;          // 記憶的文字內容
  vector: number[];      // 嵌入向量（Embedding Vector）
  importance: number;    // 重要性分數
  category: MemoryCategory; // 記憶分類
  createdAt: number;     // 建立時間戳記
};
```

每一筆記憶都包含其文字內容與對應的嵌入向量。這裡有幾個值得注意的設計選擇：

- **`importance` 分數**：這是一個數值欄位，讓系統在檢索時能優先回傳重要的記憶。例如使用者明確表達的偏好（importance 高）應該比隨口提到的細節（importance 低）更容易被想起來。這個分數可能是在記憶寫入時由語言模型自動評估的。
- **`category` 分類**：`MemoryCategory` 型別將記憶分門別類，例如事實（Facts）、偏好（Preferences）、對話摘要（Conversation Summaries）等。分類機制允許在搜尋時進行過濾——例如只搜尋使用者偏好類的記憶。
- **`vector` 嵌入向量**：以 `number[]` 陣列儲存，與文字內容配對。向量的維度取決於使用的嵌入模型（例如 OpenAI 的 text-embedding-3-small 產生 1536 維向量）。
- **`createdAt` 時間戳記**：記錄記憶的建立時間，支援按時間排序和過濾。新的記憶在某些場景下可能比舊的更相關。

### 6.2 MemoryDB 類別：延遲初始化的資料庫封裝

```typescript
// source-repo/extensions/memory-lancedb/index.ts:51-93
class MemoryDB {
  private db: LanceDB.Connection | null = null;
  private table: LanceDB.Table | null = null;
  private initPromise: Promise<void> | null = null;
  // 延遲初始化模式（Lazy Initialization Pattern）
}
```

`MemoryDB` 採用延遲初始化模式（Lazy Initialization Pattern）——資料庫連線（`db`）和表格參考（`table`）在類別建構時都是 `null`，只在首次存取時才透過 `initPromise` 進行初始化。這個設計帶來兩個好處：第一，如果這個擴充套件被載入但從未使用，就不會浪費資源去建立資料庫連線；第二，初始化只會執行一次（透過 `initPromise` 確保），即使多個併發的操作同時觸發初始化，也不會建立多個資料庫連線。

表名固定為 `"memories"`（`source-repo/extensions/memory-lancedb/index.ts:49`），這是所有記憶條目儲存的唯一表格。

該擴充套件依賴以下三個外部套件：
- **`@lancedb/lancedb`**：LanceDB 的 JavaScript 客戶端，提供向量資料庫的連線和查詢能力
- **`openai`**：用於呼叫 OpenAI 的嵌入向量 API，將文字轉換為向量
- **`@sinclair/typebox`**：用於執行時期的型別驗證，確保寫入資料庫的資料符合預期的結構

---

## 7. Active Memory 擴充套件：讓 Agent 主動想起相關記憶

Active Memory（`source-repo/extensions/active-memory/index.ts`）是記憶系統中最為先進、也最能體現「智慧」的部分。傳統的記憶系統是被動的——你必須明確地告訴 Agent「去搜尋記憶」，它才會去找。但人類的記憶不是這樣運作的：當朋友提到「上次那個餐廳」，你會自動想起上次去那間餐廳的經歷，不需要別人明確告訴你「請搜尋你的記憶」。

Active Memory 正是要實現這種主動式的記憶召回。它在每次對話輪次開始之前，自動在背景中派出一個專門的子代理（Sub-agent），分析當前的對話上下文，然後到記憶資料庫中搜尋可能相關的過往記憶，最後將這些記憶注入到即將組裝的提示中。整個過程對使用者完全透明——使用者不需要做任何事情，Agent 就會「自動想起」相關的事情。

### 7.1 預設常數：窺見設計哲學

一個系統的預設值往往透露了設計者對使用場景的理解和權衡考量。Active Memory 的預設常數就是一個很好的例子：

```typescript
// source-repo/extensions/active-memory/index.ts:19-31
const DEFAULT_TIMEOUT_MS = 15_000;          // 召回超時：15 秒
const DEFAULT_AGENT_ID = "main";            // 預設 Agent ID
const DEFAULT_MAX_SUMMARY_CHARS = 220;      // 摘要最大字元數
const DEFAULT_RECENT_USER_TURNS = 2;        // 考慮最近 2 輪使用者訊息
const DEFAULT_RECENT_ASSISTANT_TURNS = 1;   // 考慮最近 1 輪助手回覆
const DEFAULT_CACHE_TTL_MS = 15_000;        // 快取生存時間：15 秒
const DEFAULT_QUERY_MODE = "recent";        // 預設查詢模式
const DEFAULT_QMD_SEARCH_MODE = "search";   // 預設 QMD 搜尋模式
```

讓我們逐一分析這些預設值背後的設計思考：

- **15 秒超時**（`DEFAULT_TIMEOUT_MS`）：記憶召回不應該讓使用者等太久。15 秒是一個精心選擇的平衡點——足夠完成一次語義搜尋和摘要生成，但不會讓使用者覺得 Agent 反應遲鈍。如果在 15 秒內無法完成召回，系統會放棄並直接使用現有的上下文回覆。
- **主代理 ID**（`DEFAULT_AGENT_ID`）：預設為 `"main"`，表示 Active Memory 預設只為主要的對話代理服務。在多代理場景中，你可以透過配置讓 Active Memory 為特定的代理啟用。
- **摘要字元上限**（`DEFAULT_MAX_SUMMARY_CHARS`）：220 個字元的摘要上限確保了每筆召回的記憶不會佔用太多的 Token 預算，讓更多不同的記憶可以被包含在提示中。
- **最近對話輪次**（`DEFAULT_RECENT_USER_TURNS` 和 `DEFAULT_RECENT_ASSISTANT_TURNS`）：考慮最近 2 輪使用者訊息和 1 輪助手回覆來驅動記憶檢索。這是因為使用者最近的話語是判斷「需要回想什麼」的最佳線索。助手的回覆次數較少，因為我們更關心使用者說了什麼，而非 Agent 自己說了什麼。
- **快取生存時間**（`DEFAULT_CACHE_TTL_MS`）：15 秒的快取避免了在使用者快速連續發送訊息時重複執行相同的記憶查詢。如果兩條訊息間隔不到 15 秒，第二條訊息可以直接復用第一條的召回結果。
- **查詢模式**（`DEFAULT_QUERY_MODE`）：預設為 `"recent"`，使用最近幾輪對話作為查詢依據。這比只用當前單條訊息（`"message"`）更有上下文理解力，又比使用完整對話歷史（`"full"`）更有效率。

### 7.2 配置選項：高度可客製化的記憶召回

Active Memory 提供了極為豐富的配置選項，讓使用者可以根據自己的場景精確調整記憶召回的行為。以下是完整的配置型別定義及其各欄位的詳細解釋：

```typescript
// source-repo/extensions/active-memory/index.ts:57-88
type ActiveRecallPluginConfig = {
  enabled?: boolean;                    // 是否啟用 Active Memory
  agents?: string[];                    // 適用的 Agent ID 列表
  model?: string;                       // 記憶召回子代理使用的模型
  modelFallback?: string;               // 主模型不可用時的備用模型
  allowedChatTypes?: Array<             // 限制在哪些聊天類型中啟用
    "direct" | "group" | "channel"
  >;
  thinking?: "off" | "minimal" | "low"  // 子代理的思考深度
    | "medium" | "high" | "xhigh" | "adaptive";
  promptStyle?: "balanced" | "strict"   // 提示風格
    | "contextual" | "recall-heavy"
    | "precision-heavy" | "preference-only";
  timeoutMs?: number;                   // 召回超時（毫秒）
  queryMode?: "message" | "recent"      // 查詢上下文的選取模式
    | "full";
  cacheTtlMs?: number;                  // 快取有效期
  persistTranscripts?: boolean;         // 是否保存召回過程的完整記錄
  transcriptDir?: string;               // 記錄的儲存目錄
  qmd?: {                               // QMD 搜尋相關設定
    searchMode?: "inherit" | "search"
      | "vsearch" | "query";
  };
};
```

**配置亮點深入解析**：

**`thinking` 欄位**提供了七個級別的思考深度控制，這在 AI 系統中是相當罕見的精細度。`"off"` 表示完全不進行推理，直接基於關鍵字匹配；`"minimal"` 和 `"low"` 做基礎的語義理解；`"medium"` 是一般用途的平衡選擇；`"high"` 和 `"xhigh"` 會投入更多計算資源來深度推理；`"adaptive"` 則讓系統根據當前對話的複雜度自動選擇適當的思考深度。思考深度越高，召回品質通常越好，但延遲也越長。

**`promptStyle` 欄位**提供了六種預設的提示策略，每種策略代表了不同的記憶召回哲學：
- `"balanced"`：平衡模式，不偏向任何特定方向
- `"strict"`：嚴格模式，只召回高度確定相關的記憶
- `"contextual"`：上下文模式，重視與當前對話脈絡的關聯性
- `"recall-heavy"`：重召回模式，盡可能多地召回記憶，寧可多不可少
- `"precision-heavy"`：重精確模式，只召回最精確匹配的記憶
- `"preference-only"`：偏好模式，只召回使用者偏好類的記憶

**`allowedChatTypes` 欄位**在多通道場景中非常實用。例如你可能希望 Active Memory 只在私聊（`"direct"`）中啟用，因為在群組中主動提起使用者的私人記憶可能不太恰當。

**`qmd.searchMode` 欄位**控制 QMD 層面的搜尋策略：`"search"` 做標準搜尋、`"vsearch"` 做向量搜尋、`"query"` 做結構化查詢、`"inherit"` 則繼承全域設定。

### 7.3 插件註冊與核心運作機制

Active Memory 的插件定義本身相當簡潔，但它透過事件勾點（Event Hook）機制實現了強大的功能：

```typescript
// source-repo/extensions/active-memory/index.ts:1764-1768
export default definePluginEntry({
  id: "active-memory",
  name: "Active Memory",
  description: "Proactively surfaces relevant memory before eligible conversational replies.",
  register(api: OpenClawPluginApi) {
    // 行 1785：註冊 /active-memory 命令（status|on|off [--global]）
    // 行 1852：api.on("before_prompt_build", ...) — 主要勾點
  }
});
```

Active Memory 的核心運作機制是透過 `before_prompt_build` 事件勾點（Hook）（`source-repo/extensions/active-memory/index.ts:1852`）。這是 OpenClaw 插件系統提供的一個關鍵事件——它在每次 Agent 開始組裝提示之前觸發。Active Memory 監聽這個事件，並在觸發時執行以下流程：

1. **收集上下文**：根據 `queryMode` 設定，收集最近的對話歷史作為查詢依據。如果是 `"recent"` 模式，會取最近 2 輪使用者訊息和 1 輪助手回覆。
2. **啟動召回子代理**：派出一個專門的子代理，使用收集到的上下文作為查詢條件，在記憶資料庫中進行語義搜尋。
3. **摘要與篩選**：子代理將搜尋到的記憶片段進行摘要化處理（每筆最多 220 字元），篩選出最相關的結果。
4. **注入提示**：將篩選後的記憶內容注入到即將組裝的系統提示中，讓主 Agent 在回覆時能「看到」這些相關記憶。
5. **超時保護**：整個過程受 `timeoutMs`（預設 15 秒）的限制。如果在超時前無法完成，系統會放棄召回並讓主 Agent 直接回覆。

這也正是 CHANGELOG 中所描述的「dedicated sub-agent before main reply」機制（`source-repo/CHANGELOG:12`）——一個專門的子代理在主要回覆之前運行，提前準備好記憶上下文。

此外，Active Memory 還提供了命令列控制界面（`source-repo/extensions/active-memory/index.ts:1785`）。使用者可以透過 `/active-memory` 命令即時控制記憶召回的開關狀態：
- `/active-memory status`：查看當前的啟用狀態和配置
- `/active-memory on`：啟用 Active Memory
- `/active-memory off`：停用 Active Memory
- `--global` 旗標：讓設定對所有會話生效

### 7.4 Active Memory 的演進與整合

Active Memory 作為一個相對較新的功能，經歷了快速的迭代和改進。從 CHANGELOG 中我們可以追蹤它的演進軌跡：

- **初始引入**：Active Memory 以 PR #63286 的形式首次被引入系統（`source-repo/CHANGELOG:12`），其核心概念是在主要回覆之前運行一個專門的記憶召回子代理
- **QMD 搜尋模式調整**：預設的 QMD 搜尋模式經過專門調整（`source-repo/CHANGELOG:26`），以更好地配合 Active Memory 的查詢模式
- **召回品質改進**：經過多個 PR（#65049、#65395）的迭代，記憶召回的精準度和相關性得到顯著提升（`source-repo/CHANGELOG:38`）
- **與夢境系統整合**：Active Memory 已經與夢境系統（Dreaming）完成整合（`source-repo/CHANGELOG:47`），這意味著夢境階段產生的鞏固記憶可以被 Active Memory 即時召回。換句話說，Agent「睡覺」時整理好的記憶，醒來後就能自動想起來

---

## 8. 系統協作全景：記憶如何在各組件間流動

現在讓我們站在更高的視角，將所有組件串聯起來，看看一筆記憶是如何在 OpenClaw 的記憶系統中流動的。從使用者發送一條訊息開始，到 Agent 回覆，再到背景的記憶鞏固，整個過程涉及多個組件的協作：

```
使用者訊息
    │
    ▼
┌─────────────────────────────────┐
│  Active Memory (before_prompt)  │ ← 主動召回相關記憶
│  子代理執行語義檢索              │
└──────────────┬──────────────────┘
               │ 注入召回的記憶
               ▼
┌─────────────────────────────────┐
│  Context Engine.assemble()      │ ← 組裝完整上下文
│  Token Budget 控制               │
│  記憶 + 對話歷史 → 提示          │
└──────────────┬──────────────────┘
               │
               ▼
         模型產生回覆
               │
               ▼
┌─────────────────────────────────┐
│  Context Engine.ingest()        │ ← 攝入新的對話記錄
│  Context Engine.afterTurn()     │ ← 輪次後處理
└──────────────┬──────────────────┘
               │
               ▼ (若 Token 超標)
┌─────────────────────────────────┐
│  Context Engine.compact()       │ ← 壓縮歷史記錄
│  保留摘要，釋放 Token 空間       │
└──────────────┬──────────────────┘
               │
               ▼ (背景)
┌─────────────────────────────────┐
│  Dreaming System                │ ← 記憶鞏固
│  Light → Deep → REM             │
│  短期記憶 → 長期記憶             │
└─────────────────────────────────┘
```

關鍵的架構決策與設計哲學：

1. **插件化設計**：Context Engine 透過 `resolveContextEngine()`（`source-repo/src/context-engine/registry.ts:411-427`）動態解析，使用者可以在設定檔中切換不同的記憶引擎。這種設計讓記憶系統成為了一個真正的「可插拔」模組——你可以根據專案需求選擇最適合的記憶方案，從簡單的檔案式儲存到複雜的分散式向量資料庫。
2. **Context Engine Slot 保留**：系統確保 Context Engine 的槽位（Slot）設定在設定檔的正規化（Normalization）過程中不會遺失（`source-repo/CHANGELOG:242`）。這是一個看似微小但極為重要的細節——如果正規化過程不小心清除了使用者的記憶引擎設定，Agent 就會退回使用預設引擎，導致之前累積的記憶無法被正確存取。
3. **向後相容**：透過 `LegacyContextEngine`（`source-repo/src/context-engine/legacy.ts`）和 Session Key 相容性包裝層（`source-repo/src/context-engine/registry.ts:17-36`），舊版引擎插件可以在新版框架中無縫運作。這對於一個擁有豐富插件生態的開源專案來說至關重要。
4. **多層次記憶**：短期記憶透過 Context Engine 的 `ingest`/`assemble` 循環管理，長期記憶透過 Memory Core 或 LanceDB 持久化儲存，而 Active Memory 則在兩者之間架起橋樑，讓長期記憶能在適當時機自動浮現。
5. **仿生設計**：夢境系統的三個階段（Light Sleep → Deep Sleep → REM）直接模擬了人類大腦的記憶鞏固機制。這不僅僅是一個有趣的命名慣例——它反映了 OpenClaw 團隊對記憶管理的深層理解：記憶不只是「存下來」，還需要「整理」和「鞏固」。

---

## 引用來源

| 編號 | 來源路徑 | 行號範圍 | 內容描述 |
|------|----------|----------|----------|
| 1 | `source-repo/src/context-engine/types.ts` | 6-13 | `AssembleResult` 型別定義 |
| 2 | `source-repo/src/context-engine/types.ts` | 15-26 | `CompactResult` 型別定義 |
| 3 | `source-repo/src/context-engine/types.ts` | 28-29 | `IngestResult` 與 `IngestBatchResult` 型別定義 |
| 4 | `source-repo/src/context-engine/types.ts` | 38-45 | `BootstrapResult` 型別定義 |
| 5 | `source-repo/src/context-engine/types.ts` | 47-53 | `ContextEngineInfo` 型別定義 |
| 6 | `source-repo/src/context-engine/types.ts` | 55-58 | `SubagentSpawnPreparation` 型別定義 |
| 7 | `source-repo/src/context-engine/types.ts` | 60 | `SubagentEndReason` 型別定義 |
| 8 | `source-repo/src/context-engine/types.ts` | 150-281 | `ContextEngine` 完整介面定義 |
| 9 | `source-repo/src/context-engine/types.ts` | 224-238 | `assemble` 方法參數定義 |
| 10 | `source-repo/src/context-engine/types.ts` | 244-258 | `compact` 方法參數定義 |
| 11 | `source-repo/src/context-engine/registry.ts` | 10-11 | `ContextEngineFactory` 與 `ContextEngineRegistrationResult` 型別 |
| 12 | `source-repo/src/context-engine/registry.ts` | 17-36 | Session Key 向後相容方法列表與包裝函式 |
| 13 | `source-repo/src/context-engine/registry.ts` | 345-368 | `registerContextEngineForOwner()` 帶擁有者註冊 |
| 14 | `source-repo/src/context-engine/registry.ts` | 377-382 | `registerContextEngine()` 公開 SDK 註冊入口 |
| 15 | `source-repo/src/context-engine/registry.ts` | 387-389 | `getContextEngineFactory()` 取得工廠函式 |
| 16 | `source-repo/src/context-engine/registry.ts` | 394-396 | `listContextEngineIds()` 列出引擎 ID |
| 17 | `source-repo/src/context-engine/registry.ts` | 411-427 | `resolveContextEngine()` 引擎解析邏輯 |
| 18 | `source-repo/packages/memory-host-sdk/src/engine.ts` | — | SDK 引擎模組匯出 |
| 19 | `source-repo/packages/memory-host-sdk/src/query.ts` | — | 查詢工具匯出 |
| 20 | `source-repo/packages/memory-host-sdk/src/runtime.ts` | — | 執行環境匯出 |
| 21 | `source-repo/packages/memory-host-sdk/src/host/embeddings-openai.ts` | 58 行 | OpenAI 嵌入提供者 |
| 22 | `source-repo/packages/memory-host-sdk/src/host/embeddings-voyage.ts` | 82 行 | Voyage AI 嵌入提供者 |
| 23 | `source-repo/packages/memory-host-sdk/src/host/embeddings-mistral.ts` | 51 行 | Mistral 嵌入提供者 |
| 24 | `source-repo/packages/memory-host-sdk/src/host/embeddings-bedrock.ts` | 398 行 | AWS Bedrock 嵌入提供者 |
| 25 | `source-repo/packages/memory-host-sdk/src/host/embeddings-gemini.ts` | 336 行 | Google Gemini 嵌入提供者 |
| 26 | `source-repo/packages/memory-host-sdk/src/host/embeddings-ollama.ts` | 5 行 | Ollama 本地嵌入提供者 |
| 27 | `source-repo/packages/memory-host-sdk/src/host/embeddings-lmstudio.ts` | 1 行 | LM Studio 本地嵌入提供者 |
| 28 | `source-repo/packages/memory-host-sdk/src/host/embeddings-debug.ts` | 13 行 | 除錯用嵌入提供者 |
| 29 | `source-repo/packages/memory-host-sdk/src/host/qmd-process.ts` | 184 行 | QMD 二進位操作與 CLI 命令 |
| 30 | `source-repo/packages/memory-host-sdk/src/host/qmd-query-parser.ts` | 153 行 | QMD 查詢結果解析 |
| 31 | `source-repo/packages/memory-host-sdk/src/host/qmd-scope.ts` | 110 行 | QMD 作用域控制 |
| 32 | `source-repo/packages/memory-host-sdk/src/host/batch-runner.ts` | 64 行 | 批次嵌入運算協調 |
| 33 | `source-repo/packages/memory-host-sdk/src/host/session-files.ts` | 161 行 | 會話檔案管理 |
| 34 | `source-repo/packages/memory-host-sdk/src/host/remote-http.ts` | 40 行 | 嵌入向量遠端 HTTP 客戶端 |
| 35 | `source-repo/extensions/memory-core/index.ts` | 全檔 | Memory Core 插件註冊（含行 30-60） |
| 36 | `source-repo/extensions/memory-core/src/memory/manager.ts` | — | `MemoryIndexManager` 類別 |
| 37 | `source-repo/extensions/memory-core/src/memory/hybrid.ts` | — | 混合搜尋實作 |
| 38 | `source-repo/extensions/memory-core/src/memory/mmr.ts` | — | MMR 演算法實作 |
| 39 | `source-repo/extensions/memory-core/src/dreaming.ts` | — | 夢境系統核心 |
| 40 | `source-repo/extensions/memory-core/src/dreaming-phases.ts` | — | 夢境三階段實作 |
| 41 | `source-repo/extensions/memory-core/src/dreaming-repair.ts` | — | 夢境修復機制 |
| 42 | `source-repo/extensions/memory-core/src/tools.ts` | — | 記憶搜尋與取得工具 |
| 43 | `source-repo/extensions/memory-core/src/prompt-section.ts` | — | 提示區段建構 |
| 44 | `source-repo/extensions/memory-core/src/flush-plan.ts` | — | 記憶刷新計畫 |
| 45 | `source-repo/extensions/memory-lancedb/index.ts` | 29-36 | `MemoryEntry` 型別定義 |
| 46 | `source-repo/extensions/memory-lancedb/index.ts` | 49 | 表名 `"memories"` |
| 47 | `source-repo/extensions/memory-lancedb/index.ts` | 51-93 | `MemoryDB` 類別定義 |
| 48 | `source-repo/extensions/active-memory/index.ts` | 19-31 | Active Memory 預設常數 |
| 49 | `source-repo/extensions/active-memory/index.ts` | 57-88 | `ActiveRecallPluginConfig` 配置型別 |
| 50 | `source-repo/extensions/active-memory/index.ts` | 1764-1768 | Active Memory 插件註冊 |
| 51 | `source-repo/extensions/active-memory/index.ts` | 1785 | `/active-memory` CLI 命令 |
| 52 | `source-repo/extensions/active-memory/index.ts` | 1852 | `before_prompt_build` 事件勾點 |
| 53 | `source-repo/CHANGELOG` | 12 | Active Memory 子代理機制引入 |
| 54 | `source-repo/CHANGELOG` | 26 | Active Memory QMD 預設搜尋模式 |
| 55 | `source-repo/CHANGELOG` | 38-40 | 記憶召回與夢境系統改進 |
| 56 | `source-repo/CHANGELOG` | 42 | 巢狀每日筆記短期召回 |
| 57 | `source-repo/CHANGELOG` | 47 | Active Memory 與夢境整合 |
| 58 | `source-repo/CHANGELOG` | 57 | 外部記憶槽並行夢境 |
| 59 | `source-repo/CHANGELOG` | 242 | Context Engine Slot 正規化保留 |
