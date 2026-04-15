# Entry module walkthrough：src/entry.ts（逐段解析，含檔案與行數引用）

本 walkthrough 直接參考 `source-repo/src/entry.ts`（已摘錄關鍵段落），每個觀察點後面標註原始行數。結論與建議皆可回溯到對應程式碼。

---

## 1. 檔案角色與高階概念

src/entry.ts 的角色是 CLI 與 process bootstrap 的「整合器」（orchestrator bootstrap）。它負責：

- 決定是否為主程序（避免被作為 module import 時產生副作用）——見 isMainModule guard（行 32–44）。
- 註冊或啟用 runtime 與環境（例如 enableCompileCache、installProcessWarningFilter、normalizeEnv）（行 41–58、49–52）。
- 處理特殊快速路徑：版本輸出（--version），說明輸出（--help）等（行 102–123、159–198）。
- 管理 CLI 的 respawn 行為（在需要時以子程序重啟 CLI，並中斷父程序）（行 69–100）。
- 最終將控制權交給 `runCli`（行 200–206）。

因此，entry.ts 不是執行業務邏輯（business logic）的地方，而是把環境、監控（respawn / bridges）與 CLI 抽象化後，將流程移交給 `src/cli/run-main.js` 或 dist 裡的 entry 模組。

## 2. isMainModule guard（避免雙重啟動） — 行 32–44

程式碼要點：

- 使用 `isMainModule({ currentFile: fileURLToPath(import.meta.url), wrapperEntryPairs })` 判斷。
- 目的：當 bundler（或 dist/）將 entry.js 當作 library import 時，entry 的副作用不應該執行，否則會造成重複啟動（競爭 port / lock）。

後設影響：

- 這個 guard 表明 repo 支援「作為 library 被引用」與「作為可執行檔啟動」的雙重模式，實作上要非常保守地將有副作用的程式碼包進 guard 內。

## 3. CLI respawn 與 child-process bridge（行 69–100）

重要行為：

- `buildCliRespawnPlan()`（行 70）回傳 respawn 計劃（若需 respawn），會啟動一個 child process（`spawn(process.execPath, plan.argv, {...})`，行 75–78）。
- 使用 `attachChildProcessBridge(child)`（行 80）建立父子進程間的橋接（例如攔截訊息、傳遞控制或紀錄）
- 若 child process exit / error，父 process 會以該狀態碼退出（行 82–96），避免父程序與子程序同時留在系統上。

研究要點：

- 檢查 `entry.respawn.ts` 的實作（`buildCliRespawnPlan`）以了解什麼條件會觸發 respawn（例如 windows 平台、shell 限制或容器參數）。
- 檢視 `process/child-process-bridge.js`，了解父子間訊息協議及可攜帶的 metadata（利於設計監控/日誌/錯誤上報）。

## 4. 快速路徑：版本輸出（--version）（行 102–123）

實作重點：

- `tryHandleRootVersionFastPath` 會在無 container target 且為 root version invocation 時，import `./version.js` 與 `./infra/git-commit.js`，輸出 `OpenClaw ${VERSION} (${commit})`（行 109–113）。
- 非同步處理，於完成後呼叫 `process.exit(0)`（行 113），或錯誤時設置 `process.exitCode = 1`（行 120–121）。

研究要點：

- `version.js` 與 `infra/git-commit.js` 的實作可證明版本字串與 commit hash 的解析方式，這是做版本對齊（reproducibility）的重要依據。

## 5. 命令列參數預處理（container / profile / windows argv）（行 125–151）

流程摘要：

1. `normalizeWindowsArgv(process.argv)`（行 125）標準化 Windows 下的 argv 差異。
2. 嘗試 respawn；若不需要 respawn，`parseCliContainerArgs` 會檢查 `--container` 類參數（行 128–132）。
3. 解析 profile 參數（`parseCliProfileArgs`，行 134–139），並在有 `--profile` 的情況下呼叫 `applyCliProfileEnv`（行 147–151）以設置環境變數，之後交由 Commander 或 runCli 處理。

研究要點：

- container/profile 參數會改變 process.env 與最終的 argv，應在研究 CLI subcommands 前先掌握如何處理 profile 環境覆寫。

## 6. Help fast-path（行 159–198）

- `tryHandleRootHelpFastPath` 會在 root help invocation 時，嘗試載入 precomputed root help（`cli/root-help-metadata.js`）或動態 import `./cli/program/root-help.js`（行 188–195）。
- 當有 `deps.outputRootHelp` 時，會呼叫該 callback（行 182–186）——這使測試（或外部 API）能夠注入 mock 幫助輸出。

研究要點：

- 該設計顯示 help 輸出被視為可預先計算的資產，與 CLI main path 分離；在做 CLI 文件化或本地化時，可利用 `cli/root-help-metadata.js` 的 precomputed 資訊。

## 7. 導流到 main（runMainOrRootHelp）（行 200–206）

- 若非 help/respawn/version 等特殊 fast-path，最後呼叫 `import("./cli/run-main.js").then(({ runCli }) => runCli(argv))`（行 204–206）。
- `runCli` 為 CLI 的主要入口，會解析 subcommands、啟動 Gateway/skills 等。

研究要點：

- 真正要解析 runtime 行為（例如 gateway 啟動選項）時，要深入 `src/cli/run-main.js` 與該模組依賴的 runtime 檔案（`src/runtime.ts`、`src/gateway/*`）。

## 8. 測試/模擬友善（test hooks）

entry.ts 實作中留有多處為測試注入 hook 的位置，例如 `tryHandleRootHelpFastPath` 接受 `deps` 參數（行 159–166），使測試可注入 `outputRootHelp` 與 `onError`。這代表可編寫單元測試來驗證 fast-path 行為而不需要真正啟動 CLI。

## 9. 小結與後續行動（可立即執行）

- 已落實：建立 `research/versions/2026.4.12/code-architecture/02-entry-walkthrough.md`（本檔）作為 entry 導讀；來源檔：`source-repo/src/entry.ts`（具體行數已標註）。
- 建議下一步（我將自動執行）：
  1. 產生 `02-entry-walkthrough.md` 的可檢索程式碼片段（包含函式簽名與行號），並在同資料夾新增 `03-entry-callgraph.md`（以文字版 call-sequence 開始）。
  2. 逐步匯出 `src/cli/run-main.js`、`src/runtime.ts` 的相同 walkthrough。

若同意，將繼續：自動擷取 `src/cli/run-main.js` 與 `src/runtime.ts` 並建立對應的 walkthrough 文件。