# Skills 架構：目錄結構、SKILL.md 格式與 Skill vs Extension

> **本章摘要**：本文解析 OpenClaw 技能（Skill）系統的架構。技能是 AI 代理的「領域知識包」——它們不執行程式碼，而是透過結構化的 Markdown 文件向 AI 注入專業知識和操作指南。本章涵蓋技能目錄結構、SKILL.md frontmatter 規格、以及技能與擴充套件的根本區別。

---

## 1. 什麼是 Skill？

在 OpenClaw 中，**Skill**（技能）是一個自包含的知識模組，用於讓 AI 代理（Agent）在特定領域表現得像專家。

> 來自 `skill-creator` 技能的定義：  
> *"Skills are modular, self-contained packages that extend Codex's capabilities by providing specialized knowledge, workflows, and tools. Think of them as 'onboarding guides' for specific domains or tasks—they transform Codex from a general-purpose agent into a specialized agent equipped with procedural knowledge that no model can fully possess."*
>
> — `source-repo/skills/skill-creator/SKILL.md:12-16`

技能提供四類價值：
1. **專業工作流程**（Specialized workflows）：特定領域的多步驟流程
2. **工具整合指南**（Tool integrations）：特定檔案格式或 API 的操作說明
3. **領域專業知識**（Domain expertise）：公司特有的知識、schema、商業邏輯
4. **捆綁資源**（Bundled resources）：腳本、參考文件和複雜任務的素材

### 核心洞見：技能 ≠ 程式碼

技能的核心是 **SKILL.md**——一個 Markdown 文件。AI 代理在決定如何處理用戶請求時，會讀取 SKILL.md 中的知識來指導自己的行為。這與擴充套件（Extension）有根本區別：

| 面向 | Skill（技能） | Extension（擴充套件） |
|------|-------------|---------------------|
| 核心形式 | Markdown 文件 | TypeScript 程式碼 |
| 執行方式 | AI 閱讀後自行決策 | 核心載入並執行程式碼 |
| 提供什麼 | 知識、流程、指南 | API 能力、工具、通道 |
| 互動方式 | 被注入 prompt context | 透過 Plugin SDK 註冊 |
| 安裝方式 | 複製 SKILL.md 到技能目錄 | npm 套件 + 核心載入 |
| 市集 | ClawHub | 內建 |

> **來源**：`source-repo/skills/skill-creator/SKILL.md:12-24`

---

## 2. skills/ 目錄結構

### 2.1 頂層結構

```
source-repo/skills/
├── 1password/          # 每個技能一個目錄
├── apple-notes/
├── apple-reminders/
├── ...
├── coding-agent/
├── discord/
├── github/
├── ...
├── weather/
└── xurl/               # 共 53 個目錄
```

> **來源**：`source-repo/skills/` 目錄列表

### 2.2 典型技能目錄結構

絕大多數技能的結構極其簡單——只有一個 `SKILL.md` 檔案：

**最簡結構（大部分技能）**：
```
skills/github/
└── SKILL.md            # 163 行
```

**含範例的結構**：
```
skills/taskflow/
├── SKILL.md            # 149 行
└── examples/           # 範例配置/工作流
```

**含腳本的結構（skill-creator 專用）**：
```
skills/skill-creator/
├── SKILL.md
├── license.txt
└── scripts/
    ├── init_skill.py           # 初始化新技能
    ├── package_skill.py        # 打包技能
    ├── quick_validate.py       # 快速驗證
    ├── test_package_skill.py   # 打包測試
    └── test_quick_validate.py  # 驗證測試
```

> **來源**：`source-repo/skills/github/`、`source-repo/skills/taskflow/`、`source-repo/skills/skill-creator/`

### 2.3 觀察

- 53 個技能中，大部分只有 1 個檔案（SKILL.md）
- 技能的「程式碼」就是 Markdown 中的知識
- 輔助檔案（scripts/、examples/）僅少數技能擁有
- SKILL.md 的大小從約 50 行到 885 行不等（gh-issues 最長）

---

## 3. SKILL.md Frontmatter 完整規格

SKILL.md 的開頭是 YAML frontmatter，定義技能的元資料。

### 3.1 完整欄位定義

```yaml
---
name: github                        # 必要：技能名稱
description: "GitHub operations..."  # 必要：技能描述（AI 用來決定何時啟用）
homepage: https://example.com        # 選填：相關網站
user-invocable: true                 # 選填：是否可由用戶直接呼叫
allowed-tools: ["message"]           # 選填：限制可用工具
metadata:
  openclaw:
    emoji: "🐙"                      # 選填：顯示用 emoji
    os: ["darwin", "linux"]          # 選填：支援的作業系統
    skillKey: "voice-call"           # 選填：技能配置鍵
    primaryEnv: "GH_TOKEN"          # 選填：主要環境變數
    requires:
      bins: ["gh"]                   # 選填：需要的 CLI 工具
      anyBins: ["claude", "codex"]   # 選填：需要其中之一
      config: ["channels.discord.token"]  # 選填：需要的配置項
    install:                         # 選填：安裝指南
      - id: "brew"
        kind: "brew"                 # "brew" | "apt" | "node"
        formula: "gh"               # brew formula 名稱
        package: "@anthropic-ai/claude-code"  # npm 套件名稱
        tap: "steipete/tap"         # brew tap
        bins: ["gh"]                # 安裝後提供的二進位檔
        label: "Install GitHub CLI (brew)"
---
```

> **來源**：從所有 53 個 `source-repo/skills/*/SKILL.md` frontmatter 歸納

### 3.2 各欄位詳解

#### `name`（必要）
技能的識別名稱，通常與目錄名稱相同。

#### `description`（必要）
這是最重要的欄位——AI 代理根據 description 來判斷何時應該使用此技能。好的描述應該包含：
- **何時使用**（use when...）
- **何時不用**（NOT for...）
- **使用方式**（usage pattern）

範例（coding-agent）：
```yaml
description: 'Delegate coding tasks to Codex, Claude Code, or Pi agents via background 
  process. Use when: (1) building/creating new features or apps, (2) reviewing PRs, 
  (3) refactoring large codebases. NOT for: simple one-liner fixes, reading code, 
  thread-bound ACP harness requests.'
```

> **來源**：`source-repo/skills/coding-agent/SKILL.md:3-5`

#### `metadata.openclaw.emoji`
顯示在 UI 中的技能圖示。53 個技能中有 42 個定義了 emoji。

#### `metadata.openclaw.requires`
技能的前置條件：

- **`bins`**：必須存在的 CLI 工具（如 `["gh"]`、`["tmux"]`）
- **`anyBins`**：至少需要其中一個（如 `["claude", "codex", "opencode", "pi"]`）
- **`config`**：需要的配置路徑（如 `["channels.discord.token"]`）

#### `metadata.openclaw.install`
安裝指南陣列，每個元素定義一種安裝方式：

| `kind` | 說明 | 範例 |
|--------|------|------|
| `brew` | Homebrew 安裝 | `{ formula: "gh", bins: ["gh"] }` |
| `apt` | APT 套件管理 | `{ package: "gh", bins: ["gh"] }` |
| `node` | npm 全域安裝 | `{ package: "clawhub", bins: ["clawhub"] }` |

> **來源**：`source-repo/skills/github/SKILL.md:5-20`、`source-repo/skills/coding-agent/SKILL.md:6-23`

### 3.3 特殊欄位

**`user-invocable: true`**（gh-issues）：
標記此技能可由用戶透過斜線指令直接呼叫，而非僅由 AI 自動選用。

```yaml
# source-repo/skills/gh-issues/SKILL.md:4
user-invocable: true
```

**`allowed-tools`**（discord）：
限制使用此技能時可用的工具集：

```yaml
# source-repo/skills/discord/SKILL.md:4
allowed-tools: ["message"]
```

---

## 4. SKILL.md Body 結構

Frontmatter 之後的 Markdown 本體是技能的「知識」部分。AI 代理會完整閱讀這份內容。

### 4.1 常見結構模式

```markdown
# 技能名稱

## When to Use / When NOT to Use
（明確的使用/不使用場景）

## Quick Start / Golden Path
（最常見的使用路徑）

## Common Actions / Commands
（具體的操作指南、CLI 指令範例）

## Notes / Gotchas
（注意事項、常見陷阱）
```

### 4.2 技能的「教學」本質

技能本質上是在「教」AI 如何做事。以 `discord` 技能為例：

```markdown
# Discord (Via `message`)

Use the `message` tool. No provider-specific `discord` tool exposed to the agent.

## Musts

- Always: `channel: "discord"`.
- Respect gating: `channels.discord.actions.*`
- Prefer explicit ids: `guildId`, `channelId`, `messageId`, `userId`.

## Common Actions (Examples)

Send message:
{json}
{
  "action": "send",
  "channel": "discord",
  "to": "channel:123",
  "message": "hello"
}
```

> **來源**：`source-repo/skills/discord/SKILL.md:7-40`

---

## 5. Skill 與 Extension 的根本區別

### 5.1 架構層級差異

```
┌─────────────────────────────────────────┐
│              AI Agent (LLM)             │  ← 閱讀 SKILL.md
│   ┌─────────┐  ┌─────────┐             │
│   │ Skill A │  │ Skill B │  ...        │  ← 純知識（Markdown）
│   └────┬────┘  └────┬────┘             │
│        │ prompt     │ prompt            │
│   ┌────▼────────────▼────┐             │
│   │     Core Runtime     │             │  ← 載入 Extensions
│   │  ┌───────┐ ┌───────┐│             │
│   │  │Ext. A │ │Ext. B ││  ...        │  ← 可執行程式碼（TypeScript）
│   │  └───────┘ └───────┘│             │
│   └──────────────────────┘             │
└─────────────────────────────────────────┘
```

- **技能**在 prompt 層面運作——它們被注入到 AI 的 context window 中
- **擴充套件**在 runtime 層面運作——它們被核心載入並執行

### 5.2 安裝和使用差異

| 面向 | Skill | Extension |
|------|-------|-----------|
| **檔案格式** | SKILL.md（+ 可選的 scripts/、examples/） | TypeScript + package.json + openclaw.plugin.json |
| **安裝位置** | `skills/` 目錄 | `extensions/` 目錄 |
| **大小** | 通常 < 50KB | 通常 100KB - 數 MB |
| **套件系統** | ClawHub | npm |
| **版本管理** | 語意化版本（透過 ClawHub） | 隨 OpenClaw 核心發布 |
| **開發難度** | 低（寫 Markdown） | 高（寫 TypeScript + 理解 SDK） |
| **社群貢獻** | 主要社群貢獻方式 | 核心團隊維護為主 |

### 5.3 互補關係

以 Discord 為例：
- **Discord 擴充套件**（`extensions/discord/`）：提供 Discord API v10 封裝、Bot 連線、訊息收發管線
- **Discord 技能**（`skills/discord/`）：教 AI 如何正確使用 Discord——應該用 `message` 工具、必須指定 `channel: "discord"`、偏好使用 `components` 而非 `embeds`

兩者缺一不可：沒有擴充套件，AI 無法連接 Discord；沒有技能，AI 不知道 Discord 的最佳實踐。

> **來源**：`source-repo/skills/discord/SKILL.md`、`source-repo/extensions/discord/`

---

## 6. ClawHub 市集概念

ClawHub（clawhub.com）是 OpenClaw 的技能市集，類似 npm 之於 Node.js、或 Homebrew 之於 macOS。

### 6.1 ClawHub CLI

```bash
# 安裝 ClawHub CLI
npm i -g clawhub

# 認證
clawhub login
clawhub whoami

# 搜尋技能
clawhub search <keyword>

# 安裝技能
clawhub install <skill-name>

# 更新技能
clawhub update <skill-name>

# 發布技能
clawhub publish
```

> **來源**：`source-repo/skills/clawhub/SKILL.md:15-30`

### 6.2 技能生命週期

```
開發者寫 SKILL.md → 用 skill-creator 驗證 → clawhub publish → 
其他用戶 clawhub install → 技能出現在 skills/ → AI 自動使用
```

`skill-creator` 技能甚至提供了 Python 腳本來輔助這個流程：
- `init_skill.py`：初始化新技能骨架
- `quick_validate.py`：快速驗證 SKILL.md 格式
- `package_skill.py`：打包技能供發布

> **來源**：`source-repo/skills/skill-creator/scripts/`

---

## 引用來源

| 來源路徑 | 引用內容 |
|----------|----------|
| `source-repo/skills/` | 完整目錄列表（53 個技能） |
| `source-repo/skills/skill-creator/SKILL.md:12-24` | 技能定義與價值 |
| `source-repo/skills/coding-agent/SKILL.md:3-23` | Frontmatter 完整範例 |
| `source-repo/skills/github/SKILL.md:5-20` | 安裝指南範例 |
| `source-repo/skills/discord/SKILL.md:4` | allowed-tools 範例 |
| `source-repo/skills/discord/SKILL.md:7-40` | 技能知識內容範例 |
| `source-repo/skills/gh-issues/SKILL.md:4` | user-invocable 範例 |
| `source-repo/skills/clawhub/SKILL.md:15-30` | ClawHub CLI 用法 |
| `source-repo/skills/skill-creator/scripts/` | 技能開發腳本 |
| `source-repo/skills/taskflow/` | 含 examples/ 的結構 |
| `source-repo/extensions/discord/` | Extension 對比參考 |
