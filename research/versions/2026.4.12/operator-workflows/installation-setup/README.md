# Topic 1: 安裝與設定（Installation & Setup）

本章涵蓋 OpenClaw v2026.4.12 的安裝、首次設定（Onboarding）與診斷（Doctor）流程。

## 章節索引

| 檔案 | 內容 |
|------|------|
| [01-系統需求與安裝.md](./01-系統需求與安裝.md) | 系統需求、安裝方式、配置檔位置 |
| [02-Onboarding與Doctor.md](./02-Onboarding與Doctor.md) | 首次設定精靈、Provider/Channel 選擇、Doctor 診斷 |

## 摘要

OpenClaw 是一個 Node.js 應用程式，透過 npm/pnpm/bun 安裝為全域 CLI 工具。首次安裝後，使用者執行 `openclaw onboard --install-daemon` 進入互動式設定精靈，依序完成 Provider 認證、Channel 配置與 Gateway Daemon 安裝。當系統出現問題時，`openclaw doctor` 提供超過 20 項自動化健康檢查與修復功能。
