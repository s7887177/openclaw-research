# 串流通訊協定（Streaming Protocol）

本目錄剖析 OpenClaw 的串流通訊協定——從 Gateway 的 WebSocket 框架格式、SSE 串流、到各通道的差異化串流處理。串流是 OpenClaw 實現即時對話體驗的核心機制，它橫跨 gateway、channels、plugin-sdk 和客戶端多個層次。

## 章節索引

| 檔案 | 主題 | 字數概估 |
|------|------|----------|
| [01-gateway-protocol-and-websocket.md](./01-gateway-protocol-and-websocket.md) | Gateway 協定框架、WebSocket 訊息格式、SSE 串流、廣播系統 | ~10,000 |
| [02-channel-streaming-and-clients.md](./02-channel-streaming-and-clients.md) | 通道級串流差異、客戶端消費、錯誤處理與重連 | ~7,000 |

## 核心概念速覽

- **Gateway Frame**：三種框架型別（req/res/event）的 JSON 通訊格式
- **SSE**：Server-Sent Events，用於 HTTP 串流回應
- **Draft Stream**：通道級的節流串流機制
- **Provider Stream**：LLM 回應的串流抽象層
- **Broadcast**：Gateway 的事件廣播系統，含慢消費者偵測

## 關鍵原始碼路徑

| 路徑 | 說明 |
|------|------|
| `src/gateway/protocol/schema/frames.ts` | Gateway 框架 schema 定義 |
| `src/gateway/protocol/schema/error-codes.ts` | 錯誤碼與重試策略 |
| `src/gateway/http-common.ts` | SSE 標頭設定 |
| `src/gateway/server-broadcast.ts` | 廣播系統實作 |
| `src/channels/draft-stream-controls.ts` | 通道草稿串流控制 |
| `src/channels/draft-stream-loop.ts` | 串流迴圈與節流 |
| `src/plugin-sdk/channel-streaming.ts` | 通道串流型別 |
| `src/plugin-sdk/provider-stream-shared.ts` | Provider 串流抽象 |
