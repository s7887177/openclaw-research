# ReAct 推理迴圈（一）：迴圈結構、工具調用與核准流程

> **摘要**：本章深入 OpenClaw Agent 的核心推理迴圈——ReAct（Reasoning + Acting）模式
> 的實作。我們將追蹤一則使用者訊息從進入系統到 Agent 完成回覆的完整路徑，涵蓋
> 主迴圈結構、工具調用機制、核准流程、以及失敗恢復策略。

---

## 1. ReAct 模式概述

### 1.1 什麼是 ReAct？

ReAct 是一種 AI Agent 推理模式，由 Yao et al. 在 2022 年提出。核心思想是讓 LLM 
交替進行兩種行為：

- **Reasoning（推理/思考）**：分析當前狀態，決定下一步行動
- **Acting（行動）**：執行工具調用或生成回覆

在 OpenClaw 中，這表現為一個 `while (true)` 無限迴圈，不斷重複：
1. **Thought**：建構 Prompt，包含系統指令、對話歷史、工具定義
2. **Action**：發送到 LLM，收到回應（可能包含工具調用）
3. **Observation**：執行工具、處理結果、判斷是否需要繼續

### 1.2 核心檔案

主要 ReAct 迴圈的實作位於：

```
source-repo/src/agents/pi-embedded-runner/run.ts  — 主迴圈
source-repo/src/agents/pi-embedded-runner/run/attempt.ts  — 單次嘗試
source-repo/src/agents/pi-embedded-subscribe.handlers.tools.ts  — 工具處理
source-repo/src/agents/pi-tools.before-tool-call.ts  — before_tool_call hook
```

---

## 2. 主迴圈結構

### 2.1 迴圈入口

主迴圈位於 `run.ts` 的第 573-1803 行，是一個超過 1200 行的 `while (true)` 迴圈。

**關鍵狀態變數**（`source-repo/src/agents/pi-embedded-runner/run.ts:418-444`）：

```typescript
// source-repo/src/agents/pi-embedded-runner/run.ts:418-444
MAX_RUN_LOOP_ITERATIONS  // 最大迭代次數（防止無限迴圈）
runLoopIterations         // 當前迭代計數器（line 429）
overflowCompactionAttempts // 上下文溢出恢復嘗試次數（line 421）
autoCompactionCount       // 自動壓縮次數（line 428）
usageAccumulator          // 跨迭代的 Token 使用量累計
```

### 2.2 迴圈虛擬碼

```typescript
// source-repo/src/agents/pi-embedded-runner/run.ts:573-1803（虛擬碼重建）
while (true) {
  runLoopIterations++;

  // 安全閥：防止無限迴圈
  if (runLoopIterations >= MAX_RUN_LOOP_ITERATIONS) {
    return handleRetryLimitExhaustion();
  }

  // === THOUGHT PHASE ===
  // 建構 prompt：基礎 prompt + 執行快速路徑指令 + 規劃重試指令
  prompt = basePrompt + ackExecutionFastPathInstruction + planningOnlyRetryInstruction;

  // === ACTION PHASE ===
  // 發送到 LLM 並處理回應
  attempt = await runEmbeddedAttemptWithBackend({
    sessionId, sessionKey, prompt,
    provider, modelId, model,
    clientTools, disableTools,
    onPartialReply, onBlockReply, onToolResult, onAgentEvent,
    // ...50+ 其他參數
  });

  // === OBSERVATION PHASE ===
  // 解構結果
  const {
    aborted, externalAbort, promptError,
    timedOut, idleTimedOut,
    lastAssistant, currentAttemptAssistant,
    assistantTexts, toolMetas,
    attemptUsage, compactionCount,
  } = attempt;

  // 成功結束？
  if (currentAttemptAssistant?.stopReason === "end_turn") break;

  // 需要壓縮？（上下文窗口溢出）
  if (needsCompaction) {
    // 執行壓縮並重試
    continue;
  }

  // 需要 Failover？（認證失敗、速率限制等）
  if (needsFailover) {
    // 切換 auth profile 或 fallback model
    continue;
  }
}
```

### 2.3 Attempt 的結果結構

每次「嘗試」返回豐富的結果資料：

```typescript
// source-repo/src/agents/pi-embedded-runner/run.ts:722-751（概要）
const {
  aborted,                    // 是否被中止
  externalAbort,              // 外部中止訊號
  promptError,                // Prompt 建構錯誤
  timedOut,                   // 是否超時
  idleTimedOut,               // 是否因閒置超時
  timedOutDuringCompaction,   // 壓縮期間超時
  lastAssistant,              // 最後的 Assistant 訊息
  currentAttemptAssistant,    // 當前嘗試的 Assistant 訊息
  assistantTexts,             // Assistant 文字回覆陣列
  toolMetas,                  // 工具調用元資料
  attemptUsage,               // Token 使用量
  compactionCount,            // 壓縮次數
  replayMetadata,             // 重播元資料
} = attempt;
```

### 2.4 停止條件

迴圈的退出條件：
1. **`stopReason === "end_turn"`**：Agent 完成回覆，正常退出
2. **`runLoopIterations >= MAX_RUN_LOOP_ITERATIONS`**：達到最大迭代次數
3. **`aborted` 或 `externalAbort`**：被使用者或系統中止
4. **`timedOut`**：超時
5. **無法恢復的錯誤**：認證失敗且無 fallback

迴圈的 **繼續條件**：
1. **`stopReason === "tool_use"`**：需要執行更多工具調用
2. **上下文溢出**：需要壓縮後重試
3. **可恢復的 Failover**：切換 auth profile 或 model

---

## 3. 單次嘗試：Attempt

### 3.1 Attempt 的執行

每次 ReAct 迭代的「Action Phase」由 `runEmbeddedAttemptWithBackend()` 負責：

```typescript
// source-repo/src/agents/pi-embedded-runner/run.ts:630+（呼叫概要）
const attempt = await runEmbeddedAttemptWithBackend({
  sessionId, sessionKey, prompt, provider, modelId, model,
  clientTools, disableTools,
  onPartialReply, onBlockReply, onToolResult, onAgentEvent,
  // ...
});
```

### 3.2 Subscription 模式

Attempt 內部使用 Subscription 模式來處理 LLM 的串流回應：

```typescript
// source-repo/src/agents/pi-embedded-runner/run/attempt.ts:1467（概要）
const subscription = subscribeEmbeddedPiSession(params);
```

Subscription 會產生一系列事件：
- **Partial Reply**：串流回應的片段
- **Tool Call**：LLM 請求調用工具
- **End Turn**：LLM 完成回覆

---

## 4. 工具調用機制

### 4.1 工具調用的觸發

當 LLM 回應中包含工具調用（Tool Call）時，系統需要：
1. 解析工具名稱和參數
2. 執行 `before_tool_call` hook
3. 等待核准（如果需要）
4. 執行工具
5. 執行 `after_tool_call` hook
6. 將結果加入對話歷史
7. 繼續迴圈讓 LLM 處理結果

### 4.2 Tool Definition Adapter

工具在進入 Agent 之前會經過適配器包裝：

```typescript
// source-repo/src/agents/pi-tool-definition-adapter.ts:182-196
execute: async (...args: ToolExecuteArgs): Promise<AgentToolResult<unknown>> => {
  const { toolCallId, params, onUpdate, signal } = splitToolExecuteArgs(args);
  let executeParams = params;

  // before_tool_call hook 攔截
  if (!beforeHookWrapped) {
    const hookOutcome = await runBeforeToolCallHook({
      toolName: name,
      params,
      toolCallId,
    });
    if (hookOutcome.blocked) {
      throw new Error(hookOutcome.reason);  // 阻止執行
    }
    executeParams = hookOutcome.params;  // 使用修改後的參數
  }

  // 實際執行工具
  const rawResult = await tool.execute(toolCallId, executeParams, signal, onUpdate);
  // ...正規化並返回結果
}
```

### 4.3 Tool Payload 處理

工具結果的酬載（Payload）會經過提取和正規化：

```typescript
// source-repo/src/plugin-sdk/tool-payload.ts:1-43
export function extractToolPayload(
  result: ToolPayloadCarrier | null | undefined
): unknown {
  // 提取最有用的資訊：
  // result.details ?? textBlock.text ?? JSON.parse(text) ?? text
}

export type ToolPayloadCarrier = {
  details?: unknown;
  content?: unknown;
};
```

### 4.4 Tool Send（訊息發送工具特殊處理）

某些工具是用來發送訊息的。`extractToolSend()` 從工具參數中提取發送目標：

```typescript
// source-repo/src/plugin-sdk/tool-send.ts:1-25
export function extractToolSend(
  args: Record<string, unknown>,
  expectedAction = "sendMessage"
): { to: string; accountId?: string; threadId?: string } | null
```

---

## 5. Before/After Tool Call Hooks

### 5.1 Before Tool Call：攔截與修改

`before_tool_call` hook 是 OpenClaw 工具系統最重要的擴展點之一。
它允許 Plugin 在工具執行前進行三種操作：

```typescript
// source-repo/src/plugins/hook-types.ts:291-322
export type PluginHookBeforeToolCallEvent = {
  toolName: string;
  params: Record<string, unknown>;
  runId?: string;
  toolCallId?: string;
};

export type PluginHookBeforeToolCallResult = {
  params?: Record<string, unknown>;      // 修改參數
  block?: boolean;                       // 阻止執行
  blockReason?: string;                  // 阻止原因
  requireApproval?: {                    // 要求核准
    title: string;
    description: string;
    severity?: "info" | "warning" | "critical";
    timeoutMs?: number;
    timeoutBehavior?: "allow" | "deny";
    pluginId?: string;
    onResolution?: (decision: PluginApprovalResolution) => Promise<void> | void;
  };
};
```

**三種可能的結果**：
1. **通過**：不修改，繼續執行
2. **修改參數**：返回新的 `params`，用修改後的參數執行
3. **阻止**：返回 `block: true`，工具不會被執行
4. **要求核准**：返回 `requireApproval`，暫停等待使用者決定

### 5.2 Hook 執行邏輯

```typescript
// source-repo/src/plugins/hooks.ts:774-811
async function runBeforeToolCall(
  event: PluginHookBeforeToolCallEvent,
  ctx: PluginHookToolContext
): Promise<PluginHookBeforeToolCallResult | undefined> {
  return runModifyingHook<"before_tool_call", PluginHookBeforeToolCallResult>(
    "before_tool_call",
    event,
    ctx,
    {
      mergeResults: (acc, next, reg) => {
        if (acc?.block === true) return acc;  // block 是 sticky 的
        return {
          params: lastDefined(acc?.params, next.params),
          block: stickyTrue(acc?.block, next.block),
          blockReason: lastDefined(acc?.blockReason, next.blockReason),
          requireApproval: acc?.requireApproval ?? next.requireApproval,
        };
      },
      shouldStop: (result) => result.block === true,
      terminalLabel: "block=true",
    }
  );
}
```

**關鍵語意**：
- `block` 是 **sticky true**——一旦有 Plugin 返回 `block: true`，後續 Plugin 不再執行
- `params` 使用 **last defined**——後執行的 Plugin 覆蓋前面的修改
- `requireApproval` 使用 **first wins**——第一個要求核准的 Plugin 生效

### 5.3 After Tool Call：非同步通知

`after_tool_call` hook 是 fire-and-forget 的——不會阻塞工具執行流程：

```typescript
// source-repo/src/plugins/hooks.ts:814-822
async function runAfterToolCall(
  event: PluginHookAfterToolCallEvent,
  ctx: PluginHookToolContext
): Promise<void> {
  return runVoidHook("after_tool_call", event, ctx);
}
```

```typescript
// source-repo/src/plugins/hook-types.ts:324-332
export type PluginHookAfterToolCallEvent = {
  toolName: string;
  params: Record<string, unknown>;
  runId?: string;
  toolCallId?: string;
  result?: unknown;          // 工具結果
  error?: string;            // 錯誤（如果有）
  durationMs?: number;       // 執行時間
};
```

### 5.4 實際 Hook 觸發

在工具處理器中的實際觸發程式碼：

```typescript
// source-repo/src/agents/pi-embedded-subscribe.handlers.tools.ts:1104-1136
// 在工具執行完成後
if (hookRunnerAfter?.hasHooks("after_tool_call")) {
  void hookRunnerAfter.runAfterToolCall(
    {
      toolName,
      params: tool_params,
      runId: ctx.params.runId,
      toolCallId,
      result: sanitized_result,
      error: error_message,
      durationMs: Date.now() - startedAt,
    },
    {
      runId: ctx.params.runId,
      sessionId: ctx.params.sessionId,
      toolName,
      toolCallId,
    }
  ).catch(err => {
    ctx.log.warn(`after_tool_call hook failed: tool=${toolName} error=${String(err)}`);
  });
}
```

注意 `void` 前綴——結果被忽略，錯誤只記錄不傳播。

---

## 6. Tool Approval：工具核准流程

### 6.1 Execution Policy

OpenClaw 提供了 CLI 命令來管理工具執行策略：

```bash
# source-repo/src/cli/exec-policy-cli.ts
openclaw exec-policy show [--json]       # 顯示當前策略
openclaw exec-policy preset <name>       # 使用預設策略
openclaw exec-policy set --security <level> [--host <host>]  # 設定安全等級
```

### 6.2 Approval Resolution

核准結果有五種可能：

```typescript
// source-repo/src/plugins/hook-types.ts:298-307
export const PluginApprovalResolutions = {
  ALLOW_ONCE: "allow-once",       // 允許這次
  ALLOW_ALWAYS: "allow-always",   // 永遠允許
  DENY: "deny",                   // 拒絕
  TIMEOUT: "timeout",             // 超時
  CANCELLED: "cancelled",         // 取消
} as const;
```

### 6.3 Approval 的 Channel 適配

核准流程與 Channel Adapter 緊密整合——不同平台呈現核准請求的方式不同
（按鈕、鍵盤、文字 prompt 等）。

---

## 7. Context Overflow 與 Compaction

### 7.1 問題：上下文窗口有限

每個 LLM 都有上下文窗口（Context Window）限制。當對話歷史加上工具結果
超過限制時，系統需要進行「壓縮」（Compaction）。

### 7.2 壓縮流程

```typescript
// 在主迴圈的 OBSERVATION PHASE 中
if (needsCompaction) {
  // 1. 執行 before_compaction hook（讓 Plugin 有機會介入）
  // 2. 壓縮對話歷史（保留關鍵資訊，摘要其餘部分）
  // 3. 執行 after_compaction hook
  // 4. 增加 overflowCompactionAttempts
  // 5. continue 回到迴圈頂部重試
}
```

### 7.3 壓縮 Provider

壓縮本身也是一個 Provider 能力，由 `compaction-provider.ts` 管理
（`source-repo/src/plugins/compaction-provider.ts`）。

---

## 8. Failover 機制

### 8.1 當 LLM 請求失敗時

如果 LLM 回應失敗（速率限制、認證過期、服務不可用），迴圈不會立即退出，
而是嘗試 failover：

1. **切換 Auth Profile**：如果有多個認證（例如多個 API Key），嘗試下一個
2. **切換 Model**：如果設定了 fallback model，嘗試使用它
3. **等待並重試**：對於速率限制，等待後重試

### 8.2 重試限制

`MAX_RUN_LOOP_ITERATIONS` 防止無限重試。當達到限制時，
系統會返回錯誤訊息並記錄 failover 歷史。

---

## 9. 完整工具調用流程圖

```
LLM Response
    │
    ├─ 包含 Tool Call？
    │  ├─ YES ──────────────────────────────────────────┐
    │  │                                                 │
    │  │  ┌─────────────────────────┐                   │
    │  │  │ 1. before_tool_call     │                   │
    │  │  │    hook 執行             │                   │
    │  │  └────────┬────────────────┘                   │
    │  │           │                                     │
    │  │  ┌────────▼────────────────┐                   │
    │  │  │ 結果判斷                 │                   │
    │  │  ├─ block=true → 返回錯誤   │                   │
    │  │  ├─ requireApproval        │                   │
    │  │  │  ├─ 等待使用者回應        │                   │
    │  │  │  ├─ ALLOW → 繼續         │                   │
    │  │  │  ├─ DENY → 返回錯誤      │                   │
    │  │  │  └─ TIMEOUT → 依配置決定  │                   │
    │  │  └─ 通過 → 繼續             │                   │
    │  │           │                                     │
    │  │  ┌────────▼────────────────┐                   │
    │  │  │ 2. 執行工具              │                   │
    │  │  │    tool.execute(params)  │                   │
    │  │  └────────┬────────────────┘                   │
    │  │           │                                     │
    │  │  ┌────────▼────────────────┐                   │
    │  │  │ 3. after_tool_call      │                   │
    │  │  │    hook（fire-n-forget） │                   │
    │  │  └────────┬────────────────┘                   │
    │  │           │                                     │
    │  │  ┌────────▼────────────────┐                   │
    │  │  │ 4. 結果加入對話歷史      │                   │
    │  │  │    → 繼續 ReAct 迴圈     │                   │
    │  │  └─────────────────────────┘                   │
    │  │                                                 │
    │  └─────────────────────────────────────────────────┘
    │
    ├─ stopReason === "end_turn"？
    │  └─ YES → 結束迴圈，返回最終回覆
    │
    └─ 其他 → 檢查 failover/compaction
```

---

## 引用來源

| 來源檔案 | 行號 | 內容 |
|---------|------|------|
| `source-repo/src/agents/pi-embedded-runner/run.ts` | 418-444 | 迴圈狀態變數 |
| `source-repo/src/agents/pi-embedded-runner/run.ts` | 573-1803 | 主 ReAct 迴圈 |
| `source-repo/src/agents/pi-embedded-runner/run.ts` | 722-751 | Attempt 結果解構 |
| `source-repo/src/agents/pi-embedded-runner/run/attempt.ts` | 1467 | Subscription 建立 |
| `source-repo/src/agents/pi-tool-definition-adapter.ts` | 182-196 | 工具適配器 |
| `source-repo/src/agents/pi-embedded-subscribe.handlers.tools.ts` | 1104-1136 | After hook 觸發 |
| `source-repo/src/agents/pi-tools.before-tool-call.ts` | 1-463 | Before hook 實作 |
| `source-repo/src/plugins/hooks.ts` | 774-822 | Hook runner |
| `source-repo/src/plugins/hook-types.ts` | 282-332 | Tool hook 型別 |
| `source-repo/src/plugins/hook-types.ts` | 298-307 | `PluginApprovalResolutions` |
| `source-repo/src/plugin-sdk/tool-payload.ts` | 1-43 | Tool payload 提取 |
| `source-repo/src/plugin-sdk/tool-send.ts` | 1-25 | Tool send 提取 |
| `source-repo/src/cli/exec-policy-cli.ts` | — | Exec policy CLI |
| `source-repo/src/plugins/compaction-provider.ts` | — | 壓縮 Provider |
