# Agents 子章節 02：Agent 配置與 Schema

> **引用範圍**：`src/config/types.agents.ts`, `src/config/types.agent-defaults.ts`, `src/config/agent-limits.ts`

## 1. 配置層次結構

OpenClaw 的 Agent 配置採用**兩層合併策略**：

```
AgentsConfig
  ├── defaults: AgentDefaultsConfig     // 全局預設（所有 Agent 共享）
  └── list: AgentConfig[]               // 個別 Agent 覆寫
```

當解析特定 Agent 的配置時，個別 Agent 的設定**覆蓋** `defaults`；未設定的欄位則**繼承**全局預設。

> **引用來源**：`source-repo/src/config/types.agents.ts:115-118`

---

## 2. AgentConfig 型別（個別 Agent）

```typescript
// source-repo/src/config/types.agents.ts:65-113
export type AgentConfig = {
  id: string;                        // 唯一識別碼
  default?: boolean;                 // 是否為預設 Agent
  name?: string;                     // 顯示名稱
  workspace?: string;                // 工作區路徑
  agentDir?: string;                 // Agent 配置目錄
  systemPromptOverride?: string;     // 系統提示覆寫
  embeddedHarness?: AgentEmbeddedHarnessConfig;
  model?: AgentModelConfig;          // 模型配置
  thinkingDefault?: "off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "adaptive";
  verboseDefault?: "off" | "on" | "full";
  reasoningDefault?: "on" | "off" | "stream";
  fastModeDefault?: boolean;
  skills?: string[];                 // 技能白名單
  memorySearch?: MemorySearchConfig;
  humanDelay?: HumanDelayConfig;     // 人性化延遲
  heartbeat?: AgentDefaultsConfig["heartbeat"];
  identity?: IdentityConfig;         // 人格身份
  groupChat?: GroupChatConfig;       // 群聊行為
  subagents?: { ... };               // Sub-agent 設定
  embeddedPi?: { ... };              // 嵌入式 Pi 設定
  sandbox?: AgentSandboxConfig;      // 沙箱設定
  params?: Record<string, unknown>;  // Provider 參數
  tools?: AgentToolsConfig;          // 工具設定
  runtime?: AgentRuntimeConfig;      // 執行時描述
};
```

### 2.1 Thinking Level 七級

Agent 的「思考深度」有 7 個等級，控制 LLM 的推理行為：

| 等級 | 用途 |
|------|------|
| `off` | 關閉思考 |
| `minimal` | 最小思考 |
| `low` | 低度思考 |
| `medium` | 中度思考 |
| `high` | 高度思考 |
| `xhigh` | 極高思考 |
| `adaptive` | 自適應（由 LLM 決定） |

> **引用來源**：`source-repo/src/config/types.agents.ts:77`

### 2.2 Sub-agent 配置

```typescript
// source-repo/src/config/types.agents.ts:93-100
subagents?: {
  allowAgents?: string[];       // 允許生成的 Agent ID 白名單（"*" = 任意）
  model?: AgentModelConfig;     // Sub-agent 預設模型
  requireAgentId?: boolean;     // 強制要求明確指定 agentId
};
```

這些設定控制 Agent 間的委派權限。`allowAgents` 是安全關鍵設定，決定 Agent 能否將任務委派給其他 Agent。

> **引用來源**：`source-repo/src/config/types.agents.ts:93-100`

### 2.3 Runtime 型別

```typescript
// source-repo/src/config/types.agents.ts:23-30
export type AgentRuntimeConfig =
  | { type: "embedded" }            // 嵌入式（同程序）
  | { type: "acp";                  // ACP 協定（外部程序）
      acp?: AgentRuntimeAcpConfig;
    };
```

`embedded` 是預設模式，Agent 在 OpenClaw 主程序內執行。`acp` 模式透過 Agent Communication Protocol 與外部 Agent 程序通訊。

> **引用來源**：`source-repo/src/config/types.agents.ts:23-30`

### 2.4 路由綁定

```typescript
// source-repo/src/config/types.agents.ts:32-63
export type AgentBindingMatch = {
  channel: string;          // 通道 ID
  accountId?: string;       // 帳號 ID
  peer?: { kind: ChatType; id: string };  // 對話對象
  guildId?: string;         // Discord Guild
  teamId?: string;          // Slack Team
  roles?: string[];         // Discord 角色
};

export type AgentBinding = AgentRouteBinding | AgentAcpBinding;
```

綁定系統允許根據**來源通道**、**對象**、**角色**等條件，自動路由到不同 Agent。這是多 Agent 部署的核心機制。

> **引用來源**：`source-repo/src/config/types.agents.ts:32-63`

---

## 3. AgentDefaultsConfig 型別（全局預設）

`AgentDefaultsConfig` 包含所有 Agent 共享的預設設定，約 50+ 個欄位。以下是關鍵分類：

### 3.1 模型配置

```typescript
// source-repo/src/config/types.agent-defaults.ts:149-178
export type AgentDefaultsConfig = {
  params?: Record<string, unknown>;           // 全局 Provider 參數
  model?: AgentModelConfig;                   // 主模型 + fallbacks
  imageModel?: AgentModelConfig;              // 圖片理解模型
  imageGenerationModel?: AgentModelConfig;    // 圖片生成模型
  videoGenerationModel?: AgentModelConfig;    // 影片生成模型
  musicGenerationModel?: AgentModelConfig;    // 音樂生成模型
  pdfModel?: AgentModelConfig;                // PDF 處理模型
  mediaGenerationAutoProviderFallback?: boolean;  // 跨 Provider 自動 fallback
  models?: Record<string, AgentModelEntryConfig>; // 模型目錄
  // ...
};
```

注意模型系統支援**多種媒體專用模型**，每種都有獨立的 primary/fallbacks 配置。

> **引用來源**：`source-repo/src/config/types.agent-defaults.ts:149-178`

### 3.2 Bootstrap 與 Context 注入

```typescript
// source-repo/src/config/types.agent-defaults.ts:187-207
{
  skipBootstrap?: boolean;                    // 跳過 BOOTSTRAP.MD
  contextInjection?: "always" | "continuation-skip";
  bootstrapMaxChars?: number;                 // 預設 20,000
  bootstrapTotalMaxChars?: number;            // 預設 150,000
  bootstrapPromptTruncationWarning?: "off" | "once" | "always";
}
```

### 3.3 Startup Context

```typescript
// source-repo/src/config/types.agent-defaults.ts:53-66
export type AgentStartupContextConfig = {
  enabled?: boolean;               // 預設 true
  applyOn?: Array<"new" | "reset">;  // 預設 ["new", "reset"]
  dailyMemoryDays?: number;        // 預設 2
  maxFileBytes?: number;           // 預設 16,384
  maxFileChars?: number;           // 預設 2,000
  maxTotalChars?: number;          // 預設 4,500
};
```

啟動上下文在裸 `/new` 或 `/reset` 時自動注入最近的記憶摘要。

> **引用來源**：`source-repo/src/config/types.agent-defaults.ts:53-66`

### 3.4 Context Pruning

```typescript
// source-repo/src/config/types.agent-defaults.ts:30-51
export type AgentContextPruningConfig = {
  mode?: "off" | "cache-ttl";
  ttl?: string;                    // 持續時間字串，預設單位：分鐘
  keepLastAssistants?: number;
  softTrimRatio?: number;
  hardClearRatio?: number;
  minPrunableToolChars?: number;
  tools?: { allow?: string[]; deny?: string[] };
  softTrim?: { maxChars?: number; headChars?: number; tailChars?: number };
  hardClear?: { enabled?: boolean; placeholder?: string };
};
```

Context Pruning 在長對話中自動修剪舊的工具呼叫結果，減少 token 消耗。

> **引用來源**：`source-repo/src/config/types.agent-defaults.ts:30-51`

### 3.5 CLI Backend

```typescript
// source-repo/src/config/types.agent-defaults.ts:68-147
export type CliBackendConfig = {
  command: string;                 // CLI 指令
  args?: string[];                 // 基本參數
  output?: "json" | "text" | "jsonl";   // 輸出解析模式
  input?: "arg" | "stdin";         // Prompt 傳入方式
  modelArg?: string;               // 模型參數 flag
  modelAliases?: Record<string, string>;  // 模型別名
  sessionArg?: string;             // Session ID flag
  serialize?: boolean;             // 串行化執行
  reliability?: { ... };           // 可靠性微調
};
```

CLI Backend 允許將外部 CLI 工具（如 claude-cli）作為 Agent 的替代後端。這是一個極具彈性的架構，支援 JSON/text/JSONL 三種輸出格式。

> **引用來源**：`source-repo/src/config/types.agent-defaults.ts:68-147`

---

## 4. 並行與逾時限制

### 4.1 硬性限制常數

```typescript
// source-repo/src/config/agent-limits.ts:1-23 (完整)
export const DEFAULT_AGENT_MAX_CONCURRENT = 4;
export const DEFAULT_SUBAGENT_MAX_CONCURRENT = 8;
export const DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH = 1;
```

| 常數 | 值 | 含義 |
|------|-----|------|
| `DEFAULT_AGENT_MAX_CONCURRENT` | 4 | 同時執行的 Agent 上限 |
| `DEFAULT_SUBAGENT_MAX_CONCURRENT` | 8 | 同時執行的 Sub-agent 上限 |
| `DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH` | 1 | Sub-agent 巢狀深度上限 |

`MAX_SPAWN_DEPTH=1` 表示預設情況下 Sub-agent **不能再生成 Sub-agent**。這是一個安全護欄，防止遞迴生成失控。

> **引用來源**：`source-repo/src/config/agent-limits.ts:3-6`

### 4.2 逾時配置

```typescript
// source-repo/src/agents/timeout.ts:3-4
const DEFAULT_AGENT_TIMEOUT_SECONDS = 48 * 60 * 60;   // 172,800 秒 = 2 天
const MAX_SAFE_TIMEOUT_MS = 2_147_000_000;              // ~24.8 天
```

逾時值 `0` 被解釋為**無限期**（使用 `MAX_SAFE_TIMEOUT_MS`），這是為了長期運行的 Agent 場景（如持續監控、定期 cron 任務）。

> **引用來源**：`source-repo/src/agents/timeout.ts:3-4`

### 4.3 Sub-agent Registry 常數

```typescript
// source-repo/src/agents/subagent-registry.ts:128-140
const ORPHAN_RECOVERY_DEBOUNCE_MS = 1_000;
const SUBAGENT_ANNOUNCE_TIMEOUT_MS = 120_000;   // 2 分鐘
const LIFECYCLE_ERROR_RETRY_GRACE_MS = 15_000;  // 15 秒
const SESSION_RUN_TTL_MS = 5 * 60_000;          // 5 分鐘
const PENDING_ERROR_TTL_MS = 5 * 60_000;        // 5 分鐘
```

| 常數 | 值 | 用途 |
|------|-----|------|
| `SUBAGENT_ANNOUNCE_TIMEOUT_MS` | 120 秒 | Sub-agent 宣告回應超時 |
| `LIFECYCLE_ERROR_RETRY_GRACE_MS` | 15 秒 | 暫態錯誤重試寬限期 |
| `SESSION_RUN_TTL_MS` | 5 分鐘 | Session-mode 執行清理 TTL |
| `ORPHAN_RECOVERY_DEBOUNCE_MS` | 1 秒 | 孤兒回收防抖 |

> **引用來源**：`source-repo/src/agents/subagent-registry.ts:128-140`

---

## 5. Zod 驗證

OpenClaw 使用 Zod 進行配置驗證。雖然 TypeScript 型別定義在 `types.agents.ts`，但執行時期驗證由 Zod schema 負責。驗證鏈為：

```
YAML 檔案 → JSON 解析 → Zod schema 驗證 → TypeScript 型別
```

Zod schema 確保：
- `id` 欄位匹配 `[a-z0-9][a-z0-9_-]{0,63}` 格式
- `thinkingDefault` 僅接受七個合法值
- 數值欄位有合理範圍限制
- 模型參考字串格式正確（`provider/model`）

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| AgentsConfig | `source-repo/src/config/types.agents.ts` | 115-118 |
| AgentConfig | `source-repo/src/config/types.agents.ts` | 65-113 |
| AgentRuntimeConfig | `source-repo/src/config/types.agents.ts` | 23-30 |
| AgentBindingMatch | `source-repo/src/config/types.agents.ts` | 32-40 |
| AgentBinding | `source-repo/src/config/types.agents.ts` | 42-63 |
| AgentDefaultsConfig | `source-repo/src/config/types.agent-defaults.ts` | 149-250+ |
| AgentStartupContextConfig | `source-repo/src/config/types.agent-defaults.ts` | 53-66 |
| AgentContextPruningConfig | `source-repo/src/config/types.agent-defaults.ts` | 30-51 |
| CliBackendConfig | `source-repo/src/config/types.agent-defaults.ts` | 68-147 |
| 並行限制 | `source-repo/src/config/agent-limits.ts` | 3-6 |
| 逾時常數 | `source-repo/src/agents/timeout.ts` | 3-4 |
| Registry 常數 | `source-repo/src/agents/subagent-registry.ts` | 128-140 |
