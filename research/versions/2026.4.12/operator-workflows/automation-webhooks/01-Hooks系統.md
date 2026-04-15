# Hooks 系統

## 本章摘要

本章深入介紹 OpenClaw 的 Hook 系統——一個事件驅動的自動化框架。Hook 讓操作者可以在特定事件（如收到訊息、會話開始、命令執行）發生時，自動觸發自訂程式碼。系統支援內建 Hook、Plugin Hook、工作區 Hook 與第三方安裝 Hook 四種來源。

---

## 1. Hook 概念

### 1.1 什麼是 Hook？

Hook 是 OpenClaw 中的事件處理器。當系統中發生特定事件時，Hook 會被觸發執行。每個 Hook 都有：

- **名稱**（Name）：唯一識別符
- **描述**（Description）：說明 Hook 的功能
- **事件列表**（Events）：觸發此 Hook 的事件類型
- **處理器**（Handler）：實際執行的程式碼

### 1.2 Hook 類型定義

核心的 Hook 類型定義在 `types.ts` 中：

```typescript
export type Hook = {
  name: string;
  description: string;
  source: "openclaw-bundled" | "openclaw-managed" | "openclaw-workspace" | "openclaw-plugin";
  pluginId?: string;
  filePath: string;     // HOOK.md 的路徑
  baseDir: string;      // Hook 目錄
  handlerPath: string;  // 處理器模組路徑 (handler.ts/js)
};
```
> 引用：`source-repo/src/hooks/types.ts:35-43`

### 1.3 Hook 來源

| 來源 | 說明 | 優先順序 |
|------|------|---------|
| `openclaw-bundled` | 內建 Hook，隨 OpenClaw 安裝 | 最低 |
| `openclaw-managed` | 透過 CLI 安裝的 Hook | 中 |
| `openclaw-workspace` | 工作區目錄中的 Hook | 中高 |
| `openclaw-plugin` | Plugin 提供的 Hook | 最高 |

> 引用：`source-repo/src/hooks/types.ts:38`

---

## 2. Hook 中繼資料（Metadata）

### 2.1 Metadata 結構

每個 Hook 都有豐富的中繼資料：

```typescript
export type OpenClawHookMetadata = {
  always?: boolean;           // 是否總是啟用
  hookKey?: string;           // Hook 鍵值
  emoji?: string;             // 顯示表情符號
  homepage?: string;          // 首頁 URL
  events: string[];           // 處理的事件列表
  export?: string;            // 匯出名稱（預設 "default"）
  os?: string[];              // 支援的作業系統
  requires?: {
    bins?: string[];          // 必要的二進位檔案（全部必須存在）
    anyBins?: string[];       // 至少一個必須存在
    env?: string[];           // 必要的環境變數
    config?: string[];        // 必要的配置路徑
  };
  install?: HookInstallSpec[];  // 安裝指引
};
```
> 引用：`source-repo/src/hooks/types.ts:10-27`

### 2.2 Hook 安裝規格

```typescript
export type HookInstallSpec = {
  id?: string;
  kind: "bundled" | "npm" | "git";
  label?: string;
  package?: string;      // npm 套件名
  repository?: string;   // Git 倉庫 URL
  bins?: string[];        // 安裝後的二進位檔案
};
```
> 引用：`source-repo/src/hooks/types.ts:1-8`

### 2.3 Hook Entry

完整的 Hook 載入結果：

```typescript
export type HookEntry = {
  hook: Hook;
  frontmatter: ParsedHookFrontmatter;
  metadata?: OpenClawHookMetadata;
  invocation?: HookInvocationPolicy;
};
```
> 引用：`source-repo/src/hooks/types.ts:47-52`

### 2.4 Hook Snapshot

Hook 快照用於序列化和傳輸：

```typescript
export type HookSnapshot = {
  hooks: Array<{ name: string; events: string[] }>;
  resolvedHooks?: Hook[];
  version?: number;
};
```
> 引用：`source-repo/src/hooks/types.ts:63-67`

---

## 3. Hook 生命週期

### 3.1 Hook 載入

Hook 的載入由 `loader.ts` 處理：

```typescript
// src/hooks/loader.ts
// 負責從磁碟讀取 HOOK.md，解析 frontmatter，載入處理器模組
```
> 引用：`source-repo/src/hooks/loader.ts`

### 3.2 Hook 安裝

```typescript
// src/hooks/install.ts - Hook 安裝主邏輯
// src/hooks/install.runtime.ts - 執行時安裝流程
```
> 引用：`source-repo/src/hooks/install.ts`, `source-repo/src/hooks/install.runtime.ts`

### 3.3 Hook 配置

```typescript
// src/hooks/config.ts - Hook 配置載入
```
> 引用：`source-repo/src/hooks/config.ts`

### 3.4 Hook Frontmatter 解析

HOOK.md 使用 YAML frontmatter 定義 Hook 的中繼資料：

```typescript
// src/hooks/frontmatter.ts
// 解析 HOOK.md 的 YAML frontmatter
```
> 引用：`source-repo/src/hooks/frontmatter.ts`

### 3.5 Hook 政策

```typescript
// src/hooks/policy.ts
// 定義 Hook 的啟用/停用政策
```
> 引用：`source-repo/src/hooks/policy.ts`

---

## 4. 內部 Hook

### 4.1 內部 Hook 類型

OpenClaw 定義了內部 Hook 類型，用於系統內部事件處理：

```typescript
// src/hooks/internal-hook-types.ts
// 定義系統內部使用的 Hook 事件類型
```
> 引用：`source-repo/src/hooks/internal-hook-types.ts`

### 4.2 內部 Hook 註冊

```typescript
// src/hooks/internal-hooks.ts
// 註冊和管理內部 Hook
```
> 引用：`source-repo/src/hooks/internal-hooks.ts`

---

## 5. Plugin Hook

Plugin 可以提供自己的 Hook：

```typescript
// src/hooks/plugin-hooks.ts
// 管理 Plugin 提供的 Hook
```
> 引用：`source-repo/src/hooks/plugin-hooks.ts`

Plugin Hook 允許第三方 Extension 在 OpenClaw 的事件系統中註冊處理器，例如：
- 訊息收發時的前/後處理
- 會話開始/結束時的動作
- 自訂命令觸發

---

## 6. 內建 Hook

### 6.1 Bundled Hooks 目錄

OpenClaw 在 `src/hooks/bundled/` 目錄下提供內建的 Hook：

> 引用：`source-repo/src/hooks/bundled`

### 6.2 Hook 工作區管理

```typescript
// src/hooks/workspace.ts
// 管理工作區中使用者自訂的 Hook
```
> 引用：`source-repo/src/hooks/workspace.ts`

---

## 7. Gmail Hook 整合

OpenClaw 提供深度的 Gmail 整合，作為 Hook 系統的一個重要應用：

### 7.1 Gmail 監看器

```typescript
// src/hooks/gmail-watcher.ts - Gmail 監看器主邏輯
// src/hooks/gmail-watcher-lifecycle.ts - 監看器生命週期管理
// src/hooks/gmail-watcher-errors.ts - 錯誤處理
```
> 引用：`source-repo/src/hooks/gmail-watcher.ts`, `source-repo/src/hooks/gmail-watcher-lifecycle.ts`

### 7.2 Gmail 操作

```typescript
// src/hooks/gmail-ops.ts - Gmail 操作（讀取、標記、搜尋等）
// src/hooks/gmail.ts - Gmail Hook 主入口
```
> 引用：`source-repo/src/hooks/gmail-ops.ts`, `source-repo/src/hooks/gmail.ts`

### 7.3 Gmail 設定

```typescript
// src/hooks/gmail-setup-utils.ts - Gmail 設定工具函式
```
> 引用：`source-repo/src/hooks/gmail-setup-utils.ts`

Gmail Hook 讓 OpenClaw 可以：
- 監看收件匣中的新郵件
- 根據規則自動回覆
- 提取郵件內容供 AI 處理
- 管理郵件標籤和標記

---

## 8. Hook CLI

### 8.1 Hook 管理命令

```bash
openclaw hooks list      # 列出所有 Hook
openclaw hooks install   # 安裝 Hook
openclaw hooks status    # 查看 Hook 狀態
```

```typescript
// src/cli/hooks-cli.ts
// Hook CLI 命令定義
```
> 引用：`source-repo/src/cli/hooks-cli.ts`

### 8.2 Hook 狀態

```typescript
// src/hooks/hooks-status.ts
// Hook 狀態查詢
```
> 引用：`source-repo/src/hooks/hooks-status.ts`

---

## 9. 訊息 Hook

### 9.1 訊息 Hook 映射

Hook 可以在訊息處理流程中介入：

```typescript
// src/hooks/message-hook-mappers.ts
// 將收到的訊息事件映射到對應的 Hook
```
> 引用：`source-repo/src/hooks/message-hook-mappers.ts`

### 9.2 Gateway Hook 整合

Gateway 層面的 Hook 整合：

```typescript
// src/gateway/hooks.ts - Gateway Hook 管理
// src/gateway/hooks-mapping.ts - Gateway Hook 事件映射
// src/gateway/hooks-policy.ts - Gateway Hook 政策
```
> 引用：`source-repo/src/gateway/hooks.ts`, `source-repo/src/gateway/hooks-mapping.ts`

---

## 10. 自動回覆（Auto-Reply）

OpenClaw 有一個專門的自動回覆模組：

```typescript
// src/auto-reply/
// 自動回覆規則引擎
```
> 引用：`source-repo/src/auto-reply`

自動回覆允許在特定條件下（如特定頻道、特定訊息模式）自動觸發 AI 回應，無需使用者手動操作。

---

## 11. Webhook Extension

OpenClaw 提供 Webhook Extension 用於接收外部系統的 HTTP 請求：

```
extensions/webhooks/
```
> 引用：`source-repo/extensions/webhooks`

### 11.1 Webhook CLI

```bash
openclaw webhooks     # Webhook 管理命令
```

```typescript
// src/cli/webhooks-cli.ts
// Webhook CLI 命令
```
> 引用：`source-repo/src/cli/webhooks-cli.ts`

---

## 12. Fire-and-Forget Hook

對於不需要等待結果的 Hook，系統支援 Fire-and-Forget 模式：

```typescript
// src/hooks/fire-and-forget.ts
// 觸發後不等待結果的 Hook 執行模式
```
> 引用：`source-repo/src/hooks/fire-and-forget.ts`

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/hooks/types.ts:1-68` | Hook 類型定義完整檔案 |
| `source-repo/src/hooks/hooks.ts` | Hook 公開 API |
| `source-repo/src/hooks/loader.ts` | Hook 載入器 |
| `source-repo/src/hooks/config.ts` | Hook 配置 |
| `source-repo/src/hooks/install.ts` | Hook 安裝 |
| `source-repo/src/hooks/frontmatter.ts` | Frontmatter 解析 |
| `source-repo/src/hooks/policy.ts` | Hook 政策 |
| `source-repo/src/hooks/internal-hooks.ts` | 內部 Hook |
| `source-repo/src/hooks/plugin-hooks.ts` | Plugin Hook |
| `source-repo/src/hooks/gmail.ts` | Gmail Hook |
| `source-repo/src/hooks/gmail-watcher.ts` | Gmail 監看器 |
| `source-repo/src/hooks/gmail-ops.ts` | Gmail 操作 |
| `source-repo/src/hooks/fire-and-forget.ts` | Fire-and-Forget 模式 |
| `source-repo/src/cli/hooks-cli.ts` | Hook CLI |
| `source-repo/src/cli/webhooks-cli.ts` | Webhook CLI |
| `source-repo/src/gateway/hooks.ts` | Gateway Hook 整合 |
| `source-repo/extensions/webhooks` | Webhook Extension |
