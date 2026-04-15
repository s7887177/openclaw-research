# 跨平台 Daemon 服務管理（Cross-Platform Daemon Service Management）

> **版本**：OpenClaw v2026.4.12
> **對應原始碼**：`src/daemon/`

## 摘要

OpenClaw Gateway 是一個需要持續運行的常駐服務（Daemon），負責接收來自各通道（Discord、WhatsApp 等）的訊息並交給 Agent 處理。但「持續運行」在不同作業系統上意味著截然不同的實作方式——macOS 使用 launchd、Linux 使用 systemd、Windows 使用 schtasks（排程工作）。`src/daemon/` 的核心使命就是將這些差異封裝在統一的抽象層之後，讓上層程式碼可以用一致的 API 來安裝、啟動、停止、重啟和監控 Gateway 服務，而不需要關心底層是哪個作業系統。

---

## 1. 架構概覽

### 1.1 GatewayService 型別 — 服務抽象介面

`src/daemon/` 的設計核心是 `GatewayService` 型別（定義於 `service.ts:66-78`）。它是一個統一的介面，描述了一個「Gateway 服務」必須具備的所有能力：

```typescript
// service.ts:66-78
interface GatewayService {
  label: string;              // 服務標籤（如 "ai.openclaw.gateway"）
  loadedText: string;         // 服務已載入時的顯示文字
  notLoadedText: string;      // 服務未載入時的顯示文字
  stage: string;              // 服務階段標識

  install(): Promise<void>;   // 安裝服務到作業系統
  uninstall(): Promise<void>; // 從作業系統移除服務
  stop(): Promise<void>;      // 停止運行中的服務
  restart(): Promise<void>;   // 重啟服務

  isLoaded(): Promise<boolean>;          // 檢查服務是否已載入
  readCommand(): Promise<string | null>; // 讀取服務配置的啟動命令
  readRuntime(): Promise<GatewayServiceRuntime | null>; // 讀取服務運行時狀態
}
```

每個屬性和方法都有明確的職責：

- **`label`**：服務在作業系統中的唯一標識符。macOS 使用反向域名格式（`ai.openclaw.gateway`），Linux 使用 kebab-case（`openclaw-gateway`），Windows 使用人類可讀名稱（`OpenClaw Gateway`）。
- **`loadedText` / `notLoadedText`**：供 CLI 的 `doctor` 命令使用，向使用者顯示服務狀態。
- **`install()` / `uninstall()`**：管理服務的安裝與移除。安裝意味著在作業系統中註冊服務（寫入 plist 檔案、建立 systemd unit、建立排程工作），移除則是反向操作。
- **`isLoaded()`**：檢查服務是否已被作業系統載入（不等於「正在運行」——服務可能已載入但因為錯誤而停止）。
- **`readRuntime()`**：讀取即時的服務運行狀態，包括 PID、退出狀態等。

### 1.2 GATEWAY_SERVICE_REGISTRY — 平台服務註冊表

`GATEWAY_SERVICE_REGISTRY`（`service.ts:172-212`）是連接抽象介面與具體實現的橋樑：

```typescript
// service.ts:172-212
const GATEWAY_SERVICE_REGISTRY: Record<string, () => GatewayService> = {
  darwin:  () => createLaunchAgentService(),   // macOS → LaunchAgent (launchd)
  linux:   () => createSystemdService(),       // Linux → systemd user service
  win32:   () => createSchtasksService(),      // Windows → Scheduled Task
};
```

這是一個標準的工廠模式（Factory Pattern）——每個平台對應一個工廠函式，回傳符合 `GatewayService` 介面的具體實現。使用惰性建構（Lazy Construction），只有在實際需要時才建立服務物件。

**平台字串的來源**：`darwin`、`linux`、`win32` 是 Node.js `process.platform` 的回傳值，這是 Node.js 生態系中識別作業系統的標準方式。

### 1.3 resolveGatewayService() — 平台分派

```typescript
// service.ts:220-225
function resolveGatewayService(): GatewayService | null {
  const factory = GATEWAY_SERVICE_REGISTRY[process.platform];
  return factory ? factory() : null;
}
```

這個函式是所有上層程式碼的入口——無論是 CLI 的 `install-daemon` 命令、`doctor` 健康檢查、還是 Gateway 啟動流程，都透過 `resolveGatewayService()` 取得當前平台的服務管理器。如果當前平台不在註冊表中（例如 FreeBSD），函式回傳 `null`，上層程式碼可以據此顯示「不支援的平台」訊息。

---

## 2. 服務狀態管理

### 2.1 GatewayServiceState — 服務狀態快照

`GatewayServiceState`（`service-types.ts:40-47`）代表服務在某一時刻的完整狀態快照：

```typescript
// service-types.ts:40-47
interface GatewayServiceState {
  installed: boolean;                        // 服務是否已安裝
  loaded: boolean;                           // 服務是否已載入
  running: boolean;                          // 服務是否正在運行
  env: Record<string, string> | null;        // 服務配置的環境變數
  command: string | null;                    // 服務的啟動命令
  runtime: GatewayServiceRuntime | null;     // 運行時狀態
}
```

**三級狀態模型**：

```
installed → loaded → running

已安裝（Installed）：服務配置檔案存在於作業系統中
  └→ 已載入（Loaded）：服務已被服務管理器載入
       └→ 運行中（Running）：服務進程正在執行

每一級都是前一級的子集：
  running ⊂ loaded ⊂ installed
```

這種分層狀態模型準確反映了服務管理器的實際行為——你可以安裝一個服務但不載入它，也可以載入一個服務但它可能因為錯誤而沒有在運行。

### 2.2 readGatewayServiceState() — 狀態讀取

`readGatewayServiceState()`（`service.ts:93-112`）彙整各項資訊來構建完整的狀態快照：

```typescript
// service.ts:93-112（虛擬碼）
async function readGatewayServiceState(
  service: GatewayService
): Promise<GatewayServiceState> {
  // 1. 讀取服務配置的啟動命令
  const command = await service.readCommand();

  // 2. 檢查服務是否已載入
  const loaded = await service.isLoaded();

  // 3. 讀取運行時狀態
  const runtime = await service.readRuntime();

  // 4. 組合為完整狀態
  return {
    installed: command !== null,
    loaded,
    running: runtime?.status === 'running',
    env: /* 從服務配置解析 */,
    command,
    runtime,
  };
}
```

**設計決策**：`installed` 的判斷基於 `command !== null`——如果服務配置檔案中有啟動命令，就認為服務已安裝。這比檢查配置檔案是否存在更可靠，因為檔案可能存在但內容損壞。

### 2.3 startGatewayService() — 服務啟動

`startGatewayService()`（`service.ts:114-143`）處理服務啟動的各種邊界情況：

```typescript
// service.ts:114-143（虛擬碼）
async function startGatewayService(
  service: GatewayService
): Promise<GatewayServiceStartResult> {
  // 1. 檢查服務是否已安裝
  const state = await readGatewayServiceState(service);

  if (!state.installed) {
    // 服務未安裝 → 回傳 "missing-install"，讓 CLI 提示使用者執行安裝
    return 'missing-install';
  }

  // 2. 透過 restart 來啟動（restart 在服務未運行時等同於 start）
  await service.restart();

  return 'started';
}
```

**為什麼用 restart 而不是 start？** 因為在大多數服務管理器中，`restart` 是冪等的（Idempotent）——如果服務已在運行，它會先停止再啟動；如果服務未運行，它會直接啟動。使用 `restart` 避免了「先檢查狀態再決定 start 或 restart」的競態條件（Race Condition）。

### 2.4 GatewayServiceStartResult — 啟動結果

```typescript
// service-types.ts:49-52
type GatewayServiceStartResult =
  | 'started'          // 服務已成功啟動
  | 'scheduled'        // 服務已排程（Windows schtasks 場景）
  | 'missing-install'; // 服務未安裝，需要先執行 install
```

`'scheduled'` 結果是 Windows 特有的——schtasks 可能不會立即啟動任務，而是排程在下一個觸發時間。

---

## 3. Linux：systemd 整合

### 3.1 服務命名與路徑

```typescript
// constants.ts:5
const SYSTEMD_SERVICE_NAME = 'openclaw-gateway';
```

服務名稱遵循 systemd 的命名慣例——小寫字母、連字號分隔。

```typescript
// systemd.ts:43-57
function resolveSystemdUnitPath(): string {
  // ~/.config/systemd/user/{name}.service
  return path.join(
    os.homedir(),
    '.config', 'systemd', 'user',
    `${SYSTEMD_SERVICE_NAME}.service`
  );
}
```

OpenClaw 使用 **systemd 使用者服務**（User Service），而非系統級別的服務。這意味著：
- 服務以使用者身份運行，不需要 root 權限
- 服務檔案放在 `~/.config/systemd/user/` 目錄
- 使用 `systemctl --user` 命令管理
- 預設情況下，只在使用者登入時運行

### 3.2 Unit 檔案生成

`buildSystemdUnit()`（`systemd-unit.ts:38-80`）動態生成 systemd unit 檔案的內容：

```ini
# systemd-unit.ts:38-80 生成的 unit 檔案內容
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/path/to/node /path/to/openclaw gateway --start
Restart=always
RestartSec=5
KillMode=control-group
Environment=NODE_ENV=production
# ... 其他環境變數

[Install]
WantedBy=default.target
```

**每個指令的意義**：

- **`After=network-online.target`**：確保在網路完全就緒後才啟動 Gateway。Gateway 需要網路來接收通道訊息和呼叫 LLM API，在網路就緒前啟動是沒有意義的。
- **`Wants=network-online.target`**：宣告對網路目標的弱依賴——如果網路目標失敗，Gateway 仍然會嘗試啟動（它可能只需要本地通訊）。
- **`Restart=always`**：無論退出原因為何都自動重啟。這是 Daemon 的核心需求——即使崩潰也要自動恢復。
- **`RestartSec=5`**：重啟間隔 5 秒。這個值平衡了兩個需求：夠短以便快速恢復，夠長以避免在持續崩潰時消耗過多系統資源。
- **`KillMode=control-group`**：殺死整個控制群組（包括子進程）。Gateway 可能會 fork 子進程來執行使用者的命令，停止服務時需要連同這些子進程一起清理。
- **`WantedBy=default.target`**：讓服務在使用者的預設目標（default.target）下自動啟動。

### 3.3 使用者 Linger 支援

```typescript
// systemd-linger.ts
async function enableSystemdUserLinger(): Promise<void>
```

**問題**：systemd 使用者服務預設只在使用者有活躍 Session（登入）時運行。當使用者登出時，所有使用者服務都會被停止——但 Gateway 是一個 24/7 服務，不應該因為使用者登出就停止。

**解決方案**：`loginctl enable-linger` 命令可以讓指定使用者的服務在沒有 Session 時繼續運行。`enableSystemdUserLinger()` 函式會自動執行這個命令。

**使用場景**：在伺服器環境中，管理員透過 SSH 登入、安裝 OpenClaw、啟動 Gateway，然後登出 SSH。如果沒有啟用 linger，Gateway 會在 SSH 斷開時停止——這顯然不是期望的行為。

### 3.4 systemd 不可用偵測

```typescript
// systemd-unavailable.ts
function classifySystemdUnavailableDetail(): SystemdUnavailableReason
```

並非所有 Linux 發行版都使用 systemd。WSL（Windows Subsystem for Linux）的早期版本、容器環境、某些極簡發行版可能使用不同的 init 系統。`classifySystemdUnavailableDetail()` 會偵測 systemd 不可用的原因，並提供有意義的錯誤訊息，而不是讓使用者面對一個模糊的「找不到 systemctl」錯誤。

---

## 4. macOS：launchd 整合

### 4.1 服務命名與路徑

```typescript
// constants.ts:4
const LAUNCHD_LABEL = 'ai.openclaw.gateway';
```

macOS 的 launchd 使用反向域名格式（Reverse Domain Name Notation）作為服務標籤，這是 Apple 生態系的慣例（類似 iOS/macOS 應用的 Bundle ID）。

```typescript
// launchd.ts:55-61
function resolvePlistPath(): string {
  // ~/Library/LaunchAgents/{label}.plist
  return path.join(
    os.homedir(),
    'Library', 'LaunchAgents',
    `${LAUNCHD_LABEL}.plist`
  );
}
```

**LaunchAgent vs LaunchDaemon**：macOS 的 launchd 區分兩種服務：
- **LaunchAgent**：在使用者的 Session 中運行，有存取 GUI 的能力。放在 `~/Library/LaunchAgents/`。
- **LaunchDaemon**：以 root 身份運行的系統級服務。放在 `/Library/LaunchDaemons/`。

OpenClaw 使用 **LaunchAgent**，因為 Gateway 需要存取使用者的環境變數、檔案系統權限，以及可能的 GUI 互動（如桌面通知）。

### 4.2 Plist 檔案生成

`buildLaunchAgentPlist()`（`launchd-plist.ts:89-117`）生成 macOS 的 Property List 配置：

```xml
<!-- launchd-plist.ts:89-117 生成的 plist 內容 -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.gateway</string>

    <key>ProgramArguments</key>
    <array>
        <string>/path/to/node</string>
        <string>/path/to/openclaw</string>
        <string>gateway</string>
        <string>--start</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>ThrottleInterval</key>
    <integer>1</integer>

    <key>Umask</key>
    <integer>63</integer>

    <key>EnvironmentVariables</key>
    <dict>
        <!-- 環境變數 -->
    </dict>
</dict>
</plist>
```

**每個設定的意義**：

- **`RunAtLoad=true`**：服務在被 launchd 載入時立即啟動。結合 LaunchAgent 的特性，這意味著使用者登入時自動啟動。
- **`KeepAlive=true`**：如果服務意外退出，launchd 會自動重啟它。這是 Daemon 持續運行的保證，等同於 systemd 的 `Restart=always`。
- **`ThrottleInterval=1`**（秒）：最小重啟間隔。預設值是 10 秒，OpenClaw 將其降低到 1 秒以實現更快的恢復。launchd 不會在此間隔內連續重啟服務。
- **`Umask=077`**（八進位 63 = 十進位的 `0o077`）：設定檔案建立遮罩，確保 Gateway 建立的所有檔案預設只有擁有者可讀寫。這是安全最佳實踐——Gateway 可能會建立包含敏感資料（Token、金鑰等）的檔案。

**ThrottleInterval 的選擇**：1 秒看似很短，但這是有意為之。Gateway 的崩潰通常是暫態的（例如網路中斷），快速重啟可以最小化服務中斷時間。如果崩潰是持續性的（例如配置錯誤），launchd 會在多次快速重啟後自動降低重啟頻率。

### 4.3 優雅重啟交接

```typescript
// launchd-restart-handoff.ts
function scheduleDetachedLaunchdRestartHandoff(): void
```

launchd 的重啟機制與 systemd 不同——你不能簡單地 `launchctl restart`（launchd 沒有 restart 動詞）。重啟需要先 `bootout`（卸載）再 `bootstrap`（載入），但這會造成一個短暫的服務中斷窗口。

`scheduleDetachedLaunchdRestartHandoff()` 解決了這個問題：

```
優雅重啟流程：
  1. 當前 Gateway 進程啟動一個分離的（Detached）重啟腳本
  2. 當前 Gateway 進程結束（launchd 偵測到服務退出）
  3. 分離腳本等待舊進程完全退出
  4. 分離腳本觸發 launchctl bootout + bootstrap
  5. launchd 啟動新的 Gateway 進程
  6. 分離腳本自我退出
```

**為什麼需要分離進程？** 因為如果在 Gateway 進程內直接執行 `launchctl bootout`，就等於「讓自己殺死自己」——這會導致 bootout 命令本身被中斷，後續的 bootstrap 永遠不會執行。分離進程不受 launchd 管理，可以安全地協調整個重啟過程。

---

## 5. Windows：schtasks 整合

### 5.1 任務命名與腳本路徑

```typescript
// constants.ts:6
const SCHTASKS_TASK_NAME = 'OpenClaw Gateway';
```

Windows 排程工作（Scheduled Tasks）使用人類可讀的名稱。

```typescript
// schtasks.ts:47-55
function resolveTaskScriptPath(): string {
  // {stateDir}/gateway.cmd
  return path.join(resolveGatewayStateDir(), 'gateway.cmd');
}
```

**為什麼用 .cmd 腳本而不是直接執行 Node.js？** 因為 schtasks 的 `/TR`（Task Run）參數對路徑格式有嚴格要求，而且需要處理空格、引號等問題。使用一個中間的 .cmd 腳本可以封裝這些複雜性，腳本內部再呼叫 Node.js 執行 OpenClaw。

### 5.2 啟動資料夾回退

`schtasks.ts:57-76` 實現了一個重要的回退機制：

```typescript
// schtasks.ts:57-76（虛擬碼）
async function installService(): Promise<void> {
  try {
    // 1. 嘗試使用 schtasks 建立排程工作
    await execSchtasks('/Create', ...);
  } catch (error) {
    if (isAccessDenied(error)) {
      // 2. 如果權限不足（企業環境中常見），
      //    回退到啟動資料夾（Startup Folder）
      await installToStartupFolder();
    }
  }
}
```

**為什麼需要回退？** 在企業環境中，IT 管理員可能透過群組原則（Group Policy）限制使用者建立排程工作的權限。啟動資料夾（`%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup`）是一個更「寬鬆」的自動啟動方式——它只是在使用者登入時執行一個捷徑，通常不受群組原則限制。

**啟動資料夾的缺點**：
- 只在使用者**互動式登入**時啟動（不支援無人值守的伺服器環境）
- 沒有自動重啟能力（服務崩潰後不會自動恢復）
- 無法精確控制啟動順序

因此這只是一個回退方案，OpenClaw 會在日誌中記錄警告。

### 5.3 schtasks-exec.ts — 命令包裝

```typescript
// schtasks-exec.ts
async function execSchtasks(...args: string[]): Promise<ExecResult>
```

這是對 Windows `schtasks.exe` 命令的薄包裝。它處理了 schtasks 的各種行為怪癖——例如某些錯誤只透過退出碼（Exit Code）而非 stderr 報告，某些成功操作會在 stdout 輸出中文或其他語言的訊息（取決於 Windows 的系統語言設定）。

---

## 6. 共用基礎設施

### 6.1 constants.ts — 服務常數與設定檔支援

```typescript
// constants.ts
const LAUNCHD_LABEL = 'ai.openclaw.gateway';            // macOS 服務標籤
const SYSTEMD_SERVICE_NAME = 'openclaw-gateway';         // Linux 服務名稱
const SCHTASKS_TASK_NAME = 'OpenClaw Gateway';           // Windows 任務名稱
const NODE_LAUNCH_AGENT_LABEL = 'ai.openclaw.node';      // Node 服務標籤
```

#### 多設定檔支援（Multi-Profile Support）

```typescript
// constants.ts
function normalizeGatewayProfile(profile: string | undefined): string

function formatGatewayServiceDescription(
  profile: string | undefined,
  version: string
): string
```

**設定檔**（Profile）允許在同一台機器上運行多個 Gateway 實例，每個實例有自己的配置。典型用例：
- 一個用於生產環境（Production）的 Gateway
- 一個用於開發測試（Development）的 Gateway

`normalizeGatewayProfile()` 將設定檔名稱標準化（例如去除空白、轉小寫），`formatGatewayServiceDescription()` 將設定檔名稱和版本號格式化為服務描述字串。

### 6.2 paths.ts — 路徑解析

```typescript
// paths.ts
function resolveHomeDir(): string
// 回傳 OpenClaw 的家目錄（預設為 ~/.openclaw）

function resolveGatewayStateDir(): string
// 回傳 Gateway 的狀態目錄（~/.openclaw{suffix}）
// suffix 由設定檔決定：預設設定檔無 suffix，其他設定檔有
```

狀態目錄（State Directory）儲存 Gateway 運行時需要的所有持久化資料——鎖檔案、PID 檔案、日誌、資料庫等。多設定檔支援透過不同的 suffix 來隔離各實例的狀態。

### 6.3 service-runtime.ts — 運行時狀態

```typescript
// service-runtime.ts
interface GatewayServiceRuntime {
  status: 'running' | 'stopped' | 'error';  // 服務狀態
  state: string;                              // 平台特定的狀態描述
  subState: string | null;                    // 子狀態（如 systemd 的 sub state）
  pid: number | null;                         // 進程 ID
  lastExitStatus: number | null;              // 最後一次退出碼
}
```

這個型別統一了三個平台的運行時狀態表示。每個平台的具體實現需要將平台特定的狀態映射到這個通用結構：

- **systemd**：`active (running)` → `{ status: 'running', state: 'active', subState: 'running' }`
- **launchd**：PID 存在 → `{ status: 'running', pid: 12345 }`
- **schtasks**：任務狀態 "Running" → `{ status: 'running' }`

### 6.4 service-env.ts — 服務環境建構

`buildServiceEnvironment()` 是共用基礎設施中最複雜的函式之一，負責為服務建構正確的環境變數集合：

```typescript
// service-env.ts
function buildServiceEnvironment(): Record<string, string>
```

#### 最小 PATH 建構

服務管理器啟動的進程通常只有一個極簡的 PATH（例如 `/usr/bin:/bin`），這不包含使用者安裝的工具（如 Node.js、Python）。`buildServiceEnvironment()` 會構建一個包含以下路徑的 PATH：

**macOS 特定路徑**（`resolveDarwinUserBinDirs()`）：
```typescript
// 偵測並加入以下路徑：
// - fnm:  ~/Library/Application Support/fnm
// - pnpm: ~/Library/pnpm
// - Homebrew: /usr/local/bin 或 /opt/homebrew/bin（Apple Silicon）
```

**Linux 特定路徑**（`resolveLinuxUserBinDirs()`）：
```typescript
// 偵測並加入以下路徑：
// - nvm:  ~/.nvm/current/bin
// - fnm:  ~/.fnm/current/bin
// - pnpm: ~/.local/share/pnpm
```

這些路徑偵測確保了即使 Node.js 是透過版本管理器（nvm、fnm）安裝的，Gateway 服務也能找到正確的 Node.js 執行檔。

#### Proxy 環境變數傳遞

```typescript
// service-env.ts
const SERVICE_PROXY_ENV_KEYS = [
  'HTTP_PROXY',
  'HTTPS_PROXY',
  'http_proxy',
  'https_proxy',
  'NO_PROXY',
  'no_proxy',
];
```

在企業環境中，網路流量通常需要透過代理（Proxy）伺服器。`SERVICE_PROXY_ENV_KEYS` 確保使用者的代理設定會被傳遞給 Gateway 服務——否則 Gateway 在服務管理器啟動的環境中可能無法存取外部網路。

注意：同時包含大寫和小寫版本（`HTTP_PROXY` 和 `http_proxy`），因為不同工具對環境變數的大小寫敏感度不同。

#### TLS CA 憑證

在企業環境中，可能使用自簽的 CA（Certificate Authority）憑證。`buildServiceEnvironment()` 會傳遞 `NODE_EXTRA_CA_CERTS` 環境變數，確保 Gateway 信任這些自簽憑證。

#### 版本管理器整合

對於使用 nvm 或 fnm 的使用者，`buildServiceEnvironment()` 會偵測當前使用的 Node.js 版本，並將對應的 bin 目錄加入 PATH。這確保了 Gateway 使用與使用者互動式 Shell 中相同的 Node.js 版本。

### 6.5 service-audit.ts — 服務配置稽核

```typescript
// service-audit.ts
async function auditGatewayServiceConfig(): Promise<AuditResult[]>
```

`auditGatewayServiceConfig()` 是 OpenClaw `doctor` 命令的核心之一，它對已安裝的服務配置進行全面的健康檢查：

**macOS 稽核項目**：
- **RunAtLoad 檢查**：確認 plist 中 `RunAtLoad` 為 `true`。如果為 `false`，服務不會在登入時自動啟動。
- **KeepAlive 檢查**：確認 `KeepAlive` 為 `true`。如果為 `false`，服務崩潰後不會自動重啟。

**Linux 稽核項目**：
- **network-online.target 檢查**：確認 unit 檔案中有 `After=network-online.target`。
- **RestartSec 檢查**：確認 `RestartSec` 的值合理（不會太短導致 CPU 空轉，也不會太長導致恢復緩慢）。

**跨平台稽核項目**：
- **執行時偵測**（Runtime Detection）：檢查服務是否意外地在 Bun（而非 Node.js）下運行。Bun 與 Node.js 的 API 相容性不是 100%，在 Bun 下運行可能導致微妙的錯誤。
- **Token 漂移**（Token Drift）：檢查服務配置中的認證 Token 是否與當前的 Token 一致。如果使用者重新認證（re-auth）了但沒有重新安裝服務，服務可能使用過期的 Token。
- **PATH 稽核**：檢查服務配置的 PATH 是否包含必要的目錄。如果 Node.js 的路徑缺失，服務可能無法啟動。

---

## 7. Node Service

除了 Gateway 服務之外，`src/daemon/` 也支援一個獨立的 **Node Service**：

```typescript
// constants.ts:9
const NODE_LAUNCH_AGENT_LABEL = 'ai.openclaw.node';
```

Node Service 是 OpenClaw 多節點（Multi-Node）架構中的一個輕量級服務。它與 Gateway 的區別：

| 特性 | Gateway Service | Node Service |
|------|----------------|--------------|
| 標籤 | `ai.openclaw.gateway` | `ai.openclaw.node` |
| 職責 | 接收通道訊息、分派 Agent | 提供計算節點能力 |
| 必要性 | 必須運行 | 可選 |

`buildNodeServiceEnvironment()`（定義於 `service-env.ts`）為 Node Service 建構專用的環境變數集合，與 Gateway 的環境建構共用大部分邏輯，但可能有不同的 PATH 需求。

---

## 8. 與 CLI 的整合

### 8.1 install-daemon 命令

CLI 的 `install-daemon` 命令是使用者安裝 Gateway 服務的入口：

```
openclaw install-daemon

# 內部流程：
# 1. resolveGatewayService()  → 取得當前平台的服務管理器
# 2. service.install()         → 安裝服務（寫入配置檔案）
# 3. service.restart()         → 啟動服務
```

如果 `resolveGatewayService()` 回傳 `null`（不支援的平台），CLI 會顯示錯誤訊息並建議使用者手動管理進程。

### 8.2 doctor 健康檢查

`doctor` 命令使用 `auditGatewayServiceConfig()` 來檢查服務的健康狀態：

```
openclaw doctor

# 服務相關的檢查項目：
# ✅ Gateway service installed
# ✅ RunAtLoad enabled
# ✅ KeepAlive enabled
# ⚠️ Token drift detected — run 'openclaw install-daemon' to update
# ✅ PATH includes Node.js
```

`doctor` 不只是報告問題，它還會提供修復建議——例如偵測到 Token 漂移時，建議使用者重新執行 `install-daemon`。

### 8.3 服務管理命令的完整流程

```
使用者操作                     內部流程
─────────────────────────    ──────────────────────────────────
openclaw install-daemon  →   resolveGatewayService()
                              → service.install()
                              → service.restart()

openclaw uninstall-daemon →  resolveGatewayService()
                              → service.stop()
                              → service.uninstall()

openclaw doctor           →  resolveGatewayService()
                              → readGatewayServiceState()
                              → auditGatewayServiceConfig()
```

---

## 9. 引用來源

以下是本章引用的所有原始碼檔案參考：

| 功能區域 | 檔案 | 行範圍 | 關鍵匯出 |
|----------|------|--------|----------|
| 服務抽象 | `source-repo/src/daemon/service.ts` | 66-78 | `GatewayService` 型別定義 |
| 服務抽象 | `source-repo/src/daemon/service.ts` | 93-112 | `readGatewayServiceState()` |
| 服務抽象 | `source-repo/src/daemon/service.ts` | 114-143 | `startGatewayService()` |
| 服務抽象 | `source-repo/src/daemon/service.ts` | 172-212 | `GATEWAY_SERVICE_REGISTRY` |
| 服務抽象 | `source-repo/src/daemon/service.ts` | 220-225 | `resolveGatewayService()` |
| 服務型別 | `source-repo/src/daemon/service-types.ts` | 40-47 | `GatewayServiceState` |
| 服務型別 | `source-repo/src/daemon/service-types.ts` | 49-52 | `GatewayServiceStartResult` |
| 常數 | `source-repo/src/daemon/constants.ts` | 4 | `LAUNCHD_LABEL` |
| 常數 | `source-repo/src/daemon/constants.ts` | 5 | `SYSTEMD_SERVICE_NAME` |
| 常數 | `source-repo/src/daemon/constants.ts` | 6 | `SCHTASKS_TASK_NAME` |
| 常數 | `source-repo/src/daemon/constants.ts` | 9 | `NODE_LAUNCH_AGENT_LABEL` |
| 常數 | `source-repo/src/daemon/constants.ts` | — | `normalizeGatewayProfile()`, `formatGatewayServiceDescription()` |
| 路徑 | `source-repo/src/daemon/paths.ts` | — | `resolveHomeDir()`, `resolveGatewayStateDir()` |
| systemd | `source-repo/src/daemon/systemd.ts` | 43-57 | `resolveSystemdUnitPath()` |
| systemd | `source-repo/src/daemon/systemd-unit.ts` | 38-80 | `buildSystemdUnit()` |
| systemd | `source-repo/src/daemon/systemd-linger.ts` | — | `enableSystemdUserLinger()` |
| systemd | `source-repo/src/daemon/systemd-unavailable.ts` | — | `classifySystemdUnavailableDetail()` |
| launchd | `source-repo/src/daemon/launchd.ts` | 55-61 | plist 路徑解析 |
| launchd | `source-repo/src/daemon/launchd-plist.ts` | 89-117 | `buildLaunchAgentPlist()` |
| launchd | `source-repo/src/daemon/launchd-restart-handoff.ts` | — | `scheduleDetachedLaunchdRestartHandoff()` |
| schtasks | `source-repo/src/daemon/schtasks.ts` | 47-55 | `resolveTaskScriptPath()` |
| schtasks | `source-repo/src/daemon/schtasks.ts` | 57-76 | 啟動資料夾回退邏輯 |
| schtasks | `source-repo/src/daemon/schtasks-exec.ts` | — | `execSchtasks()` |
| 運行時 | `source-repo/src/daemon/service-runtime.ts` | — | `GatewayServiceRuntime` |
| 環境 | `source-repo/src/daemon/service-env.ts` | — | `buildServiceEnvironment()`, `buildNodeServiceEnvironment()` |
| 環境 | `source-repo/src/daemon/service-env.ts` | — | `resolveDarwinUserBinDirs()`, `resolveLinuxUserBinDirs()` |
| 環境 | `source-repo/src/daemon/service-env.ts` | — | `SERVICE_PROXY_ENV_KEYS` |
| 稽核 | `source-repo/src/daemon/service-audit.ts` | — | `auditGatewayServiceConfig()` |
