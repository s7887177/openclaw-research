# ReAct 推理迴圈（ReAct Reasoning Loop）

本目錄研究 OpenClaw Agent 的核心推理迴圈：思考（Thought）→ 行動（Action）→ 觀察（Observation）
的 ReAct 模式實作。

## 文件索引

| 檔案 | 內容 | 字數估計 |
|------|------|---------|
| [01-react-loop-and-tool-calling.md](./01-react-loop-and-tool-calling.md) | ReAct 迴圈結構、工具調用與核准流程 | ~9,000 |
| [02-hooks-streaming-and-state.md](./02-hooks-streaming-and-state.md) | Hook 系統、串流回應、狀態管理與子 Agent | ~8,000 |

## 核心問題

當使用者對 OpenClaw Agent 說話，Agent 不只是產生一段文字回應。
它會進入一個迴圈：思考要做什麼 → 調用工具執行 → 觀察結果 → 繼續思考，
直到完成任務。這就是 ReAct（Reasoning + Acting）模式。

理解這個迴圈是理解 OpenClaw 所有「智慧」行為的關鍵。
