# 記憶生命週期（Memory Lifecycle）

本目錄深入剖析 OpenClaw 記憶系統的完整生命週期——從對話寫入、語義檢索、到壓縮歸檔的全流程。記憶是 OpenClaw 的核心橫切機制之一，它跨越 context-engine、gateway、extensions 與 plugin-sdk 多個元件，使 agent 能在長期對話中保持上下文連貫性。

## 章節索引

| 檔案 | 主題 | 字數概估 |
|------|------|----------|
| [01-context-engine-interface.md](./01-context-engine-interface.md) | ContextEngine 介面與三大核心操作（assemble / ingest / compact） | ~8,000 |
| [02-memory-backends-and-qmd.md](./02-memory-backends-and-qmd.md) | 多記憶後端切換、QMD 格式、Active Memory、Session Files 管理 | ~9,000 |

## 核心概念速覽

- **ContextEngine**：可插拔的記憶管理介面，定義 ingest / assemble / compact 三大操作
- **Memory Backend**：記憶的底層儲存引擎（memory-core SQLite、memory-lancedb、QMD）
- **Active Memory**：sub-agent 驅動的預提示記憶回調機制
- **QMD**：Query Markdown Document，本地優先的混合搜尋側車（BM25 + 向量 + 重排序）
- **Session Files**：對話記錄的磁碟持久化與歸檔管理

## 關鍵原始碼路徑

| 路徑 | 說明 |
|------|------|
| `src/context-engine/types.ts` | ContextEngine 介面定義 |
| `src/context-engine/registry.ts` | 引擎註冊與解析 |
| `src/context-engine/delegate.ts` | 壓縮委派到內建運行時 |
| `extensions/memory-core/` | 內建 SQLite 記憶後端 |
| `extensions/memory-lancedb/` | LanceDB 向量記憶後端 |
| `extensions/active-memory/` | Active Memory 子代理插件 |
| `packages/memory-host-sdk/` | 記憶主機 SDK |
| `src/gateway/session-transcript-files.fs.ts` | Session 檔案管理 |
| `src/gateway/embeddings-http.ts` | 向量嵌入 HTTP 端點 |
