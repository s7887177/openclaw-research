# Level 4：系統行為層（System Behavior）

> **版本快照**：OpenClaw v2026.4.12  
> **層級定位**：從全局視角描述 OpenClaw 的系統行為——架構全貌、資料流、部署拓撲、擴充模型。  
> **5 Topics · 10 份文件 · ~154,000 字元**

---

## 本層概述

系統行為層是理解 OpenClaw 的最佳起點。不同於 Level 1-2 的「由下而上」微觀分析，本層採用「由上而下」的宏觀視角，描述整個系統如何運作：

- 系統的定位與核心概念是什麼？
- 一個使用者訊息從進入到回應，經過了哪些步驟？
- 系統可以如何部署？
- 擴充機制的整體設計是什麼？
- 操作者與 Session 的信任模型如何運作？

---

## Topic 索引

| # | Topic | 路徑 | 核心問題 | 檔案數 |
|---|-------|------|---------|--------|
| 1 | [架構總覽](./architecture-overview/) | `architecture-overview/` | OpenClaw 是什麼？解決什麼問題？核心設計理念為何？ | 1 |
| 2 | [資料流](./data-flow/) | `data-flow/` | 一個請求從進入到回應的完整旅程 | 1 |
| 3 | [部署拓撲](./deployment-topology/) | `deployment-topology/` | 系統可以如何部署？遠端存取如何運作？ | 1 |
| 4 | [擴充模型](./extension-model/) | `extension-model/` | 三層擴充架構（Plugins / Skills / Extensions） | 1 |
| 5 | [操作者 Session 模型](./operator-session-model/) | `operator-session-model/` | 操作者信任模型與 Session 生命週期 | 1 |

---

## 閱讀順序建議

1. **架構總覽** → 建立 OpenClaw 的全局心智模型
2. **資料流** → 理解端到端的請求處理流程
3. **擴充模型** → 了解三層擴充架構的設計
4. **操作者 Session 模型** → 理解信任與安全邊界
5. **部署拓撲** → 了解實際部署的可能性

---

## 與其他層級的關係

- **↓ Level 1-2**：本層的全局描述以 Level 1 的倉庫事實和 Level 2 的元件細節為實證基礎
- **↔ Level 3（橫切機制層）**：本層描述「系統做什麼」，Level 3 描述「系統如何做到」
- **↑ Level 5（操作者工作流層）**：本層的架構知識是 Level 5 操作實務的理論基礎
