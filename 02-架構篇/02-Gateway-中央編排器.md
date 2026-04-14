# Gateway——OpenClaw 的中央編排器

> **文件版本**：v1.0
> **適用範圍**：OpenClaw Gateway 模組設計
> **讀者對象**：系統架構師、後端開發者、資安工程師、頻道整合開發者

---

## 目錄

1. [引言](#引言)
2. [Gateway 的角色與責任](#gateway-的角色與責任)
3. [WebSocket 伺服器](#websocket-伺服器)
4. [連線管理與裝置身份驗證](#連線管理與裝置身份驗證)
5. [路由機制](#路由機制)
6. [Hub/Spoke 模型的優勢](#hubspoke-模型的優勢)
7. [頻道適配器解耦](#頻道適配器解耦)
8. [JSON Schema 驗證的安全意義](#json-schema-驗證的安全意義)
9. [設備配對流程（Pairing）](#設備配對流程pairing)
10. [錯誤處理與容錯機制](#錯誤處理與容錯機制)
11. [配置參數詳解](#配置參數詳解)
12. [總結](#總結)

---

## 引言

在任何分散式系統中，都需要一個可靠的「中樞」來協調各個元件之間的通訊和協作。在 OpenClaw 的架構中，Gateway 就扮演了這個角色。它是系統的唯一入口點，所有外部訊息必須經過 Gateway 的驗證和路由才能進入系統內部，所有系統回應也必須經過 Gateway 才能送達使用者。

Gateway 的設計深受「單一職責原則」（Single Responsibility Principle）的影響——它的核心職責就是「訊息的接收、驗證、路由和分派」。它不負責推理、不負責記憶、不負責技能執行。它就像一座大廈的門廳——所有人都必須從這裡進出，但門廳本身不處理任何業務。

本文件將深入探討 Gateway 模組的每個設計面向，從 WebSocket 伺服器的實作到連線管理、路由機制、安全策略、容錯設計和配置參數。無論你是想要理解 OpenClaw 的內部運作，還是想要開發自己的頻道適配器，本文件都將為你提供必要的知識。

理解 Gateway 的設計對於理解整個 OpenClaw 系統至關重要。作為系統的「神經中樞」，Gateway 的設計決策直接影響了系統的可靠性、安全性和擴展性。讓我們從 Gateway 的角色和責任開始，逐步深入其內部架構。

---

## Gateway 的角色與責任

### 角色定位

Gateway 在 OpenClaw 架構中有三重角色：

**第一重：訊息閘道（Message Gate）**

顧名思義，Gateway 是所有訊息進出系統的唯一「閘門」。這個設計有幾個重要的安全意涵：

1. **單一攻擊面**：攻擊者只能透過 Gateway 的 WebSocket 介面嘗試入侵，無法直接存取 Brain、Memory 或 Skills 模組。
2. **統一存取控制**：所有存取控制策略都在 Gateway 層實施，確保不會有模組被「繞過」。
3. **完整審計**：所有進出系統的訊息都可以在 Gateway 層被記錄，形成完整的審計軌跡。

```
外部世界                    OpenClaw 內部
                    ┌──────────────┐
Telegram ──────────▶│              │──────▶ Brain
LINE     ──────────▶│   Gateway    │──────▶ Memory
Discord  ──────────▶│   (閘門)     │──────▶ Skills
CLI      ──────────▶│              │──────▶ Heartbeat
                    └──────────────┘
                    ▲              ▲
                    │              │
                  入站驗證      出站格式化
```

**第二重：訊息路由器（Message Router）**

Gateway 負責將入站訊息路由到正確的內部處理器。這個路由機制基於訊息類型和內容，確保每個訊息都被正確的模組處理。

**第三重：連線管理器（Connection Manager）**

Gateway 管理所有 WebSocket 連線的完整生命週期——從連線建立到身份驗證、從心跳保活到優雅斷開。它維護一個活躍連線池，追蹤每個連線的狀態和健康資訊。

### 責任範圍

Gateway 的責任範圍是嚴格界定的：

**Gateway 負責的事情：**
- 啟動和維護 WebSocket 伺服器
- 接受和管理客戶端連線
- 驗證入站訊息的格式（JSON Schema）
- 驗證客戶端的身份（Authentication）
- 將訊息路由到正確的內部處理器
- 將處理結果路由回正確的客戶端
- 記錄所有訊息的審計日誌
- 管理連線的健康狀態
- 處理錯誤回覆和超時
- 速率限制和流量控制

**Gateway 不負責的事情：**
- LLM 推理（Brain 的職責）
- 記憶儲存和檢索（Memory 的職責）
- 技能執行（Skills 的職責）
- 排程管理（Heartbeat 的職責）
- 頻道特定的訊息格式轉換（適配器的職責）
- 業務邏輯處理

這種嚴格的職責劃分確保了 Gateway 的程式碼保持精簡和可維護。Gateway 的核心邏輯應該能在幾千行程式碼內實現，避免成為「上帝物件」（God Object）。

### 設計原則

Gateway 的設計遵循以下原則：

**無狀態原則（Stateless Processing）**

Gateway 本身不持有業務狀態。它不記住對話歷史、不快取 LLM 回應、不儲存使用者偏好。它只維護連線狀態——即哪些客戶端正在連線、它們的身份是什麼。

**快速失敗原則（Fail Fast）**

如果入站訊息格式不正確、身份驗證失敗、或目標處理器不存在，Gateway 會立即返回錯誤，而不是嘗試修復或猜測意圖。

**透明代理原則（Transparent Proxy）**

Gateway 盡可能不修改經過它的訊息內容。它的工作是驗證和路由，而非轉換或處理。這使得系統的行為更加可預測和可除錯。

---

## WebSocket 伺服器

### 為什麼選擇 WebSocket

OpenClaw 選擇 WebSocket（RFC 6455）作為 Gateway 的核心通訊協議，這是經過仔細考量的設計決策。

**全雙工通訊的必要性**

AI 助理的互動模式不同於傳統的 Request-Response 模式。以下場景需要伺服器主動向客戶端推送訊息：

1. **串流回應**：LLM 生成回應時，文字是逐步產生的。使用者期望看到「打字中」的效果，而非等待整個回應完成後才看到結果。WebSocket 允許 Brain 模組在生成回應的過程中，透過 Gateway 即時推送每個 token 或句子片段。

2. **Heartbeat 通知**：當 Heartbeat 模組在排程時間觸發了某個重要任務（例如發現異常或完成報告），需要主動通知使用者。WebSocket 的全雙工特性使這成為可能。

3. **多步驟任務進度**：當 Brain 執行 ReAct 迴圈時（思考→行動→觀察→思考→...），每個步驟的進度都可以即時推送給使用者。

4. **系統狀態更新**：Gateway 可以即時推送系統狀態變化（如新的適配器連線、技能載入完成等）。

**與 HTTP 的對比**

```
HTTP Request/Response 模式：
Client ──Request──▶ Server
Client ◀─Response── Server
Client ──Request──▶ Server    （每次都需要新的請求）
Client ◀─Response── Server

WebSocket 全雙工模式：
Client ◀══════════▶ Server    （建立一次連線，雙向通訊）
Client ──Message──▶ Server
Client ◀─Message── Server
Client ◀─Push───── Server    （伺服器主動推送）
Client ──Message──▶ Server
```

**效能考量**

WebSocket 連線建立後，每個訊息的傳輸開銷僅為 2-14 bytes 的幀頭（Frame Header），相比 HTTP 每次請求都需要攜帶完整的 HTTP 頭部（通常數百 bytes），效率提升顯著。對於頻繁互動的 AI 助理場景，這意味著更低的延遲和更少的頻寬消耗。

### 伺服器配置

Gateway 的 WebSocket 伺服器預設監聽在 `ws://localhost:18789`。以下是伺服器的詳細配置：

```yaml
# ~/.openclaw/config/gateway.yaml

server:
  # 監聽地址
  host: "127.0.0.1"        # 預設只監聽 localhost
  port: 18789               # 預設連接埠
  
  # WebSocket 配置
  websocket:
    path: "/ws"              # WebSocket 端點路徑
    max_payload_size: 1048576  # 最大訊息大小（1MB）
    ping_interval: 30        # 心跳間隔（秒）
    ping_timeout: 10         # 心跳超時（秒）
    close_timeout: 5         # 關閉握手超時（秒）
    
  # TLS 配置（可選）
  tls:
    enabled: false
    cert_file: ""
    key_file: ""
    
  # 併行限制
  concurrency:
    max_connections: 50      # 最大同時連線數
    max_pending_messages: 100  # 每個連線的最大待處理訊息數
```

**安全綁定策略**

預設情況下，Gateway 只監聽 `127.0.0.1`（localhost），這意味著只有本機的程式可以連接到 Gateway。這是一個重要的安全決策：

- 頻道適配器通常運行在同一台機器上，只需要 localhost 連線
- 不暴露任何網路介面到外部網路
- 如果需要遠端存取，應該透過 SSH 通道或 VPN，而非直接開放 Gateway 端口

若需要在區域網路內共享 Gateway（例如讓手機上的適配器連接），可以將 host 改為 `0.0.0.0`，但強烈建議同時啟用 TLS 和強認證。

### 連線埠選擇

選擇 `18789` 作為預設連線埠的考量：
- 避開常用的系統連線埠（0-1023）和常見應用連線埠
- 數字容易記憶
- 不與已知的主流服務衝突
- 可以透過配置檔自由更改

### 啟動流程

WebSocket 伺服器的啟動流程如下：

```
1. 載入配置檔
   └─ 讀取 ~/.openclaw/config/gateway.yaml
   └─ 驗證配置參數合法性
   └─ 設定預設值（如果配置項缺失）

2. 初始化安全元件
   └─ 載入 TLS 憑證（如果啟用）
   └─ 初始化認證模組
   └─ 載入授權裝置清單

3. 初始化內部模組
   └─ 建立 Brain 引擎實例
   └─ 建立 Memory 管理器實例
   └─ 建立 Heartbeat 排程器實例
   └─ 建立 Skills 管理器實例

4. 啟動 WebSocket 伺服器
   └─ 綁定地址和連線埠
   └─ 開始接受連線
   └─ 啟動心跳計時器

5. 啟動 Heartbeat
   └─ 讀取 HEARTBEAT.md 配置
   └─ 初始化排程計時器

6. 報告就緒
   └─ 輸出啟動日誌
   └─ 設定 PID 檔
   └─ 發送 systemd 就緒通知（如果適用）
```

```bash
$ openclaw start
[2025-01-16 10:00:01] INFO  Gateway starting...
[2025-01-16 10:00:01] INFO  Loading configuration from ~/.openclaw/config/gateway.yaml
[2025-01-16 10:00:01] INFO  Initializing Brain engine (provider: anthropic)
[2025-01-16 10:00:01] INFO  Initializing Memory manager (storage: ~/.openclaw/memory/)
[2025-01-16 10:00:02] INFO  Initializing Skills manager (skills: 12 loaded)
[2025-01-16 10:00:02] INFO  Initializing Heartbeat scheduler (interval: 30m)
[2025-01-16 10:00:02] INFO  WebSocket server listening on ws://127.0.0.1:18789/ws
[2025-01-16 10:00:02] INFO  Gateway ready. PID: 12345
```

---

## 連線管理與裝置身份驗證

### 連線生命週期

每個 WebSocket 連線都經歷以下生命週期階段：

```
┌─────────┐     ┌────────────┐     ┌───────────┐     ┌──────────┐
│ 連線建立  │────▶│  身份驗證   │────▶│  活躍通訊  │────▶│  連線關閉 │
│ (Connect)│     │  (Auth)    │     │  (Active) │     │  (Close) │
└─────────┘     └────────────┘     └───────────┘     └──────────┘
     │               │                   │                 │
     │          驗證失敗                  │            異常斷開
     │               │                   │                 │
     │               ▼                   │                 ▼
     │          ┌──────────┐             │          ┌──────────┐
     │          │  拒絕連線  │             │          │  重連等待 │
     │          └──────────┘             │          └──────────┘
     │                                   │                 │
     │                               心跳超時              │
     │                                   │                 │
     │                                   ▼                 │
     │                            ┌──────────┐             │
     │                            │  標記失效  │             │
     │                            └──────────┘             │
```

**階段 1：連線建立（Connect）**

當客戶端（頻道適配器）發起 WebSocket 連線時，Gateway 執行以下動作：

1. 接受 TCP 連線
2. 完成 WebSocket 握手（HTTP Upgrade）
3. 分配連線 ID
4. 將連線加入「未認證」連線池
5. 啟動認證超時計時器（預設 10 秒）

```json
// Gateway 內部連線記錄
{
  "connection_id": "conn_20250116_001",
  "remote_addr": "127.0.0.1:54321",
  "state": "pending_auth",
  "connected_at": "2025-01-16T10:00:05Z",
  "auth_deadline": "2025-01-16T10:00:15Z"
}
```

**階段 2：身份驗證（Auth）**

連線建立後，客戶端必須在超時時間內發送認證訊息。Gateway 支援多種認證方式：

```json
// 認證請求（Token 認證）
{
  "type": "auth.request",
  "version": "1.0",
  "payload": {
    "method": "token",
    "token": "oclaw_tk_a1b2c3d4e5f6...",
    "client_info": {
      "type": "adapter",
      "name": "telegram-adapter",
      "version": "1.2.0"
    }
  }
}
```

```json
// 認證請求（設備配對碼認證）
{
  "type": "auth.request",
  "version": "1.0",
  "payload": {
    "method": "pairing_code",
    "code": "ABCD-1234",
    "device_info": {
      "type": "mobile",
      "name": "Eason's iPhone",
      "os": "iOS 18.0"
    }
  }
}
```

**階段 3：活躍通訊（Active）**

認證成功後，連線進入活躍狀態。在這個階段：
- 客戶端可以發送任何已授權的訊息類型
- Gateway 定期發送 WebSocket ping 幀維持連線
- 雙方都可以隨時發送訊息

**階段 4：連線關閉（Close）**

連線關閉可能由以下原因觸發：
- 客戶端主動關閉
- 心跳超時（客戶端未回應 ping）
- 認證超時（連線後未完成認證）
- 伺服器關閉
- 訊息格式嚴重違規（安全防護）

### 身份驗證機制

Gateway 實作了分層的身份驗證機制：

**第一層：傳輸層安全（Transport Security）**

- 預設綁定 localhost，只接受本機連線
- 可選的 TLS 加密（ws:// 升級為 wss://）
- 來源 IP 白名單（可配置）

**第二層：應用層認證（Application Authentication）**

Gateway 支援三種認證方式：

1. **Token 認證**：最常用的方式。每個授權的客戶端都有一個唯一的 API Token。Token 儲存在 `~/.openclaw/config/tokens.yaml` 中。

```yaml
# ~/.openclaw/config/tokens.yaml
tokens:
  - id: "telegram-adapter"
    token: "oclaw_tk_a1b2c3d4e5f6..."
    permissions:
      - "message.send"
      - "message.receive"
    created_at: "2025-01-15T00:00:00Z"
    
  - id: "line-adapter"
    token: "oclaw_tk_g7h8i9j0k1l2..."
    permissions:
      - "message.send"
      - "message.receive"
    created_at: "2025-01-15T00:00:00Z"
    
  - id: "admin-cli"
    token: "oclaw_tk_m3n4o5p6q7r8..."
    permissions:
      - "message.send"
      - "message.receive"
      - "system.admin"
    created_at: "2025-01-15T00:00:00Z"
```

2. **配對碼認證（Pairing Code）**：適用於新裝置的首次連線。Gateway 顯示一個一次性配對碼，使用者在新裝置上輸入此碼完成配對。

3. **本地信任認證（Local Trust）**：對於運行在同一台機器上的客戶端（透過 Unix Domain Socket 或 localhost 連線），可以選擇性地信任而不需要額外的認證。這種模式僅適用於開發和測試環境。

**第三層：權限控制（Authorization）**

認證成功後，Gateway 根據客戶端的權限列表決定其可以發送的訊息類型：

```
權限                     允許的操作
─────────────────────   ──────────────────────────
message.send            發送使用者訊息
message.receive         接收系統回應
skill.invoke            直接調用技能
system.admin            管理操作（重啟、配置等）
system.status           查詢系統狀態
memory.read             讀取記憶
memory.write            寫入記憶
```

### 連線池管理

Gateway 維護一個連線池（Connection Pool），追蹤所有活躍連線：

```
連線池狀態範例：

┌────────────────────────────────────────────────────────┐
│                    Connection Pool                       │
├──────────┬───────────┬──────────┬──────────┬───────────┤
│ Conn ID  │ Client    │ State    │ Auth     │ Last Ping │
├──────────┼───────────┼──────────┼──────────┼───────────┤
│ conn_001 │ telegram  │ active   │ token    │ 5s ago    │
│ conn_002 │ line      │ active   │ token    │ 12s ago   │
│ conn_003 │ cli       │ active   │ local    │ 2s ago    │
│ conn_004 │ unknown   │ pending  │ -        │ -         │
│ conn_005 │ discord   │ closing  │ token    │ 45s ago   │
└──────────┴───────────┴──────────┴──────────┴───────────┘
```

當連線數達到上限（`max_connections`）時，Gateway 會拒絕新的連線請求，並返回適當的錯誤碼。同時，Gateway 會定期清理長時間無活動的連線和處於異常狀態的連線。

---

## 路由機制

### 路由架構

Gateway 的路由機制是其核心功能之一。它決定了每個入站訊息應該被哪個內部模組處理。路由機制由三個層次組成：

```
入站訊息
    │
    ▼
┌──────────────────────┐
│ 第一層：類型路由       │  根據 message.type 分派
│ (Type Router)        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 第二層：內容路由       │  根據訊息內容進一步分派
│ (Content Router)     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 第三層：策略路由       │  根據配置的策略決定處理方式
│ (Policy Router)      │
└──────────────────────┘
```

### 第一層：類型路由

每個 OpenClaw 訊息都有一個 `type` 欄位，類型路由器根據此欄位將訊息分派到對應的處理管道：

```
message.incoming  ──▶  對話處理管道 ──▶ Brain
skill.invoke      ──▶  技能處理管道 ──▶ Skills
memory.query      ──▶  記憶處理管道 ──▶ Memory
system.status     ──▶  系統處理管道 ──▶ 內部處理器
auth.request      ──▶  認證處理管道 ──▶ 認證模組
heartbeat.*       ──▶  排程處理管道 ──▶ Heartbeat
```

類型路由是一個快速的映射操作（O(1) 查找），不涉及複雜的邏輯判斷。

### 第二層：內容路由

某些訊息類型需要根據內容進行進一步的路由。最典型的例子是 `message.incoming`：

- 以 `/` 開頭的文字 → 系統指令處理器
- 包含 `@openclaw` 提及 → 高優先級對話處理
- 純文字訊息 → 一般對話處理（Brain）
- 包含媒體（圖片、檔案） → 多媒體處理器

```python
# 內容路由邏輯示意（偽代碼）
def route_incoming_message(message):
    text = message.payload.content.text
    
    if text.startswith("/"):
        return handle_command(text)
    
    if message.payload.content.type == "image":
        return handle_media(message)
    
    if is_high_priority(message):
        return brain.process(message, priority="high")
    
    return brain.process(message, priority="normal")
```

### 第三層：策略路由

策略路由允許使用者配置自定義的路由規則：

```yaml
# ~/.openclaw/config/routing.yaml

routing_rules:
  # 來自特定頻道的訊息使用特定的處理策略
  - match:
      channel: "telegram"
    action:
      max_response_length: 4096  # Telegram 的訊息長度限制
      
  - match:
      channel: "cli"
    action:
      streaming: true  # CLI 支援串流輸出
      
  # 特定時間段的降噪策略
  - match:
      time_range: "23:00-07:00"
    action:
      defer_non_urgent: true  # 非緊急訊息延遲到早上
      
  # 特定使用者的自定義處理
  - match:
      sender_id: "user_admin"
    action:
      permissions: ["system.admin"]
```

### 路由表

Gateway 維護一個路由表，記錄所有已註冊的處理器和它們的能力：

```
┌──────────────────────────────────────────────────────────┐
│                     Routing Table                          │
├──────────────┬───────────────────┬────────────────────────┤
│ Message Type │ Handler           │ Priority / Constraints  │
├──────────────┼───────────────────┼────────────────────────┤
│ msg.incoming │ brain.process     │ normal, async           │
│ msg.outgoing │ adapter.dispatch  │ high, route-to-source   │
│ skill.invoke │ skills.execute    │ normal, gated           │
│ skill.result │ brain.continue    │ high, resume-react      │
│ memory.query │ memory.search     │ normal, cached          │
│ memory.write │ memory.store      │ low, batched            │
│ hb.trigger   │ heartbeat.run     │ low, rate-limited       │
│ sys.status   │ internal.status   │ high, immediate         │
│ auth.request │ auth.verify       │ critical, timeout=10s   │
└──────────────┴───────────────────┴────────────────────────┘
```

### 訊息佇列

當系統負載較高或某個處理器暫時繁忙時，Gateway 使用內部訊息佇列來緩衝待處理的訊息：

```
入站訊息 ──▶ 驗證 ──▶ ┌──────────────┐ ──▶ 處理器
                       │ 訊息佇列      │
                       │              │
                       │ [msg_001] ◀──│── 高優先級
                       │ [msg_002]    │
                       │ [msg_003]    │
                       │ [msg_004] ◀──│── 低優先級
                       └──────────────┘
```

佇列的關鍵配置：
- **最大佇列深度**：每個處理器的佇列最多容納的訊息數量
- **優先級**：高優先級訊息（如系統指令）會被優先處理
- **超時**：佇列中停留過久的訊息會被丟棄並返回超時錯誤
- **背壓**：當佇列滿時，Gateway 會通知客戶端降低發送速率

---

## Hub/Spoke 模型的優勢

### 拓撲結構說明

OpenClaw 採用的 Hub/Spoke 拓撲是一種星型網路架構。Gateway 作為中央 Hub，所有頻道適配器和內部模組作為 Spoke 連接到 Hub。

```
                 Spoke 1                 Spoke 2
              ┌──────────┐           ┌──────────┐
              │ Telegram │           │   LINE   │
              │ Adapter  │           │ Adapter  │
              └────┬─────┘           └────┬─────┘
                   │                      │
                   │    ┌──────────┐      │
                   └────┤          ├──────┘
                        │ Gateway  │
                   ┌────┤  (Hub)   ├──────┐
                   │    └──────────┘      │
                   │                      │
              ┌────┴─────┐           ┌────┴─────┐
              │ Discord  │           │   CLI    │
              │ Adapter  │           │ Adapter  │
              └──────────┘           └──────────┘
                 Spoke 3                 Spoke 4
```

### 優勢分析

**優勢 1：故障隔離（Fault Isolation）**

在 Hub/Spoke 拓撲中，任何一個 Spoke 的故障都不會影響其他 Spoke。例如：

- Telegram API 發生故障 → Telegram 適配器斷開 → LINE 和 CLI 適配器完全不受影響
- LINE 適配器記憶體洩漏 → LINE 適配器崩潰 → 其他適配器和核心系統正常運作
- Discord 適配器出現安全漏洞 → 只影響 Discord 連線 → 系統核心安全不受威脅

這種隔離性在實際運行中非常重要。外部通訊頻道的 API 經常發生變動或故障，如果一個頻道的問題能擴散到整個系統，將嚴重影響系統的可靠性。

```
故障隔離示意：

正常狀態：                  故障狀態：
✅ Telegram ──┐            ❌ Telegram ──╳
✅ LINE     ──┤── Hub      ✅ LINE     ──┤── Hub ✅
✅ Discord  ──┘            ✅ Discord  ──┘

Telegram 故障不影響 LINE 和 Discord
```

**優勢 2：獨立生命週期（Independent Lifecycle）**

每個 Spoke 都有獨立的生命週期，可以獨立地啟動、停止、更新和重啟：

```bash
# 只重啟 Telegram 適配器
$ openclaw adapter restart telegram

# 更新 LINE 適配器到新版本
$ openclaw adapter upgrade line --version 2.0.0

# 新增 Discord 適配器
$ openclaw adapter install discord
$ openclaw adapter start discord

# 以上操作都不需要重啟 Gateway
```

**優勢 3：統一的安全邊界（Unified Security Boundary）**

所有流量都必須經過 Hub（Gateway），這意味著安全策略只需要在一個地方實施。這避免了在每個適配器中重複實作安全邏輯，降低了出現安全漏洞的風險。

**優勢 4：簡化的通訊拓撲（Simplified Communication）**

在全網狀（Full Mesh）拓撲中，N 個節點需要 N×(N-1)/2 條連線。在 Hub/Spoke 拓撲中，N 個 Spoke 只需要 N 條連線。對於 OpenClaw 的規模，這意味著更少的連線管理開銷和更簡單的系統架構。

```
全網狀拓撲（4 節點 = 6 條連線）：    Hub/Spoke（4 Spoke = 4 條連線）：
    A ─── B                              A
    │ ╲ ╱ │                              │
    │  ╳  │                         B ── Hub ── C
    │ ╱ ╲ │                              │
    C ─── D                              D
```

**優勢 5：集中式監控（Centralized Monitoring）**

Gateway 作為 Hub，可以觀察到所有流經系統的訊息。這使得：
- 全域效能監控變得簡單
- 異常偵測可以在一個地方完成
- 審計日誌的收集不需要從多個來源彙整
- 系統狀態的全貌可以從單一端點獲取

**優勢 6：靈活的擴展（Flexible Scaling）**

需要支援新的通訊頻道？只需要開發一個新的適配器並連接到 Gateway。不需要修改任何現有的適配器或核心系統程式碼。這使得社群貢獻新的頻道支援變得非常簡單。

---

## 頻道適配器解耦

### 解耦設計的核心理念

頻道適配器（Channel Adapter）的設計是 OpenClaw 架構中最重要的解耦實踐之一。核心理念是：**頻道適配器的崩潰不應該影響系統核心，系統核心的升級不應該要求更新所有適配器**。

這種解耦透過以下機制實現：

**明確定義的介面契約**

適配器與 Gateway 之間的通訊完全透過 WebSocket 和標準化的 JSON 訊息進行。適配器不需要知道 Gateway 的內部實作，Gateway 也不需要知道適配器如何與外部頻道互動。

```
┌──────────────────────┐     ┌──────────────────────┐
│   Telegram Adapter    │     │       Gateway         │
│                       │     │                       │
│ ┌───────────────────┐ │     │ ┌───────────────────┐ │
│ │ Telegram Bot API  │ │     │ │ WebSocket Server  │ │
│ │ (外部依賴)         │ │     │ │ (標準介面)        │ │
│ └────────┬──────────┘ │     │ └────────┬──────────┘ │
│          │            │     │          │            │
│ ┌────────▼──────────┐ │     │ ┌────────▼──────────┐ │
│ │ 格式轉換器        │ │ WS  │ │ 訊息路由器        │ │
│ │ TG → OpenClaw     │ ├─────┤ │                   │ │
│ │ OpenClaw → TG     │ │     │ │                   │ │
│ └───────────────────┘ │     │ └───────────────────┘ │
└──────────────────────┘     └──────────────────────┘
```

**獨立的錯誤域（Error Domain）**

每個適配器有自己的錯誤處理邏輯。Telegram API 的錯誤不會以 Telegram 特有的錯誤格式進入系統核心。適配器負責將外部錯誤轉換為標準的 OpenClaw 錯誤格式。

```json
// 適配器內部處理 Telegram 錯誤
// 外部錯誤：Telegram API 429 Too Many Requests
// 轉換為：
{
  "type": "system.error",
  "payload": {
    "source": "telegram-adapter",
    "error_code": "CHANNEL_RATE_LIMITED",
    "message": "Telegram API 請求頻率超限，將在 30 秒後重試",
    "retry_after": 30,
    "severity": "warning"
  }
}
```

**版本獨立性**

適配器和 Gateway 可以運行不同版本，只要它們都支援共同的協議版本。Gateway 在認證階段會交換協議版本資訊，確保相容性。

```json
// 版本協商
{
  "type": "auth.request",
  "version": "1.0",
  "payload": {
    "method": "token",
    "token": "...",
    "protocol_versions": ["1.0", "1.1"],
    "client_info": {
      "name": "telegram-adapter",
      "version": "2.3.0"
    }
  }
}

// Gateway 回應使用雙方都支援的最高版本
{
  "type": "auth.response",
  "version": "1.0",
  "payload": {
    "status": "ok",
    "negotiated_protocol": "1.0"
  }
}
```

### 適配器的最小實作

一個頻道適配器的最小實作只需要完成以下幾件事：

1. 建立與 Gateway 的 WebSocket 連線
2. 發送認證訊息
3. 監聽外部頻道的訊息
4. 將外部訊息轉換為 OpenClaw 格式並發送給 Gateway
5. 接收 Gateway 的回應訊息
6. 將 OpenClaw 格式的回應轉換為外部頻道格式

```python
# 適配器骨架（偽代碼）
class MinimalAdapter:
    def __init__(self, gateway_url, auth_token):
        self.ws = WebSocket(gateway_url)
        self.auth_token = auth_token
    
    async def start(self):
        await self.ws.connect()
        await self.authenticate()
        await asyncio.gather(
            self.listen_external(),
            self.listen_gateway()
        )
    
    async def authenticate(self):
        await self.ws.send({
            "type": "auth.request",
            "version": "1.0",
            "payload": {
                "method": "token",
                "token": self.auth_token
            }
        })
        response = await self.ws.receive()
        assert response["payload"]["status"] == "ok"
    
    async def listen_external(self):
        # 監聽外部頻道的訊息
        async for external_msg in self.external_channel.messages():
            openclaw_msg = self.convert_to_openclaw(external_msg)
            await self.ws.send(openclaw_msg)
    
    async def listen_gateway(self):
        # 接收 Gateway 的回應
        async for gateway_msg in self.ws.messages():
            external_msg = self.convert_from_openclaw(gateway_msg)
            await self.external_channel.send(external_msg)
    
    def convert_to_openclaw(self, external_msg):
        # 子類別實作：將外部格式轉換為 OpenClaw 格式
        raise NotImplementedError
    
    def convert_from_openclaw(self, openclaw_msg):
        # 子類別實作：將 OpenClaw 格式轉換為外部格式
        raise NotImplementedError
```

### 適配器健康監測

Gateway 對所有已連線的適配器進行持續的健康監測：

- **心跳檢查**：每 30 秒發送 WebSocket ping 幀，如果 10 秒內沒有收到 pong 回應，標記適配器為不健康。
- **訊息超時**：如果某個適配器長時間不發送任何訊息（超過配置的閾值），Gateway 會發送探測訊息確認適配器是否仍然活躍。
- **錯誤率監控**：Gateway 追蹤每個適配器的錯誤率。如果錯誤率超過閾值，Gateway 會主動斷開連線並記錄警告日誌。
- **重連行為監控**：如果某個適配器頻繁斷開和重連（「抖動」），Gateway 會實施退避策略（Backoff），逐漸延長允許的重連間隔。

```
健康監測狀態機：

  ┌──────────┐  心跳正常  ┌──────────┐
  │ Healthy  │◀──────────│ Healthy  │
  │          │──────────▶│          │
  └────┬─────┘  ping/pong └──────────┘
       │
       │ 心跳超時
       ▼
  ┌──────────┐
  │ Suspect  │  ← 可疑狀態，等待恢復
  │          │
  └────┬─────┘
       │
       │ 連續 3 次超時
       ▼
  ┌──────────┐
  │  Dead    │  ← 標記為失效，斷開連線
  │          │
  └──────────┘
```

---

## JSON Schema 驗證的安全意義

### 為什麼需要 Schema 驗證

在任何接受外部輸入的系統中，輸入驗證是安全防護的第一道防線。Gateway 對所有入站訊息執行嚴格的 JSON Schema 驗證，這不僅是為了確保資料格式正確，更是為了防止多種安全攻擊。

### 防禦的攻擊類型

**注入攻擊（Injection Attacks）**

JSON Schema 驗證可以防止攻擊者在訊息欄位中嵌入惡意內容：

```json
// 惡意訊息範例（會被 Schema 驗證攔截）
{
  "type": "message.incoming",
  "payload": {
    "content": {
      "type": "text",
      "text": "正常文字",
      "extra_field": "$(rm -rf /)",  // ← 額外欄位被拒絕
      "nested": {
        "deep": {
          "deeper": "..."  // ← 超出允許的巢狀深度
        }
      }
    }
  }
}
```

Schema 驗證確保：
- 不接受未定義的額外欄位（`additionalProperties: false`）
- 限制字串長度，防止緩衝區溢出
- 限制巢狀深度，防止堆疊溢出
- 驗證欄位的資料型別，防止型別混淆

**拒絕服務攻擊（DoS Attacks）**

```json
// DoS 攻擊嘗試（會被 Schema 驗證攔截）
{
  "type": "message.incoming",
  "payload": {
    "content": {
      "type": "text",
      "text": "A".repeat(100000000)  // ← 超出最大字串長度
    }
  }
}
```

Schema 配合 WebSocket 的 `max_payload_size` 配置，可以防止：
- 超大訊息消耗記憶體
- 超深巢狀消耗 CPU
- 超多欄位消耗解析時間

**協議違規（Protocol Violations）**

```json
// 格式不正確的訊息（會被 Schema 驗證攔截）
{
  "type": 123,              // ← type 應該是字串
  "version": "1.0",
  "payload": "not an object" // ← payload 應該是物件
}
```

Schema 驗證確保所有訊息都符合預期的格式，防止格式錯誤的訊息導致內部處理器崩潰。

### Schema 驗證的實作策略

**嚴格模式（Strict Mode）**

在生產環境中，Gateway 使用嚴格模式進行驗證：
- 所有必要欄位都必須存在
- 不接受任何未定義的額外欄位
- 所有欄位都必須符合定義的型別和格式
- 違反 Schema 的訊息會被直接拒絕

**寬鬆模式（Relaxed Mode）**

在開發環境中，可以啟用寬鬆模式：
- 允許額外的未定義欄位（方便除錯）
- 型別不匹配時嘗試自動轉換（如字串 "123" → 數字 123）
- 違反 Schema 的訊息會被記錄警告但仍然處理

```yaml
# 配置驗證模式
server:
  validation:
    mode: "strict"       # strict / relaxed
    log_violations: true  # 記錄所有驗證違規
    reject_unknown_types: true  # 拒絕未知的訊息類型
```

### 自定義 Schema 擴展

使用者可以在標準 Schema 的基礎上添加自定義的驗證規則：

```yaml
# ~/.openclaw/config/schemas/custom-rules.yaml

custom_validations:
  # 對 message.incoming 添加額外的內容長度限制
  - type: "message.incoming"
    rules:
      - path: "payload.content.text"
        max_length: 10000
        
  # 對技能調用添加參數白名單
  - type: "skill.invoke"
    rules:
      - path: "payload.skill_name"
        allowed_values: ["weather", "calendar", "git-helper"]
```

---

## 設備配對流程（Pairing）

### 配對的必要性

當使用者希望將新的裝置（如手機、平板、另一台電腦）連接到 OpenClaw 時，需要一個安全的配對流程。配對流程確保：

1. 只有使用者授權的裝置才能連接
2. 配對過程不需要手動複製長 Token
3. 配對過程有時間限制，過期的配對碼自動失效
4. 配對成功後自動生成並儲存裝置專屬的長期 Token

### 配對流程

```
步驟 1：使用者在已授權的裝置上發起配對請求
┌──────────────┐
│ 已授權裝置    │
│ (例如 CLI)   │ ──▶ openclaw pair new
└──────────────┘

步驟 2：Gateway 生成一次性配對碼
┌──────────────┐
│   Gateway    │ ──▶ 配對碼：ABCD-1234（有效期 5 分鐘）
└──────────────┘

步驟 3：使用者在新裝置上輸入配對碼
┌──────────────┐
│  新裝置      │
│ (例如手機)   │ ──▶ 輸入配對碼 ABCD-1234
└──────────────┘

步驟 4：新裝置連接 Gateway 並發送配對認證
┌──────────────┐     ┌──────────────┐
│  新裝置      │ ──▶ │   Gateway    │
│              │     │  驗證配對碼   │
└──────────────┘     └──────────────┘

步驟 5：配對成功，Gateway 為新裝置生成長期 Token
┌──────────────┐     ┌──────────────┐
│  新裝置      │ ◀── │   Gateway    │
│ 儲存 Token   │     │ 生成 Token   │
└──────────────┘     └──────────────┘

步驟 6：新裝置後續使用 Token 進行認證
┌──────────────┐     ┌──────────────┐
│  新裝置      │ ──▶ │   Gateway    │
│ Token 認證   │     │  驗證 Token  │
└──────────────┘     └──────────────┘
```

### 配對碼的安全設計

**一次性使用**：每個配對碼只能使用一次。使用後立即從有效碼列表中移除。

**時間限制**：配對碼有預設 5 分鐘的有效期。過期後自動失效，不能再使用。

**碼的格式**：使用易於人類辨識和輸入的格式（XXXX-XXXX），避免容易混淆的字元（如 0/O、1/I/l）。

**嘗試次數限制**：如果連續輸入錯誤的配對碼超過 5 次，暫時鎖定配對功能 15 分鐘。

**安全通知**：配對成功後，已授權的裝置會收到通知，告知有新裝置加入。使用者可以即時撤銷可疑的配對。

```json
// 配對成功通知
{
  "type": "system.notification",
  "payload": {
    "event": "device_paired",
    "device": {
      "name": "Eason's iPhone",
      "type": "mobile",
      "paired_at": "2025-01-16T10:05:00Z"
    },
    "action_required": false,
    "message": "新裝置 'Eason's iPhone' 已成功配對。如果這不是您的操作，請立即執行 'openclaw pair revoke' 撤銷。"
  }
}
```

### 裝置管理

配對成功後，使用者可以管理所有已配對的裝置：

```bash
# 列出所有已配對的裝置
$ openclaw devices list
┌────┬──────────────────┬──────────┬─────────────────────┐
│ #  │ 裝置名稱          │ 類型     │ 最後活動時間         │
├────┼──────────────────┼──────────┼─────────────────────┤
│ 1  │ Main CLI         │ desktop  │ 2025-01-16 10:00    │
│ 2  │ Telegram Adapter │ adapter  │ 2025-01-16 10:01    │
│ 3  │ Eason's iPhone   │ mobile   │ 2025-01-16 10:05    │
└────┴──────────────────┴──────────┴─────────────────────┘

# 撤銷特定裝置的配對
$ openclaw devices revoke "Eason's iPhone"
裝置 'Eason's iPhone' 的配對已撤銷。Token 已失效。

# 重新命名裝置
$ openclaw devices rename 3 "Eason's 工作手機"
```

---

## 錯誤處理與容錯機制

### 錯誤分類

Gateway 將所有可能遇到的錯誤分為以下幾類：

| 錯誤類別 | 嚴重程度 | 處理策略 | 範例 |
|---------|---------|---------|------|
| 驗證錯誤 | 低 | 拒絕訊息並回報 | JSON Schema 驗證失敗 |
| 認證錯誤 | 中 | 拒絕連線並記錄 | Token 無效或過期 |
| 路由錯誤 | 中 | 返回錯誤回應 | 找不到對應的處理器 |
| 處理器錯誤 | 高 | 返回降級回應 | Brain 模組超時 |
| 系統錯誤 | 嚴重 | 記錄並嘗試恢復 | 記憶體不足、磁碟滿 |
| 連線錯誤 | 低 | 等待重連 | WebSocket 斷開 |

### 錯誤回應格式

所有錯誤都以統一的格式回應：

```json
{
  "type": "system.error",
  "version": "1.0",
  "payload": {
    "error_code": "VALIDATION_FAILED",
    "error_class": "validation",
    "message": "訊息格式驗證失敗",
    "details": {
      "field": "payload.content.type",
      "expected": "string",
      "actual": "number",
      "schema_path": "#/properties/payload/properties/content/properties/type"
    },
    "request_id": "req_abc123",
    "timestamp": "2025-01-16T10:30:00Z"
  }
}
```

### 容錯機制

**機制 1：超時與熔斷（Timeout & Circuit Breaker）**

每個內部處理器都有配置的超時時間。如果處理器在超時時間內沒有回應，Gateway 會：

1. 返回超時錯誤給客戶端
2. 記錄超時事件
3. 如果同一處理器連續超時超過閾值，啟動熔斷器

```
熔斷器狀態機：

  ┌──────────┐   正常呼叫    ┌──────────┐
  │  Closed  │──────────────▶│  Closed  │
  │ (正常)    │◀──────────────│ (正常)    │
  └────┬─────┘   成功回應    └──────────┘
       │
       │ 連續失敗超過閾值
       ▼
  ┌──────────┐
  │   Open   │  ← 拒絕所有請求，直接返回錯誤
  │ (斷開)    │    等待冷卻期
  └────┬─────┘
       │
       │ 冷卻期結束
       ▼
  ┌──────────┐
  │Half-Open │  ← 允許一個試探性請求
  │ (半開)    │    成功 → Closed；失敗 → Open
  └──────────┘
```

**機制 2：訊息重試（Message Retry）**

對於可重試的錯誤（如暫時性的處理器超時），Gateway 支援自動重試：

```yaml
retry:
  max_retries: 3
  backoff:
    type: "exponential"
    initial_delay: 1000   # 1 秒
    max_delay: 30000      # 30 秒
    multiplier: 2
  retryable_errors:
    - "PROCESSOR_TIMEOUT"
    - "PROCESSOR_BUSY"
    - "TRANSIENT_ERROR"
```

**機制 3：降級回應（Graceful Degradation）**

當核心模組不可用時，Gateway 會提供降級回應而非完全失敗：

- Brain 不可用 → 返回預設的「系統暫時無法處理您的請求」訊息
- Memory 不可用 → Brain 在沒有記憶上下文的情況下運行
- Skills 不可用 → Brain 告知使用者當前無法執行工具操作
- Heartbeat 不可用 → 其他功能正常運作，只是沒有自動排程

**機制 4：健康檢查（Health Check）**

Gateway 提供健康檢查端點，供外部監控系統使用：

```bash
# 簡單健康檢查
$ curl http://localhost:18789/health
{"status": "healthy", "uptime": "2d 5h 30m"}

# 詳細健康報告
$ curl http://localhost:18789/health/detail
{
  "status": "healthy",
  "uptime": "2d 5h 30m",
  "modules": {
    "brain": {"status": "healthy", "provider": "anthropic"},
    "memory": {"status": "healthy", "files": 342},
    "skills": {"status": "healthy", "loaded": 12},
    "heartbeat": {"status": "healthy", "next_trigger": "15m"}
  },
  "connections": {
    "total": 3,
    "active": 3,
    "adapters": ["telegram", "line", "cli"]
  }
}
```

**機制 5：自動恢復（Self-healing）**

Gateway 能夠自動恢復某些故障狀態：

- WebSocket 連線斷開 → 清理連線狀態，等待客戶端重連
- 記憶體使用過高 → 觸發垃圾回收，清理過期的快取
- 磁碟空間不足 → 停止非必要的日誌寫入，發送警告通知
- 處理器異常 → 嘗試重新初始化模組

---

## 配置參數詳解

### 完整配置範例

```yaml
# ~/.openclaw/config/gateway.yaml
# OpenClaw Gateway 完整配置檔

# ============================================
# 伺服器配置
# ============================================
server:
  # 監聽地址
  # - "127.0.0.1"：只接受本機連線（預設，最安全）
  # - "0.0.0.0"：接受所有網路介面的連線（需要額外安全措施）
  host: "127.0.0.1"
  
  # 監聽連接埠
  # 預設 18789，可以改為任何未被佔用的連接埠
  port: 18789
  
  # WebSocket 相關配置
  websocket:
    # WebSocket 端點路徑
    path: "/ws"
    
    # 最大訊息大小（bytes）
    # 預設 1MB，對於一般文字訊息綽綽有餘
    # 如果需要傳輸大型檔案或圖片，可以適當增加
    max_payload_size: 1048576
    
    # 心跳間隔（秒）
    # Gateway 每隔這個時間發送 ping 幀
    ping_interval: 30
    
    # 心跳超時（秒）
    # 如果 ping 後這個時間內沒收到 pong，標記連線為不健康
    ping_timeout: 10
    
    # 關閉握手超時（秒）
    close_timeout: 5
  
  # TLS/SSL 配置
  tls:
    enabled: false
    cert_file: "/path/to/cert.pem"
    key_file: "/path/to/key.pem"
    # 最低 TLS 版本
    min_version: "1.2"

# ============================================
# 認證配置
# ============================================
auth:
  # 認證超時（秒）
  # 連線建立後，必須在此時間內完成認證
  timeout: 10
  
  # 是否允許本地信任認證
  # 僅建議在開發環境啟用
  allow_local_trust: false
  
  # Token 配置
  tokens:
    # Token 的最小長度
    min_length: 32
    # Token 自動過期時間（天），0 表示不過期
    expiry_days: 0
  
  # 配對碼配置
  pairing:
    # 配對碼有效期（秒）
    code_ttl: 300
    # 配對碼格式
    code_format: "XXXX-XXXX"
    # 最大嘗試次數
    max_attempts: 5
    # 鎖定時間（秒）
    lockout_duration: 900

# ============================================
# 路由配置
# ============================================
routing:
  # 預設的訊息超時時間（毫秒）
  default_timeout: 30000
  
  # 訊息佇列大小
  queue_size: 100
  
  # 是否啟用訊息優先級
  priority_enabled: true
  
  # 自定義路由規則檔案
  rules_file: "~/.openclaw/config/routing.yaml"

# ============================================
# 併行控制
# ============================================
concurrency:
  # 最大同時連線數
  max_connections: 50
  
  # 每個連線的最大待處理訊息數
  max_pending_per_connection: 100
  
  # 全域最大併行處理數
  max_concurrent_requests: 10
  
  # LLM 呼叫的最大併行數
  max_concurrent_llm_calls: 3

# ============================================
# 速率限制
# ============================================
rate_limit:
  # 是否啟用速率限制
  enabled: true
  
  # 每個連線每分鐘的最大訊息數
  per_connection:
    max_per_minute: 60
    burst: 10
  
  # 全域每分鐘的最大訊息數
  global:
    max_per_minute: 200
    burst: 30

# ============================================
# 日誌配置
# ============================================
logging:
  # 日誌等級：debug, info, warn, error
  level: "info"
  
  # 日誌檔案路徑
  file: "~/.openclaw/logs/gateway.log"
  
  # 日誌輪替配置
  rotation:
    max_size: "100MB"
    max_files: 10
    compress: true
  
  # 是否記錄訊息內容（注意隱私）
  log_message_content: false
  
  # 審計日誌（獨立於一般日誌）
  audit:
    enabled: true
    file: "~/.openclaw/logs/audit.log"

# ============================================
# 容錯配置
# ============================================
resilience:
  # 重試配置
  retry:
    max_retries: 3
    backoff:
      type: "exponential"
      initial_delay: 1000
      max_delay: 30000
      multiplier: 2
  
  # 熔斷器配置
  circuit_breaker:
    failure_threshold: 5
    reset_timeout: 60000
    half_open_max_calls: 1
  
  # 健康檢查
  health_check:
    enabled: true
    path: "/health"
    interval: 60

# ============================================
# 效能調校
# ============================================
performance:
  # 訊息壓縮（對大型訊息啟用 WebSocket 壓縮）
  compression:
    enabled: true
    threshold: 1024  # 只壓縮超過 1KB 的訊息
  
  # 訊息批次處理
  batching:
    enabled: false
    max_batch_size: 10
    max_wait: 100  # 毫秒
```

### 配置的環境變數覆寫

所有配置項都可以透過環境變數覆寫，方便在不同環境中使用：

```bash
# 環境變數命名規則：OPENCLAW_GATEWAY_{SECTION}_{KEY}
export OPENCLAW_GATEWAY_SERVER_PORT=19000
export OPENCLAW_GATEWAY_AUTH_ALLOW_LOCAL_TRUST=true
export OPENCLAW_GATEWAY_LOGGING_LEVEL=debug
```

環境變數的優先級高於配置檔，這允許在不修改配置檔的情況下臨時調整參數（例如在除錯時啟用 debug 日誌）。

### 配置驗證

Gateway 在啟動時會驗證所有配置參數的合法性：

```bash
# 驗證配置檔
$ openclaw config validate
✅ server.host: 127.0.0.1 (valid)
✅ server.port: 18789 (valid, not in use)
✅ auth.timeout: 10 (valid, range: 5-60)
✅ concurrency.max_connections: 50 (valid, range: 1-1000)
✅ logging.file: ~/.openclaw/logs/gateway.log (valid, writable)
⚠️ tls.enabled: false (warning: TLS disabled, only safe for localhost)
✅ All configurations valid.
```

### 常見配置場景

**場景 1：最小配置（開發環境）**

```yaml
server:
  host: "127.0.0.1"
  port: 18789
auth:
  allow_local_trust: true
logging:
  level: "debug"
```

**場景 2：安全配置（生產環境）**

```yaml
server:
  host: "127.0.0.1"
  port: 18789
  tls:
    enabled: true
    cert_file: "/etc/openclaw/tls/cert.pem"
    key_file: "/etc/openclaw/tls/key.pem"
auth:
  allow_local_trust: false
  tokens:
    expiry_days: 90
logging:
  level: "info"
  log_message_content: false
  audit:
    enabled: true
rate_limit:
  enabled: true
```

**場景 3：區域網路共享**

```yaml
server:
  host: "0.0.0.0"  # 接受外部連線
  port: 18789
  tls:
    enabled: true   # 必須啟用 TLS
auth:
  allow_local_trust: false  # 必須關閉本地信任
rate_limit:
  enabled: true
  per_connection:
    max_per_minute: 30  # 較嚴格的速率限制
```

---

## 總結

Gateway 是 OpenClaw 架構的核心元件，承擔著訊息閘道、路由器和連線管理器的三重角色。本文件詳細介紹了 Gateway 的各個設計面向：

### 核心設計決策

1. **WebSocket 協議**：提供全雙工通訊、低延遲、持久連線，滿足 AI 助理的即時互動需求
2. **Hub/Spoke 拓撲**：實現故障隔離、獨立生命週期、統一安全邊界
3. **JSON Schema 驗證**：確保型別安全、防禦注入攻擊、提供自文件化
4. **分層認證**：傳輸安全 + 應用認證 + 權限控制的三層防護
5. **完善的容錯**：超時熔斷、自動重試、降級回應、自動恢復

### 關鍵指標

| 指標 | 預設值 | 說明 |
|------|-------|------|
| 預設連接埠 | 18789 | 可配置 |
| 最大連線數 | 50 | 對個人使用綽綽有餘 |
| 心跳間隔 | 30 秒 | 維持連線活躍 |
| 認證超時 | 10 秒 | 防止未認證連線佔用資源 |
| 最大訊息大小 | 1 MB | 防止記憶體溢出 |
| 配對碼有效期 | 5 分鐘 | 安全與便利的平衡 |

### 延伸閱讀

- [01-系統架構總覽.md](./01-系統架構總覽.md)——回顧整體架構
- [03-Brain-推理引擎.md](./03-Brain-推理引擎.md)——了解 Gateway 將訊息路由到的主要目標
- [06-Skills-技能系統.md](./06-Skills-技能系統.md)——了解 Gateway 如何協調技能調用

---

> **上一篇**：[01-系統架構總覽.md](./01-系統架構總覽.md)
> **下一篇**：[03-Brain-推理引擎.md](./03-Brain-推理引擎.md)
