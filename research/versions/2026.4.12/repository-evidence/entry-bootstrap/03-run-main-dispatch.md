# 03 — src/cli/run-main.ts：命令派發引擎完整走讀

> **主要原始碼**：`source-repo/src/cli/run-main.ts`（~230 行，`runCli` 函式）
> **輔助原始碼**：`source-repo/src/cli/route.ts:1-65`、`source-repo/src/cli/program/build-program.ts:1-28`
> **角色**：完整 CLI 命令派發的最終站 — 從 `entry.ts` 的 `runMainOrRootHelp()` 進入
> **語言**：TypeScript

---

## 本章摘要

`src/cli/run-main.ts` 包含 `runCli()` 函式，這是整個 OpenClaw CLI 啟動序列的**最終目的地**。當使用者輸入的不是 `--help` 也不是 `--version`，而是一個實際的指令（例如 `openclaw chat`、`openclaw run` 等），程式最終會到達這裡。

此函式是一個約 230 行的長函式，負責：

1. 重複 argv 正規化（防禦性）
2. 容器委派（Container delegation）
3. 環境設定（dotenv、proxy、runtime assertion）
4. 快速路由（Fast routing）— 已知指令直接跳到對應處理函式
5. Commander.js 程式建構 — 完整的指令樹
6. 外掛指令註冊 — 從設定檔中載入使用者自訂指令
7. 最終解析與執行
8. 清理資源

本章將依據 `runCli` 的執行順序，逐步走讀每個階段。

---

## 階段一：argv 正規化與容器/Profile 重覆解析（步驟 1-3）

```typescript
// 步驟 1
normalizeWindowsArgv(argv);

// 步驟 2
parseCliContainerArgs();

// 步驟 3
parseCliProfileArgs();
```

### 為什麼這裡又做一次？

在 `entry.ts` 中已經做過這三個步驟。`run-main.ts` 再做一次的原因是**防禦性設計**（defensive programming）：

| 情境 | 說明 |
|------|------|
| 直接呼叫 `runCli()` | 如果有人從其他模組直接 `import` 並呼叫 `runCli`，不經過 `entry.ts`，這些正規化就是必要的 |
| 測試環境 | 單元測試中可能直接以任意 argv 呼叫 `runCli`，需要在此處也處理 |
| 未來重構 | 如果 `entry.ts` 的流程改變，`run-main.ts` 仍能獨立運作 |

每個步驟都設計為**冪等**（idempotent）的 — 重複執行不會產生副作用。

---

## 階段二：容器委派（步驟 4）

```typescript
// 步驟 4
maybeRunCliInContainer();
```

### 容器委派的概念

OpenClaw 支援在 Docker 容器中執行 CLI 指令。當使用者指定 `--container <name>` 旗標時，`maybeRunCliInContainer()` 會：

1. 解析容器目標名稱
2. 啟動（或連接到）對應的 Docker 容器
3. 在容器內執行原始指令
4. 將容器內的輸出轉發回使用者終端
5. **直接 return** — 後續的所有本地 CLI 邏輯都不會執行

```
openclaw --container my-env chat "hello"
        │
        ▼
  maybeRunCliInContainer()
        │
        ▼
  docker exec my-env openclaw chat "hello"
        │
        ▼
  輸出轉發回使用者
  (函式 return，不繼續執行)
```

---

## 階段三：環境設定（步驟 5-9）

這是 `runCli` 中最密集的環境設定區段，包含五個子步驟：

### 步驟 5：載入 .env 檔案

```typescript
loadCliDotEnv();
```

`loadCliDotEnv` 會搜尋並載入專案根目錄或使用者家目錄中的 `.env` 檔案。載入順序和優先級遵循 dotenv 的標準慣例。

### 步驟 6：正規化環境變數

```typescript
normalizeEnv();
```

再次呼叫 `normalizeEnv()`（`entry.ts` 中也有呼叫），確保在 `.env` 載入後環境變數也被正規化。因為 `loadCliDotEnv()` 可能注入了新的環境變數，這些也需要正規化。

### 步驟 7：初始化除錯代理擷取

```typescript
initializeDebugProxyCapture("cli");
```

此函式在開發/除錯模式下啟用 HTTP 代理擷取（proxy capture），用於記錄所有 HTTP 請求和回應。參數 `"cli"` 標示呼叫來源為 CLI（相對於可能的 SDK 或 daemon 來源）。

### 步驟 8：設定全域 HTTP 代理

```typescript
ensureGlobalUndiciEnvProxyDispatcher();
```

此函式設定 Node.js `undici`（Node 內建 HTTP 客戶端）的全域代理調度器（proxy dispatcher），使所有透過 `fetch()` 發出的 HTTP 請求都經過環境變數中指定的代理伺服器（`HTTP_PROXY`、`HTTPS_PROXY`、`NO_PROXY`）。

### 步驟 9：執行環境斷言

```typescript
assertSupportedRuntime();
```

最終的執行環境檢查。與 `openclaw.mjs` 中的版本門檻不同，此處可能檢查更細緻的條件，例如：

- Node.js 的特定功能旗標是否啟用
- 是否在受支援的作業系統上
- 是否有必要的系統依賴

### 環境設定完整流程圖

```
載入 .env 檔案
       │
       ▼
正規化環境變數
       │
       ▼
初始化除錯代理擷取
       │
       ▼
設定全域 HTTP 代理
       │
       ▼
執行環境斷言
       │
       ▼
繼續命令派發 ...
```

---

## 階段四：Root Help 檢查（步驟 10）

```typescript
// 步驟 10
outputPrecomputedRootHelpText() || outputRootHelp();
```

即使在這個階段，仍然會檢查是否為 `--help` 呼叫。這是因為：

1. `entry.ts` 的 fast-path 可能因為某些原因未攔截（例如 `--help` 與其他旗標組合）
2. `run-main.ts` 可能被直接呼叫，跳過 `entry.ts` 的 fast-path

此處的邏輯是：先嘗試輸出預運算的 help 文字（極快），失敗才呼叫完整的 `outputRootHelp()`（需要建構 Commander.js 程式來產生 help）。

---

## 階段五：快速路由（步驟 11）— `tryRouteCli()`

```typescript
// 步驟 11
tryRouteCli(normalizedArgv);
```

### route.ts 的完整邏輯（`source-repo/src/cli/route.ts:1-65`）

```typescript
// 概念性程式碼
export function tryRouteCli(argv: string[]): boolean {
  if (process.env.OPENCLAW_DISABLE_ROUTE_FIRST) return false;
  
  const route = findRoutedCommand(argv);
  if (!route) return false;
  
  prepareRoutedCommand(route);
  route.run(argv);
  return true;
}
```

### 什麼是快速路由？

快速路由是一種**跳過 Commander.js 建構**的優化機制。

在傳統 CLI 框架中，每次執行任何指令都需要：

1. 建構完整的指令樹（所有子命令、選項、help 文字）
2. 解析 argv 找到匹配的指令
3. 執行該指令

但對於已知的高頻指令，這些步驟是浪費的。快速路由的做法是：

```
傳統路徑:
argv → 建構完整指令樹 → 解析 → 找到指令 → 執行
        （耗時：載入所有子命令模組）

快速路由:
argv → 匹配已知指令路徑 → 直接載入並執行該指令
        （耗時：只載入一個指令模組）
```

### 路由流程

| 步驟 | 函式 | 說明 |
|------|------|------|
| 1 | 環境變數檢查 | `OPENCLAW_DISABLE_ROUTE_FIRST` 可以禁用快速路由（除錯用途） |
| 2 | `findRoutedCommand(argv)` | 從 argv 中提取指令路徑（如 `["chat"]`），在預先註冊的路由表中查找 |
| 3 | `prepareRoutedCommand(route)` | 執行指令特定的前置準備（如載入設定、檢查認證） |
| 4 | `route.run(argv)` | 直接呼叫指令的執行函式 |

### 快速路由 vs Commander.js 的抉擇

```
tryRouteCli 成功？
     │
 ┌───┴───┐
 Yes     No
 │       │
 │       ▼
 │   建構完整 Commander.js 程式
 │       │
 ▼       ▼
指令執行  指令執行
(快速)   (完整)
```

如果 `tryRouteCli` 回傳 `true`，表示指令已經執行完畢，後續的 Commander.js 建構和解析都會被跳過。這對冷啟動效能有顯著影響。

---

## 階段六：Console Capture（步驟 12）

```typescript
// 步驟 12
enableConsoleCapture();
```

`enableConsoleCapture()` 將 `console.log`、`console.error` 等方法的輸出攔截，轉為**結構化日誌**（structured logs）。這意味著：

- 所有 `console.log(...)` 呼叫會同時寫入 stdout **和** OpenClaw 的內部日誌系統
- 日誌可以附帶時間戳、來源模組、嚴重等級等元資料
- 在除錯模式下，這些結構化日誌可以被分析和過濾

此步驟放在快速路由之後、Commander.js 建構之前，是因為快速路由的指令可能不需要 console capture（保持極簡），而完整 CLI 流程需要。

---

## 階段七：Commander.js 程式建構（步驟 13-14）

### buildProgram()（`source-repo/src/cli/program/build-program.ts:1-28`）

```typescript
export function buildProgram() {
  const program = new Command();
  program.enablePositionalOptions();
  program.exitOverride((err) => {
    process.exitCode = typeof err.exitCode === "number" ? err.exitCode : 1;
    throw err;
  });
  const ctx = createProgramContext();
  setProgramContext(program, ctx);
  configureProgramHelp(program, ctx);
  registerPreActionHooks(program, ctx.programVersion);
  registerProgramCommands(program, ctx, argv);
  return program;
}
```

### 逐行解析

| 行 | 程式碼 | 說明 |
|----|--------|------|
| 1 | `new Command()` | 建立 Commander.js 的根命令物件。Commander.js 是 Node.js 生態中最受歡迎的 CLI 框架之一。 |
| 2 | `enablePositionalOptions()` | 啟用位置選項（positional options），讓選項可以不用旗標前綴而以位置來指定。 |
| 3-5 | `exitOverride(...)` | **覆蓋 Commander.js 的預設退出行為**。預設情況下，Commander.js 遇到錯誤會直接呼叫 `process.exit()`。覆蓋後改為設定 `exitCode` 並拋出例外，讓上層可以攔截和處理。 |
| 6 | `createProgramContext()` | 建立程式上下文（program context），包含共用的服務和設定。 |
| 7 | `setProgramContext(program, ctx)` | 將上下文附加到 Commander.js 程式物件上，讓所有子命令都能存取。 |
| 8 | `configureProgramHelp(program, ctx)` | 自訂 help 文字的格式和內容（可能加入品牌化、色彩等）。 |
| 9 | `registerPreActionHooks(program, ctx.programVersion)` | 註冊在任何指令執行前都會觸發的 hook，例如檢查更新、輸出除錯資訊等。 |
| 10 | `registerProgramCommands(program, ctx, argv)` | **註冊所有內建子命令** — 這是最耗時的步驟，會載入所有指令模組。 |

### 為什麼 exitOverride 很重要？

Commander.js 的預設行為是在遇到無法辨識的旗標時直接 `process.exit(1)`。這會中斷 OpenClaw 的錯誤處理和清理流程。透過 `exitOverride`，Commander.js 改為拋出例外，讓 `runCli` 的 try-catch 可以攔截並做適當處理（例如記錄錯誤日誌、清理暫存檔案等）。

### registerProgramCommands — 指令樹建構

`registerProgramCommands` 會註冊所有 OpenClaw 的內建子命令。概念上類似：

```typescript
// 概念性，非實際程式碼
program.command("chat").description("啟動對話").action(chatHandler);
program.command("run").description("執行腳本").action(runHandler);
program.command("config").description("管理設定").action(configHandler);
// ... 所有其他子命令
```

「解析主要指令」（resolve primary command）是指從 argv 中確定使用者要執行的是哪個子命令，以便在需要時只深度載入該子命令的模組。

---

## 階段八：外掛指令註冊（步驟 15）

```typescript
// 步驟 15: Register plugin CLI commands from validated config
```

OpenClaw 支援透過設定檔（config）註冊**外掛指令**（plugin commands）。這些是由使用者或第三方開發者提供的自訂 CLI 子命令。

### 外掛指令的載入流程

```
讀取 OpenClaw 設定檔
        │
        ▼
驗證設定檔格式
        │
        ▼
遍歷設定中的 plugin CLI 定義
        │
        ▼
對每個 plugin：
  1. 驗證 plugin 路徑和簽名
  2. 建立 Commander.js 子命令
  3. 註冊到主程式
        │
        ▼
繼續到最終解析
```

外掛指令註冊放在內建指令之後，確保：

- 內建指令優先（不可被外掛覆蓋）
- 外掛可以存取到完整的 program context
- 設定驗證在外掛載入之前完成

---

## 階段九：最終解析與執行（步驟 16）

```typescript
// 步驟 16
program.parseAsync(parseArgv);
```

`program.parseAsync()` 是 Commander.js 的非同步解析方法。它會：

1. 解析 `parseArgv` 中的所有引數和旗標
2. 匹配到對應的子命令
3. 驗證必要參數和選項
4. 呼叫該子命令的 action handler
5. 等待 action handler 完成（如果是 async function）

`parseArgv` 是經過前面所有正規化步驟處理過的 argv。

### 為什麼用 `parseAsync` 而非 `parse`？

Commander.js 提供兩種解析方法：

| 方法 | 行為 |
|------|------|
| `parse()` | 同步解析。如果 action handler 回傳 Promise，不會等待。 |
| `parseAsync()` | 非同步解析。會 `await` action handler 回傳的 Promise。 |

OpenClaw 的幾乎所有指令都是非同步的（涉及網路請求、檔案 I/O、AI API 呼叫等），所以必須使用 `parseAsync()` 才能正確處理執行結果和錯誤。

---

## 階段十：清理資源（步驟 17）

```typescript
// 步驟 17: Finally
closeCliMemoryManagers();
```

`closeCliMemoryManagers()` 在 CLI 執行完畢後（無論成功或失敗）呼叫，負責：

| 清理項目 | 說明 |
|----------|------|
| 記憶體管理器 | 關閉 OpenClaw 的記憶體儲存系統（可能涉及快取寫入磁碟） |
| 開啟的連線 | 關閉未關閉的 HTTP 連線、WebSocket 連線等 |
| 暫存檔案 | 清理可能建立的暫存檔案 |
| 日誌緩衝 | 將尚未寫入的日誌刷出（flush） |

「Finally」語義暗示此步驟包在 try-finally 區塊中，確保即使指令執行中拋出例外，清理工作仍會執行。

---

## runCli 完整步驟總覽

| 步驟 | 函式 | 類別 | 可能的提前退出 |
|------|------|------|---------------|
| 1 | `normalizeWindowsArgv(argv)` | argv 前處理 | — |
| 2 | `parseCliContainerArgs()` | argv 前處理 | ❌ 解析失敗 |
| 3 | `parseCliProfileArgs()` | argv 前處理 | ❌ 解析失敗 |
| 4 | `maybeRunCliInContainer()` | 容器委派 | ✅ 在容器中執行 |
| 5 | `loadCliDotEnv()` | 環境設定 | — |
| 6 | `normalizeEnv()` | 環境設定 | — |
| 7 | `initializeDebugProxyCapture("cli")` | 環境設定 | — |
| 8 | `ensureGlobalUndiciEnvProxyDispatcher()` | 環境設定 | — |
| 9 | `assertSupportedRuntime()` | 環境設定 | ❌ 不支援的執行環境 |
| 10 | `outputPrecomputedRootHelpText()` / `outputRootHelp()` | Help 處理 | ✅ 輸出 help |
| 11 | `tryRouteCli(normalizedArgv)` | 快速路由 | ✅ 已知指令直接執行 |
| 12 | `enableConsoleCapture()` | 日誌設定 | — |
| 13 | `buildProgram()` | 程式建構 | — |
| 14 | `registerProgramCommands()` + resolve | 程式建構 | — |
| 15 | 外掛指令註冊 | 程式建構 | — |
| 16 | `program.parseAsync(parseArgv)` | 執行 | — |
| 17 | `closeCliMemoryManagers()` | 清理 | — |

---

## 效能優化層級圖

整個啟動序列設計了**多層快速路徑**，從最快到最慢：

```
Layer 0 (最快): openclaw.mjs → 預運算 help JSON        ~5ms
Layer 1:        entry.ts     → --version fast-path     ~20ms
Layer 2:        entry.ts     → root help fast-path     ~30ms
Layer 3:        run-main.ts  → tryRouteCli 快速路由     ~50ms
Layer 4 (最慢): run-main.ts  → Commander.js 完整解析    ~200ms+
```

每一層都在嘗試：「能不能在不載入更多模組的情況下完成任務？」只有當所有快速路徑都不適用時，才會走到最慢的 Commander.js 完整流程。

---

## 三個核心輔助系統的角色

| 系統 | 檔案 | 在 runCli 中的角色 |
|------|------|-------------------|
| **路由系統** | `src/cli/route.ts` | 提供指令 fast-path，跳過 Commander.js |
| **程式建構** | `src/cli/program/build-program.ts` | 建構完整的 Commander.js 指令樹 |
| **記憶體管理** | （多個檔案） | 提供 CLI 執行期間的狀態管理和清理 |

---

## 引用來源

| 檔案路徑 | 行數範圍 | 本文引用段落 |
|----------|---------|-------------|
| `source-repo/src/cli/run-main.ts` | 全檔 (~230 行) | runCli 完整流程（步驟 1-17） |
| `source-repo/src/cli/route.ts` | 1-65 | 快速路由機制 |
| `source-repo/src/cli/program/build-program.ts` | 1-28 | Commander.js 程式建構 |
| `source-repo/src/entry.ts` | 129-138 | runMainOrRootHelp 進入點 |
