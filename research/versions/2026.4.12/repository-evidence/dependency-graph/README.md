# 依賴關係圖譜（Dependency Graph）——倉庫證據層索引

> **本主題摘要**：本主題系統性記錄 OpenClaw 原始碼倉庫中的依賴關係全貌，涵蓋三個維度：原始碼內部模組之間的引用拓撲、對外部 npm 套件的依賴清單、以及擴充（Extension）與核心（Core）之間透過 plugin-sdk 建立的隔離邊界。所有結論均直接來自原始碼分析，附有明確的檔案路徑與行號引用。

---

## 目錄

| # | 子章節 | 檔案 | 主題概述 |
|---|--------|------|----------|
| 1 | [src/ 內部模組依賴關係](./01-src-internal-module-dependencies.md) | `01-src-internal-module-dependencies.md` | 從 `entry.ts` 出發，追蹤 60+ 個 `src/` 子目錄之間的 import 關係，繪製模組間的依賴拓撲。重點分析入口層（entry layer）、CLI 層、基礎設施層（infra）、外觀層（library facade）各自的依賴模式。 |
| 2 | [外部依賴清單](./02-external-dependencies.md) | `02-external-dependencies.md` | 列舉 `package.json` 中全部 70+ 個 production dependencies 與 23 個 devDependencies，依功能分類（AI SDK、通道整合、Web 框架、向量資料庫、CLI 工具、媒體處理、TTS/STT 等），分析各類別在系統架構中的角色。 |
| 3 | [擴充/核心邊界與 plugin-sdk](./03-extension-core-boundary.md) | `03-extension-core-boundary.md` | 記錄 OpenClaw 如何透過 `plugin-sdk` 在核心程式碼與第三方擴充之間建立嚴格的 API 邊界。分析邊界檢查腳本的三條規則、TypeScript path alias 的映射機制、以及 workspace packages 的零依賴設計。 |

---

## 閱讀建議

本主題的三份文件彼此互補，建議依序閱讀：

1. **第 01 章**先建立對 `src/` 內部結構的整體認識——哪些模組是「核心骨幹」、哪些是「葉節點功能」。
2. **第 02 章**補充外部世界的連接點——OpenClaw 依賴了哪些第三方套件來完成實際工作。
3. **第 03 章**最為關鍵——它解釋了 OpenClaw 如何在「核心不可替換的部分」與「使用者可擴充的部分」之間畫出清晰的界線，這直接影響我們未來開發自定義 plugin 的方式。

## 資料來源

本主題所有內容均來自 OpenClaw 原始碼倉庫（對應版本快照 2026.4.12）的直接分析，主要涉及以下檔案：

| 來源檔案 | 用途 |
|----------|------|
| `source-repo/src/entry.ts` | 應用程式進入點，分析啟動時依賴 |
| `source-repo/src/index.ts` | 模組匯出入口，分析 library vs CLI 模式 |
| `source-repo/src/library.ts` | 公開 API 外觀層，分析對外暴露的功能 |
| `source-repo/src/runtime.ts` | 執行時環境設定 |
| `source-repo/src/cli/run-main.ts` | CLI 主程式，分析 CLI 層依賴 |
| `source-repo/src/extensionAPI.ts` | 已棄用的擴充 API 入口 |
| `source-repo/src/plugin-sdk/index.ts` | 插件 SDK 公開介面 |
| `source-repo/package.json` | 依賴清單 |
| `source-repo/tsconfig.json` | TypeScript 路徑別名設定 |
| `source-repo/tsdown.config.ts` | 打包設定與 neverBundle 清單 |
| `source-repo/scripts/check-extension-plugin-sdk-boundary.mjs` | 邊界檢查腳本 |

---

## 與其他主題的交叉引用

- **[monorepo-structure](../monorepo-structure/)**：依賴圖譜的「容器」——先理解 monorepo 的目錄佈局，才能定位各模組的物理位置。
- **[entry-bootstrap](../entry-bootstrap/)**：啟動流程是依賴圖的「時間軸展開」——entry-bootstrap 記錄啟動順序，本主題記錄靜態依賴關係。
- **[build-and-toolchain](../build-and-toolchain/)**：建置工具鏈決定了哪些依賴會被 bundle、哪些保持外部引用，直接影響依賴圖的最終形態。
