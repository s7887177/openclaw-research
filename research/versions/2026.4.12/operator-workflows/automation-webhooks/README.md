# Topic 3: 自動化、Webhook 與排程（Automation, Webhooks & Cron）

本章涵蓋 OpenClaw 的事件驅動自動化能力。

## 章節索引

| 檔案 | 內容 |
|------|------|
| [01-Hooks系統.md](./01-Hooks系統.md) | Hook 架構、類型、載入、Gmail 整合 |
| [02-Cron排程系統.md](./02-Cron排程系統.md) | Cron 排程、投遞機制、Webhook |

## 摘要

OpenClaw 提供兩大自動化機制：**Hooks**（事件驅動的程式碼鉤子）與 **Cron**（時間驅動的排程任務）。Hook 可以在特定事件（如收到訊息、開始會話）發生時自動觸發自訂邏輯；Cron 則在指定時間或間隔自動執行任務。兩者都支援將結果投遞到 Channel 或 Webhook。
