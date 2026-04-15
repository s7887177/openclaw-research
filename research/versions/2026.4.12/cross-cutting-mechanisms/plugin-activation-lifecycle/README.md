# Plugin 啟動生命週期（Plugin Activation Lifecycle）

本目錄研究 OpenClaw Plugin 從靜態 manifest 宣告到 runtime 動態執行的完整生命週期。

## 文件索引

| 檔案 | 內容 | 字數估計 |
|------|------|---------|
| [01-manifest-and-discovery.md](./01-manifest-and-discovery.md) | Manifest 解析、驗證、與 Plugin 發現機制 | ~8,000 |
| [02-loading-registration-execution.md](./02-loading-registration-execution.md) | 載入、註冊、啟動、執行與卸載 | ~9,000 |

## 核心問題

OpenClaw 擁有超過 100 個 extension（在 `extensions/` 目錄下），涵蓋 LLM provider、
通訊平台、記憶系統等。這些 extension 全部遵循統一的 Plugin 生命週期：

```
manifest 解析 → 發現掃描 → 啟動規劃 → 模組載入 → 函式註冊 → 全域啟用 → 執行 → 卸載
```

理解這個生命週期是理解 OpenClaw 所有其他機制的基礎。
