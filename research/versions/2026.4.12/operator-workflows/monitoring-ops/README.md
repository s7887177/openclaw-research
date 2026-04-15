# Topic 4: 監控、日誌與故障排除（Monitoring & Ops）

本章涵蓋 OpenClaw 的運維監控能力。

## 章節索引

| 檔案 | 內容 |
|------|------|
| [01-健康檢查與日誌.md](./01-健康檢查與日誌.md) | Healthcheck、Logger、WS 日誌、Channel 監控 |
| [02-診斷與可觀測性.md](./02-診斷與可觀測性.md) | OpenTelemetry、Session Logs、Model Usage、Doctor 診斷 |

## 摘要

OpenClaw 提供多層次的監控能力：Gateway 的 `/healthz` 與 `/readyz` 探測端點、結構化的日誌系統（含 WebSocket 通訊日誌）、Channel 健康監控器、OpenTelemetry 整合、以及 Doctor 命令的自動化診斷。操作者可以透過這些工具全面掌握系統健康狀態。
