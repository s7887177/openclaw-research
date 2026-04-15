# Entry Bootstrap — 原始碼證據層：主題索引

> **層級**：Repository Evidence（原始碼證據層）
> **主題**：entry-bootstrap — OpenClaw CLI 啟動引導序列
> **版本快照**：2026.4.12
> **文件數**：4 篇深度走讀 + 本索引

---

## 本章摘要

本主題記錄 OpenClaw 從使用者在終端機輸入 `openclaw` 指令開始，一路到 Commander.js 程式解析並派發子命令為止的**完整啟動引導序列**（bootstrap sequence）。

啟動過程橫跨五個主要階段：

1. **Node.js 包裝層**（`openclaw.mjs`）— 版本檢查、編譯快取、fast-path help、動態匯入 `dist/entry.js`
2. **TypeScript 進入點**（`src/entry.ts`）— 主模組判定、環境正規化、respawn 計畫、容器 / profile 解析、版本快速路徑、進入 `runCli`
3. **命令派發引擎**（`src/cli/run-main.ts`）— dotenv 載入、proxy 設定、快速路由、Commander.js 建構、外掛指令註冊、最終解析
4. **執行環境與 NPM 匯出**（`src/runtime.ts` + `src/index.ts` + `src/library.ts`）— 執行環境抽象、公開 API 表面、惰性載入邊界

---

## 文件索引

| # | 檔案 | 涵蓋原始碼 | 核心關注點 |
|---|------|-----------|-----------|
| 01 | [openclaw.mjs bootstrap wrapper](./01-openclaw-mjs-bootstrap.md) | `openclaw.mjs:1-181` | Node 版本門檻、compile cache、help fast-path、entry 動態匯入 |
| 02 | [entry.ts 完整走讀](./02-entry-ts-walkthrough.md) | `src/entry.ts:1-213`、`src/entry.respawn.ts`、`src/infra/is-main.ts` | 主模組守衛、respawn 機制、容器/profile 解析、版本快速路徑 |
| 03 | [run-main.ts 命令派發](./03-run-main-dispatch.md) | `src/cli/run-main.ts`、`src/cli/route.ts`、`src/cli/program/build-program.ts` | 快速路由、Commander.js 程式建構、外掛指令註冊、最終解析 |
| 04 | [runtime / index / library](./04-runtime-and-library.md) | `src/runtime.ts`、`src/index.ts`、`src/library.ts` | OutputRuntimeEnv 抽象、NPM 公開 API、惰性載入邊界 |

---

## 啟動序列全景圖

```
使用者輸入 `openclaw <cmd>`
        │
        ▼
┌─────────────────────────┐
│  openclaw.mjs           │  ← npm bin 連結指向此檔
│  1. Node ≥ 22.12 檢查    │
│  2. enableCompileCache  │
│  3. bare --help 快速路徑  │
│  4. import dist/entry.js │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│  entry.ts               │
│  1. isMainModule 守衛    │
│  2. 環境正規化           │
│  3. respawn 計畫         │
│  4. container / profile  │
│  5. version fast-path    │
│  6. runMainOrRootHelp()  │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│  run-main.ts (runCli)   │
│  1. container 委派       │
│  2. dotenv / proxy       │
│  3. tryRouteCli 快速路由  │
│  4. buildProgram()       │
│  5. registerCommands     │
│  6. program.parseAsync   │
│  7. cleanup              │
└─────────────────────────┘
```

---

## 關鍵設計決策摘要

| 決策 | 理由 | 出處 |
|------|------|------|
| 以 `.mjs` 作為最外層包裝 | 確保以 ESM 模式啟動，可使用 top-level `await` | `openclaw.mjs:1` |
| Node ≥ 22.12 硬性門檻 | 需要 `module.enableCompileCache` 等新 API | `openclaw.mjs:8-9` |
| Bare `--help` 預運算 fast-path | 避免載入完整 CLI 只為印出 help 文字 | `openclaw.mjs:131-162` |
| `isMainModule` 守衛 | 同一檔案可作為 CLI 進入點或被當作函式庫匯入 | `src/entry.ts:37-42` |
| Respawn 機制 | 注入 `NODE_OPTIONS` 或 CA cert 後重啟 Node 程序 | `src/entry.respawn.ts` |
| `tryRouteCli` 快速路由 | 已知指令可跳過 Commander.js 建構，加快冷啟動 | `src/cli/route.ts` |
| Library 惰性載入 | 大型依賴（binaries、exec、web-channel）延遲到實際呼叫時才載入 | `src/library.ts` |

---

## 引用來源

| 檔案路徑 | 行數範圍 | 說明 |
|----------|---------|------|
| `source-repo/openclaw.mjs` | 1-181 | 外層 ESM 包裝、完整檔案 |
| `source-repo/src/entry.ts` | 1-213 | TypeScript 進入點、完整檔案 |
| `source-repo/src/entry.respawn.ts` | 1-60 | Respawn 計畫邏輯 |
| `source-repo/src/infra/is-main.ts` | 1-62 | 主模組判定 |
| `source-repo/src/cli/run-main.ts` | ~230 行 | 命令派發主流程 |
| `source-repo/src/cli/route.ts` | 1-65 | 快速路由 |
| `source-repo/src/cli/program/build-program.ts` | 1-28 | Commander.js 建構 |
| `source-repo/src/runtime.ts` | 1-117 | 執行環境抽象 |
| `source-repo/src/index.ts` | 1-103 | NPM 套件匯出表面 |
| `source-repo/src/library.ts` | 1-92 | 惰性載入邊界 |
