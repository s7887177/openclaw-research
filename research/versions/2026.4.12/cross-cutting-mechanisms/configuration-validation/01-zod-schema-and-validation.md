# Zod Schema 架構、驗證執行與錯誤處理

## 本章摘要

OpenClaw 的設定驗證系統以 **Zod** 為核心，將整個設定結構定義為一棵型別安全的 schema 樹。主 schema `OpenClawSchema` 由十多個子 schema 組合而成，涵蓋 agents、providers、channels、secrets、sandbox 等所有面向。驗證不只是簡單的型別檢查——它包含跨欄位的語義驗證（如 broadcast 的 agent ID 必須存在於 agents list）、legacy 設定偵測、以及自訂的錯誤映射機制。

---

## 1. Zod Schema 架構

### 1.1 主 Schema 入口

整個設定的 Zod schema 定義在 `src/config/zod-schema.ts`：

```typescript
// source-repo/src/config/zod-schema.ts:233-999
// OpenClawSchema 是主要的 Zod schema
// 它組合了 11 個子 schema 檔案（zod-schema.*.ts）
```

匯入的子 schema（`source-repo/src/config/zod-schema.ts:8-26`）：

| 子 Schema 檔案 | 涵蓋範圍 |
|----------------|----------|
| `zod-schema.core.ts` | SecretRef、SecretInput、Models、Colors |
| `zod-schema.agents.ts` | Agents 列表、Bindings、Broadcast、Audio |
| `zod-schema.providers.ts` | Channels 設定、Provider 正規化 |
| `zod-schema.channels.ts` | 通道特定設定 |
| `zod-schema.hooks.ts` | Hooks 設定 |
| `zod-schema.installs.ts` | 安裝設定 |
| `zod-schema.approvals.ts` | 審批設定 |
| `zod-schema.allowdeny.ts` | 允許/拒絕清單 |
| `zod-schema.session.ts` | Session 設定 |
| `zod-schema.talk.ts` | Talk/語音設定 |
| `zod-schema.sensitive.ts` | 敏感欄位標記 |
| `zod-schema.secret-input-validation.ts` | 密鑰輸入驗證 |

所有子 schema 透過 Zod 的物件組合（object composition）合併，並使用 `.strict()` 拒絕未知的 key。

### 1.2 跨欄位驗證

Zod 的 `superRefine` 方法用於跨欄位的語義驗證：

```typescript
// source-repo/src/config/zod-schema.ts:969-999
// superRefine 驗證：
// broadcast channel 的 agent ID 必須匹配 agents.list 中的條目
```

這種驗證無法用簡單的欄位型別表達——它需要在整個設定物件的層級進行檢查。

### 1.3 核心子 Schema：SecretRef

SecretRef 是密鑰引用的核心型別，定義了三種來源（`source-repo/src/config/zod-schema.core.ts:29-82`）：

```typescript
// source-repo/src/config/zod-schema.core.ts:29-45
// EnvSecretRefSchema - 環境變數引用
{
  source: "env",
  id: z.string().regex(/^[A-Z][A-Z0-9_]{0,127}$/) // 大寫字母開頭
}

// source-repo/src/config/zod-schema.core.ts:47-63
// FileSecretRefSchema - 檔案引用（JSON pointer）
{
  source: "file",
  id: // JSON pointer 格式如 "/providers/openai/apiKey" 或 "value"
}

// source-repo/src/config/zod-schema.core.ts:65-76
// ExecSecretRefSchema - 可執行檔引用
{
  source: "exec",
  id: // 透過 isValidExecSecretRefId() 驗證
}
```

三者的聯合型別：

```typescript
// source-repo/src/config/zod-schema.core.ts:78-84
// SecretRefSchema = discriminated union (source: "env"|"file"|"exec")
// SecretInputSchema = union of string | SecretRefSchema
```

環境變數引用的 ID 必須符合 `^[A-Z][A-Z0-9_]{0,127}$`——大寫字母開頭、只含大寫字母數字底線、最長 128 字元。

### 1.4 Agents 子 Schema

```typescript
// source-repo/src/config/zod-schema.agents.ts:7-92
// AgentsSchema - 定義 agent 列表與預設值
// BindingsSchema - 路由綁定（route vs ACP）
// BroadcastSchema - 廣播規則與策略
// AudioSchema - 語音轉錄設定
```

### 1.5 Secrets 設定 Schema

```typescript
// source-repo/src/config/zod-schema.core.ts:151-182
// SecretsConfigSchema - 定義 secrets providers
// 支援 env / file / exec 三種 provider 型別
```

### 1.6 敏感欄位標記

```typescript
// source-repo/src/config/zod-schema.sensitive.ts:1-6
// sensitive: Zod registry for marking fields as sensitive
// 被標記的欄位在 Control UI 中會被遮蔽（redacted）
```

---

## 2. 驗證執行

### 2.1 主驗證函式

驗證的入口位於 `src/config/validation.ts`：

```typescript
// source-repo/src/config/validation.ts:572-626
export function validateConfigObjectRaw(raw: unknown):
  | { ok: true; config: OpenClawConfig }
  | { ok: false; issues: ConfigValidationIssue[] }
{
  // 1. 檢查不支援的可變 SecretRef 政策問題
  const policyIssues = collectUnsupportedSecretRefPolicyIssues(raw);

  // 2. 偵測 legacy 設定問題
  const legacyIssues = findLegacyConfigIssues(raw, raw,
    listPluginDoctorLegacyConfigRules({ pluginIds: collectRelevantDoctorPluginIds(raw) })
  );
  if (legacyIssues.length > 0) return { ok: false, issues: legacyIssues.map(...) };

  // 3. Zod schema 驗證
  const validated = OpenClawSchema.safeParse(raw);
  if (!validated.success) {
    const schemaIssues = validated.error.issues.map(mapZodIssueToConfigIssue);
    return { ok: false, issues: mergeUnsupportedMutableSecretRefIssues(policyIssues, schemaIssues) };
  }

  // 4. 重複 agent 目錄檢查
  const duplicates = findDuplicateAgentDirs(validatedConfig);
  if (duplicates.length > 0) return { ok: false, issues: [...] };

  // 5. Avatar 設定驗證
  const avatarIssues = validateIdentityAvatar(validatedConfig);
  if (avatarIssues.length > 0) return { ok: false, issues: avatarIssues };

  // 6. Gateway Tailscale 綁定驗證
  const gatewayTailscaleBindIssues = validateGatewayTailscaleBind(validatedConfig);
  if (gatewayTailscaleBindIssues.length > 0) return { ok: false, issues: ... };

  return { ok: true, config: validatedConfig };
}
```

驗證流程是**逐層、短路**的——第一層驗證失敗就立即回傳，不會進入後續驗證。

### 2.2 帶 Plugin 的驗證

```typescript
// source-repo/src/config/validation.ts:653-658
export function validateConfigObjectWithPlugins(
  raw: unknown,
  params?: { env?: NodeJS.ProcessEnv },
): ValidateConfigWithPluginsResult {
  return validateConfigObjectWithPluginsBase(raw, { applyDefaults: true, env: params?.env });
}
```

```typescript
// source-repo/src/config/validation.ts:660-665
export function validateConfigObjectRawWithPlugins(
  raw: unknown,
  params?: { env?: NodeJS.ProcessEnv },
): ValidateConfigWithPluginsResult {
  return validateConfigObjectWithPluginsBase(raw, { applyDefaults: false, env: params?.env });
}
```

兩者的差異在於是否套用預設值。帶 plugin 的驗證額外回傳 `warnings` 陣列。

### 2.3 物質化驗證

```typescript
// source-repo/src/config/validation.ts:628-639
export function validateConfigObject(raw: unknown):
  | { ok: true; config: OpenClawConfig }
  | { ok: false; issues: ConfigValidationIssue[] }
{
  const result = validateConfigObjectRaw(raw);
  if (!result.ok) return result;
  return {
    ok: true,
    config: materializeRuntimeConfig(result.config, "snapshot"),
  };
}
```

`validateConfigObject` 在驗證成功後還會呼叫 `materializeRuntimeConfig()` 來產出運行時設定快照。

---

## 3. 錯誤映射

### 3.1 Zod Issue → Config Issue

```typescript
// source-repo/src/config/validation.ts:447-480
// mapZodIssueToConfigIssue()
// 將 Zod 的驗證問題轉換為 ConfigValidationIssue
// ConfigValidationIssue = { path: string, message: string }
```

### 3.2 允許值提取

```typescript
// source-repo/src/config/validation.ts:173-186
// collectAllowedValuesFromCustomIssue()
// 從驗證錯誤中提取允許的值
// 用於向使用者顯示「有效值為 X, Y, Z」
```

### 3.3 Bindings 聯合型別錯誤

```typescript
// source-repo/src/config/validation.ts:281-357
// extractBindingsSpecificUnionIssue()
// 處理 route vs ACP binding 聯合型別的錯誤
// Zod 的聯合型別錯誤通常很冗長，這裡精簡它
```

### 3.4 SecretRef 政策問題

```typescript
// source-repo/src/config/validation.ts:375-396
// collectUnsupportedMutableSecretRefIssues()
// 驗證 SecretRef 的可變性政策
```

---

## 4. Schema 建構與合併

### 4.1 主 Schema 建構器

`src/config/schema.ts` 負責將核心 schema 與 plugin/channel schema 合併：

```typescript
// source-repo/src/config/schema.ts:484-528
// buildConfigSchema()
export function buildConfigSchema(params?: {
  plugins?: PluginUiMetadata[];
}) {
  // 1. 產出基礎 JSON Schema（從 Zod）
  // 2. 套用 plugin schema (applyPluginSchemas)
  // 3. 套用 channel schema (applyChannelSchemas)
  // 4. 快取合併結果
}
```

### 4.2 Schema 查找

```typescript
// source-repo/src/config/schema.ts:681
// lookupConfigSchema()
// 基於路徑的 schema 查找（如 "agents.defaults.sandbox.mode"）
```

### 4.3 JSON Schema 產出

```typescript
// source-repo/src/config/schema-base.ts:232-250
// computeBaseConfigSchemaStablePayload()
// 將 OpenClawSchema (Zod) 轉換為 JSON Schema (draft-07)
// 套用 FIELD_LABELS 和 FIELD_HELP 欄位文件
// 為 Control UI 移除 channel-specific schema
// 套用衍生標籤（derived tags）給 UI hints
```

### 4.4 Schema 工具函式

```typescript
// source-repo/src/config/schema.shared.ts:1-89
// cloneSchema() - 使用 structuredClone 或 JSON parse/stringify
// schemaHasChildren() - 檢查是否有 properties, items, 或組合分支
// findWildcardHintMatch() - UI hints 的萬用字元匹配
```

---

## 5. Plugin configSchema 合併

### 5.1 Plugin Manifest 中的 configSchema

插件可以在 manifest 中定義自己的設定 schema：

```typescript
// source-repo/src/plugins/manifest-registry.ts:67+
// PluginManifestRecord 型別
// Line 102: configSchema?: Record<string, unknown>
```

### 5.2 合併流程

```typescript
// source-repo/src/plugins/manifest-registry.ts:464-638
// loadPluginManifestRegistry()
// Line 556: 從 manifest 的 configSchema 屬性解析 schema
// Line 596-638: 映射 schema 到 registry entries
```

Plugin 的 schema 透過 `buildConfigSchema()` 中的 `applyPluginSchemas()` 合併到核心 schema：

```typescript
// source-repo/src/config/schema.ts:518
// applyPluginSchemas() 將 plugin schema 合併到基礎 JSON Schema
```

### 5.3 Plugin UI Metadata

```typescript
// source-repo/src/config/schema.ts:122-131
// PluginUiMetadata 型別：
// configSchema?: JsonSchemaNode      - 插件的 JSON Schema
// configUiHints?: Record<string, Pick<ConfigUiHint, ...>> - UI 提示
```

UI hints 包含表單元件型別、標籤、說明等，讓 Control UI 能動態呈現插件的設定表單。

---

## 6. 驗證的設計模式

### 6.1 Zod → JSON Schema 雙軌模式

OpenClaw 的設定驗證同時使用兩種 schema 表達：

1. **Zod Schema**（TypeScript 層）：用於**運行時驗證**，提供型別推斷
2. **JSON Schema**（序列化層）：用於 **Control UI** 的表單產生和文件產出

兩者透過 `computeBaseConfigSchemaStablePayload()` 橋接。

### 6.2 短路驗證模式

驗證採用逐層短路：legacy → Zod → 語義 → 跨欄位。這避免了在 legacy 設定上浪費 Zod 驗證的計算資源，也避免了在 schema 驗證失敗後進行無意義的語義檢查。

### 6.3 錯誤訊息的人性化

原始的 Zod 錯誤訊息（如聯合型別的多層巢狀錯誤）對使用者不友善。OpenClaw 投入大量邏輯來精簡和重組錯誤訊息，例如 `extractBindingsSpecificUnionIssue()` 將冗長的 Zod union 錯誤轉換為簡潔的 route/ACP binding 指引。

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/config/zod-schema.ts:8-26` | 子 schema 匯入 |
| `source-repo/src/config/zod-schema.ts:233-999` | OpenClawSchema 主 schema |
| `source-repo/src/config/zod-schema.ts:969-999` | superRefine 跨欄位驗證 |
| `source-repo/src/config/zod-schema.core.ts:29-82` | SecretRef schema 定義 |
| `source-repo/src/config/zod-schema.core.ts:84` | SecretInputSchema |
| `source-repo/src/config/zod-schema.core.ts:151-182` | SecretsConfigSchema |
| `source-repo/src/config/zod-schema.agents.ts:7-92` | Agents/Bindings/Broadcast schema |
| `source-repo/src/config/zod-schema.sensitive.ts:1-6` | 敏感欄位 registry |
| `source-repo/src/config/zod-schema.secret-input-validation.ts:1-103` | 密鑰輸入自訂驗證 |
| `source-repo/src/config/validation.ts:173-186` | 允許值提取 |
| `source-repo/src/config/validation.ts:281-357` | Bindings 聯合型別錯誤處理 |
| `source-repo/src/config/validation.ts:375-396` | SecretRef 政策問題 |
| `source-repo/src/config/validation.ts:447-480` | Zod → Config issue 映射 |
| `source-repo/src/config/validation.ts:572-626` | validateConfigObjectRaw() |
| `source-repo/src/config/validation.ts:628-665` | validateConfigObject/WithPlugins |
| `source-repo/src/config/schema.ts:122-131` | PluginUiMetadata |
| `source-repo/src/config/schema.ts:484-528` | buildConfigSchema() |
| `source-repo/src/config/schema.ts:681` | lookupConfigSchema() |
| `source-repo/src/config/schema-base.ts:232-250` | JSON Schema 產出 |
| `source-repo/src/config/schema.shared.ts:1-89` | Schema 工具函式 |
| `source-repo/src/plugins/manifest-registry.ts:102` | Plugin configSchema 欄位 |
| `source-repo/src/plugins/manifest-registry.ts:464-638` | Plugin manifest registry 載入 |
