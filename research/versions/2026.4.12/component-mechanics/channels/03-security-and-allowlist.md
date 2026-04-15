# Channels 子章節 03：安全與允許清單

> **引用範圍**：`src/channels/allowlist-match.ts`, `allow-from.ts`, `plugins/types.core.ts`（安全相關）

## 1. 概觀

Channel 安全系統解決兩個核心問題：

1. **誰可以跟 Bot 對話？** — 允許清單（Allowlist）
2. **Bot 在群聊中應該回應什麼？** — Mention 門控（見 02-core-subsystems.md）

允許清單是 OpenClaw 的**第一道防線**，在訊息到達 Agent 之前就進行身份過濾。

---

## 2. 允許清單匹配引擎

### 2.1 匹配來源

```typescript
// source-repo/src/channels/allowlist-match.ts:6-16
export type AllowlistMatchSource =
  | "wildcard"       // "*" — 允許所有人
  | "id"             // 平台使用者 ID
  | "name"           // 顯示名稱
  | "tag"            // 使用者標籤（如 user#1234）
  | "username"       // 使用者名稱
  | "prefixed-id"    // 帶前綴的 ID（如 "id:12345"）
  | "prefixed-user"  // 帶前綴的使用者名稱
  | "prefixed-name"  // 帶前綴的名稱
  | "slug"           // Slug 格式
  | "localpart";     // Matrix @user:server 的 user 部分
```

10 種匹配來源覆蓋了各平台不同的身份識別方式。

> **引用來源**：`source-repo/src/channels/allowlist-match.ts:6-16`

### 2.2 匹配結果

```typescript
// source-repo/src/channels/allowlist-match.ts:18-22
export type AllowlistMatch<TSource extends string = AllowlistMatchSource> = {
  allowed: boolean;       // 是否允許
  matchKey?: string;      // 匹配到的鍵
  matchSource?: TSource;  // 匹配來源類型
};
```

匹配結果不僅返回是/否，還返回**具體匹配方式**，方便除錯和日誌記錄。

### 2.3 編譯允許清單

```typescript
// source-repo/src/channels/allowlist-match.ts:24-27
export type CompiledAllowlist = {
  set: ReadonlySet<string>;   // 已正規化的允許清單集合
  wildcard: boolean;           // 是否包含 "*"
};

// source-repo/src/channels/allowlist-match.ts:35-41
export function compileAllowlist(entries: ReadonlyArray<string>): CompiledAllowlist {
  const set = new Set(entries.filter(Boolean));
  return {
    set,
    wildcard: set.has("*"),
  };
}
```

允許清單在載入時「編譯」為 `Set`，查詢時間複雜度為 O(1)。`wildcard` 旗標避免每次都檢查 `"*"`。

> **引用來源**：`source-repo/src/channels/allowlist-match.ts:24-41`

### 2.4 候選匹配流程

```typescript
// source-repo/src/channels/allowlist-match.ts:51-60
export function resolveAllowlistCandidates<TSource extends string>(params: {
  compiledAllowlist: CompiledAllowlist;
  candidates: Array<{ value?: string; source: TSource }>;
}): AllowlistMatch<TSource>
```

匹配流程：

```
1. 檢查 wildcard → 如果 "*" 在清單中，立即允許
2. 遍歷 candidates 陣列
   ├── candidate.value 存在？
   │   ├── 在 set 中？→ 回傳 { allowed: true, matchKey, matchSource }
   │   └── 不在？→ 繼續下一個 candidate
   └── 不存在？→ 跳過
3. 全部不匹配 → 回傳 { allowed: false }
```

`candidates` 是**有序**的，第一個匹配就返回。通道 Plugin 決定候選的優先順序，例如：

```typescript
// 典型 candidates 順序
[
  { value: senderId, source: "id" },           // 平台 ID（最精確）
  { value: username, source: "username" },      // 使用者名稱
  { value: displayName, source: "name" },       // 顯示名稱（最模糊）
]
```

> **引用來源**：`source-repo/src/channels/allowlist-match.ts:51-60`

### 2.5 格式化工具

```typescript
// source-repo/src/channels/allowlist-match.ts:29-33
export function formatAllowlistMatchMeta(
  match?: { matchKey?: string; matchSource?: string } | null,
): string {
  return `matchKey=${match?.matchKey ?? "none"} matchSource=${match?.matchSource ?? "none"}`;
}
```

用於日誌記錄，格式化匹配元資料。

> **引用來源**：`source-repo/src/channels/allowlist-match.ts:29-33`

---

## 3. DM 安全策略

### 3.1 ChannelSecurityDmPolicy

```typescript
// source-repo/src/channels/plugins/types.core.ts:268-275
export type ChannelSecurityDmPolicy = {
  policy: string;                              // 策略名稱
  allowFrom?: Array<string | number> | null;   // 允許清單
  policyPath?: string;                         // 策略配置路徑
  allowFromPath: string;                       // 允許清單配置路徑
  approveHint: string;                         // 審批提示
  normalizeEntry?: (raw: string) => string;    // 項目正規化函式
};
```

每個通道 Plugin 透過 `ChannelSecurityAdapter` 回傳其 DM 策略。策略通常基於配置檔中的 `allowFrom` 欄位。

> **引用來源**：`source-repo/src/channels/plugins/types.core.ts:268-275`

### 3.2 安全上下文

```typescript
// source-repo/src/channels/plugins/types.core.ts:277-281
export type ChannelSecurityContext<ResolvedAccount = unknown> = {
  cfg: OpenClawConfig;
  accountId?: string | null;
  account: ResolvedAccount;
};
```

安全決策需要完整的配置和帳號上下文。

---

## 4. 帳號狀態

### 4.1 ChannelAccountState

```typescript
// source-repo/src/channels/plugins/types.core.ts:121-127
export type ChannelAccountState =
  | "linked"
  | "not linked"
  | "configured"
  | "not configured"
  | "enabled"
  | "disabled";
```

帳號有 6 種狀態，形成兩個維度：
- **配置維度**：not configured → configured
- **連結維度**：not linked → linked
- **啟用維度**：disabled → enabled

> **引用來源**：`source-repo/src/channels/plugins/types.core.ts:121-127`

---

## 5. 狀態問題報告

```typescript
// source-repo/src/channels/plugins/types.core.ts:113-119
export type ChannelStatusIssue = {
  channel: ChannelId;
  accountId: string;
  kind: "intent" | "permissions" | "config" | "auth" | "runtime";
  message: string;
  fix?: string;    // 修復建議
};
```

5 種問題類型：

| kind | 說明 |
|------|------|
| `intent` | 意圖問題（帳號未啟用） |
| `permissions` | 權限不足 |
| `config` | 配置錯誤 |
| `auth` | 認證失敗 |
| `runtime` | 執行時期錯誤 |

`fix` 欄位提供人類可讀的修復建議，提升使用者體驗。

> **引用來源**：`source-repo/src/channels/plugins/types.core.ts:113-119`

---

## 6. 群組安全上下文

```typescript
// source-repo/src/channels/plugins/types.core.ts:239-250
export type ChannelGroupContext = {
  cfg: OpenClawConfig;
  groupId?: string | null;
  groupChannel?: string | null;    // 群組通道（如 #general）
  groupSpace?: string | null;
  accountId?: string | null;
  senderId?: string | null;
  senderName?: string | null;
  senderUsername?: string | null;
  senderE164?: string | null;      // E.164 電話號碼格式
};
```

群組上下文提供了傳送者的多種識別資訊，用於在群聊中識別不同使用者。

> **引用來源**：`source-repo/src/channels/plugins/types.core.ts:239-250`

---

## 7. 審批能力

```typescript
// source-repo/src/channels/plugins/types.plugin.ts:77
approvalCapability?: ChannelApprovalCapability;
```

審批能力允許通道 Plugin 處理工具執行審批。當 Agent 需要執行敏感操作（如檔案寫入、Shell 指令）時，審批請求會透過通道傳送給使用者。

審批種類（`ChannelApprovalKind`）定義在 `infra/approval-types.ts`，包括 Exec 審批和 Plugin 審批。

> **引用來源**：`source-repo/src/channels/plugins/types.adapters.ts:6-12`

---

## 8. 安全設計原則

綜觀 Channel 安全系統的設計，有幾個核心原則：

1. **白名單優先**：預設拒絕所有未知來源，必須明確加入允許清單
2. **多重識別**：支援 10 種匹配來源，適應不同平台的身份系統
3. **可審計**：匹配結果包含匹配方式和匹配鍵，便於追蹤
4. **漸進式開放**：從 `"*"`（開放）到精確 ID 匹配，靈活度高
5. **通道無關**：安全邏輯在核心層實作，各平台只需提供身份資訊

---

## 引用來源

| 事實 | 檔案 | 行號 |
|------|------|------|
| AllowlistMatchSource | `source-repo/src/channels/allowlist-match.ts` | 6-16 |
| AllowlistMatch | `source-repo/src/channels/allowlist-match.ts` | 18-22 |
| CompiledAllowlist | `source-repo/src/channels/allowlist-match.ts` | 24-27 |
| compileAllowlist() | `source-repo/src/channels/allowlist-match.ts` | 35-41 |
| resolveAllowlistCandidates() | `source-repo/src/channels/allowlist-match.ts` | 51-60 |
| formatAllowlistMatchMeta() | `source-repo/src/channels/allowlist-match.ts` | 29-33 |
| ChannelSecurityDmPolicy | `source-repo/src/channels/plugins/types.core.ts` | 268-275 |
| ChannelSecurityContext | `source-repo/src/channels/plugins/types.core.ts` | 277-281 |
| ChannelAccountState | `source-repo/src/channels/plugins/types.core.ts` | 121-127 |
| ChannelStatusIssue | `source-repo/src/channels/plugins/types.core.ts` | 113-119 |
| ChannelGroupContext | `source-repo/src/channels/plugins/types.core.ts` | 239-250 |
| approvalCapability | `source-repo/src/channels/plugins/types.plugin.ts` | 77 |
