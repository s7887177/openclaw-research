# 基礎設施工具庫（Infrastructure Utilities）

> **版本**：OpenClaw v2026.4.12
> **對應原始碼**：`src/infra/`（322 個檔案）

## 摘要

OpenClaw 的 `src/infra/` 目錄是整個系統的地基——它提供了所有上層模組共用的底層工具與抽象。這個目錄包含 322 個檔案，涵蓋了從錯誤處理到網路探索、從進程管理到安全執行的各個面向。任何想要深入理解 OpenClaw 運作機制的人，都必須先理解這一層。它不直接面對使用者，但所有使用者可見的功能——聊天、語音、技能執行、MCP 工具呼叫——都建立在這些基礎設施之上。理解 `src/infra/` 等同於理解 OpenClaw 的「骨骼系統」。

---

## 1. 目錄概覽與功能分類

`src/infra/` 的 322 個檔案可以依功能分為以下幾個主要群組。本節將逐一介紹每個群組的職責、關鍵檔案、以及它們匯出的核心 API。

### 1.1 錯誤與韌性（Error & Resilience）

**檔案數量**：約 10 個檔案
**核心職責**：統一錯誤分類、自動重試、退避策略、中斷信號管理

這是 OpenClaw 面對不穩定外部世界的第一道防線。與 LLM API 互動時，429 速率限制（Rate Limit）、逾時（Timeout）、上下文長度超限（Context Length Exceeded）等錯誤家常便飯。這個群組的目標是讓上層程式碼不需要逐一處理這些錯誤，而是透過統一的分類與自動重試機制來優雅地應對。

#### 1.1.1 errors.ts — 錯誤分類引擎

`errors.ts` 是錯誤處理的核心，它匯出了一系列用於解析、分類、格式化錯誤的工具函式：

```typescript
// 從錯誤物件中提取錯誤碼
extractErrorCode(error: unknown): string | undefined

// 讀取錯誤名稱（優先使用 error.name，回退到 constructor.name）
readErrorName(error: unknown): string | undefined

// 收集錯誤圖（Error Graph）中的所有候選錯誤
// 處理巢狀的 cause 鏈與 AggregateError
collectErrorGraphCandidates(error: unknown): Error[]

// 檢查是否為 errno 類型的系統錯誤
isErrno(error: unknown): boolean
hasErrnoCode(error: unknown, code: string): boolean

// 格式化錯誤訊息為人類可讀的字串
formatErrorMessage(error: unknown): string
```

最關鍵的是 `detectErrorKind()` 函式，它將任意錯誤歸類為五種已知類型之一：

```typescript
type ErrorKind =
  | 'refusal'         // 模型拒絕回應（內容政策等）
  | 'timeout'         // 請求逾時
  | 'rate_limit'      // 429 速率限制
  | 'context_length'  // 上下文長度超過模型限制
  | 'unknown';        // 無法分類的錯誤

detectErrorKind(error: unknown): ErrorKind
```

這個分類結果會直接影響重試策略的選擇——`rate_limit` 會觸發較長的退避時間，`context_length` 不會重試（因為重試也會失敗），`timeout` 會立即重試，而 `refusal` 會向使用者顯示明確的錯誤訊息。

**設計洞察**：`collectErrorGraphCandidates()` 的存在說明 OpenClaw 需要處理深層巢狀的錯誤結構。在現代 JavaScript 中，錯誤經常透過 `cause` 屬性鏈接，而 `AggregateError` 可能包含多個子錯誤。這個函式會遍歷整個「錯誤圖」（不僅僅是錯誤鏈），確保不會遺漏任何有用的資訊。

#### 1.1.2 backoff.ts — 指數退避（Exponential Backoff）

`backoff.ts` 是一個極為精簡的檔案（僅 29 行），提供退避計算的核心數學：

```typescript
// 計算第 n 次重試的等待時間（毫秒）
// base * 2^attempt + random_jitter
computeBackoff(attempt: number, base?: number): number

// 等待指定時間，但可被 AbortSignal 中斷
sleepWithAbort(ms: number, signal?: AbortSignal): Promise<void>
```

`computeBackoff()` 的核心邏輯是指數退避加抖動（Exponential Backoff with Jitter）。抖動的目的是避免「驚群效應」（Thundering Herd）——當多個客戶端同時遇到 429 錯誤時，如果它們都在完全相同的時間重試，就會再次同時觸發速率限制。加入隨機抖動後，重試時間會分散開來。

`sleepWithAbort()` 則是退避等待的實際執行者。它接受一個 `AbortSignal`，這意味著如果使用者取消了操作（例如按下 Ctrl+C），等待會立即中斷，而不是傻傻地等完整個退避時間。

#### 1.1.3 retry.ts — 通用重試框架

`retry.ts`（138 行）是一個完整的非同步重試框架：

```typescript
// 核心重試函式
async function retryAsync<T>(
  fn: () => Promise<T>,
  options: RetryOptions
): Promise<T>

// 將使用者提供的簡化選項解析為完整的重試配置
function resolveRetryConfig(options: Partial<RetryOptions>): RetryConfig

// 在基礎退避時間上加入抖動
function applyJitter(baseDelay: number, jitterFactor: number): number
```

核心型別定義：

```typescript
interface RetryConfig {
  maxRetries: number;       // 最大重試次數
  baseDelay: number;        // 基礎等待時間（毫秒）
  maxDelay: number;         // 最大等待時間上限
  jitterFactor: number;     // 抖動係數（0-1）
  retryableCheck?: (error: unknown) => boolean;  // 判斷錯誤是否值得重試
}

interface RetryInfo {
  attempt: number;          // 當前嘗試次數
  error: unknown;           // 最後一次錯誤
  totalElapsed: number;     // 累計耗時
}
```

**設計重點**：`retryableCheck` 回呼函式讓呼叫端可以自訂「什麼錯誤值得重試」。例如，HTTP 500 可以重試，但 HTTP 401（認證失敗）重試也沒用。這種可注入的判斷邏輯使得重試框架保持通用性。

#### 1.1.4 retry-policy.ts — 領域特定重試策略

如果 `retry.ts` 是通用重試框架，那 `retry-policy.ts`（119 行）就是針對 OpenClaw 特定場景的重試策略工廠：

```typescript
// 建立針對速率限制的重試執行器
function createRateLimitRetryRunner(options?: Partial<RetryConfig>): RetryRunner

// 建立針對 Channel API 呼叫的重試執行器
function createChannelApiRetryRunner(options?: Partial<RetryConfig>): RetryRunner

// Channel API 的預設重試配置
const CHANNEL_API_RETRY_DEFAULTS: RetryConfig
```

這個檔案內含正則表達式（Regex）用於偵測 429 狀態碼和逾時錯誤，將通用的 `retryAsync()` 包裝成特定用途的執行器。`CHANNEL_API_RETRY_DEFAULTS` 定義了與各通道（Discord、WhatsApp 等）API 互動時的預設重試參數。

**架構意義**：這種「通用框架 + 領域策略」的分層設計，是 `src/infra/` 中反覆出現的模式。底層提供靈活的抽象，上層提供開箱即用的具體實現。

#### 1.1.5 abort-signal.ts 與 fetch.ts

`abort-signal.ts` 提供 `AbortSignal` 的工具函式，用於協調取消操作在整個系統中的傳播。`fetch.ts` 則是對原生 `fetch()` 的包裝，整合了重試策略與中斷信號。

#### 1.1.6 unhandled-rejections.ts

這個檔案處理 Node.js 的 `unhandledRejection` 事件——當一個 Promise 被 reject 但沒有被 catch 時，它確保這些「漏網之魚」不會導致進程靜默崩潰，而是被正確記錄並視情況優雅退出。

---

### 1.2 Process 與 Gateway 管理（Process & Gateway Management）

**檔案數量**：約 15 個檔案
**核心職責**：Gateway 進程鎖、進程生命週期管理、跨平台重啟、重啟後狀態恢復

OpenClaw 的 Gateway 是一個長期運行的常駐服務（Daemon），負責接收來自各通道的訊息並分派給 Agent 處理。由於 Gateway 必須保證同一時間只有一個實例在運行（避免重複回應訊息），進程管理變得至關重要。

#### 1.2.1 gateway-lock.ts — 分散式進程鎖

`gateway-lock.ts`（345 行）是這個群組中最複雜的檔案，實現了跨平台的進程鎖（Process Lock）機制：

```typescript
async function acquireGatewayLock(options: LockOptions): Promise<LockHandle | null>
```

這個函式的職責看似簡單——「確保只有一個 Gateway 實例在運行」——但實現起來充滿了平台特定的陷阱。

**平台特定的進程偵測**：

- **Linux**：讀取 `/proc/{pid}/cmdline` 來確認 PID 對應的進程是否真的是 Gateway。這是必要的，因為 PID 可能被作業系統回收再利用——鎖檔案中記錄的 PID 42 可能已經不是原來的 Gateway，而是一個全新的不相關進程。
- **macOS**：使用類似的機制，但路徑和格式略有不同。
- **Windows**：透過 Windows API 查詢進程資訊。

**鎖的完整生命週期**：

1. **嘗試取得鎖**：讀取鎖檔案，檢查是否已有活躍的 Gateway
2. **PID 存活檢查**：確認鎖檔案中的 PID 是否仍然存活
3. **Port 存活偵測**：即使 PID 存活，也要確認它是否真的在監聽預期的埠
4. **過期鎖處理**：如果鎖的持有者已經死亡，清除過期鎖並重新取得
5. **寫入新鎖**：原子性地寫入自己的 PID

**設計洞察**：光靠 PID 檢查是不夠的，因為：
- PID 可能被回收（Linux 的 PID 空間是有限的）
- 進程可能活著但已經卡住（Zombie Process）
- 多台機器可能共用同一個 HOME 目錄（NFS 掛載）

因此 `gateway-lock.ts` 採用了多層驗證策略：PID 存活 → 進程身份確認 → 網路埠存活 → 鎖檔案時間戳。

#### 1.2.2 gateway-processes.ts — 進程資訊查詢

`gateway-processes.ts`（60 行）提供進程級別的查詢工具：

```typescript
// 同步讀取 Gateway 進程的命令列參數
function readGatewayProcessArgsSync(pid: number): string[] | null

// 向已驗證的 Gateway PID 發送信號
function signalVerifiedGatewayPidSync(pid: number, signal: string): boolean

// 在指定埠上尋找已驗證的 Gateway 監聽器 PID
function findVerifiedGatewayListenerPidsOnPortSync(port: number): number[]
```

這些函式的共通特徵是「已驗證」（Verified）——它們不只是盲目地對 PID 發送信號，而是會先確認該 PID 確實屬於一個 Gateway 進程。

#### 1.2.3 gateway-process-argv.ts — 命令列解析

`gateway-process-argv.ts`（65 行）負責解析不同平台上的進程命令列：

```typescript
// 解析 Linux /proc/{pid}/cmdline 的 NUL 分隔格式
function parseProcCmdline(buffer: Buffer): string[]

// 解析 Windows 的命令列格式（處理引號與空格）
function parseWindowsCmdline(cmdline: string): string[]

// 判斷一組命令列參數是否屬於 Gateway 進程
function isGatewayArgv(argv: string[]): boolean
```

`isGatewayArgv()` 是進程身份驗證的最終判斷——它檢查命令列中是否包含 Gateway 特有的標記參數。

#### 1.2.4 restart.ts — 重啟協調器

`restart.ts`（524 行）是整個進程管理群組中最大的檔案，處理 Gateway 重啟的各種場景：

```typescript
// 發出 Gateway 重啟事件
function emitGatewayRestart(reason: string): void

// 透過 SIGUSR1 信號觸發優雅重啟
function scheduleGatewaySigusr1Restart(): void

// 延遲重啟直到所有活躍對話結束
function deferGatewayRestartUntilIdle(): void

// 觸發完整的 OpenClaw 重啟（平台特定）
function triggerOpenClawRestart(method: 'systemd' | 'launchctl' | 'schtasks'): void

// 設定預重啟延遲檢查（用於判斷是否可以安全重啟）
function setPreRestartDeferralCheck(check: () => boolean): void
```

**重啟策略的設計考量**：

Gateway 不能在對話進行中突然重啟——這會導致使用者的訊息丟失。因此 `deferGatewayRestartUntilIdle()` 會等待所有活躍的對話完成後才執行重啟。但如果有一個對話永遠不結束呢？`setPreRestartDeferralCheck()` 允許設定一個逾時機制，在等待過久時強制重啟。

`triggerOpenClawRestart()` 的三個參數對應三個作業系統的服務管理器：
- `systemd`：透過 `systemctl --user restart` 命令
- `launchctl`：透過 `launchctl kickstart` 命令
- `schtasks`：透過 Windows 排程工作重新執行

#### 1.2.5 restart-sentinel.ts — 重啟哨兵

`restart-sentinel.ts`（147 行）解決了一個微妙的問題：**重啟後如何繼續之前的對話？**

```typescript
// 寫入重啟哨兵檔案（記錄重啟前的投遞上下文）
function writeRestartSentinel(context: DeliveryContext): void

// 讀取重啟哨兵（但不消費）
function readRestartSentinel(): DeliveryContext | null

// 讀取並刪除重啟哨兵（一次性消費）
function consumeRestartSentinel(): DeliveryContext | null
```

當 Gateway 需要重啟時，它會將當前正在處理的投遞上下文（Delivery Context）——包括通道、對話 ID、最後的訊息等——寫入一個哨兵檔案。重啟完成後，新的 Gateway 實例會消費這個哨兵，恢復之前的上下文，並繼續處理。

**「消費」語意**：`consumeRestartSentinel()` 採用「讀取後刪除」的語意，確保同一個重啟上下文不會被處理兩次。

#### 1.2.6 process-respawn.ts — 進程重生

`process-respawn.ts`（68 行）處理更底層的進程重生：

```typescript
// 以全新 PID 重啟 Gateway 進程
function restartGatewayProcessWithFreshPid(): void
```

這個函式會啟動一個分離的（Detached）子進程來替代當前進程。它會偵測當前是否在服務管理器（Supervisor）下運行：

- 如果在 `launchd` 下運行，讓 launchd 負責重啟
- 如果在 `systemd` 下運行，讓 systemd 負責重啟
- 如果在 `schtasks` 下運行，重新排程任務
- 如果是裸進程（無監管器），自行啟動分離子進程

#### 1.2.7 supervisor-markers.ts — 監管器偵測

`supervisor-markers.ts`（44 行）透過環境變數偵測當前的服務管理器：

```typescript
// 偵測當前進程的服務管理器類型
function detectRespawnSupervisor(): 'launchd' | 'systemd' | 'schtasks' | null
```

偵測方式是檢查特定的環境變數——例如 systemd 會設定 `INVOCATION_ID`，launchd 有其特定的環境標記。

---

### 1.3 網路與通訊（Network & Communication）

**檔案數量**：約 25 個檔案
**核心職責**：Gateway 探索、SSH 隧道、區域網路通訊、DNS 記錄生成

OpenClaw Gateway 不只是一個本地服務——它可以被遠端存取，甚至在區域網路中被自動發現。這個群組處理所有與網路相關的基礎設施。

#### 1.3.1 bonjour.ts / bonjour-discovery.ts — mDNS 服務探索

這兩個檔案實現了 Gateway 的自動發現機制，使用 mDNS/Bonjour 協定讓同一區域網路中的裝置能夠自動找到 Gateway：

- **macOS**：使用系統內建的 `dns-sd` 命令進行服務註冊與探索
- **Linux**：使用 `avahi-browse` 命令（Avahi 是 Linux 上的 mDNS 實現）

**使用場景**：想像你在家裡的桌機上運行 Gateway，然後在筆電上使用 OpenClaw CLI。CLI 可以透過 Bonjour 自動發現同一 Wi-Fi 網路中的 Gateway，無需手動輸入 IP 和 Port。

#### 1.3.2 ssh-tunnel.ts — SSH 隧道管理

當 Gateway 在遠端伺服器上運行時，`ssh-tunnel.ts` 提供 SSH 隧道管理，包括暫態埠（Ephemeral Port）分配——系統會自動選擇一個未使用的本地埠來建立隧道，避免埠衝突。

#### 1.3.3 tailnet.ts — Tailscale 網路偵測

`tailnet.ts`（47 行）提供 Tailscale VPN 網路的偵測功能：

```typescript
// 判斷 IPv4 位址是否屬於 Tailscale 的 CGNAT 範圍
function isTailnetIPv4(ip: string): boolean  // 檢查 100.64.0.0/10

// 列舉本機的所有 Tailscale 位址
function listTailnetAddresses(): string[]
```

Tailscale 使用 `100.64.0.0/10` 的 CGNAT（Carrier-Grade NAT）位址範圍。`isTailnetIPv4()` 透過位址範圍檢查來判斷一個 IP 是否來自 Tailscale 網路。

**使用場景**：當 Gateway 偵測到自己在 Tailscale 網路中時，可以將 Tailscale IP 作為服務位址廣播，讓同一 Tailnet 中的其他裝置直接連線。

#### 1.3.4 widearea-dns.ts — 廣域 DNS 探索

`widearea-dns.ts`（199 行）生成 DNS 區域檔案（Zone File），用於廣域網路的 Gateway 探索。這是比 Bonjour（僅限區域網路）更大範圍的探索機制——透過 DNS TXT/SRV 記錄，讓任何能存取該 DNS 的裝置都能找到 Gateway。

#### 1.3.5 jsonl-socket.ts — JSONL Unix Socket 協定

`jsonl-socket.ts`（64 行）實現了基於 Unix Socket 的 JSONL（JSON Lines）通訊協定：

```typescript
// JSONL 格式：每行一個 JSON 物件，以換行符分隔
// 使用半關閉（Half-Close）模式：
//   1. 客戶端發送完畢後關閉寫入端
//   2. 伺服端讀取完畢後處理並回應
//   3. 伺服端回應完畢後關閉連線
```

**半關閉模式（Half-Close Pattern）** 的優勢在於它提供了明確的「訊息結束」信號——當客戶端關閉寫入端時，伺服端知道它已經收到了完整的請求。這比使用分隔符或長度前綴更簡單可靠。

這個協定被用於本地進程間通訊，例如 CLI 與 Gateway 之間的交互。

#### 1.3.6 ws.ts — WebSocket 工具

`ws.ts`（22 行）是一個極小的工具檔案，提供 WebSocket RawData 到 Buffer/String 的轉換。

#### 1.3.7 ports.ts — 埠管理

`ports.ts`（96 行）提供埠相關的工具：

```typescript
// 確認指定埠是否可用（嘗試綁定後立即釋放）
function ensurePortAvailable(port: number): Promise<boolean>

// 描述埠的當前佔用者（透過 lsof/netstat）
function describePortOwner(port: number): string | null
```

`describePortOwner()` 在排錯時特別有用——當 Gateway 無法啟動因為埠被佔用時，它可以告訴使用者是哪個進程在用這個埠。

---

### 1.4 心跳與監控（Heartbeat & Monitoring）

**檔案數量**：約 20 個檔案
**核心職責**：Gateway 心跳執行、事件發佈、確定性調度

心跳機制是 Gateway 健康監控的核心——它定期執行健康檢查，確保 Gateway 的各個子系統正常運作，並在發現問題時觸發告警或自動修復。

#### 1.4.1 heartbeat-runner.ts — 心跳執行引擎

`heartbeat-runner.ts` 是心跳系統的核心，負責定期執行心跳任務。每次心跳會收集系統狀態（記憶體使用、活躍連線數、佇列長度等）並決定是否需要採取行動。

#### 1.4.2 heartbeat-events.ts — 心跳事件

`heartbeat-events.ts`（70 行）提供心跳事件的發佈/訂閱機制：

```typescript
// 發出心跳事件
function emitHeartbeatEvent(event: HeartbeatEvent): void

// 監聽心跳事件
function onHeartbeatEvent(handler: (event: HeartbeatEvent) => void): Disposable
```

這種發佈/訂閱（Pub/Sub）模式讓心跳系統與其消費者解耦——心跳執行引擎不需要知道誰在關心心跳結果，它只管發出事件。

#### 1.4.3 heartbeat-schedule.ts — 確定性調度

`heartbeat-schedule.ts`（60 行）使用 SHA-256 種子來實現確定性的心跳相位調度：

```typescript
// 基於設備 ID 的 SHA-256 雜湊值來決定心跳的相位偏移
// 確保多個 Gateway 實例（例如在叢集中）不會同時執行心跳
```

**為什麼需要確定性調度？**

如果多個 Gateway 實例都在同一時間執行心跳（例如每分鐘的第 0 秒），它們會同時對後端服務發起健康檢查請求，造成流量尖峰。透過 SHA-256 種子計算每個實例的相位偏移，心跳時間會自然分散。

這個設計的巧妙之處在於它是**確定性的**——同一個設備 ID 永遠會得到同一個相位偏移。這意味著重啟後心跳不會跳到不同的時間點，有助於監控系統的一致性。

---

### 1.5 系統事件與存在感（System Events & Presence）

**核心職責**：系統級事件佇列、線上存在感追蹤、系統訊息標記

#### 1.5.1 system-events.ts — 系統事件佇列

`system-events.ts`（212 行）管理系統級別的事件：

```typescript
// 將系統事件加入佇列
function enqueueSystemEvent(event: SystemEvent): void

// 排出（Drain）佇列中的所有事件
function drainSystemEventEntries(): SystemEvent[]
```

**設計限制**：每個 Session 最多 20 個事件。這個硬限制防止事件佇列在異常情況下無限增長——例如當某個子系統瘋狂地產生事件時。超過限制的事件會被靜默丟棄，優先保留最近的事件。

#### 1.5.2 system-presence.ts — 系統存在感

`system-presence.ts`（288 行）追蹤系統中各實體的存在狀態：

```typescript
// 更新實體的存在感（類似「最後上線時間」）
function updateSystemPresence(entityId: string, metadata?: object): void
```

**設計參數**：
- **TTL（存活時間）**：5 分鐘。如果 5 分鐘內沒有更新，該實體被視為離線。
- **LRU 最大條目數**：200。當條目數超過 200 時，最久未更新的條目會被淘汰。

這些參數的選擇反映了 OpenClaw 的典型使用場景——通常不會有超過 200 個同時活躍的實體（裝置、節點等）。

#### 1.5.3 system-message.ts — 系統訊息標記

`system-message.ts`（21 行）是一個極小的工具，定義了系統訊息的前綴標記：

```typescript
const SYSTEM_MARK = '⚙️';  // 所有系統訊息都以此符號開頭
```

這個看似微不足道的標記在 UI 層很重要——它讓使用者能夠一眼區分系統訊息和 Agent 回應。

---

### 1.6 執行與安全（Execution & Security）

**檔案數量**：約 50 個檔案
**核心職責**：安全的命令執行、Shell 環境管理、執行核准、執行時版本檢查

這是 `src/infra/` 中最大的功能群組，反映了「讓 AI 執行命令」這件事的安全複雜度。

#### 1.6.1 exec-host.ts — HMAC-SHA256 簽名執行

`exec-host.ts`（81 行）實現了安全的命令執行協定：

```typescript
// 透過 JSONL Socket 執行命令，所有請求都經過 HMAC-SHA256 簽名
// 簽名流程：
//   1. CLI/Agent 構建執行請求
//   2. 使用共享金鑰對請求內容計算 HMAC-SHA256 簽名
//   3. 將請求 + 簽名透過 JSONL Socket 發送給執行主機
//   4. 執行主機驗證簽名後才執行命令
```

**為什麼需要簽名？** 因為 OpenClaw 的 Gateway 可能監聽在網路上（而不僅僅是 localhost），如果任何人都能透過 Socket 發送執行請求，那就等於一個遠端程式碼執行（RCE）漏洞。HMAC-SHA256 簽名確保只有持有共享金鑰的合法客戶端才能請求執行命令。

#### 1.6.2 exec-safety.ts — 執行安全檢查

`exec-safety.ts`（45 行）是最後一道安全防線：

```typescript
// 檢查要執行的值是否安全
function isSafeExecutableValue(value: string): boolean
```

這個函式會拒絕包含以下內容的執行值：
- **Shell 元字符**（Shell Metacharacters）：`|`, `;`, `&`, `` ` ``, `$()`, `>`, `<` 等。這些字符可能被用來注入額外的命令。
- **控制字符**（Control Characters）：ASCII 0x00-0x1F（除了換行和 Tab）。這些可能被用來混淆日誌或繞過安全檢查。

**設計哲學**：「白名單」思維——與其嘗試辨識所有危險的命令（這是不可能的），不如只允許格式安全的值通過。

#### 1.6.3 exec-approvals.ts — 執行核准追蹤

`exec-approvals.ts` 管理使用者對命令執行的核准決策。它維護一個允許清單（Allowlist）模式匹配系統——使用者可以核准「所有 `git *` 命令」或「特定目錄下的 `npm install`」，而不需要逐一核准每個命令。

#### 1.6.4 shell-env.ts — Shell 環境建構

`shell-env.ts`（250 行）負責為執行的命令提供正確的環境變數。這比聽起來要複雜得多：

```typescript
// 問題場景：
//   Gateway 以 systemd 服務運行 → 沒有使用者的登入 Shell 環境
//   使用者的 .bashrc / .zshrc 中設定了必要的 PATH
//   執行的命令需要這些 PATH 才能找到 node/python/etc.
//
// 解決方案：
//   啟動一個登入 Shell，讀取其環境變數，作為執行命令的環境
```

**ZDOTDIR 攻擊防護**：Zsh 的 ZDOTDIR 環境變數可以重定向 .zshrc 的讀取位置。如果攻擊者能控制這個變數，就能注入惡意的 Shell 配置。`shell-env.ts` 會主動清除 ZDOTDIR，防止此攻擊向量。

#### 1.6.5 runtime-guard.ts — 執行時版本守衛

`runtime-guard.ts`（123 行）確保 OpenClaw 運行在足夠新的 Node.js 版本上：

```typescript
// 強制要求 Node.js >= 22.14.0
// 如果版本不足，顯示明確的錯誤訊息並退出
```

**為什麼是 22.14.0？** 這暗示 OpenClaw 依賴了 Node.js 22 中引入的特定 API——可能是 `require(esm)` 支援、改進的 `AbortSignal` API、或 WebSocket 的原生支援。

---

### 1.7 更新管理（Update Management）

**檔案數量**：約 15 個檔案
**核心職責**：版本檢查、自動更新、更新排程

#### 1.7.1 update-check.ts — 版本檢查

`update-check.ts`（423 行）執行全面的版本檢查：

```typescript
// 檢查項目：
//   1. npm registry 上的最新版本
//   2. 是否從 git 倉庫安裝（不支援自動更新）
//   3. 依賴項目的狀態（是否有已知的安全漏洞）
```

它會查詢 npm registry 的 `latest` tag，與當前版本比較，並根據語意版本（Semantic Versioning）判斷是否有可用的更新、是否為安全性更新（Patch 版本）或功能性更新（Minor/Major 版本）。

#### 1.7.2 update-runner.ts — 自動更新執行器

`update-runner.ts` 是自動更新的執行器。它負責下載、安裝新版本，並在安裝完成後觸發 Gateway 重啟。更新過程帶有抖動（Jitter），避免大量實例同時下載更新導致 npm registry 過載。

#### 1.7.3 update-startup.ts — 更新排程

`update-startup.ts`（528 行）在 Gateway 啟動時設定定期更新檢查的排程。它的大小（528 行）暗示了更新排程的複雜性——需要處理各種邊界情況，例如：
- 離線環境中的檢查失敗
- 更新過程中的 Gateway 重啟
- 同時多個實例嘗試更新
- 使用者手動更新與自動更新的衝突

---

### 1.8 設備與配對（Device & Pairing）

**檔案數量**：約 15 個檔案
**核心職責**：設備身份、Token 配對、節點配對

#### 1.8.1 device-identity.ts — 設備身份

`device-identity.ts` 為每個 OpenClaw 實例生成唯一的設備身份：

```typescript
// 使用 Ed25519 金鑰對作為設備身份
// SHA-256 指紋用於快速比對
//
// Ed25519 被選中的原因：
//   - 金鑰小（32 bytes）
//   - 簽名快
//   - 安全性高（抗側通道攻擊）
```

設備身份是整個信任模型的基礎——所有的配對、認證、HMAC 簽名都建立在設備身份之上。

#### 1.8.2 device-pairing.ts / device-bootstrap.ts — 設備配對

這兩個檔案實現了新設備加入 OpenClaw 網路的配對流程：

```typescript
// 配對流程：
//   1. 已信任的設備生成一次性配對 Token
//   2. 新設備提交 Token + 自己的公鑰
//   3. 已信任的設備驗證 Token 並記錄新設備的公鑰
//   4. 配對完成，新設備獲得信任
```

#### 1.8.3 node-pairing.ts — 節點配對

`node-pairing.ts` 處理 OpenClaw 多節點架構中的節點配對。配對 Token 有 5 分鐘的 TTL（存活時間），過期後需要重新生成。短 TTL 是安全設計——即使 Token 被洩漏，攻擊窗口也只有 5 分鐘。

---

### 1.9 檔案與資料管理（File & Data Management）

**檔案數量**：約 30 個檔案
**核心職責**：安全的檔案讀寫、機密檔案保護、壓縮檔處理、剪貼簿存取

#### 1.9.1 json-file.ts — 原子性 JSON 檔案操作

`json-file.ts`（130 行）提供安全的 JSON 檔案讀寫：

```typescript
// 原子寫入（Atomic Write）：
//   1. 寫入暫存檔案（temp file）
//   2. fsync 確保資料寫入磁碟
//   3. rename 替換目標檔案（原子操作）
//
// 安全特性：
//   - 符號連結追蹤（Symlink Traversal）：追蹤符號連結到最終目標
//   - 檔案權限 0o600：只有擁有者可讀寫
```

**為什麼需要原子寫入？** 如果在寫入過程中進程崩潰，普通的 `writeFile()` 可能留下一個半寫入的檔案——這對 JSON 來說意味著一個格式損壞的檔案。原子寫入確保檔案要麼是完整的舊版本，要麼是完整的新版本，永遠不會處於中間狀態。

#### 1.9.2 secret-file.ts — 機密檔案保護

`secret-file.ts`（141 行）為機密檔案（Token、金鑰等）提供額外的安全層：

```typescript
// 安全檢查：
//   1. 符號連結拒絕（Symlink Rejection）：不允許機密檔案是符號連結
//      → 防止攻擊者透過符號連結將機密檔案指向可讀取的位置
//   2. 大小限制：16KB
//      → 防止惡意的超大檔案耗盡記憶體
```

**為什麼拒絕符號連結？** 如果攻擊者能在機密檔案的位置建立一個符號連結（指向 `/etc/shadow` 或其他敏感檔案），當 OpenClaw 讀取「自己的」機密檔案時，實際上讀取的是被連結的目標。這是一種經典的 TOCTOU（Time-of-Check to Time-of-Use）攻擊。

#### 1.9.3 archive.ts — 壓縮檔處理

`archive.ts` 支援 ZIP 和 TAR 格式的解壓縮，並內建了**符號連結穿越防護**（Symlink Traversal Prevention）。惡意的壓縮檔可能包含指向系統敏感目錄的符號連結，解壓縮後就能存取這些目錄。`archive.ts` 會在解壓縮時檢查並拒絕此類符號連結。

#### 1.9.4 clipboard.ts — 跨平台剪貼簿

`clipboard.ts`（26 行）提供跨平台的剪貼簿存取：

```typescript
// 回退鏈（Fallback Chain）：
//   macOS:   pbcopy
//   Linux:   xclip → wl-copy（Wayland 支援）
//   Windows: clip.exe
//   WSL:     clip.exe（透過 Windows 互操作）
```

回退鏈的設計確保了在各種環境中都能正常運作——從原生桌面到 WSL（Windows Subsystem for Linux）。

---

### 1.10 環境與設定（Environment & Configuration）

**檔案數量**：約 20 個檔案
**核心職責**：家目錄解析、套件根目錄解析、資料庫遷移

#### 1.10.1 home-dir.ts — 家目錄解析

`home-dir.ts`（145 行）負責找到 OpenClaw 的資料儲存位置：

```typescript
// 優先順序：
//   1. OPENCLAW_HOME 環境變數（使用者自訂位置）
//   2. ~ 展開（HOME 環境變數或 os.homedir()）
//
// 支援 ~ 波浪號展開
```

`OPENCLAW_HOME` 環境變數允許使用者將 OpenClaw 的資料儲存在非預設位置——例如放在加密磁碟上，或放在 NFS 共享目錄上以便多台機器共用設定。

#### 1.10.2 openclaw-root.ts — 套件根目錄解析

`openclaw-root.ts`（136 行）解析 OpenClaw 套件本身的安裝位置：

```typescript
// 支援符號連結感知（Symlink Awareness）：
//   - nvm：~/.nvm/versions/node/vX.Y.Z/lib/node_modules/openclaw → 實際位置
//   - fnm：類似 nvm 的符號連結結構
//   - Homebrew：/usr/local/lib/node_modules/openclaw → Cellar 中的實際位置
```

這對更新機制至關重要——自動更新需要知道 OpenClaw 的實際安裝位置，而不是符號連結的位置，才能正確地替換檔案。

#### 1.10.3 state-migrations.ts — 資料庫遷移

`state-migrations.ts` 提供資料庫結構遷移（Schema Migration）的基礎設施。當 OpenClaw 更新時，可能需要修改本地資料庫的結構——新增欄位、修改索引等。這個檔案確保遷移是有序的、冪等的（可以安全地重複執行），且不會丟失現有資料。

---

## 2. 關鍵設計模式

### 2.1 Gateway Lock — 分散式進程鎖

Gateway Lock 是 OpenClaw 確保單一實例運行的核心機制。它的設計需要解決分散式系統中的經典問題：

```
問題：如何在沒有中央協調者的情況下，確保只有一個 Gateway 實例在運行？

解決方案：
  ┌──────────────────────────────────────────┐
  │            acquireGatewayLock()           │
  │                                          │
  │  1. 讀取鎖檔案                           │
  │     ├→ 無鎖 → 直接取得鎖               │
  │     └→ 有鎖 → 驗證持有者               │
  │                                          │
  │  2. 驗證持有者                           │
  │     ├→ PID 不存在 → 過期鎖 → 清除並取得│
  │     └→ PID 存在 → 進一步驗證           │
  │                                          │
  │  3. 進一步驗證                           │
  │     ├→ 進程身份不匹配 → 過期鎖         │
  │     └→ 身份匹配 → 檢查埠存活           │
  │                                          │
  │  4. 埠存活                               │
  │     ├→ 埠沒回應 → 殭屍鎖 → 清除       │
  │     └→ 埠有回應 → 鎖有效 → 取得失敗   │
  └──────────────────────────────────────────┘
```

**跨平台差異**：
- **Linux**：透過 `/proc/{pid}/cmdline` 驗證進程身份
- **macOS**：使用類似方式但路徑格式不同
- **Windows**：透過 Windows 進程 API 查詢

### 2.2 重試策略 — 指數退避 + 抖動

OpenClaw 的重試策略採用經典的「指數退避 + 抖動」（Exponential Backoff with Jitter）模式：

```
重試等待時間 = base × 2^attempt + random(0, base × jitterFactor)

範例（base=1000ms, jitter=0.5）：
  第 1 次重試: 1000 × 2^0 + random(0, 500) = 1000~1500ms
  第 2 次重試: 1000 × 2^1 + random(0, 500) = 2000~2500ms
  第 3 次重試: 1000 × 2^2 + random(0, 500) = 4000~4500ms
  第 4 次重試: 1000 × 2^3 + random(0, 500) = 8000~8500ms
  ...直到 maxDelay 上限
```

**三層抽象**：

1. **computeBackoff()**（`backoff.ts`）：純數學，計算等待時間
2. **retryAsync()**（`retry.ts`）：通用框架，接受任何非同步函式
3. **createRateLimitRetryRunner()**（`retry-policy.ts`）：領域策略，內建 429 偵測

**整合點**：`retry-policy.ts` 中的 `createChannelApiRetryRunner()` 使用正則表達式偵測 HTTP 429 狀態碼和逾時錯誤訊息，自動選擇適當的退避策略。速率限制錯誤會得到較長的退避時間（因為 API 明確告訴你要慢下來），而一般逾時只需短暫重試。

### 2.3 安全執行 — HMAC-SHA256 簽名 + Allowlist

OpenClaw 的命令執行安全模型是多層防禦（Defense in Depth）的體現：

```
外層：exec-approvals.ts ── 使用者必須核准命令類別
  ↓
中層：exec-safety.ts ── 拒絕含危險字符的命令
  ↓
內層：exec-host.ts ── HMAC-SHA256 簽名驗證
  ↓
底層：shell-env.ts ── 清潔的 Shell 環境（防止環境變數注入）
```

每一層各自獨立地提供保護：
- **Allowlist**（`exec-approvals.ts`）：確保使用者意識到並同意執行的命令類別
- **安全值檢查**（`exec-safety.ts`）：即使通過核准，仍然拒絕含 Shell 元字符的命令
- **簽名驗證**（`exec-host.ts`）：即使前兩層被繞過，網路上的未授權請求也無法執行
- **環境清潔**（`shell-env.ts`）：即使命令本身安全，也防止透過環境變數（如 `ZDOTDIR`）注入惡意配置

### 2.4 心跳系統 — 確定性調度

心跳調度的確定性設計是一個精巧的工程選擇：

```typescript
// 虛擬碼
function computeHeartbeatPhase(deviceId: string, intervalMs: number): number {
  const hash = sha256(deviceId);
  const seed = hash.readUInt32BE(0);  // 取 hash 的前 4 bytes
  return seed % intervalMs;            // 相位偏移 = hash mod 間隔
}

// 設備 A (deviceId="abc...") → 相位偏移 23,456ms
// 設備 B (deviceId="def...") → 相位偏移 47,891ms
// 設備 C (deviceId="ghi...") → 相位偏移 12,345ms
// → 三個設備的心跳自然分散在不同時間點
```

**確定性的好處**：
1. **可預測**：運維人員可以根據設備 ID 計算心跳時間
2. **穩定**：重啟後心跳時間不變
3. **均勻分散**：SHA-256 的均勻分布特性確保相位偏移均勻分散
4. **無協調**：不需要任何中央服務來分配心跳時間

---

## 3. 引用來源

以下是本章引用的所有原始碼檔案參考：

| 功能群組 | 檔案 | 行數 | 關鍵匯出 |
|----------|------|------|----------|
| 錯誤與韌性 | `source-repo/src/infra/errors.ts` | — | `extractErrorCode()`, `readErrorName()`, `collectErrorGraphCandidates()`, `isErrno()`, `hasErrnoCode()`, `formatErrorMessage()`, `detectErrorKind()` |
| 錯誤與韌性 | `source-repo/src/infra/backoff.ts` | 29 行 | `computeBackoff()`, `sleepWithAbort()` |
| 錯誤與韌性 | `source-repo/src/infra/retry.ts` | 138 行 | `retryAsync()`, `resolveRetryConfig()`, `applyJitter()`, `RetryConfig`, `RetryInfo`, `RetryOptions` |
| 錯誤與韌性 | `source-repo/src/infra/retry-policy.ts` | 119 行 | `createRateLimitRetryRunner()`, `createChannelApiRetryRunner()`, `CHANNEL_API_RETRY_DEFAULTS` |
| 錯誤與韌性 | `source-repo/src/infra/abort-signal.ts` | — | AbortSignal 工具 |
| 錯誤與韌性 | `source-repo/src/infra/fetch.ts` | — | 增強型 fetch |
| 錯誤與韌性 | `source-repo/src/infra/unhandled-rejections.ts` | — | 未處理 Promise 拒絕 |
| 進程管理 | `source-repo/src/infra/gateway-lock.ts` | 345 行 | `acquireGatewayLock()` |
| 進程管理 | `source-repo/src/infra/gateway-processes.ts` | 60 行 | `readGatewayProcessArgsSync()`, `signalVerifiedGatewayPidSync()`, `findVerifiedGatewayListenerPidsOnPortSync()` |
| 進程管理 | `source-repo/src/infra/gateway-process-argv.ts` | 65 行 | `parseProcCmdline()`, `parseWindowsCmdline()`, `isGatewayArgv()` |
| 進程管理 | `source-repo/src/infra/restart.ts` | 524 行 | `emitGatewayRestart()`, `scheduleGatewaySigusr1Restart()`, `deferGatewayRestartUntilIdle()`, `triggerOpenClawRestart()`, `setPreRestartDeferralCheck()` |
| 進程管理 | `source-repo/src/infra/restart-sentinel.ts` | 147 行 | `writeRestartSentinel()`, `readRestartSentinel()`, `consumeRestartSentinel()` |
| 進程管理 | `source-repo/src/infra/process-respawn.ts` | 68 行 | `restartGatewayProcessWithFreshPid()` |
| 進程管理 | `source-repo/src/infra/supervisor-markers.ts` | 44 行 | `detectRespawnSupervisor()` |
| 網路通訊 | `source-repo/src/infra/bonjour.ts` | — | mDNS 服務註冊 |
| 網路通訊 | `source-repo/src/infra/bonjour-discovery.ts` | — | mDNS 服務探索（dns-sd / avahi-browse） |
| 網路通訊 | `source-repo/src/infra/ssh-tunnel.ts` | — | SSH 隧道管理 |
| 網路通訊 | `source-repo/src/infra/tailnet.ts` | 47 行 | `isTailnetIPv4()`, `listTailnetAddresses()` |
| 網路通訊 | `source-repo/src/infra/widearea-dns.ts` | 199 行 | DNS 區域檔案生成 |
| 網路通訊 | `source-repo/src/infra/jsonl-socket.ts` | 64 行 | JSONL Unix Socket 協定 |
| 網路通訊 | `source-repo/src/infra/ws.ts` | 22 行 | WebSocket RawData 轉換 |
| 網路通訊 | `source-repo/src/infra/ports.ts` | 96 行 | `ensurePortAvailable()`, `describePortOwner()` |
| 心跳監控 | `source-repo/src/infra/heartbeat-runner.ts` | — | 心跳執行引擎 |
| 心跳監控 | `source-repo/src/infra/heartbeat-events.ts` | 70 行 | `emitHeartbeatEvent()`, `onHeartbeatEvent()` |
| 心跳監控 | `source-repo/src/infra/heartbeat-schedule.ts` | 60 行 | SHA-256 種子確定性調度 |
| 系統事件 | `source-repo/src/infra/system-events.ts` | 212 行 | `enqueueSystemEvent()`, `drainSystemEventEntries()` |
| 系統事件 | `source-repo/src/infra/system-presence.ts` | 288 行 | `updateSystemPresence()` |
| 系統事件 | `source-repo/src/infra/system-message.ts` | 21 行 | `SYSTEM_MARK` |
| 執行安全 | `source-repo/src/infra/exec-host.ts` | 81 行 | HMAC-SHA256 簽名執行 |
| 執行安全 | `source-repo/src/infra/exec-safety.ts` | 45 行 | `isSafeExecutableValue()` |
| 執行安全 | `source-repo/src/infra/exec-approvals.ts` | — | 執行核准追蹤 |
| 執行安全 | `source-repo/src/infra/shell-env.ts` | 250 行 | 登入 Shell 環境回退、ZDOTDIR 防護 |
| 執行安全 | `source-repo/src/infra/runtime-guard.ts` | 123 行 | Node.js >= 22.14.0 版本強制 |
| 更新管理 | `source-repo/src/infra/update-check.ts` | 423 行 | 版本檢查 |
| 更新管理 | `source-repo/src/infra/update-runner.ts` | — | 自動更新執行 |
| 更新管理 | `source-repo/src/infra/update-startup.ts` | 528 行 | 更新排程 |
| 設備配對 | `source-repo/src/infra/device-identity.ts` | — | Ed25519 設備身份 |
| 設備配對 | `source-repo/src/infra/device-pairing.ts` | — | Token 配對 |
| 設備配對 | `source-repo/src/infra/device-bootstrap.ts` | — | 設備引導 |
| 設備配對 | `source-repo/src/infra/node-pairing.ts` | — | 節點配對（5min TTL） |
| 檔案管理 | `source-repo/src/infra/json-file.ts` | 130 行 | 原子 JSON 寫入、符號連結追蹤、mode 0o600 |
| 檔案管理 | `source-repo/src/infra/secret-file.ts` | 141 行 | 符號連結拒絕、16KB 限制 |
| 檔案管理 | `source-repo/src/infra/archive.ts` | — | ZIP/TAR 解壓、符號連結穿越防護 |
| 檔案管理 | `source-repo/src/infra/clipboard.ts` | 26 行 | pbcopy → xclip → wl-copy → clip.exe |
| 環境設定 | `source-repo/src/infra/home-dir.ts` | 145 行 | `OPENCLAW_HOME` 支援 |
| 環境設定 | `source-repo/src/infra/openclaw-root.ts` | 136 行 | 符號連結感知套件根目錄 |
| 環境設定 | `source-repo/src/infra/state-migrations.ts` | — | 資料庫結構遷移 |
