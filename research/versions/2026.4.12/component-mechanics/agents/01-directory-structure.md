# Agents 子章節 01：目錄結構與入口

> **引用範圍**：`src/agents/agent-command.ts`, `agent-scope.ts`, `timeout.ts`

## 1. 模組組織

`src/agents/` 是 OpenClaw 最大的程式碼模組，包含超過 650 個 TypeScript 檔案。模組以**扁平結構**為主，核心邏輯集中在根目錄，子目錄負責特定功能領域。

### 1.1 根目錄核心檔案

根目錄約 782 個檔案，以下為最重要的檔案分類：

**指令入口**：
- `agent-command.ts`（1,026 行）— Agent 執行的主入口
- `agent-scope.ts`（371 行）— Agent 配置解析 Hub
- `call.ts` / `call.runtime.ts` — Agent 呼叫邏輯

**Sub-agent 系統**：
- `subagent-spawn.ts`（907 行）— 生成子 Agent
- `subagent-registry.ts`（933 行）— 生命週期追蹤
- `subagent-registry.types.ts` — 執行記錄型別
- `subagent-spawn-plan.ts` — 模型與 Thinking 解析
- `subagent-registry-completion.ts` — 完成回調

**工具系統**：
- `openclaw-tools.ts` — 工具工廠（建立 30+ 核心工具）
- `openclaw-tools.registration.ts` — 工具註冊策略
- `tool-display-config.ts`（701 行）— 工具顯示設定

**記憶與模型**：
- `memory-search.ts` — 記憶體搜尋配置解析
- `model-selection.ts`（883 行）— 模型選擇與回退

**Prompt 系統**：
- `agent-prompt.ts` — Agent 系統提示建構

### 1.2 子目錄概覽

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `pi-embedded-runner/` | 122 | **嵌入式 Agent 執行器**：在獨立程序中執行 Agent，管理 stdin/stdout 通訊、記憶體限制、沙箱隔離 |
| `tools/` | 110 | **核心工具實作**：每個工具一個檔案，包含 web search、web fetch、image gen、code exec 等 |
| `sandbox/` | 71 | **沙箱系統**：Docker 容器沙箱、檔案系統隔離、網路限制 |
| `auth-profiles/` | 32 | **認證 Profile**：管理不同 LLM 提供者的認證配置 |
| `skills/` | 27 | **技能系統**：工作區技能探索、YAML metadata 解析、技能註冊 |
| `pi-embedded-helpers/` | 16 | **嵌入式輔助函式**：程序間通訊、資源管理 |
| `command/` | 12 | **指令管線**：Agent 指令解析與路由 |
| `cli-runner/` | 10 | **CLI 後端**：提供 CLI 客戶端的 Agent 執行能力 |
| `pi-hooks/` | 9 | **生命週期 Hooks**：Agent 啟動/結束/錯誤等事件 |
| `harness/` | 7 | **測試 Harness**：Agent 整合測試工具 |
| `schema/` | 4 | **Schema 工具**：配置 schema 清理與驗證 |

---

## 2. 主入口：agentCommand()

`agent-command.ts` 是所有 Agent 執行的起點：

```typescript
// source-repo/src/agents/agent-command.ts:979 (概要)
export async function agentCommand(params: {
  cfg: OpenClawConfig;
  sessionKey: string;
  prompt: string;
  storePath: string;
  // ... 更多參數
}): Promise<AgentCommandResult>
```

此函式負責：
1. 解析 Agent ID 與配置
2. 載入 Session 歷史
3. 建立 Agent 工具集
4. 呼叫 LLM 並管理回應串流
5. 持久化結果到 Session Store
6. 觸發生命週期事件

### 2.1 Ingress 變體

```typescript
// source-repo/src/agents/agent-command.ts:999 (概要)
export async function agentCommandFromIngress(params: {
  // ... agentCommand 的參數 +
  ingressContext: IngressContext;
}): Promise<AgentCommandResult>
```

`agentCommandFromIngress()` 是為外部入口（如 Channel Adapter）設計的變體，附加了入口上下文。

> **引用來源**：`source-repo/src/agents/agent-command.ts:979-1023`

---

## 3. Agent 配置解析 Hub：agent-scope.ts

`agent-scope.ts` 是 Agent 配置的**中央解析器**，提供所有與 Agent 身份、配置、工作區相關的查詢函式。

### 3.1 列出 Agent

```typescript
// source-repo/src/agents/agent-scope.ts:66
export function listAgentEntries(cfg: OpenClawConfig): AgentEntry[]
// 從 cfg.agents?.list[] 取得所有合法 Agent 項目

// source-repo/src/agents/agent-scope.ts:74
export function listAgentIds(cfg: OpenClawConfig): string[]
// 回傳正規化後的 Agent ID 列表
```

### 3.2 解析預設 Agent

```typescript
// source-repo/src/agents/agent-scope.ts:92
export function resolveDefaultAgentId(cfg: OpenClawConfig): string
// 找到第一個 default=true 的 Agent，或第一個 Agent
```

### 3.3 解析 Agent 配置

```typescript
// source-repo/src/agents/agent-scope.ts:137
export function resolveAgentConfig(
  cfg: OpenClawConfig,
  agentId: string
): ResolvedAgentConfig
```

回傳的 `ResolvedAgentConfig` 包含：
- `name`, `workspace`, `agentDir` — 基本身份
- `systemPromptOverride` — 自訂系統提示
- `model` — 模型配置
- `thinkingDefault`, `verboseDefault`, `reasoningDefault`, `fastModeDefault` — 行為預設
- `skills` — 技能白名單
- `memorySearch` — 記憶體搜尋配置
- `humanDelay`, `heartbeat`, `identity`, `groupChat` — 互動設定
- `subagents` — Sub-agent 配置
- `embeddedPi`, `sandbox`, `tools` — 執行時配置

> **引用來源**：`source-repo/src/agents/agent-scope.ts:66-137`

### 3.4 模型解析

```typescript
// source-repo/src/agents/agent-scope.ts:193
export function resolveAgentExplicitModelPrimary(
  cfg: OpenClawConfig,
  agentId: string
): string | undefined
// Agent 明確指定的主模型

// source-repo/src/agents/agent-scope.ts:201
export function resolveAgentEffectiveModelPrimary(
  cfg: OpenClawConfig,
  agentId: string
): string | undefined
// Agent 的有效主模型（含 fallback）

// source-repo/src/agents/agent-scope.ts:216
export function resolveAgentModelFallbacksOverride(
  cfg: OpenClawConfig,
  agentId: string
): string[] | undefined
// Agent 的 fallback 模型列表
```

> **引用來源**：`source-repo/src/agents/agent-scope.ts:193-216`

### 3.5 工作區解析

```typescript
// source-repo/src/agents/agent-scope.ts:279
export function resolveAgentWorkspaceDir(
  cfg: OpenClawConfig,
  agentId: string
): string
// 建構工作區路徑：
// - 預設 Agent → ~/.openclaw/workspace
// - 非預設 Agent → ~/.openclaw/workspace-{agentId}

// source-repo/src/agents/agent-scope.ts:323
export function resolveAgentIdsByWorkspacePath(
  cfg: OpenClawConfig,
  workspacePath: string
): string[]
// 反向查詢：根據工作區路徑找到對應的 Agent ID
```

> **引用來源**：`source-repo/src/agents/agent-scope.ts:279-323`

### 3.6 技能過濾

```typescript
// source-repo/src/agents/agent-scope.ts:186
export function resolveAgentSkillsFilter(
  cfg: OpenClawConfig,
  agentId: string
): string[] | undefined
// 回傳 Agent 的技能白名單
// 如果未設定，繼承 agents.defaults.skills
```

> **引用來源**：`source-repo/src/agents/agent-scope.ts:186`

---

## 4. 逾時管理：timeout.ts

```typescript
// source-repo/src/agents/timeout.ts:1-48 (完整)
const DEFAULT_AGENT_TIMEOUT_SECONDS = 48 * 60 * 60;  // 172,800 秒 = 2 天
const MAX_SAFE_TIMEOUT_MS = 2_147_000_000;            // ~24.8 天（32-bit 整數上限）
```

### 4.1 秒級解析

```typescript
// source-repo/src/agents/timeout.ts:9-13
export function resolveAgentTimeoutSeconds(cfg?: OpenClawConfig): number {
  const raw = normalizeNumber(cfg?.agents?.defaults?.timeoutSeconds);
  const seconds = raw ?? DEFAULT_AGENT_TIMEOUT_SECONDS;
  return Math.max(seconds, 1);  // 最少 1 秒
}
```

### 4.2 毫秒級解析（支援覆寫）

```typescript
// source-repo/src/agents/timeout.ts:15-48
export function resolveAgentTimeoutMs(opts: {
  cfg?: OpenClawConfig;
  overrideMs?: number | null;     // 毫秒覆寫
  overrideSeconds?: number | null; // 秒覆寫
  minMs?: number;                  // 最小值
}): number
```

**特殊值處理**：
- `0` → `NO_TIMEOUT_MS`（`MAX_SAFE_TIMEOUT_MS`）= 無限期
- 負數 → 使用預設值
- 正數 → clamp 到 `[minMs, MAX_SAFE_TIMEOUT_MS]`

> **引用來源**：`source-repo/src/agents/timeout.ts:1-48`

---

## 5. 配置載入鏈

Agent 的配置解析遵循以下載入鏈：

```
1. 從磁碟載入 OpenClawConfig（src/config/config.js）
   │
2. 存取 cfg.agents.list[]（AgentConfig 陣列）
   │
3. 正規化 Agent ID（routing/session-key.js 的 normalizeAgentId()）
   │
4. 透過 resolveAgentConfig() 解析個別 Agent 配置
   │
5. 合併 agents.defaults 與 Agent 特定配置
   │
6. 解析工作區目錄
   │
7. 解析技能過濾器
   │
8. 最終配置供 agentCommand() 使用
```

> **引用來源**：`source-repo/src/agents/agent-scope.ts`（整體設計模式）

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| agentCommand() | `source-repo/src/agents/agent-command.ts` | 979 |
| agentCommandFromIngress() | `source-repo/src/agents/agent-command.ts` | 999 |
| listAgentEntries() | `source-repo/src/agents/agent-scope.ts` | 66 |
| listAgentIds() | `source-repo/src/agents/agent-scope.ts` | 74 |
| resolveDefaultAgentId() | `source-repo/src/agents/agent-scope.ts` | 92 |
| resolveAgentConfig() | `source-repo/src/agents/agent-scope.ts` | 137 |
| 模型解析函式 | `source-repo/src/agents/agent-scope.ts` | 193-216 |
| 工作區解析 | `source-repo/src/agents/agent-scope.ts` | 279, 323 |
| 技能過濾 | `source-repo/src/agents/agent-scope.ts` | 186 |
| DEFAULT_AGENT_TIMEOUT_SECONDS | `source-repo/src/agents/timeout.ts` | 3 |
| MAX_SAFE_TIMEOUT_MS | `source-repo/src/agents/timeout.ts` | 4 |
| resolveAgentTimeoutSeconds() | `source-repo/src/agents/timeout.ts` | 9-13 |
| resolveAgentTimeoutMs() | `source-repo/src/agents/timeout.ts` | 15-48 |
