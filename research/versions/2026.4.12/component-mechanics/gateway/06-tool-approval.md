# Gateway 子章節 06：Tool 調用與核准工作流

> **引用範圍**：`src/gateway/tools-invoke-http.ts`, `tool-resolution.ts`, `exec-approval-manager.ts`, `node-invoke-system-run-approval.ts`

## 1. HTTP Tool 調用端點

Gateway 提供 HTTP 端點讓外部系統直接調用 Agent 工具：

```
POST /tools/invoke
```

### 1.1 處理器

```typescript
// source-repo/src/gateway/tools-invoke-http.ts:130-150+ (概要)
export async function handleToolsInvokeHttpRequest(params: {
  req: http.IncomingMessage;
  res: http.ServerResponse;
  resolvedAuth: ResolvedGatewayAuth;
  // ...
}): Promise<void>
```

### 1.2 常數

```typescript
// source-repo/src/gateway/tools-invoke-http.ts:34-35
const DEFAULT_BODY_BYTES = 2 * 1024 * 1024;  // 2 MB — 請求 body 上限
const MEMORY_TOOL_NAMES = new Set(["memory_search", "memory_get"]);
```

`MEMORY_TOOL_NAMES` 標記了記憶體相關工具，這些工具在某些測試場景下可被停用。

### 1.3 處理流程

1. 驗證認證
2. 解析 JSON body（最大 2 MB）
3. 驗證 Session Key 和工具名稱
4. 呼叫 `resolveGatewayScopedTools()` 取得可用工具
5. 查找請求的工具
6. 如果是 dry-run 模式 → 回傳工具定義
7. 否則 → 執行工具並回傳結果

### 1.4 Action 合併

```typescript
// source-repo/src/gateway/tools-invoke-http.ts:75-99
export function mergeActionIntoArgsIfSupported(params: {
  toolName: string;
  action?: string;
  args: Record<string, unknown>;
}): Record<string, unknown>
```

將 `action` 參數合併到工具呼叫的 `args` 中，用於支援工具的多動作模式。

### 1.5 記憶體工具停用檢查

```typescript
// source-repo/src/gateway/tools-invoke-http.ts:52-73
export function resolveMemoryToolDisableReasons(params: {
  toolName: string;
  cfg: OpenClawConfig;
}): string[] | null
```

檢查記憶體工具是否在測試環境中被停用，回傳停用原因列表。

> **引用來源**：`source-repo/src/gateway/tools-invoke-http.ts:34-150+`

---

## 2. 工具範圍解析（Tool Scoping）

`tool-resolution.ts` 決定一個 Session 可以使用哪些工具：

```typescript
// source-repo/src/gateway/tool-resolution.ts:26-39 (概要)
export function resolveGatewayScopedTools(params: {
  cfg: OpenClawConfig;
  sessionKey: string;
  messageProvider?: string;
  accountId?: string;
  agentTo?: string;
  agentThreadId?: string | number;
}): ResolvedTool[]
```

**工具範圍解析層級**：

```
1. 全域工具目錄（tools.catalog）
   │
2. Agent 有效工具（tools.effective）
   │     ↓ 過濾
3. Profile 工具策略
   │     ↓ 過濾
4. Group 工具策略
   │     ↓ 過濾
5. Sub-agent 工具策略
   │     ↓ 過濾
6. HTTP 表面限制
   │
   └→ 最終可用工具列表
```

每一層都可以縮減工具範圍，確保不同角色和上下文只能存取被授權的工具。

> **引用來源**：`source-repo/src/gateway/tool-resolution.ts:26-39`

---

## 3. 執行核准管理器（Exec Approval Manager）

當 Agent 嘗試執行敏感操作（如系統命令）時，需要操作者核准。這由 `ExecApprovalManager` 管理：

### 3.1 核准記錄

```typescript
// source-repo/src/gateway/exec-approval-manager.ts:13-25
type ExecApprovalRecord<TPayload> = {
  id: string;                      // 核准記錄 ID
  request: TPayload;               // 核准請求內容
  createdAtMs: number;             // 建立時間
  expiresAtMs: number;             // 過期時間
  requestedByConnId: string;       // 請求者連線 ID
  requestedByDeviceId: string;     // 請求者裝置 ID
  requestedByClientId: string;     // 請求者客戶端 ID
  resolvedAtMs?: number;           // 解決時間
  decision?: string;               // 決定（approve/reject）
  resolvedBy?: string;             // 解決者
};
```

### 3.2 寬限期

```typescript
// source-repo/src/gateway/exec-approval-manager.ts:9
const RESOLVED_ENTRY_GRACE_MS = 15_000;  // 15 秒 — 已解決條目的保留寬限期
```

已解決的核准記錄會保留 15 秒，以處理延遲到達的決定。

### 3.3 Manager 方法

```typescript
// source-repo/src/gateway/exec-approval-manager.ts:40+ (概要)
export class ExecApprovalManager<TPayload> {
  create(params: {
    request: TPayload;
    timeoutMs: number;
    requestedByConnId: string;
    requestedByDeviceId: string;
    requestedByClientId: string;
  }): ExecApprovalRecord<TPayload>;                    // 建立核准記錄

  register(id: string): Promise<ExecApprovalDecision>;  // 註冊並等待決定

  resolve(params: {
    id: string;
    decision: string;
    resolvedBy: string;
  }): boolean;                                          // 解決核准

  expire(id: string): void;                              // 過期處理
}
```

**核准工作流**：

```
Agent 要執行命令
    │
    ├→ create() — 建立核准記錄
    ├→ 推送 exec.approval.requested 事件給客戶端
    ├→ register() — 開始等待決定（Promise）
    │
    │  操作者在 Control UI 或 CLI 看到核准請求
    │     │
    │     ├→ 核准 → resolve(decision: "approve")
    │     └→ 拒絕 → resolve(decision: "reject")
    │
    ├→ Promise 解決 → Agent 繼續或中止
    └→ 超時 → expire() → Agent 收到超時錯誤
```

> **引用來源**：`source-repo/src/gateway/exec-approval-manager.ts:9-104+`

---

## 4. 系統命令核准（System Run Approval）

針對 `system.run`（作業系統命令執行）的特殊核准邏輯：

```typescript
// source-repo/src/gateway/node-invoke-system-run-approval.ts:83-91+ (概要)
export function sanitizeSystemRunParamsForForwarding(params: {
  client: GatewayClient;
  approvalRecord: ExecApprovalRecord<SystemRunPayload>;
  decision: string;
}): { sanitized: true } | { sanitized: false; reason: string }
```

### 4.1 權限檢查

```typescript
// source-repo/src/gateway/node-invoke-system-run-approval.ts:49-52
function clientHasApprovals(client: GatewayClient): boolean {
  // 檢查客戶端是否有 operator.approvals 或 operator.admin 範圍
}
```

只有具備 `operator.approvals` 或 `operator.admin` scope 的客戶端才能核准系統命令。

### 4.2 決定正規化

```typescript
// source-repo/src/gateway/node-invoke-system-run-approval.ts:44-46
function normalizeApprovalDecision(raw: string): "approve" | "reject" | null
```

確保核准決定只能是 `approve` 或 `reject`，防止注入異常值。

### 4.3 核准匹配

`node-invoke-system-run-approval-match.ts` 處理核准請求與待執行命令的匹配邏輯，確保核准的是**確切被請求的命令**，防止核准被重播到不同命令（核准滑脫攻擊）。

> **引用來源**：`source-repo/src/gateway/node-invoke-system-run-approval.ts:44-91+`, `source-repo/src/gateway/node-invoke-system-run-approval-match.ts`

---

## 5. WS 核准方法

| 方法 | 說明 |
|------|------|
| `exec.approvals.get` | 取得全域核准設定 |
| `exec.approvals.set` | 設定全域核准策略 |
| `exec.approvals.node.get` | 取得節點級核准設定 |
| `exec.approvals.node.set` | 設定節點級核准策略 |
| `exec.approval.get` | 取得特定核准記錄 |
| `exec.approval.list` | 列出待處理核准 |
| `exec.approval.request` | 發起核准請求 |
| `exec.approval.waitDecision` | 等待核准決定 |
| `exec.approval.resolve` | 解決核准（核准/拒絕） |
| `plugin.approval.list` | 列出 Plugin 核准 |
| `plugin.approval.request` | Plugin 核准請求 |
| `plugin.approval.waitDecision` | 等待 Plugin 核准 |
| `plugin.approval.resolve` | 解決 Plugin 核准 |

> **引用來源**：`source-repo/src/gateway/server-methods-list.ts:31-43`

---

## 6. WS 核准事件

| 事件 | 觸發時機 |
|------|---------|
| `exec.approval.requested` | Agent 請求執行核准 |
| `exec.approval.resolved` | 核准被解決（核准/拒絕/超時） |
| `plugin.approval.requested` | Plugin 請求核准 |
| `plugin.approval.resolved` | Plugin 核准被解決 |

> **引用來源**：`source-repo/src/gateway/server-methods-list.ts:161-164`

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| Tool 調用端點常數 | `source-repo/src/gateway/tools-invoke-http.ts` | 34-35 |
| Tool 調用處理器 | `source-repo/src/gateway/tools-invoke-http.ts` | 130-150+ |
| Action 合併 | `source-repo/src/gateway/tools-invoke-http.ts` | 75-99 |
| 記憶體工具停用 | `source-repo/src/gateway/tools-invoke-http.ts` | 52-73 |
| 工具範圍解析 | `source-repo/src/gateway/tool-resolution.ts` | 26-39 |
| ExecApprovalRecord | `source-repo/src/gateway/exec-approval-manager.ts` | 13-25 |
| RESOLVED_ENTRY_GRACE_MS | `source-repo/src/gateway/exec-approval-manager.ts` | 9 |
| ExecApprovalManager 方法 | `source-repo/src/gateway/exec-approval-manager.ts` | 40-104+ |
| 系統命令核准 | `source-repo/src/gateway/node-invoke-system-run-approval.ts` | 44-91+ |
| 核准 WS 方法 | `source-repo/src/gateway/server-methods-list.ts` | 31-43 |
| 核准 WS 事件 | `source-repo/src/gateway/server-methods-list.ts` | 161-164 |
