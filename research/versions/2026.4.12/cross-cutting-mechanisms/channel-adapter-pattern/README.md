# Channel 適配器模式（Channel Adapter Pattern）

本目錄研究 OpenClaw 如何透過組合式適配器模式（Composition-based Adapter Pattern）
將 25+ 通訊平台統一為相同介面。

## 文件索引

| 檔案 | 內容 | 字數估計 |
|------|------|---------|
| [01-channel-plugin-interface.md](./01-channel-plugin-interface.md) | ChannelPlugin 介面定義與適配器組合 | ~8,000 |
| [02-normalization-and-transport.md](./02-normalization-and-transport.md) | 訊息正規化、傳輸層、核准流程與新增 channel 流程 | ~8,000 |

## 核心問題

OpenClaw 支援 Discord、Slack、Telegram、WhatsApp、Line、Matrix、IRC 等
超過 25 種通訊平台。每個平台的 API 完全不同：有的用 WebSocket，有的用 Webhook；
有的支援 Thread，有的不支援；有的能發送 Reaction，有的只能發文字。

如何讓核心 Agent 程式碼不需要知道這些差異？答案是 Channel Adapter Pattern。
