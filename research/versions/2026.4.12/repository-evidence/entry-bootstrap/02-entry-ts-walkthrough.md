# 02 — src/entry.ts：TypeScript 進入點完整走讀

> **原始碼**：`source-repo/src/entry.ts:1-213`（完整檔案）
> **輔助檔案**：`source-repo/src/entry.respawn.ts:1-60`、`source-repo/src/infra/is-main.ts:1-62`
> **角色**：CLI 的真正進入點 — 被 `openclaw.mjs` 動態匯入後執行
> **語言**：TypeScript

---

## 本章摘要

`src/entry.ts` 是 OpenClaw CLI 的**核心進入點**。在上一章中，`openclaw.mjs` 透過 `import("./dist/entry.js")` 動態匯入此檔案的編譯產物。一旦匯入成功，此檔案的頂層程式碼就會立即執行。

此檔案的職責可以分為三大區塊：

1. **主模組守衛**（Main Module Guard）— 判斷自己是被當作 CLI 進入點執行，還是被當作函式庫匯入。只有前者才執行 CLI 邏輯。
2. **環境準備與 Respawn** — 設定 `process.title`、正規化環境變數、啟用 compile cache、判斷是否需要 respawn（重啟）Node 程序以注入 `NODE_OPTIONS`。
3. **命令前置處理** — 解析 `--container`、`--profile`/`--dev` 旗標，處理 `--version` 快速路徑，最終進入 `runMainOrRootHelp()`。

本章會逐段走讀整個檔案，並深入分析 `entry.respawn.ts` 和 `infra/is-main.ts` 兩個關鍵輔助模組。

---

## 第一段：匯入宣告（第 1-16 行）

```typescript
#!/usr/bin/env node
import { spawn } from "node:child_process";
import { enableCompileCache } from "node:module";
import process from "node:process";
import { fileURLToPath } from "node:url";
import { isRootHelpInvocation, isRootVersionInvocation } from "./cli/argv.js";
import { parseCliContainerArgs, resolveCliContainerTarget } from "./cli/container-target.js";
import { applyCliProfileEnv, parseCliProfileArgs } from "./cli/profile.js";
import { normalizeWindowsArgv } from "./cli/windows-argv.js";
import { buildCliRespawnPlan } from "./entry.respawn.js";
import { isTruthyEnvValue, normalizeEnv } from "./infra/env.js";
import { isMainModule } from "./infra/is-main.js";
import { ensureOpenClawExecMarkerOnProcess } from "./infra/openclaw-exec-env.js";
import { installProcessWarningFilter } from "./infra/warning-filter.js";
import { attachChildProcessBridge } from "./process/child-process-bridge.js";
```

### 匯入分類

| 類別 | 模組 | 用途 |
|------|------|------|
| **Node 內建** | `child_process.spawn` | Respawn 時啟動子程序 |
| **Node 內建** | `module.enableCompileCache` | 啟用 V8 編譯快取 |
| **Node 內建** | `process` | 環境變數、argv、程序標題 |
| **Node 內建** | `url.fileURLToPath` | 將 `import.meta.url` 轉為檔案路徑 |
| **CLI 解析** | `cli/argv.js` | 判斷是否為 `--help` 或 `--version` 呼叫 |
| **CLI 解析** | `cli/container-target.js` | 解析 `--container` 旗標 |
| **CLI 解析** | `cli/profile.js` | 解析 `--profile`/`--dev` 旗標 |
| **CLI 解析** | `cli/windows-argv.js` | Windows 下的 argv 正規化 |
| **Respawn** | `entry.respawn.js` | 判斷是否需要重啟 Node 程序 |
| **基礎設施** | `infra/env.js` | 環境變數正規化與布林值判定 |
| **基礎設施** | `infra/is-main.js` | 主模組判定 |
| **基礎設施** | `infra/openclaw-exec-env.js` | 在 process 上設定執行標記 |
| **基礎設施** | `infra/warning-filter.js` | Node 警告過濾 |
| **程序管理** | `process/child-process-bridge.js` | 子程序信號橋接 |

注意：所有匯入使用 `.js` 副檔名（而非 `.ts`），這是 TypeScript ESM 專案的慣例 — 匯入路徑指向**編譯後**的檔案。

---

## 第二段：包裝層對應表（第 18-21 行）

```typescript
const ENTRY_WRAPPER_PAIRS = [
  { wrapperBasename: "openclaw.mjs", entryBasename: "entry.js" },
  { wrapperBasename: "openclaw.js", entryBasename: "entry.js" },
] as const;
```

此常數定義了「哪些外層包裝檔案會匯入哪些進入點檔案」的對應關係。用途是在 `isMainModule` 中判斷：即使 `process.argv[1]` 指向 `openclaw.mjs`（包裝層），而目前執行的檔案是 `entry.js`（進入點），也要認定為「主模組」。

### 為什麼需要這個對應表？

在 Node.js ESM 中，`process.argv[1]` 記錄的是使用者直接執行的檔案路徑（即 `openclaw.mjs`），但 `entry.ts` 是被 `import()` 動態載入的。如果單純比較 `argv[1]` 和 `import.meta.url`，它們不會相等。`ENTRY_WRAPPER_PAIRS` 讓 `isMainModule` 知道：「如果 `argv[1]` 是 `openclaw.mjs`，而我是 `entry.js`，那我就是主模組」。

---

## 第三段：shouldForceReadOnlyAuthStore（第 23-31 行）

```typescript
function shouldForceReadOnlyAuthStore(argv: string[]): boolean {
  const tokens = argv.slice(2).filter((token) => token.length > 0 && !token.startsWith("-"));
  for (let index = 0; index < tokens.length - 1; index += 1) {
    if (tokens[index] === "secrets" && tokens[index + 1] === "audit") {
      return true;
    }
  }
  return false;
}
```

此函式檢查使用者是否正在執行 `openclaw secrets audit` 指令。如果是，將 auth store 強制設為唯讀模式。

**邏輯步驟：**

1. `argv.slice(2)` — 去掉 `node` 和腳本路徑，只留使用者輸入的參數
2. `.filter(...)` — 過濾掉空字串和以 `-` 開頭的旗標，只留「位置參數」（positional arguments）
3. 遍歷相鄰的兩個 token，找到 `"secrets"` 後緊接 `"audit"` 的模式

**為什麼 `secrets audit` 需要唯讀？**

`secrets audit` 是稽核指令，只應讀取認證資料而不應修改。強制唯讀是一種**最小權限原則**（principle of least privilege）的實踐。

---

## 第四段：主模組守衛（第 33-42 行）

```typescript
if (
  !isMainModule({
    currentFile: fileURLToPath(import.meta.url),
    wrapperEntryPairs: [...ENTRY_WRAPPER_PAIRS],
  })
) {
  // Imported as a dependency — skip all entry-point side effects.
} else {
  // ... 整個 CLI 進入點邏輯 ...
}
```

這是整個檔案最關鍵的**結構性守衛**。

### `isMainModule` 的判定邏輯（`source-repo/src/infra/is-main.ts:1-62`）

```typescript
export function isMainModule({
  currentFile, argv = process.argv, env = process.env, cwd = process.cwd(),
  wrapperEntryPairs = [],
}: IsMainModuleOptions): boolean {
  // 1. 比較 currentFile 與 argv[1]（正規化路徑後）
  // 2. 檢查 PM2 的 pm_exec_path 環境變數
  // 3. 遍歷 wrapperEntryPairs，看 argv[1] 的 basename 是否匹配某個 wrapper
}
```

判定流程：

| 步驟 | 條件 | 結果 |
|------|------|------|
| 1 | `currentFile === normalize(argv[1])` | ✅ 是主模組 |
| 2 | `env.pm_exec_path` 匹配 `currentFile` | ✅ 是主模組（PM2 部署） |
| 3 | `argv[1]` 的 basename 匹配某個 `wrapperBasename`，且 `currentFile` 的 basename 匹配對應的 `entryBasename` | ✅ 是主模組（透過包裝層啟動） |
| 4 | 以上都不匹配 | ❌ 不是主模組 |

**為什麼需要這個守衛？**

`entry.ts` 既是 CLI 進入點，也可能被其他程式碼 `import` 作為函式庫使用。守衛確保 CLI 的副作用（設定 `process.title`、修改環境變數等）只在作為進入點時才發生。

---

## 第五段：環境初始化（第 43-60 行）

```typescript
const { installGaxiosFetchCompat } = await import("./infra/gaxios-fetch-compat.js");
await installGaxiosFetchCompat();
process.title = "openclaw";
ensureOpenClawExecMarkerOnProcess();
installProcessWarningFilter();
normalizeEnv();
if (!isTruthyEnvValue(process.env.NODE_DISABLE_COMPILE_CACHE)) {
  try { enableCompileCache(); } catch { }
}
if (shouldForceReadOnlyAuthStore(process.argv)) {
  process.env.OPENCLAW_AUTH_STORE_READONLY = "1";
}
if (process.argv.includes("--no-color")) {
  process.env.NO_COLOR = "1";
  process.env.FORCE_COLOR = "0";
}
```

### 逐行解析

| 行為 | 說明 |
|------|------|
| `installGaxiosFetchCompat()` | 安裝 Gaxios（Google API 客戶端）與原生 `fetch` 的相容層。使用動態 `import()` 延遲載入，因為不是所有指令都需要此功能。 |
| `process.title = "openclaw"` | 設定程序標題，讓 `ps` 或 `top` 中顯示 `openclaw` 而非 `node`。 |
| `ensureOpenClawExecMarkerOnProcess()` | 在 `process` 物件上設定一個標記，讓其他模組可以知道自己是在 OpenClaw CLI 上下文中執行。 |
| `installProcessWarningFilter()` | 過濾 Node.js 實驗性功能的警告訊息（與 `openclaw.mjs` 中的功能相同，此處是 TypeScript 版本）。 |
| `normalizeEnv()` | 正規化環境變數（例如統一大小寫、補齊預設值等）。 |
| `enableCompileCache()` | 再次嘗試啟用 compile cache。雖然 `openclaw.mjs` 已經呼叫過，但如果 `entry.ts` 是直接執行的（例如在開發中 `node dist/entry.js`），這裡再呼叫一次作為保險。 |
| `shouldForceReadOnlyAuthStore` | 如前述，`secrets audit` 指令強制唯讀。 |
| `--no-color` 處理 | 偵測到 `--no-color` 時，同時設定 `NO_COLOR` 和 `FORCE_COLOR=0`，確保所有下游函式庫都遵守無色輸出。 |

---

## 第六段：Respawn 機制（第 62-82 行）

```typescript
function ensureCliRespawnReady(): boolean {
  const plan = buildCliRespawnPlan();
  if (!plan) return false;
  const child = spawn(process.execPath, plan.argv, {
    stdio: "inherit",
    env: plan.env,
  });
  attachChildProcessBridge(child);
  child.once("exit", (code, signal) => {
    if (signal) { process.exitCode = 1; return; }
    process.exit(code ?? 1);
  });
  child.once("error", (error) => {
    console.error("[openclaw] Failed to respawn CLI:", error instanceof Error ? (error.stack ?? error.message) : error);
    process.exit(1);
  });
  return true;
}
```

### 什麼是 Respawn？

Respawn 是指**重新啟動 Node.js 程序**。某些功能需要在 Node 啟動時就透過 `NODE_OPTIONS` 設定，但使用者執行 `openclaw` 時可能沒有設定這些選項。Respawn 機制的做法是：

1. 偵測目前的 `NODE_OPTIONS` 是否缺少必要設定
2. 如果缺少，計算出完整的 `NODE_OPTIONS` 和 argv
3. 用 `spawn()` 啟動一個新的 Node 程序，帶上正確的設定
4. 原始程序變成「代理」，只負責轉發退出碼和信號

### `buildCliRespawnPlan()`（`source-repo/src/entry.respawn.ts:1-60`）

```
buildCliRespawnPlan()
        │
        ▼
  shouldSkipRespawnForArgv()?  ← 某些指令不需要 respawn
        │
    ┌───┴───┐
    Yes     No
    │       │
  null      ▼
        hasExperimentalWarningSuppressed()?
        需要注入 --disable-warning=ExperimentalWarning 嗎？
            │
            ▼
        resolveNodeStartupTlsEnvironment()
        需要注入 NODE_EXTRA_CA_CERTS 嗎？
            │
            ▼
        任一需要注入？
        ┌───┴───┐
        No      Yes
        │       │
      null    { argv, env }
              包含修改後的 NODE_OPTIONS 和環境變數
```

**關鍵常數：**

| 常數 | 值 | 用途 |
|------|---|------|
| `EXPERIMENTAL_WARNING_FLAG` | `"--disable-warning=ExperimentalWarning"` | 抑制 Node.js 實驗性 API 警告 |
| `OPENCLAW_NODE_OPTIONS_READY` | （環境變數名） | 標記 NODE_OPTIONS 已注入，避免無限 respawn |
| `OPENCLAW_NODE_EXTRA_CA_CERTS_READY` | （環境變數名） | 標記 CA 憑證已設定 |

### 子程序橋接

```typescript
attachChildProcessBridge(child);
```

`attachChildProcessBridge` 確保父程序收到的信號（如 `SIGINT`、`SIGTERM`）會被轉發給子程序。同時，子程序的退出碼會被正確傳遞回父程序。這使得整個 respawn 過程對使用者完全透明。

### 防止無限 Respawn

`buildCliRespawnPlan` 中會設定 `OPENCLAW_NODE_OPTIONS_READY` 環境變數。當 respawn 後的子程序再次執行到這段程式碼時，看到此環境變數就知道不需要再次 respawn。

---

## 第七段：版本快速路徑（第 84-99 行）

```typescript
function tryHandleRootVersionFastPath(argv: string[]): boolean {
  if (resolveCliContainerTarget(argv)) return false;
  if (!isRootVersionInvocation(argv)) return false;
  Promise.all([import("./version.js"), import("./infra/git-commit.js")])
    .then(([{ VERSION }, { resolveCommitHash }]) => {
      const commit = resolveCommitHash({ moduleUrl: import.meta.url });
      console.log(commit ? `OpenClaw ${VERSION} (${commit})` : `OpenClaw ${VERSION}`);
      process.exit(0);
    })
    .catch((error) => {
      console.error("[openclaw] Failed to resolve version:", ...);
      process.exitCode = 1;
    });
  return true;
}
```

### 邏輯流程

1. **跳過容器模式** — 如果有 `--container` 旗標，版本資訊應由容器內的 OpenClaw 報告，不是外層的
2. **檢查是否為版本查詢** — `isRootVersionInvocation` 檢查 `argv` 是否為 `openclaw --version` 或 `openclaw -V`
3. **平行載入版本資訊** — `Promise.all` 同時載入 `version.js`（版本號）和 `git-commit.js`（git commit hash）
4. **輸出格式** — 如果有 commit hash：`OpenClaw 1.2.3 (abc1234)`；沒有則只顯示版本號

這又是一個**快速路徑**模式 — 不需要建構完整的 Commander.js 程式，只需要兩個輕量模組就能回答版本查詢。

---

## 第八段：主流程 — argv 正規化到命令派發（第 101-127 行）

```typescript
process.argv = normalizeWindowsArgv(process.argv);

if (!ensureCliRespawnReady()) {
  const parsedContainer = parseCliContainerArgs(process.argv);
  if (!parsedContainer.ok) { console.error(`[openclaw] ${parsedContainer.error}`); process.exit(2); }
  const parsed = parseCliProfileArgs(parsedContainer.argv);
  if (!parsed.ok) { console.error(`[openclaw] ${parsed.error}`); process.exit(2); }
  const containerTargetName = resolveCliContainerTarget(process.argv);
  if (containerTargetName && parsed.profile) {
    console.error("[openclaw] --container cannot be combined with --profile/--dev");
    process.exit(2);
  }
  if (parsed.profile) {
    applyCliProfileEnv({ profile: parsed.profile });
    process.argv = parsed.argv;
  }
  if (!tryHandleRootVersionFastPath(process.argv)) {
    runMainOrRootHelp(process.argv);
  }
}
```

### 完整決策流程

```
normalizeWindowsArgv(process.argv)
       │
       ▼
 ensureCliRespawnReady()?
       │
   ┌───┴───┐
  true    false
   │       │
  (等待    ▼
  子程序)  parseCliContainerArgs()
           │
       ┌───┴───┐
      err      ok
       │       │
    exit(2)    ▼
          parseCliProfileArgs()
               │
           ┌───┴───┐
          err      ok
           │       │
        exit(2)    ▼
              --container + --profile 互斥檢查
                   │
               ┌───┴───┐
              衝突    ok
               │       │
            exit(2)    ▼
                  applyCliProfileEnv (如有 profile)
                       │
                       ▼
                  tryHandleRootVersionFastPath?
                       │
                   ┌───┴───┐
                  true    false
                   │       │
                  (版本    ▼
                  已輸出)  runMainOrRootHelp(argv)
```

### 旗標解析順序

注意解析是**有順序**的：

1. **Container** 先解析 — 因為 `--container` 會改變 argv 結構
2. **Profile** 後解析 — 使用 container 解析後的 argv
3. **互斥檢查** — `--container` 和 `--profile` 不能同時使用（兩者是不同的執行環境概念）

**退出碼 2** 的使用是 Unix 慣例 — 表示「使用方式錯誤」（invalid usage），有別於退出碼 1 的「一般錯誤」。

---

## 第九段：runMainOrRootHelp — 最終派發（第 129-138 行）

```typescript
function runMainOrRootHelp(argv: string[]): void {
  if (tryHandleRootHelpFastPath(argv)) return;
  import("./cli/run-main.js")
    .then(({ runCli }) => runCli(argv))
    .catch((error) => {
      console.error("[openclaw] Failed to start CLI:", error instanceof Error ? (error.stack ?? error.message) : error);
      process.exitCode = 1;
    });
}
```

這是 `entry.ts` 的**最後一哩路**。

### 兩種路徑

1. **Root Help Fast-path** — `tryHandleRootHelpFastPath(argv)` 嘗試快速輸出頂層 help（與 `openclaw.mjs` 中的邏輯類似，但此處是 TypeScript 版本，處理更多邊界情況）
2. **完整 CLI** — 動態 `import("./cli/run-main.js")` 載入命令派發引擎，呼叫 `runCli(argv)`

注意 `import()` 是**動態匯入** — `run-main.js` 及其所有依賴（Commander.js、所有子命令模組等）只有在這個 `import()` 被呼叫時才會載入。這是整個 bootstrap 鏈中最大的一次模組載入。

### 錯誤處理策略

`.catch()` 中使用 `process.exitCode = 1` 而非 `process.exit(1)`。差別在於：

- `process.exit(1)` — 立即終止程序，可能中斷正在進行的非同步操作
- `process.exitCode = 1` — 設定退出碼，讓程序自然結束

在 CLI 的 catch handler 中使用 `process.exitCode` 是更安全的做法，因為此時可能有其他非同步操作正在清理資源。

---

## 啟動序列完整時間線

以下整合 `openclaw.mjs` → `entry.ts` 的完整啟動時間線：

| 階段 | 檔案 | 動作 | 可能的提前退出 |
|------|------|------|---------------|
| T0 | `openclaw.mjs` | Node 版本檢查 | ❌ 版本不足 → `exit(1)` |
| T1 | `openclaw.mjs` | 啟用 compile cache | — |
| T2 | `openclaw.mjs` | 嘗試 bare `--help` fast-path | ✅ 直接輸出 help |
| T3 | `openclaw.mjs` | 安裝警告過濾器 | — |
| T4 | `openclaw.mjs` | 動態匯入 `dist/entry.js` | ❌ 缺失 → throw Error |
| T5 | `entry.ts` | `isMainModule` 守衛 | ✅ 非主模組 → 無動作 |
| T6 | `entry.ts` | 環境初始化 | — |
| T7 | `entry.ts` | `ensureCliRespawnReady()` | ✅ 需要 respawn → spawn 子程序 |
| T8 | `entry.ts` | Container / Profile 解析 | ❌ 格式錯誤 → `exit(2)` |
| T9 | `entry.ts` | `--version` fast-path | ✅ 輸出版本 → `exit(0)` |
| T10 | `entry.ts` | `--help` fast-path | ✅ 輸出 help |
| T11 | `entry.ts` | 動態匯入 `run-main.js` | — |
| T12 | `run-main.ts` | 完整命令派發 | （見下一章） |

---

## 引用來源

| 檔案路徑 | 行數範圍 | 本文引用段落 |
|----------|---------|-------------|
| `source-repo/src/entry.ts` | 1-16 | 匯入宣告 |
| `source-repo/src/entry.ts` | 18-21 | 包裝層對應表 |
| `source-repo/src/entry.ts` | 23-31 | shouldForceReadOnlyAuthStore |
| `source-repo/src/entry.ts` | 33-42 | 主模組守衛 |
| `source-repo/src/entry.ts` | 43-60 | 環境初始化 |
| `source-repo/src/entry.ts` | 62-82 | Respawn 機制 |
| `source-repo/src/entry.ts` | 84-99 | 版本快速路徑 |
| `source-repo/src/entry.ts` | 101-127 | 主流程 |
| `source-repo/src/entry.ts` | 129-138 | runMainOrRootHelp |
| `source-repo/src/entry.respawn.ts` | 1-60 | Respawn 計畫邏輯 |
| `source-repo/src/infra/is-main.ts` | 1-62 | 主模組判定 |
