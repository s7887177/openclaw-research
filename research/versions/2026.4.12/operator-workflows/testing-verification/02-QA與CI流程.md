# QA 場景與 CI 流程

## 本章摘要

本章介紹 OpenClaw 的 QA（品質保證）場景系統、Multipass VM Runner、CI Pipeline 配置與 Plugin 驗證契約。OpenClaw 採用獨特的 Markdown 驅動 QA 場景方法論，配合 Multipass 虛擬機 Runner 實現可復現的自動化測試。

---

## 1. QA 場景系統

### 1.1 概述

OpenClaw 的 QA 系統使用 Markdown 檔案定義測試場景。每個場景描述一個需要驗證的使用者故事或系統行為。

### 1.2 場景目錄

```
qa/
├── README.md                      # QA 總覽
├── scenarios/                     # 場景定義
│   ├── index.md                   # Pack-level 啟動資料
│   ├── active-memory-preprompt-recall.md
│   ├── approval-turn-tool-followthrough.md
│   ├── channel-chat-baseline.md
│   └── ... (45 個場景檔)
├── scenarios.md                   # 場景文件
├── new-scenarios-2026-04.md       # 新增場景
├── convex-credential-broker/      # Convex 憑證代理
└── frontier-harness-plan.md       # Frontier Harness 計畫
```
> 引用：`source-repo/qa/`

### 1.3 QA README

```markdown
# QA
# 場景的標準來源在 qa/scenarios/
# 關鍵工作流: qa suite (可執行 frontier/regression), qa manual (個性/風格探測)
# 保持此資料夾在 git 中。在接入自動化之前先在這裡新增場景。
```
> 引用：`source-repo/qa/README.md:1-17`

### 1.4 場景索引（index.md）

`qa/scenarios/index.md` 定義了 QA 操作者的身份和啟動任務：

- **身份**：C-3PO 人格（協議導向、精確、認真）
- **啟動任務**：「先從源碼 + 文件理解這個 OpenClaw 倉庫，再行動」
- **要求**：DM + Channel 行為、Lobster Invaders 建構、Cron 任務

> 引用：`source-repo/qa/scenarios/index.md`

### 1.5 場景類型

QA 場景涵蓋多個面向：

| 類別 | 範例場景 |
|------|---------|
| 記憶 | `active-memory-preprompt-recall.md` |
| 審批 | `approval-turn-tool-followthrough.md` |
| Channel | `channel-chat-baseline.md` |
| Cron | 排程相關場景 |
| 安全 | 權限與沙箱場景 |
| 工具 | 工具呼叫與結果驗證 |

### 1.6 QA 工作流

```bash
openclaw qa suite     # 可執行的 frontier/regression 測試
openclaw qa manual    # 個性/風格探測（需人工判斷）
```
> 引用：`source-repo/qa/README.md`

---

## 2. Multipass QA Runner

### 2.1 概述

OpenClaw 在 CHANGELOG 中描述了 Multipass QA Runner 功能：

> QA/testing: add a --runner multipass lane for openclaw qa suite so repo-backed QA scenarios can run inside a disposable Linux VM and write back the usual report, summary, and VM logs.

> 引用：`source-repo/CHANGELOG.md:17`（PR #63426）

### 2.2 功能

Multipass Runner 提供：
- **拋棄式 Linux VM**：每次測試在全新的虛擬機中執行
- **報告生成**：自動寫回測試報告、摘要和 VM 日誌
- **可重現**：標準化的測試環境確保結果一致

### 2.3 其他 QA Lane

CHANGELOG 中記錄了多種 QA Lane：

| PR | QA Lane | 說明 |
|----|---------|------|
| #65596 | QA/lab | Convex-backed Telegram 憑證租借 |
| #64303 | QA/Telegram | 私密群組 bot-to-bot 檢查 |
| #64489 | QA/Matrix | 基於拋棄式 Matrix homeserver 的測試 |

> 引用：`source-repo/CHANGELOG.md:11, 22, 125`

---

## 3. QA Lab Extension

### 3.1 建構

QA Lab 是一個專門的 Extension，在 Docker 建構中會被建構：

```dockerfile
RUN pnpm qa:lab:build
```
> 引用：`source-repo/Dockerfile:101`

### 3.2 Convex 憑證代理

```
qa/convex-credential-broker/
```
> 引用：`source-repo/qa/convex-credential-broker`

Convex 憑證代理讓 QA 測試可以安全地租借 Telegram 等 Channel 的測試憑證。

---

## 4. CI Pipeline

### 4.1 主 CI 工作流

```yaml
# .github/workflows/ci.yml

# 並行控制 (Lines 12-14):
#   取消進行中的 PR 建構
#   以 workflow + PR 號碼分組

# 環境 (Line 17):
#   FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: "true"
```
> 引用：`source-repo/.github/workflows/ci.yml:12-17`

### 4.2 Preflight Job

CI Pipeline 的第一個步驟是 Preflight，負責路由決策：

```yaml
# Preflight Job (Lines 22-57):
#   建立下游所有 Job 的路由矩陣
#   Output: docs, node, macOS, Android, Windows 測試矩陣
#   偵測 docs-only 變更
#   偵測變更的 Extension
#   決定哪些 Job 需要執行
```
> 引用：`source-repo/.github/workflows/ci.yml:22-57`

### 4.3 Base Commit 偵測

```yaml
# Lines 67-71:
#   比較 base branch / PR base
#   用於增量測試決策
```
> 引用：`source-repo/.github/workflows/ci.yml:67-71`

### 4.4 多平台支援

CI 支援在多個平台上執行測試：

| 平台 | 用途 |
|------|------|
| Linux (Node) | 主要測試平台 |
| macOS | 原生功能測試（launchd、iMessage 等）|
| Android | 行動裝置功能測試 |
| Windows | WSL2 相容性測試 |

---

## 5. Plugin 驗證契約

### 5.1 概述

`packages/plugin-package-contract` 定義了外部 Plugin 的驗證規則，確保第三方 Provider 與 OpenClaw 相容。

### 5.2 測試覆蓋

```typescript
// packages/plugin-package-contract/src/index.test.ts (86 行)

// 測試 1 (Lines 10-31): 正規化外部 Plugin 相容性區塊
// 測試 2 (Lines 33-51): 回退到 install.minHostVersion 和 package version
// 測試 3 (Lines 53-58): 列出必要欄位
// 測試 4 (Lines 60-84): 回報缺失的必要欄位
```
> 引用：`source-repo/packages/plugin-package-contract/src/index.test.ts:10-84`

### 5.3 驗證的必要欄位

```typescript
const EXTERNAL_CODE_PLUGIN_REQUIRED_FIELD_PATHS = [
  "openclaw.compat.pluginApi",        // Plugin API 版本範圍
  "openclaw.build.openclawVersion",   // 建構版本
] as const;
```
> 引用：`source-repo/packages/plugin-package-contract/src/index.ts:23-26`

---

## 6. Test Fixtures

### 6.1 目錄

```
test-fixtures/
└── talk-config-contract.json    # Talk 配置契約 (3,336 bytes)
```
> 引用：`source-repo/test-fixtures/`

大多數測試 Fixtures 是在測試執行時動態生成的，或存放在各測試檔案鄰近的目錄中。

### 6.2 CLI 測試 Fixtures

```
src/cli/requirements-test-fixtures.ts    # CLI 需求測試 fixtures
```
> 引用：`source-repo/src/cli/requirements-test-fixtures.ts`

---

## 7. 測試輔助工具

### 7.1 測試輔助模組

OpenClaw 有豐富的測試輔助工具：

```
src/test-helpers/     # 全域測試輔助
src/test-utils/       # 測試工具函式
src/gateway/test-helpers.*.ts    # Gateway 測試輔助（多個檔案）
```

### 7.2 Gateway 測試基礎設施

Gateway 有完整的測試基礎設施：

| 檔案 | 功能 |
|------|------|
| `test-helpers.server.ts` | 伺服器測試輔助 |
| `test-helpers.config-runtime.ts` | 配置執行時測試 |
| `test-helpers.mocks.ts` | Mock 物件 |
| `test-helpers.channels.ts` | Channel 測試 |
| `test-helpers.speech.ts` | 語音測試 |
| `test-helpers.agent-results.ts` | Agent 結果驗證 |
| `test-with-server.ts` | 內建伺服器的測試 |
| `server.e2e-ws-harness.ts` | E2E WebSocket Harness |

> 引用：`source-repo/src/gateway/test-helpers.*.ts`

---

## 8. 品質保證總覽

### 8.1 多層防線

```
Pre-commit Hooks (15+ 項)
    ↓
oxlint (型別感知靜態分析)
    ↓
oxfmt (格式化)
    ↓
TypeScript strict mode (型別檢查)
    ↓
Vitest (66 個專案, 單元/整合/E2E)
    ↓
Plugin Contract Validation (外部 Plugin)
    ↓
QA Scenarios (45 個 Markdown 場景)
    ↓
Multipass VM Runner (拋棄式 VM)
    ↓
CI Pipeline (Linux/macOS/Android/Windows)
    ↓
knip (死碼偵測)
```

### 8.2 測試層級

| 層級 | 工具 | 速度 | 覆蓋面 |
|------|------|------|--------|
| 靜態分析 | oxlint, TypeScript | 秒級 | 型別安全、程式碼品質 |
| 單元測試 | Vitest (test:fast) | 分鐘級 | 個別函式 |
| 整合測試 | Vitest (test) | 分鐘級 | 模組間互動 |
| E2E 測試 | Vitest (test:e2e) | 分鐘-小時級 | 完整流程 |
| QA 場景 | qa suite | 小時級 | 使用者故事 |
| 跨平台 | CI Pipeline | 小時級 | 多 OS 相容性 |

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/qa/README.md:1-17` | QA 總覽 |
| `source-repo/qa/scenarios/` | 45 個 QA 場景 |
| `source-repo/qa/scenarios/index.md` | QA 場景索引 |
| `source-repo/CHANGELOG.md:17, 11, 22, 125` | Multipass QA Runner 與其他 QA Lane |
| `source-repo/Dockerfile:101` | QA Lab 建構 |
| `source-repo/.github/workflows/ci.yml:12-71` | CI Pipeline 配置 |
| `source-repo/packages/plugin-package-contract/src/index.test.ts:10-84` | Plugin 驗證測試 |
| `source-repo/packages/plugin-package-contract/src/index.ts:23-26` | 必要欄位定義 |
| `source-repo/test-fixtures/` | 測試 Fixtures |
| `source-repo/src/gateway/test-helpers.*.ts` | Gateway 測試基礎設施 |
