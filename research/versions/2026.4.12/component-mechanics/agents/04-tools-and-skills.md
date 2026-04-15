# Agents 子章節 04：工具與技能系統

> **引用範圍**：`src/agents/openclaw-tools.ts`, `src/agents/tools/`, `src/agents/skills/`

## 1. 工具工廠：createOpenClawTools()

`createOpenClawTools()` 是 Agent 工具集的核心工廠函式，定義在 `openclaw-tools.ts:51`。它接收大量上下文參數，回傳 `AnyAgentTool[]` 陣列。

### 1.1 函式簽名

```typescript
// source-repo/src/agents/openclaw-tools.ts:51-114
export function createOpenClawTools(
  options?: {
    sandboxBrowserBridgeUrl?: string;
    allowHostBrowserControl?: boolean;
    agentSessionKey?: string;
    agentChannel?: GatewayMessageChannel;
    agentAccountId?: string;
    agentTo?: string;
    agentThreadId?: string | number;
    agentDir?: string;
    sandboxRoot?: string;
    sandboxContainerWorkdir?: string;
    sandboxFsBridge?: SandboxFsBridge;
    fsPolicy?: ToolFsPolicy;
    sandboxed?: boolean;
    config?: OpenClawConfig;
    pluginToolAllowlist?: string[];
    modelHasVision?: boolean;
    modelProvider?: string;
    modelId?: string;
    // ... 更多參數（共 30+ 個選項）
  } & SpawnedToolContext,
): AnyAgentTool[]
```

> **引用來源**：`source-repo/src/agents/openclaw-tools.ts:51-114`

### 1.2 上下文感知設計

工具工廠的參數設計體現了 OpenClaw 的**上下文感知**理念：

- **Agent 身份**：`agentSessionKey`, `agentAccountId`, `requesterAgentIdOverride`
- **通道資訊**：`agentChannel`, `agentTo`, `agentThreadId`
- **沙箱環境**：`sandboxRoot`, `sandboxFsBridge`, `fsPolicy`, `sandboxed`
- **模型能力**：`modelHasVision`, `modelProvider`, `modelId`
- **安全**：`senderIsOwner`, `requesterSenderId`

這些參數讓工具能根據呼叫者的身份、環境、能力自動調整行為。

---

## 2. 內建工具清單

工廠函式建立以下核心工具：

### 2.1 Session 管理工具

| 工具名稱 | 建立函式 | 用途 |
|----------|----------|------|
| `sessions_list` | `createSessionsListTool()` | 列出所有 Session |
| `sessions_history` | `createSessionsHistoryTool()` | 查看 Session 歷史 |
| `sessions_send` | `createSessionsSendTool()` | 傳送訊息到其他 Session |
| `sessions_spawn` | `createSessionsSpawnTool()` | 生成 Sub-agent |
| `sessions_yield` | `createSessionsYieldTool()` | 暫停並回傳中間結果 |
| `session_status` | `createSessionStatusTool()` | 查看 Session 狀態 |
| `subagents` | `createSubagentsTool()` | 管理 Sub-agent |
| `agents_list` | `createAgentsListTool()` | 列出可用 Agent |

> **引用來源**：`source-repo/src/agents/openclaw-tools.ts:230-303`

### 2.2 媒體工具

| 工具名稱 | 建立函式 | 用途 |
|----------|----------|------|
| `image` | `createImageTool()` | 圖片理解（需 agentDir） |
| `image_generate` | `createImageGenerateTool()` | 圖片生成 |
| `video_generate` | `createVideoGenerateTool()` | 影片生成 |
| `music_generate` | `createMusicGenerateTool()` | 音樂生成 |
| `pdf` | `createPdfTool()` | PDF 處理（需 agentDir） |

> **引用來源**：`source-repo/src/agents/openclaw-tools.ts:142-185`

### 2.3 通訊與網路工具

| 工具名稱 | 建立函式 | 用途 |
|----------|----------|------|
| `web_search` | `createWebSearchTool()` | 網路搜尋 |
| `web_fetch` | `createWebFetchTool()` | 網頁擷取 |
| `message` | `createMessageTool()` | 傳送訊息到通道 |
| `tts` | `createTtsTool()` | 文字轉語音 |

> **引用來源**：`source-repo/src/agents/openclaw-tools.ts:186-213`

### 2.4 系統工具

| 工具名稱 | 建立函式 | 用途 |
|----------|----------|------|
| `canvas` | `createCanvasTool()` | 畫布操作 |
| `nodes` | `createNodesTool()` | 節點操作（複合工具） |
| `cron` | `createCronTool()` | 排程任務 |
| `gateway` | `createGatewayTool()` | Gateway 操作 |
| `update_plan` | `createUpdatePlanTool()` | 更新計劃（條件啟用） |

> **引用來源**：`source-repo/src/agents/openclaw-tools.ts:230-258`

### 2.5 條件性工具

某些工具只在特定條件下啟用：

```typescript
// source-repo/src/agents/openclaw-tools.ts:142-151
// image tool — 需要 agentDir 才啟用
const imageTool = options?.agentDir?.trim()
  ? createImageTool({ ... })
  : null;

// source-repo/src/agents/openclaw-tools.ts:177-185
// pdf tool — 同上
const pdfTool = options?.agentDir?.trim()
  ? createPdfTool({ ... })
  : null;

// source-repo/src/agents/openclaw-tools.ts:250-258
// update_plan — 需要特定配置啟用
...(isUpdatePlanToolEnabledForOpenClawTools({ ... })
  ? [createUpdatePlanTool()]
  : []),

// source-repo/src/agents/openclaw-tools.ts:196-213
// message tool — 可透過 disableMessageTool 關閉
const messageTool = options?.disableMessageTool
  ? null
  : createMessageTool({ ... });
```

> **引用來源**：`source-repo/src/agents/openclaw-tools.ts:142-258`

---

## 3. 工具註冊策略

```typescript
// source-repo/src/agents/openclaw-tools.registration.ts
collectPresentOpenClawTools([...])   // 過濾 null 工具
isUpdatePlanToolEnabledForOpenClawTools({ ... })  // 判斷 update_plan 是否啟用
```

`collectPresentOpenClawTools()` 用於過濾掉 `null`（未啟用）的工具，確保工具列表中只包含有效工具。

> **引用來源**：`source-repo/src/agents/openclaw-tools.registration.ts`

---

## 4. Plugin 工具

除了內建工具，工廠還會載入 Plugin 工具：

```typescript
// source-repo/src/agents/openclaw-tools.ts:306-316
if (options?.disablePluginTools) {
  return tools;
}

const wrappedPluginTools = resolveOpenClawPluginToolsForOptions({
  options,
  resolvedConfig,
  existingToolNames: new Set(tools.map((tool) => tool.name)),
});

return [...tools, ...wrappedPluginTools];
```

Plugin 工具透過 `resolveOpenClawPluginToolsForOptions()`（在 `openclaw-plugin-tools.ts` 中）解析。系統會：
1. 檢查 `pluginToolAllowlist`
2. 確認沒有與內建工具重名
3. 包裝 Plugin 工具以符合 `AnyAgentTool` 介面

> **引用來源**：`source-repo/src/agents/openclaw-tools.ts:306-316`

---

## 5. 工具顯示配置

`tool-display-config.ts`（701 行）定義了所有工具的顯示設定，包括：
- 工具名稱的人類可讀標籤
- 圖示
- 分類
- 排序順序
- 顯示/隱藏條件

這是一個純配置檔案，不包含邏輯。

---

## 6. 技能系統

### 6.1 目錄結構

```
src/agents/skills/
├── plugin-skills.ts          # 技能 Plugin 載入
├── workspace.ts              # 工作區技能探索
├── skill-contract.ts         # 技能合約驗證
├── frontmatter.ts            # YAML frontmatter 解析
├── agent-dir-skills.ts       # Agent 目錄技能
├── skill-resolution.ts       # 技能解析
└── ... (共 27 個檔案)
```

### 6.2 技能載入流程

```
1. 掃描工作區中的技能定義
   └─ workspace.ts: 搜尋 .md, .yaml 格式的技能檔案

2. 解析 YAML frontmatter
   └─ frontmatter.ts: 提取技能元資料（名稱、描述、參數）

3. 驗證技能合約
   └─ skill-contract.ts: 確認技能符合預期格式

4. 應用技能過濾器
   └─ agent-scope.ts:186: 根據 Agent 的 skills[] 白名單過濾

5. 註冊為可用技能
   └─ plugin-skills.ts: 整合到 Agent 的工具列表中
```

### 6.3 技能過濾

```typescript
// source-repo/src/agents/agent-scope.ts:186
export function resolveAgentSkillsFilter(
  cfg: OpenClawConfig,
  agentId: string
): string[] | undefined
```

技能過濾規則：
- Agent 設定 `skills: ["web_search", "code_review"]` → 只啟用這兩個技能
- 未設定 → 繼承 `agents.defaults.skills`
- 未設定且無全局預設 → 所有技能都可用

> **引用來源**：`source-repo/src/agents/agent-scope.ts:84, 186`

---

## 7. Nodes 工具的工作區防護

`nodes` 工具有一個額外的工作區防護層：

```typescript
// source-repo/src/agents/openclaw-tools.ts:214-229
const nodesToolBase = createNodesTool({ ... });
const nodesTool = applyNodesToolWorkspaceGuard(nodesToolBase, {
  fsPolicy: options?.fsPolicy,
  sandboxContainerWorkdir: options?.sandboxContainerWorkdir,
  sandboxRoot: options?.sandboxRoot,
  workspaceDir,
});
```

`applyNodesToolWorkspaceGuard()`（在 `openclaw-tools.nodes-workspace-guard.ts` 中）確保 `nodes` 工具只能存取允許的檔案系統路徑。

> **引用來源**：`source-repo/src/agents/openclaw-tools.ts:214-229`

---

## 8. tools/ 子目錄

`src/agents/tools/`（110 個檔案）包含每個工具的獨立實作。主要檔案：

| 檔案 | 工具 |
|------|------|
| `web-tools.ts` | web_search, web_fetch |
| `image-tool.ts` | image |
| `image-generate-tool.ts` | image_generate |
| `video-generate-tool.ts` | video_generate |
| `music-generate-tool.ts` | music_generate |
| `pdf-tool.ts` | pdf |
| `message-tool.ts` | message |
| `tts-tool.ts` | tts |
| `canvas-tool.ts` | canvas |
| `nodes-tool.ts` | nodes |
| `cron-tool.ts` | cron |
| `gateway-tool.ts` | gateway |
| `agents-list-tool.ts` | agents_list |
| `sessions-spawn-tool.ts` | sessions_spawn |
| `sessions-list-tool.ts` | sessions_list |
| `sessions-history-tool.ts` | sessions_history |
| `sessions-send-tool.ts` | sessions_send |
| `sessions-yield-tool.ts` | sessions_yield |
| `session-status-tool.ts` | session_status |
| `subagents-tool.ts` | subagents |
| `update-plan-tool.ts` | update_plan |
| `common.ts` | AnyAgentTool 型別定義 |

> **引用來源**：`source-repo/src/agents/openclaw-tools.ts:16-37`（import 列表）

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| createOpenClawTools() | `source-repo/src/agents/openclaw-tools.ts` | 51 |
| 工具清單（import） | `source-repo/src/agents/openclaw-tools.ts` | 16-37 |
| 工具組裝 | `source-repo/src/agents/openclaw-tools.ts` | 230-304 |
| Plugin 工具載入 | `source-repo/src/agents/openclaw-tools.ts` | 306-316 |
| 條件性 image tool | `source-repo/src/agents/openclaw-tools.ts` | 142-151 |
| 條件性 message tool | `source-repo/src/agents/openclaw-tools.ts` | 196-213 |
| Nodes 工作區防護 | `source-repo/src/agents/openclaw-tools.ts` | 214-229 |
| 技能過濾 | `source-repo/src/agents/agent-scope.ts` | 186 |
| 工具註冊 | `source-repo/src/agents/openclaw-tools.registration.ts` | — |
