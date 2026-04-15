# Level 1：倉庫證據層（Repository Evidence）

> **版本快照**：OpenClaw v2026.4.12  
> **層級定位**：記錄倉庫的原始事實（Raw Facts）——目錄結構、建置工具、依賴關係、進入點啟動流程。  
> **4 Topics · 18 份文件 · ~308,000 字元**

---

## 本層概述

倉庫證據層是整個研究的地基。在這一層，我們不做推測、不做概括，只記錄從原始碼中直接觀察到的事實。這些事實包括：

- OpenClaw 的 Monorepo 如何組織
- 程式進入點如何一步步啟動整個框架
- TypeScript 建置工具鏈如何運作
- 內部模組之間的依賴關係圖

所有上層分析（元件機制、橫切機制、系統行為）都以這一層的事實為基礎。

---

## Topic 索引

| # | Topic | 路徑 | 核心問題 | 檔案數 |
|---|-------|------|---------|--------|
| 1 | [Monorepo 結構](./monorepo-structure/) | `monorepo-structure/` | OpenClaw 的 pnpm workspace 如何組織？ | 3 |
| 2 | [進入點與啟動流程](./entry-bootstrap/) | `entry-bootstrap/` | `openclaw` 指令執行後發生了什麼？ | 4 |
| 3 | [建置與工具鏈](./build-and-toolchain/) | `build-and-toolchain/` | TypeScript 如何編譯？CI/CD 如何運作？ | 4 |
| 4 | [依賴圖](./dependency-graph/) | `dependency-graph/` | 模組之間的依賴關係為何？ | 3 |

---

## 閱讀順序建議

1. **Monorepo 結構** → 先理解整體目錄組織
2. **進入點與啟動流程** → 追蹤 `openclaw.mjs` → `entry.ts` → `run()` 的啟動鏈
3. **建置與工具鏈** → 了解 TypeScript 編譯與打包流程
4. **依賴圖** → 看清模組之間的關聯，為 Level 2 元件研究做準備

---

## 與其他層級的關係

- **↑ Level 2（元件機制層）**：Level 1 的模組邊界直接對應 Level 2 的元件拆分
- **↑ Level 4（系統行為層）**：Level 1 的啟動流程是 Level 4 架構總覽的實證基礎
