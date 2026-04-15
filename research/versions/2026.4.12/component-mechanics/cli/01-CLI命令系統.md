# CLI 命令系統：OpenClaw 的命令列介面完整解析

## 本章摘要

OpenClaw 的命令列介面（CLI, Command-Line Interface）是使用者與系統互動的主要入口之一。整個 CLI 系統建構於 Node.js 之上，使用 Commander.js 作為命令框架，提供了超過 28 個子命令（subcommand），涵蓋從系統管理、安全稽核、通道配置到 ACP 橋接等完整功能。

本章將從目錄結構開始，逐層深入 CLI 的核心機制：命令分發器（command dispatcher）如何接收與路由指令、快速路徑（fast-path routing）如何繞過完整的程式建構以提升效能、參數解析（argument parsing）如何處理跨平台相容性，以及各子命令如何與 Gateway 互動。我們也會探討 ACP（Agent Client Protocol）橋接模式——這是 OpenClaw 與 IDE 整合的關鍵通道，以及互動式提示、配置管理和安全稽核等實用功能。

讀完本章後，你將完整理解 OpenClaw CLI 從使用者輸入指令到最終執行的完整生命週期。

---

## 1. `src/cli/` 目錄結構

OpenClaw 的 CLI 源碼集中在 `src/cli/` 目錄下，包含超過 **218 個檔案**。這些檔案大致可分為以下幾類：

### 1.1 核心基礎設施

| 檔案 | 用途 |
|------|------|
| `run-main.ts` | CLI 主入口與命令分發器（main entry point & command dispatcher） |
| `route.ts` | 快速路徑命令路由（fast-path command routing） |
| `argv.ts` | 參數解析工具（argument parsing utilities） |
| `argv-invocation.ts` | CLI 調用解析（invocation resolution） |
| `cli-name.ts` | CLI 名稱解析，預設為 `"openclaw"` |
| `cli-utils.ts` | 共用工具函式 |
| `windows-argv.ts` | Windows 平台的 argv 正規化（normalization） |

> **引用**：`source-repo/src/cli/` 目錄，共 218 個檔案（含測試檔案）。

### 1.2 命令管理

| 檔案 | 用途 |
|------|------|
| `command-options.ts` | 共用命令選項處理 |
| `command-format.ts` | 命令格式化（自動補上 `--container`/`--profile` 旗標） |
| `command-catalog.ts` | 命令目錄（定義每個命令的載入策略與路由設定） |
| `command-registration-policy.ts` | 命令註冊策略（控制是否只註冊主要命令或全部命令） |
| `command-startup-policy.ts` | 啟動策略（控制 banner 顯示、config guard、插件載入等） |

### 1.3 環境與設定

| 檔案 | 用途 |
|------|------|
| `dotenv.ts` | `.env` 檔案載入 |
| `profile.ts` / `profile-utils.ts` | CLI 配置檔案（profile）管理 |
| `container-target.ts` | 容器目標定位（container targeting） |
| `banner.ts` / `banner-config-lite.ts` | 啟動橫幅（startup banner）顯示 |
| `prompt.ts` | 互動式 Y/N 提示 |

### 1.4 子命令實作

每個子命令通常有一個對應的 `*-cli.ts` 檔案。我們將在第 4 節詳細列出完整清單。

### 1.5 Gateway 通訊

| 檔案 | 用途 |
|------|------|
| `gateway-rpc.ts` | Gateway RPC 客戶端介面 |
| `gateway-rpc.runtime.ts` | Gateway RPC 執行時實作 |
| `gateway-rpc.types.ts` | Gateway RPC 型別定義 |
| `gateway-secret-options.ts` | Gateway 認證選項解析 |

---

## 2. `run-main.ts`：主命令分發器

`run-main.ts` 是整個 CLI 的心臟。當你在終端輸入 `openclaw` 加上任何參數時，最終都會流經這個檔案中的 `runCli()` 函式。

### 2.1 `runCli()` 完整流程

`runCli()` 函式位於第 150-287 行，其執行流程可以拆解為以下步驟：

```
使用者輸入
    │
    ▼
① normalizeWindowsArgv(argv)     ─── Windows 相容性正規化
    │
    ▼
② parseCliContainerArgs          ─── 解析 --container 旗標或 OPENCLAW_CONTAINER 環境變數
    │
    ▼
③ parseCliProfileArgs            ─── 解析 --profile / --dev 旗標
    │
    ▼
④ maybeRunCliInContainer         ─── 如有容器目標，委派至容器內執行
    │
    ▼
⑤ shouldLoadCliDotEnv()          ─── 從 cwd 或 state dir 載入 .env
    │
    ▼
⑥ normalizeEnv()                 ─── 環境變數正規化
    │
    ▼
⑦ initializeDebugProxyCapture    ─── 除錯代理擷取初始化
    │
    ▼
⑧ ensureGlobalUndiciEnvProxy...  ─── HTTP 代理設定
    │
    ▼
⑨ assertSupportedRuntime()       ─── Node.js 版本檢查
    │
    ▼
⑩ shouldUseRootHelpFastPath?     ─── 如果是根層級 help，輸出預先計算的 help 文字
    │
    ▼
⑪ tryRouteCli(normalizedArgv)    ─── 快速路徑路由（route-first fast path）
    │
    ▼
⑫ enableConsoleCapture()         ─── 啟用結構化日誌擷取
    │
    ▼
⑬ buildProgram()                 ─── 建構 Commander.js 程式
    │
    ▼
⑭ 註冊主要命令（lazy loading）   ─── 只載入需要的命令
    │
    ▼
⑮ 註冊插件 CLI 命令              ─── 除非被策略跳過
    │
    ▼
⑯ program.parseAsync(parseArgv)  ─── 執行命令
    │
    ▼
⑰ closeCliMemoryManagers()       ─── 清理記憶體管理器
```

> **引用**：`source-repo/src/cli/run-main.ts:150-287`

### 2.2 關鍵步驟詳解

#### 步驟 ①：Windows 正規化

```typescript
const originalArgv = normalizeWindowsArgv(argv);
```

Windows 環境下的 `argv` 可能包含控制字元（control characters）、多餘的引號或重複的 `node.exe` 路徑。`normalizeWindowsArgv` 會清除這些異常。

> **引用**：`source-repo/src/cli/run-main.ts:151`

#### 步驟 ②-④：容器與 Profile 解析

```typescript
const parsedContainer = parseCliContainerArgs(originalArgv);
// ...
const parsedProfile = parseCliProfileArgs(parsedContainer.argv);
// ...
const containerTarget = maybeRunCliInContainer(originalArgv);
if (containerTarget.handled) {
  if (containerTarget.exitCode !== 0) {
    process.exitCode = containerTarget.exitCode;
  }
  return;
}
```

容器目標（container target）允許 CLI 將命令轉發到正在運行的 Docker 或 Podman 容器中。如果指定了 `--container` 旗標或設定了 `OPENCLAW_CONTAINER` 環境變數，CLI 會透過 `docker exec` 或 `podman exec` 在容器內執行對應命令。

需注意：`--container` 和 `--profile`/`--dev` **不能同時使用**：

```typescript
if (containerTargetName && parsedProfile.profile) {
  throw new Error("--container cannot be combined with --profile/--dev");
}
```

> **引用**：`source-repo/src/cli/run-main.ts:152-175`

#### 步驟 ⑤：.env 載入邏輯

```typescript
function shouldLoadCliDotEnv(env: NodeJS.ProcessEnv = process.env): boolean {
  if (existsSync(path.join(process.cwd(), ".env"))) {
    return true;
  }
  return existsSync(path.join(resolveStateDir(env), ".env"));
}
```

CLI 會檢查兩個位置的 `.env` 檔案：
1. **當前工作目錄**（current working directory）的 `.env`
2. **狀態目錄**（state directory）的 `.env`

只要任一存在就會載入。載入時，CWD 的 `.env` 優先，全域 state dir 的 `.env` 作為後備且不覆蓋已設定的變數。

> **引用**：`source-repo/src/cli/run-main.ts:143-148`，`source-repo/src/cli/dotenv.ts:1-17`

#### 步驟 ⑩-⑪：快速路徑

CLI 在建構完整的 Commander.js 程式之前，會先嘗試兩種快速路徑：

1. **Root Help 快速路徑**：如果使用者只輸入了 `openclaw --help`，直接輸出預先計算好的 help 文字，無需建構完整程式
2. **Route-first 快速路徑**：透過 `tryRouteCli()` 嘗試直接路由到已註冊的命令處理器

```typescript
if (shouldUseRootHelpFastPath(normalizedArgv)) {
  const { outputPrecomputedRootHelpText } = await import("./root-help-metadata.js");
  if (!outputPrecomputedRootHelpText()) {
    const { outputRootHelp } = await import("./program/root-help.js");
    await outputRootHelp();
  }
  return;
}

if (await tryRouteCli(normalizedArgv)) {
  return;
}
```

> **引用**：`source-repo/src/cli/run-main.ts:197-208`

### 2.3 `rewriteUpdateFlagArgv`：`--update` 旗標重寫

```typescript
export function rewriteUpdateFlagArgv(argv: string[]): string[] {
  const index = argv.indexOf("--update");
  if (index === -1) {
    return argv;
  }
  const next = [...argv];
  next.splice(index, 1, "update");
  return next;
}
```

這個函式將 `openclaw --update` 重寫為 `openclaw update`，讓使用者可以用更直覺的旗標方式觸發更新。

> **引用**：`source-repo/src/cli/run-main.ts:51-60`

### 2.4 `resolveMissingPluginCommandMessage`：友善錯誤訊息

當使用者輸入的命令找不到對應的插件時，此函式（第 74-141 行）會提供有幫助的錯誤訊息。它會區分三種情況：

| 情境 | 錯誤訊息範例 |
|------|-------------|
| 命令是插件提供的，但該插件不在 `plugins.allow` 列表中 | `"xxx" is not a plugin; it is a command provided by the "yyy" plugin. Add "yyy" to plugins.allow.` |
| 插件被明確停用 | `The openclaw xxx command is unavailable because plugins.entries.yyy.enabled=false.` |
| 命令是執行時斜線命令（runtime slash command），不是 CLI 命令 | `"xxx" is a runtime slash command (/xxx), not a CLI command.` |

> **引用**：`source-repo/src/cli/run-main.ts:74-141`

---

## 3. CLI Route 系統：快速路徑路由

### 3.1 `tryRouteCli`：繞過 Commander.js 的捷徑

Route 系統是 OpenClaw CLI 的效能最佳化關鍵。某些常用命令（如 `health`、`status`、`config get`）被預先註冊為「可路由命令」（routed commands），能在不建構完整 Commander.js 程式的情況下直接執行。

```typescript
export async function tryRouteCli(argv: string[]): Promise<boolean> {
  if (isTruthyEnvValue(process.env.OPENCLAW_DISABLE_ROUTE_FIRST)) {
    return false;
  }
  const invocation = resolveCliArgvInvocation(argv);
  if (invocation.hasHelpOrVersion) {
    return false;
  }
  if (!invocation.commandPath[0]) {
    return false;
  }
  const route = findRoutedCommand(invocation.commandPath);
  if (!route) {
    return false;
  }
  await prepareRoutedCommand({
    argv,
    commandPath: invocation.commandPath,
    loadPlugins: route.loadPlugins,
  });
  return route.run(argv);
}
```

> **引用**：`source-repo/src/cli/route.ts:41-62`

### 3.2 路由流程

Route 系統的運作分為四個階段：

1. **環境檢查**：如果設定了 `OPENCLAW_DISABLE_ROUTE_FIRST=1`，整個快速路徑會被停用
2. **調用解析**：透過 `resolveCliArgvInvocation` 解析命令路徑（command path），如果帶有 `--help` 或 `--version` 旗標則跳過路由（讓 Commander.js 處理 help 輸出）
3. **路由查找**：`findRoutedCommand` 在預先註冊的路由表中查找匹配的命令
4. **命令準備**：`prepareRoutedCommand` 處理啟動策略（startup policy）、banner 顯示和必要的引導程序（bootstrap）

### 3.3 `prepareRoutedCommand`：路由命令的準備工作

```typescript
async function prepareRoutedCommand(params: {
  argv: string[];
  commandPath: string[];
  loadPlugins?: boolean | ((argv: string[]) => boolean);
}) {
  const { startupPolicy } = resolveCliExecutionStartupContext({
    argv: params.argv,
    jsonOutputMode: hasFlag(params.argv, "--json"),
    env: process.env,
    routeMode: true,
  });
  const { VERSION } = await import("../version.js");
  await applyCliExecutionStartupPresentation({
    argv: params.argv,
    routeLogsToStderrOnSuppress: false,
    startupPolicy,
    showBanner: process.stdout.isTTY && !startupPolicy.suppressDoctorStdout,
    version: VERSION,
  });
  // ... ensureCliExecutionBootstrap
}
```

此函式確保路由命令在執行前：
- 決定是否顯示 banner（TTY 且非 JSON 輸出模式時才顯示）
- 執行必要的啟動引導（如載入配置、初始化插件等）

> **引用**：`source-repo/src/cli/route.ts:12-39`

### 3.4 可路由命令清單

在 `command-catalog.ts` 中定義了哪些命令支援快速路由：

| 命令路徑 | 路由 ID | 備註 |
|---------|---------|------|
| `health` | `health` | 預載插件（text-only 模式） |
| `status` | `status` | 預載插件（text-only 模式） |
| `gateway status` | `gateway-status` | 總是啟用 config guard |
| `sessions` | `sessions` | 不確保 CLI path |
| `agents list` | `agents-list` | — |
| `config get` | `config-get` | 不確保 CLI path |
| `config unset` | `config-unset` | 不確保 CLI path |
| `models list` | `models-list` | 不確保 CLI path |
| `models status` | `models-status` | 不確保 CLI path |

> **引用**：`source-repo/src/cli/command-catalog.ts:32-129`

---

## 4. 子命令完整清單

OpenClaw CLI 提供了豐富的子命令集，每個子命令對應一個 `*-cli.ts` 檔案。以下是完整清單：

### 4.1 核心管理命令

| 命令 | 對應檔案 | 說明 |
|------|---------|------|
| `config` | `config-cli.ts` | 配置管理（設定、取得、刪除配置值） |
| `security` | `security-cli.ts` | 安全稽核與修復 |
| `system` | `system-cli.ts` | 系統工具（事件、心跳、存在感） |
| `update` | `update-cli.ts` | CLI 自我更新 |
| `logs` | `logs-cli.ts` | 日誌管理 |

### 4.2 通道與通訊命令

| 命令 | 對應檔案 | 說明 |
|------|---------|------|
| `channels` | `channels-cli.ts` | 通道管理（Discord、WhatsApp 等） |
| `webhooks` | `webhooks-cli.ts` | Webhook 管理 |
| `hooks` | `hooks-cli.ts` | 勾點（hook）管理 |
| `send` | `send-runtime/`（目錄） | 對外訊息發送 |

### 4.3 插件與技能命令

| 命令 | 對應檔案 | 說明 |
|------|---------|------|
| `plugins` | `plugins-cli.ts` | 插件管理（安裝、移除、更新、列表） |
| `skills` | `skills-cli.ts` | 技能（Skill）管理 |

### 4.4 安全與權限命令

| 命令 | 對應檔案 | 說明 |
|------|---------|------|
| `secrets` | `secrets-cli.ts` | 秘密管理（Secret management） |
| `exec-approvals` | `exec-approvals-cli.ts` | 執行核准管理 |
| `exec-policy` | `exec-policy-cli.ts` | 執行策略管理 |

### 4.5 整合與協定命令

| 命令 | 對應檔案 | 說明 |
|------|---------|------|
| `acp` | `acp-cli.ts` | ACP 橋接（Agent Client Protocol） |
| `mcp` | `mcp-cli.ts` | MCP 伺服器管理（Model Context Protocol） |
| `models` | `models-cli.ts` | 模型管理 |

### 4.6 裝置與配對命令

| 命令 | 對應檔案 | 說明 |
|------|---------|------|
| `devices` | `devices-cli.ts` | 裝置管理 |
| `pairing` | `pairing-cli.ts` | 裝置配對 |
| `qr` | `qr-cli.ts` | QR code 顯示 |

### 4.7 基礎設施命令

| 命令 | 對應檔案 | 說明 |
|------|---------|------|
| `sandbox` | `sandbox-cli.ts` | 沙箱管理 |
| `proxy` | `proxy-cli.ts` | 代理管理 |
| `dns` | `dns-cli.ts` | DNS 管理 |
| `capability` | `capability-cli.ts` | 能力回報（capability reporting） |
| `completion` | `completion-cli.ts` | Shell 自動補全（bash, zsh, fish） |

### 4.8 UI 與資訊命令

| 命令 | 對應檔案 | 說明 |
|------|---------|------|
| `tui` | `tui-cli.ts` | 終端機使用者介面（Terminal UI） |
| `docs` | `docs-cli.ts` | 文件連結 |
| `clawbot` | `clawbot-cli.ts` | ClawBot 管理 |
| `directory` | `directory-cli.ts` | 目錄服務 |

### 4.9 排程命令

| 命令 | 對應目錄 | 說明 |
|------|---------|------|
| `cron` | `cron-cli/`（目錄） | 排程任務管理（cron-add、cron-edit、cron-simple） |

> **引用**：`source-repo/src/cli/` 目錄中所有 `*-cli.ts` 檔案。

---

## 5. 命令解析模式

### 5.1 argv 解析（`argv.ts`）

`argv.ts` 提供了底層的參數解析工具，是整個 CLI 命令辨識的基礎。

#### 旗標常數

```typescript
const HELP_FLAGS = new Set(["-h", "--help"]);
const VERSION_FLAGS = new Set(["-V", "--version"]);
const ROOT_VERSION_ALIAS_FLAG = "-v";
```

OpenClaw 使用大寫 `-V` 作為版本旗標，而小寫 `-v` 只在**根層級**（沒有子命令時）作為版本別名。這是為了避免與子命令的 `-v`（verbose）旗標衝突。

> **引用**：`source-repo/src/cli/argv.ts:8-10`

#### 核心解析函式

| 函式 | 說明 |
|------|------|
| `hasHelpOrVersion(argv)` | 檢查 argv 中是否存在 help 或 version 旗標 |
| `hasFlag(argv, name)` | 檢查特定旗標是否存在（在 `--` 終止符之前） |
| `hasRootVersionAlias(argv)` | 檢查 `-v` 別名——只有在沒有子命令時才有效 |
| `getCommandPath(argv, depth)` | 從 argv 中提取命令路徑（command path） |
| `getPrimaryCommand(argv)` | 取得主要命令名稱 |
| `getFlagValue(argv, name)` | 取得旗標的值（支援 `--flag value` 和 `--flag=value` 兩種寫法） |
| `getVerboseFlag(argv)` | 檢查 `--verbose`（選擇性也包含 `--debug`） |

> **引用**：`source-repo/src/cli/argv.ts:12-135`

#### `hasFlag` 的 `--` 終止符處理

```typescript
export function hasFlag(argv: string[], name: string): boolean {
  const args = argv.slice(2);
  for (const arg of args) {
    if (arg === FLAG_TERMINATOR) {
      break;
    }
    if (arg === name) {
      return true;
    }
  }
  return false;
}
```

此函式嚴格遵守 POSIX 的 `--` 終止符慣例：`--` 之後的所有參數都被視為位置參數（positional arguments），不再解析為旗標。

> **引用**：`source-repo/src/cli/argv.ts:26-37`

#### `hasRootVersionAlias`：根層級 `-v` 的特殊邏輯

```typescript
export function hasRootVersionAlias(argv: string[]): boolean {
  const args = argv.slice(2);
  let hasAlias = false;
  for (let i = 0; i < args.length; i += 1) {
    const arg = args[i];
    // ...
    if (arg === ROOT_VERSION_ALIAS_FLAG) {
      hasAlias = true;
      continue;
    }
    // ... 跳過已知的根層級選項
    if (arg.startsWith("-")) {
      continue;
    }
    return false; // 遇到子命令名稱就停止，-v 不再有效
  }
  return hasAlias;
}
```

這段邏輯確保 `openclaw -v` 會顯示版本，但 `openclaw config -v` 不會——因為 `-v` 在子命令上下文中可能代表 `--verbose`。

> **引用**：`source-repo/src/cli/argv.ts:39-65`

### 5.2 CLI 調用解析（`argv-invocation.ts`）

`resolveCliArgvInvocation` 是一個整合函式，一次性解析出所有需要的調用資訊：

```typescript
export type CliArgvInvocation = {
  argv: string[];
  commandPath: string[];         // 例如 ["config", "set"]
  primary: string | null;        // 例如 "config"
  hasHelpOrVersion: boolean;
  isRootHelpInvocation: boolean;
};

export function resolveCliArgvInvocation(argv: string[]): CliArgvInvocation {
  return {
    argv,
    commandPath: getCommandPathWithRootOptions(argv, 2),
    primary: getPrimaryCommand(argv),
    hasHelpOrVersion: hasHelpOrVersion(argv),
    isRootHelpInvocation: isRootHelpInvocation(argv),
  };
}
```

這個型別在整個 CLI 系統中被廣泛使用，是命令路由的基礎數據結構。

> **引用**：`source-repo/src/cli/argv-invocation.ts:1-24`

### 5.3 CLI 名稱解析（`cli-name.ts`）

```typescript
export const DEFAULT_CLI_NAME = "openclaw";

const CLI_PREFIX_RE = /^(?:((?:pnpm|npm|bunx|npx)\s+))?(openclaw)\b/;

export function resolveCliName(argv: string[] = process.argv): string {
  const argv1 = argv[1];
  if (!argv1) {
    return DEFAULT_CLI_NAME;
  }
  const base = path.basename(argv1).trim();
  if (KNOWN_CLI_NAMES.has(base)) {
    return base;
  }
  return DEFAULT_CLI_NAME;
}
```

`replaceCliName` 函式能智慧地替換命令字串中的 CLI 名稱，支援 `pnpm`、`npm`、`bunx`、`npx` 等包管理器前綴：

```typescript
export function replaceCliName(command: string, cliName = resolveCliName()): string {
  if (!CLI_PREFIX_RE.test(command)) {
    return command;
  }
  return command.replace(CLI_PREFIX_RE, (_match, runner: string | undefined) => {
    return `${runner ?? ""}${cliName}`;
  });
}
```

> **引用**：`source-repo/src/cli/cli-name.ts:1-31`

### 5.4 Windows argv 正規化（`windows-argv.ts`）

Windows 平台下的 `process.argv` 可能包含各種異常情況：

- 控制字元（control characters，ASCII < 32 或 127）
- 多餘的引號包裹
- 重複的 `node.exe` 路徑（Windows 的 shim 機制可能插入額外的路徑）
- UNC 路徑前綴（`\\?\`）

`normalizeWindowsArgv` 會：
1. 檢測平台是否為 `win32`，非 Windows 直接返回原始 argv
2. 移除控制字元和多餘引號
3. 辨識並刪除重複的 `node.exe` 路徑條目

```typescript
export function normalizeWindowsArgv(argv: string[]): string[] {
  if (process.platform !== "win32") {
    return argv;
  }
  // ... 移除控制字元、引號、重複的 node.exe 路徑
}
```

> **引用**：`source-repo/src/cli/windows-argv.ts:1-79`

---

## 6. 與 Gateway 的互動

OpenClaw 的架構是 CLI + Gateway（閘道）分離的。CLI 本身不執行 AI 推理，而是將命令轉發給 Gateway 處理。這個通訊是透過 **WebSocket RPC** 完成的。

### 6.1 Gateway RPC 客戶端

`gateway-rpc.ts` 提供了兩個關鍵函式：

```typescript
export function addGatewayClientOptions(cmd: Command) {
  return cmd
    .option("--url <url>", "Gateway WebSocket URL (defaults to gateway.remote.url when configured)")
    .option("--token <token>", "Gateway token (if required)")
    .option("--timeout <ms>", "Timeout in ms", "30000")
    .option("--expect-final", "Wait for final response (agent)", false);
}

export async function callGatewayFromCli(
  method: string,
  opts: GatewayRpcOpts,
  params?: unknown,
  extra?: { expectFinal?: boolean; progress?: boolean },
) {
  const runtime = await loadGatewayRpcRuntime();
  return await runtime.callGatewayFromCliRuntime(method, opts, params, extra);
}
```

`addGatewayClientOptions` 為任何需要與 Gateway 通訊的命令添加標準選項：URL、token、timeout 和 expect-final 旗標。`callGatewayFromCli` 則使用延遲載入（lazy loading）來避免在不需要 Gateway 通訊時載入相關模組。

> **引用**：`source-repo/src/cli/gateway-rpc.ts:1-31`

### 6.2 System CLI 的 Gateway 互動範例

`system-cli.ts` 是 Gateway RPC 的典型使用範例。它註冊了以下子命令，每個都透過 `callGatewayFromCli` 與 Gateway 通訊：

```typescript
export function registerSystemCli(program: Command) {
  const system = program
    .command("system")
    .description("System tools (events, heartbeat, presence)");

  // system event --text <text> --mode (now|next-heartbeat)
  addGatewayClientOptions(
    system.command("event")
      .requiredOption("--text <text>", "System event text")
      .option("--mode <mode>", "Wake mode (now|next-heartbeat)", "next-heartbeat")
  ).action(async (opts) => {
    // ...
    return await callGatewayFromCli("wake", opts, { mode, text }, { expectFinal: false });
  });

  // system heartbeat last/enable/disable
  // system presence
}
```

Gateway RPC 方法對照：

| CLI 命令 | Gateway RPC 方法 | 說明 |
|---------|-----------------|------|
| `system event` | `wake` | 發送系統事件，可選擇立即或下次心跳時觸發 |
| `system heartbeat last` | `last-heartbeat` | 查詢最近一次心跳 |
| `system heartbeat enable` | `set-heartbeats` | 啟用心跳（`enabled: true`） |
| `system heartbeat disable` | `set-heartbeats` | 停用心跳（`enabled: false`） |
| `system presence` | `system-presence` | 列出系統存在感條目 |

> **引用**：`source-repo/src/cli/system-cli.ts:42-133`

---

## 7. ACP 模式：IDE 整合橋接

### 7.1 什麼是 ACP？

ACP（Agent Client Protocol）是一個用於 IDE 與 AI 代理（agent）溝通的協定。OpenClaw 的 `openclaw acp` 命令提供了一個 ACP 橋接器，讓支援 ACP 的 IDE（如 Zed 編輯器）能透過 stdio 與 OpenClaw Gateway 互動。

### 7.2 架構概覽

```
┌──────────────┐    stdio/NDJSON    ┌─────────────────┐    WebSocket    ┌─────────────┐
│  IDE (Zed)   │ ◄────────────────► │  openclaw acp   │ ◄────────────► │   Gateway   │
│  ACP Client  │                    │  ACP Bridge     │                │             │
└──────────────┘                    └─────────────────┘                └─────────────┘
```

關鍵設計目標：
- **最小化 ACP 介面**：僅使用 stdio + NDJSON
- **穩定的 Session 映射**：ACP session 與 Gateway session 之間可靠對應
- **安全預設值**：預設使用隔離的 `acp:<uuid>` session key

### 7.3 命令選項

```typescript
export function registerAcpCli(program: Command) {
  const acp = program.command("acp").description("Run an ACP bridge backed by the Gateway");

  acp
    .option("--url <url>", "Gateway WebSocket URL")
    .option("--token <token>", "Gateway token (if required)")
    .option("--token-file <path>", "Read gateway token from file")
    .option("--password <password>", "Gateway password (if required)")
    .option("--password-file <path>", "Read gateway password from file")
    .option("--session <key>", "Default session key (e.g. agent:main:main)")
    .option("--session-label <label>", "Default session label to resolve")
    .option("--require-existing", "Fail if the session key/label does not exist", false)
    .option("--reset-session", "Reset the session key before first use", false)
    .option("--no-prefix-cwd", "Do not prefix prompts with the working directory", false)
    .option("--provenance <mode>", "ACP provenance mode: off, meta, or meta+receipt")
    .option("-v, --verbose", "Verbose logging to stderr", false)
    .action(async (opts) => {
      // ... serveAcpGateway(...)
    });
}
```

> **引用**：`source-repo/src/cli/acp-cli.ts:11-54`

### 7.4 Session 映射

ACP 的 session 管理方式如下：

| 場景 | Session Key |
|------|-------------|
| 預設（無指定） | `acp:<uuid>`（每次連線產生新的隔離 session） |
| 指定 `--session` | 使用者自訂的 key，例如 `agent:main:main` |
| 指定 `--session-label` | 解析 label 對應的 session key |

使用者可透過 `--reset-session` 在首次使用前重置 session，或透過 `--require-existing` 要求 session 必須已存在。

> **引用**：`source-repo/docs.acp.md:1-50`

### 7.5 ACP 協定轉換

| ACP 概念 | Gateway 對應 |
|----------|-------------|
| ACP `prompt` | Gateway `chat.send` |
| ACP `cancel` | Gateway `abort` |
| ACP `listSessions` | Gateway `sessions.list` |
| ACP streaming 訊息 | Gateway 串流更新（message/tool_call） |
| ACP stop reason `complete` | Gateway stop reason `stop` |
| ACP stop reason `aborted` | Gateway stop reason `cancel` |
| ACP stop reason `error` | Gateway stop reason `error` |

> **引用**：`source-repo/docs.acp.md:1-100`

### 7.6 ACP 互動式測試客戶端

`openclaw acp client` 子命令提供了一個互動式測試客戶端：

```typescript
acp
  .command("client")
  .description("Run an interactive ACP client against the local ACP bridge")
  .option("--cwd <dir>", "Working directory for the ACP session")
  .option("--server <command>", "ACP server command (default: openclaw)")
  .option("--server-args <args...>", "Extra arguments for the ACP server")
  .option("--server-verbose", "Enable verbose logging on the ACP server", false)
  .option("-v, --verbose", "Verbose client logging", false)
  .action(async (opts, command) => {
    await runAcpClientInteractive({...});
  });
```

> **引用**：`source-repo/src/cli/acp-cli.ts:56-79`

### 7.7 Zed 編輯器整合設定

在 `~/.config/zed/settings.json` 中添加以下配置即可在 Zed 中使用 OpenClaw ACP：

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": ["acp"],
      "env": {}
    }
  }
}
```

若需要指定 Gateway 或 agent：

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": [
        "acp",
        "--url", "wss://gateway-host:18789",
        "--token", "<token>",
        "--session", "agent:design:main"
      ],
      "env": {}
    }
  }
}
```

> **引用**：`source-repo/docs.acp.md:107-145`

---

## 8. Daemon 相關命令

### 8.1 System CLI

`system-cli.ts` 提供了系統層級的管理功能：

**事件管理**：
```bash
openclaw system event --text "部署完成" --mode now
openclaw system event --text "排程檢查" --mode next-heartbeat
```

`--mode` 參數控制事件的觸發時機：
- `now`：立即觸發
- `next-heartbeat`：延遲到下一次心跳時觸發（預設值）

**心跳管理**：
```bash
openclaw system heartbeat last       # 查看最後一次心跳
openclaw system heartbeat enable     # 啟用心跳
openclaw system heartbeat disable    # 停用心跳
```

**存在感管理**：
```bash
openclaw system presence             # 列出系統存在感條目
```

> **引用**：`source-repo/src/cli/system-cli.ts:42-133`

### 8.2 更新與裝置管理

| 命令 | 說明 |
|------|------|
| `openclaw update` | CLI 自我更新 |
| `openclaw devices` | 裝置管理 |
| `openclaw pairing` | 裝置配對 |

需注意：在容器模式下，`openclaw update` 是被阻止的（blocked）。容器內的 CLI 應透過重建映像檔（rebuild image）來更新：

```typescript
function isBlockedContainerCommand(argv: string[]): boolean {
  if (resolveCliArgvInvocation(["node", "openclaw", ...argv]).primary === "update") {
    return true;
  }
  // ...
}
```

> **引用**：`source-repo/src/cli/container-target.ts:186-208`

---

## 9. 互動式 CLI 功能

### 9.1 Y/N 提示（`prompt.ts`）

```typescript
export async function promptYesNo(question: string, defaultYes = false): Promise<boolean> {
  if (isVerbose() && isYes()) {
    return true;
  }
  if (isYes()) {
    return true;
  }
  const rl = readline.createInterface({ input, output });
  const suffix = defaultYes ? " [Y/n] " : " [y/N] ";
  const answer = normalizeLowercaseStringOrEmpty(await rl.question(`${question}${suffix}`));
  rl.close();
  if (!answer) {
    return defaultYes;
  }
  return answer.startsWith("y");
}
```

核心行為：
- 如果全域 `--yes` 旗標被設定（`isYes()` 為 `true`），直接返回 `true`，不顯示提示——這是**非互動模式**（non-interactive mode）的關鍵機制
- 使用 Node.js 的 `readline` 模組讀取使用者輸入
- 支援 `defaultYes` 參數控制預設值：`[Y/n]` 表示預設 Yes，`[y/N]` 表示預設 No
- 空白輸入（直接按 Enter）採用預設值

> **引用**：`source-repo/src/cli/prompt.ts:1-22`

### 9.2 Terminal UI（TUI）

`tui-cli.ts` 註冊了 `openclaw tui` 命令，提供終端機使用者介面。這是一個獨立的互動式介面，提供比純 CLI 更豐富的操作體驗。

---

## 10. Config CLI：配置管理

### 10.1 命令結構

`config-cli.ts` 是整個 CLI 中最複雜的子命令之一（檔案大小約 47KB），提供了三種配置設定模式：

| 模式 | 說明 | 範例 |
|------|------|------|
| **Value 模式** | 直接設定值 | `openclaw config set gateway.port 19001 --strict-json` |
| **Ref/Provider Builder 模式** | 設定秘密參考 | `openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN` |
| **Provider 模式** | 設定秘密提供者 | `openclaw config set secrets.providers.vault --provider-source file --provider-path /etc/openclaw/secrets.json` |
| **Batch JSON 模式** | 批量設定 | `openclaw config set --batch-file ./config-set.batch.json --dry-run` |

> **引用**：`source-repo/src/cli/config-cli.ts:86-96`

### 10.2 路徑解析

Config CLI 使用點分隔（dot-separated）路徑來定位配置值。路徑解析器支援：

- **點分隔**：`gateway.port` → `["gateway", "port"]`
- **方括號語法**：`channels[discord].token` → `["channels", "discord", "token"]`
- **反斜線跳脫**：`path\.with\.dots` → `["path.with.dots"]`

```typescript
function parsePath(raw: string): PathSegment[] {
  const trimmed = raw.trim();
  // 支援 '.'、'[...]'、'\.' 語法
  // ...
}
```

> **引用**：`source-repo/src/cli/config-cli.ts:110-159`

### 10.3 秘密參考（Secret Reference）設定

Config CLI 的一個重要功能是設定秘密參考。這允許配置值指向環境變數或外部秘密管理系統，而不是直接嵌入明文密碼：

```bash
# 讓 Discord token 從環境變數讀取
openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN

# 設定外部秘密提供者
openclaw config set secrets.providers.vault \
  --provider-source file \
  --provider-path /etc/openclaw/secrets.json \
  --provider-mode json
```

> **引用**：`source-repo/src/cli/config-cli.ts:77-84`

---

## 11. Security CLI：安全稽核

### 11.1 命令選項

```typescript
export function registerSecurityCli(program: Command) {
  const security = program
    .command("security")
    .description("Audit local config and state for common security foot-guns");

  security
    .command("audit")
    .option("--deep", "Attempt live Gateway probe (best-effort)", false)
    .option("--token <token>", "Use explicit gateway token for deep probe auth")
    .option("--password <password>", "Use explicit gateway password for deep probe auth")
    .option("--fix", "Apply safe fixes (tighten defaults + chmod state/config)", false)
    .option("--json", "Print JSON", false)
    .action(async (opts) => {
      // ...
    });
}
```

> **引用**：`source-repo/src/cli/security-cli.ts:35-63`

### 11.2 稽核模式

| 選項 | 說明 |
|------|------|
| 無選項（預設） | 本地安全稽核：檢查配置和狀態目錄 |
| `--deep` | 加上即時 Gateway 探測檢查（best-effort） |
| `--deep --token <token>` | 使用明確 token 進行 deep probe |
| `--deep --password <password>` | 使用明確密碼進行 deep probe |
| `--fix` | 套用安全修復（收緊預設值 + 修正檔案權限） |
| `--json` | 輸出機器可讀的 JSON 格式 |

### 11.3 稽核流程

```typescript
.action(async (opts: SecurityAuditOptions) => {
  // 1. 如果指定了 --fix，先套用安全修復
  const fixResult = opts.fix ? await fixSecurityFootguns().catch((_err) => null) : null;

  // 2. 載入配置
  const sourceConfig = loadConfig();

  // 3. 解析秘密參考
  const { resolvedConfig: cfg, diagnostics: secretDiagnostics } =
    await resolveCommandSecretRefsViaGateway({...});

  // 4. 執行安全稽核
  const report = await runSecurityAudit({
    config: cfg,
    sourceConfig,
    deep: Boolean(opts.deep),
    includeFilesystem: true,
    includeChannelSecurity: true,
    deepProbeAuth: token || password ? {...} : undefined,
  });

  // 5. 輸出報告（JSON 或人類可讀格式）
});
```

報告摘要使用三個嚴重等級分類：

| 等級 | 標記 | 說明 |
|------|------|------|
| Critical | `CRITICAL` | 嚴重安全問題，必須立即處理 |
| Warn | `WARN` | 潛在風險，建議修復 |
| Info | `INFO` | 資訊性發現，供參考 |

> **引用**：`source-repo/src/cli/security-cli.ts:63-194`

---

## 12. 命令註冊與啟動策略

### 12.1 命令註冊策略（`command-registration-policy.ts`）

OpenClaw CLI 使用延遲載入（lazy loading）策略來減少啟動時間。並非所有命令都在啟動時完整註冊。

```typescript
export function shouldRegisterPrimaryCommandOnly(argv: string[]): boolean {
  const invocation = resolveCliArgvInvocation(argv);
  return invocation.primary !== null || !invocation.hasHelpOrVersion;
}

export function shouldSkipPluginCommandRegistration(params: {
  argv: string[];
  primary: string | null;
  hasBuiltinPrimary: boolean;
}): boolean {
  if (params.hasBuiltinPrimary) {
    return true;  // 內建命令存在時跳過插件命令註冊
  }
  if (!params.primary) {
    return resolveCliArgvInvocation(params.argv).hasHelpOrVersion;
  }
  return false;
}
```

邏輯解說：
- 如果指定了明確的子命令（primary command），只註冊該命令即可
- 如果主要命令是內建的，則跳過插件命令的註冊
- 可透過環境變數 `OPENCLAW_DISABLE_LAZY_SUBCOMMANDS` 強制完整註冊

> **引用**：`source-repo/src/cli/command-registration-policy.ts:1-33`

### 12.2 啟動策略（`command-startup-policy.ts`）

每個命令路徑都有對應的啟動策略，控制以下行為：

| 策略屬性 | 說明 |
|---------|------|
| `bypassConfigGuard` | 是否繞過配置檢查（例如 `backup`、`doctor`、`secrets`） |
| `routeConfigGuard` | 路由模式下的配置檢查策略：`never`、`always`、`when-suppressed` |
| `loadPlugins` | 插件載入策略：`never`、`always`、`text-only` |
| `hideBanner` | 是否隱藏啟動橫幅（例如 `completion`、`update`） |
| `ensureCliPath` | 是否確保 CLI 可執行檔在 PATH 中 |

這些策略也可透過環境變數 `OPENCLAW_HIDE_BANNER` 全域控制 banner 的顯示。

> **引用**：`source-repo/src/cli/command-startup-policy.ts:1-62`

---

## 13. Profile 系統

### 13.1 Profile 概念

Profile（配置檔）允許在同一台機器上運行多個隔離的 OpenClaw 實例。每個 profile 擁有獨立的：
- 狀態目錄（state directory）
- 配置檔案路徑
- Gateway 連接埠（`dev` profile 預設使用 19001）

### 13.2 Profile 解析

```typescript
export function parseCliProfileArgs(argv: string[]): CliProfileParseResult {
  // 掃描 argv，提取 --profile 或 --dev 旗標
  // --dev 等同於 --profile dev
  // --dev 和 --profile 不能同時使用
}

export function applyCliProfileEnv(params: { profile: string }) {
  const env = process.env;
  env.OPENCLAW_PROFILE = profile;

  // 設定 state dir：~/.openclaw-{profile}
  const stateDir = resolveProfileStateDir(profile, env, homedir);
  env.OPENCLAW_STATE_DIR = stateDir;
  env.OPENCLAW_CONFIG_PATH = path.join(stateDir, "openclaw.json");

  // dev profile 的特殊處理
  if (profile === "dev" && !env.OPENCLAW_GATEWAY_PORT?.trim()) {
    env.OPENCLAW_GATEWAY_PORT = "19001";
  }
}
```

Profile 的狀態目錄命名規則：
- `default` profile → `~/.openclaw`
- 其他 profile → `~/.openclaw-{name}`（例如 `~/.openclaw-dev`）

> **引用**：`source-repo/src/cli/profile.ts:18-122`

---

## 14. 容器目標定位

### 14.1 概念

容器目標定位（Container Targeting）允許 CLI 將命令轉發到在 Docker 或 Podman 容器內運行的 OpenClaw 實例。

### 14.2 使用方式

```bash
# 透過旗標指定
openclaw --container my-openclaw-instance status

# 透過環境變數指定
OPENCLAW_CONTAINER=my-openclaw-instance openclaw status
```

### 14.3 執行流程

1. 解析 `--container` 旗標或 `OPENCLAW_CONTAINER` 環境變數
2. 嘗試 Podman 和 Docker 兩種容器執行時（先 Podman 後 Docker）
3. 確認容器正在運行（透過 `inspect --format {{.State.Running}}`）
4. 使用 `docker exec` 或 `podman exec` 在容器內執行 `openclaw` 命令
5. 自動傳遞 TTY 設定（`-i` 和可選的 `-t`）
6. 設定 `OPENCLAW_CLI_CONTAINER_BYPASS=1` 防止遞迴

```typescript
function buildContainerExecArgs(params: {...}): string[] {
  const envFlag = params.exec.runtime === "docker" ? "-e" : "--env";
  return [
    ...params.exec.argsPrefix,
    "exec",
    ...interactiveFlags,
    envFlag, `OPENCLAW_CONTAINER_HINT=${params.containerName}`,
    envFlag, "OPENCLAW_CLI_CONTAINER_BYPASS=1",
    params.containerName,
    "openclaw",
    ...params.argv,
  ];
}
```

> **引用**：`source-repo/src/cli/container-target.ts:1-267`

---

## 15. Banner 系統

### 15.1 啟動橫幅

OpenClaw CLI 在啟動時會顯示一個包含龍蝦（lobster 🦞）圖案的橫幅。橫幅僅在以下條件下顯示：

- 輸出目標是 TTY（終端機）
- 沒有 `--json` 旗標
- 沒有 `--version` 或 `-V` 旗標
- 命令的啟動策略沒有設定 `hideBanner`
- 沒有設定 `OPENCLAW_HIDE_BANNER` 環境變數

```typescript
export function emitCliBanner(version: string, options: BannerOptions = {}) {
  if (bannerEmitted) { return; }
  const argv = options.argv ?? process.argv;
  if (!process.stdout.isTTY) { return; }
  if (hasJsonFlag(argv)) { return; }
  if (hasVersionFlag(argv)) { return; }
  const line = formatCliBannerLine(version, options);
  process.stdout.write(`\n${line}\n\n`);
  bannerEmitted = true;
}
```

橫幅格式：`🦞 OpenClaw {version} ({commit}) — {tagline}`

在支援色彩的終端中（rich TTY），橫幅會使用主題色彩渲染。

> **引用**：`source-repo/src/cli/banner.ts:1-154`

### 15.2 ASCII Art

Banner 系統還包含一個完整的 ASCII art 龍蝦圖案：

```
▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
██░▄▄▄░██░▄▄░██░▄▄▄██░▀██░██░▄▄▀██░████░▄▄▀██░███░██
██░███░██░▀▀░██░▄▄▄██░█░█░██░█████░████░▀▀░██░█░█░██
██░▀▀▀░██░█████░▀▀▀██░██▄░██░▀▀▄██░▀▀░█░██░██▄▀▄▀▄██
▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀
                  🦞 OPENCLAW 🦞
```

> **引用**：`source-repo/src/cli/banner.ts:88-96`

---

## 16. 命令格式化（`command-format.ts`）

`formatCliCommand` 函式能智慧地為命令字串補上必要的旗標，確保使用者看到的提示命令在容器或 profile 環境下仍然正確：

```typescript
export function formatCliCommand(
  command: string,
  env: Record<string, string | undefined> = process.env,
): string {
  const cliName = resolveCliName();
  const normalizedCommand = replaceCliName(command, cliName);
  const container = env.OPENCLAW_CONTAINER_HINT?.trim();
  const profile = normalizeProfileName(env.OPENCLAW_PROFILE);
  // 如果在容器環境中，自動加上 --container 旗標
  // 如果使用了 profile，自動加上 --profile 旗標
  // update 命令不加 --container（容器內不支援 update）
}
```

例如，在名為 `prod` 的容器中，`openclaw security audit --deep` 會自動被格式化為 `openclaw --container prod security audit --deep`。

> **引用**：`source-repo/src/cli/command-format.ts:1-47`

---

## 本章小結

OpenClaw 的 CLI 系統展現了一個成熟的命令列工具架構。其核心設計特色包括：

1. **效能最佳化**：透過 Route 快速路徑和延遲載入策略，避免每次執行都建構完整的 Commander.js 程式
2. **跨平台相容性**：Windows argv 正規化、容器執行時支援（Docker/Podman）
3. **靈活的配置**：Profile 系統允許多實例隔離，Container Targeting 支援容器化部署
4. **安全意識**：內建安全稽核工具，秘密參考機制避免明文密碼
5. **IDE 整合**：ACP 橋接讓 OpenClaw 能無縫接入支援 ACP 的 IDE
6. **友善的錯誤處理**：缺少插件命令時提供具體的修復建議

整個 CLI 系統的約 218 個檔案構成了一個完整的命令生態系統，從底層的 argv 解析到高層的 ACP 橋接，每一層都有清晰的職責劃分。

---

## 引用來源

| 編號 | 來源檔案 | 行數範圍 | 說明 |
|------|---------|---------|------|
| 1 | `source-repo/src/cli/run-main.ts` | 1-292 | CLI 主入口與命令分發器完整實作 |
| 2 | `source-repo/src/cli/run-main.ts` | 150-287 | `runCli()` 主函式 |
| 3 | `source-repo/src/cli/run-main.ts` | 51-60 | `rewriteUpdateFlagArgv` 旗標重寫 |
| 4 | `source-repo/src/cli/run-main.ts` | 74-141 | `resolveMissingPluginCommandMessage` 錯誤訊息 |
| 5 | `source-repo/src/cli/run-main.ts` | 143-148 | `shouldLoadCliDotEnv` .env 載入判斷 |
| 6 | `source-repo/src/cli/route.ts` | 1-62 | 快速路徑路由系統 |
| 7 | `source-repo/src/cli/route.ts` | 41-62 | `tryRouteCli` 函式 |
| 8 | `source-repo/src/cli/route.ts` | 12-39 | `prepareRoutedCommand` 路由準備 |
| 9 | `source-repo/src/cli/argv.ts` | 1-329 | 參數解析工具完整實作 |
| 10 | `source-repo/src/cli/argv.ts` | 8-10 | 旗標常數定義 |
| 11 | `source-repo/src/cli/argv.ts` | 26-37 | `hasFlag` 函式 |
| 12 | `source-repo/src/cli/argv.ts` | 39-65 | `hasRootVersionAlias` 函式 |
| 13 | `source-repo/src/cli/argv-invocation.ts` | 1-24 | CLI 調用解析 |
| 14 | `source-repo/src/cli/cli-name.ts` | 1-31 | CLI 名稱解析 |
| 15 | `source-repo/src/cli/windows-argv.ts` | 1-79 | Windows argv 正規化 |
| 16 | `source-repo/src/cli/gateway-rpc.ts` | 1-31 | Gateway RPC 客戶端 |
| 17 | `source-repo/src/cli/system-cli.ts` | 42-133 | System CLI 子命令 |
| 18 | `source-repo/src/cli/acp-cli.ts` | 1-79 | ACP CLI 完整實作 |
| 19 | `source-repo/docs.acp.md` | 1-240 | ACP 橋接文件 |
| 20 | `source-repo/src/cli/prompt.ts` | 1-22 | 互動式 Y/N 提示 |
| 21 | `source-repo/src/cli/config-cli.ts` | 1-160 | Config CLI 設定管理 |
| 22 | `source-repo/src/cli/security-cli.ts` | 1-195 | Security CLI 安全稽核 |
| 23 | `source-repo/src/cli/command-catalog.ts` | 1-129 | 命令目錄定義 |
| 24 | `source-repo/src/cli/command-registration-policy.ts` | 1-33 | 命令註冊策略 |
| 25 | `source-repo/src/cli/command-startup-policy.ts` | 1-62 | 啟動策略 |
| 26 | `source-repo/src/cli/profile.ts` | 1-122 | Profile 管理 |
| 27 | `source-repo/src/cli/container-target.ts` | 1-267 | 容器目標定位 |
| 28 | `source-repo/src/cli/banner.ts` | 1-154 | 啟動橫幅系統 |
| 29 | `source-repo/src/cli/command-format.ts` | 1-47 | 命令格式化 |
| 30 | `source-repo/src/cli/dotenv.ts` | 1-17 | .env 檔案載入 |
