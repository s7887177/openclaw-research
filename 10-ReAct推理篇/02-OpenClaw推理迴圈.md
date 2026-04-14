# OpenClaw 的推理迴圈——深度解析

> **摘要**：本文深入解析 OpenClaw 的推理迴圈（Inference Loop）實作，從完整管線的每個階段開始，涵蓋生命週期事件、串流回應機制、工具選擇策略、會話序列化、Hooks 機制、除錯觀測和效能調校。配合 TypeScript/Python 虛擬碼幫助理解程式碼層級的運作。

---

## 目錄

1. [完整管線概覽](#1-完整管線概覽)
2. [Intake 階段](#2-intake-階段)
3. [Context Assembly 階段](#3-context-assembly-階段)
4. [Model Inference 階段](#4-model-inference-階段)
5. [Tool Execution 階段](#5-tool-execution-階段)
6. [Streaming 階段](#6-streaming-階段)
7. [Persistence 階段](#7-persistence-階段)
8. [生命週期事件](#8-生命週期事件)
9. [串流回應機制](#9-串流回應機制)
10. [工具選擇策略](#10-工具選擇策略)
11. [會話序列化](#11-會話序列化)
12. [Hooks 機制](#12-hooks-機制)
13. [除錯與觀測](#13-除錯與觀測)
14. [效能調校](#14-效能調校)
15. [程式碼層級理解](#15-程式碼層級理解)

---

## 1. 完整管線概覽

### 1.1 管線流程圖

```
使用者訊息（Discord/Telegram/DM/API）
│
▼
┌─────────────────────────────────────────────────────────────┐
│  1. INTAKE（接收）                                          │
│  ├── 通道 Adapter 接收原始訊息                              │
│  ├── 解析訊息格式（文字/語音/圖片/指令）                     │
│  ├── 識別使用者身份                                         │
│  ├── 路由到目標 Agent                                       │
│  └── 建立或恢復 Session                                     │
├─────────────────────────────────────────────────────────────┤
│  2. CONTEXT ASSEMBLY（上下文組裝）                           │
│  ├── 載入系統提示（SOUL.md + 安全指令）                      │
│  ├── 載入使用者資料（USER.md）                              │
│  ├── 檢索相關記憶（Memory Recall）                          │
│  ├── 載入會話歷史                                           │
│  ├── 組裝工具定義                                           │
│  └── Token 預算分配                                         │
├─────────────────────────────────────────────────────────────┤
│  3. MODEL INFERENCE（模型推理）                              │
│  ├── 發送上下文到 AI 模型 API                               │
│  ├── 接收串流回應                                           │
│  ├── 解析回應（文字 / 工具調用）                             │
│  └── 如果是工具調用 → 進入步驟 4                            │
│      如果是純文字 → 跳到步驟 5                               │
├─────────────────────────────────────────────────────────────┤
│  4. TOOL EXECUTION（工具執行）                               │
│  ├── 九層權限檢查                                           │
│  ├── 參數驗證與清理                                         │
│  ├── 沙箱路由決策                                           │
│  ├── 人工批准檢查                                           │
│  ├── 執行工具                                               │
│  ├── 收集觀察結果                                           │
│  └── 將結果加入上下文 → 回到步驟 3（下一輪推理）            │
├─────────────────────────────────────────────────────────────┤
│  5. STREAMING（串流輸出）                                    │
│  ├── 透過 WebSocket/SSE 串流回應給前端                      │
│  ├── 通道 Adapter 格式化輸出                                │
│  └── 發送到 Discord/Telegram/等                              │
├─────────────────────────────────────────────────────────────┤
│  6. PERSISTENCE（持久化）                                    │
│  ├── 儲存對話歷史                                           │
│  ├── 更新記憶（Memory Write）                               │
│  ├── 更新使用統計                                           │
│  └── 記錄日誌                                               │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 內部資料結構

```typescript
// 核心資料結構
interface InferenceContext {
  sessionId: string;
  agentId: string;
  providerId: string;
  userId: string;
  channelId: string;

  // 上下文訊息
  messages: Message[];

  // 可用工具
  tools: ToolDefinition[];

  // 配置
  config: AgentConfig;

  // 元資料
  metadata: {
    startTime: number;
    iteration: number;
    totalTokens: number;
    toolCalls: ToolCallRecord[];
  };
}

interface Message {
  role: "system" | "user" | "assistant" | "tool";
  content: string;
  name?: string;
  toolCallId?: string;
  timestamp: number;
}

interface ToolCallRecord {
  id: string;
  tool: string;
  args: Record<string, any>;
  result: string;
  duration: number;
  status: "success" | "error" | "denied" | "timeout";
  permissionCheckResult: PermissionResult;
}
```

---

## 2. Intake 階段

### 2.1 訊息接收

```typescript
// 通道 Adapter 接收訊息
class DiscordAdapter {
  async onMessage(discordMessage: DiscordMessage): Promise<void> {
    // 1. 過濾：忽略 Bot 自己的訊息
    if (discordMessage.author.bot) return;

    // 2. 解析：提取訊息內容
    const message = this.parseMessage(discordMessage);

    // 3. 識別使用者
    const user = await this.identifyUser(discordMessage.author);

    // 4. 路由：決定使用哪個 Agent
    const agentId = this.routeToAgent(discordMessage.channel);

    // 5. Session 管理
    const session = await this.getOrCreateSession(
      user.id,
      discordMessage.channel.id,
      agentId
    );

    // 6. 發送到推理引擎
    await this.inferenceEngine.process({
      session,
      message,
      user,
      agentId,
      channel: {
        type: "discord",
        id: discordMessage.channel.id,
        guildId: discordMessage.guild?.id
      }
    });
  }
}
```

### 2.2 訊息格式解析

```typescript
interface ParsedMessage {
  type: "text" | "voice" | "image" | "command" | "reaction";
  content: string;
  attachments: Attachment[];
  mentions: Mention[];
  replyTo?: string;  // 回覆的訊息 ID
  raw: any;          // 原始通道訊息
}

function parseDiscordMessage(msg: DiscordMessage): ParsedMessage {
  return {
    type: msg.attachments.size > 0 ? "image" : "text",
    content: msg.content,
    attachments: Array.from(msg.attachments.values()),
    mentions: msg.mentions.users.map(u => ({
      id: u.id,
      name: u.username,
      isBot: u.bot
    })),
    replyTo: msg.reference?.messageId,
    raw: msg
  };
}
```

---

## 3. Context Assembly 階段

### 3.1 上下文組裝流程

```typescript
class ContextAssembler {
  async assemble(request: InferenceRequest): Promise<InferenceContext> {
    const config = await this.loadAgentConfig(request.agentId);
    const tokenBudget = config.maxContextTokens || 8192;

    // 1. 系統提示（最高優先級，不可壓縮）
    const systemPrompt = await this.buildSystemPrompt(config);
    let usedTokens = this.estimateTokens(systemPrompt);

    // 2. 工具定義（高優先級）
    const tools = await this.getAvailableTools(request.agentId, config);
    usedTokens += this.estimateToolTokens(tools);

    // 3. 使用者資料
    const userProfile = await this.loadUserProfile(request.user.id);
    usedTokens += this.estimateTokens(userProfile);

    // 4. 記憶檢索（根據相關性）
    const remainingBudget = tokenBudget - usedTokens;
    const memoryBudget = Math.floor(remainingBudget * 0.3); // 30% 給記憶
    const memories = await this.recallMemories(
      request.message.content,
      memoryBudget
    );
    usedTokens += this.estimateTokens(memories.join("\n"));

    // 5. 會話歷史（最近的優先）
    const historyBudget = tokenBudget - usedTokens - 500; // 預留 500 給回應
    const history = await this.loadSessionHistory(
      request.session.id,
      historyBudget
    );

    // 6. 組裝最終上下文
    return {
      sessionId: request.session.id,
      agentId: request.agentId,
      messages: [
        { role: "system", content: systemPrompt },
        ...history,
        { role: "user", content: request.message.content }
      ],
      tools,
      config,
      metadata: { startTime: Date.now(), iteration: 0, totalTokens: 0, toolCalls: [] }
    };
  }
}
```

### 3.2 系統提示組裝

```typescript
async buildSystemPrompt(config: AgentConfig): Promise<string> {
  const parts: string[] = [];

  // SOUL.md（人格定義）
  const soul = await fs.readFile(config.personaPath, "utf-8");
  parts.push(soul);

  // 安全指令（自動附加）
  parts.push(SECURITY_INSTRUCTIONS);

  // 日期時間上下文
  parts.push(`當前時間：${new Date().toISOString()}`);

  // 使用者資料
  if (config.userProfilePath) {
    const userMd = await fs.readFile(config.userProfilePath, "utf-8");
    parts.push(`\n## 使用者資料\n${userMd}`);
  }

  // Agent 特定指令
  if (config.systemPromptSuffix) {
    parts.push(config.systemPromptSuffix);
  }

  return parts.join("\n\n");
}
```

### 3.3 記憶檢索

```typescript
async recallMemories(query: string, tokenBudget: number): Promise<string[]> {
  // 使用向量搜尋找到相關記憶
  const results = await this.memoryStore.search({
    query,
    limit: 20,
    threshold: 0.7  // 相似度閾值
  });

  // 按相關性排序，在 token 預算內選取
  const selected: string[] = [];
  let usedTokens = 0;

  for (const result of results) {
    const tokens = this.estimateTokens(result.content);
    if (usedTokens + tokens > tokenBudget) break;
    selected.push(result.content);
    usedTokens += tokens;
  }

  return selected;
}
```

---

## 4. Model Inference 階段

### 4.1 模型調用

```typescript
class InferenceEngine {
  async infer(context: InferenceContext): Promise<InferenceResult> {
    const provider = this.getProvider(context.config.model);

    // 建構 API 請求
    const request = {
      model: context.config.model.split("/")[1],
      messages: context.messages,
      tools: context.tools.map(t => t.toAPIFormat()),
      temperature: context.config.temperature || 0.7,
      max_tokens: context.config.maxTokensPerRequest || 4096,
      stream: true  // 啟用串流
    };

    // 發送請求
    const stream = await provider.createChatCompletion(request);

    // 處理串流回應
    const result = await this.processStream(stream, context);

    return result;
  }
}
```

### 4.2 回應解析

```typescript
interface InferenceResult {
  type: "text" | "tool_calls" | "mixed";
  text?: string;
  toolCalls?: ToolCall[];
  usage: {
    promptTokens: number;
    completionTokens: number;
    totalTokens: number;
  };
  finishReason: "stop" | "tool_calls" | "length" | "content_filter";
}

interface ToolCall {
  id: string;
  function: {
    name: string;
    arguments: string;  // JSON 字串
  };
}
```

---

## 5. Tool Execution 階段

### 5.1 工具執行流程

```typescript
class ToolExecutor {
  async execute(
    toolCall: ToolCall,
    context: InferenceContext
  ): Promise<ToolResult> {

    const startTime = Date.now();

    // 1. 解析參數
    let args: Record<string, any>;
    try {
      args = JSON.parse(toolCall.function.arguments);
    } catch (e) {
      return { status: "error", content: "Invalid JSON arguments" };
    }

    // 2. 九層權限檢查
    const permCheck = await this.permissionEngine.check({
      tool: toolCall.function.name,
      args,
      agentId: context.agentId,
      providerId: context.providerId,
      sessionId: context.sessionId
    });

    if (!permCheck.allowed) {
      this.logger.warn("PERMISSION_DENIED", {
        tool: toolCall.function.name,
        layer: permCheck.deniedAtLayer,
        reason: permCheck.reason
      });
      return {
        status: "denied",
        content: `Permission denied: ${permCheck.reason}`
      };
    }

    // 3. 沙箱路由
    const executor = permCheck.sandbox
      ? this.sandboxExecutor
      : this.directExecutor;

    // 4. 人工批准檢查
    if (permCheck.requiresApproval) {
      const approved = await this.requestApproval(toolCall, context);
      if (!approved) {
        return { status: "denied", content: "Approval denied by operator" };
      }
    }

    // 5. 執行工具
    try {
      const result = await Promise.race([
        executor.run(toolCall.function.name, args),
        this.timeout(context.config.toolTimeout || 60000)
      ]);

      const duration = Date.now() - startTime;

      // 記錄
      context.metadata.toolCalls.push({
        id: toolCall.id,
        tool: toolCall.function.name,
        args,
        result: result.content,
        duration,
        status: "success",
        permissionCheckResult: permCheck
      });

      return { status: "success", content: result.content };

    } catch (error) {
      return { status: "error", content: `Tool error: ${error.message}` };
    }
  }
}
```

---

## 6. Streaming 階段

### 6.1 串流輸出機制

```typescript
class StreamingHandler {
  async streamToChannel(
    result: InferenceResult,
    channel: ChannelInfo,
    adapter: ChannelAdapter
  ): Promise<void> {

    if (result.type === "text" || result.type === "mixed") {
      // 串流文字回應
      const chunks = this.splitIntoChunks(result.text, 2000); // Discord 限制

      for (const chunk of chunks) {
        await adapter.send(channel.id, chunk);
        // 如果有多個 chunk，加入短暫延遲
        if (chunks.length > 1) {
          await this.delay(500);
        }
      }
    }

    if (result.type === "tool_calls") {
      // 工具調用中，發送 typing 指示器
      await adapter.sendTyping(channel.id);
    }
  }
}
```

---

## 7. Persistence 階段

### 7.1 持久化流程

```typescript
class PersistenceManager {
  async persist(context: InferenceContext, result: InferenceResult): Promise<void> {
    // 1. 儲存對話歷史
    await this.sessionStore.appendMessages(context.sessionId, [
      ...context.messages.slice(-2),  // 最新的使用者訊息和助手回覆
    ]);

    // 2. 自動記憶寫入
    if (this.shouldAutoMemorize(context, result)) {
      await this.memoryStore.write({
        key: `conversation_${context.sessionId}_${Date.now()}`,
        content: this.summarizeInteraction(context, result),
        metadata: {
          userId: context.userId,
          agentId: context.agentId,
          timestamp: Date.now()
        }
      });
    }

    // 3. 使用統計
    await this.usageTracker.record({
      sessionId: context.sessionId,
      providerId: context.providerId,
      model: context.config.model,
      promptTokens: result.usage.promptTokens,
      completionTokens: result.usage.completionTokens,
      toolCalls: context.metadata.toolCalls.length,
      duration: Date.now() - context.metadata.startTime
    });

    // 4. 日誌
    this.logger.info("INFERENCE_COMPLETE", {
      sessionId: context.sessionId,
      iterations: context.metadata.iteration,
      toolCalls: context.metadata.toolCalls.length,
      totalTokens: result.usage.totalTokens,
      duration: Date.now() - context.metadata.startTime
    });
  }
}
```

---

## 8. 生命週期事件

### 8.1 事件清單

```typescript
enum LifecycleEvent {
  // 推理開始
  INFERENCE_START = "inference:start",

  // 工具相關
  TOOL_CALL_REQUEST = "tool:call_request",
  TOOL_PERMISSION_CHECK = "tool:permission_check",
  TOOL_PERMISSION_DENIED = "tool:permission_denied",
  TOOL_APPROVAL_REQUEST = "tool:approval_request",
  TOOL_APPROVAL_RESULT = "tool:approval_result",
  TOOL_EXECUTION_START = "tool:execution_start",
  TOOL_EXECUTION_END = "tool:execution_end",
  TOOL_ERROR = "tool:error",

  // 串流
  STREAM_START = "stream:start",
  STREAM_CHUNK = "stream:chunk",
  STREAM_END = "stream:end",

  // 推理結束
  INFERENCE_END = "inference:end",
  INFERENCE_ERROR = "inference:error",

  // 記憶
  MEMORY_RECALL = "memory:recall",
  MEMORY_WRITE = "memory:write",

  // Session
  SESSION_CREATE = "session:create",
  SESSION_RESUME = "session:resume",
  SESSION_END = "session:end"
}
```

### 8.2 事件資料結構

```typescript
interface LifecycleEventData {
  "inference:start": {
    sessionId: string;
    agentId: string;
    userId: string;
    messagePreview: string;
  };

  "tool:call_request": {
    toolName: string;
    args: Record<string, any>;
    agentId: string;
    iteration: number;
  };

  "tool:execution_end": {
    toolName: string;
    duration: number;
    status: "success" | "error" | "denied" | "timeout";
    resultPreview: string;
  };

  "inference:end": {
    sessionId: string;
    totalIterations: number;
    totalToolCalls: number;
    totalTokens: number;
    totalDuration: number;
    finishReason: string;
  };
}
```

### 8.3 事件監聽範例

```typescript
// 監聽所有事件
gateway.on("*", (event, data) => {
  console.log(`[${event}]`, data);
});

// 監聽特定事件
gateway.on("tool:permission_denied", (data) => {
  alertOperator(`⚠️ Tool denied: ${data.toolName} for agent ${data.agentId}`);
});

gateway.on("inference:error", (data) => {
  alertOperator(`❌ Inference error: ${data.error}`);
});
```

---

## 9. 串流回應機制

### 9.1 WebSocket SSE

```
OpenClaw 支援兩種串流協議：

1. WebSocket（雙向通訊）
   ├── 適合需要即時互動的場景
   ├── 支援 binary 資料（語音）
   └── 支持全雙工通訊

2. Server-Sent Events (SSE)（單向串流）
   ├── 適合純文字串流
   ├── 基於 HTTP，更容易通過代理和防火牆
   └── 自動重連
```

### 9.2 串流協議

```
WebSocket 串流格式：

→ 連線建立
← {"type": "connected", "sessionId": "abc123"}

→ {"type": "message", "content": "今天天氣如何？"}
← {"type": "inference_start", "agentId": "default"}
← {"type": "stream_chunk", "content": "讓"}
← {"type": "stream_chunk", "content": "我"}
← {"type": "stream_chunk", "content": "查"}
← {"type": "stream_chunk", "content": "詢"}
← {"type": "tool_call", "tool": "search_web", "args": {"query": "台北天氣"}}
← {"type": "tool_result", "content": "台北 28°C 晴天"}
← {"type": "stream_chunk", "content": "根據查詢，"}
← {"type": "stream_chunk", "content": "台北今天..."}
← {"type": "inference_end", "totalTokens": 150}
```

---

## 10. 工具選擇策略

### 10.1 模型如何選擇工具

```
AI 模型選擇工具時考慮三個維度：

1. 實用性（Utility）
   → 這個工具能幫助回答問題嗎？
   → 例：使用者問天氣 → search_web 有用 → memory_read 無用

2. 風險（Risk）
   → 使用這個工具的風險有多大？
   → 例：search_web 低風險 → exec 高風險

3. 相關性（Relevance）
   → 這個工具和當前任務相關嗎？
   → 例：使用者問程式碼 → code_analyze 相關 → weather_query 不相關
```

### 10.2 工具排序

```typescript
function rankTools(
  tools: ToolDefinition[],
  userQuery: string,
  context: InferenceContext
): RankedTool[] {
  return tools.map(tool => ({
    tool,
    score: calculateToolScore(tool, userQuery, context)
  })).sort((a, b) => b.score - a.score);
}

function calculateToolScore(
  tool: ToolDefinition,
  query: string,
  context: InferenceContext
): number {
  let score = 0;

  // 相關性（基於工具描述和查詢的語意相似度）
  score += semanticSimilarity(tool.description, query) * 0.5;

  // 歷史使用頻率（常用工具權重更高）
  score += getUsageFrequency(tool.name) * 0.2;

  // 安全性（安全的工具優先）
  score += getToolSafetyScore(tool.name) * 0.3;

  return score;
}
```

---

## 11. 會話序列化

### 11.1 為什麼 Session 是串列的？

```
OpenClaw 的設計選擇：每個 Session 內的推理是串列執行的。

原因：

1. 上下文一致性
   如果兩個推理並行執行，它們可能同時修改記憶或狀態，
   導致不一致。

   並行（危險）：
     推理 A：讀取記憶 → 修改記憶 → 寫入
     推理 B：讀取記憶 → 修改記憶 → 寫入
     → B 可能覆蓋 A 的修改！

   串列（安全）：
     推理 A：讀取 → 修改 → 寫入（完成）
     推理 B：讀取 → 修改 → 寫入（基於 A 的結果）
     → 不會衝突

2. 因果關係
   對話有明確的因果順序：
   使用者問 A → Agent 答 A → 使用者問 B（基於 A 的回答）
   如果 A 和 B 並行處理，B 無法參考 A 的回答。

3. 實作簡單性
   串列執行更容易實作和除錯。
```

### 11.2 Session 佇列

```typescript
class SessionQueue {
  private queues: Map<string, AsyncQueue> = new Map();

  async enqueue(sessionId: string, task: InferenceTask): Promise<void> {
    if (!this.queues.has(sessionId)) {
      this.queues.set(sessionId, new AsyncQueue());
    }

    const queue = this.queues.get(sessionId)!;
    await queue.push(task);
  }
}

class AsyncQueue {
  private processing = false;
  private tasks: InferenceTask[] = [];

  async push(task: InferenceTask): Promise<void> {
    this.tasks.push(task);
    if (!this.processing) {
      this.processing = true;
      while (this.tasks.length > 0) {
        const next = this.tasks.shift()!;
        await next.execute();
      }
      this.processing = false;
    }
  }
}
```

### 11.3 跨 Session 並行

```
雖然 Session 內是串列的，不同 Session 之間是並行的：

Session A (使用者甲): ███░░░███░░░███  → 串列處理
Session B (使用者乙): ░░░███░░░███░░░  → 串列處理
Session C (使用者丙): ██░░░░░██░░░░██  → 串列處理

但 A、B、C 之間是並行的！
→ 不同使用者不會互相等待
```

---

## 12. Hooks 機制

### 12.1 Hook 類型

```typescript
interface InferenceHooks {
  // 推理前（可以修改上下文）
  preInference?: (context: InferenceContext) => Promise<InferenceContext>;

  // 推理後（可以修改結果）
  postInference?: (context: InferenceContext, result: InferenceResult) => Promise<InferenceResult>;

  // 工具調用前（可以修改參數或拒絕）
  preTool?: (toolCall: ToolCall, context: InferenceContext) => Promise<ToolCall | null>;

  // 工具調用後（可以修改結果）
  postTool?: (toolCall: ToolCall, result: ToolResult, context: InferenceContext) => Promise<ToolResult>;

  // 串流前
  preStream?: (chunk: string, context: InferenceContext) => Promise<string>;

  // 持久化前
  prePersist?: (context: InferenceContext, result: InferenceResult) => Promise<void>;
}
```

### 12.2 Hook 使用範例

```typescript
// 範例 1：在每次推理前注入當前時間
const timeHook: InferenceHooks = {
  preInference: async (context) => {
    const timeInfo = `當前時間：${new Date().toLocaleString("zh-TW")}`;
    context.messages[0].content += `\n${timeInfo}`;
    return context;
  }
};

// 範例 2：在工具調用前記錄
const auditHook: InferenceHooks = {
  preTool: async (toolCall, context) => {
    await auditLog.write({
      tool: toolCall.function.name,
      args: toolCall.function.arguments,
      agent: context.agentId,
      user: context.userId,
      timestamp: new Date()
    });
    return toolCall;
  }
};

// 範例 3：過濾敏感資訊
const filterHook: InferenceHooks = {
  postTool: async (toolCall, result, context) => {
    // 過濾工具結果中的敏感資訊
    result.content = redactSensitiveInfo(result.content);
    return result;
  },
  preStream: async (chunk, context) => {
    // 過濾串流輸出中的敏感資訊
    return redactSensitiveInfo(chunk);
  }
};
```

---

## 13. 除錯與觀測

### 13.1 日誌層級

```json5
{
  logging: {
    level: "debug",  // 開發時使用 debug
    // level: "info",  // 生產時使用 info

    // 格式
    format: "json",  // json | pretty | compact

    // 分類過濾
    categories: {
      "inference": "debug",   // 推理相關
      "tools": "debug",       // 工具相關
      "permissions": "info",  // 權限相關
      "memory": "info",       // 記憶相關
      "channels": "warn",     // 通道相關
      "sandbox": "debug"      // 沙箱相關
    }
  }
}
```

### 13.2 OpenTelemetry 整合

```json5
{
  telemetry: {
    enabled: true,
    exporter: "otlp",
    endpoint: "http://localhost:4317",

    // Traces（分散式追蹤）
    traces: {
      enabled: true,
      // 追蹤每個推理迴圈
      sample_rate: 1.0  // 100% 取樣（開發用）
    },

    // Metrics（指標）
    metrics: {
      enabled: true,
      export_interval: 10000  // 每 10 秒輸出
    },

    // Logs
    logs: {
      enabled: true
    }
  }
}
```

### 13.3 除錯工具

```bash
# 觀察即時推理過程
openclaw run --verbose

# 模擬工具調用（不實際執行）
openclaw call search_web --args '{"query": "test"}' --dry-run

# 檢查權限
openclaw probe tool exec --agent social-bot --provider openai

# 輸出：
# Permission check for tool 'exec':
#   L1 (Global Settings): PASS
#   L2 (Provider): PASS
#   L3 (Global Tools): FAIL - tool in denied list
# Result: DENIED at Layer 3

# 查看使用統計
openclaw usage-cost --period today
```

---

## 14. 效能調校

### 14.1 Token 壓縮

```typescript
class TokenCompressor {
  async compress(context: InferenceContext): Promise<InferenceContext> {
    const messages = context.messages;

    // 策略 1：摘要舊對話
    if (messages.length > 20) {
      const oldMessages = messages.slice(1, -10); // 保留系統提示和最近 10 條
      const summary = await this.summarize(oldMessages);
      context.messages = [
        messages[0],  // 系統提示
        { role: "system", content: `之前的對話摘要：\n${summary}` },
        ...messages.slice(-10)  // 最近 10 條
      ];
    }

    // 策略 2：截斷長的工具結果
    for (const msg of context.messages) {
      if (msg.role === "tool" && msg.content.length > 2000) {
        msg.content = msg.content.slice(0, 2000) + "\n...(結果已截斷)";
      }
    }

    return context;
  }
}
```

### 14.2 工具結果快取

```json5
{
  tools: {
    cache: {
      enabled: true,
      // 快取搜尋結果 5 分鐘
      search_web: { ttl: 300 },
      // 快取天氣 30 分鐘
      weather_query: { ttl: 1800 },
      // 不快取記憶操作（資料可能變化）
      memory_read: { ttl: 0 },
      memory_write: { ttl: 0 }
    }
  }
}
```

### 14.3 模型預熱

```json5
{
  performance: {
    // 預先建立模型連線
    pre_warm: {
      enabled: true,
      models: ["openai/gpt-4", "ollama/llama3"],
      // 在 Gateway 啟動時就建立連線
      on_start: true
    },

    // 併發控制
    concurrency: {
      // 最大並行推理數
      max_concurrent_inferences: 10,
      // 每個提供者的最大並行請求
      max_per_provider: 5
    }
  }
}
```

---

## 15. 程式碼層級理解

### 15.1 完整推理迴圈（TypeScript 虛擬碼）

```typescript
class OpenClawInferenceLoop {
  async run(request: InferenceRequest): Promise<void> {
    // === INTAKE ===
    const session = await this.sessionManager.getOrCreate(request);
    this.emit("inference:start", { sessionId: session.id });

    // === CONTEXT ASSEMBLY ===
    let context = await this.contextAssembler.assemble(request);
    context = await this.hooks.preInference(context);

    // === INFERENCE LOOP ===
    for (let i = 0; i < this.config.maxIterations; i++) {
      context.metadata.iteration = i;

      // Model Inference
      let result = await this.inferenceEngine.infer(context);
      result = await this.hooks.postInference(context, result);

      // 純文字回應 → 串流輸出
      if (result.text) {
        const filtered = await this.hooks.preStream(result.text, context);
        await this.streamingHandler.stream(filtered, request.channel);
      }

      // 沒有工具調用 → 結束
      if (!result.toolCalls || result.toolCalls.length === 0) {
        break;
      }

      // 處理工具調用
      for (const toolCall of result.toolCalls) {
        // Pre-tool hook
        const modifiedCall = await this.hooks.preTool(toolCall, context);
        if (!modifiedCall) continue; // hook 拒絕了

        this.emit("tool:call_request", {
          toolName: toolCall.function.name,
          args: JSON.parse(toolCall.function.arguments)
        });

        // 執行工具
        const toolResult = await this.toolExecutor.execute(modifiedCall, context);

        // Post-tool hook
        const filteredResult = await this.hooks.postTool(
          modifiedCall, toolResult, context
        );

        this.emit("tool:execution_end", {
          toolName: toolCall.function.name,
          status: toolResult.status,
          duration: toolResult.duration
        });

        // 將工具結果加入上下文
        context.messages.push({
          role: "assistant",
          content: null,
          toolCalls: [toolCall]
        });
        context.messages.push({
          role: "tool",
          content: filteredResult.content,
          toolCallId: toolCall.id
        });
      }

      // Token 壓縮
      if (this.estimateTokens(context) > this.config.maxContextTokens * 0.8) {
        context = await this.tokenCompressor.compress(context);
      }
    }

    // === PERSISTENCE ===
    await this.hooks.prePersist(context, result);
    await this.persistenceManager.persist(context, result);

    this.emit("inference:end", {
      sessionId: session.id,
      totalIterations: context.metadata.iteration,
      totalToolCalls: context.metadata.toolCalls.length,
      totalTokens: context.metadata.totalTokens,
      totalDuration: Date.now() - context.metadata.startTime
    });
  }
}
```

---

## 總結

OpenClaw 的推理迴圈是一個精心設計的管線，每個階段都有明確的職責：

| 階段 | 職責 | 關鍵技術 |
|------|------|----------|
| Intake | 接收和路由 | 通道 Adapter、Session 管理 |
| Context Assembly | 組裝上下文 | Token 預算、記憶檢索 |
| Model Inference | AI 推理 | 串流 API、回應解析 |
| Tool Execution | 工具執行 | 九層權限、沙箱 |
| Streaming | 輸出 | WebSocket/SSE |
| Persistence | 持久化 | 記憶寫入、日誌 |

理解這個管線是深入使用和擴展 OpenClaw 的基礎。

---

> **下一章**：[CLI 完整指令參考](../11-CLI指令參考/01-完整CLI指令參考.md)
