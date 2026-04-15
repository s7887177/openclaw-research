# Local-First 設計哲學：你的 AI、你的裝置、你的規則

> **摘要**  
> 本章深入探討 OpenClaw 的 Local-First 設計哲學——這不只是一個技術選擇，而是一個關於信任、自主權和隱私的根本立場。我們將從 Local-First 的定義與歷史出發，分析 OpenClaw 的信任模型如何從「個人助理」概念出發建構安全邊界，展示其技術實現（本地 Gateway、本地記憶、本地配置），討論在 LLM 仍需遠端 API 的現實中如何務實地折衷，並將 OpenClaw 與雲端 AI 助手做本質比較。最後，我們追溯 OpenClaw 從 Warelay 到 OpenClaw 的演化歷程，理解這個設計哲學如何從一個個人實驗逐步成形。

---

## 1. 什麼是 Local-First？

### 1.1 定義

**Local-First**（本地優先）是一種軟體設計哲學，其核心主張是：

> **資料和運算首先在使用者的本地裝置上進行。雲端是可選的增強，而非必要的依賴。**

這個概念由 Martin Kleppmann 等人在 2019 年的論文《Local-First Software: You Own Your Data, in Spite of the Cloud》中系統性地闡述。他們提出了 Local-First 軟體的七個理想特性：

| # | 特性 | 含義 |
|---|------|------|
| 1 | 無延遲（No Spinners） | 操作即時回應，不等待伺服器 |
| 2 | 你的資料在你的裝置上 | 本地儲存，不依賴遠端伺服器 |
| 3 | 網路是可選的 | 離線也能工作 |
| 4 | 無縫協作 | 多裝置間能同步 |
| 5 | 長期保存 | 資料不會因服務關閉而消失 |
| 6 | 安全與隱私 | 預設端對端加密 |
| 7 | 使用者控制 | 使用者決定資料去向 |

### 1.2 為什麼在 AI 時代更重要？

在傳統軟體中，Local-First 主要是一個關於效能和離線能力的討論。但在 AI Agent 的時代，它的重要性上升到了一個全新的層次：

**AI Agent 知道你的一切。**

想像一個能讀取你的檔案、瀏覽你的網頁、管理你的訊息、記住你的偏好的 AI 助手。它的記憶中儲存著你的程式碼、你的對話、你的工作模式。這些資料的敏感程度遠超傳統應用——它不只是「資料」，而是你的數位自我（Digital Self）的一個映射。

在這個語境下，「你的資料在誰的伺服器上？」不再只是一個技術問題——它是一個**信任問題**。

```
┌─────────────────────────────────────────────────────┐
│               AI Agent 的資料敏感性                    │
│                                                      │
│  傳統軟體              AI Agent                       │
│  ─────────            ─────────                      │
│  文件 ────────────▶   + 你的對話歷史                   │
│  照片 ────────────▶   + 你的思考過程                   │
│  通訊 ────────────▶   + 你的工作模式                   │
│                       + 你的偏好與習慣                  │
│                       + 你的檔案內容                   │
│                       + 你的命令歷史                   │
│                       + 你的人際關係圖譜                │
│                                                      │
│  資料敏感度：中等       資料敏感度：極高                  │
└─────────────────────────────────────────────────────┘
```

### 1.3 Local-First vs Cloud-First vs Cloud-Only

| 模式 | 資料位置 | 運算位置 | 典型例子 |
|------|---------|---------|---------|
| **Cloud-Only** | 雲端伺服器 | 雲端伺服器 | ChatGPT, Claude.ai |
| **Cloud-First** | 雲端為主，本地快取 | 雲端為主 | Notion, Google Docs |
| **Local-First** | 本地為主，選擇性同步 | 本地為主 | Obsidian, **OpenClaw** |
| **Local-Only** | 純本地 | 純本地 | 離線筆記本 |

OpenClaw 選擇了 **Local-First**——而非 Local-Only。這是一個重要的區別：它承認某些能力（如大型 LLM 推理）目前仍需要雲端，但堅持所有可以在本地完成的事情都在本地完成。

---

## 2. OpenClaw 的哲學宣言

### 2.1 核心理念

OpenClaw 的設計哲學濃縮在 VISION.md 的開頭兩行：

> *"OpenClaw is the AI that actually does things."*  
> *"It runs on your devices, in your channels, with your rules."*  
> — `source-repo/VISION.md:3-4`

這兩句話包含了三個核心主張：

1. **Actually does things**：不只是聊天，而是真正執行任務（Agent vs Chatbot）
2. **On your devices**：在你的裝置上運行（Local-First）
3. **With your rules**：由你控制行為（使用者主權）

### 2.2 設計目標

VISION.md 進一步闡述了設計目標（`source-repo/VISION.md:15-16`）：

> *"The goal: a personal assistant that is easy to use, supports a wide range of platforms, and respects privacy and security."*

注意「respects privacy and security」被放在同一句話中——在 OpenClaw 的世界觀裡，隱私和安全不是附加功能，而是與易用性和平台支援同等重要的核心目標。

### 2.3 演化歷程：從個人實驗到開源專案

OpenClaw 不是一個從商業計畫出發的產品，它源自個人的好奇心和需求：

> *"OpenClaw started as a personal playground to learn AI and build something genuinely useful: an assistant that can run real tasks on a real computer. It evolved through several names and shells: Warelay → Clawdbot → Moltbot → OpenClaw."*  
> — `source-repo/VISION.md:11-13`

這段演化歷程揭示了一個重要事實：Local-First 不是事後加上的標籤，而是從第一天就內建在 DNA 裡的特質。一個「個人遊樂場」天然就是 Local-First 的——你在自己的電腦上玩耍，你的資料自然在你的電腦上。當這個專案成長為一個開源框架時，這個基因被保留了下來。

```
Warelay ──▶ Clawdbot ──▶ Moltbot ──▶ OpenClaw
  │            │            │            │
  │ 個人學習   │ 加入通道    │ 更多功能    │ 開源社群
  │ 本地實驗   │ 仍在本地    │ 仍在本地    │ 仍在本地
  ▼            ▼            ▼            ▼
  Local-First 基因始終貫穿整個演化歷程
```

---

## 3. 信任模型的根基

### 3.1 「個人助理」模型

OpenClaw 的安全架構建立在一個根本假設之上：

> *"OpenClaw's security model is 'personal assistant' (one trusted operator, potentially many agents), not 'shared multi-tenant bus.'"*  
> — `source-repo/SECURITY.md:162`

這個「個人助理」模型的含義是：

1. **一個操作者**：一個 Gateway 實例服務一個使用者
2. **完全信任**：經過認證的 Gateway 呼叫者被視為受信任的操作者
3. **非多租戶**：不存在「不同使用者有不同權限」的概念

### 3.2 為什麼不是多租戶？

大多數雲端服務採用多租戶（Multi-tenant）架構——多個使用者共享同一個基礎設施，透過權限系統隔離。OpenClaw 明確拒絕了這個模式：

> *"OpenClaw does not model one gateway as a multi-tenant, adversarial user boundary."*  
> — `source-repo/SECURITY.md:97`

這不是技術限制，而是**設計選擇**。原因在於信任邊界的本質差異：

```
┌─────────────────────────────────┐   ┌─────────────────────────────────┐
│     多租戶模型 (Cloud AI)        │   │    個人助理模型 (OpenClaw)       │
│                                  │   │                                  │
│   使用者 A ──┐                   │   │                                  │
│   使用者 B ──┤──▶ 共享伺服器     │   │   操作者 ──▶ 個人 Gateway       │
│   使用者 C ──┘   (權限隔離)      │   │              (完全信任)          │
│                                  │   │                                  │
│   信任問題：                     │   │   信任問題：                     │
│   • A 能看到 B 的資料嗎？       │   │   • 不存在——只有你一個人         │
│   • 伺服器管理員可信嗎？        │   │   • 你就是管理員                 │
│   • 資料加密了嗎？              │   │   • 資料在你的硬碟上             │
│   • 服務商會用我的資料訓練嗎？  │   │   • 不會——沒有服務商             │
└─────────────────────────────────┘   └─────────────────────────────────┘
```

### 3.3 Gateway 的 Shared-Secret 認證

Gateway 使用簡單的共享密鑰（Shared Secret）認證模式（`source-repo/SECURITY.md:102-108`）：

> *"Shared-secret bearer auth (token / password) authenticates possession of the gateway operator secret. Those requests receive the full default operator scope set (operator.admin, operator.read, operator.write, operator.approvals, operator.pairing)."*

這意味著：知道密鑰 = 完全的操作者權限。在多租戶系統中，這是不可接受的安全設計。但在「個人助理」模型中，這是完全合理的——你不需要對自己設定不同等級的權限。

### 3.4 信任邊界的劃分

OpenClaw 的信任邊界是**機器級別**的，不是使用者級別的：

> *"Recommended mode: one user per machine/host (or VPS), one gateway for that user, and one or more agents inside that gateway."*  
> — `source-repo/SECURITY.md:112`

這意味著：

- **信任邊界 = 機器邊界**：一台機器上的所有東西在同一個信任圈內
- **多使用者 = 多機器**：如果多人需要 OpenClaw，用不同的 VPS
- **隔離靠 OS**：作業系統的使用者隔離提供底層安全

```
┌──────────────────────────────────────┐
│            信任邊界 = 機器            │
│                                       │
│   ┌───────────────────────────────┐  │
│   │        Gateway 實例            │  │
│   │   ┌─────┐ ┌─────┐ ┌─────┐   │  │
│   │   │Agent│ │Agent│ │Agent│   │  │
│   │   │  A  │ │  B  │ │  C  │   │  │
│   │   └─────┘ └─────┘ └─────┘   │  │
│   └───────────────────────────────┘  │
│                                       │
│   ~/.openclaw/                        │
│   ├── openclaw.json    (配置)         │
│   ├── sessions/        (對話歷史)     │
│   └── plugin-runtimes/ (記憶資料)     │
│                                       │
│   所有資料都在這台機器上               │
│   所有運算都在這台機器上               │
│   只有操作者能存取                     │
└──────────────────────────────────────┘
```

---

## 4. Local-First 的技術實現

### 4.1 Gateway 在本地運行

Gateway 是 OpenClaw 的控制平面（Control Plane），它作為一個**本地 Daemon** 運行在你的機器上。

**跨平台 Daemon 支援**（`source-repo/src/daemon/`）：

| 平台 | 機制 | 實現檔案 |
|------|------|---------|
| macOS | launchd | `src/daemon/launchd.ts` |
| Linux | systemd | `src/daemon/systemd.ts` |
| Windows | Task Scheduler | `src/daemon/schtasks.ts` |

**預設綁定 127.0.0.1**

Gateway 預設只監聽本地迴環地址（Loopback），不對外暴露：

```typescript
// source-repo/src/gateway/net.ts:298-310
export async function resolveGatewayBindHost(
  bind: GatewayBindMode | undefined,
  customHost?: string,
): Promise<string> {
  const mode = bind ?? "loopback";  // 預設為 loopback

  if (mode === "loopback") {
    if (await canBindToHost("127.0.0.1")) {
      return "127.0.0.1";
    }
    return "0.0.0.0"; // 極端備援
  }
  // ...
}
```

`resolveGatewayBindHost()` 函數清楚地展示了 Local-First 的預設立場：
- **預設模式是 `"loopback"`**——只接受來自本機的連線
- 其他模式（`"lan"`, `"tailnet"`, `"auto"`, 自定義）是可選的
- 即使在 `"tailnet"` 模式下，如果 Tailscale 不可用，也會退回 `127.0.0.1`

SECURITY.md 對此有明確的指導（`source-repo/SECURITY.md:266-278`）：

> *"Recommended: keep the Gateway loopback-only (127.0.0.1 / ::1). Do not expose it to the public internet."*

### 4.2 配置檔在本地

所有配置都儲存在本地的 `~/.openclaw/` 目錄中：

```typescript
// source-repo/src/config/paths.ts:20-23
const LEGACY_STATE_DIRNAMES = [".clawdbot"] as const;
const NEW_STATE_DIRNAME = ".openclaw";
const CONFIG_FILENAME = "openclaw.json";
```

`resolveStateDir()` 函數（`source-repo/src/config/paths.ts:60-89`）負責定位狀態目錄：

```typescript
export function resolveStateDir(
  env: NodeJS.ProcessEnv = process.env,
  homedir: () => string = envHomedir(env),
): string {
  const override = env.OPENCLAW_STATE_DIR?.trim();
  if (override) {
    return resolveUserPath(override, env, effectiveHomedir);
  }
  const newDir = newStateDir(effectiveHomedir); // ~/.openclaw
  // ...向下相容 legacy 目錄 (.clawdbot)
  return newDir;
}
```

配置結構（從 `resolveCanonicalConfigPath()` 可以看出，`source-repo/src/config/paths.ts:106-114`）：

```
~/.openclaw/
├── openclaw.json           # 主配置檔
├── sessions/               # 對話歷史（Session Transcript）
├── logs/                   # Gateway 日誌
│   ├── gateway.log
│   └── gateway.err.log
└── plugin-runtimes/        # Plugin 運行時資料
    └── memory-lancedb/     # 向量記憶資料庫
        └── lancedb/
```

注意 Legacy 支援——程式碼中保留了對 `.clawdbot` 目錄的相容（`source-repo/src/config/paths.ts:21`），這是 OpenClaw 從 Clawdbot 改名時的遷移支援，也佐證了 VISION.md 中描述的演化歷程。

### 4.3 記憶儲存在本地

OpenClaw 的長期記憶使用 **LanceDB**——一個嵌入式向量資料庫（Embedded Vector Database），資料直接存在本地檔案系統。

```typescript
// source-repo/extensions/memory-lancedb/index.ts:57-75
class MemoryDB {
  constructor(
    private readonly dbPath: string,  // 本地資料庫路徑
  ) {}
  
  async connect() {
    const lancedb = await loadLanceDbModule();
    this.db = await lancedb.connect(this.dbPath);
  }
}
```

LanceDB 的資料路徑解析（`source-repo/extensions/memory-lancedb/lancedb-runtime.ts:96`）：

```
{stateDir}/plugin-runtimes/memory-lancedb/lancedb/
```

這意味著你的所有記憶——Agent 記住的每一件事、每一次對話的語義索引——都是你硬碟上的檔案。沒有雲端資料庫，沒有第三方服務，沒有資料傳輸。

**為什麼選擇 LanceDB？**

| 特性 | LanceDB | 傳統向量資料庫（Pinecone, Weaviate） |
|------|---------|--------------------------------------|
| 部署方式 | 嵌入式（In-process） | 獨立服務或雲端 |
| 資料位置 | 本地檔案 | 遠端伺服器 |
| 依賴 | 無外部依賴 | 需要網路連線 |
| 成本 | 免費 | 按用量付費 |
| 隱私 | 完全本地 | 資料在第三方 |

LanceDB 的選擇完美契合 Local-First 理念——它是一個不需要任何外部服務的向量資料庫。

### 4.4 UI 透過本地 WebSocket 連接

OpenClaw 的 Web UI 和 CLI 都透過 WebSocket 連接到本地 Gateway：

```
CLI / Web UI  ◀──WebSocket──▶  Gateway (127.0.0.1:18789)
                                    │
                                    ├── Agent 執行
                                    ├── 工具調用
                                    └── 記憶存取
```

Gateway 的 HTTP 伺服器監聽預設端口 18789（`source-repo/src/config/paths.ts`），這是一個純本地的通訊——資料不需要離開你的機器。

---

## 5. 折衷與務實

### 5.1 LLM 仍需遠端 API

Local-First 不代表完全離線。OpenClaw 面對一個現實：當前最強大的 LLM（GPT-4o、Claude、Gemini）都是雲端服務。這意味著：

- **使用者的提示詞（Prompt）會被傳送到遠端** LLM 提供者
- **LLM 的回應會從遠端傳回**
- 這個過程中，你的對話內容經過了第三方伺服器

OpenClaw 對此的立場是**務實的折衷**：接受這個現實，但盡量減少洩露的資訊，並給使用者選擇權。

### 5.2 本地模型支援

OpenClaw 提供了完全本地的 LLM 選項：

**Ollama 整合**（`source-repo/extensions/ollama/src/defaults.ts:1`）：

```typescript
export const OLLAMA_DEFAULT_BASE_URL = "http://127.0.0.1:11434";
```

Ollama 運行在 `127.0.0.1:11434`——又一個純本地的服務。使用 Ollama，你的提示詞完全不會離開你的機器。

**LM Studio 整合**（`source-repo/src/plugin-sdk/lmstudio.ts`）：

LM Studio 提供了另一個本地模型運行環境，OpenClaw 同樣預設連接本地地址。

**MLX 整合**：

Apple Silicon 裝置可以透過 MLX 框架在本地運行模型，OpenClaw 的 Provider 抽象層讓切換本地/遠端模型只需修改一行配置。

```
┌─────────────────────────────────────────────────┐
│              OpenClaw LLM 連接選項               │
│                                                  │
│  ┌──────────────┐   完全本地                     │
│  │   Ollama     │──▶ 127.0.0.1:11434            │
│  │   LM Studio  │──▶ 127.0.0.1:1234             │
│  │   MLX        │──▶ 本地推理                    │
│  └──────────────┘                                │
│                                                  │
│  ┌──────────────┐   遠端 API                     │
│  │   OpenAI     │──▶ api.openai.com              │
│  │   Anthropic  │──▶ api.anthropic.com           │
│  │   Google     │──▶ generativelanguage.google   │
│  │   50+ 其他   │──▶ 各提供者 API               │
│  └──────────────┘                                │
│                                                  │
│  操作者的選擇：完全本地 or 遠端，或混合使用      │
└─────────────────────────────────────────────────┘
```

### 5.3 Remote Access 的選項

如果操作者需要從外部存取 Gateway（例如從手機控制家裡的 AI），OpenClaw 推薦**安全隧道**而非直接暴露：

- **Tailscale**：透過 WireGuard VPN 建立私有網路
- **SSH Tunnel**：經典的加密隧道
- **Tailscale Serve/Funnel**：Tailscale 的內建服務暴露功能

Gateway 的 `resolveGatewayBindHost()` 直接支援 Tailscale 模式（`source-repo/src/gateway/net.ts:312-320`）：

```typescript
if (mode === "tailnet") {
  const tailnetIP = pickPrimaryTailnetIPv4();
  if (tailnetIP && (await canBindToHost(tailnetIP))) {
    return tailnetIP;
  }
  if (await canBindToHost("127.0.0.1")) {
    return "127.0.0.1";  // Tailscale 不可用時退回 loopback
  }
  // ...
}
```

注意退回策略：如果 Tailscale 不可用，自動退回 loopback。**安全優先，功能退化優於暴露風險。**

SECURITY.md 對此的指導（`source-repo/SECURITY.md:279`）：

> *"If you need remote access, prefer an SSH tunnel or Tailscale serve/funnel (so the Gateway still binds to loopback), plus strong Gateway auth."*

---

## 6. 與雲端 AI 助手的本質差異

### 6.1 架構比較

| 維度 | ChatGPT / Claude.ai | OpenClaw |
|------|---------------------|----------|
| **運行位置** | OpenAI / Anthropic 伺服器 | 你的機器上 |
| **對話儲存** | 雲端資料庫 | `~/.openclaw/sessions/` |
| **記憶儲存** | 雲端（不透明） | 本地 LanceDB |
| **配置** | Web 介面設定 | `openclaw.json`（你能 `cat` 它） |
| **工具執行** | 受限的沙箱 | 你的完整系統（可配置沙箱） |
| **停機影響** | 完全無法使用 | 雲端 LLM 不可用時退回本地模型 |
| **資料擁有權** | 服務商 | 你 |
| **自訂程度** | 有限（Custom GPTs） | 完全（Plugin / Skill / Config） |
| **通道支援** | 官方 App / API | 25+ 通道（Discord, Telegram, Slack...） |
| **成本模型** | 月費訂閱 | 按 API 使用量（或完全免費用本地模型） |

### 6.2 控制權的本質差異

最根本的差異不在技術，而在**控制權**：

**雲端模式**：你是**使用者**（User）——你使用別人的服務，遵守別人的規則，資料存在別人的伺服器上。服務商可以：
- 變更定價
- 修改功能
- 存取你的資料（根據 Terms of Service）
- 關閉服務

**Local-First 模式**：你是**操作者**（Operator）——你運行自己的系統，制定自己的規則，資料在自己的硬碟上。你可以：
- 選擇任何 LLM 提供者
- 自訂任何行為
- 完全掌控資料
- 永遠不會被「停服」

OpenClaw 的 VISION.md 正是這個差異的體現：*"with your rules"*。

### 6.3 Agent 信任模型的差異

```
┌─────────────────────────────────┐   ┌─────────────────────────────────┐
│   雲端 AI 助手的信任模型         │   │   OpenClaw 的信任模型            │
│                                  │   │                                  │
│   使用者 ──▶ 服務商伺服器        │   │   操作者 ──▶ 本地 Gateway       │
│              (你信任服務商)       │   │              (你信任自己)        │
│                                  │   │                                  │
│   信任鏈：                       │   │   信任鏈：                       │
│   你 → 服務商 → 基礎設施 → ...  │   │   你 → 你的機器 → 你的配置      │
│   (長而不可見的信任鏈)           │   │   (短而完全透明的信任鏈)         │
│                                  │   │                                  │
│   模型（LLM）= 不受信任的       │   │   模型（LLM）= 不受信任的       │
│   (但伺服器是受信任的)           │   │   (但操作者是受信任的)           │
└─────────────────────────────────┘   └─────────────────────────────────┘
```

一個值得注意的共同點：**兩者都不信任 LLM 本身。** OpenClaw 的 SECURITY.md 明確指出（`source-repo/SECURITY.md:192-193`）：

> *"The model/agent is not a trusted principal. Assume prompt/content injection can manipulate behavior."*
> *"Security boundaries come from host/config trust, auth, tool policy, sandboxing, and exec approvals."*

這是一個深刻的洞察：信任不應該建立在 AI 模型上，而應該建立在你能控制的基礎設施上。在雲端模式中，「你能控制的」只有你的帳號密碼；在 Local-First 模式中，「你能控制的」是整台機器。

---

## 7. Plugin 與記憶的本地信任模型

### 7.1 Plugin 信任邊界

OpenClaw 的 Plugin 系統同樣體現了 Local-First 的信任模型：

> *"Plugins/extensions are loaded in-process with the Gateway and are treated as trusted code."*
> *"Plugins can execute with the same OS privileges as the OpenClaw process."*  
> — `source-repo/SECURITY.md:220-222`

這意味著 Plugin 不在沙箱中運行——它與 Gateway 具有相同的權限。在多租戶系統中，這會是一個嚴重的安全問題。但在「個人助理」模型中，這是合理的：**你安裝的 Plugin 就像你安裝的任何其他軟體——你選擇信任它。**

### 7.2 記憶的信任邊界

工作區記憶（`MEMORY.md` 和 `memory/*.md`）被視為受信任的本地操作者狀態：

> *"MEMORY.md and memory/*.md are plain workspace files and are treated as trusted local operator state."*  
> — `source-repo/SECURITY.md:211`

如果有人能編輯你的記憶檔案，他們已經跨越了信任邊界——因為他們已經能存取你的檔案系統了。在 Local-First 模型中，檔案系統的存取控制就是信任邊界。

### 7.3 Exec 行為

命令執行預設在主機上直接運行，不經過沙箱（`source-repo/SECURITY.md:115-118`）：

> *"Exec behavior is host-first by default: agents.defaults.sandbox.mode defaults to off."*
> *"This is expected in OpenClaw's one-user trusted-operator model."*

這又是 Local-First 理念的體現：在你自己的機器上，AI 助手執行命令就像你自己執行一樣——因為它就是代替你在執行。

---

## 8. 安全的務實策略

### 8.1 強預設，彈性配置

OpenClaw 的安全策略可以總結為（`source-repo/VISION.md:43-44`）：

> *"Security in OpenClaw is a deliberate tradeoff: strong defaults without killing capability."*
> *"The goal is to stay powerful for real work while making risky paths explicit and operator-controlled."*

這種「強預設，彈性配置」的策略體現在：

| 面向 | 預設值（安全） | 可調整為（強大） |
|------|--------------|---------------|
| Gateway 綁定 | `loopback`（127.0.0.1） | `lan`, `tailnet`, 自定義 |
| 沙箱模式 | `off`（信任操作者） | `non-main`, `all` |
| 工具策略 | 完整工具集（信任操作者） | `messaging`（限制集） |
| Exec 審批 | 操作者護欄 | 可配置自動審批 |

### 8.2 終端優先的設計

OpenClaw 目前是「終端優先」（Terminal-First）的設計（`source-repo/VISION.md:87-89`）：

> *"OpenClaw is currently terminal-first by design. This keeps setup explicit: users see docs, auth, permissions, and security posture up front."*
> *"We do not want convenience wrappers that hide critical security decisions from users."*

這是 Local-First 理念的另一個面向：使用者應該能看到並理解系統在做什麼。圖形介面的便利性不能以犧牲透明度為代價。

---

## 9. TypeScript 的選擇：可駭入性

VISION.md 解釋了為什麼選擇 TypeScript（`source-repo/VISION.md:95-98`）：

> *"OpenClaw is primarily an orchestration system: prompts, tools, protocols, and integrations."*
> *"TypeScript was chosen to keep OpenClaw hackable by default."*
> *"It is widely known, fast to iterate in, and easy to read, modify, and extend."*

「Hackable by default」（預設可駭入）是 Local-First 理念在語言選擇上的延伸。如果你的 AI 助手運行在你的機器上，你應該能夠修改它的原始碼。TypeScript 作為一個廣泛熟知的語言，降低了這個門檻。

---

## 10. 總結：Local-First 作為一種世界觀

OpenClaw 的 Local-First 不只是一個技術架構決策——它是一種完整的世界觀：

> **你的 AI 助手應該是你的，而不是某家公司的。**

這個世界觀的具體體現：

| 面向 | 實現 |
|------|------|
| **資料主權** | 所有資料在 `~/.openclaw/`，你可以 `ls`、`cat`、`rm` |
| **運算主權** | Gateway 在你的機器上運行 |
| **模型選擇權** | 50+ 雲端提供者 + 本地模型（Ollama, LM Studio, MLX） |
| **配置透明** | `openclaw.json` 是一個你能讀懂的 JSON 檔案 |
| **通道自主** | 你選擇在哪些通道上部署（不被綁定到特定平台） |
| **安全自主** | 信任邊界由你定義，不是由服務商定義 |
| **原始碼可讀** | TypeScript + 開源 = 你能閱讀和修改每一行程式碼 |

當我們在後續章節中分析 OpenClaw 的每一個技術細節時——從 Channel 適配器到 Context Engine、從 Plugin SDK 到 MCP 橋接——都可以在這個 Local-First 世界觀中找到它的設計動機。

---

## 引用來源

| 來源 | 路徑 | 引用內容 |
|------|------|---------|
| VISION.md | `source-repo/VISION.md:3-4` | "AI that actually does things, on your devices, with your rules" |
| VISION.md | `source-repo/VISION.md:11-13` | 演化歷程：Warelay → Clawdbot → Moltbot → OpenClaw |
| VISION.md | `source-repo/VISION.md:15-16` | 設計目標：易用、多平台、尊重隱私與安全 |
| VISION.md | `source-repo/VISION.md:43-44` | 安全是刻意的折衷 |
| VISION.md | `source-repo/VISION.md:74-80` | MCP 透過 mcporter 整合 |
| VISION.md | `source-repo/VISION.md:87-89` | 終端優先設計 |
| VISION.md | `source-repo/VISION.md:95-98` | TypeScript 選擇理由 |
| SECURITY.md | `source-repo/SECURITY.md:97` | 非多租戶設計 |
| SECURITY.md | `source-repo/SECURITY.md:102-108` | Shared-secret 認證模式 |
| SECURITY.md | `source-repo/SECURITY.md:112` | 一人一機器一 Gateway |
| SECURITY.md | `source-repo/SECURITY.md:115-118` | Exec 預設 host-first |
| SECURITY.md | `source-repo/SECURITY.md:162` | 個人助理安全模型 |
| SECURITY.md | `source-repo/SECURITY.md:192-193` | 模型不受信任 |
| SECURITY.md | `source-repo/SECURITY.md:211` | 記憶檔案信任邊界 |
| SECURITY.md | `source-repo/SECURITY.md:220-222` | Plugin 信任邊界 |
| SECURITY.md | `source-repo/SECURITY.md:266-278` | Web 介面安全指導 |
| config/paths.ts | `source-repo/src/config/paths.ts:20-23` | 狀態目錄與配置檔名 |
| config/paths.ts | `source-repo/src/config/paths.ts:60-89` | resolveStateDir() 函數 |
| config/paths.ts | `source-repo/src/config/paths.ts:106-114` | resolveCanonicalConfigPath() |
| gateway/net.ts | `source-repo/src/gateway/net.ts:298-310` | resolveGatewayBindHost() 預設 loopback |
| gateway/net.ts | `source-repo/src/gateway/net.ts:312-320` | Tailscale 模式與退回策略 |
| memory-lancedb | `source-repo/extensions/memory-lancedb/index.ts:57-75` | LanceDB 本地向量資料庫 |
| lancedb-runtime | `source-repo/extensions/memory-lancedb/lancedb-runtime.ts:96` | LanceDB 儲存路徑 |
| ollama defaults | `source-repo/extensions/ollama/src/defaults.ts:1` | Ollama 預設 127.0.0.1:11434 |
| lmstudio | `source-repo/src/plugin-sdk/lmstudio.ts:15` | LM Studio 預設本地地址 |
| daemon | `source-repo/src/daemon/launchd.ts` | macOS Daemon 支援 |
| daemon | `source-repo/src/daemon/systemd.ts` | Linux Daemon 支援 |
| daemon | `source-repo/src/daemon/schtasks.ts` | Windows Daemon 支援 |
