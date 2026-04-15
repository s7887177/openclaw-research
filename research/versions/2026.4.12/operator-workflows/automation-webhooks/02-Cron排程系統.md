# Cron 排程系統

## 本章摘要

本章深入介紹 OpenClaw 的 Cron 排程系統——一個功能完整的時間驅動自動化框架。Cron 讓操作者可以在指定的時間或間隔自動執行 AI 任務，並將結果投遞到 Channel 或 Webhook。系統支援三種排程方式：固定時間（at）、固定間隔（every）與 Cron 表達式（cron）。

---

## 1. Cron 概念

### 1.1 排程類型

OpenClaw 的 Cron 支援三種排程方式：

```typescript
export type CronSchedule =
  | { kind: "at"; at: string }
  | { kind: "every"; everyMs: number; anchorMs?: number }
  | { kind: "cron"; expr: string; tz?: string; staggerMs?: number };
```
> 引用：`source-repo/src/cron/types.ts:6-15`

| 類型 | 說明 | 範例 |
|------|------|------|
| `at` | 固定時間觸發 | `"08:00"` |
| `every` | 固定間隔觸發 | `everyMs: 3600000`（每小時）|
| `cron` | Cron 表達式 | `"0 9 * * 1-5"`（工作日 9:00）|

### 1.2 Session 目標

Cron 任務執行時可以指定 Session 類型：

```typescript
export type CronSessionTarget = "main" | "isolated" | "current" | `session:${string}`;
```
> 引用：`source-repo/src/cron/types.ts:17`

| 目標 | 說明 |
|------|------|
| `main` | 在主 Session 中執行 |
| `isolated` | 在獨立的隔離 Session 中執行 |
| `current` | 在當前 Session 中執行 |
| `session:xxx` | 在指定的 Session 中執行 |

### 1.3 喚醒模式

```typescript
export type CronWakeMode = "next-heartbeat" | "now";
```
> 引用：`source-repo/src/cron/types.ts:18`

- `next-heartbeat`：等待下一次心跳週期
- `now`：立即執行

---

## 2. 投遞系統（Delivery）

### 2.1 投遞模式

Cron 任務的結果可以透過三種模式投遞：

```typescript
export type CronDeliveryMode = "none" | "announce" | "webhook";
```
> 引用：`source-repo/src/cron/types.ts:22`

| 模式 | 說明 |
|------|------|
| `none` | 不投遞（僅執行）|
| `announce` | 投遞到 Channel |
| `webhook` | 投遞到 Webhook URL |

### 2.2 投遞配置

```typescript
export type CronDelivery = {
  mode: CronDeliveryMode;
  channel?: CronMessageChannel;     // 投遞的 Channel
  to?: string;                      // 投遞目標
  threadId?: string | number;       // 支援執行緒/話題
  accountId?: string;               // 多帳號場景的帳號 ID
  bestEffort?: boolean;             // 盡力投遞（失敗不重試）
  failureDestination?: CronFailureDestination;  // 失敗通知目的地
};
```
> 引用：`source-repo/src/cron/types.ts:24-35`

### 2.3 失敗通知

如果 Cron 任務執行失敗，可以配置單獨的通知目的地：

```typescript
export type CronFailureDestination = {
  channel?: CronMessageChannel;
  to?: string;
  accountId?: string;
  mode?: "announce" | "webhook";
};
```
> 引用：`source-repo/src/cron/types.ts:37-42`

### 2.4 投遞實作

投遞邏輯在 `delivery.ts` 中實作：

```typescript
// src/cron/delivery.ts
// 處理 Cron 任務結果的投遞
```
> 引用：`source-repo/src/cron/delivery.ts`

投遞計畫（Delivery Plan）：

```typescript
// src/cron/delivery-plan.ts
// 建立投遞計畫
// src/cron/delivery-field-schemas.ts
// 投遞欄位的 Schema 驗證
```
> 引用：`source-repo/src/cron/delivery-plan.ts`, `source-repo/src/cron/delivery-field-schemas.ts`

---

## 3. 執行狀態與遙測

### 3.1 執行狀態

```typescript
export type CronRunStatus = "ok" | "error" | "skipped";
export type CronDeliveryStatus = "delivered" | "not-delivered" | "unknown" | "not-requested";
```
> 引用：`source-repo/src/cron/types.ts:46-47`

### 3.2 使用量摘要

每次 Cron 任務執行都會記錄 Token 使用量：

```typescript
export type CronUsageSummary = {
  input_tokens?: number;
  output_tokens?: number;
  total_tokens?: number;
  cache_read_tokens?: number;
  cache_write_tokens?: number;
};
```
> 引用：`source-repo/src/cron/types.ts:49-55`

### 3.3 執行遙測

```typescript
export type CronRunTelemetry = {
  model?: string;
  provider?: string;
  usage?: CronUsageSummary;
};
```
> 引用：`source-repo/src/cron/types.ts:57-60`

---

## 4. Cron 服務核心

### 4.1 排程管理

```typescript
// src/cron/schedule.ts
// 排程計算與管理
```
> 引用：`source-repo/src/cron/schedule.ts`

### 4.2 Cron 服務

```typescript
// src/cron/service.ts
// Cron 服務主邏輯：
// - 管理排程計時器
// - 觸發任務執行
// - 處理投遞結果
```
> 引用：`source-repo/src/cron/service.ts`

服務的功能涵蓋多個面向，有大量的測試驗證：

| 測試檔案 | 驗證內容 |
|---------|---------|
| `service.every-jobs-fire.test.ts` | Every 類型任務正確觸發 |
| `service.failure-alert.test.ts` | 失敗警報投遞 |
| `service.heartbeat-ok-summary-suppressed.test.ts` | 心跳摘要抑制 |
| `service.prevents-duplicate-timers.test.ts` | 防止重複計時器 |
| `service.restart-catchup.test.ts` | 重啟後的追補執行 |
| `service.session-reaper-in-finally.test.ts` | Session 清理 |

> 引用：`source-repo/src/cron/service.*.test.ts`

### 4.3 儲存層

```typescript
// src/cron/store.ts
// Cron 任務的持久化儲存
```
> 引用：`source-repo/src/cron/store.ts`

### 4.4 心跳政策

```typescript
// src/cron/heartbeat-policy.ts
// 心跳檢測政策，確保 Cron 服務健康運行
```
> 引用：`source-repo/src/cron/heartbeat-policy.ts`

### 4.5 Stagger（錯開機制）

為避免大量 Cron 任務同時觸發，系統支援 Stagger 機制：

```typescript
// src/cron/stagger.ts
// 自動錯開同一時間點的多個任務
```
> 引用：`source-repo/src/cron/stagger.ts`

### 4.6 執行日誌

```typescript
// src/cron/run-log.ts
// 記錄每次 Cron 任務的執行結果
```
> 引用：`source-repo/src/cron/run-log.ts`

---

## 5. 隔離 Agent（Isolated Agent）

Cron 任務可以在隔離的 Agent 環境中執行，避免影響主 Session：

```typescript
// src/cron/isolated-agent/
// 隔離 Agent 的完整實作
```
> 引用：`source-repo/src/cron/isolated-agent`

隔離 Agent 的功能包括：
- 獨立的 Session 管理
- 獨立的模型選擇
- 安全的投遞上下文
- Hook 內容封裝

測試涵蓋多種場景：

| 測試 | 驗證內容 |
|------|---------|
| `isolated-agent.delivery.test-helpers.ts` | 投遞輔助 |
| `isolated-agent.direct-delivery-core-channels.test.ts` | 核心 Channel 直投 |
| `isolated-agent.direct-delivery-forum-topics.test.ts` | 論壇話題直投 |
| `isolated-agent.model-overrides.test.ts` | 模型覆蓋 |
| `isolated-agent.session-identity.test.ts` | Session 身份 |

> 引用：`source-repo/src/cron/isolated-agent.*.test.ts`

---

## 6. Cron CLI

### 6.1 管理命令

```bash
openclaw cron list      # 列出所有排程任務
openclaw cron add       # 新增排程任務
openclaw cron remove    # 移除排程任務
openclaw cron run       # 手動觸發任務
```

```typescript
// src/cli/cron-cli.ts
// Cron CLI 主命令
// src/cli/cron-cli/
// Cron CLI 子命令目錄
```
> 引用：`source-repo/src/cli/cron-cli.ts`, `source-repo/src/cli/cron-cli/`

---

## 7. Gateway Cron 整合

Cron 服務與 Gateway 的整合在 `server-cron.ts` 中：

```typescript
// src/gateway/server-cron.ts
// Gateway 啟動時初始化 Cron 服務
// 管理 Cron 計時器的生命週期
```
> 引用：`source-repo/src/gateway/server-cron.ts`

---

## 8. Webhook URL

Cron 支援將結果投遞到外部 Webhook：

```typescript
// src/cron/webhook-url.ts
// Webhook URL 驗證與處理
```
> 引用：`source-repo/src/cron/webhook-url.ts`

---

## 9. 正規化與驗證

### 9.1 任務身份正規化

```typescript
// src/cron/normalize-job-identity.ts
// 正規化 Cron 任務的身份標識
```
> 引用：`source-repo/src/cron/normalize-job-identity.ts`

### 9.2 排程正規化

```typescript
// src/cron/normalize.ts
// 正規化排程配置
```
> 引用：`source-repo/src/cron/normalize.ts`

### 9.3 時間戳驗證

```typescript
// src/cron/validate-timestamp.ts
// 驗證 Cron 時間戳的有效性
```
> 引用：`source-repo/src/cron/validate-timestamp.ts`

---

## 10. 活動任務管理

```typescript
// src/cron/active-jobs.ts
// 管理當前活動的 Cron 任務
```
> 引用：`source-repo/src/cron/active-jobs.ts`

### 10.1 Session Reaper

長時間運行的 Cron 任務會建立 Session，Session Reaper 負責清理過期的 Session：

```typescript
// src/cron/session-reaper.ts
// 清理 Cron 任務遺留的 Session
```
> 引用：`source-repo/src/cron/session-reaper.ts`

---

## 11. 配置格式

Cron 配置在 `openclaw.json` 中的結構，類型定義在：

```typescript
// src/config/types.cron.ts
// Cron 配置類型定義
```
> 引用：`source-repo/src/config/types.cron.ts`

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/cron/types.ts:1-60` | Cron 核心類型定義 |
| `source-repo/src/cron/types-shared.ts` | 共用類型 |
| `source-repo/src/cron/schedule.ts` | 排程計算 |
| `source-repo/src/cron/service.ts` | Cron 服務主邏輯 |
| `source-repo/src/cron/store.ts` | 持久化儲存 |
| `source-repo/src/cron/delivery.ts` | 投遞系統 |
| `source-repo/src/cron/delivery-plan.ts` | 投遞計畫 |
| `source-repo/src/cron/heartbeat-policy.ts` | 心跳政策 |
| `source-repo/src/cron/stagger.ts` | 任務錯開機制 |
| `source-repo/src/cron/run-log.ts` | 執行日誌 |
| `source-repo/src/cron/isolated-agent/` | 隔離 Agent |
| `source-repo/src/cron/webhook-url.ts` | Webhook URL |
| `source-repo/src/cron/session-reaper.ts` | Session 清理 |
| `source-repo/src/cron/active-jobs.ts` | 活動任務管理 |
| `source-repo/src/cli/cron-cli.ts` | Cron CLI 命令 |
| `source-repo/src/gateway/server-cron.ts` | Gateway Cron 整合 |
