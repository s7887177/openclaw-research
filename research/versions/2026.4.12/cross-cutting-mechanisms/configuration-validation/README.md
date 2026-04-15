# 設定驗證（Configuration Validation）

本目錄剖析 OpenClaw 的 Zod schema 驅動設定驗證模式——從 schema 定義、驗證執行、到設定層次的合併與運行時重載。設定驗證是 OpenClaw 的核心橫切機制，確保從 YAML 設定檔到環境變數再到 CLI 旗標的每一層設定都符合合約。

## 章節索引

| 檔案 | 主題 | 字數概估 |
|------|------|----------|
| [01-zod-schema-and-validation.md](./01-zod-schema-and-validation.md) | Zod Schema 架構、驗證執行、錯誤處理 | ~9,000 |
| [02-config-layers-and-runtime-reload.md](./02-config-layers-and-runtime-reload.md) | 設定層次、Plugin Config、運行時重載、Secret Ref 驗證 | ~8,000 |

## 核心概念速覽

- **OpenClawSchema**：主 Zod schema，定義整個 OpenClaw 設定結構
- **Config Layers**：defaults → file → env substitution → CLI → validation
- **Plugin configSchema**：插件自定義的 JSON Schema，合併到核心 schema
- **Hot Reload**：檔案監控 + 差異偵測 + 計畫（hot vs restart）
- **Secret Ref**：安全的密鑰引用機制（env / file / exec 三種來源）
- **Config Audit**：每次設定寫入都記錄審計日誌

## 關鍵原始碼路徑

| 路徑 | 說明 |
|------|------|
| `src/config/zod-schema.ts` | 主 Zod schema（OpenClawSchema） |
| `src/config/zod-schema.core.ts` | 核心子 schema（SecretRef、Models 等） |
| `src/config/validation.ts` | 驗證執行與錯誤映射 |
| `src/config/schema.ts` | Schema 建構與合併 |
| `src/config/schema-base.ts` | JSON Schema 產出 |
| `src/config/io.ts` | 設定檔 I/O |
| `src/config/defaults.ts` | 預設值套用 |
| `src/config/env-substitution.ts` | 環境變數替換 |
| `src/gateway/config-reload.ts` | 運行時設定重載 |
