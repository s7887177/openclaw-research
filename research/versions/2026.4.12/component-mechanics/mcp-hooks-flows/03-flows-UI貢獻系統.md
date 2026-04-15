# Flows UI 貢獻系統（Flow Contribution System）

> **引用範圍**：`src/flows/` 全部 11 個檔案
> **字數目標**：5,000–8,000 字

OpenClaw 的 Flow 系統是一個**UI 貢獻框架**（contribution framework），讓不同模組——channel adapter、LLM provider、搜尋引擎——能透過統一的介面向 CLI 和設定精靈（setup wizard）註冊選項。當使用者執行 `openclaw setup` 或 `openclaw doctor` 時，看到的每一個可選項目背後都是一個 `FlowContribution` 物件。本章將從型別系統出發，逐一拆解 channel setup、doctor health、model picker、provider 與 search 五大 flow。

---

## 1. Flow 型別系統（types.ts，59 行）

### 1.1 核心型別

```typescript
// source-repo/src/flows/types.ts:6
type FlowContributionKind = "channel" | "core" | "provider" | "search";
```

四種貢獻類型（kind）對應了 OpenClaw 的四大可擴充面向：

| Kind | 說明 | 範例 |
|------|------|------|
| `channel` | 通訊頻道 | WhatsApp、Discord、Telegram |
| `core` | 核心功能 | 健康檢查、安全設定 |
| `provider` | LLM 提供者 | OpenAI、Anthropic、Ollama |
| `search` | 搜尋引擎 | Tavily、Brave Search |

```typescript
// source-repo/src/flows/types.ts:8
type FlowContributionSurface = "auth-choice" | "health" | "model-picker" | "setup";
```

四種表面（surface）定義了選項出現在哪個 UI 場景中：

| Surface | 說明 |
|---------|------|
| `auth-choice` | 認證選擇畫面 |
| `health` | Doctor 健康檢查 |
| `model-picker` | 模型選擇器 |
| `setup` | 初始設定精靈 |

### 1.2 FlowOption

```typescript
// source-repo/src/flows/types.ts:16-24（概要）
interface FlowOption<Value> {
  value: Value;                    // 選項值
  label: string;                   // 顯示標籤
  hint?: string;                   // 提示文字
  group?: string;                  // 分組名稱
  docs?: string;                   // 文件連結
  assistantPriority?: number;      // AI 助手排序優先度
  assistantVisibility?: "visible" | "manual-only"; // AI 助手可見性
}
```

`assistantVisibility` 是一個有趣的設計：

- `"visible"`：AI 助手可以主動建議這個選項
- `"manual-only"`：只有使用者明確要求時才會顯示

這讓某些進階或罕用的選項不會干擾一般使用者的體驗，但仍然可以被直接存取。

### 1.3 FlowContribution

```typescript
// source-repo/src/flows/types.ts:26-32（概要）
interface FlowContribution<Value> {
  id: string;                      // 唯一識別
  kind: FlowContributionKind;      // 類型
  surface: FlowContributionSurface; // 表面
  option: FlowOption<Value>;       // 選項內容
  source: string;                  // 來源模組
}
```

### 1.4 合併函式

```typescript
// source-repo/src/flows/types.ts:34-48（概要）
function mergeFlowContributions<V>(
  primary: FlowContribution<V>[],
  fallbacks: FlowContribution<V>[]
): FlowContribution<V>[] {
  const seen = new Set(primary.map(c => c.option.value));
  const merged = [...primary];

  for (const fb of fallbacks) {
    if (!seen.has(fb.option.value)) {
      merged.push(fb);
      seen.add(fb.option.value);
    }
  }

  return merged;
}
```

`mergeFlowContributions()` 使用「primary 優先」策略：如果 primary 和 fallbacks 中有相同 value 的選項，保留 primary 的版本。這讓 plugin 可以覆蓋內建選項的標籤或提示，同時不丟失內建選項。

### 1.5 排序函式

```typescript
// source-repo/src/flows/types.ts:50-58（概要）
function sortFlowContributionsByLabel<V>(
  contributions: FlowContribution<V>[]
): FlowContribution<V>[] {
  return contributions.sort((a, b) =>
    a.option.label.localeCompare(b.option.label)
  );
}
```

依標籤字母順序排列，確保使用者每次看到的選項順序一致。

---

## 2. Channel Setup Flow

Channel Setup Flow 是使用者設定通訊頻道時的互動流程。

### 2.1 主流程（channel-setup.ts）

Channel Setup 採用**後寫 hook 收集模式**（post-write hook collection），支援多帳號設定：

```
使用者選擇頻道 → 選擇操作 → 執行操作 → 收集後寫 hook → 執行
```

支援的操作：

| 操作 | 說明 |
|------|------|
| `update` | 更新頻道配置（如更新 API key） |
| `disable` | 停用頻道（保留配置但不啟用） |
| `delete` | 刪除頻道配置 |
| `skip` | 跳過此頻道 |

它使用 **Adapter 模式**讓每個 plugin 定義自己的 channel setup 邏輯。核心只負責流程編排（orchestration），具體的 UI 提示和配置寫入由各 channel plugin 實作。

### 2.2 頻道狀態彙整（channel-setup.status.ts，172 行）

```typescript
// source-repo/src/flows/channel-setup.status.ts（概要）
async function collectChannelStatus(): Promise<{
  installedPlugins: PluginInfo[];      // 已安裝的 channel plugin
  catalogEntries: CatalogEntry[];      // 可安裝的頻道
  statusByChannel: Map<string, ChannelStatus>; // 各頻道狀態
  statusLines: string[];               // 狀態摘要行（供 CLI 顯示）
}> {
  // 彙整已安裝 + 可安裝的頻道資訊
}
```

每個頻道會有三種可能的狀態：

```typescript
type ChannelStatus =
  | { state: "configured"; enabled: boolean }  // 已配置（啟用或停用）
  | { state: "installed" }                      // 已安裝但未配置
  | { state: "not-configured" };                // 未配置
```

#### 快速啟動預設值

```typescript
// source-repo/src/flows/channel-setup.status.ts（概要）
function resolveQuickstartDefault(
  statusByChannel: Map<string, ChannelStatus>
): string | undefined {
  // 選擇最高分的頻道作為快速啟動預設值
  // 評分可能基於：安裝狀態、使用率、平台支援等
}
```

### 2.3 頻道設定提示（channel-setup.prompts.ts）

```typescript
// source-repo/src/flows/channel-setup.prompts.ts（概要）

// 已配置頻道的操作選擇
async function promptConfiguredAction(): Promise<"update" | "disable" | "delete" | "skip"> {
  // 顯示選項列表讓使用者選擇
}

// 多帳號頻道的帳號選擇
async function promptRemovalAccountId(accounts: Account[]): Promise<string> {
  // 當頻道有多個帳號時，讓使用者選擇要操作的帳號
}
```

此外還處理了 **DM 政策配置**（Direct Message policy），讓使用者控制是否接受陌生人的直接訊息。

---

## 3. Doctor Health Flow

Doctor 是 OpenClaw 的**自我診斷系統**，透過 Flow 機制收集各模組的健康狀態。

### 3.1 主流程（doctor-health.ts，65 行）

```typescript
// source-repo/src/flows/doctor-health.ts（概要）
async function runDoctorHealthFlow(cfg: OpenClawConfig) {
  // 1. 載入配置
  const config = await loadConfig();

  // 2. 檢查更新
  await checkForUpdates();

  // 3. 收集所有健康貢獻並執行
  const contributions = collectHealthContributions();
  for (const contribution of contributions) {
    await runHealthCheck(contribution);
  }
}
```

### 3.2 健康檢查項目（doctor-health-contributions.ts）

健康貢獻涵蓋了系統的方方面面：

| 檢查項目 | 說明 |
|----------|------|
| **認證（Auth）** | 確認 LLM 提供者的 API key 有效 |
| **記憶（Memory）** | 確認 memory plugin 正常運作 |
| **插件（Plugins）** | 確認所有啟用的 plugin manifest 完整 |
| **閘道（Gateway）** | 確認閘道可達且版本相容 |
| **服務（Services）** | 確認外部依賴服務（如 STT、TTS）可用 |
| **安全（Security）** | 確認沙箱配置、權限設定正確 |
| **沙箱（Sandbox）** | 確認 Docker / 容器環境可用 |

每個檢查項目除了回報狀態，還可以嘗試**自動修復**：

| 修復動作 | 說明 |
|----------|------|
| 舊版遷移（Legacy migrations） | 將舊版配置格式轉為新版 |
| Plugin manifest 修復 | 重新產生損壞的 manifest |
| 打包依賴修復 | 重新安裝缺失的依賴 |
| OAuth 更新 | 刷新過期的 OAuth token |

### 3.3 平台特定注意事項

Doctor 還會根據作業系統提供平台特定的建議：

- **macOS**：檢查 launchd 服務配置，確認 OpenClaw daemon 正確載入
- **Linux**：檢查 systemd linger 設定，確認使用者服務在登出後仍能運行

---

## 4. Model Picker Flow（model-picker.ts）

Model Picker 讓使用者從已配置的 LLM 提供者中選擇模型。

### 4.1 核心邏輯

```
使用者觸發模型選擇
        │
        ▼
檢查 provider 認證狀態
        │
        ▼
驗證模型白名單（allowlist）
        │
        ▼
延遲載入 picker 執行環境
        │
        ▼
顯示可用模型清單
        │
        ▼
處理 provider setup（若需要）
        │
        ▼
回傳模型目錄
```

### 4.2 關鍵設計

- **Provider 認證檢查**：在顯示模型之前，先確認對應 provider 的 API key 已設定且有效
- **模型白名單驗證**：管理員可以透過配置限制可用的模型清單
- **延遲載入**（lazy-load）：picker 的 UI 執行環境只在需要時才載入，減少啟動時間
- **Provider setup flow**：如果使用者選擇了尚未設定的 provider，自動引導進入 provider setup flow
- **模型目錄**（model catalog）：彙整所有已配置 provider 的可用模型，包含名稱、能力、定價等資訊

---

## 5. Provider Flow（provider-flow.ts）

Provider Flow 管理 LLM 提供者的設定流程。

### 5.1 選項解析

```typescript
// source-repo/src/flows/provider-flow.ts（概要）
function resolveProviderSetupFlowOptions(): FlowOption<string>[] {
  // 列出所有可用的 provider setup 選項
  // 包含：名稱、描述、文件連結
}
```

### 5.2 貢獻解析

```typescript
// source-repo/src/flows/provider-flow.ts（概要）
function resolveProviderSetupFlowContributions(): FlowContribution<string>[] {
  // 將 provider 選項轉為 FlowContribution 物件
  // 設定 kind: "provider", surface: "setup"
}
```

### 5.3 能力範圍（Scopes）

Provider 可以宣告自己支援的能力：

| Scope | 說明 |
|-------|------|
| `text-inference` | 文字推論（聊天、程式碼生成等） |
| `image-generation` | 圖片生成 |

不同 scope 的 provider 出現在不同的 setup 場景中。

### 5.4 文件連結

每個 provider 貢獻都可以附帶 `docs` 連結，指向該 provider 的設定說明文件。這些連結在 CLI 中以可點擊的超連結形式顯示（支援的終端機中）。

---

## 6. Search Setup Flow（search-setup.ts）

Search Setup 讓使用者配置 web search provider。

### 6.1 流程結構

```typescript
// source-repo/src/flows/search-setup.ts（概要）

// 搜尋提供者選項
function resolveSearchProviderOptions(): FlowOption<string>[] {
  // Tavily, Brave Search 等選項
  // 包含 credential label 格式化
}
```

### 6.2 關鍵功能

- **Web search provider 選項**：列出所有支援的搜尋引擎，每個都附帶設定指引
- **Credential label 格式化**：將 API key 的顯示格式化為安全的形式（如只顯示前後幾個字元）
- **Onboarding scope 過濾**：在初次設定時，只顯示 `text-inference` 相關的搜尋選項，避免資訊過載

---

## 7. Plugin 如何貢獻 Flow

Plugin 透過 Flow 貢獻系統向 OpenClaw 的 UI 注入選項，這個過程有三個層面：

### 7.1 貢獻物件

Plugin 在初始化時產生 `FlowContribution` 物件：

```typescript
// 範例：一個 channel plugin 的 flow 貢獻
const contribution: FlowContribution<string> = {
  id: "my-channel-setup",
  kind: "channel",
  surface: "setup",
  option: {
    value: "my-channel",
    label: "My Custom Channel",
    hint: "Connect to My Custom Channel",
    docs: "https://docs.example.com/setup",
    assistantVisibility: "visible",
  },
  source: "my-channel-plugin",
};
```

### 7.2 合併策略

當 plugin 的貢獻與內建貢獻有相同的 `value` 時，`mergeFlowContributions()` 會根據呼叫方式決定優先順序：

```typescript
// Plugin 貢獻作為 primary，內建貢獻作為 fallback
const merged = mergeFlowContributions(pluginContributions, builtinContributions);
```

這表示 plugin 可以**覆蓋**內建選項的標籤、提示或文件連結，但不能移除內建選項——如果 plugin 沒有提供替代，內建版本仍會出現。

### 7.3 AI 助手可見性控制

```typescript
// 進階選項：只在使用者明確要求時顯示
option: {
  value: "advanced-channel",
  label: "Advanced Channel (Beta)",
  assistantVisibility: "manual-only",
}
```

`assistantVisibility: "manual-only"` 的選項不會被 AI 助手主動建議。這適用於：

- 實驗性功能
- 需要特殊環境的選項
- 可能造成混淆的進階設定

---

## 8. 引用來源

| 引用代碼 | 檔案路徑 | 行號範圍 | 說明 |
|----------|----------|----------|------|
| FL-TYP-1 | `source-repo/src/flows/types.ts` | 6 | `FlowContributionKind` 型別 |
| FL-TYP-2 | `source-repo/src/flows/types.ts` | 8 | `FlowContributionSurface` 型別 |
| FL-TYP-3 | `source-repo/src/flows/types.ts` | 16–24 | `FlowOption` 介面 |
| FL-TYP-4 | `source-repo/src/flows/types.ts` | 26–32 | `FlowContribution` 介面 |
| FL-TYP-5 | `source-repo/src/flows/types.ts` | 34–48 | `mergeFlowContributions()` |
| FL-TYP-6 | `source-repo/src/flows/types.ts` | 50–58 | `sortFlowContributionsByLabel()` |
| FL-CHS-1 | `source-repo/src/flows/channel-setup.ts` | — | Channel setup 主流程 |
| FL-CSS-1 | `source-repo/src/flows/channel-setup.status.ts` | 1–172 | `collectChannelStatus()` |
| FL-CSP-1 | `source-repo/src/flows/channel-setup.prompts.ts` | — | 頻道設定提示函式 |
| FL-DOC-1 | `source-repo/src/flows/doctor-health.ts` | 1–65 | Doctor 主流程 |
| FL-DOC-2 | `source-repo/src/flows/doctor-health-contributions.ts` | — | 健康檢查貢獻項目 |
| FL-MDL-1 | `source-repo/src/flows/model-picker.ts` | — | Model picker 流程 |
| FL-PRV-1 | `source-repo/src/flows/provider-flow.ts` | — | Provider setup 流程 |
| FL-SRC-1 | `source-repo/src/flows/search-setup.ts` | — | Search setup 流程 |
