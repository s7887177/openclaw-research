# Auto-Reply — 自動回覆子系統

> **版本**：OpenClaw v2026.4.12
> **對應原始碼**：`source-repo/src/auto-reply/`

本目錄深入剖析 OpenClaw 的自動回覆（Auto-Reply）子系統——負責接收、解析、派發與回覆所有進站訊息的核心引擎。

---

## 章節索引

| 章節 | 檔案 | 說明 |
|------|------|------|
| 01 | [01-auto-reply-訊息派發與回覆引擎.md](./01-auto-reply-訊息派發與回覆引擎.md) | 訊息派發流程（dispatch）、回覆引擎（reply engine）、訊息上下文（MsgContext）、信封格式化（envelope formatting）、命令註冊表（command registry）與回覆子目錄（reply/ subdirectory）的完整架構解析。 |
| 02 | [02-auto-reply-輔助模組與狀態管理.md](./02-auto-reply-輔助模組與狀態管理.md) | 分塊處理（chunking）、心跳回覆（heartbeat）、Token 管理、思考模式（thinking）、群組啟動、媒體註解、模型選擇、技能指令、指令認證、防抖動、發送策略、狀態監控等輔助模組全覽。 |

---

## 快速導覽

`src/auto-reply/` 是整個 OpenClaw 訊息處理流水線的核心入口。當一則來自任何通道（Discord、WhatsApp、Telegram 等）的訊息抵達時，它會經過以下流程：

```
Inbound Message
  → dispatch.ts (派發入口)
    → finalizeInboundContext() (上下文定案，default-deny)
    → withReplyDispatcher() (生命週期管理)
      → dispatchReplyFromConfig() (根據設定派發回覆)
        → reply/ subdirectory (200+ 檔案的回覆引擎)
```

此目錄包含約 90+ 個非測試 TypeScript 檔案，`reply/` 子目錄更有 200+ 個檔案，涵蓋 Agent 執行引擎、指令處理、會話管理、ACP 派發等完整功能。
