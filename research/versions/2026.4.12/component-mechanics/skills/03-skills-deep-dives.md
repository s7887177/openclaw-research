# Skills 深入分析：重點技能與 ClawHub 生態

> **本章摘要**：本文深入剖析六個具代表性的技能——coding-agent（AI 代理編排）、taskflow（持久工作流）、gh-issues（GitHub 自動化）、discord（通道操作知識）、canvas（跨裝置渲染）、以及 skill-creator（技能開發工具），並探討 ClawHub 市集的運作機制。

---

## 1. coding-agent：AI 代理編排的「超級技能」

**SKILL.md 大小**：317 行  
**依賴**：`anyBins: ["claude", "codex", "opencode", "pi"]`

### 1.1 設計理念

coding-agent 是最複雜的內建技能之一。它不是教 AI 如何寫程式碼，而是教 AI **如何管理其他 AI 來寫程式碼**——本質上是一個「AI 經理」的操作手冊。

核心概念：OpenClaw 代理本身不直接寫程式碼，而是啟動外部編碼代理（Codex、Claude Code、Pi、OpenCode）作為背景程序，然後監控和管理它們。

> *"Use bash (with optional background mode) for all coding agent work."*
>
> — `source-repo/skills/coding-agent/SKILL.md:35`

### 1.2 關鍵知識點

#### PTY 模式差異

技能中最重要的操作知識之一——不同代理需要不同的執行模式：

| 代理 | 執行方式 | 原因 |
|------|----------|------|
| Codex / Pi / OpenCode | `bash pty:true` | 互動式終端應用程式 |
| Claude Code | `--print --permission-mode bypassPermissions`（無 PTY） | `--dangerously-skip-permissions` + PTY 會在確認對話框後退出 |

> **來源**：`source-repo/skills/coding-agent/SKILL.md:37-55`

#### Bash Tool 參數

技能詳細列出了 bash 工具的完整參數，讓 AI 知道如何精確控制程序執行：

- `command`：Shell 指令
- `pty`：分配偽終端（編碼代理必須）
- `workdir`：工作目錄（代理只看到此目錄的上下文）
- `background`：背景執行，返回 sessionId
- `timeout`：超時秒數
- `elevated`：在主機而非沙盒執行

> **來源**：`source-repo/skills/coding-agent/SKILL.md:59-66`

#### 程序管理動作

背景模式下，AI 透過 `process` 工具管理子代理：

- `list`：列出所有 session
- `poll`：檢查 session 狀態
- `log`：取得輸出（支援 offset/limit）
- `write` / `submit`：發送輸入
- `send-keys`：發送按鍵
- `kill`：終止 session

> **來源**：`source-repo/skills/coding-agent/SKILL.md:68-77`

#### 安全規則

技能明確列出了九條規則，包括：
- **永不在 OpenClaw 狀態目錄內啟動 Codex**——它會讀到 `soul.md` 等敏感文件
- **永不在 `~/Projects/openclaw/` 切換分支**——那是正在運行的 OpenClaw 實例
- 代理失敗時「不要偷偷接手」——告知用戶

> **來源**：`source-repo/skills/coding-agent/SKILL.md:230-240`

### 1.3 架構啟示

coding-agent 展示了 OpenClaw 的「代理編排」能力：一個 AI 代理管理多個子代理，每個在獨立的背景程序中工作。這是 OpenClaw 實現複雜自動化的核心模式。

---

## 2. taskflow：持久工作流引擎

**SKILL.md 大小**：149 行  
**依賴**：無

### 2.1 定位

TaskFlow 是 OpenClaw 的「持久工作流基底」。當任務需要跨越一個 prompt 或一次 detached 執行的生命週期時，TaskFlow 提供了統一的狀態管理。

> *"Use TaskFlow when a job needs to outlive one prompt or one detached run, but you still want one owner session, one return context, and one place to inspect or resume the work."*
>
> — `source-repo/skills/taskflow/SKILL.md:14-15`

### 2.2 TaskFlow 的擁有權

TaskFlow 管理以下面向：
- **流程身份**（flow identity）
- **擁有者 session 和請求來源**（owner session / requester origin）
- **當前步驟、狀態、等待條件**（`currentStep`、`stateJson`、`waitJson`）
- **子任務連結**（linked child tasks）
- **狀態機**：finish → fail → cancel → waiting → blocked
- **revision tracking**（衝突安全的變更追蹤）

它**不擁有**分支邏輯——那屬於上層（Lobster、ACPX、或呼叫端程式碼）。

> **來源**：`source-repo/skills/taskflow/SKILL.md:22-36`

### 2.3 Runtime API

```typescript
// 取得 TaskFlow API
const taskFlow = api.runtime.tasks.flow.fromToolContext(ctx);

// 管理式生命週期
const created = taskFlow.createManaged({
  controllerId: "my-plugin/inbox-triage",
  goal: "triage inbox",
  currentStep: "classify",
  stateJson: { businessThreads: [], personalItems: [] },
});

// 連結子任務
taskFlow.runTask({
  flowId: created.flowId,
  runtime: "acp",
  childSessionKey: "agent:main:subagent:classifier",
  runId: "inbox-classify-1",
  task: "Classify inbox messages",
  status: "running",
  startedAt: Date.now(),
  lastEventAt: Date.now(),
});

// 進入等待狀態
taskFlow.setWaiting({
  flowId: created.flowId,
  expectedRevision: created.revision,
  currentStep: "await_business_reply",
  stateJson: { ... },
  waitJson: { kind: "reply", channel: "slack", threadKey: "slack:thread-1" },
});

// 恢復 → 完成
taskFlow.resume({ ... });
taskFlow.finish({ ... });
```

> **來源**：`source-repo/skills/taskflow/SKILL.md:61-117`

### 2.4 設計原則

- 每次變更都檢查 `revision`（樂觀並發控制）
- `stateJson` 是唯一的持久狀態袋——沒有獨立的 output API
- 用 `runTask()` 連結子任務，而非手動建立 detached task
- 將條件邏輯放在呼叫端，TaskFlow 只負責「流轉」

> **來源**：`source-repo/skills/taskflow/SKILL.md:39-52`

---

## 3. gh-issues：GitHub 全自動修復流水線

**SKILL.md 大小**：885 行（最大的內建技能）  
**依賴**：`bins: ["curl", "git", "gh"]`  
**特殊**：`user-invocable: true`

### 3.1 概覽

gh-issues 是一個完整的自動化流水線，設計為由用戶透過 `/gh-issues` 指令直接呼叫。它的 6 個階段：

1. **Phase 1 — 解析參數**：解析 `--label`、`--limit`、`--milestone`、`--fork`、`--watch`、`--cron` 等 flag
2. **Phase 2 — 取得 Issues**：透過 GitHub REST API（用 `curl`，非 `gh` CLI）
3. **Phase 3 — 確認處理**：顯示 issues 供用戶確認
4. **Phase 4 — 子代理實作**：為每個 issue 生成子代理來修復
5. **Phase 5 — 開 PR**：自動推送分支並建立 Pull Request
6. **Phase 6 — 監控 Review**：地址 PR review comments

> **來源**：`source-repo/skills/gh-issues/SKILL.md:16-70`

### 3.2 特殊模式

| Flag | 效果 |
|------|------|
| `--fork user/repo` | Issues 從來源 repo 取得，程式碼推送到 fork，PR 從 fork 開向來源 |
| `--watch` | 完成後持續輪詢新 issues 和 review comments |
| `--cron` | Cron 安全模式：取得 issues、生成子代理後立即退出 |
| `--reviews-only` | 跳過 issue 處理，只監控現有 PR 的 review comments |
| `--notify-channel` | 完成後發送結果到 Telegram 頻道 |

> **來源**：`source-repo/skills/gh-issues/SKILL.md:24-50`

### 3.3 為何 885 行？

gh-issues 是教科書級的「超詳細技能」範例。它之所以需要 885 行，是因為：
- 必須教 AI 完整的 6 階段流程
- 每個 GitHub REST API 呼叫都要指定精確的 curl 格式
- 錯誤處理、重試邏輯、分支命名慣例都需要明確指示
- 支援多種執行模式（fork mode、watch mode、cron mode）的組合

---

## 4. discord：通道操作的「最佳實踐」文件

**SKILL.md 大小**：197 行  
**依賴**：`config: ["channels.discord.token"]`  
**特殊**：`allowed-tools: ["message"]`

### 4.1 技能 vs 擴充套件的完美範例

Discord 技能展示了技能和擴充套件的互補關係：

- **Discord 擴充套件**（`extensions/discord/`，918 行 TypeScript）提供底層 Discord API 整合
- **Discord 技能**（`skills/discord/`，197 行 Markdown）教 AI 如何「正確地」使用 Discord

### 4.2 核心規則

```markdown
## Musts
- Always: `channel: "discord"`.
- Respect gating: `channels.discord.actions.*`
- Prefer explicit ids: `guildId`, `channelId`, `messageId`, `userId`.
```

### 4.3 操作指南範例

技能以 JSON 範例教 AI 每個動作的用法：

```json
// 發送訊息
{ "action": "send", "channel": "discord", "to": "channel:123", "message": "hello" }

// 回覆
{ "action": "reply", "channel": "discord", "to": "channel:123", "replyToMessageId": "456", "message": "..." }

// 使用 Components（偏好）
{ "action": "send", "to": "channel:123", "components": [{ "type": "buttons", "children": [...] }] }
```

> **來源**：`source-repo/skills/discord/SKILL.md:7-40`

### 4.4 `allowed-tools` 的安全意義

`allowed-tools: ["message"]` 意味著當 AI 處於「Discord 模式」時，只能使用 message 工具——不能讀檔案、不能執行 bash。這是一種安全限制，防止 AI 在處理 Discord 互動時意外觸發系統操作。

---

## 5. canvas：跨裝置 HTML 渲染

**SKILL.md 大小**：200 行  
**依賴**：無（frontmatter 也沒有）

### 5.1 架構

Canvas 是 OpenClaw 獨特的能力——讓 AI 在連接的裝置（Mac/iOS/Android）上顯示 HTML 內容：

```
Canvas Host (Port 18793) → Node Bridge (Port 18790) → Node App (WebView)
```

> **來源**：`source-repo/skills/canvas/SKILL.md:17-23`

### 5.2 Tailscale 整合

Canvas Host 的綁定方式取決於 `gateway.bind` 設定：

| 綁定模式 | 伺服器綁定 | Canvas URL |
|----------|-----------|------------|
| `loopback` | 127.0.0.1 | localhost |
| `lan` | LAN 介面 | LAN IP |
| `tailnet` | Tailscale 介面 | Tailscale hostname |
| `auto` | 最佳可用 | Tailscale > LAN > loopback |

> **來源**：`source-repo/skills/canvas/SKILL.md:31-43`

### 5.3 可用動作

| 動作 | 說明 |
|------|------|
| `present` | 顯示 canvas（可指定目標 URL） |
| `hide` | 隱藏 canvas |
| `navigate` | 導航到新 URL |
| `eval` | 在 canvas 中執行 JavaScript |
| `snapshot` | 擷取 canvas 截圖 |

> **來源**：`source-repo/skills/canvas/SKILL.md:49-55`

---

## 6. skill-creator：技能開發的元技能

**SKILL.md 大小**：373 行  
**依賴**：無

### 6.1 獨特之處

skill-creator 是一個「元技能」——它教 AI 如何建立新技能。它包含：
- SKILL.md 格式規範
- 目錄結構最佳實踐
- 打包和發布流程

### 6.2 附帶的 Python 腳本

這是極少數包含程式碼的技能之一：

```
skills/skill-creator/scripts/
├── init_skill.py           # 初始化新技能骨架
├── package_skill.py        # 打包技能（準備發布到 ClawHub）
├── quick_validate.py       # 快速驗證 SKILL.md 格式正確性
├── test_package_skill.py   # 打包功能的測試
└── test_quick_validate.py  # 驗證功能的測試
```

> **來源**：`source-repo/skills/skill-creator/scripts/`

---

## 7. ClawHub 生態系統

### 7.1 ClawHub 的角色

ClawHub（clawhub.com）是 OpenClaw 的技能市集，扮演類似 npm 或 Homebrew 的角色。它解決了技能分享和版本管理的問題。

### 7.2 CLI 工作流

```bash
# 安裝 CLI
npm i -g clawhub

# 登入
clawhub login
clawhub whoami

# 搜尋和安裝
clawhub search <keyword>
clawhub install <skill-name>
clawhub install <skill-name>@<version>

# 更新
clawhub update <skill-name>

# 發布
clawhub publish
```

> **來源**：`source-repo/skills/clawhub/SKILL.md:15-30`

### 7.3 技能生態的成長路徑

```
想法 → 用 skill-creator 建立 SKILL.md → 測試 → clawhub publish
                                                     ↓
其他用戶 → clawhub search → clawhub install → 技能生效
                                                     ↓
                                            feedback → 改進 → clawhub publish（新版本）
```

### 7.4 內建技能 vs ClawHub 技能

| 面向 | 內建技能 | ClawHub 技能 |
|------|----------|-------------|
| 位置 | `source-repo/skills/` | 用戶 `~/.openclaw/skills/` |
| 維護者 | OpenClaw 核心團隊 | 社群 |
| 更新方式 | 隨 OpenClaw 版本更新 | `clawhub update` |
| 品質保證 | 核心團隊 review | 社群評價 |

---

## 8. 技能設計模式總結

從分析 53 個技能中歸納出的設計模式：

### 8.1 「CLI 包裝器」模式（最常見）

38 個技能依賴外部 CLI 工具。典型結構：
1. 說明何時使用
2. CLI 安裝方法
3. 常用指令範例
4. 注意事項

代表：github、1password、obsidian、tmux

### 8.2 「操作指南」模式

教 AI 如何使用 OpenClaw 內部能力。不依賴外部工具。
代表：discord、slack、voice-call、canvas

### 8.3 「編排者」模式

教 AI 如何管理複雜的多步驟工作流。
代表：coding-agent、gh-issues、taskflow

### 8.4 「診斷」模式

教 AI 如何排查問題。
代表：node-connect、healthcheck

---

## 引用來源

| 來源路徑 | 引用內容 |
|----------|----------|
| `source-repo/skills/coding-agent/SKILL.md:35` | bash-first 理念 |
| `source-repo/skills/coding-agent/SKILL.md:37-55` | PTY 模式差異 |
| `source-repo/skills/coding-agent/SKILL.md:59-77` | Bash Tool 參數與 Process 動作 |
| `source-repo/skills/coding-agent/SKILL.md:230-240` | 安全規則 |
| `source-repo/skills/taskflow/SKILL.md:14-15` | TaskFlow 定位 |
| `source-repo/skills/taskflow/SKILL.md:22-36` | TaskFlow 擁有權 |
| `source-repo/skills/taskflow/SKILL.md:39-52` | 設計原則 |
| `source-repo/skills/taskflow/SKILL.md:61-117` | Runtime API 範例 |
| `source-repo/skills/gh-issues/SKILL.md:16-70` | 6 階段流程 |
| `source-repo/skills/gh-issues/SKILL.md:24-50` | 特殊模式 flag |
| `source-repo/skills/discord/SKILL.md:7-40` | Discord 操作指南 |
| `source-repo/skills/canvas/SKILL.md:17-23` | Canvas 架構 |
| `source-repo/skills/canvas/SKILL.md:31-43` | Tailscale 整合 |
| `source-repo/skills/canvas/SKILL.md:49-55` | Canvas 動作 |
| `source-repo/skills/skill-creator/scripts/` | Python 腳本 |
| `source-repo/skills/clawhub/SKILL.md:15-30` | ClawHub CLI |
