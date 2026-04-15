# AI Agent 基礎理論

> **Topic 定位**：概念綜合層（Level 6）的理論根基  
> **版本快照**：OpenClaw v2026.4.12  
> **字數統計**：~12,000 字

---

## 概述

本 Topic 從第一性原理出發，建立 AI Agent 的完整概念框架。這是整份研究的理論根基——所有後續 Topic 的技術分析，都可以回溯到這裡建立的概念模型。

我們回答一個核心問題：**OpenClaw 是什麼？** 不是從功能列表或技術規格的角度，而是從 AI Agent 的本質出發。

---

## 章節索引

| # | 章節 | 檔案 | 核心內容 |
|---|------|------|---------|
| 1 | [什麼是 AI Agent？](./01-什麼是AI-Agent.md) | `01-什麼是AI-Agent.md` | Agent 定義、Chatbot 對比、核心能力架構、ReAct 範式、工具調用、多 Agent 系統、人格一致性、OpenClaw 實作對照 |

---

## 核心概念框架

```
┌──────────────────────────────────────────────┐
│              AI Agent 概念模型                │
│                                               │
│   感知 ─▶ 推理 ─▶ 行動 ─▶ 觀察              │
│    ▲                           │              │
│    └───────────────────────────┘              │
│                                               │
│   記憶：短期（Context Window）                 │
│         工作（Session Transcript）             │
│         長期（Vector Database）                │
│                                               │
│   人格：System Prompt + Context Files          │
│   協作：Agent Routing + Sub-agents             │
└──────────────────────────────────────────────┘
```

---

## 本 Topic 與 OpenClaw 的對應

| 理論概念 | OpenClaw 實現 | 關鍵程式碼位置 |
|---------|--------------|-------------|
| Agent 執行迴圈 | `runEmbeddedPiAgent()` | `src/agents/pi-embedded-runner/run.ts:166` |
| ReAct Attempt | `attempt.ts` 單次迴圈 | `src/agents/pi-embedded-runner/run/attempt.ts` |
| 工具調用 | Built-in Tools + Plugin SDK + MCP | `src/agents/system-prompt.ts:433-466` |
| 記憶系統 | Context Engine 介面 | `src/context-engine/types.ts:150-281` |
| 長期記憶 | LanceDB 向量資料庫 | `extensions/memory-lancedb/index.ts` |
| 人格設定 | System Prompt + Context Files | `src/agents/system-prompt.ts:39-47` |
| 多 Agent 路由 | `resolveAgentRoute()` | `src/routing/resolve-route.ts` |
| 子 Agent 生成 | `sessions_spawn` 工具 | Session Key 體系 |

---

## 閱讀前提

- 不需要 AI 背景知識（本 Topic 從零開始建立概念）
- 建議已讀過 L4 架構總覽，了解 OpenClaw 的整體結構
- 閱讀本 Topic 後，建議接續 [Local-First 哲學](../local-first-philosophy/) 理解 OpenClaw 的設計理念
