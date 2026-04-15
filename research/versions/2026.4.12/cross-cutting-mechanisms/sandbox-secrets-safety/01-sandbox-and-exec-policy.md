# 沙箱模式與執行政策

## 本章摘要

當 AI agent 需要執行程式碼、操作檔案、甚至管理容器時，安全隔離就變得至關重要。本章深入分析 OpenClaw 的沙箱（sandbox）機制——從三級沙箱策略，到 Docker 容器的具體隔離配置，再到控制 agent 能執行什麼命令的執行政策（exec policy）系統。

---

## 1. 沙箱模式

### 1.1 三級策略

OpenClaw 定義了三個沙箱模式（`source-repo/src/config/types.agents-shared.ts:25`）：

```typescript
type SandboxMode = "off" | "non-main" | "all";
```

| 模式 | 說明 | 適用場景 |
|------|------|----------|
| `"off"` | 不啟用沙箱 | 開發環境、受信任場景 |
| `"non-main"` | 僅非主 agent 在沙箱中 | 生產環境預設——主 agent 直接執行，子代理被沙箱化 |
| `"all"` | 所有 agent 都在沙箱中 | 高安全需求環境 |

`"non-main"` 是設計上的甜蜜點：主 agent 通常需要存取本機檔案系統（例如寫入配置），但子代理的行為較不可預測，應該被隔離。

### 1.2 Agent 級沙箱設定

```typescript
// source-repo/src/config/types.agents-shared.ts:23-28
// 每個 agent 可以有自己的 sandbox 設定
// sandboxMode 控制是否啟用
// sandboxConfig 指向 Docker 沙箱配置
```

---

## 2. Docker 沙箱配置

### 2.1 完整配置結構

Docker 沙箱的配置型別定義在 `source-repo/src/config/types.sandbox.ts:3-62`：

```typescript
type DockerSandboxConfig = {
  // 基礎設定
  image?: string;                     // 容器映像
  readOnlyRoot?: boolean;             // 唯讀根檔案系統
  tmpfs?: string[];                   // tmpfs 掛載點
  network?: string;                   // Docker 網路模式
  capDrop?: string[];                 // 移除的 Linux capabilities

  // 資源限制
  memory?: string;                    // 記憶體限制 (e.g., "512m")
  cpus?: number;                      // CPU 限制
  pidsLimit?: number;                 // PID 數量限制

  // 安全配置檔
  seccompProfile?: string;            // Seccomp 設定檔路徑
  apparmorProfile?: string;           // AppArmor 設定檔名稱

  // 掛載
  binds?: string[];                   // Bind mount 列表
  volumes?: string[];                 // Volume mount 列表

  // 危險覆蓋旗標
  dangerouslyAllowReservedContainerTargets?: boolean;
  dangerouslyAllowExternalBindSources?: boolean;
  dangerouslyAllowContainerNamespaceJoin?: boolean;
};
```

### 2.2 唯讀根檔案系統

`readOnlyRoot: true` 將容器的根檔案系統設為唯讀，搭配 `tmpfs` 掛載點提供可寫的臨時目錄：

```yaml
readOnlyRoot: true
tmpfs:
  - /tmp
  - /run
```

這確保 agent 無法修改系統檔案，只能寫入指定的臨時目錄。

### 2.3 Capability Drop

```typescript
// capDrop 移除 Linux capabilities
// 典型配置會 drop ALL 然後只 add 必要的
capDrop: ["ALL"]
```

Linux capabilities 是將傳統的 root 超級權限分拆成更細粒度的權限單元。`capDrop: ["ALL"]` 移除所有特權能力，是容器安全的最佳實踐。

### 2.4 危險覆蓋旗標

三個 `dangerously*` 旗標是安全逃生艙：

| 旗標 | 說明 | 風險 |
|------|------|------|
| `dangerouslyAllowReservedContainerTargets` | 允許掛載到保留路徑（如 `/etc`、`/usr`） | 可能覆蓋系統配置 |
| `dangerouslyAllowExternalBindSources` | 允許從容器外部路徑 bind mount | 可能存取宿主機敏感檔案 |
| `dangerouslyAllowContainerNamespaceJoin` | 允許加入其他容器的 namespace | 可能突破容器隔離 |

命名中的 `dangerously` 前綴是**有意識的摩擦設計**——讓開發者在啟用時必須明確意識到風險。

---

## 3. 沙箱容器映像

### 3.1 映像層級

OpenClaw 提供三個 Dockerfile 用於建構沙箱映像：

**基礎映像**（`source-repo/Dockerfile.sandbox`）：
```dockerfile
# 最小化的 Alpine/Debian 基礎映像
# 僅包含執行 agent 所需的最小依賴
```

**瀏覽器映像**（`source-repo/Dockerfile.sandbox-browser`）：
```dockerfile
# 基於基礎映像，額外安裝 Chromium
# 用於需要瀏覽器能力的 agent（如網頁擷取、截圖）
```

**開發工具映像**（`source-repo/Dockerfile.sandbox-common`）：
```dockerfile
# 基於基礎映像，額外安裝常用開發工具
# git, curl, jq, python 等
```

### 3.2 Docker Socket 掛載

```yaml
# source-repo/docker-compose.yml 中的 socket 掛載
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

Docker socket 掛載讓 OpenClaw 可以管理沙箱容器，但這本身是一個安全考量——擁有 Docker socket 存取權等同於 root 權限。

---

## 4. 執行政策（Exec Policy）

### 4.1 執行安全等級

```typescript
// source-repo/src/infra/exec-approvals.ts:20-30
type ExecSecurity = "deny" | "allowlist" | "full";
```

| 等級 | 說明 |
|------|------|
| `"deny"` | 拒絕所有命令執行 |
| `"allowlist"` | 僅允許白名單中的命令 |
| `"full"` | 允許所有命令 |

### 4.2 執行詢問策略

```typescript
// source-repo/src/infra/exec-approvals.ts:32-38
type ExecAsk = "off" | "on-miss" | "always";
```

| 策略 | 說明 |
|------|------|
| `"off"` | 從不詢問使用者 |
| `"on-miss"` | 白名單未命中時才詢問 |
| `"always"` | 每次執行都詢問 |

### 4.3 完整的審批型別

```typescript
// source-repo/src/infra/exec-approvals.ts:40-58
type ExecApprovalConfig = {
  security: ExecSecurity;
  ask: ExecAsk;
  allowedCommands?: string[];
  allowedPaths?: string[];
  trustDirs?: string[];
};
```

### 4.4 政策預設

CLI 提供三種預設政策（`source-repo/src/cli/exec-policy-cli.ts:37-56`）：

```typescript
// "yolo" 預設
{ security: "full", ask: "off" }
// 完全信任 agent，不詢問，不限制

// "cautious" 預設
{ security: "allowlist", ask: "on-miss" }
// 僅允許白名單命令，未命中時詢問使用者

// "deny-all" 預設
{ security: "deny", ask: "off" }
// 拒絕所有命令執行
```

這三個預設的命名精確反映了安全立場：`yolo`（不在乎安全）、`cautious`（謹慎）、`deny-all`（全面拒絕）。

### 4.5 安全二進位信任

```typescript
// source-repo/src/infra/exec-safe-bin-trust.ts:1-15
// DEFAULT_SAFE_BIN_TRUST_DIRS = ["/bin", "/usr/bin"]
// 只信任來自這些目錄的二進位檔
```

這是一個重要的安全邊界——即使在 `"allowlist"` 模式下，也只信任來自已知系統目錄的可執行檔。這防止了 agent 執行使用者目錄中的惡意腳本。

### 4.6 執行環境注入

```typescript
// source-repo/src/infra/exec-approvals.ts:45-58
// ExecContext 包含：
// - cwd: 工作目錄
// - env: 環境變數
// - timeout: 超時限制
```

---

## 5. 沙箱的橫切模式

沙箱機制橫切多個系統層面：

```
Agent 收到工具呼叫（如 "execute_command"）
  │
  ├── Exec Policy 評估
  │   ├── security == "deny" → 拒絕
  │   ├── security == "allowlist"
  │   │   ├── 命令在白名單 → 允許
  │   │   └── 命令不在白名單 → ask 策略
  │   │       ├── ask == "off" → 拒絕
  │   │       ├── ask == "on-miss" → 詢問使用者
  │   │       └── ask == "always" → 詢問使用者
  │   └── security == "full" → 允許
  │
  ├── Sandbox Mode 評估
  │   ├── mode == "off" → 直接在宿主機執行
  │   ├── mode == "non-main"
  │   │   ├── 主 agent → 宿主機
  │   │   └── 子代理 → 容器
  │   └── mode == "all" → 容器
  │
  ├── Container 設定（如果需要容器）
  │   ├── 選擇映像
  │   ├── 套用 readOnlyRoot, capDrop, seccomp
  │   ├── 設定資源限制
  │   └── 掛載必要的 binds/volumes
  │
  └── 執行命令
      ├── Safe Bin Trust 驗證
      └── 回傳結果
```

---

## 引用來源

| 來源 | 說明 |
|------|------|
| `source-repo/src/config/types.agents-shared.ts:23-28` | 沙箱模式定義 |
| `source-repo/src/config/types.sandbox.ts:3-62` | Docker 沙箱完整配置 |
| `source-repo/Dockerfile.sandbox` | 基礎沙箱映像 |
| `source-repo/Dockerfile.sandbox-browser` | 瀏覽器沙箱映像 |
| `source-repo/Dockerfile.sandbox-common` | 開發工具沙箱映像 |
| `source-repo/docker-compose.yml` | Docker socket 掛載 |
| `source-repo/src/infra/exec-approvals.ts:20-58` | 執行審批型別 |
| `source-repo/src/cli/exec-policy-cli.ts:37-56` | 政策預設 |
| `source-repo/src/infra/exec-safe-bin-trust.ts:1-15` | 安全二進位信任 |
