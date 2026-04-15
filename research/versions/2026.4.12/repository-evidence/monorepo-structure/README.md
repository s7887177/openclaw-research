# Monorepo Structure — 原始碼證據索引

> **證據層級**：Repository Evidence（原始碼直接觀察）
> **對應版本**：OpenClaw 2026.4.12
> **觀察日期**：2025-07
> **資料來源**：`source-repo/` 目錄下的實際檔案

---

## 本章摘要

本索引涵蓋 OpenClaw 2026.4.12 版本的 **Monorepo 結構**原始碼證據。OpenClaw 是一個以 pnpm workspace 管理的大型 JavaScript/TypeScript monorepo，單一倉庫中包含核心引擎（core）、111 個擴充套件（extensions）、53 個技能包（skills）、4 個原生應用程式（apps）、1 個 Web UI，以及 3 個共享套件（packages）。這些文件記錄了我們從原始碼中直接觀察到的事實——不含推測、不含二手資訊——目的是為後續的概念合成層（Conceptual Synthesis）與系統行為分析層（System Behavior）提供堅實的事實基礎。

理解 monorepo 結構是理解 OpenClaw 一切的起點。它決定了程式碼如何組織、依賴如何流動、建置如何執行、以及擴充套件（extensions）和技能（skills）如何被發現與載入。

---

## 子章節索引

| 編號 | 檔案 | 主題 | 核心問題 |
|------|------|------|----------|
| 01 | [01-pnpm-workspace-config.md](./01-pnpm-workspace-config.md) | pnpm Workspace 配置 | OpenClaw 如何定義 workspace 邊界？哪些目錄是 workspace 成員？依賴安全策略為何？ |
| 02 | [02-top-level-directory-structure.md](./02-top-level-directory-structure.md) | 頂層目錄結構與職責 | 倉庫根目錄有哪些目錄？各自承擔什麼角色？程式碼的規模分佈如何？ |
| 03 | [03-package-json-key-fields.md](./03-package-json-key-fields.md) | package.json 關鍵欄位分析 | 根 package.json 透露了哪些架構決策？exports 映射揭示了什麼 SDK 設計？依賴選擇反映了什麼技術方向？ |

---

## 關鍵數字速覽

以下數字直接取自原始碼觀察，為整個 monorepo 的規模提供量化的第一印象：

| 指標 | 數值 | 來源 |
|------|------|------|
| Workspace 成員區域 | 4 個 glob 模式 + 根目錄自身 | `source-repo/pnpm-workspace.yaml:1-4` |
| 共享套件（packages/） | 3 個 | `source-repo/packages/` 目錄列表 |
| 擴充套件（extensions/） | 111 個 | `source-repo/extensions/` 目錄列表 |
| 技能包（skills/） | 53 個 | `source-repo/skills/` 目錄列表 |
| 原生應用（apps/） | 4 個（Android, iOS, macOS, shared） | `source-repo/apps/` 目錄列表 |
| Web UI workspace | 1 個 | `source-repo/ui/` 目錄 |
| 核心原始碼子目錄（src/） | 60+ 個 | `source-repo/src/` 目錄列表 |
| 生產依賴（dependencies） | 70 個 | `source-repo/package.json` |
| 開發依賴（devDependencies） | 23 個 | `source-repo/package.json` |
| Plugin SDK export 路徑 | 264 個 | `source-repo/package.json` exports 欄位 |
| 允許執行建置腳本的套件 | 11 個 | `source-repo/pnpm-workspace.yaml:31-43` |

---

## 閱讀建議

### 如果你是 OpenClaw 新手

建議按順序閱讀：

1. **先讀 01**（pnpm workspace 配置）——理解 monorepo 的「邊界」在哪裡，哪些東西被視為一個 workspace 單位
2. **再讀 02**（頂層目錄結構）——建立對整個倉庫的空間感，知道「什麼東西在哪裡」
3. **最後讀 03**（package.json 分析）——深入理解架構決策：這個專案如何定義自己、如何對外暴露 API、依賴了什麼

### 如果你已經熟悉 Node.js monorepo

可以直接跳到 [03-package-json-key-fields.md](./03-package-json-key-fields.md)，那裡有最多的架構洞察。

### 如果你想理解 OpenClaw 的可擴充性

重點關注 02 中關於 `extensions/` 和 `skills/` 的段落，以及 03 中關於 `exports` 欄位的分析——這兩者共同揭示了 OpenClaw 的插件生態系統是如何在倉庫層級被組織的。

---

## 與其他證據文件的關聯

本主題（monorepo-structure）提供的事實，會被以下其他主題引用：

- **build-and-toolchain/**：建置工具鏈的配置直接依賴 workspace 結構的定義
- **dependency-graph/**：依賴關係圖的繪製需要先了解 workspace 成員有哪些
- **entry-bootstrap/**：啟動流程的分析需要知道 `bin` 和 `main` 欄位指向哪裡

---

## 方法論說明

本系列文件嚴格遵循以下原則：

1. **只記錄直接觀察到的事實**：所有陳述都可以追溯到具體的檔案路徑和行號
2. **不做推測**：如果原始碼沒有明確表達某件事，我們不會假設它
3. **引用透明**：每個章節末尾都有完整的引用來源表，列出所有被引用的檔案和行號範圍
4. **版本鎖定**：所有觀察鎖定在 2026.4.12 版本，不會混入其他版本的資訊

這種方法確保了後續的分析層可以信任這些事實，並在事實改變時精確地知道需要更新哪些段落。

---

## 引用來源

| 檔案路徑 | 行號範圍 | 用途 |
|----------|----------|------|
| `source-repo/pnpm-workspace.yaml` | 1-45 | workspace 配置全文 |
| `source-repo/.npmrc` | 1-4 | npm 配置 |
| `source-repo/package.json` | 1-1523 | 根 package.json 全文 |
| `source-repo/packages/plugin-sdk/package.json` | — | plugin-sdk 套件定義 |
| `source-repo/packages/memory-host-sdk/package.json` | — | memory-host-sdk 套件定義 |
| `source-repo/packages/plugin-package-contract/package.json` | — | plugin-package-contract 套件定義 |
| `source-repo/extensions/` | — | 擴充套件目錄列表 |
| `source-repo/skills/` | — | 技能包目錄列表 |
| `source-repo/apps/` | — | 原生應用目錄列表 |
| `source-repo/ui/` | — | Web UI 目錄 |
| `source-repo/src/` | — | 核心原始碼目錄列表 |
