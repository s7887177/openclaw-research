# Topic 6: 自訂 LLM Provider 開發（Custom Provider Development）

本章涵蓋 OpenClaw 自訂 LLM Provider 的完整開發指南。

## 章節索引

| 檔案 | 內容 |
|------|------|
| [01-Provider架構與介面.md](./01-Provider架構與介面.md) | Plugin 清單、Provider 介面、認證機制、Model Discovery |
| [02-實作與參考.md](./02-實作與參考.md) | HTTP 處理、Streaming、參考實作（OpenAI/Anthropic/Ollama）、驗證 |

## 摘要

OpenClaw 的 LLM Provider 系統採用 Plugin 架構。每個 Provider 是一個 Extension，透過 `openclaw.plugin.json` 清單宣告能力，並在 `register()` 回呼中透過 `api.registerProvider()` 註冊。Provider 需要實作認證流程、模型發現、串流回應等核心功能。系統透過 `packages/plugin-package-contract` 驗證外部 Plugin 的相容性。
