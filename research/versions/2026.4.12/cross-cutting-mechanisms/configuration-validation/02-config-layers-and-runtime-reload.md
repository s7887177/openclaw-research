# 設定層次、運行時重載與 Secret Ref 驗證

## 本章摘要

上一章介紹了 Zod Schema 的架構與驗證執行。本章關注設定的**動態面向**：設定值如何從多個來源層疊合併、如何在運行時熱重載、以及 Secret Ref 如何被安全地解析和驗證。這些機制讓 OpenClaw 能在不重啟的情況下響應設定變更，同時確保密鑰永遠不會以明文形式暴露。

---

## 1. 設定的層次結構

OpenClaw 的設定遵循經典的層次覆蓋模式：

```
1. 程式碼預設值（Defaults）
   ↓ 被覆蓋
2. 設定檔（File: JSON5 格式）
   ↓ 被覆蓋
3. 環境變數替換（Env Substitution: ${VAR_NAME}）
   ↓ 被覆蓋
4. CLI 旗標（CLI Flags）
   ↓ 經過
5. 驗證（Validation: Zod + 語義檢查）
   ↓ 產出
6. 運行時設定（Runtime Config）
```

### 1.1 預設值（Defaults）

預設值定義在 `src/config/defaults.ts`：

```typescript
// source-repo/src/config/defaults.ts
// applyMessageDefaults() (Line 91-100) - 加入預設的 ack 反應範圍
// applyAgentDefaults() (Line 284-300) - 設定最大並行限制
// applyModelDefaultsForConfig() - 套用模型特定預設值（tokens, reasoning 等）
// resolveNormalizedProviderModelMaxTokens() (Line 68-84) - 鉗制 Mistral provider 的 tokens
```

預設值不是簡單的靜態物件——它們是根據其他設定值（如 model ID、provider 名稱）動態計算的。

### 1.2 設定檔 I/O

**位置**：`source-repo/src/config/io.ts`（超過 1800 行）

設定檔使用 **JSON5** 格式，支援註解和尾隨逗號：

```typescript
// source-repo/src/config/io.ts
// Line 8: JSON5.parse() 解析設定檔
// loadConfig() (Line 1731) - 同步載入（有快取）
// readConfigFileSnapshot() (Line 1747) - 非同步快照（含有效性檢查）
// readSourceConfigSnapshot() (Line 1751) - 原始來源（不物質化）
// writeConfigFile() (Line 1763+) - 寫入設定（含審計追蹤）
// parseConfigJson5() (Line 897-1000) - 解析 JSON5 設定檔
```

設定檔處理流程：
1. `JSON5.parse()` 解析設定檔
2. `resolveConfigEnvVars()` 替換 `${VAR}` 參照
3. `applyConfigEnvVars()` 將設定中的環境變數加入 `process.env`
4. `resolveConfigIncludes()` 處理 `includes` 指令
5. `validateConfigObjectWithPlugins()` 驗證載入的設定

### 1.3 Include 指令

設定檔支援 `includes` 指令來引入其他設定片段：

```typescript
// source-repo/src/config/includes.ts
// resolveConfigIncludes() 處理 includes 指令
```

### 1.4 環境變數替換

```typescript
// source-repo/src/config/env-substitution.ts:1-200+
// resolveConfigEnvVars()
// 替換 ${VAR_NAME} 佔位符（僅大寫字母）
// 模式: /^[A-Z_][A-Z0-9_]*$/ (Line 27)
```

關鍵特性：
- **僅大寫**：只有 `${UPPERCASE_VAR}` 會被替換
- **逃脫語法**：`$${VAR}` → `${VAR}`（字面值）（Line 51-60）
- **缺失處理**：未找到的變數觸發 `MissingEnvVarError`（Line 29-37），可透過 `onMissing` 回調自訂行為
- **偵測未解析**：`containsEnvVarReference()`（Line 137-166）偵測是否仍有未解析的 `${VAR}` 佔位符

```typescript
// source-repo/src/config/config-env-vars.ts:57-97
// collectConfigRuntimeEnvVars() (Line 57-59) - 收集 cfg.env.vars
// applyConfigEnvVars() (Line 79-97) - 套用到 process.env
// 跳過含未解析 ${VAR} 的值
```

---

## 2. 設定 I/O 與審計

### 2.1 審計系統

每次設定寫入都會留下審計記錄（`source-repo/src/config/io.audit.ts`）：

```typescript
// source-repo/src/config/io.audit.ts:8-47
// ConfigWriteAuditRecord - 寫入審計記錄
// 包含檔案 metadata、變更來源、結果

// source-repo/src/config/io.audit.ts:49-100
// ConfigObserveAuditRecord - 讀取審計記錄

// source-repo/src/config/io.audit.ts:102
// ConfigAuditRecord = union of both
```

審計日誌位置：`<STATE_DIR>/logs/config-audit.jsonl`

```typescript
// source-repo/src/config/io.audit.ts:311-337
// appendConfigAuditRecord() - 非同步附加到審計日誌
// appendConfigAuditRecordSync() - 同步附加（盡力而為）
// createConfigWriteAuditRecordBase() (Line 192-242) - 基礎審計記錄
// finalizeConfigWriteAuditRecord() (Line 244-287) - 附加結果/錯誤資訊
```

### 2.2 寫入前準備

```typescript
// source-repo/src/config/io.write-prepare.ts
// createMergePatch() (Line 14-47) - 產生最小的 JSON patch
// projectSourceOntoRuntimeShape() (Line 49-62) - 保留運行時已知的 key
// resolvePersistCandidateForWrite() (Line 64-72) - 合併來源與運行時變更
// unsetPathForWrite() (Line 195+) - 移除指定 key
// restoreEnvRefsFromMap() - 寫入前恢復 ${VAR} 佔位符
```

`restoreEnvRefsFromMap()` 是一個關鍵函式——它確保當設定透過 API 修改後重新寫入磁碟時，原本的 `${VAR_NAME}` 佔位符會被恢復，而不是寫入已解析的值。

### 2.3 合併補丁

```typescript
// source-repo/src/config/merge-patch.ts
// merge-patch 實作，用於設定的增量更新
// 支援原型污染防護 (merge-patch.proto-pollution.test.ts)
```

---

## 3. 運行時設定重載

### 3.1 熱重載架構

```typescript
// source-repo/src/gateway/config-reload.ts:77-284
export function startGatewayConfigReloader(/* ... */) {
  // Line 239: 使用 chokidar 監控檔案變更
  // Line 114-115, 59-71: 防抖（debounce）設定
  // Line 168: 呼叫 buildGatewayReloadPlan() 決定 hot vs restart
  // Line 150-157: 設定無效時跳過重載
  // Line 133-148: 設定檔缺失時重試（最多 2 次）
}
```

### 3.2 重載設定

```typescript
// source-repo/src/gateway/config-reload.ts:59-71
// resolveGatewayReloadSettings()
// 讀取 cfg.gateway.reload.mode
// 模式：
//   "off"     - 不重載
//   "restart" - 完全重啟
//   "hot"     - 熱重載（不重啟）
//   "hybrid"  - 混合模式
// 預設 debounce: 300ms (Line 23)
```

### 3.3 差異偵測

```typescript
// source-repo/src/gateway/config-reload.ts:28-57
export function diffConfigPaths(
  oldConfig: unknown,
  newConfig: unknown,
): string[] {
  // 深度比較找出變更的設定路徑
  // Line 49-55: 陣列使用 isDeepStrictEqual 結構比較
}
```

差異偵測結果會被餵給 `buildGatewayReloadPlan()`，它根據變更的路徑決定：
- **Hot reload**：只更新變更的部分（如 agent 設定）
- **Restart**：需要完全重啟（如 gateway 綁定地址變更）

### 3.4 運行時設定快照

```typescript
// source-repo/src/config/runtime-snapshot.ts
// 運行時設定快照的狀態管理
// 寫入時透過 RuntimeConfigWriteNotification 通知監聽者
```

### 3.5 運行時設定解析

```typescript
// source-repo/src/gateway/server-runtime-config.ts:40-186
// resolveGatewayRuntimeConfig()
// 在運行時解析：綁定主機、認證、Tailscale、Control UI 設定
// 驗證約束（例如 Tailscale 需要 loopback 綁定）
```

### 3.6 運行時 Schema

```typescript
// source-repo/src/config/runtime-schema.ts:1-50
// loadGatewayRuntimeConfigSchema() - 載入含 plugin 的設定 schema
// readBestEffortRuntimeConfigSchema() - 盡力載入（設定無效時回退）
```

---

## 4. Secret Ref 驗證與解析

### 4.1 Secret Ref 的遮蔽

```typescript
// source-repo/src/config/redact-snapshot.secret-ref.ts:1-21
// isSecretRefShape() (Line 1-5) - 檢查值是否有 source 和 id 屬性
// redactSecretRefId() (Line 7-20) - 遮蔽 secret ID 以安全顯示
// Line 15: 跳過環境變數佔位符的遮蔽
```

### 4.2 Telegram / Slack 特殊驗證

```typescript
// source-repo/src/config/zod-schema.secret-input-validation.ts:44-72
// validateTelegramWebhookSecretRequirements()
// Line 50-56: 當設定了 webhookUrl 時，webhookSecret 為必填

// source-repo/src/config/zod-schema.secret-input-validation.ts:74-102
// validateSlackSigningSecretRequirements()
// Slack 的 signing secret 驗證
```

### 4.3 Secret 解析限制

Secret 的解析有嚴格的資源限制（`source-repo/src/secrets/resolve.ts:161-174`）：

```typescript
// 並行數：最多 4 個 provider 同時解析
maxProviderConcurrency: normalizePositiveInt(resolution?.maxProviderConcurrency, 4)

// 每 provider 參照數上限：512
maxRefsPerProvider: normalizePositiveInt(resolution?.maxRefsPerProvider, 512)

// 批次大小上限：256KB
maxBatchBytes: normalizePositiveInt(resolution?.maxBatchBytes, 256 * 1024)
```

### 4.4 檔案 Provider 安全

檔案型別 secret provider 的安全措施（`source-repo/src/secrets/resolve.ts:202-300+`）：

- **絕對路徑強制**（Windows UNC 路徑支援）
- **符號連結處理**：允許一層，使用 realpath 解析
- **權限驗證**：檔案權限必須為 `0o600`，且為使用者所有
- **受信任目錄強制**（若有設定）
- **超時**：預設 5000ms
- **大小限制**：預設 1MB

### 4.5 JSON Pointer 實作

檔案型別 secret 使用 JSON Pointer（RFC 6901）來定位密鑰值：

```typescript
// source-repo/src/secrets/json-pointer.ts:1-99
// JSON pointer 規範實作
// 如 "/providers/openai/apiKey" → 取出 JSON 檔案中 providers.openai.apiKey 的值
```

### 4.6 Ref Contract 驗證

```typescript
// source-repo/src/secrets/ref-contract.ts:1-109
// FILE_SECRET_REF_ID_PATTERN: RFC 6901 JSON pointer 格式
// EXEC_SECRET_REF_ID_PATTERN: /^[A-Za-z0-9][A-Za-z0-9._:/-]{0,255}$/
//   不允許 "." 或 ".." 路徑段
// SECRET_PROVIDER_ALIAS_PATTERN: /^[a-z][a-z0-9_-]{0,63}$/
```

---

## 5. 設定物質化（Materialization）

「物質化」是將驗證後的設定轉換為運行時可用的完整設定的過程：

```typescript
// source-repo/src/config/materialize.ts
// materializeRuntimeConfig()
// 套用預設值、解析相對路徑、計算衍生值
```

物質化包括：
1. 套用所有預設值（model defaults、agent defaults 等）
2. 解析相對路徑為絕對路徑
3. 計算衍生設定值
4. 正規化通道設定
5. 產出不可變的運行時快照

---

## 6. 設定驗證的橫切模式

### 6.1 防禦性層疊

設定驗證體現了 OpenClaw 的「防禦性深度」原則：

```
來源1（設定檔） ──→ JSON5 解析 ──→ 環境變數替換
來源2（ENV 變數）───────────────────→ │
來源3（CLI 旗標）───────────────────→ 合併
                                       │
                                       ▼
                          Legacy 偵測（攔截舊格式）
                                       │
                                       ▼
                          Zod 驗證（型別 + 結構）
                                       │
                                       ▼
                          語義驗證（跨欄位邏輯）
                                       │
                                       ▼
                          Plugin Schema 合併
                                       │
                                       ▼
                          物質化（預設值 + 路徑解析）
                                       │
                                       ▼
                          運行時快照（不可變）
```

### 6.2 Key 原則

1. **Zod 是唯一的真理來源**：所有的型別驗證都由 Zod 負責
2. **JSON Schema 是衍生品**：從 Zod 產出，不手動維護
3. **Plugin Schema 是擴展**：透過 JSON Schema 合併，不汙染核心 schema
4. **Secret Ref 是特殊公民**：有獨立的解析管線和安全約束
5. **審計無例外**：每次寫入都留下 JSONL 審計記錄

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/config/defaults.ts:68-300` | 預設值套用函式 |
| `source-repo/src/config/io.ts:1-1813` | 設定檔 I/O 完整實作 |
| `source-repo/src/config/io.audit.ts:8-337` | 審計系統 |
| `source-repo/src/config/io.write-prepare.ts:14-195` | 寫入前準備 |
| `source-repo/src/config/env-substitution.ts:27-166` | 環境變數替換 |
| `source-repo/src/config/config-env-vars.ts:57-97` | 設定環境變數套用 |
| `source-repo/src/config/includes.ts` | Include 指令 |
| `source-repo/src/config/merge-patch.ts` | 合併補丁 |
| `source-repo/src/config/materialize.ts` | 設定物質化 |
| `source-repo/src/config/runtime-snapshot.ts` | 運行時快照 |
| `source-repo/src/config/runtime-schema.ts:1-50` | 運行時 Schema |
| `source-repo/src/config/redact-snapshot.secret-ref.ts:1-21` | Secret Ref 遮蔽 |
| `source-repo/src/config/zod-schema.secret-input-validation.ts:44-102` | Telegram/Slack 驗證 |
| `source-repo/src/gateway/config-reload.ts:23-284` | 設定熱重載 |
| `source-repo/src/gateway/server-runtime-config.ts:40-186` | 運行時設定解析 |
| `source-repo/src/secrets/resolve.ts:161-300` | Secret 解析與安全 |
| `source-repo/src/secrets/json-pointer.ts:1-99` | JSON Pointer 實作 |
| `source-repo/src/secrets/ref-contract.ts:1-109` | Ref Contract 驗證 |
