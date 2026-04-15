# 01 — openclaw.mjs：Bootstrap 包裝層完整走讀

> **原始碼**：`source-repo/openclaw.mjs:1-181`（完整檔案）
> **角色**：npm `bin` 連結指向的最外層進入點
> **語言**：純 JavaScript（ESM）

---

## 本章摘要

`openclaw.mjs` 是整個 OpenClaw CLI 的**第一道關卡**。當使用者在終端機輸入 `openclaw` 時，作業系統透過 npm 在 `node_modules/.bin/` 建立的符號連結（symlink）執行此檔案。

此檔案完全使用純 JavaScript 撰寫，不依賴任何第三方套件，也不經過 TypeScript 編譯。它的職責是在最短時間內完成以下工作：

1. **Node.js 版本檢查** — 確認 Node ≥ 22.12，否則印出友善提示並退出
2. **啟用模組編譯快取** — `module.enableCompileCache()` 加速後續載入
3. **Bare `--help` 快速路徑** — 如果使用者只是要看頂層 help，直接輸出預運算文字，跳過完整 CLI 初始化
4. **動態匯入真正的進入點** — `dist/entry.js` 或 `dist/entry.mjs`
5. **建置缺失偵測** — 若找不到進入點，判斷是原始碼樹還是損壞的安裝

這層包裝的核心設計理念是：**儘量不載入任何東西**。能用預運算結果回答就用預運算結果，不行才動態匯入完整 CLI。

---

## 第一段：Shebang 與匯入（第 1-6 行）

```javascript
#!/usr/bin/env node

import { readFileSync } from "node:fs";
import { access } from "node:fs/promises";
import module from "node:module";
import { fileURLToPath } from "node:url";
```

**逐行解析：**

| 行 | 說明 |
|----|------|
| `#!/usr/bin/env node` | Shebang 行。告訴 Unix/macOS 作業系統用 `node` 來執行此腳本。Windows 上由 npm 的 `.cmd` shim 處理，不需要 shebang。 |
| `readFileSync` | 同步讀檔。僅用於讀取預運算的 help 文字 JSON，因為是 fast-path，刻意使用同步 I/O 減少非同步開銷。 |
| `access` | 非同步檢查檔案是否存在。用於判斷 `src/entry.ts` 是否存在（判斷是否為原始碼樹）。 |
| `module` | Node.js 內建 `module` 模組，提供 `enableCompileCache()` API。 |
| `fileURLToPath` | 將 `file://` URL 轉為檔案系統路徑。ESM 中 `import.meta.url` 是 URL 格式，需要轉換才能做字串比對。 |

**為什麼只用 Node.js 內建模組？**

因為此檔案在任何 `npm install` 的依賴載入之前就要執行。它連 TypeScript 都不經過編譯，直接以 `.mjs` 格式存在於專案根目錄。如果它依賴第三方套件，那些套件可能尚未安裝，或版本不相容。

---

## 第二段：Node.js 版本門檻（第 8-42 行）

```javascript
const MIN_NODE_MAJOR = 22;
const MIN_NODE_MINOR = 12;
const MIN_NODE_VERSION = `${MIN_NODE_MAJOR}.${MIN_NODE_MINOR}`;

const parseNodeVersion = (rawVersion) => {
  const [majorRaw = "0", minorRaw = "0"] = rawVersion.split(".");
  return {
    major: Number(majorRaw),
    minor: Number(minorRaw),
  };
};

const isSupportedNodeVersion = (version) =>
  version.major > MIN_NODE_MAJOR ||
  (version.major === MIN_NODE_MAJOR && version.minor >= MIN_NODE_MINOR);

const ensureSupportedNodeVersion = () => {
  if (isSupportedNodeVersion(parseNodeVersion(process.versions.node))) {
    return;
  }
  process.stderr.write(
    `openclaw: Node.js v${MIN_NODE_VERSION}+ is required (current: v${process.versions.node}).\n` +
      "If you use nvm, run:\n" +
      `  nvm install ${MIN_NODE_MAJOR}\n` +
      `  nvm use ${MIN_NODE_MAJOR}\n` +
      `  nvm alias default ${MIN_NODE_MAJOR}\n`,
  );
  process.exit(1);
};

ensureSupportedNodeVersion();
```

### 版本解析邏輯

`parseNodeVersion` 接收如 `"22.14.2"` 的字串，用 `.split(".")` 拆開後只取前兩段（major 和 minor），忽略 patch 版本。解構賦值中的預設值 `"0"` 確保在極端情況下（例如傳入空字串）不會得到 `NaN`。

### 版本比較邏輯

`isSupportedNodeVersion` 的比較使用**短路評估**（short-circuit evaluation）：

- 如果 `major > 22`（例如 Node 23.x），直接通過 — 不需要看 minor
- 如果 `major === 22`，則 `minor` 必須 `>= 12`
- 如果 `major < 22`，直接不通過

這是一個經典的**語義化版本比較**（semantic version comparison）模式。

### 錯誤訊息設計

注意錯誤訊息使用 `process.stderr.write()` 而非 `console.error()`。這是因為在 bootstrap 階段，`console` 物件可能尚未被正確設定（例如未來可能有 console capture 機制），直接寫入 stderr 是最安全的做法。

訊息中還貼心地附上了 nvm 指令，引導使用者升級 Node.js。

### 為什麼是 Node 22.12？

Node 22.12 引入了 `module.enableCompileCache()` 的穩定版本。OpenClaw 大量使用此功能來加速 CLI 冷啟動。這個版本門檻不只是「建議」，而是**硬性要求** — 不符合就直接 `process.exit(1)`。

---

## 第三段：啟用模組編譯快取（第 44-50 行）

```javascript
if (module.enableCompileCache && !process.env.NODE_DISABLE_COMPILE_CACHE) {
  try {
    module.enableCompileCache();
  } catch {
    // Ignore errors
  }
}
```

### 什麼是 Compile Cache？

Node.js 的模組編譯快取（Module Compile Cache）會將 JavaScript 原始碼編譯為 V8 引擎的位元碼（bytecode）後存入磁碟快取。下次載入同一模組時，可以直接讀取位元碼，跳過解析和編譯步驟。

### 防禦性檢查

此段程式碼做了**三層防禦**：

1. **功能存在檢查** — `module.enableCompileCache` 是否存在（向後相容更舊的 Node 版本，雖然前面已經卡了版本門檻）
2. **環境變數禁用** — `NODE_DISABLE_COMPILE_CACHE` 可以關閉此功能（除錯用途）
3. **try-catch** — 即使呼叫失敗也不中斷啟動流程

這體現了 bootstrap 層的核心原則：**永遠不要因為非關鍵功能的失敗而中斷 CLI 啟動**。

---

## 第四段：模組載入錯誤判定工具（第 52-72 行）

```javascript
const isModuleNotFoundError = (err) =>
  err && typeof err === "object" && "code" in err && err.code === "ERR_MODULE_NOT_FOUND";

const isDirectModuleNotFoundError = (err, specifier) => {
  if (!isModuleNotFoundError(err)) {
    return false;
  }
  const expectedUrl = new URL(specifier, import.meta.url);
  if ("url" in err && err.url === expectedUrl.href) {
    return true;
  }
  const message = "message" in err && typeof err.message === "string" ? err.message : "";
  const expectedPath = fileURLToPath(expectedUrl);
  return (
    message.includes(`Cannot find module '${expectedPath}'`) ||
    message.includes(`Cannot find module "${expectedPath}"`)
  );
};
```

### 為什麼需要 `isDirectModuleNotFoundError`？

`import("./dist/entry.js")` 失敗可能有兩種情況：

1. **直接不存在** — `dist/entry.js` 本身找不到（表示尚未建置）
2. **間接缺失** — `dist/entry.js` 存在，但它內部 `import` 的某個依賴找不到（表示依賴安裝不完整）

這兩種情況需要不同的處理方式。情況 1 可以嘗試下一個候選檔案（`.mjs`）；情況 2 應該直接拋出錯誤，因為問題不是檔案不存在，而是依賴鏈斷了。

`isDirectModuleNotFoundError` 透過比對**錯誤物件中的 URL 或訊息內容**，精確判斷是否為「指定的那個檔案找不到」。它同時處理了兩種 Node.js 錯誤格式：

- 較新版本：錯誤物件包含 `.url` 屬性
- 較舊版本：僅在 `.message` 中包含路徑

---

## 第五段：警告過濾器安裝（第 74-88 行）

```javascript
const installProcessWarningFilter = async () => {
  for (const specifier of ["./dist/warning-filter.js", "./dist/warning-filter.mjs"]) {
    try {
      const mod = await import(specifier);
      if (typeof mod.installProcessWarningFilter === "function") {
        mod.installProcessWarningFilter();
        return;
      }
    } catch (err) {
      if (isDirectModuleNotFoundError(err, specifier)) {
        continue;
      }
      throw err;
    }
  }
};
```

此函式嘗試載入 `warning-filter` 模組（先嘗試 `.js` 再嘗試 `.mjs`），用於過濾 Node.js 的實驗性功能警告（ExperimentalWarning）。

**雙副檔名嘗試模式**在整個 `openclaw.mjs` 中反覆出現。這是因為建置輸出可能是 `.js`（CommonJS 或 ESM，取決於 `package.json` 的 `type` 設定）或 `.mjs`（明確的 ESM）。bootstrap 層不假設建置產物的副檔名，而是兩者都嘗試。

---

## 第六段：動態匯入與存在檢查工具（第 90-110 行）

```javascript
const tryImport = async (specifier) => {
  try {
    await import(specifier);
    return true;
  } catch (err) {
    if (isDirectModuleNotFoundError(err, specifier)) {
      return false;
    }
    throw err;
  }
};

const exists = async (specifier) => {
  try {
    await access(new URL(specifier, import.meta.url));
    return true;
  } catch {
    return false;
  }
};
```

- `tryImport` — 嘗試動態匯入，成功回傳 `true`，直接缺失回傳 `false`，其他錯誤向上拋出
- `exists` — 純粹檢查檔案是否存在，使用 `fs.access` 而非 `import`

---

## 第七段：建置缺失錯誤訊息（第 112-124 行）

```javascript
const buildMissingEntryErrorMessage = async () => {
  const lines = ["openclaw: missing dist/entry.(m)js (build output)."];
  if (!(await exists("./src/entry.ts"))) {
    return lines.join("\n");
  }
  lines.push("This install looks like an unbuilt source tree or GitHub source archive.");
  lines.push("Build locally with `pnpm install && pnpm build`, or install a built package instead.");
  lines.push("For pinned GitHub installs, use `npm install -g github:openclaw/openclaw#<ref>` ...");
  lines.push("For releases, use `npm install -g openclaw@latest`.");
  return lines.join("\n");
};
```

此函式根據上下文產生不同的錯誤訊息：

| 條件 | 訊息策略 |
|------|---------|
| `src/entry.ts` 不存在 | 簡短訊息：建置產物缺失 |
| `src/entry.ts` 存在 | 詳細訊息：看起來是未建置的原始碼樹，提供 `pnpm build` 等指引 |

這種**情境感知的錯誤訊息**對使用者體驗非常重要。直接從 GitHub tarball 下載原始碼安裝是常見的錯誤操作，專門偵測這種情況可以省去使用者大量的 debug 時間。

---

## 第八段：Bare Root Help 快速路徑（第 126-162 行）

```javascript
const isBareRootHelpInvocation = (argv) =>
  argv.length === 3 && (argv[2] === "--help" || argv[2] === "-h");

const loadPrecomputedRootHelpText = () => {
  try {
    const raw = readFileSync(new URL("./dist/cli-startup-metadata.json", import.meta.url), "utf8");
    const parsed = JSON.parse(raw);
    return typeof parsed?.rootHelpText === "string" && parsed.rootHelpText.length > 0
      ? parsed.rootHelpText
      : null;
  } catch {
    return null;
  }
};

const tryOutputBareRootHelp = async () => {
  if (!isBareRootHelpInvocation(process.argv)) {
    return false;
  }
  const precomputed = loadPrecomputedRootHelpText();
  if (precomputed) {
    process.stdout.write(precomputed);
    return true;
  }
  for (const specifier of ["./dist/cli/program/root-help.js", "./dist/cli/program/root-help.mjs"]) {
    try {
      const mod = await import(specifier);
      if (typeof mod.outputRootHelp === "function") {
        mod.outputRootHelp();
        return true;
      }
    } catch (err) {
      if (isDirectModuleNotFoundError(err, specifier)) {
        continue;
      }
      throw err;
    }
  }
  return false;
};
```

### 快速路徑邏輯

這是整個 `openclaw.mjs` 中**最重要的效能優化**。完整流程如下：

```
使用者輸入: openclaw --help
        │
        ▼
  isBareRootHelpInvocation?
  (argv 恰好 3 個元素，且 argv[2] 是 --help 或 -h)
        │
    ┌───┴───┐
    No      Yes
    │       │
    │       ▼
    │   loadPrecomputedRootHelpText()
    │   嘗試讀取 dist/cli-startup-metadata.json
    │       │
    │   ┌───┴───┐
    │   有值    null
    │   │       │
    │   │       ▼
    │   │   動態匯入 root-help 模組
    │   │   (.js 或 .mjs)
    │   │       │
    │   ▼       ▼
    │   stdout.write  outputRootHelp()
    │       │           │
    │       ▼           ▼
    │      程式結束    程式結束
    ▼
  繼續正常啟動流程
```

### 預運算 Help 的來源

`dist/cli-startup-metadata.json` 是在**建置時期**產生的 JSON 檔案，其中 `rootHelpText` 欄位包含了完整的頂層 help 文字。使用 `readFileSync`（同步讀取）是刻意的選擇 — 在這個 fast-path 中，同步 I/O 的延遲遠低於啟動一個完整的非同步事件迴圈所需的時間。

### 為什麼這個優化重要？

`openclaw --help` 可能是使用者最常執行的指令之一（尤其是初次使用者），但如果走完整啟動流程，需要：

- 載入 TypeScript 編譯後的所有模組
- 初始化 Commander.js
- 註冊所有子命令
- 建構 help 文字

這可能需要數百毫秒。預運算 fast-path 只需讀取一個 JSON 檔案並輸出字串，可在數毫秒內完成。

---

## 第九段：主流程 — 頂層 await（第 164-181 行）

```javascript
if (await tryOutputBareRootHelp()) {
  // OK
} else {
  await installProcessWarningFilter();
  if (await tryImport("./dist/entry.js")) {
    // OK
  } else if (await tryImport("./dist/entry.mjs")) {
    // OK
  } else {
    throw new Error(await buildMissingEntryErrorMessage());
  }
}
```

這是整個檔案的**主執行流程**，使用 top-level `await`（ESM 的特性）。

### 執行決策樹

```
┌──────────────────────────┐
│ tryOutputBareRootHelp()  │
│ 是 bare --help 嗎？      │
└──────────┬───────────────┘
       ┌───┴───┐
      true    false
       │       │
       │       ▼
      結束  installProcessWarningFilter()
            安裝警告過濾器
               │
               ▼
         tryImport("./dist/entry.js")
               │
           ┌───┴───┐
          true    false
           │       │
          結束      ▼
              tryImport("./dist/entry.mjs")
                   │
               ┌───┴───┐
              true    false
               │       │
              結束      ▼
                    throw Error
                    (建置產物缺失)
```

### 關鍵觀察

1. **Bare help 優先** — 如果是 `--help`，完全不需要載入 `entry.js`，直接結束
2. **`.js` 優先於 `.mjs`** — 先嘗試 `.js`，失敗才嘗試 `.mjs`
3. **匯入即執行** — `tryImport("./dist/entry.js")` 的 `import()` 一旦成功，`entry.js`（即 `entry.ts` 的編譯產物）內部的頂層程式碼就會立即執行。`tryImport` 回傳 `true` 時，CLI 其實已經啟動了
4. **最終防線** — 如果兩個候選都找不到，拋出帶有情境感知訊息的 Error

---

## 設計模式總結

| 模式 | 實例 | 目的 |
|------|------|------|
| 零依賴 Bootstrap | 整個檔案只用 `node:*` 內建模組 | 確保在任何安裝狀態下都能執行 |
| 雙副檔名嘗試 | `.js` → `.mjs` fallback | 相容不同建置輸出格式 |
| 預運算 Fast-path | `cli-startup-metadata.json` | 極速回應 `--help` |
| 情境感知錯誤訊息 | 偵測原始碼樹 vs 損壞安裝 | 引導使用者自助修復 |
| 防禦性錯誤處理 | `isDirectModuleNotFoundError` 區分直接/間接缺失 | 精準錯誤傳播 |
| 功能降級 | compile cache 失敗不中斷啟動 | 非關鍵功能的優雅降級 |

---

## 引用來源

| 檔案路徑 | 行數範圍 | 本文引用段落 |
|----------|---------|-------------|
| `source-repo/openclaw.mjs` | 1-6 | Shebang 與匯入 |
| `source-repo/openclaw.mjs` | 8-42 | Node.js 版本門檻 |
| `source-repo/openclaw.mjs` | 44-50 | 啟用模組編譯快取 |
| `source-repo/openclaw.mjs` | 52-72 | 模組載入錯誤判定工具 |
| `source-repo/openclaw.mjs` | 74-88 | 警告過濾器安裝 |
| `source-repo/openclaw.mjs` | 90-110 | 動態匯入與存在檢查工具 |
| `source-repo/openclaw.mjs` | 112-124 | 建置缺失錯誤訊息 |
| `source-repo/openclaw.mjs` | 126-162 | Bare Root Help 快速路徑 |
| `source-repo/openclaw.mjs` | 164-181 | 主流程 — 頂層 await |
