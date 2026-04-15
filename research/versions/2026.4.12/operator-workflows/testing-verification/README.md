# Topic 7: 測試策略與品質驗證（Testing & Verification）

本章涵蓋 OpenClaw 的完整測試策略與品質保證流程。

## 章節索引

| 檔案 | 內容 |
|------|------|
| [01-測試框架與工具.md](./01-測試框架與工具.md) | Vitest、型別檢查、Linting、Pre-commit Hooks |
| [02-QA與CI流程.md](./02-QA與CI流程.md) | QA 場景、Multipass Runner、CI Pipeline、Plugin 驗證 |

## 摘要

OpenClaw 採用多層次的品質保證策略：Vitest 單元/整合測試框架（66 個專案配置）、TypeScript strict mode 型別檢查、oxlint 靜態分析、oxfmt 格式化、pre-commit hooks（15+ 項自動化檢查）、QA 場景測試（45 個 Markdown 場景檔）、Multipass VM runner、以及 CI 流水線（多平台支援）。測試檔案採用共置模式（co-location），與源碼放在同一目錄。
