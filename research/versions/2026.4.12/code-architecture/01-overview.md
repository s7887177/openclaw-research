# OpenClaw 程式架構初探（對應 source-repo v2026.4.12）

此檔為「新研究」的第一篇程式碼導讀，採「從事實出發、由下而上」的方式：以 source-repo 作為唯一事實來源（package.json 版本：2026.4.12），逐檢視主要進入點、模組邊界與運行時行為，並提出可驗證的觀察與後續研究題目。所有結論皆可回溯到指定檔案與行數。

---

## 1. 總覽（high level）

- authoritative source: `source-repo/`（在本研究中作為唯一事實來源）
- 主要 entrypoint: `openclaw.mjs`（repo root）
- 核心程式碼庫：`source-repo/src/`
- 技能（Skills）目錄：`source-repo/skills/`
- 打包/發行相關：`dist/`（在開發樹上通常不存在，openclaw.mjs 會檢查並提示使用者執行 build）

註：openclaw.mjs 的 bootstrap（見 source-repo/openclaw.mjs 行 24–180）會檢查 Node 版本、嘗試載入程式輸出（`./dist/entry.js` / `./dist/entry.mjs`），如不存在則報錯並提示如何建置（`pnpm install && pnpm build`）。這表示：

- 開發者工作流程為「源碼 -> build -> dist -> 發行/執行」，而 repo 同時保有可在未構建情況下的防護訊息。
- 真正的執行邏輯通常在 `dist/entry.*`（打包輸出）或對應的 `src/entry.ts`（原始碼）中；因此研究應同時參考 `src/` 與 `dist/`（若存在）以取得正確的執行行為。

## 2. 主要模組與責任邊界（module boundaries）

根據 `source-repo/src/` 目錄：

- cli / entry.ts / entry.*：CLI 與啟動流程，負責解析命令列、載入 runtime 與啟動服務。
- gateway / daemon / runtime.ts / runtime-like files：負責長時間執行的 orchestrator、網路通訊與通道管理。
- skills / plugin-sdk / plugins：技能（Skill）定義、SDK 與插件啟動邊界。
- mcp：與 Model Context Protocol（MCP）相關的整合與 protocol 層。
- memory-host-sdk / memory / shared：記憶存取層，包含長期/短期記憶格式與檔案儲存策略。
- tts / stt / realtime-voice / realtime-transcription：語音子系統（語音輸入、轉錄、產生）
- logging / logger.ts / infra：日誌、監控與運營相關邏輯。

每個目錄代表明確的責任範圍。研究重點應為：
1) 界定每個模組的 public surface（exported functions / classes / CLI flags），
2) 找出跨模組依賴（例如：skills -> mcp -> models），
3) 標註可替換點（plugin activation boundary、skill sandboxing）以利安全與測試。

## 3. Bootstrap 行為細節（摘錄與解讀）

openclaw.mjs（部分重點）：

- Node 版本檢查（lines 8–39）：保證執行環境為 Node.js v22.12+，程式會在不符合時輸出使用者指引並退出。
- Compile cache（lines 41–48）：嘗試啟用 Node 的 module compile cache 以提升冷啟速度。
- 模組解析錯誤判斷（lines 50–69）：openclaw.mjs 區分「直接缺少 entry 檔」與其他轉譯/解析錯誤，僅吞下直接的 entry 解析失敗以嘗試其他備援。
- 載入策略（lines 71–180）：順序為
  1. 嘗試載入 precomputed root help
  2. 嘗試載入 `./dist/entry.js` 或 `./dist/entry.mjs`
  3. 若都失敗，判斷是 unbuilt source tree，輸出建置提示

結論：openclaw.mjs 僅是啟動器/守護進程 bootstrap，核心邏輯集中在 dist 或 src 的 entry 模組。因此，深入研究應把焦點放在 `src/entry.ts`、`src/runtime.ts`、`src/gateway`、`src/skills` 等檔案。

## 4. 推薦的研究切入點（第一輪）

- entry 與 CLI：逐行追蹤 `src/entry.ts` 與 `src/cli/` 中解析選項與 subcommands 的實作，了解啟動參數如何影響 runtime（例如 gateway port、channels enabled、sandbox 設定）。
- 技能（Skills）生命週期：檢視 `plugin-activation-boundary.test.ts` 與 `plugin-sdk/`，找出 activation hook、sandbox API 與技能輸入/輸出的資料格式（SKILL.md 與 skills/ 中的範例）。
- 通道（Channels）與整合：檢視 `src/channels/`、`src/gateway/` 與 `src/channel-web.ts`，理解 adapter 模式如何將不同通訊平台抽象化為 channel。
- 記憶系統：檢視 `memory-host-sdk/`、`memory/`、`shared/` 的 interface，並找到 MEMORY.md 的映射關係，確認 long-term vs short-term 的儲存實際格式與索引策略。
- 安全邊界：檢視 `security/`、`plugin-activation-boundary.test.ts`、以及 sandbox/ container config，確定權限限制點與可注入風險。

## 5. 可驗證的下一步（短期任務）

1. 以 `src/entry.ts` 為藍本，產出「Entry module walkthrough」文件，並在文件中貼出重要程式片段（含檔案與行數引用）。
2. 針對 `skills/` 與 `plugin-sdk/` 撰寫「Skill API 規格對照表」，逐條比對 SKILL.md 與 SDK 介面。注意：所有命題須以 `source-repo/` 原始檔為依據。
3. 生成 call-graph：針對 runtime 起點（entry -> runtime -> gateway -> skills）產生一張 call sequence（文字版或 plantuml），僅在可直接由程式碼推導／解析的情況下建立。

---

若同意，本輪將：
- 建立第一份「Entry module walkthrough」於 `research/versions/2026.4.12/code-architecture/02-entry-walkthrough.md`，逐段複寫並解說 `src/entry.ts`（含程式片段與行數引用）。
- 建立一份 `research/versions/2026.4.12/code-architecture/03-skill-api.md`，抓取 `plugin-sdk/` 與 `skills/` 中的 interface 與說明。

請回覆 "開始解析 entry.ts" 以繼續（或告訴我想先檢視哪個子模組）。
