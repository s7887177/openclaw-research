# 建置與工具鏈（Build & Toolchain）— 主題索引

> **層級**：Repository Evidence · Topic 4  
> **版本快照**：2026.4.12  
> **最後更新**：2026-07-21

---

## 本章摘要

本主題記錄 OpenClaw 專案在「建置系統」與「開發工具鏈」方面的所有原始事實（Raw Facts）。OpenClaw 是一個以 TypeScript 為主要語言的大型 AI 自動化框架，其建置流程涉及多層工具協同運作——從 TypeScript 編譯設定、tsdown 打包器、pnpm 腳本體系，到 CI/CD 流水線與程式碼品質閘門。理解這些基礎設施，是深入研究 OpenClaw 架構的必要前提。

本主題共分為四個子章節，涵蓋建置與品質保證的完整鏈路：

---

## 子章節索引

| # | 檔案 | 標題 | 內容概要 |
|---|------|------|----------|
| 01 | [01-tsconfig-configuration.md](./01-tsconfig-configuration.md) | TypeScript 編譯設定詳解 | 三份 `tsconfig.json` 的完整走讀：主設定檔的模組解析策略（NodeNext）、路徑別名（Path Aliases）、ES2023 目標；Plugin SDK 型別宣告專用設定；oxlint 專用設定。深入解析每一個 `compilerOptions` 欄位的技術意涵與設計考量。 |
| 02 | [02-tsdown-build-config.md](./02-tsdown-build-config.md) | tsdown 打包器設定詳解 | OpenClaw 的核心建置設定檔 `tsdown.config.ts` 完整走讀：基於 Rolldown 的打包策略、25+ 進入點定義、外部依賴排除清單、Plugin SDK 子路徑處理、警告抑制策略。包含建置產物結構分析。 |
| 03 | [03-pnpm-scripts.md](./03-pnpm-scripts.md) | pnpm 腳本參考手冊 | `package.json` 中 100+ 腳本的完整分類與解析：核心建置腳本、開發模式、檢查流水線、格式化與靜態分析、測試體系、架構守衛（Architectural Guard）腳本。包含腳本間的依賴關係與執行順序分析。 |
| 04 | [04-ci-and-code-quality.md](./04-ci-and-code-quality.md) | CI/CD 與程式碼品質工具 | GitHub Actions CI 流水線（1378 行、20+ 工作）的完整分析；oxlint 靜態分析設定；oxfmt 格式化設定；vitest 測試框架；knip 死碼偵測；pre-commit hooks；以及其他輔助工具（jscpd、madge、markdownlint、shellcheck、detect-secrets、zizmor）。 |

---

## 核心開發依賴總覽

以下表格彙整 OpenClaw 建置與品質工具鏈的關鍵 devDependencies，作為四個子章節的共同參考：

| 套件 | 版本 | 用途 | 相關子章節 |
|------|------|------|-----------|
| `typescript` | ^6.0.2 | TypeScript 型別檢查（tsc） | 01 |
| `@typescript/native-preview` | 7.0.0-dev | tsgo 原生型別檢查器 | 01, 03 |
| `tsdown` | 0.21.7 | 基於 Rolldown 的打包器 | 02 |
| `vitest` | ^4.1.4 | 測試框架 | 03, 04 |
| `@vitest/coverage-v8` | ^4.1.4 | V8 引擎覆蓋率報告 | 04 |
| `oxlint` | ^1.59.0 | 靜態分析工具（Linter） | 03, 04 |
| `oxfmt` | 0.44.0 | 程式碼格式化工具 | 03, 04 |
| `tsx` | ^4.21.0 | TypeScript 直接執行器 | 03 |
| `jscpd` | 4.0.9 | 複製貼上偵測器 | 04 |
| `madge` | ^8.0.0 | 模組匯入循環偵測器 | 03, 04 |
| `jsdom` | ^29.0.2 | DOM 測試環境 | 04 |
| `semver` | 7.7.4 | 語意化版本處理 | 03 |
| `lit` | ^3.3.2 | Web Components 框架（UI 用） | — |

---

## 技術棧概覽圖

```
┌─────────────────────────────────────────────────────────┐
│                     開發者工作流程                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐            │
│  │ TypeScript│   │  oxlint  │   │  oxfmt   │            │
│  │ tsconfig  │   │ .oxlintrc│   │.oxfmtrc  │            │
│  │  × 3 份   │   │  靜態分析 │   │ 格式化   │            │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘            │
│       │              │              │                    │
│       ▼              ▼              ▼                    │
│  ┌──────────────────────────────────────────┐           │
│  │        pnpm scripts (100+ 腳本)          │           │
│  │  build / check / lint / test / format    │           │
│  └──────────────────┬───────────────────────┘           │
│                     │                                    │
│       ┌─────────────┼─────────────┐                     │
│       ▼             ▼             ▼                      │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐               │
│  │  tsdown  │  │  vitest  │  │ pre-commit│              │
│  │  打包器   │  │  測試框架 │  │  hooks   │              │
│  └────┬────┘  └────┬─────┘  └────┬─────┘               │
│       │            │             │                       │
│       ▼            ▼             ▼                       │
│  ┌──────────────────────────────────────────┐           │
│  │    GitHub Actions CI (ci.yml, 1378 行)    │           │
│  │    20+ 工作 · 多平台 · 分片測試            │           │
│  └──────────────────────────────────────────┘           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 閱讀建議

1. **初次閱讀**：建議依序閱讀 01 → 02 → 03 → 04，因為後面的章節會引用前面章節所建立的概念。
2. **快速查閱**：若只需查找特定腳本或工具的設定，可直接跳到對應子章節。
3. **搭配原始碼**：建議搭配 `source-repo/` 目錄中的原始碼同步閱讀，以獲得最完整的理解。

---

## 引用來源

| 來源檔案 | 行號範圍 | 用途 |
|----------|----------|------|
| `source-repo/tsconfig.json` | 1-33 | 主 TypeScript 設定 |
| `source-repo/tsconfig.plugin-sdk.dts.json` | 1-15 | Plugin SDK 型別宣告設定 |
| `source-repo/tsconfig.oxlint.json` | 1-5 | oxlint 專用 TypeScript 設定 |
| `source-repo/tsdown.config.ts` | 1-195 | tsdown 打包器設定 |
| `source-repo/package.json` | scripts 區段 | pnpm 腳本定義 |
| `source-repo/.github/workflows/ci.yml` | 1-1378 | CI/CD 流水線定義 |
| `source-repo/.oxlintrc.json` | 1-67 | oxlint 靜態分析設定 |
| `source-repo/.oxfmtrc.jsonc` | 1-20 | oxfmt 格式化設定 |
| `source-repo/vitest.config.ts` | 1-7 | Vitest 測試設定 |
| `source-repo/knip.config.ts` | 1-122 | 死碼偵測設定 |
| `source-repo/.pre-commit-config.yaml` | 1-30+ | Pre-commit hooks 設定 |
| `source-repo/.oxlintrc.json` | 1-67 | Linter 設定 |
| `source-repo/.jscpd.json` | — | 複製貼上偵測設定 |
| `source-repo/.markdownlint-cli2.jsonc` | — | Markdown lint 設定 |
| `source-repo/.shellcheckrc` | — | Shell script 檢查設定 |
| `source-repo/.detect-secrets.cfg` | — | 密鑰偵測設定 |
| `source-repo/zizmor.yml` | — | GitHub Actions 安全掃描設定 |
