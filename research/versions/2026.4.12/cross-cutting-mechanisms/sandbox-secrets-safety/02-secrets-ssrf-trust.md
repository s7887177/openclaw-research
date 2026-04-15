# 秘密解析、SSRF 防護與信任模型

## 本章摘要

本章分析 OpenClaw 安全體系的三個支柱：**秘密管理**（如何安全地將 API key、token 等敏感資料注入 agent 配置）、**SSRF 防護**（防止 agent 透過伺服器端請求存取內部網路）、以及**信任模型**（如何動態評估 agent 在特定環境中的安全等級）。

---

## 1. 秘密管理概述

OpenClaw 的秘密管理系統位於 `src/secrets/`，負責：
- 解析配置中的秘密引用（secret refs）
- 從不同的 provider 取得秘密值
- 確保秘密在傳輸和使用過程中的安全性

### 1.1 目錄結構

```
src/secrets/
├── apply.ts          # 將解析後的秘密套用到配置
├── resolve.ts        # 秘密解析引擎（核心）
├── ref-contract.ts   # 秘密引用的格式定義
├── secret-value.ts   # 秘密值型別
├── shared.ts         # 共用工具
└── json-pointer.ts   # JSON Pointer 解析
```

---

## 2. 秘密引用格式

### 2.1 Ref Contract

```typescript
// source-repo/src/secrets/ref-contract.ts:1-42
// SecretRef 格式：
// provider://path/to/secret#key
// 或
// provider://path/to/secret
```

秘密引用是一個 URI 格式的字串，包含：
- **provider**：秘密來源（如 `env`、`file`、`vault`）
- **path**：秘密在 provider 中的路徑
- **key**（可選）：如果秘密是 JSON，指定要提取的鍵

### 2.2 引用驗證

```typescript
// source-repo/src/secrets/ref-contract.ts:15-30
// parseSecretRef()
// 驗證格式、提取 provider, path, key
// 無效格式拋出 InvalidSecretRefError
```

### 2.3 在配置中使用

```yaml
# 配置中的秘密引用範例
openai:
  apiKey: ${{ secrets.OPENAI_API_KEY }}
# 或
discord:
  token: file:///run/secrets/discord-token
```

---

## 3. 秘密解析引擎

### 3.1 解析流程

```typescript
// source-repo/src/secrets/resolve.ts:1-80
// resolveSecrets()
// 主要的秘密解析函式
```

解析流程：

```
配置中發現秘密引用
  │
  ├── 解析 ref → { provider, path, key }
  │
  ├── 按 provider 分組
  │   └── 每個 provider 批次解析
  │
  ├── 安全限制檢查
  │   ├── maxProviderConcurrency = 4
  │   ├── maxRefsPerProvider = 512
  │   └── maxBatchBytes = 256KB
  │
  ├── Provider 回傳秘密值
  │
  └── 套用到配置（replace refs with values）
```

### 3.2 安全限制

```typescript
// source-repo/src/secrets/resolve.ts:15-20
// 硬編碼的安全限制：
// maxProviderConcurrency = 4    - 最多 4 個 provider 同時解析
// maxRefsPerProvider = 512      - 每個 provider 最多 512 個引用
// maxBatchBytes = 256 * 1024    - 每批最多 256KB
```

這些限制防止：
- **DoS**：限制並發和批量大小
- **資料外洩**：限制單次可解析的秘密數量
- **記憶體溢出**：限制批次大小

### 3.3 秘密套用

```typescript
// source-repo/src/secrets/apply.ts:1-45
// applySecrets()
// 使用 JSON Pointer 將解析後的秘密值寫入配置物件
```

### 3.4 JSON Pointer

```typescript
// source-repo/src/secrets/json-pointer.ts:1-30
// resolveJsonPointer()
// 依據 RFC 6901 解析 JSON Pointer
// 例如: "/openai/apiKey" → config.openai.apiKey
```

---

## 4. 檔案 Provider 安全

檔案 provider 是最常用的秘密 provider 之一，但也是安全風險最高的：

```typescript
// source-repo/src/secrets/shared.ts:1-35
// File provider 安全措施：
// - 只允許絕對路徑（防止路徑穿越）
// - 符號連結只跟隨 1 層（防止 symlink 攻擊）
// - 檔案權限要求 0o600（僅擁有者可讀寫）
// - 讀取超時 5000ms（防止掛載點阻塞）
// - 檔案大小上限 1MB（防止記憶體耗盡）
```

| 安全措施 | 值 | 目的 |
|---------|-----|------|
| 路徑限制 | 僅絕對路徑 | 防止 `../../etc/passwd` |
| Symlink 深度 | 最多 1 層 | 防止 symlink 循環/攻擊 |
| 檔案權限 | `0o600` | 確保只有擁有者可讀 |
| 讀取超時 | 5000ms | 防止掛載點阻塞導致 hang |
| 大小上限 | 1MB | 防止大檔案消耗記憶體 |

---

## 5. Plugin SDK 秘密介面

```typescript
// source-repo/src/secrets/secret-value.ts:1-20
// SecretValue 型別
// - value: string（實際秘密值）
// - metadata?: 附加資訊
```

Plugin 透過 SDK 可以：
- 宣告需要的秘密（在 manifest 中）
- 在初始化時收到已解析的秘密值
- 無法直接存取秘密 provider

---

## 6. SSRF 防護

### 6.1 什麼是 SSRF

SSRF（Server-Side Request Forgery，伺服器端請求偽造）是指攻擊者讓伺服器代為發送請求，存取內部網路資源。在 AI agent 場景中，agent 可能被提示工程（prompt injection）誘導去存取內部 API 或雲端 metadata endpoint。

### 6.2 核心實作

```typescript
// source-repo/src/infra/net/ssrf.ts:1-80
// SsrfBlockedError extends Error
// checkSsrf() - 主要的 SSRF 檢查函式
```

### 6.3 封鎖主機名

```typescript
// source-repo/src/infra/net/ssrf.ts:10-25
// BLOCKED_HOSTNAMES:
// - "localhost"
// - "metadata.google.internal"    // GCP metadata
// - "169.254.169.254"             // AWS/Azure metadata
// - 其他雲端 metadata endpoints
```

雲端 metadata endpoint（`169.254.169.254`）是 SSRF 攻擊的首要目標——它可以洩漏 IAM token、服務帳號金鑰等敏感資訊。

### 6.4 私有 IP 封鎖

```typescript
// source-repo/src/infra/net/ssrf.ts:30-55
// isPrivateIp()
// 封鎖的 IP 範圍：
// - 10.0.0.0/8        (Private A)
// - 172.16.0.0/12     (Private B)
// - 192.168.0.0/16    (Private C)
// - 127.0.0.0/8       (Loopback)
// - 169.254.0.0/16    (Link-local)
// - ::1               (IPv6 loopback)
// - fc00::/7          (IPv6 private)
// - fe80::/10         (IPv6 link-local)
```

### 6.5 DNS Pinning

```typescript
// source-repo/src/infra/net/ssrf.ts:57-75
// DNS pinning：先解析 DNS，再檢查 IP
// 防止 DNS rebinding 攻擊
```

DNS rebinding 攻擊的流程：
1. 攻擊者設定 DNS 記錄指向公開 IP（通過檢查）
2. DNS TTL 極短，第二次查詢時指向內部 IP
3. 應用程式使用第二次的 IP 發送請求

DNS pinning 的對策：解析 DNS 後直接使用 IP 連線，不再二次查詢。

### 6.6 Plugin SDK SSRF 介面

```typescript
// source-repo/src/plugin-sdk/ssrf-runtime.ts:1-27
// 匯出給 plugin 使用的 SSRF 安全函式：
// - safeFetch() - 安全的 fetch wrapper
// - checkUrl() - URL 安全檢查
```

Plugin 開發者不需自行實作 SSRF 防護——SDK 提供了安全的 HTTP 函式。

---

## 7. 信任模型

### 7.1 安全稽核型別

```typescript
// source-repo/src/security/audit.types.ts:1-30
// SecurityAuditSeverity = "info" | "warn" | "critical"
//
// SecurityAuditFinding = {
//   id: string;
//   severity: SecurityAuditSeverity;
//   title: string;
//   description: string;
//   recommendation: string;
// }
```

### 7.2 曝光度 × 工具權限矩陣

OpenClaw 的信任模型基於兩個維度的交叉評估：

| | 低權限工具 | 高權限工具 |
|---|---|---|
| **封閉群組** | 🟢 低風險 | 🟡 中風險 |
| **開放群組** | 🟡 中風險 | 🔴 高風險（CRITICAL） |

核心邏輯：
- 開放群組（任何人都可加入）+ 高權限工具（如 `execute_command`）= **CRITICAL** 安全問題
- 這是因為開放群組中的任何人都可以透過 prompt injection 操控 agent

### 7.3 Exposure 檢查

```typescript
// source-repo/src/security/ 目錄下的曝光度檢查
// 評估 agent 所在群組的開放程度
// 結合 agent 擁有的工具權限
// 產生安全稽核報告
```

---

## 8. 安全邊界的分層防禦

OpenClaw 的安全模型遵循**縱深防禦**（Defense in Depth）原則：

```
第 1 層：Exec Policy（命令執行前）
  ├── deny / allowlist / full
  ├── Safe Bin Trust 驗證
  └── 使用者確認（ask）

第 2 層：Sandbox（執行環境）
  ├── 容器隔離
  ├── readOnly root + tmpfs
  ├── capDrop ALL
  └── 資源限制（memory, cpu, pids）

第 3 層：Network（網路層）
  ├── SSRF 防護（DNS pinning + IP blacklist）
  ├── 私有 IP 封鎖
  └── Metadata endpoint 封鎖

第 4 層：Secrets（資料保護）
  ├── Provider 隔離
  ├── 批量限制
  ├── 檔案權限檢查
  └── JSON Pointer 精確注入

第 5 層：Trust Model（風險評估）
  ├── 群組曝光度評估
  ├── 工具權限分析
  └── 安全稽核報告
```

每一層都獨立運作，即使某一層被繞過，其他層仍然提供保護。

---

## 9. 安全配置的橫切模式

安全機制橫切所有 OpenClaw 元件：

```
Plugin 載入時
  └── Secret Ref 解析 → 秘密注入

Agent 收到訊息時
  └── Trust Model 評估 → 安全稽核

Agent 呼叫工具時
  ├── Exec Policy 評估 → 允許/拒絕
  ├── Sandbox Mode 決策 → 隔離級別
  └── SSRF Check（如果是 HTTP 請求）

Agent 寫入回應時
  └── Send Policy 評估 → 允許/拒絕

配置變更時
  ├── Zod 驗證 → 格式正確性
  └── Secret Ref 重新解析 → 秘密更新
```

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/secrets/ref-contract.ts:1-42` | 秘密引用格式 |
| `source-repo/src/secrets/resolve.ts:1-80` | 秘密解析引擎 |
| `source-repo/src/secrets/apply.ts:1-45` | 秘密套用 |
| `source-repo/src/secrets/json-pointer.ts:1-30` | JSON Pointer |
| `source-repo/src/secrets/shared.ts:1-35` | 檔案 provider 安全 |
| `source-repo/src/secrets/secret-value.ts:1-20` | 秘密值型別 |
| `source-repo/src/plugin-sdk/ssrf-runtime.ts:1-27` | SDK SSRF 介面 |
| `source-repo/src/infra/net/ssrf.ts:1-80` | SSRF 防護核心 |
| `source-repo/src/security/audit.types.ts:1-30` | 安全稽核型別 |
