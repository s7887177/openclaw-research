# Skills 完整分類目錄

> **本章摘要**：OpenClaw 內建 53 個技能，涵蓋開發工具、通訊平台、生產力、智慧家居/IoT、音頻/媒體、系統管理、網路/資料、OpenClaw 平台自身等八大類別。本章提供完整目錄，包含每個技能的名稱、emoji、用途摘要和外部依賴。

---

## 1. 分類概覽

| 類別 | 數量 | 涵蓋範圍 |
|------|------|----------|
| 🔧 開發工具 | 8 | 編碼代理、GitHub、CI/CD、技能開發 |
| 💬 通訊平台 | 7 | Discord、Slack、iMessage、WhatsApp、Email |
| 📝 生產力 | 9 | 筆記、任務管理、日曆、Notion、Trello |
| 🏠 智慧家居 / IoT | 6 | 燈光、音箱、睡眠、攝影機、天氣 |
| 🎵 音頻 / 媒體 | 8 | TTS、STT、音樂、影片、GIF |
| 🖥️ 系統 / 自動化 | 6 | tmux、macOS UI、healthcheck、session logs |
| 🌐 網路 / 資料 | 5 | RSS、搜尋、PDF、摘要、X/Twitter |
| ⚙️ OpenClaw 平台 | 4 | ClawHub、Canvas、Node Connect、TaskFlow |

> **來源**：所有 53 個 `source-repo/skills/*/SKILL.md` 的 frontmatter 及內容

---

## 2. 開發工具（8 個）

| 技能 | Emoji | 依賴 | 摘要 |
|------|-------|------|------|
| **coding-agent** | 🧩 | `claude` / `codex` / `opencode` / `pi`（anyBins） | 將編碼任務委派給 Codex、Claude Code 或 Pi 代理，透過背景程序執行。適用場景：建構新功能、審查 PR、重構大型程式碼庫 |
| **github** | 🐙 | `gh` | GitHub 操作：issues、PRs、CI runs、code review、API 查詢 |
| **gh-issues** | — | `curl`, `git`, `gh` | 自動抓取 GitHub issues，生成子代理實作修復並開 PR，監控 review comments。支援 `--watch`、`--cron` 模式。885 行，最大的內建技能 |
| **gemini** | ✨ | `gemini` | Gemini CLI 用於一次性問答、摘要和生成 |
| **oracle** | 🧿 | `oracle` | oracle CLI 的最佳實踐——prompt bundling、引擎、sessions、檔案附件 |
| **skill-creator** | — | — | 建立、編輯、改進或審計 AgentSkills。附帶 Python 腳本 |
| **mcporter** | 📦 | `mcporter` | MCP server/tools 的直接列表、配置、認證和呼叫 |
| **node-connect** | — | — | 診斷 OpenClaw 節點連線和配對失敗（Android/iOS/macOS） |

> **來源**：`source-repo/skills/coding-agent/SKILL.md`、`source-repo/skills/github/SKILL.md`、`source-repo/skills/gh-issues/SKILL.md`、`source-repo/skills/gemini/SKILL.md`、`source-repo/skills/oracle/SKILL.md`、`source-repo/skills/skill-creator/SKILL.md`、`source-repo/skills/mcporter/SKILL.md`、`source-repo/skills/node-connect/SKILL.md`

---

## 3. 通訊平台（7 個）

| 技能 | Emoji | 依賴 | 摘要 |
|------|-------|------|------|
| **discord** | 🎮 | config: `channels.discord.token` | Discord 操作：透過 `message` 工具（channel=discord）。限制只用 message 工具。包含 send/reply/react/embed/components 指南 |
| **slack** | 💬 | config: `channels.slack` | 透過 slack 工具控制 Slack，包含 react/thread/schedule/upload 功能 |
| **bluebubbles** | 🫧 | config: `channels.bluebubbles` | 透過 BlueBubbles 發送/管理 iMessage（推薦的 iMessage 整合方案） |
| **imsg** | 📨 | `imsg`（macOS only） | iMessage/SMS CLI：列出聊天記錄、搜尋、發送訊息 |
| **wacli** | 📱 | `wacli` | WhatsApp 訊息發送、搜尋/同步聊天記錄 |
| **himalaya** | 📧 | `himalaya` | IMAP/SMTP 郵件管理：列出、讀取、撰寫、回覆、轉寄、搜尋。258 行，功能豐富 |
| **voice-call** | 📞 | config: `plugins.entries.voice-call.enabled` | 啟動語音通話，透過 OpenClaw voice-call 插件 |

> **來源**：`source-repo/skills/discord/SKILL.md`、`source-repo/skills/slack/SKILL.md`、`source-repo/skills/bluebubbles/SKILL.md`、`source-repo/skills/imsg/SKILL.md`、`source-repo/skills/wacli/SKILL.md`、`source-repo/skills/himalaya/SKILL.md`、`source-repo/skills/voice-call/SKILL.md`

---

## 4. 生產力（9 個）

| 技能 | Emoji | 依賴 | 摘要 |
|------|-------|------|------|
| **apple-notes** | 📝 | `memo`（macOS only） | macOS Apple Notes 管理：建立、檢視、編輯、刪除、搜尋、匯出 |
| **apple-reminders** | ⏰ | `remindctl`（macOS only） | Apple Reminders 管理：清單、日期過濾、完成/刪除 |
| **bear-notes** | 🐻 | `grizzly`（macOS only） | Bear 筆記的建立、搜尋和管理 |
| **notion** | 📝 | — | Notion API：建立和管理頁面、資料庫、區塊 |
| **obsidian** | 💎 | `obsidian-cli` | Obsidian vault 操作及自動化（純 Markdown 筆記） |
| **things-mac** | ✅ | `things`（macOS only） | Things 3 任務管理：透過 URL scheme 新增/更新專案和待辦事項 |
| **trello** | 📋 | `jq` | Trello REST API：看板、清單、卡片管理 |
| **1password** | 🔐 | `op` | 1Password CLI 設定和使用：安裝、桌面整合、密碼管理 |
| **gog** | 🎮 | `gog` | Google Workspace CLI：Gmail、Calendar、Drive、Contacts、Sheets、Docs |

> **來源**：`source-repo/skills/apple-notes/SKILL.md`、`source-repo/skills/notion/SKILL.md`、`source-repo/skills/obsidian/SKILL.md`、`source-repo/skills/things-mac/SKILL.md`、`source-repo/skills/trello/SKILL.md`、`source-repo/skills/1password/SKILL.md`、`source-repo/skills/gog/SKILL.md`、`source-repo/skills/apple-reminders/SKILL.md`、`source-repo/skills/bear-notes/SKILL.md`

---

## 5. 智慧家居 / IoT（6 個）

| 技能 | Emoji | 依賴 | 摘要 |
|------|-------|------|------|
| **openhue** | 💡 | `openhue` | 控制 Philips Hue 燈光和場景 |
| **sonoscli** | 🔊 | `sonos` | Sonos 音箱控制：發現、狀態、播放、音量、分組 |
| **blucli** | 🫐 | `blu` | BluOS CLI：發現、播放、分組、音量 |
| **eightctl** | 🛌 | `eightctl` | Eight Sleep 智慧床墊控制：狀態、溫度、鬧鐘、排程 |
| **camsnap** | 📸 | `camsnap` | 從 RTSP/ONVIF 攝影機擷取畫面或片段 |
| **weather** | ☔ | `curl` | 透過 wttr.in 或 Open-Meteo 取得天氣和預報 |

> **來源**：`source-repo/skills/openhue/SKILL.md`、`source-repo/skills/sonoscli/SKILL.md`、`source-repo/skills/blucli/SKILL.md`、`source-repo/skills/eightctl/SKILL.md`、`source-repo/skills/camsnap/SKILL.md`、`source-repo/skills/weather/SKILL.md`

---

## 6. 音頻 / 媒體（8 個）

| 技能 | Emoji | 依賴 | 摘要 |
|------|-------|------|------|
| **openai-whisper** | 🎤 | `whisper` | 本地 Whisper CLI 語音轉文字（無需 API key） |
| **openai-whisper-api** | 🌐 | `curl` | OpenAI Audio Transcriptions API（雲端 Whisper） |
| **sag** | 🔊 | `sag` | ElevenLabs TTS，mac-style `say` 語法體驗 |
| **sherpa-onnx-tts** | 🔉 | —（darwin/linux/win32） | 本地 sherpa-onnx TTS（離線，無需雲端） |
| **spotify-player** | 🎵 | `spogo` / `spotify_player`（anyBins） | 終端機 Spotify 播放/搜尋 |
| **video-frames** | 🎬 | `ffmpeg` | 用 ffmpeg 從影片擷取畫面或短片段 |
| **gifgrep** | 🧲 | `gifgrep` | 搜尋 GIF 提供者、下載結果、擷取靜態圖/圖表 |
| **songsee** | 🌊 | `songsee` | 從音頻生成頻譜圖和特徵面板視覺化 |

> **來源**：`source-repo/skills/openai-whisper/SKILL.md`、`source-repo/skills/sag/SKILL.md`、`source-repo/skills/sherpa-onnx-tts/SKILL.md`、`source-repo/skills/spotify-player/SKILL.md`、`source-repo/skills/video-frames/SKILL.md`、`source-repo/skills/gifgrep/SKILL.md`、`source-repo/skills/songsee/SKILL.md`、`source-repo/skills/openai-whisper-api/SKILL.md`

---

## 7. 系統 / 自動化（6 個）

| 技能 | Emoji | 依賴 | 摘要 |
|------|-------|------|------|
| **tmux** | 🧵 | `tmux`（darwin/linux） | 遠端控制 tmux sessions：發送按鍵、擷取窗格輸出 |
| **peekaboo** | 👀 | `peekaboo`（macOS only） | 擷取和自動化 macOS UI |
| **healthcheck** | — | — | 主機安全加固和風險容忍度配置，支援 OpenClaw cron 排程檢查 |
| **session-logs** | 📜 | `jq`, `rg` | 搜尋和分析自身的 session 日誌（較舊/父對話） |
| **model-usage** | 📊 | `codexbar`（macOS only） | CodexBar CLI 本地成本使用摘要（按模型統計） |
| **taskflow** | 🪝 | — | 跨多個獨立任務的工作管理，單一擁有者追蹤 |

> **來源**：`source-repo/skills/tmux/SKILL.md`、`source-repo/skills/peekaboo/SKILL.md`、`source-repo/skills/healthcheck/SKILL.md`、`source-repo/skills/session-logs/SKILL.md`、`source-repo/skills/model-usage/SKILL.md`、`source-repo/skills/taskflow/SKILL.md`

---

## 8. 網路 / 資料（5 個）

| 技能 | Emoji | 依賴 | 摘要 |
|------|-------|------|------|
| **xurl** | 🐦 | `xurl` | X（Twitter）API 認證請求。462 行，非常詳細的 API 使用指南 |
| **blogwatcher** | 📰 | `blogwatcher` | 監控部落格和 RSS/Atom feed 更新 |
| **summarize** | 🧾 | `summarize` | 摘要或擷取 URL/播客/本地檔案的文字/逐字稿 |
| **nano-pdf** | 📄 | `nano-pdf` | 用自然語言指令編輯 PDF |
| **goplaces** | 📍 | `goplaces` | Google Places API（New）查詢：文字搜尋、地點詳情、評論 |

> **來源**：`source-repo/skills/xurl/SKILL.md`、`source-repo/skills/blogwatcher/SKILL.md`、`source-repo/skills/summarize/SKILL.md`、`source-repo/skills/nano-pdf/SKILL.md`、`source-repo/skills/goplaces/SKILL.md`

---

## 9. OpenClaw 平台（4 個）

| 技能 | Emoji | 依賴 | 摘要 |
|------|-------|------|------|
| **clawhub** | — | `clawhub` | ClawHub CLI：搜尋、安裝、更新、發布 agent skills |
| **canvas** | — | — | 在連接的 OpenClaw 節點（Mac/iOS/Android）上顯示 HTML 內容。Canvas Host（port 18793）→ Node Bridge（port 18790）→ Node App |
| **node-connect** | — | — | 診斷節點連線和配對問題（QR 碼、Wi-Fi、VPS、Tailscale） |
| **taskflow-inbox-triage** | 📥 | — | TaskFlow 的收件匣分流範例模式 |

> **來源**：`source-repo/skills/clawhub/SKILL.md`、`source-repo/skills/canvas/SKILL.md`、`source-repo/skills/node-connect/SKILL.md`、`source-repo/skills/taskflow-inbox-triage/SKILL.md`

---

## 10. 統計觀察

### 10.1 依賴模式

| 依賴類型 | 數量 | 說明 |
|----------|------|------|
| 需要特定 CLI 工具（`bins`） | 38 | 大部分技能依賴外部 CLI |
| 需要其一（`anyBins`） | 2 | coding-agent、spotify-player |
| 需要配置項（`config`） | 4 | discord、slack、bluebubbles、voice-call |
| 無外部依賴 | 9 | canvas、healthcheck、node-connect、notion、skill-creator、taskflow、taskflow-inbox-triage、sherpa-onnx-tts、peekaboo（部分靠 OS 功能） |

### 10.2 平台限制

| OS 限制 | 技能 |
|---------|------|
| macOS only (`darwin`) | apple-notes、apple-reminders、bear-notes、imsg、peekaboo、things-mac、model-usage |
| darwin + linux | tmux、sherpa-onnx-tts |
| 全平台 | 其餘 44 個技能 |

### 10.3 技能大小（SKILL.md 行數）

| 範圍 | 數量 | 代表 |
|------|------|------|
| < 50 行 | 5 | nano-pdf(39)、openai-whisper(39)、gemini(44) |
| 50-100 行 | 14 | 1password(71)、blogwatcher(70)、ordercli(79) |
| 100-200 行 | 22 | discord(198)、canvas(200)、github(164) |
| 200-400 行 | 9 | coding-agent(317)、himalaya(258)、healthcheck(246) |
| > 400 行 | 3 | xurl(462)、gh-issues(886)、skill-creator(373) |

> **來源**：所有 53 個 `source-repo/skills/*/SKILL.md` 的統計分析

---

## 引用來源

| 來源路徑 | 引用內容 |
|----------|----------|
| `source-repo/skills/*/SKILL.md`（全部 53 個） | 所有技能的 frontmatter 和內容 |
| 各技能分類引用見各節底部 | 個別技能的 SKILL.md |
