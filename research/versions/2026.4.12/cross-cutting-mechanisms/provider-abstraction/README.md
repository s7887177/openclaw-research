# Provider 抽象層（Provider Abstraction）

本目錄研究 OpenClaw 如何透過 Plugin SDK 中的 Provider 介面，
將 50+ LLM 提供者統一為相同的推理後端。

## 文件索引

| 檔案 | 內容 | 字數估計 |
|------|------|---------|
| [01-provider-interface-and-auth.md](./01-provider-interface-and-auth.md) | Provider 介面定義、認證抽象與模型發現 | ~8,000 |
| [02-runtime-streaming-and-tools.md](./02-runtime-streaming-and-tools.md) | Runtime 執行、串流處理、工具正規化與具體實作 | ~8,000 |

## 核心問題

OpenClaw 支援 OpenAI、Anthropic、Google、Mistral、Ollama 等超過 50 個 LLM 提供者。
每個提供者的 API 格式、認證方式、模型命名、串流協定都不同。

如何讓 Agent 的推理迴圈不需要關心底層用的是哪個 LLM？答案是 Provider Abstraction Layer。
