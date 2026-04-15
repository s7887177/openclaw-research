# Onboarding 精靈與 Doctor 診斷

## 本章摘要

本章深入介紹 OpenClaw 的兩大使用者引導機制：**Onboarding 精靈**（首次設定流程）與 **Doctor 診斷**（健康檢查與自動修復）。Onboarding 精靈引導使用者完成 Provider 認證、Channel 配置與 Daemon 安裝；Doctor 則提供超過 20 項自動化健康檢查，能偵測並修復常見問題。

---

## 1. Onboarding 精靈

### 1.1 啟動方式

Onboarding 精靈是 OpenClaw 推薦的首次設定路徑：

```bash
openclaw onboard --install-daemon
```
> 引用：`source-repo/README.md:28-29, 107`

精靈使用 `@clack/prompts` 套件提供美觀的終端機互動介面。

### 1.2 核心架構

Onboarding 流程由以下幾個關鍵檔案組成：

| 檔案 | 職責 |
|------|------|
| `src/commands/onboard.ts` | 主命令入口，處理認證、重設範圍、風險確認 |
| `src/commands/onboard-interactive.ts` | 互動式設定函式 `runInteractiveSetup()` |
| `src/wizard/setup.ts` | 設定精靈主協調器 `runSetupWizard()` |
| `src/wizard/setup.gateway-config.ts` | Gateway 配置步驟 |
| `src/flows/channel-setup.ts` | Channel 認證與配置 |
| `src/flows/provider-flow.ts` | Provider 認證流程 |

### 1.3 設定流程步驟

精靈的完整流程如下：

#### 步驟 1：歡迎與模式選擇

`runSetupWizard()` 會先顯示歡迎訊息，然後讓使用者選擇設定模式：

- **Quickstart（快速開始）**：使用預設值，最少互動
- **Advanced（進階）**：完整的自訂配置

> 引用：`source-repo/src/wizard/setup.ts:123-200`

#### 步驟 2：Provider 認證（Auth Choice）

系統會列出所有可用的 LLM Provider，讓使用者選擇認證方式：

```typescript
// promptAuthChoiceGrouped() - 互動式 Provider 選擇
// applyAuthChoice() - 套用選擇的 Provider 認證
```
> 引用：`source-repo/src/wizard/setup.ts:487-553`

每個 Provider 在 `openclaw.plugin.json` 中定義了自己的認證選項（`providerAuthChoices`）。例如 OpenAI 提供兩種認證方式：

1. **OpenAI Codex (ChatGPT OAuth)**：透過瀏覽器登入
2. **OpenAI API key**：直接輸入 API 金鑰

```json
"providerAuthChoices": [
  {
    "provider": "openai-codex",
    "method": "oauth",
    "choiceId": "openai-codex",
    "choiceLabel": "OpenAI Codex (ChatGPT OAuth)",
    "choiceHint": "Browser sign-in",
    "groupId": "openai",
    "groupLabel": "OpenAI"
  },
  {
    "provider": "openai",
    "method": "api-key",
    "choiceId": "openai-api-key",
    "choiceLabel": "OpenAI API key",
    "groupId": "openai",
    "groupLabel": "OpenAI"
  }
]
```
> 引用：`source-repo/extensions/openai/openclaw.plugin.json:12-37`

Anthropic 也有類似結構：
- **Claude CLI**：重用本地 Claude CLI 的登入狀態
- **API Key**：直接輸入 Anthropic API 金鑰

> 引用：`source-repo/extensions/anthropic/openclaw.plugin.json:12-37`

#### 步驟 3：Gateway 配置

`configureGatewayForSetup()` 處理 Gateway 的認證模式選擇：

- **token**：使用共享 Token 認證
- **password**：密碼認證
- **remote**：遠端存取配置
- **oauth**：OAuth 認證

> 引用：`source-repo/src/wizard/setup.gateway-config.ts:54+`

#### 步驟 4：Channel 配置

`setupChannels()` 引導使用者設定通訊頻道：

```typescript
// src/flows/channel-setup.ts:107+
// setupChannels() - 互動式 Channel 認證與配置
```
> 引用：`source-repo/src/flows/channel-setup.ts:107+`

OpenClaw 支援超過 20 種 Channel，包括：WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage、BlueBubbles、IRC、Microsoft Teams、Matrix、Feishu、LINE、Mattermost、Nextcloud Talk、Nostr、Synology Chat、Tlon、Twitch、Zalo、WeChat、WebChat。

> 引用：`source-repo/README.md:22`

#### 步驟 5：Daemon 安裝

如果指定了 `--install-daemon` 參數，精靈會根據作業系統安裝對應的 Daemon 服務：

```typescript
// src/commands/daemon-install-plan.shared.ts
// resolveDaemonInstallRuntimeInputs() - 解析 Daemon 安裝參數
// emitDaemonInstallRuntimeWarning() - 發出執行時警告
```
> 引用：`source-repo/src/commands/daemon-install-plan.shared.ts:1-50+`

Daemon CLI 提供以下操作：

| 命令 | 函式 | 說明 |
|------|------|------|
| `openclaw daemon start` | `runDaemonStart` | 啟動 Daemon |
| `openclaw daemon stop` | `runDaemonStop` | 停止 Daemon |
| `openclaw daemon status` | `runDaemonStatus` | 查看 Daemon 狀態 |
| `openclaw daemon install` | `runDaemonInstall` | 安裝 Daemon 服務 |
| `openclaw daemon uninstall` | `runDaemonUninstall` | 解除安裝 Daemon 服務 |

> 引用：`source-repo/src/cli/daemon-cli.ts:1-16`

在 Linux 上，如果需要使用 systemd user service，精靈還會處理 systemd linger 的設定：

```typescript
// ensureSystemdUserLingerInteractive()
```
> 引用：`source-repo/src/flows/doctor-health-contributions.ts:52`

### 1.4 配置寫入

所有設定完成後，精靈會將配置寫入 `~/.openclaw/openclaw.json`：

```typescript
import { CONFIG_PATH, readConfigFileSnapshot, writeConfigFile } from "../config/config.js";
```
> 引用：`source-repo/src/flows/doctor-health-contributions.ts:53`

精靈還會套用 Wizard 中繼資料（Metadata），記錄設定的來源和時間戳：

```typescript
import { applyWizardMetadata, randomToken } from "../commands/onboard-helpers.js";
```
> 引用：`source-repo/src/flows/doctor-health-contributions.ts:51`

---

## 2. Doctor 診斷

### 2.1 啟動方式

```bash
openclaw doctor
```

Doctor 命令在 CLI 中的註冊：

```
.command("doctor")
// 描述: "Health checks + quick fixes for the gateway and channels"
// 文件連結: docs.openclaw.ai/cli/doctor
// 選項: --no-workspace-suggestions (停用記憶系統建議)
```
> 引用：`source-repo/src/cli/program/register.maintenance.ts:12-29`

### 2.2 Doctor 主流程

Doctor 的主入口是 `doctorCommand()`：

```typescript
export async function doctorCommand(
  runtime: RuntimeEnv = defaultRuntime,
  options: DoctorOptions = {},
) {
  const prompter = createDoctorPrompter({ runtime, options });
  printWizardHeader(runtime);
  intro("OpenClaw doctor");
  // ... 執行檢查 ...
  await runDoctorHealthContributions(ctx);
  outro("Doctor complete.");
}
```
> 引用：`source-repo/src/flows/doctor-health.ts:19-65`

流程步驟：
1. 建立 Doctor Prompter（用於互動式修復確認）
2. 檢查是否有更新版本（`maybeOfferUpdateBeforeDoctor`）
3. 修復 UI Protocol 新鮮度（`maybeRepairUiProtocolFreshness`）
4. 檢查原始碼安裝問題（`noteSourceInstallIssues`）
5. 提供啟動最佳化提示（`noteStartupOptimizationHints`）
6. 載入並可能遷移配置（`loadAndMaybeMigrateDoctorConfig`）
7. 執行所有健康檢查貢獻（`runDoctorHealthContributions`）

> 引用：`source-repo/src/flows/doctor-health.ts:33-62`

### 2.3 健康檢查項目

`doctor-health-contributions.ts` 是 Doctor 的核心，它聚合了超過 20 項健康檢查。以下是完整的檢查項目清單：

#### 認證相關

| 檢查項目 | 函式 | 說明 |
|----------|------|------|
| OAuth Profile 修復 | `maybeRepairLegacyOAuthProfileIds` | 修復舊版 OAuth Profile ID |
| Auth Profile 健康 | `noteAuthProfileHealth` | 檢查認證設定狀態 |
| Legacy Codex Override | `noteLegacyCodexProviderOverride` | 偵測過時的 Provider 覆蓋 |
| OpenAI OAuth TLS | `noteOpenAIOAuthTlsPrerequisites` | 檢查 OAuth 所需的 TLS 前置條件 |

> 引用：`source-repo/src/flows/doctor-health-contributions.ts:13-16, 50`

#### Gateway 與 Daemon

| 檢查項目 | 函式 | 說明 |
|----------|------|------|
| Gateway 健康 | `checkGatewayHealth` | 探測 Gateway 是否正在運行 |
| Gateway Memory 狀態 | `probeGatewayMemoryStatus` | 檢查 Gateway 記憶體狀態 |
| Daemon 修復 | `maybeRepairGatewayDaemon` | 修復 Daemon 配置 |
| Gateway Service 配置 | `maybeRepairGatewayServiceConfig` | 修復 Gateway 服務配置 |
| 額外 Gateway Services | `maybeScanExtraGatewayServices` | 掃描其他 Gateway 服務 |

> 引用：`source-repo/src/flows/doctor-health-contributions.ts:23-28`

#### Plugin 與 Sandbox

| 檢查項目 | 函式 | 說明 |
|----------|------|------|
| Plugin Manifest 修復 | `maybeRepairLegacyPluginManifestContracts` | 修復舊版 Plugin 清單 |
| Bundled Plugin 依賴 | `maybeRepairBundledPluginRuntimeDeps` | 修復內建 Plugin 的執行時依賴 |
| Sandbox 映像 | `maybeRepairSandboxImages` | 修復沙箱 Docker 映像 |
| Sandbox 範圍警告 | `noteSandboxScopeWarnings` | 警告沙箱權限問題 |

> 引用：`source-repo/src/flows/doctor-health-contributions.ts:19, 38, 40-41`

#### 記憶與工作區

| 檢查項目 | 函式 | 說明 |
|----------|------|------|
| Memory Recall 修復 | `maybeRepairMemoryRecallHealth` | 修復記憶召回功能 |
| Memory Recall 狀態 | `noteMemoryRecallHealth` | 檢查記憶召回健康度 |
| Memory Search 狀態 | `noteMemorySearchHealth` | 檢查記憶搜尋功能 |
| 工作區狀態 | `noteWorkspaceStatus` | 檢查工作區配置 |
| 工作區備份提示 | `noteWorkspaceBackupTip` | 建議備份策略 |

> 引用：`source-repo/src/flows/doctor-health-contributions.ts:30-33, 48-49`

#### 系統狀態

| 檢查項目 | 函式 | 說明 |
|----------|------|------|
| Bootstrap 檔案大小 | `noteBootstrapFileSize` | 驗證 Bootstrap 初始化 |
| Chrome MCP Browser | `noteChromeMcpBrowserReadiness` | 檢查瀏覽器自動化就緒度 |
| Claude CLI 健康 | `noteClaudeCliHealth` | 檢查 Claude CLI 狀態 |
| Shell Completion | `doctorShellCompletion` | 修復 Shell 自動完成 |
| Cron Store 修復 | `maybeRepairLegacyCronStore` | 修復舊版 Cron 資料儲存 |
| Channel Plugin 啟動 | `runChannelPluginStartupMaintenance` | 執行 Channel Plugin 啟動維護 |
| 安全性警告 | `noteSecurityWarnings` | 檢查安全性設定 |
| Session Lock 健康 | `noteSessionLockHealth` | 檢查 Session 鎖定狀態 |
| 狀態完整性 | `noteStateIntegrity` | 驗證狀態資料完整性 |
| Legacy 遷移 | `detectLegacyStateMigrations` / `runLegacyStateMigrations` | 偵測並執行舊版資料遷移 |
| macOS LaunchAgent | `noteMacLaunchAgentOverrides` / `noteMacLaunchctlGatewayEnvOverrides` | macOS 特有的 LaunchAgent 檢查 |
| Memory System 建議 | `shouldSuggestMemorySystem` | 建議啟用記憶系統 |

> 引用：`source-repo/src/flows/doctor-health-contributions.ts:17-52`

### 2.4 Doctor 子模組

Doctor 的檢查邏輯分散在多個專門的模組中：

```
src/commands/
├── doctor.ts                           # 主入口（re-export）
├── doctor-auth.ts                      # 認證健康檢查
├── doctor-bootstrap-size.ts            # Bootstrap 檔案大小
├── doctor-browser.ts                   # 瀏覽器就緒度
├── doctor-bundled-plugin-runtime-deps.ts # 內建 Plugin 依賴
├── doctor-claude-cli.ts                # Claude CLI 健康
├── doctor-completion.ts                # Shell 自動完成
├── doctor-config-flow.ts               # 配置驗證流程
├── doctor-cron.ts                      # Cron 資料遷移
├── doctor-gateway-daemon-flow.ts       # Gateway Daemon 修復
├── doctor-gateway-health.ts            # Gateway 健康探測
├── doctor-gateway-services.ts          # Gateway 服務配置
├── doctor-install.ts                   # 原始碼安裝問題
├── doctor-memory-search.ts             # 記憶搜尋修復
├── doctor-platform-notes.ts            # 平台特有提示
├── doctor-plugin-manifests.ts          # Plugin 清單修復
├── doctor-sandbox.ts                   # Sandbox 映像修復
├── doctor-security.ts                  # 安全性警告
├── doctor-session-locks.ts             # Session 鎖定健康
├── doctor-state-integrity.ts           # 狀態完整性
├── doctor-state-migrations.ts          # 狀態遷移
├── doctor-ui.ts                        # UI 資產檢查
├── doctor-update.ts                    # 更新/重建檢查
├── doctor-workspace.ts                 # 工作區配置
└── doctor-workspace-status.ts          # 工作區狀態
```

### 2.5 Doctor 的修復能力

Doctor 不只是診斷工具——它能主動修復問題。以「maybeRepair」為前綴的函式都有自動修復能力：

1. **互動式修復**：Doctor 會詢問使用者是否要套用修復
2. **自動遷移**：舊版資料格式會自動遷移到新版
3. **配置修復**：無效的配置值會被更正

當配置無效時，系統會引導使用者執行 Doctor：

```typescript
// src/cli/program/config-guard.ts
// Doctor 繞過一般的配置驗證（允許在配置損壞時仍能執行）
// 訊息中會引用 `openclaw doctor --fix` 來修復
```
> 引用：`source-repo/src/cli/program/config-guard.ts`

### 2.6 使用場景

| 場景 | 建議操作 |
|------|---------|
| 首次安裝後 | `openclaw doctor` — 驗證所有設定 |
| 升級 OpenClaw 後 | `openclaw doctor` — 處理可能的遷移 |
| Gateway 無法啟動 | `openclaw doctor` — 檢查配置與 Daemon |
| Channel 連線失敗 | `openclaw doctor` — 檢查 Channel 認證 |
| 記憶功能異常 | `openclaw doctor` — 修復記憶系統 |
| 配置檔損壞 | `openclaw doctor --fix` — 嘗試自動修復 |

---

## 3. CLI 依賴管理

OpenClaw CLI 使用懶載入（Lazy Loading）策略來管理 Channel 模組依賴：

```typescript
// src/cli/deps.ts
export function createDefaultDeps(): CliDeps {
  // 遍歷 listChannelPlugins()
  // 為每個 Channel 建立懶載入的 sender proxy
  // 避免不必要的模組載入
}
```
> 引用：`source-repo/src/cli/deps.ts:27-60`

這種設計確保 CLI 啟動速度不會因為大量 Channel 模組而變慢——只有在實際使用某個 Channel 時才會載入對應模組。

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/README.md:28-29, 107, 110` | Onboarding 命令與說明 |
| `source-repo/src/commands/onboard.ts:20-80` | Onboard 命令入口 |
| `source-repo/src/commands/onboard-interactive.ts:9-31` | 互動式設定 |
| `source-repo/src/wizard/setup.ts:123-200, 487-553, 582-592` | 設定精靈主協調器 |
| `source-repo/src/wizard/setup.gateway-config.ts:54+` | Gateway 認證模式配置 |
| `source-repo/src/flows/channel-setup.ts:107+` | Channel 設定流程 |
| `source-repo/src/flows/doctor-health.ts:1-65` | Doctor 主流程 |
| `source-repo/src/flows/doctor-health-contributions.ts:1-55` | Doctor 健康檢查聚合 |
| `source-repo/src/cli/daemon-cli.ts:1-16` | Daemon CLI 命令 |
| `source-repo/src/cli/daemon-cli/register.ts:6-14` | Daemon 命令註冊 |
| `source-repo/src/commands/daemon-install-plan.shared.ts:1-50+` | Daemon 安裝計畫 |
| `source-repo/extensions/openai/openclaw.plugin.json:12-37` | OpenAI 認證選項 |
| `source-repo/extensions/anthropic/openclaw.plugin.json:12-37` | Anthropic 認證選項 |
| `source-repo/src/cli/deps.ts:27-60` | CLI 懶載入依賴 |
| `source-repo/src/cli/program/register.maintenance.ts:12-29` | Doctor 命令註冊 |
| `source-repo/src/cli/program/config-guard.ts` | 配置防護與 Doctor 繞過 |
