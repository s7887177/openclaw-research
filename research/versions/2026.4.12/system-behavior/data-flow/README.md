# Data Flow — 請求回應資料流

本章追蹤一則訊息在 OpenClaw 系統中從接收到回應的完整流動路徑，包含 Hook、串流、錯誤處理與記憶讀寫時機。

## 章節索引

| 檔案 | 內容 |
|------|------|
| [01-完整請求回應流程.md](./01-完整請求回應流程.md) | 以 Discord 訊息為例的端到端資料流、Hook 觸發、串流與錯誤處理 |

## 前置知識

- 了解 OpenClaw 架構與 Session 模型（見前兩章）
- 基本的 LLM API 呼叫概念（chat completion、tool use）
