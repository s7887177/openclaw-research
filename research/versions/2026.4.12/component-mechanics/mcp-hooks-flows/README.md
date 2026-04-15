# MCP · Hooks · Flows 機制詳解

本目錄深入解析 OpenClaw v2026.4.12 中三個緊密相關的橫切面（cross-cutting）子系統：MCP 整合、Hook 事件系統、以及 Flow UI 貢獻系統。

## 章節索引

| 檔案 | 標題 | 說明 |
|------|------|------|
| [01-mcp-模型上下文協議整合.md](./01-mcp-模型上下文協議整合.md) | MCP 模型上下文協議整合 | `src/mcp/` 全 7 檔解析：channel server 建立、gateway 橋接器、MCP 工具註冊、共享型別、plugin 工具伺服 |
| [02-hooks-事件系統.md](./02-hooks-事件系統.md) | Hooks 事件系統 | `src/hooks/` 全 28+ 檔解析：事件型別、全域註冊表、優先序策略、載入器、plugin hook 探索、前言解析、訊息映射器 |
| [03-flows-UI貢獻系統.md](./03-flows-UI貢獻系統.md) | Flows UI 貢獻系統 | `src/flows/` 全 11 檔解析：Flow 型別系統、channel/doctor/model/provider/search 各 setup flow、plugin 貢獻機制 |

## 三者的關係

```
Plugin / Channel
      │
      ▼
┌──────────┐    觸發     ┌──────────┐    註冊      ┌──────────┐
│  Flows   │───────────▶│  Hooks   │◀────────────│   MCP    │
│ (UI 面)  │            │ (事件面) │             │ (協議面) │
└──────────┘            └──────────┘             └──────────┘
      │                       │                       │
      └───── 共同服務於 ──────┴──── Gateway 核心 ─────┘
```

- **MCP**：透過 Model Context Protocol 讓外部 AI 客戶端（如 Claude Desktop）與 OpenClaw 通訊
- **Hooks**：提供事件驅動的擴充點，讓 plugin 與內部模組在生命週期關鍵節點插入邏輯
- **Flows**：定義 UI 面的貢獻系統，讓 channel、provider、搜尋引擎等透過統一介面註冊選項
