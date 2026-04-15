# 訊息派發與回覆引擎（Message Dispatch & Reply Engine）

> **版本**：OpenClaw v2026.4.12
> **範圍**：`source-repo/src/auto-reply/dispatch.ts`、`dispatch-dispatcher.ts`、`reply.ts`、`templating.ts`、`envelope.ts`、`commands-registry*.ts`、`reply/` 子目錄

---

## 開頭摘要

OpenClaw 的自動回覆子系統（Auto-Reply subsystem）是整個平台訊息處理流水線的核心。當一則訊息從任何通道（Discord、WhatsApp、Telegram、WebChat 等）抵達時，它首先被封裝為一個「訊息上下文」（MsgContext）——一個包含 70 多個欄位的結構體，攜帶了訊息內容、寄件人資訊、媒體附件、群組上下文、命令授權等所有必要資訊。隨後，`dispatch.ts` 中的派發函式接手：先透過 `finalizeInboundContext()` 將上下文定案（default-deny 安全策略，確保 `CommandAuthorized` 預設為 `false`），再透過 `withReplyDispatcher()` 管理派發器的完整生命週期，最後呼叫 `dispatchReplyFromConfig()` 將訊息交付給龐大的 `reply/` 子目錄進行實際回覆生成。在這個過程中，訊息會經過信封格式化（envelope formatting）以加入通道、時間戳、寄件人等後設資料，並通過命令註冊表（command registry）判斷是否為斜線命令。整個系統以模組化、延遲載入、外掛驅動的方式建構，使得單一程式碼庫即可支援數十種通道與數百種命令。

---

## 1. 目錄概覽

`src/auto-reply/` 目錄包含約 90 個以上的非測試 TypeScript 檔案，而 `reply/` 子目錄更擁有 200 多個檔案（其中非測試檔案約 211 個，測試檔案約 128 個，共 339 個檔案加上 6 個子目錄）。以下是核心模組的快速索引：

### 1.1 頂層核心模組

| 檔案 | 行數 | 角色 |
|------|------|------|
| `dispatch.ts` | 82 | 派發入口，匯出三個主要派發函式 |
| `dispatch-dispatcher.ts` | 19 | `withReplyDispatcher()` 生命週期包裝器 |
| `reply.ts` | 12 | 桶檔案（Barrel file），從 `reply/` 子目錄重新匯出 |
| `templating.ts` | 258 | 定義 `MsgContext`、`TemplateContext`、`applyTemplate()` |
| `envelope.ts` | 267 | 信封格式化——為訊息加上通道、時間戳等前綴 |
| `commands-registry.ts` | 327 | 命令註冊表——命令探索、解析、選項解析 |
| `commands-registry-list.ts` | 62 | 命令清單與啟用檢查 |
| `commands-registry-normalize.ts` | 183 | 命令正規化與文字別名解析 |
| `commands-registry.data.ts` | 68 | 內建命令 + 動態 Dock 命令資料 |
| `commands-registry.types.ts` | 83 | 命令型別定義 |
| `commands-text-routing.ts` | 241 | 命令幫助訊息建構（分頁支援） |

### 1.2 reply/ 子目錄結構

```
reply/
├── agent-runner*.ts        — Agent 執行引擎（記憶體、Payload、會話重置等）
├── get-reply*.ts           — 回覆解析管線（核心入口）
├── dispatch-from-config.ts — 設定驅動的派發
├── session*.ts             — 會話管理（初始化、過期、分叉）
├── commands*.ts            — 所有斜線命令實作（96 個 commands-*.ts 檔案）
├── directive-handling*.ts  — 行內指令解析（[[...]] 語法）
├── queue/                  — 訊息佇列子目錄
├── commands-acp/           — ACP 特定命令
├── commands-subagents/     — 子代理管理命令
├── exec/                   — 執行相關工具
├── export-html/            — HTML 匯出功能
├── groups.ts               — 群組訊息處理
├── reply-delivery.ts       — 回覆交付至通道
├── block-reply-pipeline.ts — 區塊串流管線
└── test-fixtures/          — 測試資料
```

---

## 2. 訊息上下文（MsgContext）

`MsgContext` 是整個自動回覆系統的「血液」——每一則進站訊息都會被封裝為這個型別，攜帶所有必要的後設資料貫穿整個處理流水線。它定義在 `templating.ts`（第 24-198 行），包含 70 多個欄位。

### 2.1 型別定義總覽

```typescript
// source-repo/src/auto-reply/templating.ts:24-198
export type MsgContext = {
  // === 訊息內容 ===
  Body?: string;           // 原始訊息內容
  BodyForAgent?: string;   // 經處理的 Agent 可用內容
  RawBody?: string;        // 完全未處理的原始內容
  CommandBody?: string;    // 命令專用的內容
  BodyForCommands?: string;// 命令系統使用的內容

  // === 寄收件資訊 ===
  From?: string;
  To?: string;
  SessionKey?: string;
  AccountId?: string;
  ParentSessionKey?: string;

  // === 訊息識別碼 ===
  MessageSid?: string;
  MessageSidFull?: string;
  MessageSids?: string[];
  MessageSidFirst?: string;
  MessageSidLast?: string;

  // === 回覆串接 ===
  ReplyThreading?: unknown;
  ReplyToId?: string;
  ReplyToBody?: string;
  ReplyToSender?: string;
  ReplyToIsQuote?: boolean;
  RootMessageId?: string;
  ThreadParentId?: string;

  // === 轉發資訊 ===
  ForwardedFrom?: string;
  ForwardedFromType?: string;
  ForwardedFromId?: string;
  ForwardedFromUsername?: string;
  ForwardedFromTitle?: string;
  ForwardedFromSignature?: string;
  ForwardedFromChatType?: string;
  ForwardedFromDate?: string;

  // === 討論串 ===
  ThreadStarterBody?: string;
  ThreadHistoryBody?: string;
  IsFirstThreadTurn?: boolean;
  ThreadLabel?: string;

  // === 媒體資訊 ===
  MediaPath?: string;
  MediaUrl?: string;
  MediaType?: string;
  MediaDir?: string;
  MediaPaths?: string[];
  MediaUrls?: string[];
  MediaTypes?: string[];
  MediaRemoteHost?: string;
  Transcript?: string;
  MediaUnderstanding?: unknown;
  MediaUnderstandingDecisions?: unknown;
  LinkUnderstanding?: unknown;

  // === 群組上下文 ===
  ChatType?: string;
  ConversationLabel?: string;
  GroupSubject?: string;
  GroupChannel?: string;
  GroupSpace?: string;
  GroupMembers?: string;
  GroupSystemPrompt?: string;

  // === 寄件人詳情 ===
  SenderName?: string;
  SenderId?: string;
  SenderUsername?: string;
  SenderTag?: string;
  SenderE164?: string;

  // === 平台資訊 ===
  Provider?: string;
  Surface?: string;
  BotUsername?: string;
  WasMentioned?: boolean;
  GatewayClientScopes?: unknown;

  // === 命令授權 ===
  CommandArgs?: string;
  CommandAuthorized?: boolean;
  CommandSource?: string;
  CommandTargetSessionKey?: string;
  AcpDispatchTailAfterReset?: string;

  // === 路由 ===
  OriginatingChannel?: OriginatingChannelType;
  OriginatingTo?: string;
  ExplicitDeliverRoute?: string;

  // === 進階欄位 ===
  NativeChannelId?: string;
  NativeDirectUserId?: string;
  IsForum?: boolean;
  TopicRequiredButMissing?: boolean;
  MessageThreadId?: string;
  Prompt?: string;
  MaxChars?: number;
  InputProvenance?: unknown;
  UntrustedContext?: unknown;
  HookMessages?: unknown;
  OutputDir?: string;
  OutputBase?: string;
  // ... 更多欄位
};
```

### 2.2 OriginatingChannelType 品牌型別

```typescript
// source-repo/src/auto-reply/templating.ts:10-12
export type OriginatingChannelType = string & { __brand: "OriginatingChannel" };
```

這是一個「品牌型別」（Branded Type）——在 TypeScript 的結構型別系統中，透過交叉一個虛擬的 `__brand` 屬性來防止普通字串被意外當作通道路由識別符使用。這確保了通道路由的型別安全性。

### 2.3 StickerContextMetadata

```typescript
// source-repo/src/auto-reply/templating.ts
export type StickerContextMetadata = {
  emoji?: string;
  animations?: unknown;
  fileIds?: unknown;
  description?: string;
};
```

專為 Telegram 貼圖設計的後設資料結構，攜帶對應的表情符號（emoji）、動畫資訊、檔案識別碼和文字描述。

### 2.4 FinalizedMsgContext — default-deny 安全模型

```typescript
// source-repo/src/auto-reply/templating.ts
export type FinalizedMsgContext = Omit<MsgContext, "CommandAuthorized"> & {
  CommandAuthorized: boolean;  // 保證為 boolean，永不為 undefined
};
```

這是整個安全模型的關鍵設計：原始 `MsgContext` 的 `CommandAuthorized` 欄位是 `boolean | undefined`，代表「尚未決定」。當訊息通過 `finalizeInboundContext()` 後，`undefined` 會被強制轉換為 `false`——這就是 **default-deny** 策略。任何未經明確授權的訊息都不會被允許執行命令。

### 2.5 MsgContext 欄位分類深入解析

為了更清晰地理解 `MsgContext` 的設計邏輯，我們可以將 70 多個欄位分為以下幾個功能群組：

#### 訊息內容群組

這組欄位代表了同一則訊息的多種「視角」——不同的處理階段需要不同形式的訊息內容：

- **`Body`**：原始訊息內容，是使用者實際發送的文字
- **`BodyForAgent`**：經過預處理的版本，供 AI Agent 消費——可能移除了命令前綴、剝離了控制 Token 等
- **`RawBody`**：完全未經任何處理的原始內容，包含所有原始標記
- **`CommandBody`**：命令系統專用的內容，已經過正規化以便命令解析
- **`BodyForCommands`**：另一個命令導向的內容變體

這種多重內容欄位的設計避免了「在處理管線中反覆修改同一個字串」的反模式——每個處理階段都能取得適合自己的內容版本，而不必擔心其他階段的修改影響自己。

#### 寄收件與會話識別群組

- **`From`** / **`To`**：通道級的寄收件識別符（如 Discord 使用者 ID、WhatsApp 電話號碼等）
- **`SessionKey`**：會話金鑰，用於在會話存儲（session store）中查找會話狀態
- **`ParentSessionKey`**：父會話金鑰，用於會話分叉（fork）場景
- **`AccountId`**：帳號識別符，用於多帳號設定中區分不同的 Agent 帳號

#### 回覆串接與轉發群組

OpenClaw 對回覆串接（reply threading）有完整的支援：

- **`ReplyToId`** / **`ReplyToBody`** / **`ReplyToSender`**：被回覆的訊息資訊
- **`ReplyToIsQuote`**：是否為引用回覆（某些平台區分回覆和引用）
- **`RootMessageId`** / **`ThreadParentId`**：討論串的根訊息和父訊息
- **`ThreadStarterBody`** / **`ThreadHistoryBody`**：討論串的起始訊息和歷史摘要
- **`IsFirstThreadTurn`**：是否為討論串的第一個回合

轉發訊息也被完整追蹤：`ForwardedFrom`、`ForwardedFromType`、`ForwardedFromId` 等欄位攜帶了原始來源的完整資訊，讓 Agent 能理解轉發內容的上下文。

#### 媒體群組

媒體處理是 OpenClaw 的一個重要特性，支援圖片、音頻、影片等多種媒體類型：

- **`MediaPath`** / **`MediaPaths`**：媒體檔案的本地路徑（單一或陣列）
- **`MediaUrl`** / **`MediaUrls`**：媒體的遠端 URL
- **`MediaType`** / **`MediaTypes`**：MIME 類型
- **`Transcript`**：音頻轉錄文字
- **`MediaUnderstanding`** / **`MediaUnderstandingDecisions`**：媒體理解結果和決策記錄
- **`LinkUnderstanding`**：連結內容理解

### 2.6 TemplateContext 與模板插值

```typescript
// source-repo/src/auto-reply/templating.ts
export type TemplateContext = MsgContext & {
  BodyStripped?: string;
  SessionId?: string;
  IsNewSession?: boolean;
};
```

`TemplateContext` 在 `MsgContext` 基礎上擴展了三個會話相關欄位，用於系統提示詞（system prompt）的模板插值：

- **`BodyStripped`**：剝離命令和指令後的純文字內容
- **`SessionId`**：唯一的會話識別符
- **`IsNewSession`**：布林值，標記這是否為一個全新的會話

模板插值的典型使用場景是系統提示詞（system prompt）。例如：

```
你正在與 {{SenderName}} 對話。
目前的群組主題是：{{GroupSubject}}
這是一個{{IsNewSession}}的會話。
```

```typescript
// source-repo/src/auto-reply/templating.ts:250-258
export function applyTemplate(template: string | undefined, ctx: TemplateContext): string {
  if (!template) return "";
  return template.replace(/\{\{\s*(\w+)\s*\}\}/g, (_, key) => {
    return formatTemplateValue(ctx[key as keyof TemplateContext]);
  });
}
```

`applyTemplate()` 函式實作了 `{{Placeholder}}` 語法的模板插值。正則表達式 `/\{\{\s*(\w+)\s*\}\}/g` 匹配雙花括號包裹的鍵名，其中 `\s*` 表示空白容錯——`{{ Body }}`、`{{Body}}`、`{{ Body}}` 都是合法的。

`formatTemplateValue()` 輔助函式（第 214-247 行）負責將 TypeScript 的各種型別安全地轉換為字串：

- `null` / `undefined` → 空字串（避免在模板中出現 "undefined"）
- 字串 → 直接傳回
- 數字、布林值、BigInt → 字串轉換
- Symbol、函式 → `toString()`
- 陣列 → 以逗號連接非空元素（過濾掉 null/undefined 元素）
- 物件 → 空字串（避免出現 "[object Object]"）

這個設計確保了模板插值永遠不會產生意外的輸出——無論 `MsgContext` 中的欄位是什麼型別或是否為空，結果都是一個可預期的字串。

---

## 3. 信封格式化（Envelope Formatting）

信封（Envelope）是 OpenClaw 為每則訊息加上的結構化前綴，讓 AI 模型能理解訊息的來源、時間和通道資訊。這個功能實作在 `envelope.ts`（267 行）。

### 3.1 核心函式

#### formatAgentEnvelope()

```typescript
// source-repo/src/auto-reply/envelope.ts:156-195
export function formatAgentEnvelope(params: AgentEnvelopeParams): string {
  // 1. 清理通道、寄件人、主機、IP 等標頭部分
  const channel = sanitizeEnvelopeHeaderPart(params.channel);
  const from = sanitizeEnvelopeHeaderPart(params.from);
  // ...

  // 2. 建構方括號標頭：[Channel From +elapsed Time]
  let header = `[${channel} ${from}`;
  if (params.previousTimestamp) {
    header += ` +${formatElapsed(elapsed)}`;
  }
  header += ` ${formatEnvelopeTimestamp(params.timestamp, options)}]`;

  // 3. 回傳：標頭 + 空格 + 訊息內容
  return `${header} ${params.body}`;
}
```

輸出格式範例：`[Discord Alice +5m Wed 15:30] Hello everyone`

這個格式讓 AI 模型能夠：
- 辨識訊息來自哪個通道（Discord、WhatsApp 等）
- 識別寄件人身份
- 感知時間流逝（`+5m` 代表距上一則訊息已過 5 分鐘）
- 理解當前時間（含星期縮寫，幫助模型理解時間概念）

#### formatInboundEnvelope()

```typescript
// source-repo/src/auto-reply/envelope.ts:197-228
export function formatInboundEnvelope(params): string {
  // 判斷是直接訊息還是群組訊息
  // 群組訊息包含寄件人標籤
  // 自己的訊息加上 (self) 前綴
  // 委派給 formatAgentEnvelope()
}
```

此函式在 `formatAgentEnvelope()` 的基礎上增加了群組訊息的特殊處理——在群組中，需要為每則訊息標註寄件人身份。

#### formatInboundFromLabel()

```typescript
// source-repo/src/auto-reply/envelope.ts:230-251
export function formatInboundFromLabel(params): string {
  // 群組：包含標籤和 ID
  // 直接：包含標籤，條件性地包含 ID（當 ID 與標籤不同時）
}
```

#### formatThreadStarterEnvelope()

```typescript
// source-repo/src/auto-reply/envelope.ts:253-267
export function formatThreadStarterEnvelope(params): string {
  // 簡化版信封，用於討論串的起始訊息
  // 以作者（author）作為 from 欄位
}
```

### 3.2 安全性：sanitizeEnvelopeHeaderPart()

```typescript
// source-repo/src/auto-reply/envelope.ts:58-67
function sanitizeEnvelopeHeaderPart(part: string): string {
  return part
    .replace(/[\n\r]/g, " ")          // 移除換行
    .replace(/\[/g, "(")              // 中和左方括號
    .replace(/\]/g, ")")              // 中和右方括號
    .replace(/\s+/g, " ")            // 收合多餘空白
    .trim();
}
```

這個函式的安全用途值得特別注意：它將方括號 `[]` 替換為圓括號 `()`。這是為了防止「方括號注入」（bracket injection）——惡意使用者可能在名稱中嵌入 `]` 來提前結束信封標頭，從而偽造訊息來源。

### 3.3 時區解析

```typescript
// source-repo/src/auto-reply/envelope.ts:109-154
export function formatEnvelopeTimestamp(timestamp, options): string {
  // 1. 驗證時間戳存在且有效
  // 2. 正規化選項（使用預設值）
  // 3. 檢查 includeTimestamp 旗標
  // 4. 解析時區模式（UTC / local / IANA）
  // 5. 使用 Intl.DateTimeFormat 進行本地化格式化
  // 6. 前置星期縮寫（幫助 AI 模型理解時間）
}
```

時區解析支援三種模式：
- **UTC**：標準 UTC 時間
- **local**：伺服器本地時間
- **IANA 時區**：使用者指定的 IANA 時區名稱（如 `Asia/Taipei`）

時區設定來源為 `agents.defaults.envelopeTimezone` 設定項，透過 `resolveEnvelopeFormatOptions()` 解析。

### 3.4 設計理念：為什麼需要信封？

信封格式的存在有幾個重要的設計考量：

1. **時間感知**：AI 模型本身沒有時間概念。透過在每則訊息前加上時間戳和經過時間（`+5m`），模型能理解對話的時間脈絡——例如「上一則訊息是 5 分鐘前的」暗示使用者可能已經離開。

2. **多通道辨識**：同一個 Agent 可能同時在 Discord、WhatsApp 和 Telegram 上運行。信封中的通道標記讓模型能理解當前的互動環境，從而調整回應風格（例如在 SMS 中使用更簡短的回覆）。

3. **群組身份辨識**：在群組聊天中，多個使用者可能同時參與對話。信封中的寄件人標記讓模型能區分不同的說話者。

4. **安全防護**：`sanitizeEnvelopeHeaderPart()` 的方括號中和功能防止了一類「提示注入」攻擊——惡意使用者無法透過在名稱中嵌入 `]` 來偽造信封標頭。

5. **星期縮寫**：在時間戳前加上星期縮寫（如 `Wed`）是一個精巧的設計——研究表明 AI 模型對星期的理解比對日期的理解更直覺。

---

## 4. 命令系統（Command Registry）

命令系統是 OpenClaw 的互動式控制介面，讓使用者能夠透過斜線命令（`/command`）控制 Agent 的行為。它由多個檔案組成，形成一個完整的命令生命週期管理系統。

### 4.1 命令定義型別

```typescript
// source-repo/src/auto-reply/commands-registry.types.ts
export type ChatCommandDefinition = {
  key: string;                      // 內部鍵名
  description: string;              // 使用者可見的說明
  nativeName?: string;              // 平台原生命令名稱
  acceptsArgs?: boolean;            // 是否接受引數
  args?: CommandArgDefinition[];    // 引數定義陣列
  scope?: CommandScope;             // "text" | "native" | "both"
  category?: CommandCategory;       // 命令分類
  textAliases?: string[];           // 文字別名
  featureFlag?: string;             // 功能旗標
};

export type CommandScope = "text" | "native" | "both";
export type CommandCategory = "session" | "options" | "status" | "management" | "media" | "tools" | "docks";

export type CommandArgDefinition = {
  name: string;
  type?: CommandArgType;
  choices?: CommandArgChoice[] | CommandArgChoicesProvider;
  captureRemaining?: boolean;
  menu?: CommandArgMenuSpec;
};
```

### 4.2 命令資料來源（commands-registry.data.ts）

```typescript
// source-repo/src/auto-reply/commands-registry.data.ts (68 行)
export function getChatCommands(): ChatCommandDefinition[] {
  // 1. 從內建命令清單取得基底命令
  // 2. 從已載入的通道外掛取得動態 Dock 命令
  //    （支援 nativeCommands 能力的外掛）
  // 3. 合併並驗證完整命令註冊表
  // 4. 依外掛通道註冊表版本快取結果
}

export function getNativeCommandSurfaces(): Set<string> {
  // 回傳支援原生命令的介面集合
}
```

### 4.3 命令正規化（commands-registry-normalize.ts）

命令正規化是將使用者的原始輸入轉換為系統可理解的命令格式的過程：

```typescript
// source-repo/src/auto-reply/commands-registry-normalize.ts (183 行)

// 正規化命令內容
export function normalizeCommandBody(raw: string, options?: CommandNormalizeOptions): string {
  // 移除前導 '/'
  // 處理冒號格式命令（/cmd:args → /cmd args）
  // 解析 bot 提及（@botname）
}

// 命令偵測
export function getCommandDetection(cfg?: OpenClawConfig): CommandDetection {
  // 建構正則表達式模式（含記憶化快取）
  // 支援文字別名匹配
}

// 文字別名解析
export function maybeResolveTextAlias(raw: string, cfg?: OpenClawConfig): string | null {
  // 將文字別名轉換為正式命令名稱
  // 大小寫不敏感
}

// 完整命令解析
export function resolveTextCommand(raw: string, cfg?: OpenClawConfig): {
  command: ChatCommandDefinition;
  args?: string;
} | null {
  // 從正規化文字中解析命令 + 引數
  // 支援不接受引數的命令
}
```

### 4.4 命令清單與啟用檢查（commands-registry-list.ts）

```typescript
// source-repo/src/auto-reply/commands-registry-list.ts (62 行)
export function listChatCommands(params?: {
  skillCommands?: SkillCommandSpec[];
}): ChatCommandDefinition[] {
  // 建構技能命令定義（格式：skill:{skillName}）
  // 與內建命令合併
}

export function isCommandEnabled(cfg: OpenClawConfig, commandKey: string): boolean {
  // 檢查功能旗標：config、mcp、plugins、debug、bash
  // 透過 isCommandFlagEnabled 驗證
}

export function listChatCommandsForConfig(cfg: OpenClawConfig, params?: {
  skillCommands?: SkillCommandSpec[];
}): ChatCommandDefinition[] {
  // 過濾出在當前設定下已啟用的命令
}
```

### 4.5 命令註冊表主檔案（commands-registry.ts）

`commands-registry.ts`（327 行）是命令系統的中央樞紐，整合了上述所有子模組：

```typescript
// source-repo/src/auto-reply/commands-registry.ts

// 原生命令規格（用於通道外掛的原生命令介面）
export function listNativeCommandSpecs(cfg, params?): NativeCommandSpec[] {
  // 過濾出 scope !== "text" 的命令
  // 透過通道外掛解析平台特定命令名稱
}

// 命令文字建構
export function buildCommandText(command, args?): string {
  // 回傳 "/commandName args" 格式
}

// 引數解析
export function parseCommandArgs(definition, raw): CommandArgs {
  // 位置引數解析（parsePositionalArgs）
  // 尊重 captureRemaining 旗標（多字引數）
  // 支援格式化器（custom formatters）
}

// 引數選項解析
export function resolveCommandArgChoices(arg, context): ResolvedCommandArgChoice[] {
  // 支援靜態陣列和動態函式
  // 正規化為 { value, label } 格式
}

// 引數選單解析
export function resolveCommandArgMenu(definition, args, context): {
  arg: CommandArgDefinition;
  choices: ResolvedCommandArgChoice[];
  title?: string;
} | null {
  // "auto" 模式：自動選擇第一個有選項的引數
  // 顯式模式：依引數名稱選擇
  // 若已填入或無選項則回傳 null
}

// 命令偵測
export function isCommandMessage(text?: string): boolean {
  // 檢查文字是否以 '/' 開頭（正規化後）
}

export function isNativeCommandSurface(surface?: string): boolean {
  // 檢查介面是否在原生命令介面集合中
  // 依外掛註冊表版本快取
}

export function shouldHandleTextCommands(params): boolean {
  // 根據命令來源（native vs text）和設定決定是否處理
  // 非原生介面或設定未停用文字命令時回傳 true
}
```

#### 命令引數解析的精細設計

引數解析系統值得特別關注。`parsePositionalArgs()`（第 127-160 行）將原始文字依空白分割為位置引數，但尊重 `captureRemaining` 旗標——當某個引數定義設定了 `captureRemaining: true` 時，該引數會捕獲剩餘的所有文字（包含空白）。這對於需要自由格式引數的命令非常重要，例如 `/set systemPrompt You are a helpful assistant` 中的 `systemPrompt` 後面的所有內容。

`resolveCommandArgMenu()`（第 269-305 行）是為支援互動式 UI 的通道設計的——在某些平台上，使用者可以從選單中選擇引數值，而不是手動輸入。這個函式決定哪個引數需要顯示選單，以及選單的內容。

#### 原生命令與文字命令的雙軌機制

OpenClaw 對命令有「原生」（native）和「文字」（text）兩種處理路徑：

- **原生命令**：透過平台的原生命令介面（如 Discord 的 Slash Commands、Telegram 的 BotCommands）註冊和調用。這些命令由平台負責解析和驗證。
- **文字命令**：使用者在普通文字訊息中輸入 `/command` 格式，由 OpenClaw 自行偵測和解析。

`CommandScope` 型別（`"text" | "native" | "both"`）控制每個命令在哪種路徑上可用。`isNativeCommandSurface()` 判斷當前介面是否支援原生命令，`shouldHandleTextCommands()` 則根據設定和介面決定是否啟用文字命令解析。

#### 命令共享邏輯（commands-registry.shared.ts）

```typescript
// source-repo/src/auto-reply/commands-registry.shared.ts (45 行)
export function isNativeCommandSurface(surface?: string): boolean {
  // 查詢活躍的外掛註冊表
  // 依註冊表版本快取結果以避免重複計算
}

export function shouldHandleTextCommands(params: ShouldHandleTextCommandsParams): boolean {
  // 若非原生介面或設定未停用文字命令 → true
  // 若命令來源為 native → false（已由平台處理）
  // 回退到文字命令處理
}
```

### 4.6 命令路由與幫助訊息

```typescript
// source-repo/src/auto-reply/commands-text-routing.ts (241 行)
export function buildHelpMessage(cfg?: OpenClawConfig): string {
  // 建構多行幫助文字，含 session、options、status、skills 區段
}

export function buildCommandsMessage(cfg?, skillCommands?, options?): string {
  // 依分類群組命令，使用預定義標籤和順序
  // 格式化命令條目，含別名和範圍標籤
}

export function buildCommandsMessagePaginated(cfg?, skillCommands?, options?): CommandsMessageResult {
  // 支援分頁輸出（每頁 8 個命令）
  // 用於有自訂 UI 的通道
}
```

### 4.7 引數格式化器

```typescript
// source-repo/src/auto-reply/commands-args.ts (163 行)
export const COMMAND_ARG_FORMATTERS: Record<string, CommandArgsFormatter> = {
  config: formatActionArgs,   // show/get, set/unset 操作
  mcp: formatActionArgs,
  plugins: formatActionArgs,
  debug: formatActionArgs,
  queue: formatQueueArgs,     // mode, debounce, cap, drop 參數
  exec: formatExecArgs,       // host, security, ask, node 參數
};
```

---

## 5. 派發流程

派發流程是自動回覆系統的入口點，定義在 `dispatch.ts`（82 行）。它由三個逐層封裝的函式組成：

### 5.1 dispatchInboundMessage() — 核心派發

```typescript
// source-repo/src/auto-reply/dispatch.ts:20-39
export async function dispatchInboundMessage(
  ctx: MsgContext,
  cfg: OpenClawConfig,
  dispatcher: ReplyDispatcher,
  replyOptions?: Partial<GetReplyOptions>,
  replyResolver?: unknown
): Promise<DispatchInboundResult> {
  // 1. 定案上下文：finalizeInboundContext()
  //    → CommandAuthorized: undefined → false (default-deny)
  const finalized = finalizeInboundContext(ctx);

  // 2. 生命週期包裝：withReplyDispatcher()
  return withReplyDispatcher(dispatcher, async () => {
    // 3. 根據設定派發回覆
    return dispatchReplyFromConfig(finalized, cfg, dispatcher, replyOptions, replyResolver);
  });
}
```

### 5.2 dispatchInboundMessageWithBufferedDispatcher() — 帶緩衝的派發

```typescript
// source-repo/src/auto-reply/dispatch.ts:41-65
export async function dispatchInboundMessageWithBufferedDispatcher(
  ctx: MsgContext,
  cfg: OpenClawConfig,
  replyOptions?: Partial<GetReplyOptions>,
  replyResolver?: unknown
): Promise<DispatchInboundResult> {
  // 1. 建立帶輸入指示器的回覆派發器
  const dispatcher = createReplyDispatcherWithTyping(/* ... */);

  try {
    // 2. 合併使用者提供的 replyOptions 與派發器生成的選項
    const result = await dispatchInboundMessage(ctx, cfg, dispatcher, mergedOptions, replyResolver);
    return result;
  } finally {
    // 3. 確保清理：標記執行完成、等待派發器閒置
    dispatcher.markRunComplete();
    await dispatcher.markDispatchIdle();
  }
}
```

這個變體會建立一個帶有「正在輸入」（typing indicator）功能的派發器，讓使用者在等待回覆時能看到 Agent 正在處理。`finally` 區塊確保即使發生錯誤也會正確清理資源。

### 5.3 dispatchInboundMessageWithDispatcher() — 簡單派發

```typescript
// source-repo/src/auto-reply/dispatch.ts:67-82
export async function dispatchInboundMessageWithDispatcher(
  ctx: MsgContext,
  cfg: OpenClawConfig,
  replyOptions?: Partial<GetReplyOptions>
): Promise<DispatchInboundResult> {
  // 簡化版本：建立基本回覆派發器（無緩衝/輸入指示器）
  const dispatcher = createReplyDispatcher(/* ... */);
  return dispatchInboundMessage(ctx, cfg, dispatcher, replyOptions);
}
```

### 5.4 withReplyDispatcher() — 生命週期管理

```typescript
// source-repo/src/auto-reply/dispatch-dispatcher.ts:3-19
export async function withReplyDispatcher<T>(
  dispatcher: ReplyDispatcher,
  run: () => Promise<T>,
  onSettled?: () => Promise<void>
): Promise<T> {
  try {
    return await run();
  } finally {
    // 1. 標記完成：通知派發器不會再有新回覆
    dispatcher.markComplete();
    // 2. 等待閒置：等待所有保留的回覆都已發送
    await dispatcher.waitForIdle();
    // 3. 執行結算回呼（可選）
    await onSettled?.();
  }
}
```

這個包裝器確保了派發器的完整生命週期管理：無論 `run()` 成功或失敗，都會呼叫 `markComplete()` 通知派發器「不會再有新回覆了」，然後等待所有已排隊的回覆都完成發送，最後執行可選的結算回呼。

### 5.5 完整流程圖

```
Inbound Message (MsgContext)
       │
       ▼
dispatchInboundMessageWithBufferedDispatcher()
  ├── createReplyDispatcherWithTyping()  ← 建立帶 typing 的派發器
  │
  ▼
dispatchInboundMessage()
  ├── finalizeInboundContext()           ← default-deny: CommandAuthorized → false
  │
  ▼
withReplyDispatcher()
  ├── dispatchReplyFromConfig()          ← 根據設定派發回覆
  │     ├── resolveSendableOutboundReplyParts()
  │     ├── resolveConversationBindingRecord()
  │     ├── Plugin hooks (message received, claims)
  │     ├── resolveSendPolicy()
  │     └── getReplyFromConfig()         ← 進入 reply/ 子目錄
  │
  └── finally:
        ├── dispatcher.markComplete()    ← 標記完成
        ├── dispatcher.waitForIdle()     ← 等待所有回覆發送完畢
        └── onSettled?.()                ← 結算回呼
```

---

## 6. 回覆子目錄（reply/ Subdirectory）

`reply/` 子目錄是整個自動回覆系統最龐大的部分，包含 339 個檔案（211 個非測試檔案）和 6 個子目錄。它涵蓋了從 Agent 執行引擎到命令處理的所有核心邏輯。

### 6.1 reply.ts — 桶檔案

```typescript
// source-repo/src/auto-reply/reply.ts (12 行)
export { extractElevatedDirective } from "./reply/directives.js";
export { extractReasoningDirective } from "./reply/directives.js";
export { extractTraceDirective } from "./reply/directives.js";
export { extractThinkDirective } from "./reply/directives.js";
export { extractVerboseDirective } from "./reply/directives.js";
export { getReplyFromConfig } from "./reply/get-reply.js";
export { extractExecDirective } from "./reply/exec.js";
export { extractQueueDirective } from "./reply/queue.js";
export { extractReplyToTag } from "./reply/reply-tags.js";
export type { GetReplyOptions } from "./types.js";
export type { ReplyPayload } from "./reply-payload.js";
```

這個 12 行的桶檔案（barrel file）是 `reply/` 子目錄的公開 API。它精心挑選了最重要的匯出：指令提取函式、核心回覆函式、和型別定義。

### 6.2 Agent 執行引擎（agent-runner*.ts）

Agent 執行引擎是回覆生成的核心，負責協調 AI 模型呼叫、記憶管理、會話狀態和 Payload 建構。

```typescript
// source-repo/src/auto-reply/reply/agent-runner.ts
export async function runReplyAgent(params) {
  // 1. 從持久化存儲刷新會話條目
  // 2. 建構行內外掛狀態 Payload
  // 3. 解析上下文 Token、模型授權、使用追蹤
  // 4. 管理記憶體刷新和預飛壓縮
  // 5. 處理後續執行（followup runs）和佇列設定
  // 6. 呼叫 runAgentTurnWithFallback() 執行 Agent 回合
  // 7. 建構回覆 Payload（buildReplyPayloads）
}
```

相關檔案：
- `agent-runner-execution.ts`：Agent 回合執行
- `agent-runner-memory.ts`：記憶體管理（去重）
- `agent-runner-payloads.ts`：Payload 建構
- `agent-runner-helpers.ts`：輔助工具
- `agent-runner-session-reset.ts`：會話重置
- `agent-runner-auth-profile.ts`：認證設定檔
- `agent-runner-usage-line.ts`：用量行格式化
- `agent-runner-reminder-guard.ts`：提醒防護

### 6.3 回覆解析管線（get-reply*.ts）

```typescript
// source-repo/src/auto-reply/reply/get-reply.ts
export async function getReplyFromConfig(
  ctx: FinalizedMsgContext,
  opts?: Partial<GetReplyOptions>,
  configOverride?: OpenClawConfig
): Promise<ReplyPayload | ReplyPayload[] | undefined> {
  // 1. 快速路徑檢查
  //    - shouldHandleFastReplyTextCommands()
  //    - shouldUseReplyFastDirectiveExecution()
  // 2. 技能篩選器合併（mergeSkillFilters）
  // 3. 媒體與連結理解
  //    - hasInboundMedia() → applyMediaUnderstandingIfNeeded()
  //    - hasLinkCandidate() → applyLinkUnderstandingIfNeeded()
  // 4. 會話狀態初始化（initSessionState）
  // 5. 行內動作處理（handleInlineActions）
  // 6. 指令解析（resolveReplyDirectives）
  // 7. 預備回覆執行（runPreparedReply）
}
```

輔助函式：
- `mergeSkillFilters()`：合併通道和 Agent 的技能篩選器
- `hasInboundMedia()`：檢查 MsgContext 中是否有媒體
- `hasLinkCandidate()`：偵測 HTTP/HTTPS 連結
- `createFastTestModelSelectionState()`：快速模型選擇狀態

延遲載入的模組（Lazy-loaded runtimes）：
- `session-reset-model.runtime.js`
- `stage-sandbox-media.runtime.js`
- `hook-runner-global.js`
- `origin-routing.js`

### 6.4 設定驅動的派發（dispatch-from-config.ts）

```typescript
// source-repo/src/auto-reply/reply/dispatch-from-config.ts
export async function dispatchFromConfig(params: DispatchFromConfigParams): Promise<DispatchFromConfigResult> {
  // 1. 解析可發送的出站回覆部分
  // 2. 會話綁定記錄管理
  // 3. 外掛鉤子觸發（訊息接收、聲明、會話綁定）
  // 4. 發送策略解析
  // 5. TTS 處理（maybeApplyTtsToReplyPayload）
  // 6. 入站音頻偵測（isInboundAudioContext）
}
```

入站音頻偵測支援兩種模式：
- MIME 類型偵測（audio/* 媒體類型）
- 佔位符偵測（`<media:audio>` 語法），透過 `AUDIO_PLACEHOLDER_RE` 和 `AUDIO_HEADER_RE` 正則表達式

### 6.5 會話管理（session*.ts）

```typescript
// source-repo/src/auto-reply/reply/session.ts
export type SessionInitResult = {
  sessionCtx: TemplateContext;      // 含會話資訊的模板上下文
  sessionEntry: SessionEntry;       // 當前會話條目
  previousSessionEntry?: SessionEntry; // 前一個會話條目（重置時）
  sessionStore: Record<string, SessionEntry>;
  sessionKey: string;
  sessionId: string;
  isNewSession: boolean;
  resetTriggered: boolean;
  systemSent: boolean;
  abortedLastRun: boolean;
  storePath: string;
  sessionScope: SessionScope;
  groupResolution?: GroupKeyResolution;
  isGroup: boolean;
  bodyStripped?: string;
  triggerBodyNormalized: string;
};
```

會話管理功能涵蓋：
- **會話金鑰解析**：群組會話、討論串資訊解析
- **新鮮度偵測**：`evaluateSessionFreshness()` 判斷會話是否過期
- **重置策略**：`resolveSessionResetPolicy()` 支援閒置過期（idle）和每日過期（daily）
- **會話分叉**：`forkSessionFromParent()` 從父會話建立子會話
- **生命週期鉤子**：`buildSessionStartHookPayload()` / `buildSessionEndHookPayload()`
- **帳號預設值**：`resolveSessionDefaultAccountId()` 從設定或持久化狀態解析

### 6.6 命令處理（commands*.ts）

`reply/` 子目錄中有 96 個 `commands-*.ts` 檔案，涵蓋了所有的斜線命令實作。命令入口點是 `commands.ts`：

```typescript
// source-repo/src/auto-reply/reply/commands.ts
export { buildCommandContext } from "./commands-context.js";
export { handleCommands } from "./commands-core.js";
export { buildStatusReply } from "./commands-status.js";
export type { CommandContext, CommandHandlerResult, HandleCommandsParams } from "./commands-types.js";
```

主要命令分類：

| 分類 | 檔案 | 命令範例 |
|------|------|----------|
| 核心 | `commands-core.ts` | 命令路由與執行 |
| 會話 | `commands-session.ts`、`commands-reset.ts` | `/new`、`/reset` |
| 模型 | `commands-models.ts`、`commands-compact.ts` | `/model`、`/compact` |
| 狀態 | `commands-status.ts`、`commands-info.ts` | `/status`、`/info` |
| 外掛 | `commands-plugins.ts`、`commands-mcp.ts` | `/plugins`、`/mcp` |
| 工具 | `commands-bash.ts`、`commands-tasks.ts` | `/bash`、`/tasks` |
| ACP | `commands-acp.ts`（+ `commands-acp/` 子目錄） | ACP 相關命令 |
| 子代理 | `commands-subagents.ts`（+ `commands-subagents/` 子目錄） | 子代理管理 |
| 設定 | `commands-config.ts`、`commands-setunset.ts` | `/config`、`/set`、`/unset` |
| 媒體 | `commands-tts.ts`、`commands-export-session.ts` | `/tts`、`/export` |
| 其他 | `commands-whoami.ts`、`commands-btw.ts` | `/whoami`、`/btw` |

### 6.7 指令處理（directive-handling*.ts）

指令（Directives）是使用者嵌入在訊息中的行內控制語法，使用 `[[...]]` 格式：

```typescript
// source-repo/src/auto-reply/reply/directive-handling.ts
export { applyInlineDirectivesFastLane } from "./directive-handling.fast-lane.js";
export * from "./directive-handling.impl.js";
export type { InlineDirectives } from "./directive-handling.parse.js";
export { isDirectiveOnly } from "./directive-handling.directive-only.js";
export { parseInlineDirectives } from "./directive-handling.parse.js";
export { persistInlineDirectives } from "./directive-handling.persist.js";
export { resolveDefaultModel } from "./directive-handling.defaults.js";
export { formatDirectiveAck } from "./directive-handling.shared.js";
```

關鍵模組：
- **解析**：`parseInlineDirectives()` 解析 `[[...]]` 語法
- **快速通道**：`applyInlineDirectivesFastLane()` 跳過完整管線的快速執行
- **持久化**：`persistInlineDirectives()` 保存指令狀態
- **預設模型**：`resolveDefaultModel()` 解析預設模型
- **確認格式化**：`formatDirectiveAck()` 格式化確認訊息
- **純指令偵測**：`isDirectiveOnly()` 檢查訊息是否只包含指令

頂層匯出的指令提取函式：
- `extractElevatedDirective`：提升權限指令（@elevated）
- `extractReasoningDirective`：推理指令（@reasoning）
- `extractTraceDirective`：追蹤指令（@trace）
- `extractThinkDirective`：思考指令（@think）
- `extractVerboseDirective`：詳細輸出指令（@verbose）
- `extractExecDirective`：執行指令
- `extractQueueDirective`：佇列指令

### 6.8 群組訊息處理（groups.ts）

```typescript
// source-repo/src/auto-reply/reply/groups.ts
export function resolveGroupRequireMention(): boolean { /* ... */ }
export function defaultGroupActivation(): "always" | "mention" { /* ... */ }
export function buildGroupChatContext(params): string { /* ... */ }
export function buildGroupIntro(params): string { /* ... */ }
```

群組上下文建構包含：
- **通道偵測**：辨識 iMessage、WhatsApp、WebChat 等
- **參與者資訊**：成員清單、群組主題
- **啟動模式**：「永遠啟動」（always-on）vs「需要提及」（mention-only）
- **靜默 Token 指引**：指導 Agent 如何使用 `SILENT_REPLY_TOKEN`

行為準則（嵌入群組上下文中）：
- **潛伏優先**（lurk-first）哲學
- **選擇性參與**
- **人類化書寫風格**（禁止 Markdown 表格）
- **最少空白和換行**

### 6.9 回覆交付（reply-delivery.ts）

```typescript
// source-repo/src/auto-reply/reply/reply-delivery.ts
export type ReplyDirectiveParseMode = "always" | "auto" | "never";

export function normalizeReplyPayloadDirectives(params): {
  payload: ReplyPayload;
  isSilent: boolean;
} { /* ... */ }

export function createBlockReplyDeliveryHandler(params): (
  payload: ReplyPayload
) => Promise<void> { /* ... */ }
```

回覆交付管線：
- **指令解析**：處理 `[[...]]` 和 `MEDIA:` 指令
- **Payload 正規化**：修剪空白、套用回覆標籤
- **區塊串流**：與 `block-reply-pipeline.ts` 整合
- **輸入指示器**：為串流內容發送「正在輸入」狀態
- **媒體處理**：正規化媒體路徑、將音頻轉為語音
- **去重**：追蹤已發送的鍵以避免重複

### 6.10 訊息佇列（queue*.ts）

```typescript
// source-repo/src/auto-reply/reply/queue.ts
export { extractQueueDirective } from "./queue/directive.js";
export { clearSessionQueues } from "./queue/cleanup.js";
export { scheduleFollowupDrain } from "./queue/drain.js";
export {
  enqueueFollowupRun,
  getFollowupQueueDepth,
  resetRecentQueuedMessageIdDedupe,
} from "./queue/enqueue.js";
export { resolveQueueSettings } from "./queue/settings-runtime.js";
export { clearFollowupQueue, refreshQueuedFollowupSession } from "./queue/state.js";

export type {
  FollowupRun,
  QueueDedupeMode,
  QueueDropPolicy,
  QueueMode,
  QueueSettings,
} from "./queue/types.js";
```

佇列子系統分為 6 個子模組：
- `directive.js`：從訊息中提取佇列指令
- `cleanup.js`：清理會話佇列
- `drain.js`：排程佇列排出操作
- `enqueue.js`：後續執行（followup run）入隊，含去重
- `settings-runtime.js`：執行時期佇列設定解析
- `state.js`：佇列狀態管理

### 6.11 ReplyPayload 結構

```typescript
// source-repo/src/auto-reply/reply-payload.ts (47 行)
export type ReplyPayload = {
  text?: string;                   // 文字內容
  media?: string[];                // 媒體 URL 陣列
  interactive?: unknown;           // 互動式元素
  replyToId?: string;              // 回覆目標訊息 ID
  replyToCurrent?: boolean;        // 回覆當前訊息
  replyToTag?: string;             // 回覆標籤
  isReasoningBlock?: boolean;      // 是否為推理區塊
  isCompactionNotice?: boolean;    // 是否為壓縮通知
};

export type ReplyPayloadMetadata = {
  // 後設資料結構（透過 WeakMap 關聯，避免直接修改 Payload）
};

// 使用 WeakMap 進行非侵入式後設資料關聯
export function setReplyPayloadMetadata<T>(payload: T, metadata: ReplyPayloadMetadata): T;
export function getReplyPayloadMetadata(payload: object): ReplyPayloadMetadata | undefined;
```

值得注意的是，後設資料使用 `WeakMap` 進行關聯——這是一個優雅的設計選擇，允許在不修改原始 Payload 物件的情況下附加額外資訊，同時讓垃圾回收器能夠自動清理。`WeakMap` 的鍵是物件參考，當 Payload 物件不再被引用時，對應的後設資料也會自動被回收，避免記憶體洩漏。

### 6.12 GetReplyOptions 結構

```typescript
// source-repo/src/auto-reply/get-reply-options.types.ts (150 行)
export type GetReplyOptions = {
  runId?: string;
  abortSignal?: AbortSignal;
  images?: unknown;

  // 生命週期回呼
  onAgentRunStart?: () => void;
  onReplyStart?: () => void;

  // 輸入指示器
  typing?: TypingPolicy;

  // 串流輸出
  onPartialReply?: (text: string) => void;
  onBlockReply?: (payload: ReplyPayload) => void;
  onToolResult?: (result: unknown) => void;

  // 詳細事件追蹤
  onItemEvent?: (event: unknown) => void;
  onPlanUpdate?: (plan: unknown) => void;
  onApprovalEvent?: (event: unknown) => void;

  // 推理區塊回呼
  // 壓縮鉤子
  // 模型選擇
  // Bootstrap 上下文模式
  // 技能篩選器
  // 逾時覆寫
};
```

`GetReplyOptions` 是回覆系統的「控制面板」——它透過一系列回呼函式（callbacks）提供了對回覆生成過程的細粒度觀察和控制能力。幾個值得注意的設計點：

- **`abortSignal`**：使用標準的 `AbortSignal` API，允許外部取消正在進行的回覆生成。這對於使用者按下「停止」或發送新訊息時取消舊回覆非常重要。
- **`onPartialReply`** / **`onBlockReply`**：支援兩種串流粒度——文字級（每個 Token）和區塊級（每個完整段落）。
- **`onToolResult`**：當 Agent 使用工具（如執行 bash 命令、讀取檔案等）時觸發，讓 UI 能即時顯示工具使用結果。
- **`onApprovalEvent`**：配合人機協作（human-in-the-loop）流程，當 Agent 需要人工審批時觸發。

### 6.13 架構模式總結

綜觀 `reply/` 子目錄的架構，可以歸納出幾個核心設計模式：

1. **延遲載入（Lazy Loading）**：大量使用 `.runtime.ts` 檔案和動態 `import()` 來延遲載入重型模組。這確保了啟動時只載入必要的程式碼，減少記憶體佔用和啟動時間。

2. **桶檔案聚合（Barrel Exports）**：核心入口點（如 `reply.ts`、`commands.ts`、`queue.ts`）都是精心策劃的桶檔案，只暴露穩定的公開 API，隱藏內部實作細節。

3. **回呼驅動的事件系統**：`GetReplyOptions` 中的回呼函式組成了一個輕量級的事件系統，讓消費者能觀察回覆生成的每個階段，而不需要引入完整的事件發射器（EventEmitter）基礎設施。

4. **設定驅動的行為**：幾乎所有行為都可以透過 `OpenClawConfig` 進行設定，從模型選擇到發送策略，從佇列參數到思考等級。

5. **外掛擴展性**：通道外掛（channel plugins）深度整合到每個處理階段，從命令名稱解析到 allowFrom 驗證，確保了對新通道的可擴展性。

---

## 7. 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/auto-reply/dispatch.ts:1-82` | 派發入口，三個主要派發函式 |
| `source-repo/src/auto-reply/dispatch-dispatcher.ts:1-19` | `withReplyDispatcher()` 生命週期管理 |
| `source-repo/src/auto-reply/reply.ts:1-12` | 桶檔案，重新匯出 reply/ 子目錄 |
| `source-repo/src/auto-reply/templating.ts:1-258` | MsgContext（~70+ 欄位）、TemplateContext、applyTemplate() |
| `source-repo/src/auto-reply/envelope.ts:1-267` | 信封格式化（formatAgentEnvelope、formatInboundEnvelope 等） |
| `source-repo/src/auto-reply/commands-registry.ts:1-327` | 命令註冊表主檔案 |
| `source-repo/src/auto-reply/commands-registry-list.ts:1-62` | 命令清單與啟用檢查 |
| `source-repo/src/auto-reply/commands-registry-normalize.ts:1-183` | 命令正規化與別名解析 |
| `source-repo/src/auto-reply/commands-registry.data.ts:1-68` | 內建命令 + 動態 Dock 命令 |
| `source-repo/src/auto-reply/commands-registry.types.ts:1-83` | 命令型別定義 |
| `source-repo/src/auto-reply/commands-registry.shared.ts:1-45` | 命令共享邏輯 |
| `source-repo/src/auto-reply/commands-registry.runtime.ts:1-2` | Runtime 匯出聚合 |
| `source-repo/src/auto-reply/commands-registry.runtime-types.ts:1-4` | Runtime 型別 |
| `source-repo/src/auto-reply/commands-text-routing.ts:1-241` | 幫助訊息建構（含分頁） |
| `source-repo/src/auto-reply/commands-args.ts:1-163` | 引數格式化器 |
| `source-repo/src/auto-reply/commands-args.types.ts:1-8` | 引數值型別定義 |
| `source-repo/src/auto-reply/reply-payload.ts:1-47` | ReplyPayload 結構與 WeakMap 後設資料 |
| `source-repo/src/auto-reply/get-reply-options.types.ts:1-150` | GetReplyOptions 完整選項型別 |
| `source-repo/src/auto-reply/reply/agent-runner.ts` | Agent 執行引擎入口 |
| `source-repo/src/auto-reply/reply/get-reply.ts` | 回覆解析管線入口 |
| `source-repo/src/auto-reply/reply/dispatch-from-config.ts` | 設定驅動的派發 |
| `source-repo/src/auto-reply/reply/session.ts` | 會話管理（初始化、過期、分叉） |
| `source-repo/src/auto-reply/reply/commands.ts` | 命令處理入口 |
| `source-repo/src/auto-reply/reply/directive-handling.ts` | 指令處理入口 |
| `source-repo/src/auto-reply/reply/groups.ts` | 群組訊息處理 |
| `source-repo/src/auto-reply/reply/reply-delivery.ts` | 回覆交付管線 |
| `source-repo/src/auto-reply/reply/queue.ts` | 訊息佇列入口 |
| `source-repo/src/auto-reply/reply/block-reply-pipeline.ts` | 區塊串流管線 |
