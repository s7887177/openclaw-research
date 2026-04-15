# Agents 子章節 03：多 Agent 編排

> **引用範圍**：`src/agents/subagent-spawn.ts`, `subagent-registry.ts`, `subagent-registry.types.ts`

## 1. 概觀：樹狀任務分解

OpenClaw 的多 Agent 編排採用**樹狀結構**：父 Agent 透過 `sessions_spawn` 工具生成子 Agent（Sub-agent），子 Agent 完成任務後將結果回報給父 Agent。

```
Root Agent (depth=0)
├── Sub-agent A (depth=1) — 文件撰寫
├── Sub-agent B (depth=1) — 程式碼審查
└── Sub-agent C (depth=1) — 測試執行
```

預設 `maxSpawnDepth=1`，表示 Sub-agent 不能再生成子 Agent。可配置增加深度。

---

## 2. 核心型別

### 2.1 SpawnSubagentParams

```typescript
// source-repo/src/agents/subagent-spawn.ts:83-103
export type SpawnSubagentParams = {
  task: string;                    // 任務描述（必填）
  label?: string;                  // 人類可讀標籤
  agentId?: string;                // 目標 Agent ID（可選）
  model?: string;                  // 模型覆寫
  thinking?: string;               // Thinking 等級覆寫
  runTimeoutSeconds?: number;      // 執行逾時
  thread?: boolean;                // 要求建立獨立 Thread
  mode?: SpawnSubagentMode;        // 執行模式
  cleanup?: "delete" | "keep";     // 完成後是否清理
  sandbox?: SpawnSubagentSandboxMode;  // 沙箱要求
  lightContext?: boolean;          // 輕量上下文（減少 token）
  expectsCompletionMessage?: boolean;  // 是否期待完成訊息
  attachments?: Array<{            // 檔案附件
    name: string;
    content: string;
    encoding?: "utf8" | "base64";
    mimeType?: string;
  }>;
  attachMountPath?: string;        // 附件掛載路徑
};
```

> **引用來源**：`source-repo/src/agents/subagent-spawn.ts:83-103`

### 2.2 SpawnSubagentContext

```typescript
// source-repo/src/agents/subagent-spawn.ts:105-117
export type SpawnSubagentContext = {
  agentSessionKey?: string;         // 父 Agent Session Key
  agentChannel?: string;            // 來源通道
  agentAccountId?: string;          // 來源帳號
  agentTo?: string;                 // 目標地址
  agentThreadId?: string | number;  // Thread ID
  agentGroupId?: string | null;     // 群組 ID
  agentGroupChannel?: string | null;
  agentGroupSpace?: string | null;
  requesterAgentIdOverride?: string;
  workspaceDir?: string;            // 工作區繼承
};
```

> **引用來源**：`source-repo/src/agents/subagent-spawn.ts:105-117`

### 2.3 SpawnSubagentResult

```typescript
// source-repo/src/agents/subagent-spawn.ts:119-133
export type SpawnSubagentResult = {
  status: "accepted" | "forbidden" | "error";
  childSessionKey?: string;        // 子 Session Key
  runId?: string;                  // 執行 ID
  mode?: SpawnSubagentMode;        // 實際執行模式
  note?: string;                   // 附加說明
  modelApplied?: boolean;          // 模型覆寫是否生效
  error?: string;                  // 錯誤訊息
  attachments?: {                  // 附件處理結果
    count: number;
    totalBytes: number;
    files: Array<{ name: string; bytes: number; sha256: string }>;
    relDir: string;
  };
};
```

> **引用來源**：`source-repo/src/agents/subagent-spawn.ts:119-133`

---

## 3. 生成流程：spawnSubagentDirect()

`spawnSubagentDirect()` 是 Sub-agent 生成的核心函式（346 行開始），流程如下：

### 3.1 前置驗證（6 道防線）

```
1. Agent ID 格式驗證
   └─ isValidAgentId(): 必須匹配 [a-z0-9][a-z0-9_-]{0,63}
   └─ 拒絕惡意/錯誤格式（防止建立幽靈工作區，ref #31311）

2. 深度限制檢查
   └─ callerDepth >= maxSpawnDepth → forbidden
   └─ 預設 maxSpawnDepth=1

3. 子 Agent 數量限制
   └─ activeChildren >= maxChildrenPerAgent → forbidden
   └─ 預設 maxChildrenPerAgent=5

4. requireAgentId 檢查
   └─ 如果配置要求明確 agentId，但未提供 → forbidden

5. 目標 Agent 白名單
   └─ allowAgents 列表驗證
   └─ "*" 表示允許任意目標

6. 沙箱安全約束
   └─ 沙箱 Agent 不能生成非沙箱 Sub-agent
   └─ sandbox="require" 需要目標支援沙箱
```

> **引用來源**：`source-repo/src/agents/subagent-spawn.ts:346-494`

### 3.2 Session Key 生成

```typescript
// source-repo/src/agents/subagent-spawn.ts:472
const childSessionKey = `agent:${targetAgentId}:subagent:${crypto.randomUUID()}`;
```

子 Session 的 Key 格式為 `agent:{agentId}:subagent:{uuid}`，確保全局唯一。

### 3.3 執行模式

```typescript
// source-repo/src/agents/subagent-spawn.types.ts
export const SUBAGENT_SPAWN_MODES = ["background", "session"] as const;
export type SpawnSubagentMode = (typeof SUBAGENT_SPAWN_MODES)[number];
```

| 模式 | 說明 |
|------|------|
| `background` | 後台執行，完成後回報結果 |
| `session` | 持久 Session（需 `thread=true`），綁定到 Thread |

### 3.4 模型解析

模型覆寫透過 `resolveSubagentModelAndThinkingPlan()`（在 `subagent-spawn-plan.ts` 中）解析：

```
優先順序：
1. params.model（明確指定）
2. 父 Agent 的 subagents.model 配置
3. defaults.subagents.model
4. 父 Agent 的主模型
5. 全局預設模型
```

### 3.5 Thread 綁定

當 `thread=true` 時，系統透過生命週期 Hook 請求通道 Plugin 建立 Thread：

```typescript
// source-repo/src/agents/subagent-spawn.ts:308-337
const result = await hookRunner.runSubagentSpawning({
  childSessionKey, agentId, label, mode, requester,
  threadRequested: true,
}, { childSessionKey, requesterSessionKey });
```

如果通道 Plugin 不支援 Thread，操作失敗。

> **引用來源**：`source-repo/src/agents/subagent-spawn.ts:300-344`

---

## 4. Sub-agent Registry

`subagent-registry.ts`（933 行）管理所有 Sub-agent 的生命週期。

### 4.1 依賴注入架構

```typescript
// source-repo/src/agents/subagent-registry.ts:66-80
type SubagentRegistryDeps = {
  callGateway: typeof callGateway;
  captureSubagentCompletionReply: ...;
  cleanupBrowserSessionsForLifecycleEnd: ...;
  getSubagentRunsSnapshotForRead: ...;
  loadConfig: typeof loadConfig;
  onAgentEvent: typeof onAgentEvent;
  persistSubagentRunsToDisk: ...;
  resolveAgentTimeoutMs: typeof resolveAgentTimeoutMs;
  restoreSubagentRunsFromDisk: ...;
  runSubagentAnnounceFlow: ...;
  ensureContextEnginesInitialized?: () => void;
  ensureRuntimePluginsLoaded?: ...;
  resolveContextEngine?: (cfg) => Promise<ContextEngine>;
};
```

Registry 使用**依賴注入**模式，所有外部依賴透過 `SubagentRegistryDeps` 注入，方便測試和替換。

> **引用來源**：`source-repo/src/agents/subagent-registry.ts:66-80`

### 4.2 模組拆分

Registry 被拆分為多個子模組：

| 模組 | 職責 |
|------|------|
| `subagent-registry.ts` | 主邏輯、生命週期事件監聽 |
| `subagent-registry-memory.ts` | 記憶體中的 Run 記錄 (`subagentRuns`) |
| `subagent-registry-queries.ts` | 查詢函式（按 session、requester 等） |
| `subagent-registry-state.ts` | 磁碟持久化（snapshot/restore） |
| `subagent-registry-helpers.ts` | 工具函式（狀態解析、孤兒偵測） |
| `subagent-registry-completion.ts` | 完成回調、Hook 發射 |
| `subagent-registry-lifecycle.ts` | 生命週期控制器建立 |
| `subagent-registry-run-manager.ts` | Run 管理（建立/更新/刪除） |
| `subagent-registry-steer-runtime.ts` | 運行時轉向配置 |

### 4.3 查詢 API

```typescript
// source-repo/src/agents/subagent-registry-queries.ts (匯出)
countActiveDescendantRunsFromRuns(...)     // 計算活動後代 Run 數
countActiveRunsForSessionFromRuns(...)     // 計算 Session 的活動 Run 數
countPendingDescendantRunsFromRuns(...)    // 計算待處理後代 Run 數
findRunIdsByChildSessionKeyFromRuns(...)   // 用子 Session Key 找 Run ID
listRunsForControllerFromRuns(...)         // 列出控制器的 Run
listDescendantRunsForRequesterFromRuns(...)  // 列出請求者的後代 Run
listRunsForRequesterFromRuns(...)          // 列出請求者的所有 Run
resolveRequesterForChildSessionFromRuns(...)  // 反查請求者
shouldIgnorePostCompletionAnnounceForSessionFromRuns(...)  // 完成後宣告篩選
```

> **引用來源**：`source-repo/src/agents/subagent-registry.ts:37-47`

### 4.4 狀態持久化

Registry 狀態持久化到磁碟，確保程序重啟後能恢復 Sub-agent 狀態：

```typescript
// source-repo/src/agents/subagent-registry-state.ts
getSubagentRunsSnapshotForRead(...)   // 讀快照
persistSubagentRunsToDisk(...)        // 寫磁碟
restoreSubagentRunsFromDisk(...)      // 從磁碟恢復
```

### 4.5 Sweeper 機制

Registry 使用 Sweeper（`sweeper` 變數）定期掃描：

```typescript
// source-repo/src/agents/subagent-registry.ts:122-123
let sweeper: NodeJS.Timeout | null = null;
let sweepInProgress = false;
```

Sweeper 負責：
- 偵測孤兒 Run（父 Agent 已結束但子 Agent 仍在運行）
- 清理超時的 Run
- 刪除過期的 Session 記錄

---

## 5. 生命週期事件

Sub-agent 的生命週期由以下事件驅動：

```
spawning → started → (running) → ended
                                   ├── complete
                                   ├── error
                                   └── killed
```

```typescript
// source-repo/src/agents/subagent-lifecycle-events.ts
SUBAGENT_ENDED_REASON_COMPLETE   // 正常完成
SUBAGENT_ENDED_REASON_ERROR      // 錯誤結束
SUBAGENT_ENDED_REASON_KILLED     // 被終止
```

### 5.1 錯誤重試寬限

```typescript
// source-repo/src/agents/subagent-registry.ts:136
const LIFECYCLE_ERROR_RETRY_GRACE_MS = 15_000;
```

嵌入式 Run 可能在 Provider/Model 重試期間發出暫態 `error` 事件。Registry 延遲 15 秒處理終態錯誤，讓後續的 `start`/`end` 事件有機會取消錯誤宣告。

> **引用來源**：`source-repo/src/agents/subagent-registry.ts:131-136`

### 5.2 宣告機制

Sub-agent 完成後透過「宣告」（Announce）機制通知父 Agent：

```
Sub-agent 完成
  → captureSubagentCompletionReply()  // 捕獲結果
  → runSubagentAnnounceFlow()         // 執行宣告流程
  → 父 Agent 收到完成通知
```

宣告有超時限制：

```typescript
const SUBAGENT_ANNOUNCE_TIMEOUT_MS = 120_000;  // 2 分鐘
const MAX_ANNOUNCE_RETRY_COUNT = ...;           // 最大重試次數
```

> **引用來源**：`source-repo/src/agents/subagent-registry.ts:130`, `subagent-registry-helpers.ts:25-26`

---

## 6. 孤兒回收

當父 Agent 意外結束時，其子 Agent 變成「孤兒」。Registry 提供回收機制：

```typescript
// source-repo/src/agents/subagent-registry.ts:195
export function scheduleSubagentOrphanRecovery(params?: {
  delayMs?: number;
  maxRetries?: number;
})
```

孤兒回收有防抖機制（`ORPHAN_RECOVERY_DEBOUNCE_MS = 1000ms`），避免短時間內重複觸發。

回收流程：
1. 偵測到孤兒 Run（父 Session 不存在或已結束）
2. 等待防抖期
3. 嘗試恢復（重新綁定到新的控制器）
4. 如果無法恢復，則終止並清理

> **引用來源**：`source-repo/src/agents/subagent-registry.ts:195-200`, `subagent-registry-helpers.ts`

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| SpawnSubagentParams | `source-repo/src/agents/subagent-spawn.ts` | 83-103 |
| SpawnSubagentContext | `source-repo/src/agents/subagent-spawn.ts` | 105-117 |
| SpawnSubagentResult | `source-repo/src/agents/subagent-spawn.ts` | 119-133 |
| spawnSubagentDirect() | `source-repo/src/agents/subagent-spawn.ts` | 346 |
| Agent ID 驗證 | `source-repo/src/agents/subagent-spawn.ts` | 358-363 |
| 深度限制 | `source-repo/src/agents/subagent-spawn.ts` | 419-426 |
| 子 Agent 數量限制 | `source-repo/src/agents/subagent-spawn.ts` | 428-435 |
| Session Key 格式 | `source-repo/src/agents/subagent-spawn.ts` | 472 |
| Thread 綁定 | `source-repo/src/agents/subagent-spawn.ts` | 300-344 |
| SubagentRegistryDeps | `source-repo/src/agents/subagent-registry.ts` | 66-80 |
| Registry 查詢 | `source-repo/src/agents/subagent-registry.ts` | 37-47 |
| Sweeper | `source-repo/src/agents/subagent-registry.ts` | 122-123 |
| 錯誤寬限 | `source-repo/src/agents/subagent-registry.ts` | 136 |
| 宣告超時 | `source-repo/src/agents/subagent-registry.ts` | 130 |
| 孤兒回收 | `source-repo/src/agents/subagent-registry.ts` | 195-200 |
