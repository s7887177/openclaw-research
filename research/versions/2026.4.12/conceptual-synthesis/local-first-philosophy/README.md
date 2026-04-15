# Local-First 設計哲學

> **Topic 定位**：概念綜合層（Level 6）的設計理念分析  
> **版本快照**：OpenClaw v2026.4.12  
> **字數統計**：~12,000 字

---

## 概述

本 Topic 深入探討 OpenClaw 的 Local-First 設計哲學。這不只是一個技術選擇，而是一個關於信任、自主權和隱私的根本立場——它回答了一個核心問題：**在 AI Agent 時代，你的數位自我應該由誰掌控？**

OpenClaw 的回答是：**由你自己。**

---

## 章節索引

| # | 章節 | 檔案 | 核心內容 |
|---|------|------|---------|
| 1 | [Local-First 設計哲學](./01-Local-First設計哲學.md) | `01-Local-First設計哲學.md` | Local-First 定義與歷史、哲學宣言、信任模型、技術實現、務實折衷、雲端比較、演化歷程 |

---

## 核心論點

```
┌──────────────────────────────────────────────────────┐
│            OpenClaw Local-First 哲學                  │
│                                                       │
│  "It runs on your devices, in your channels,          │
│   with your rules."                                   │
│                                    — VISION.md        │
│                                                       │
│  ┌─────────────┐   ┌─────────────┐   ┌────────────┐ │
│  │ 資料主權     │   │ 運算主權     │   │ 選擇自由   │ │
│  │ ~/.openclaw/ │   │ 本地 Gateway │   │ 50+ LLMs  │ │
│  │ 本地記憶     │   │ 127.0.0.1   │   │ + Ollama   │ │
│  └─────────────┘   └─────────────┘   └────────────┘ │
│                                                       │
│  信任模型：個人助理（一操作者 = 完全信任）              │
│  安全策略：強預設 + 彈性配置                           │
│  技術哲學：TypeScript = 可駭入性                       │
└──────────────────────────────────────────────────────┘
```

---

## 關鍵證據來源

| 面向 | 主要來源 | 檔案位置 |
|------|---------|---------|
| 設計宣言 | VISION.md | `source-repo/VISION.md` |
| 信任模型 | SECURITY.md | `source-repo/SECURITY.md` |
| 本地綁定 | `resolveGatewayBindHost()` | `src/gateway/net.ts:298-310` |
| 本地配置 | `resolveStateDir()` | `src/config/paths.ts:60-89` |
| 本地記憶 | LanceDB 整合 | `extensions/memory-lancedb/index.ts` |
| 本地模型 | Ollama / LM Studio | `extensions/ollama/src/defaults.ts` |
| Daemon 模式 | launchd/systemd/schtasks | `src/daemon/` |

---

## 閱讀前提

- 建議先讀 [AI Agent 基礎理論](../ai-agent-foundations/) 了解 Agent 的概念模型
- 對 L4 部署拓撲有基本了解會有幫助
- 本 Topic 與 L2 的 Config、Gateway、Memory 元件機制有交叉引用
