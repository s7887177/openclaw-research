# 輔助模組與狀態管理（Auxiliary Modules & State Management）

> **版本**：OpenClaw v2026.4.12
> **範圍**：`source-repo/src/auto-reply/` 中除 `dispatch*.ts`、`reply.ts`、`reply/`、`templating.ts`、`envelope.ts`、`commands-registry*.ts` 以外的所有輔助模組

---

## 開頭摘要

OpenClaw 的自動回覆子系統除了核心的派發引擎和回覆管線之外，還包含一系列不可或缺的輔助模組。這些模組處理了訊息處理流水線中的每一個「邊緣情境」（edge case）：當回覆文字超過通道字元限制時，`chunk.ts` 負責智慧分塊；當 Agent 閒置時，`heartbeat.ts` 排程定期心跳以維持互動；`tokens.ts` 管理靜默回覆（NO_REPLY）和心跳確認（HEARTBEAT_OK）的 Token 偵測；`thinking.ts` 提供多層次的思考模式控制；`command-auth.ts` 實作了一套複雜的多階段命令授權系統——包含供應商偵測、擁有者解析、寄件人驗證等。每個模組都遵循相同的設計哲學：模組化、可設定、外掛驅動，並透過延遲載入（lazy loading）和記憶化（memoization）最佳化效能。本文逐一剖析這些模組的實作細節。

---

## 1. 分塊處理（Chunking）

**檔案**：`chunk.ts`（483 行）

當 AI 模型生成的回覆超過通道的字元限制（例如 WhatsApp 的 4096 字元、SMS 的 160 字元）時，分塊模組負責將長文字智慧地切割成多個片段，同時保持語義完整性。

### 1.1 核心常數與設定解析

```typescript
// source-repo/src/auto-reply/chunk.ts:25-26
const DEFAULT_CHUNK_LIMIT = 4000;
const DEFAULT_CHUNK_MODE: ChunkMode = "length";

export type TextChunkProvider = ChannelId;   // 通道識別符
export type ChunkMode = "length" | "newline"; // 分塊模式
```

分塊限制的解析遵循一個三層級的「設定瀑布」（configuration cascade）：

```typescript
// source-repo/src/auto-reply/chunk.ts:56-79
export function resolveTextChunkLimit(
  cfg: OpenClawConfig,
  provider?: TextChunkProvider,
  accountId?: string,
  opts?: { fallback?: number }
): number {
  // 1. 帳號級覆寫（account-level override）
  // 2. 供應商級覆寫（provider-level override）
  // 3. 全域回退值（global fallback → DEFAULT_CHUNK_LIMIT = 4000）
}
```

### 1.2 分塊策略

系統提供兩種主要分塊模式和多種具體策略：

#### 依長度分塊（chunkText）

```typescript
// source-repo/src/auto-reply/chunk.ts:308-319
export function chunkText(text: string, limit: number): string[] {
  // 括號感知的斷點偵測
  // 優先在換行處斷開，其次在空白處斷開
  // 使用 scanParenAwareBreakpoints() 避免在括號內斷開
}
```

`scanParenAwareBreakpoints()`（第 448-482 行）是一個精巧的斷點掃描器：它追蹤括號深度（parenthesis depth），確保不會在函式呼叫或括號表達式的中間斷開文字。

#### 依換行分塊（chunkByNewline）

```typescript
// source-repo/src/auto-reply/chunk.ts:120-173
export function chunkByNewline(text: string, maxLineLength: number, opts?): string[] {
  // 在換行處切割
  // 修剪空白行
  // 收合連續空行
}
```

#### 依段落分塊（chunkByParagraph）

```typescript
// source-repo/src/auto-reply/chunk.ts:184-248
export function chunkByParagraph(text: string, limit: number, opts?): string[] {
  // 在段落邊界（空行）切割
  // 圍欄感知（fence-aware）：不在程式碼區塊中間斷開
  // 對過長的段落回退到長度切割
}
```

#### Markdown 感知分塊（chunkMarkdownText）

```typescript
// source-repo/src/auto-reply/chunk.ts:321-419
export function chunkMarkdownText(text: string, limit: number): string[] {
  // 最精密的分塊策略：
  // 1. 使用 parseFenceSpans() 識別程式碼圍欄區塊
  // 2. 避免在圍欄內部斷開
  // 3. 若必須在圍欄內斷開：
  //    - 在前一塊末尾關閉圍欄（加入 ``` ）
  //    - 在下一塊開頭重新開啟圍欄（加入 ```language）
  // 4. 追蹤圍欄狀態跨越多個分塊
}
```

圍欄安全斷開（fence-safe breaking）是最複雜的功能：當一個程式碼區塊跨越多個分塊時，系統會自動關閉和重新開啟 Markdown 圍欄標記（如 `` ``` ``），確保每個分塊都是有效的 Markdown。

#### 統一入口

```typescript
// source-repo/src/auto-reply/chunk.ts:253-277
export function chunkMarkdownTextWithMode(text: string, limit: number, mode: ChunkMode): string[] {
  // 根據 mode 選擇段落或 Markdown 分塊
}

export function chunkTextWithMode(text: string, limit: number, mode: ChunkMode): string[] {
  // 根據 mode 選擇換行或長度分塊
}
```

---

## 2. 心跳回覆（Heartbeat Reply）

**檔案**：`heartbeat.ts`（299 行）、`heartbeat-filter.ts`（97 行）、`heartbeat-reply-payload.ts`（24 行）

心跳系統讓 Agent 能在無人互動時定期「甦醒」，檢查待辦任務或發送主動訊息。

### 2.1 心跳設定

```typescript
// source-repo/src/auto-reply/heartbeat.ts
export const DEFAULT_HEARTBEAT_EVERY = "30m";        // 預設每 30 分鐘
export const DEFAULT_HEARTBEAT_ACK_MAX_CHARS = 300;   // 確認訊息最大字元數
export const HEARTBEAT_PROMPT = "...";                // 心跳觸發提示詞

export type HeartbeatTask = {
  name: string;
  interval: string;   // 持續時間字串（如 "30m"、"1h"）
  prompt: string;      // 任務特定提示詞
};
```

### 2.2 心跳內容偵測

```typescript
// source-repo/src/auto-reply/heartbeat.ts:14-67
export function isHeartbeatContentEffectivelyEmpty(content: string): boolean {
  // 若內容僅包含以下元素，視為「有效空白」：
  // - 空白字元
  // - Markdown 標頭（#）
  // - 程式碼圍欄（```）
  // - 空的列表項目（- [ ]）
  // 使用正則表達式逐一移除上述元素後檢查是否為空
}
```

### 2.3 心跳 Token 剝離

```typescript
// source-repo/src/auto-reply/heartbeat.ts:124-185
export function stripHeartbeatToken(
  raw?: string,
  opts?: { mode?: StripHeartbeatMode; maxAckChars?: number }
): { shouldSkip: boolean; text: string; didStrip: boolean } {
  // "heartbeat" 模式：僅在結果符合 maxAckChars 限制時才剝離
  // "message" 模式：無條件剝離
  // 處理 HTML 和 Markdown 包裝器
  // 使用 stripTokenAtEdges() 迭代移除邊緣的 HEARTBEAT_TOKEN
}
```

`stripTokenAtEdges()`（第 76-122 行）是一個迭代剝離器——它在一個迴圈中反覆移除文字頭尾的 `HEARTBEAT_TOKEN`，直到不再發現任何 Token 為止。這處理了 Agent 可能在回覆中重複 Token 的情況。

### 2.4 任務排程

```typescript
// source-repo/src/auto-reply/heartbeat.ts:196-282
export function parseHeartbeatTasks(content: string): HeartbeatTask[] {
  // 簡易的 YAML 式解析器（非完整 YAML）
  // 基於縮排的狀態機
  // 追蹤 tasks: 區塊
  // 解析每個任務的 name:、interval:、prompt: 欄位
  // 支援前瞻式解析巢狀欄位
}

// source-repo/src/auto-reply/heartbeat.ts:287-298
export function isTaskDue(
  lastRunMs: number | undefined,
  interval: string,
  nowMs: number
): boolean {
  // 比較 nowMs - lastRunMs >= parseDurationMs(interval)
  // 首次執行（lastRunMs 為 undefined）始終到期
}
```

### 2.5 心跳過濾

```typescript
// source-repo/src/auto-reply/heartbeat-filter.ts:36-55
export function isHeartbeatUserMessage(message, heartbeatPrompt?): boolean {
  // 偵測 role === "user" 且訊息以 heartbeatPrompt 開頭
}

// source-repo/src/auto-reply/heartbeat-filter.ts:57-69
export function isHeartbeatOkResponse(message, ackMaxChars?): boolean {
  // 偵測 role === "assistant" 且訊息含 HEARTBEAT_OK Token
  // 拒絕含非文字內容的訊息
}

// source-repo/src/auto-reply/heartbeat-filter.ts:71-96
export function filterHeartbeatPairs<T>(messages: T[], ackMaxChars?, heartbeatPrompt?): T[] {
  // 線性掃描，移除連續的 [heartbeat user, heartbeat ok assistant] 配對
  // 用於上下文壓縮：從會話歷史中移除無意義的心跳交換
}
```

`filterHeartbeatPairs()` 用於上下文壓縮——在發送給 AI 模型的會話歷史中，心跳問答配對不攜帶有意義的資訊，移除它們可以節省寶貴的上下文 Token。

### 2.6 心跳回覆 Payload 解析

```typescript
// source-repo/src/auto-reply/heartbeat-reply-payload.ts:1-24
export function resolveHeartbeatReplyPayload(
  replyResult: ReplyPayload | ReplyPayload[] | undefined
): ReplyPayload | undefined {
  // 從單一或陣列 Payload 中找到最後一個有出站內容的 Payload
  // 使用 hasOutboundReplyContent() 從外掛 SDK 驗證
}
```

---

## 3. Token 管理

**檔案**：`tokens.ts`（180 行）

Token 模組定義和偵測兩個關鍵的控制 Token：`HEARTBEAT_OK`（心跳確認）和 `NO_REPLY`（靜默回覆）。

### 3.1 核心常數

```typescript
// source-repo/src/auto-reply/tokens.ts
export const HEARTBEAT_TOKEN = "HEARTBEAT_OK";
export const SILENT_REPLY_TOKEN = "NO_REPLY";
```

### 3.2 靜默回覆偵測

```typescript
// source-repo/src/auto-reply/tokens.ts:32-42
export function isSilentReplyText(text?: string, token?: string): boolean {
  // 精確匹配：^\s*{token}\s*$
  // 使用記憶化的正則表達式
}

// source-repo/src/auto-reply/tokens.ts:46-72
export function isSilentReplyEnvelopeText(text?: string, token?: string): boolean {
  // 偵測 JSON 格式：{"action":"NO_REPLY"}
  // 驗證 JSON 結構的完整性
}

// source-repo/src/auto-reply/tokens.ts:74-79
export function isSilentReplyPayloadText(text?: string, token?: string): boolean {
  // 合併檢查：文字格式或 JSON 信封格式
}
```

### 3.3 Token 剝離

```typescript
// source-repo/src/auto-reply/tokens.ts:86-88
export function stripSilentToken(text: string, token?: string): string {
  // 移除尾端的 Token + 可選的標點符號
}

// source-repo/src/auto-reply/tokens.ts:92-126
export function stripLeadingSilentToken(text: string, token?: string): string {
  // 移除前端的 Token
  // 使用 Unicode 屬性跳脫（\p{L}\p{N}）偵測 Token 黏附到文字的情況
  // 正則：^\s*(?:{token}\s+)*{token}(?=[\p{L}\p{N}])
}
```

### 3.4 串流前綴偵測

```typescript
// source-repo/src/auto-reply/tokens.ts:132-179
export function isSilentReplyPrefixText(text?: string, token?: string): boolean {
  // 處理串流中的不完整 Token 片段
  // "NO_REPLY" 可能以 "NO"、"NO_"、"NO_RE" 等形式出現
  // 防護：排除自然的大寫文字（如 "NONE"、"NOTIFY"）
  // 僅在裸 "NO" 時允許匹配（因為 NO_REPLY 可能從 "NO" 開始串流）
}
```

這是一個特別精巧的設計——在串流（streaming）場景中，模型可能逐字輸出 `NO_REPLY`，此時系統需要在看到 `NO` 時就判斷是否可能是靜默回覆。但 `NO` 也可能是正常英文單字的開頭，因此函式包含了一系列防護措施（第 155-167 行），排除如 `NONE`、`NOTHING`、`NOTIFY` 等自然文字。

### 3.5 效能最佳化：正則快取

```typescript
// source-repo/src/auto-reply/tokens.ts:6-19
const silentExactRegexByToken = new Map<string, RegExp>();
const silentTrailingRegexByToken = new Map<string, RegExp>();
const silentLeadingAttachedRegexByToken = new Map<string, RegExp>();
const silentLeadingRegexByToken = new Map<string, RegExp>();
```

所有正則表達式都透過 `Map` 進行記憶化快取——以 Token 字串為鍵，編譯後的 `RegExp` 物件為值。這避免了每次呼叫都重新編譯正則表達式的開銷，對於高頻呼叫路徑尤其重要。

---

## 4. 思考模式（Thinking Mode）

**檔案**：`thinking.ts`（132 行）、`thinking.shared.ts`（244 行）

思考模式控制 AI 模型的推理深度，從完全關閉到最高強度的思考。

### 4.1 思考等級（ThinkLevel）

```typescript
// source-repo/src/auto-reply/thinking.shared.ts
export type ThinkLevel = "off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "adaptive";
```

`normalizeThinkLevel()` 函式支援多種別名對映：
- `"on"` → `"low"`
- `"min"` → `"minimal"`
- `"xhigh"` / `"extrahigh"` → `"xhigh"`
- `"auto"` → `"adaptive"`

### 4.2 相關等級型別

```typescript
// source-repo/src/auto-reply/thinking.shared.ts
export type VerboseLevel = "off" | "on" | "trace";
export type TraceLevel = "off" | "on";
export type NoticeLevel = "off" | "on";
export type ElevatedLevel = "off" | "on" | "auto";
export type ElevatedMode = "off" | "enabled" | "auto";
export type ReasoningLevel = "off" | "low" | "medium" | "high";
export type UsageDisplayLevel = "off" | "on" | "cost";
```

每個等級型別都有對應的 `normalize*()` 函式，將使用者輸入的各種拼寫和別名正規化為標準值。

### 4.3 外掛驅動的能力偵測

```typescript
// source-repo/src/auto-reply/thinking.ts:43-61
export function isBinaryThinkingProvider(provider?: string, model?: string): boolean {
  // 查詢外掛系統：resolveProviderBinaryThinking()
  // 某些供應商只支援開/關兩種思考模式（無中間等級）
}

// source-repo/src/auto-reply/thinking.ts:63-83
export function supportsXHighThinking(provider?: string, model?: string): boolean {
  // 查詢外掛系統：是否支援 xhigh（超高強度）思考
}

// source-repo/src/auto-reply/thinking.ts:110-131
export function resolveThinkingDefaultForModel(params: {
  provider?: string;
  model?: string;
  catalog?: ThinkingCatalogEntry[];
}): ThinkLevel {
  // 1. 在 catalog 中尋找匹配的 provider/model 條目
  // 2. 將推理資料傳遞給外掛決策函式
  // 3. 回退到 resolveThinkingDefaultForModelFallback()
}
```

所有能力偵測都委派給外掛系統——這確保了當新的 AI 供應商或模型被加入時，不需要修改核心程式碼。

### 4.4 等級列表與格式化

```typescript
// source-repo/src/auto-reply/thinking.ts:85-108
export function listThinkingLevels(provider?, model?): ThinkLevel[] {
  // 取得基礎等級列表
  // 若支援 xhigh，則在最後一個元素前插入 "xhigh"
}

export function listThinkingLevelLabels(provider?, model?): string[] {
  // 二元供應商：["off", "on"]
  // 其他：回退到標準等級列表
}

export function formatThinkingLevels(provider?, model?, separator?): string {
  // 以分隔符連接等級標籤
}
```

---

## 5. 群組啟動（Group Activation）

**檔案**：`group-activation.ts`（38 行）

控制 Agent 在群組聊天中何時應該回應。

```typescript
// source-repo/src/auto-reply/group-activation.ts
export type GroupActivationMode = "mention" | "always";

// 正規化啟動模式
export function normalizeGroupActivation(raw?: string): GroupActivationMode | undefined {
  // 轉換為小寫後驗證 "mention" 或 "always"
}

// 解析啟動命令
export function parseActivationCommand(raw?: string): {
  hasCommand: boolean;
  mode?: GroupActivationMode;
} {
  // 支援兩種語法：
  //   /activation mention
  //   /activation: mention（冒號可選）
  // 正規化命令體後使用正則匹配
}
```

兩種啟動模式：
- **`"always"`**：永遠回應（適合專用的 Agent 頻道）
- **`"mention"`**：僅在被 @提及 時回應（適合共用群組）

---

## 6. 媒體註解（Media Notes）

**檔案**：`media-note.ts`（184 行）

媒體註解模組為 Agent 的上下文生成媒體附件的文字摘要，讓 AI 模型知道訊息附帶了哪些檔案。

### 6.1 媒體項目格式化

```typescript
// source-repo/src/auto-reply/media-note.ts:15-32
function formatMediaAttachedLine(params): string {
  // 格式：[media attached: path (type) | url]
}
```

### 6.2 音頻轉錄偵測

```typescript
// source-repo/src/auto-reply/media-note.ts:35-48
const AUDIO_EXTENSIONS = new Set([
  // 音頻檔案副檔名集合（用於轉錄偵測）
]);

// source-repo/src/auto-reply/media-note.ts:50-61
function isAudioPath(path: string): boolean {
  // 檢查檔案路徑是否為音頻副檔名
}

// source-repo/src/auto-reply/media-note.ts:67-101
function collectTranscribedAudioAttachmentIndices(ctx: MsgContext): Set<number> {
  // 從 ctx.MediaUnderstanding 尋找 kind === "audio.transcription"
  // 從 ctx.MediaUnderstandingDecisions 尋找成功的音頻能力結果
  // 回傳已成功轉錄的音頻附件索引集合
}
```

### 6.3 主要建構函式

```typescript
// source-repo/src/auto-reply/media-note.ts:103-183
export function buildInboundMediaNote(ctx: MsgContext): string | undefined {
  // 1. 從 ctx.MediaPaths 陣列或回退 ctx.MediaPath 收集路徑
  // 2. 取得已轉錄的音頻索引
  // 3. 過濾附件：
  //    - 剝離已轉錄的音頻（因為轉錄文字已在上下文中）
  //    - 保留圖片/影片描述（有損壓縮，需要原始媒體）
  //    - 依 MIME 類型和檔案副檔名判斷
  // 4. 格式化輸出：
  //    - 單一附件：單行格式
  //    - 多個附件：多行列表格式
}
```

關鍵設計決策：已成功轉錄的音頻會從媒體註解中移除，因為轉錄文字已經作為 `Transcript` 欄位存在於 `MsgContext` 中。但如果轉錄失敗，音頻仍會保留在媒體註解中，讓 Agent 知道有音頻附件存在。

---

## 7. 模型選擇（Model Runtime）

**檔案**：`model.ts`（52 行）、`model-runtime.ts`（98 行）

### 7.1 模型指令提取

```typescript
// source-repo/src/auto-reply/model.ts:17-50
export function extractModelDirective(
  body?: string,
  options?: { aliases?: string[] }
): {
  cleaned: string;         // 移除指令後的清理文字
  rawModel?: string;       // 原始模型參考
  rawProfile?: string;     // 認證設定檔
  hasDirective: boolean;   // 是否找到指令
} {
  // 主要正則：/model 後接可選的模型參考
  //   模型參考格式：英數字、:、.、@、-、/
  //   範例：/model openai/gpt-4、/model claude-3
  // 別名匹配（若無主要指令且提供了別名列表）
  // 使用 splitTrailingAuthProfile() 分離模型和認證設定檔
}
```

### 7.2 模型參考格式化

```typescript
// source-repo/src/auto-reply/model-runtime.ts:7-24
export function formatProviderModelRef(providerRaw: string, modelRaw: string): string {
  // 若無供應商 → 僅回傳模型
  // 移除模型中的冗餘供應商前綴
  // 避免 "openai/openai/gpt-4" 的重複
}

// source-repo/src/auto-reply/model-runtime.ts:32-45
function normalizeModelWithinProvider(provider: string, model: string): string {
  // 從模型名稱中剝離供應商前綴（若冗餘）
}

// source-repo/src/auto-reply/model-runtime.ts:47-72
function normalizeModelRef(raw: string, options?: {
  provider?: string;
  parseEmbeddedProvider?: boolean;
}): { provider?: string; model: string; label: string } {
  // 解析原始模型字串
  // 偵測嵌入式供應商語法（provider/model，含 '/' 分隔符）
  // 正規化並生成標籤
}
```

### 7.3 選定與實際模型解析

```typescript
// source-repo/src/auto-reply/model-runtime.ts:74-97
export function resolveSelectedAndActiveModel(params): {
  selected: { provider?: string; model: string; label: string };
  active: { provider?: string; model: string; label: string };
  activeDiffers: boolean;  // 實際使用的模型是否與選定的不同
} {
  // 正規化選定的模型
  // 從會話條目中正規化實際使用的模型
  // 偵測兩者是否不同（例如因回退而改變）
}
```

`activeDiffers` 旗標在「回退」（fallback）場景中特別重要——當選定的模型不可用時，系統可能會切換到替代模型，此旗標讓 `/status` 命令能向使用者顯示實際使用的模型。

---

## 8. 技能指令（Skill Commands）

**檔案**：`skill-commands.ts`（135 行）、`skill-commands-base.ts`（100 行）

### 8.1 保留命令名稱

```typescript
// source-repo/src/auto-reply/skill-commands-base.ts:1-100
export function listReservedChatSlashCommandNames(extraNames?: string[]): Set<string> {
  // 收集所有內建命令的名稱（原生名稱 + 文字別名）
  // 用於防止技能命令與內建命令衝突
}
```

### 8.2 技能命令調用解析

```typescript
// source-repo/src/auto-reply/skill-commands-base.ts
export function resolveSkillCommandInvocation(params: {
  commandBodyNormalized: string;
  skillCommands: SkillCommandSpec[];
}): { command: SkillCommandSpec; args?: string } | null {
  // 支援兩種語法：
  //   /skill <name> [args]   — 通用技能調用
  //   /<skillName> [args]     — 直接技能命令
  // 空格/底線處理（模糊匹配）
  // 匹配 skill.name 或 skill.skillName 欄位
}
```

### 8.3 工作空間技能命令聚合

```typescript
// source-repo/src/auto-reply/skill-commands.ts:22-42
export function listSkillCommandsForWorkspace(params): SkillCommandSpec[] {
  // 為單一工作空間建構技能命令規格
  // 傳遞遠端資格和保留名稱
}

// source-repo/src/auto-reply/skill-commands.ts:60-130
export function listSkillCommandsForAgents(params): SkillCommandSpec[] {
  // 跨多個 Agent 聚合技能命令
  // 技能篩選器合併策略：
  //   undefined = 無限制（無允許清單）
  //   [] = 明確禁止所有技能
  //   非空 = 取聯集
  // 以 fs.realpathSync() 去重工作空間（處理符號連結）
  // 追蹤已使用的命令名稱以避免重複
}
```

工作空間去重使用了 `fs.realpathSync()` 來解析符號連結——這處理了同一個實體目錄可能透過不同路徑被多個 Agent 引用的情況。

---

## 9. 指令認證與控制

### 9.1 指令認證（command-auth.ts，731 行）

這是整個自動回覆系統中最複雜的模組之一，實作了一套多階段的命令授權系統。

```typescript
// source-repo/src/auto-reply/command-auth.ts
export type CommandAuthorization = {
  providerId?: string;            // 解析出的供應商 ID
  ownerList: string[];            // 擁有者清單
  senderId?: string;              // 解析出的寄件人 ID
  senderIsOwner: boolean;         // 寄件人是否為擁有者
  isAuthorizedSender: boolean;    // 寄件人是否被授權
  from?: string;                  // 原始 From 欄位
  to?: string;                    // 原始 To 欄位
};
```

#### 階段一：供應商解析（第 56-135 行）

```typescript
function resolveProviderFromContext(ctx, cfg): string | undefined {
  // 多策略偵測（依優先順序）：
  // 1. 明確通道（Surface、OriginatingChannel、Provider）
  // 2. 從 From/To 欄位推斷（以 ':' 分隔）
  // 3. 通道外掛探測（probeInferredProviders）
}
```

#### 階段二：AllowFrom 解析（第 137-260 行）

```typescript
function resolveProviderAllowFrom(params): ProviderAllowFromResolution {
  // 呼叫外掛的 resolveAllowFrom()
  // 回退機制：外掛錯誤時使用日誌記錄並繼續
}
```

萬用字元語義：
- `"*"` = 允許所有人
- 空或未設定 = 回退到通道設定

#### 階段三：擁有者授權（第 270-428 行）

```typescript
function resolveOwnerAllowFromList(cfg): string[] {
  // 解析通道前綴的擁有者（如 "discord:@user"）
}

function resolveCommandsAllowFromList(cfg): string[] {
  // 獨立的命令範圍 allowFrom 設定
}

function resolveOwnerAuthorizationState(params): OwnerAuthorizationState {
  // 合併設定擁有者 allowFrom + 上下文擁有者 allowFrom
  // 計算擁有者允許清單
}
```

#### 階段四：寄件人解析（第 430-528 行）

```typescript
function resolveSenderCandidates(params): string[] {
  // 建構候選寄件人清單
  // 外掛偏好 E164 格式（電話號碼）勝過 ID
  // DM 場景中以 From 作為寄件人回退
  // 正規化每個候選人
}
```

#### 主函式：resolveCommandAuthorization（第 626-730 行）

```typescript
export function resolveCommandAuthorization(params: {
  ctx: MsgContext;
  cfg: OpenClawConfig;
  commandAuthorized?: boolean;
}): CommandAuthorization {
  // 1. 解析供應商
  // 2. 取得供應商外掛
  // 3. 建構 allowFrom 解析
  // 4. 解析擁有者狀態
  // 5. 解析寄件人候選人
  // 6. 計算授權旗標：
  //    - senderIsOwnerByIdentity：寄件人在擁有者清單中
  //    - senderIsOwnerByScope：內部通道 + operator.admin 範圍
  //    - senderIsOwner：從身份/範圍/萬用字元計算
  //    - isOwnerForCommands：若設定要求，強制擁有者身份
  // 7. 回傳 CommandAuthorization 結構
}
```

### 9.2 指令偵測（command-detection.ts，97 行）

```typescript
// source-repo/src/auto-reply/command-detection.ts
export function hasControlCommand(text?, cfg?, options?): boolean {
  // 精確命令偵測：正規化文字 → 剝離後設資料 → 比對註冊表
  // 支援帶引數的命令（若 acceptsArgs=true 且後跟空白）
}

export function isControlCommandMessage(text?, cfg?, options?): boolean {
  // 擴展偵測：hasControlCommand() + isAbortTrigger()
}

export function hasInlineCommandTokens(text?): boolean {
  // 粗略偵測：正則 (?:^|\s)[/!][a-z]（大小寫不敏感）
  // 容忍誤報以避免漏報
}

export function shouldComputeCommandAuthorized(text?, cfg?, options?): boolean {
  // 閘道函式：只在偵測到控制命令或行內 Token 時才計算授權
  // 最佳化：避免對無命令訊息執行昂貴的授權計算
}
```

`hasInlineCommandTokens()` 有意設計為「寧可誤報，不可漏報」——它使用簡單的正則表達式 `(?:^|\s)[/!][a-z]` 來偵測可能的命令語法，然後由更精確的 `hasControlCommand()` 進行最終判斷。

---

## 10. 工具元資料

**檔案**：`tool-meta.ts`（145 行）

格式化工具使用資訊，用於 UI 顯示。

```typescript
// source-repo/src/auto-reply/tool-meta.ts
export function shortenPath(p: string): string {
  // 縮短 Home 目錄路徑（如 /home/user → ~）
}

export function shortenMeta(meta: string): string {
  // 在字串中縮短 Home 目錄路徑
}

export function formatToolAggregate(toolName?, metas?, options?): string {
  // 1. 解析工具顯示（emoji + 標籤）
  // 2. 依目錄分組後設資料
  // 3. 使用大括號語法收合同目錄的檔案名稱
  //    例如：src/{auth.ts,user.ts}
  // 4. 連接各區段
}

export function formatToolPrefix(toolName?, meta?): string {
  // 格式化工具名稱 + 可選後設資料
}
```

對於 `exec`/`bash` 類型的工具，有特殊處理：

```typescript
// source-repo/src/auto-reply/tool-meta.ts:79-95
function formatMetaForDisplay(meta, options): string {
  // 分離旗標（elevated、pty）和命令體
  // 使用 '·' 分隔符解析旗標
}

// source-repo/src/auto-reply/tool-meta.ts:97-115
function splitExecFlags(meta): { flags: string[]; body: string } {
  // 解析 '·' 分隔的後設資料旗標
}
```

---

## 11. 回退狀態

**檔案**：`fallback-state.ts`（185 行）

當選定的 AI 模型不可用時（速率限制、超時、過載等），系統會切換到替代模型。此模組追蹤這些回退的狀態轉換。

### 11.1 暫態錯誤偵測

```typescript
// source-repo/src/auto-reply/fallback-state.ts:11-14
const TRANSIENT_FALLBACK_REASONS = new Set([/* 速率限制、超時、過載 */]);
const TRANSIENT_ERROR_DETAIL_HINT_RE = /timeout|quota/i;  // 暫態錯誤提示
```

### 11.2 回退原因格式化

```typescript
// source-repo/src/auto-reply/fallback-state.ts:45-62
export function formatFallbackAttemptReason(attempt): string {
  // 優先順序：錯誤預覽 > 原因 > 錯誤碼 > HTTP 狀態 > 錯誤文字
  // 截斷至 80 字元（加省略號）
}

// source-repo/src/auto-reply/fallback-state.ts:68-75
export function buildFallbackReasonSummary(attempts): string {
  // 第一次嘗試的原因 + "(+N more attempts)"
}
```

### 11.3 回退通知

```typescript
// source-repo/src/auto-reply/fallback-state.ts:83-97
export function buildFallbackNotice(params): string | null {
  // 格式：↪️ Model Fallback: {active} (selected {selected}; {reason})
  // 選定 = 實際時回傳 null（無需通知）
}

// source-repo/src/auto-reply/fallback-state.ts:99-110
export function buildFallbackClearedNotice(params): string {
  // 回退已清除的通知
}
```

### 11.4 狀態轉換追蹤

```typescript
// source-repo/src/auto-reply/fallback-state.ts:133-184
export type ResolvedFallbackTransition = {
  isActive: boolean;       // 回退是否正在生效
  isTransitioned: boolean; // 是否剛發生狀態轉換
  isCleared: boolean;      // 是否剛從回退恢復
  nextState: unknown;      // 下一個持久化狀態
};

export function resolveFallbackTransition(params): ResolvedFallbackTransition {
  // 1. 讀取前一個狀態
  // 2. 計算回退啟用/轉換/清除旗標
  // 3. 計算下一個狀態（selectedModel、activeModel、reason）
  // 4. 偵測狀態變化
}
```

---

## 12. 防抖動（Inbound Debounce）

**檔案**：`inbound-debounce.ts`（236 行）

當使用者快速連續發送多則訊息時，防抖動模組將它們合併為一次處理，避免 Agent 對每則訊息獨立回覆。

### 12.1 防抖動時間解析

```typescript
// source-repo/src/auto-reply/inbound-debounce.ts:21-34
function resolveInboundDebounceMs(params): number {
  // 解析瀑布：通道覆寫 > 按通道設定 > 基本值 > 0（無防抖動）
}
```

### 12.2 防抖動器建立

```typescript
// source-repo/src/auto-reply/inbound-debounce.ts
export type InboundDebounceCreateParams<T> = {
  // 設定參數
};

export function createInboundDebouncer<T>(params): {
  enqueue: (item: T, key: string) => Promise<void>;
  flushKey: (key: string) => Promise<void>;
} {
  // 建立防抖動器實例
}
```

### 12.3 核心邏輯

```typescript
// source-repo/src/auto-reply/inbound-debounce.ts:165-232（enqueue 函式）
// 非防抖動路徑：立即刷新
// 現有緩衝區路徑：追加 + 重新排程
// 飽和回退路徑：以鍵為單位的立即執行
// 新緩衝區路徑：建立帶保留任務的新緩衝區
```

關鍵設計模式：

- **鍵序執行**（Key-ordered execution，第 84-95 行）：使用 Promise 鏈確保同一鍵的項目依序處理
- **緩衝區保留**（Buffer reservation，第 97-116 行）：在強制刷新待處理緩衝區之前先保留執行槽位，防止「後來居上」（overtaking）
- **飽和處理**（Saturation handling，第 206-212 行）：當防抖動 Map 已滿時，回退到以鍵為單位的立即執行
- **每項目防抖動時間**（Per-item debounce time，第 63-69 行）：允許每個項目有不同的防抖動間隔

---

## 13. 發送策略

**檔案**：`send-policy.ts`（48 行）

控制 Agent 是否被允許發送回覆。

```typescript
// source-repo/src/auto-reply/send-policy.ts
export type SendPolicyOverride = "allow" | "deny";

export function normalizeSendPolicyOverride(raw?: string): SendPolicyOverride | undefined {
  // "allow" / "on"  → "allow"
  // "deny"  / "off" → "deny"
  // 其他 → undefined
}

export function parseSendPolicyCommand(raw?: string): {
  hasCommand: boolean;
  mode?: SendPolicyOverride | "inherit";
} {
  // 解析 /send 命令
  // "inherit" / "default" / "reset" → "inherit"（恢復到預設策略）
  // 其他 → normalizeSendPolicyOverride()
}
```

---

## 14. 狀態監控

**檔案**：`status.ts`（946 行）、`status.runtime.ts`（2 行）

這是輔助模組中最龐大的檔案，負責建構使用者執行 `/status` 命令時看到的完整狀態報告。

### 14.1 Token 格式化

```typescript
// source-repo/src/auto-reply/status.ts:198-213
function formatTokens(total?, contextTokens?): string {
  // 格式：total/context (pct%)
}

// source-repo/src/auto-reply/status.ts:215-240
function formatQueueDetails(params): string {
  // 格式化佇列深度、防抖動、容量、丟棄策略
}
```

### 14.2 用量讀取

```typescript
// source-repo/src/auto-reply/status.ts:242-317
function readUsageFromSessionLog(params): {
  input?: number;
  output?: number;
  cacheRead?: number;
  cacheWrite?: number;
  total?: number;
} {
  // 從會話記錄檔中讀取 Token 使用量
  // 解析會話檔案路徑
  // 呼叫 readLatestSessionUsageFromTranscript()
  // 提取 input、output、cache read/write、total
}
```

### 14.3 快取與媒體格式化

```typescript
// source-repo/src/auto-reply/status.ts:328-356
function formatCacheLine(params): string {
  // 格式化快取命中率 + 新 Token
}

// source-repo/src/auto-reply/status.ts:358-402
function formatMediaUnderstandingLine(params): string {
  // 格式化媒體決策結果（音頻、圖片、影片）
}
```

### 14.4 主要狀態建構器

```typescript
// source-repo/src/auto-reply/status.ts
export function buildStatusMessage(args: StatusArgs): string {
  // 第 421-499 行：模型選擇與正規化
  //   解析選定/實際模型，處理嵌入式供應商的傳統會話
  
  // 第 500-557 行：Token 用量聚合
  //   偏好記錄用量（若較大）
  
  // 第 559-641 行：上下文窗口解析
  //   選定 vs 實際上下文 Token
  //   持久化上下文 vs 設定上限
  //   複雜的回退瀑布邏輯
  
  // 第 643-715 行：狀態選項格式化
  //   Think/Verbose/Fast/Reasoning/Elevated 等級
  //   執行時期標籤
  //   選項行組裝
  
  // 第 722-809 行：模型認證與回退狀態
  //   認證模式偵測
  //   回退通知生成
  //   成本估算（僅 API 金鑰認證）
  
  // 第 839-859 行：最終訊息組裝
  //   過濾假值行後以換行連接
  //   包含：版本、模型、用量、上下文、會話、啟動、選項
}
```

### 14.5 工具訊息

```typescript
// source-repo/src/auto-reply/status.ts:893-945
export function buildToolsMessage(result, options?): string {
  // 依分類分組工具
  // 格式化精簡或詳細輸出
  // 包含設定檔和工具可用性注記
}
```

`StatusArgs`（第 76-103 行）是一個包含 20 多個欄位的綜合參數型別，涵蓋了建構狀態訊息所需的所有上下文：模型、會話、Token、佇列、通道、認證等。

---

## 15. 引用來源

| 來源 | 行數 | 說明 |
|------|------|------|
| `source-repo/src/auto-reply/chunk.ts:1-483` | 483 | 分塊處理（長度、換行、段落、Markdown） |
| `source-repo/src/auto-reply/heartbeat.ts:1-299` | 299 | 心跳任務管理與 Token 剝離 |
| `source-repo/src/auto-reply/heartbeat-filter.ts:1-97` | 97 | 心跳訊息過濾與配對移除 |
| `source-repo/src/auto-reply/heartbeat-reply-payload.ts:1-24` | 24 | 心跳回覆 Payload 解析 |
| `source-repo/src/auto-reply/tokens.ts:1-180` | 180 | 靜默/心跳 Token 偵測（含正則快取） |
| `source-repo/src/auto-reply/thinking.ts:1-132` | 132 | 思考等級解析（外掛驅動） |
| `source-repo/src/auto-reply/thinking.shared.ts:1-244` | 244 | 思考等級型別與正規化函式 |
| `source-repo/src/auto-reply/group-activation.ts:1-38` | 38 | 群組啟動模式解析 |
| `source-repo/src/auto-reply/media-note.ts:1-184` | 184 | 媒體附件註解建構 |
| `source-repo/src/auto-reply/model.ts:1-52` | 52 | 模型指令提取 |
| `source-repo/src/auto-reply/model-runtime.ts:1-98` | 98 | 模型參考格式化與選定/實際解析 |
| `source-repo/src/auto-reply/skill-commands.ts:1-135` | 135 | 技能命令聚合（工作空間去重） |
| `source-repo/src/auto-reply/skill-commands-base.ts:1-100` | 100 | 保留命令名稱與技能調用解析 |
| `source-repo/src/auto-reply/command-auth.ts:1-731` | 731 | 命令授權（四階段：供應商→AllowFrom→擁有者→寄件人） |
| `source-repo/src/auto-reply/command-detection.ts:1-97` | 97 | 命令偵測（精確 + 粗略） |
| `source-repo/src/auto-reply/command-detection.runtime-types.ts:1-15` | 15 | 命令偵測 Runtime 型別 |
| `source-repo/src/auto-reply/tool-meta.ts:1-145` | 145 | 工具元資料格式化 |
| `source-repo/src/auto-reply/fallback-state.ts:1-185` | 185 | 回退狀態轉換追蹤 |
| `source-repo/src/auto-reply/inbound-debounce.ts:1-236` | 236 | 入站訊息防抖動 |
| `source-repo/src/auto-reply/send-policy.ts:1-48` | 48 | 發送策略覆寫 |
| `source-repo/src/auto-reply/status.ts:1-946` | 946 | 狀態訊息建構（Token、模型、認證、成本） |
| `source-repo/src/auto-reply/status.runtime.ts:1-2` | 2 | 狀態 Runtime 匯出 |
| `source-repo/src/auto-reply/types.ts:1-10` | 10 | 型別重新匯出聚合 |
