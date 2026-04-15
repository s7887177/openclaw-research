# Agents 元件機制：OpenClaw 多 Agent 系統

> **層級**：Level 2 — Component Mechanics  
> **版本**：2026.4.12  
> **範圍**：`src/agents/` 目錄（約 655+ 個檔案，188K+ 行）

## 本章摘要

Agent 系統是 OpenClaw 中**最龐大的模組**，實現了從單一對話 Agent 到複雜多 Agent 編排的完整功能。每個 Agent 是一個獨立的 LLM 驅動實體，擁有自己的配置、工具集、記憶體、和工作區。Agent 可以**生成子 Agent**（Sub-agent），形成樹狀的任務分解結構。

本章涵蓋：

| 子章節 | 檔案 | 內容 |
|--------|------|------|
| [01-目錄結構與入口](./01-directory-structure.md) | `agent-command.ts`, `agent-scope.ts` | 模組組織、入口函式 |
| [02-Agent 配置與 Schema](./02-config-schema.md) | `config/types.agents.ts`, `agent-limits.ts` | 配置型別、Zod 驗證、限制常數 |
| [03-多 Agent 編排](./03-multi-agent-orchestration.md) | `subagent-spawn.ts`, `subagent-registry.ts` | Sub-agent 生成、Registry、生命週期 |
| [04-工具與技能系統](./04-tools-and-skills.md) | `openclaw-tools.ts`, `skills/` | 工具工廠、技能載入 |
| [05-記憶與模型](./05-memory-and-model.md) | `memory-search.ts`, `model-selection.ts` | 記憶體搜尋、模型覆寫 |

---

## 目錄結構總覽

`src/agents/` 的子目錄組織如下：

| 子目錄 | 檔案數 | 用途 |
|--------|--------|------|
| （根目錄） | ~782 | 核心 Agent 邏輯 |
| `pi-embedded-runner/` | 122 | 嵌入式 Agent 執行器 |
| `tools/` | 110 | 30+ 核心工具實作 |
| `sandbox/` | 71 | 執行沙箱 |
| `auth-profiles/` | 32 | 認證 Profile 管理 |
| `skills/` | 27 | 技能載入與註冊 |
| `test-helpers/` | 21 | 測試工具 |
| `pi-embedded-helpers/` | 16 | 嵌入式輔助 |
| `command/` | 12 | 指令執行管線 |
| `cli-runner/` | 10 | CLI 後端支援 |
| `pi-hooks/` | 9 | 生命週期 Hooks |
| `harness/` | 7 | 測試 Harness |
| `schema/` | 4 | Schema 清理工具 |

---

## 核心檔案概覽

### 入口與指令

| 檔案 | 行數 | 用途 |
|------|------|------|
| `agent-command.ts` | ~1,026 | 主入口：`agentCommand()` |
| `agent-scope.ts` | 371 | Agent 配置解析 Hub |
| `timeout.ts` | 49 | 逾時解析 |

### Sub-agent 系統

| 檔案 | 行數 | 用途 |
|------|------|------|
| `subagent-spawn.ts` | 907 | Sub-agent 生成 |
| `subagent-registry.ts` | 933 | 生命週期追蹤 |
| `subagent-registry.types.ts` | — | 執行記錄型別 |
| `subagent-spawn-plan.ts` | — | 模型與 Thinking 解析 |
| `subagent-registry-completion.ts` | — | 完成回調 |

### 工具與技能

| 檔案 | 用途 |
|------|------|
| `openclaw-tools.ts` | 工具工廠（30+ 工具） |
| `openclaw-tools.registration.ts` | 工具註冊策略 |
| `tool-display-config.ts` | 工具顯示設定（701 行） |
| `skills/plugin-skills.ts` | 技能 Plugin 載入 |
| `skills/workspace.ts` | 工作區技能探索 |
| `skills/skill-contract.ts` | 技能合約驗證 |
| `skills/frontmatter.ts` | YAML frontmatter 解析 |

### 記憶與模型

| 檔案 | 用途 |
|------|------|
| `memory-search.ts` | 記憶體搜尋配置 |
| `model-selection.ts` | 模型選擇（883 行） |

---

## 關鍵常數速查

| 常數 | 值 | 說明 | 來源 |
|------|-----|------|------|
| `DEFAULT_AGENT_MAX_CONCURRENT` | 4 | Agent 最大並行數 | `config/agent-limits.ts:3` |
| `DEFAULT_SUBAGENT_MAX_CONCURRENT` | 8 | Sub-agent 最大並行數 | `config/agent-limits.ts:4` |
| `DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH` | 1 | Sub-agent 最大巢狀深度 | `config/agent-limits.ts:6` |
| `DEFAULT_AGENT_TIMEOUT_SECONDS` | 172,800 (2 天) | Agent 預設逾時 | `agents/timeout.ts:3` |
| `MAX_SAFE_TIMEOUT_MS` | 2,147,000,000 | 計時器安全上限 | `agents/timeout.ts:4` |
| `SUBAGENT_ANNOUNCE_TIMEOUT_MS` | 120,000 (2 分鐘) | Sub-agent 宣告逾時 | `agents/subagent-registry.ts:130` |
| `SESSION_RUN_TTL_MS` | 300,000 (5 分鐘) | Session 執行 TTL | `agents/subagent-registry.ts:138` |
| `DEFAULT_CHUNK_TOKENS` | 400 | 記憶體分塊 Token 數 | `agents/memory-search.ts:97` |

---

## 引用來源總表

| 引用 | 檔案路徑 | 行號 |
|------|----------|------|
| Agent 並行限制 | `source-repo/src/config/agent-limits.ts` | 3-6 |
| Agent 逾時 | `source-repo/src/agents/timeout.ts` | 3-4 |
| agentCommand() | `source-repo/src/agents/agent-command.ts` | 979 |
| Agent 配置解析 | `source-repo/src/agents/agent-scope.ts` | 66-371 |
| Sub-agent 生成 | `source-repo/src/agents/subagent-spawn.ts` | 83-133, 346 |
| Sub-agent Registry | `source-repo/src/agents/subagent-registry.ts` | 23-138 |
| 工具工廠 | `source-repo/src/agents/openclaw-tools.ts` | 51+ |
| 記憶體搜尋 | `source-repo/src/agents/memory-search.ts` | 97-100 |
| 模型選擇 | `source-repo/src/agents/model-selection.ts` | 79 |
| AgentConfig 型別 | `source-repo/src/config/types.agents.ts` | 65 |
| AgentDefaultsConfig | `source-repo/src/config/types.agent-defaults.ts` | 149+ |
