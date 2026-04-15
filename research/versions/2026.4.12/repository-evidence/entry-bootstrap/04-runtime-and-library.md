# 04 — runtime.ts / index.ts / library.ts：執行環境抽象、NPM 匯出與惰性載入

> **原始碼**：
> - `source-repo/src/runtime.ts:1-117` — 執行環境抽象層
> - `source-repo/src/index.ts:1-103` — NPM 套件匯出表面（public API surface）
> - `source-repo/src/library.ts:1-92` — 惰性載入邊界定義
>
> **角色**：定義 OpenClaw 作為**函式庫**（library）被使用時的對外介面與執行環境抽象
> **語言**：TypeScript

---

## 本章摘要

前三章走讀了 OpenClaw 作為 **CLI 工具**的啟動路徑。但 OpenClaw 不只是 CLI — 它也可以作為 **NPM 套件**被其他 Node.js 程式匯入使用。本章探討三個支撐此雙重身份的核心檔案：

1. **`runtime.ts`** — 定義 `RuntimeEnv` 和 `OutputRuntimeEnv` 抽象，將「寫入 stdout」、「寫入 stderr」、「退出程序」等副作用封裝為可替換的介面。這使得同一份業務邏輯可以在 CLI、測試、SDK 等不同環境中執行。

2. **`index.ts`** — NPM 套件的 `main` 進入點。當其他程式 `import "openclaw"` 時，載入的就是這個檔案。它有**雙模式行為**：作為主模組時啟動 CLI，作為依賴匯入時匯出函式庫 API。

3. **`library.ts`** — 定義了所有**惰性載入邊界**（lazy-load boundaries）。重型模組（如二進位管理、程序執行、Web channel 監控等）只在第一次呼叫時才載入，避免匯入 `openclaw` 時就拉入整個依賴樹。

這三個檔案共同構成了 OpenClaw 的**公開 API 架構**。

---

## 一、runtime.ts — 執行環境抽象層

### 1.1 核心型別定義

`runtime.ts` 定義了兩個關鍵型別：

```typescript
// 概念性型別定義（基於原始碼推測）
type RuntimeEnv = {
  log: (...args: any[]) => void;     // 寫入日誌（通常是 stderr）
  error: (...args: any[]) => void;   // 寫入錯誤
  exit: (code: number) => never;     // 退出程序
};

type OutputRuntimeEnv = RuntimeEnv & {
  writeStdout: (data: string) => void;   // 寫入標準輸出
  writeJson: (data: unknown) => void;    // 寫入 JSON 格式輸出
};
```

### 1.2 為什麼需要 RuntimeEnv 抽象？

| 使用場景 | `log` 實作 | `error` 實作 | `exit` 實作 |
|----------|-----------|-------------|-------------|
| **生產 CLI** | `console.error` | `console.error` | `process.exit` |
| **單元測試** | 靜默（no-op）或收集到陣列 | 靜默或收集 | 拋出例外（不真正退出） |
| **SDK 嵌入** | 記錄到宿主應用的日誌系統 | 記錄到宿主 | 拋出例外 |
| **Vitest 環境** | 由 `shouldEmitRuntimeLog()` 決定 | 同左 | 拋出例外 |

這種模式在軟體工程中稱為**依賴注入**（Dependency Injection, DI）。業務邏輯不直接呼叫 `console.log` 或 `process.exit`，而是透過 `RuntimeEnv` 介面間接呼叫。這讓同一份程式碼可以在完全不同的環境中執行，而不需要修改業務邏輯。

### 1.3 shouldEmitRuntimeLog / shouldEmitRuntimeStdout

```typescript
// 概念性實作
function shouldEmitRuntimeLog(): boolean {
  // 在 vitest 環境中，除非有 mock 設定，否則抑制日誌輸出
  if (isVitestEnvironment() && !hasMockRuntimeEnv()) {
    return false;
  }
  return true;
}

function shouldEmitRuntimeStdout(): boolean {
  // 類似邏輯，針對 stdout
  if (isVitestEnvironment() && !hasMockRuntimeEnv()) {
    return false;
  }
  return true;
}
```

這兩個函式的目的是在測試環境中**自動抑制輸出**。

在 Vitest（OpenClaw 使用的測試框架）中執行測試時，大量的 `console.log` 輸出會干擾測試結果的可讀性。但如果測試明確設定了 mock runtime（表示它想驗證輸出內容），則允許輸出。

### 1.4 defaultRuntime — 生產環境實作

```typescript
// 概念性實作
const defaultRuntime: OutputRuntimeEnv = {
  log: (...args) => {
    if (shouldEmitRuntimeLog()) {
      console.error(...args);
    }
  },
  error: (...args) => {
    if (shouldEmitRuntimeLog()) {
      console.error(...args);
    }
  },
  exit: (code) => {
    process.exit(code);
  },
  writeStdout: (data) => {
    if (shouldEmitRuntimeStdout()) {
      process.stdout.write(data);
    }
  },
  writeJson: (data) => {
    // 見 writeRuntimeJson
  },
};
```

注意 `log` 和 `error` 都寫入 **stderr** 而非 stdout。這是 CLI 工具的慣例：

- **stdout** 用於程式的「正式輸出」（可以被管道到其他程式）
- **stderr** 用於日誌、進度訊息、錯誤訊息等「旁白」

### 1.5 createNonExitingRuntime — 測試用實作

```typescript
// 概念性實作
function createNonExitingRuntime(): OutputRuntimeEnv {
  return {
    ...defaultRuntime,
    exit: (code) => {
      throw new ExitError(code);  // 拋出例外而非真正退出
    },
  };
}
```

在測試中，我們不希望 `runtime.exit(1)` 真的結束 Node.js 程序（那會中斷所有其他測試）。`createNonExitingRuntime` 將 `exit` 替換為拋出特殊例外 `ExitError`，測試可以用 `expect(...).toThrow(ExitError)` 來驗證退出行為。

### 1.6 writeRuntimeJson

```typescript
// 概念性實作
function writeRuntimeJson(runtime: OutputRuntimeEnv, data: unknown): void {
  runtime.writeStdout(JSON.stringify(data, null, 2) + "\n");
}
```

此工具函式將任意資料序列化為格式化的 JSON 並寫入 stdout。OpenClaw 的許多指令支援 `--json` 輸出模式，此函式是統一的 JSON 輸出管道。

---

## 二、index.ts — NPM 套件進入點

### 2.1 雙模式架構

`index.ts` 是 `package.json` 中 `"main"` 或 `"exports"` 指向的檔案。它的行為取決於**如何被載入**：

```
模式一：作為主模組（CLI）
$ node dist/index.js chat "hello"
→ 安裝錯誤處理器
→ 呼叫 runLegacyCliEntry()

模式二：作為依賴（Library）
import { loadConfig, runExec } from "openclaw";
→ 匯出 library.ts 中定義的所有 API
```

### 2.2 主模組模式（第 84-103 行概念）

```typescript
// 概念性結構
if (isMainModule({ currentFile: fileURLToPath(import.meta.url) })) {
  // 安裝全域未攔截例外處理器
  process.on("uncaughtException", (error) => {
    console.error("[openclaw] Uncaught exception:", error);
    process.exitCode = 1;
  });
  process.on("unhandledRejection", (reason) => {
    console.error("[openclaw] Unhandled rejection:", reason);
    process.exitCode = 1;
  });
  
  // 呼叫舊版 CLI 進入點
  runLegacyCliEntry();
}
```

`runLegacyCliEntry` 暗示這是一個**向後相容**的進入點。新的 CLI 啟動路徑是 `openclaw.mjs → entry.ts → run-main.ts`，但為了相容舊版安裝或直接執行 `node dist/index.js` 的情況，`index.ts` 保留了獨立的 CLI 啟動能力。

### 2.3 函式庫模式（第 60-83 行概念）

```typescript
// 惰性匯出（概念性）
export {
  // 直接 re-export（輕量模組）
  applyTemplate,
  createDefaultDeps,
  deriveSessionKey,
  describePortOwner,
  ensurePortAvailable,
  handlePortError,
  loadConfig,
  loadSessionStore,
  normalizeE164,
  PortInUseError,
  resolveSessionKey,
  resolveStorePath,
  saveSessionStore,
  waitForever,
  
  // 惰性 re-export（重型模組，來自 library.ts）
  ensureBinary,
  getReplyFromConfig,
  monitorWebChannel,
  promptYesNo,
  runCommandWithTimeout,
  runExec,
} from "./library.js";
```

### 2.4 完整匯出 API 表面

以下是 `index.ts` 匯出的所有公開 API，按功能分類：

| 類別 | API 名稱 | 說明 | 載入方式 |
|------|---------|------|---------|
| **設定** | `loadConfig` | 載入 OpenClaw 設定檔 | 直接匯出 |
| **設定** | `createDefaultDeps` | 建立預設依賴注入容器 | 直接匯出 |
| **會話管理** | `loadSessionStore` | 載入會話（session）儲存 | 直接匯出 |
| **會話管理** | `saveSessionStore` | 儲存會話資料 | 直接匯出 |
| **會話管理** | `deriveSessionKey` | 衍生會話金鑰 | 直接匯出 |
| **會話管理** | `resolveSessionKey` | 解析會話金鑰 | 直接匯出 |
| **會話管理** | `resolveStorePath` | 解析儲存路徑 | 直接匯出 |
| **範本** | `applyTemplate` | 套用文字範本 | 直接匯出 |
| **網路** | `describePortOwner` | 描述佔用指定 port 的程序 | 直接匯出 |
| **網路** | `ensurePortAvailable` | 確保 port 可用 | 直接匯出 |
| **網路** | `handlePortError` | 處理 port 佔用錯誤 | 直接匯出 |
| **網路** | `PortInUseError` | Port 佔用錯誤類別 | 直接匯出 |
| **電話號碼** | `normalizeE164` | 正規化 E.164 電話號碼格式 | 直接匯出 |
| **工具** | `waitForever` | 無限等待（保持程序存活） | 直接匯出 |
| **AI 回覆** | `getReplyFromConfig` | 從設定取得 AI 回覆 | **惰性載入** |
| **互動** | `promptYesNo` | Yes/No 互動提示 | **惰性載入** |
| **二進位管理** | `ensureBinary` | 確保外部二進位檔存在 | **惰性載入** |
| **程序執行** | `runExec` | 執行外部程序 | **惰性載入** |
| **程序執行** | `runCommandWithTimeout` | 執行外部程序（帶逾時） | **惰性載入** |
| **Web Channel** | `monitorWebChannel` | 監控 Web 頻道 | **惰性載入** |

---

## 三、library.ts — 惰性載入邊界

### 3.1 什麼是惰性載入（Lazy Loading）？

惰性載入是一種**延遲初始化**模式。模組不在 `import` 時載入，而是在**第一次使用時**才載入。

```
一般匯入:
import { runExec } from "./process/exec.js";
// ↑ 此行執行時，exec.js 及其所有依賴就會被載入到記憶體中

惰性匯入:
const runExec = lazyLoad(() => import("./process/exec.js").then(m => m.runExec));
// ↑ 此行執行時，什麼都不會被載入
// 直到有人呼叫 runExec() 時，才會觸發 import()
```

### 3.2 為什麼 OpenClaw 需要惰性載入？

當其他程式 `import "openclaw"` 時，如果所有模組都立即載入，會發生：

| 問題 | 影響 |
|------|------|
| 啟動時間增加 | 載入數十個不一定需要的模組 |
| 記憶體佔用增加 | 所有模組的 AST 和位元碼都佔記憶體 |
| 副作用風險 | 某些模組可能在載入時就執行副作用（如建立連線） |
| 可能的載入失敗 | 某些模組依賴外部二進位檔或系統功能，在某些環境下無法載入 |

惰性載入確保使用者只付出**實際使用功能的成本**。

### 3.3 五個惰性載入邊界

`library.ts` 定義了**五個**惰性載入邊界，每個對應一個重型模組：

#### 邊界 1：AI 回覆系統

```typescript
// source-repo/src/library.ts — 惰性載入邊界 1
// 實際模組: ./auto-reply/reply.runtime.js
// 匯出: getReplyFromConfig
```

| 屬性 | 值 |
|------|---|
| 來源模組 | `./auto-reply/reply.runtime.js` |
| 匯出函式 | `getReplyFromConfig` |
| 為何惰性 | AI 回覆系統依賴 LLM 客戶端、提示樣板等重型依賴，大多數 library 使用者不需要 |

#### 邊界 2：互動提示

```typescript
// source-repo/src/library.ts — 惰性載入邊界 2
// 實際模組: ./cli/prompt.js
// 匯出: promptYesNo
```

| 屬性 | 值 |
|------|---|
| 來源模組 | `./cli/prompt.js` |
| 匯出函式 | `promptYesNo` |
| 為何惰性 | 互動提示依賴 readline 或 inquirer 等 CLI 互動函式庫，在非 CLI 環境（如 SDK）中不需要 |

#### 邊界 3：二進位檔管理

```typescript
// source-repo/src/library.ts — 惰性載入邊界 3
// 實際模組: ./infra/binaries.js
// 匯出: ensureBinary
```

| 屬性 | 值 |
|------|---|
| 來源模組 | `./infra/binaries.js` |
| 匯出函式 | `ensureBinary` |
| 為何惰性 | 二進位檔管理涉及平台偵測、下載邏輯、校驗等，是相當重型的模組 |

#### 邊界 4：程序執行

```typescript
// source-repo/src/library.ts — 惰性載入邊界 4
// 實際模組: ./process/exec.js
// 匯出: runExec, runCommandWithTimeout
```

| 屬性 | 值 |
|------|---|
| 來源模組 | `./process/exec.js` |
| 匯出函式 | `runExec`、`runCommandWithTimeout` |
| 為何惰性 | 程序執行模組依賴 `child_process` 的進階封裝和信號處理邏輯 |

#### 邊界 5：Web Channel 監控

```typescript
// source-repo/src/library.ts — 惰性載入邊界 5
// 實際模組: ./plugins/runtime/runtime-web-channel-plugin.js
// 匯出: monitorWebChannel
```

| 屬性 | 值 |
|------|---|
| 來源模組 | `./plugins/runtime/runtime-web-channel-plugin.js` |
| 匯出函式 | `monitorWebChannel` |
| 為何惰性 | Web channel 監控涉及 WebSocket 連線和事件系統，是最重型的模組之一 |

### 3.4 直接匯出（非惰性）

以下模組因為夠輕量，不需要惰性載入：

```typescript
// source-repo/src/library.ts — 直接 re-export
export {
  applyTemplate,         // 純字串處理，無外部依賴
  createDefaultDeps,     // 建立物件，無 I/O
  deriveSessionKey,      // 純密碼學計算
  describePortOwner,     // 輕量 net 操作
  ensurePortAvailable,   // 輕量 net 操作
  handlePortError,       // 錯誤處理邏輯
  loadConfig,            // 讀取設定檔
  loadSessionStore,      // 讀取 session 檔案
  normalizeE164,         // 純字串處理
  PortInUseError,        // 錯誤類別定義
  resolveSessionKey,     // 純計算
  resolveStorePath,      // 路徑解析
  saveSessionStore,      // 寫入 session 檔案
  waitForever,           // setTimeout 封裝
};
```

### 3.5 惰性載入 vs 直接匯出的分界線

```
                   模組複雜度 / 依賴數量
                   ──────────────────→
                   │
  直接匯出區域      │  惰性載入區域
                   │
  applyTemplate    │  getReplyFromConfig
  loadConfig       │  (依賴 LLM 客戶端)
  normalizeE164    │
  resolveStorePath │  ensureBinary
  waitForever      │  (依賴平台偵測、下載)
  PortInUseError   │
  createDefaultDeps│  runExec
  deriveSessionKey │  (依賴進階 child_process)
  ...              │
                   │  monitorWebChannel
                   │  (依賴 WebSocket)
                   │
                   │  promptYesNo
                   │  (依賴 readline/inquirer)
```

分界的**經驗法則**：

- **直接匯出**：只依賴 Node.js 內建模組或純計算邏輯
- **惰性載入**：依賴第三方套件、涉及網路 I/O、或具有平台特定邏輯

---

## 四、三個檔案的協作關係

```
使用者程式碼:
  import { loadConfig, runExec } from "openclaw";

                    │
                    ▼
            ┌──────────────┐
            │   index.ts   │  ← package.json "exports" 指向
            │              │
            │  re-exports  │
            │  from        │
            │  library.ts  │
            └──────┬───────┘
                   │
                   ▼
            ┌──────────────┐
            │ library.ts   │
            │              │
            │ loadConfig   │──→ 直接從 ./config/... 匯出
            │              │
            │ runExec      │──→ 惰性載入 ./process/exec.js
            │ (lazy)       │    ↑ 第一次呼叫 runExec() 時才觸發
            └──────────────┘

所有業務邏輯中:
  function someHandler(runtime: OutputRuntimeEnv) {
    runtime.log("Processing...");     // ← 透過 runtime 抽象
    runtime.writeStdout(result);
    if (fatal) runtime.exit(1);
  }
                    │
                    ▼
            ┌──────────────┐
            │  runtime.ts  │
            │              │
            │ CLI 模式:    │──→ process.stdout / stderr / exit
            │ 測試模式:    │──→ 靜默 / 收集 / throw ExitError
            │ SDK 模式:    │──→ 宿主應用的日誌系統
            └──────────────┘
```

---

## 五、設計哲學總結

### 5.1 runtime.ts 體現的原則

| 原則 | 實踐 |
|------|------|
| **關注點分離**（Separation of Concerns） | 業務邏輯與 I/O 副作用分離 |
| **依賴反轉**（Dependency Inversion） | 高層模組依賴抽象介面，不依賴具體實作 |
| **可測試性**（Testability） | 可以注入 mock runtime 進行純邏輯測試 |
| **環境無關性**（Environment Agnosticism） | 同一份程式碼在 CLI、測試、SDK 中都能執行 |

### 5.2 index.ts 體現的原則

| 原則 | 實踐 |
|------|------|
| **雙模式進入點**（Dual-mode Entry） | 同一檔案支援 CLI 和 Library 兩種使用方式 |
| **向後相容**（Backward Compatibility） | `runLegacyCliEntry` 保留舊版啟動路徑 |
| **最小公開表面**（Minimal API Surface） | 只匯出經過篩選的 API，內部模組不公開 |

### 5.3 library.ts 體現的原則

| 原則 | 實踐 |
|------|------|
| **惰性初始化**（Lazy Initialization） | 重型模組延遲到使用時才載入 |
| **按需付費**（Pay-as-you-go） | 只載入實際使用的功能模組 |
| **漸進揭露**（Progressive Disclosure） | 輕量 API 立即可用，重型 API 按需載入 |
| **載入邊界明確化** | 五個惰性邊界有清晰的劃分標準 |

---

## 引用來源

| 檔案路徑 | 行數範圍 | 本文引用段落 |
|----------|---------|-------------|
| `source-repo/src/runtime.ts` | 1-117 | 一、runtime.ts 全段 |
| `source-repo/src/index.ts` | 1-103 | 二、index.ts 全段 |
| `source-repo/src/index.ts` | 60-83 | 函式庫模式匯出 |
| `source-repo/src/index.ts` | 84-103 | 主模組模式（Legacy CLI） |
| `source-repo/src/library.ts` | 1-92 | 三、library.ts 全段 |
| `source-repo/src/library.ts` | （5 個邊界） | 惰性載入邊界定義 |
| `source-repo/src/library.ts` | （直接匯出區段） | 直接 re-export 列表 |
