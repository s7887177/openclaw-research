# Session 管理、路由與線上狀態（Sessions, Routing & Presence）

本目錄剖析 OpenClaw 的 session 管理、訊息路由與使用者線上狀態機制。這些是 OpenClaw 實現多 agent、多通道對話的核心橫切關注點——每一則訊息都必須被路由到正確的 agent，在正確的 session 上下文中處理，並追蹤使用者的存在狀態。

## 章節索引

| 檔案 | 主題 | 字數概估 |
|------|------|----------|
| [01-sessions-and-routing.md](./01-sessions-and-routing.md) | Session 概念、Session Key、路由引擎、多 Agent 路由 | ~10,000 |
| [02-presence-retry-persistence.md](./02-presence-retry-persistence.md) | Presence、Retry 機制、Session 持久化、ACP 映射、Send Policy | ~7,000 |

## 核心概念速覽

- **Session Key**：`agent:agentId:rest` 格式的正規化路由鍵
- **Routing Engine**：8 層匹配層級的路由決策引擎
- **Bindings**：將 channel/account/peer 綁定到特定 agent 的規則
- **DM Scope**：控制 session 隔離粒度的策略
- **ACP Session Mapping**：IDE ↔ ACP ↔ Gateway 的 session 映射
- **Send Policy**：規則引擎驅動的訊息發送政策
- **Presence**：Discord 等平台的 bot 狀態追蹤

## 關鍵原始碼路徑

| 路徑 | 說明 |
|------|------|
| `src/sessions/` | Session 核心型別與工具 |
| `src/routing/resolve-route.ts` | 路由決策引擎 |
| `src/routing/session-key.ts` | Session key 建構與解析 |
| `src/routing/bindings.ts` | Binding 查找與索引 |
| `src/routing/account-id.ts` | 帳號 ID 正規化 |
| `src/gateway/session-lifecycle-state.ts` | Session 生命週期狀態 |
| `src/gateway/session-archive.fs.ts` | Session 歸檔 |
| `src/acp/session.ts` | ACP session 管理 |
| `src/sessions/send-policy.ts` | 發送政策引擎 |
