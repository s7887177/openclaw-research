# Skill 架構與格式

## 本章摘要

本章從零開始介紹 OpenClaw Skill 的架構設計、SKILL.md 格式規範、Frontmatter 中繼資料結構、需求宣告機制與安裝規格。Skill 是 OpenClaw 擴展 AI 助手能力的核心機制——它不是程式碼模組，而是一份結構化的 Markdown 文件，透過 YAML frontmatter 定義中繼資料，並以 Markdown 正文提供 AI 可理解的操作指引。

---

## 1. 什麼是 Skill？

Skill 是一份 `SKILL.md` 檔案，放在以 Skill 名稱命名的目錄中。OpenClaw 在啟動時會掃描 Skill 目錄，將符合條件的 Skill 載入到 AI 助手的 Prompt 中，讓助手知道如何使用這些能力。

Skill 的本質：
- **不是可執行的 Plugin**——Skill 是給 AI 閱讀的指引文件
- **不需要編譯**——純 Markdown + YAML
- **可以附帶腳本**——但腳本由 AI 依據指引自行呼叫
- **有條件載入**——根據 `requires` 判斷是否可用

---

## 2. 目錄結構

### 2.1 標準結構

```
skill-name/
├── SKILL.md             # 必要：Skill 定義檔
├── scripts/             # 可選：可執行腳本
│   ├── do_something.py
│   └── helper.sh
├── references/          # 可選：參考文件
│   └── api-docs.md
└── assets/              # 可選：資源檔案
    └── template.txt
```

### 2.2 最簡結構

最簡單的 Skill 只需要一個檔案：

```
my-skill/
└── SKILL.md
```

### 2.3 參考實作

**coding-agent**（僅有 SKILL.md，11,825 bytes）：
```
skills/coding-agent/
└── SKILL.md
```
> 引用：`source-repo/skills/coding-agent/SKILL.md`

**taskflow**（含範例檔案）：
```
skills/taskflow/
├── SKILL.md
└── examples/
    ├── inbox-triage.lobster
    └── pr-intake.lobster
```
> 引用：`source-repo/skills/taskflow/`

**skill-creator**（含腳本）：
```
skills/skill-creator/
├── SKILL.md    (372 行)
└── scripts/
```
> 引用：`source-repo/skills/skill-creator/`

---

## 3. SKILL.md 格式

### 3.1 結構

SKILL.md 由兩部分組成：

1. **YAML Frontmatter**：以 `---` 包圍的中繼資料
2. **Markdown Body**：AI 閱讀的操作指引

```markdown
---
name: my-skill
description: "My custom skill description"
metadata:
  {
    "openclaw": {
      "emoji": "🔧",
      "requires": { "bins": ["some-tool"] },
      "install": [{ "id": "brew", "kind": "brew", "formula": "some-tool" }]
    }
  }
---

# My Skill

以下是 AI 需要知道的操作指引...
```

### 3.2 必要欄位

只有兩個欄位是必要的：

```yaml
name: "skill-name"        # Skill 名稱（必要）
description: "說明文字"     # Skill 描述（必要）
```

> 引用：`source-repo/skills/skill-creator/SKILL.md:329`

### 3.3 Frontmatter 解析

Frontmatter 由 `parseFrontmatter()` 函式解析：

```typescript
// src/agents/skills/frontmatter.ts (223 行)
// parseFrontmatter(content: string): ParsedSkillFrontmatter  (Line 23)
// resolveOpenClawMetadata(...) — 提取 openclaw metadata 區塊
// resolveSkillInvocationPolicy(...) — 決定呼叫權限
// parseInstallSpec(input: unknown): SkillInstallSpec | undefined  (Lines 108+)
```
> 引用：`source-repo/src/agents/skills/frontmatter.ts:23, 108`

安裝規格的驗證模式：

```typescript
// Line 26: BREW_FORMULA_PATTERN — Brew formula 驗證
// Line 27: GO_MODULE_PATTERN — Go module 驗證
// Line 28: UV_PACKAGE_PATTERN — UV package 驗證
// Line 50-56: npm registry spec 驗證
// Lines 78-86: URL 驗證（download 類型）
```
> 引用：`source-repo/src/agents/skills/frontmatter.ts:26-86`

---

## 4. OpenClaw Metadata

### 4.1 完整結構

```typescript
type OpenClawSkillMetadata = {
  always?: boolean;          // 是否總是載入
  skillKey?: string;         // Skill 鍵值
  primaryEnv?: string;       // 主要環境
  emoji?: string;            // 顯示表情
  homepage?: string;         // 首頁 URL
  os?: string[];             // 支援的作業系統
  requires?: {
    bins?: string[];         // 必要的二進位（全部必須存在）
    anyBins?: string[];      // 至少一個必須存在
    env?: string[];          // 必要的環境變數
    config?: string[];       // 必要的配置路徑
  };
  install?: SkillInstallSpec[];  // 安裝指引
};
```
> 引用：`source-repo/src/agents/skills/types.ts:20-33`

### 4.2 requires 需求宣告

`requires` 讓 Skill 宣告自己需要什麼才能運作：

**範例 1：coding-agent（任一即可）**
```json
"requires": {
  "anyBins": ["claude", "codex", "opencode", "pi"]
}
```
> 引用：`source-repo/skills/coding-agent/SKILL.md:4-26`

**範例 2：github（全部必須）**
```json
"requires": { "bins": ["gh"] }
```
> 引用：`source-repo/skills/github/SKILL.md:9-24`

### 4.3 install 安裝指引

```typescript
type SkillInstallSpec = {
  id?: string;                // 唯一安裝 ID
  kind: "brew" | "node" | "go" | "uv" | "download";  // 安裝方式
  label?: string;             // 使用者可見的標籤
  bins?: string[];            // 安裝後要驗證的二進位
  os?: string[];              // 目標作業系統
  formula?: string;           // Brew formula 名稱
  package?: string;           // npm/uv 套件名
  module?: string;            // Go module 路徑
  url?: string;               // 下載 URL
  archive?: string;           // 壓縮檔名
  extract?: boolean;          // 下載後解壓
  stripComponents?: number;   // Tar strip 前綴
  targetDir?: string;         // 安裝目標目錄
};
```
> 引用：`source-repo/src/agents/skills/types.ts:1-17`

**範例：coding-agent 的安裝指引**
```json
"install": [
  {
    "id": "node-claude",
    "kind": "node",
    "package": "@anthropic-ai/claude-code",
    "bins": ["claude"],
    "label": "Install Claude Code CLI (npm)"
  }
]
```
> 引用：`source-repo/skills/coding-agent/SKILL.md:8-17`

**範例：github 的安裝指引**
```json
"install": [
  {
    "id": "brew",
    "kind": "brew",
    "formula": "gh",
    "bins": ["gh"],
    "label": "Install GitHub CLI (brew)"
  }
]
```
> 引用：`source-repo/skills/github/SKILL.md:9-24`

---

## 5. Skill 來源

### 5.1 四種來源

OpenClaw 會從四個位置掃描 Skill：

| 來源 | 說明 | 優先順序 |
|------|------|---------|
| Bundled | 隨 OpenClaw 安裝的內建 Skill | 最低 |
| Managed | 透過 CLI（`openclaw skills install`）安裝 | 中 |
| Workspace | 工作區 `skills/` 目錄中的 Skill | 中高 |
| Extra | 配置中 `skills.load.extraDirs` 指定的額外目錄 | 最高 |

### 5.2 內建 Skill 列表

OpenClaw v2026.4.12 內建超過 60 個 Skill，涵蓋：

- **AI 工具**：coding-agent, skill-creator, summarize
- **開發工具**：github, gh-issues, taskflow
- **通訊**：discord, slack, imsg, wacli
- **媒體**：camsnap, peekaboo, video-frames, gifgrep
- **生產力**：apple-notes, apple-reminders, bear-notes, notion, obsidian, things-mac, trello
- **系統**：session-logs, model-usage, healthcheck, tmux
- **其他**：weather, spotify-player, openai-whisper

> 引用：`source-repo/skills/` 目錄

---

## 6. Skill 載入流程

### 6.1 載入管線

Skill 的載入由以下模組協作完成：

```
1. 發現 Skill 目錄 (bundled-dir.ts)
2. 讀取 SKILL.md 檔案 (local-loader.ts)
3. 解析 YAML frontmatter (frontmatter.ts)
4. 評估執行資格 (config.ts)
5. 建構 Snapshot Prompt (workspace.ts)
```

### 6.2 載入模組

**local-loader.ts**（161 行）：

```typescript
// loadSingleSkillDirectory(params) — 載入單一 Skill 目錄 (Lines 32-72)
//   讀取 SKILL.md、解析 frontmatter、驗證必要欄位
// listCandidateSkillDirs(dir) — 列出候選目錄 (Lines 74-91)
//   過濾隱藏目錄和 node_modules
// readSkillFileSync(params) — 安全檔案讀取 (Lines 16-31)
//   使用 openVerifiedFileSync、拒絕 symlink、大小限制
```
> 引用：`source-repo/src/agents/skills/local-loader.ts:16-91`

**workspace.ts**（951 行）— 主載入入口：

```typescript
// loadWorkspaceSkillEntries() — 載入所有工作區 Skill
// buildWorkspaceSkillSnapshot() — 建構 Skill Prompt Snapshot
// filterWorkspaceSkillEntries() — 按資格過濾
```
> 引用：`source-repo/src/agents/skills/workspace.ts`

### 6.3 資格評估

```typescript
// src/agents/skills/config.ts (105 行)
// evaluateRuntimeEligibility() — 判斷 Skill 是否符合執行條件
// 檢查: bins, anyBins, env, config, os
```
> 引用：`source-repo/src/agents/skills/config.ts`

### 6.4 安全機制

載入時的安全保護：
- 使用 `openVerifiedFileSync` 防止路徑遍歷
- 拒絕 symlink
- 強制 `maxBytes` 大小限制
- 路徑包含性驗證

> 引用：`source-repo/src/agents/skills/local-loader.ts:16-31`

---

## 7. Skill 配置

### 7.1 配置類型

```typescript
// src/config/types.skills.ts (48 行)

export type SkillConfig = {
  enabled?: boolean;              // 啟用/停用
  apiKey?: SecretInput;           // API 金鑰
  env?: Record<string, string>;   // 環境變數
  config?: Record<string, unknown>; // 自訂配置
};
```
> 引用：`source-repo/src/config/types.skills.ts:3-8`

### 7.2 載入配置

```typescript
export type SkillsLoadConfig = {
  extraDirs?: string[];      // 額外 Skill 資料夾
  watch?: boolean;           // 監視變更
  watchDebounceMs?: number;  // 防抖延遲
};
```
> 引用：`source-repo/src/config/types.skills.ts:10-20`

### 7.3 安裝偏好

```typescript
export type SkillsInstallConfig = {
  preferBrew?: boolean;                           // 偏好 Brew
  nodeManager?: "npm" | "pnpm" | "yarn" | "bun"; // Node 套件管理器
};
```
> 引用：`source-repo/src/config/types.skills.ts:22-25`

### 7.4 限制設定

```typescript
export type SkillsLimitsConfig = {
  maxCandidatesPerRoot?: number;     // 每個根目錄最大候選數
  maxSkillsLoadedPerSource?: number; // 每個來源最大載入數
  maxSkillsInPrompt?: number;        // Prompt 中最大 Skill 數
  maxSkillsPromptChars?: number;     // Prompt 最大字元數
  maxSkillFileBytes?: number;        // SKILL.md 最大大小
};
```
> 引用：`source-repo/src/config/types.skills.ts:27-38`

### 7.5 根配置

```typescript
export type SkillsConfig = {
  allowBundled?: string[];    // 內建 Skill 白名單
  load?: SkillsLoadConfig;
  install?: SkillsInstallConfig;
  limits?: SkillsLimitsConfig;
  entries?: Record<string, SkillConfig>;  // 每個 Skill 的配置
};
```
> 引用：`source-repo/src/config/types.skills.ts:40-47`

---

## 8. 參考 Skill 分析

### 8.1 coding-agent Skill

**用途**：產生（spawn）外部編碼代理來執行複雜任務。

**Frontmatter 重點**：
- 表情：🧩
- 需求：`anyBins: ["claude", "codex", "opencode", "pi"]`
- 多種安裝指引（node 套件）

**內容重點**（316 行）：
- PTY 模式操作指南
- Bash 工具參數
- Process 工具動作
- 快速啟動模式
- One-shot vs 背景任務

> 引用：`source-repo/skills/coding-agent/SKILL.md:1-316`

### 8.2 taskflow Skill

**用途**：多步驟背景工作的持久化流程管理。

**核心概念**：
- Flow 身份（Identity）、Owner Session、`currentStep`、`stateJson`、`waitJson`
- Runtime Shape: `api.runtime.tasks.flow`
- 綁定方法: `fromToolContext(ctx)`, `bindSession({...})`

**生命週期**：
```
createManaged → runTask → setWaiting/finish/fail → resume → cancel
```

> 引用：`source-repo/skills/taskflow/SKILL.md:1-149`

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/agents/skills/types.ts:1-33` | Skill 類型定義 |
| `source-repo/src/agents/skills/frontmatter.ts:23-108` | Frontmatter 解析 |
| `source-repo/src/agents/skills/local-loader.ts:16-91` | 磁碟載入器 |
| `source-repo/src/agents/skills/workspace.ts` | 工作區載入 |
| `source-repo/src/agents/skills/config.ts` | 資格評估 |
| `source-repo/src/config/types.skills.ts:1-48` | Skill 配置類型 |
| `source-repo/skills/coding-agent/SKILL.md:1-316` | Coding Agent Skill |
| `source-repo/skills/github/SKILL.md:9-24` | GitHub Skill |
| `source-repo/skills/taskflow/SKILL.md:1-149` | TaskFlow Skill |
| `source-repo/skills/skill-creator/SKILL.md:1-372` | Skill Creator |
