# 原生應用程式、OpenClawKit SDK 與 A2UI

> **本章摘要**：OpenClaw 提供 Android（Kotlin/Jetpack Compose）、iOS（Swift/SwiftUI）和 macOS（Swift）原生應用。三平台透過共享的 OpenClawKit Swift SDK 統一核心邏輯。此外，A2UI（Agent-to-User Interface）是一個宣告式 JSON UI 標準，讓 AI 代理能安全地產生跨平台的富互動介面。

---

## 1. apps/ 目錄結構

```
apps/
├── android/            # Android 應用（Kotlin + Jetpack Compose）
├── ios/                # iOS 應用（Swift + SwiftUI）
├── macos/              # macOS 應用（Swift，原生 Menu Bar）
└── shared/
    └── OpenClawKit/    # iOS/macOS 共享 SDK（Swift Package）
```

> **來源**：`source-repo/apps/` 目錄結構

---

## 2. Android 應用

### 2.1 專案配置

| 欄位 | 值 |
|------|-----|
| Namespace | `ai.openclaw.app` |
| compileSdk | 36 |
| 建構系統 | Gradle + Kotlin DSL |
| UI 框架 | Jetpack Compose |

> **來源**：`source-repo/apps/android/app/build.gradle.kts`

### 2.2 原始碼結構

```
apps/android/app/src/main/java/ai/openclaw/app/
├── MainActivity.kt              # 主 Activity
├── MainViewModel.kt             # 主 ViewModel
├── NodeApp.kt                   # 節點應用管理
├── NodeRuntime.kt               # 節點運行時
├── NodeForegroundService.kt     # 前景服務（保持連線）
├── AssistantLaunch.kt           # 助手啟動
├── DeviceNames.kt               # 裝置名稱
├── LocationMode.kt              # 定位模式
├── VoiceWakeMode.kt             # 語音喚醒模式
├── WakeWords.kt                 # 喚醒詞
├── SecurePrefs.kt               # 安全偏好儲存
├── SessionKey.kt                # Session Key
├── CameraHudState.kt            # 相機 HUD 狀態
├── PermissionRequester.kt       # 權限請求
├── NotificationForwardingPolicy.kt  # 通知轉發
├── chat/                        # 聊天相關元件
├── gateway/                     # Gateway 通訊
├── node/                        # 節點管理
├── protocol/                    # OpenClaw 協議
│   ├── OpenClawProtocolConstants.kt
│   └── OpenClawCanvasA2UIAction.kt  # A2UI 渲染
├── tools/                       # 工具整合
├── ui/                          # UI 元件
│   ├── RootScreen.kt            # 根畫面
│   ├── ChatSheet.kt             # 聊天底部頁
│   ├── OnboardingFlow.kt        # 引導流程
│   ├── ConnectTabScreen.kt      # 連線設定頁
│   ├── VoiceTabScreen.kt        # 語音設定頁
│   ├── SettingsSheet.kt         # 設定頁
│   ├── GatewayConfigResolver.kt # Gateway 配置
│   ├── GatewayDiagnostics.kt    # 連線診斷
│   ├── GatewayPairingRetry.kt   # 配對重試
│   ├── TalkOrbOverlay.kt        # 語音球懸浮層
│   ├── CameraHudOverlay.kt      # 相機 HUD
│   ├── OpenClawTheme.kt         # 主題
│   ├── MobileUiTokens.kt        # UI Token
│   └── chat/
│       ├── ChatSheetContent.kt  # 聊天內容
│       └── ChatMarkdown.kt      # Markdown 渲染
└── voice/                       # 語音相關
```

> **來源**：`source-repo/apps/android/app/src/main/java/ai/openclaw/app/` 目錄結構

### 2.3 關鍵觀察

- **前景服務**：`NodeForegroundService.kt` 確保應用在背景時仍保持 Gateway 連線
- **語音喚醒**：`VoiceWakeMode.kt` + `WakeWords.kt` 支援語音喚醒
- **A2UI 整合**：`protocol/OpenClawCanvasA2UIAction.kt` 處理 A2UI JSON 的渲染
- **安全**：`SecurePrefs.kt` 使用 Android Keystore 安全儲存

---

## 3. iOS 應用

### 3.1 專案結構

```
apps/ios/
├── project.yml              # Xcode 專案定義
├── Sources/
│   ├── Model/               # 應用模型
│   │   ├── NodeAppModel.swift         # 核心模型
│   │   ├── NodeAppModel+Canvas.swift  # Canvas 擴展
│   │   └── WatchReplyCoordinator.swift
│   ├── Voice/               # 語音功能
│   │   ├── VoiceWakeManager.swift     # 語音喚醒管理
│   │   ├── TalkModeManager.swift      # 對話模式
│   │   ├── TalkOrbOverlay.swift       # 語音球 UI
│   │   └── VoiceWakePreferences.swift
│   ├── Camera/              # 相機功能
│   │   └── CameraController.swift
│   ├── Location/            # 定位功能
│   │   ├── LocationService.swift
│   │   └── SignificantLocationMonitor.swift
│   ├── Calendar/            # 日曆整合
│   │   └── CalendarService.swift
│   ├── Reminders/           # 提醒事項
│   │   └── RemindersService.swift
│   ├── Media/               # 媒體
│   │   └── PhotoLibraryService.swift
│   ├── LiveActivity/        # iOS Live Activity
│   │   ├── LiveActivityManager.swift
│   │   └── OpenClawActivityAttributes.swift
│   ├── Capabilities/        # 能力路由
│   │   └── NodeCapabilityRouter.swift
│   └── Services/            # 系統服務
│       ├── NotificationService.swift
│       ├── WatchMessagingService.swift
│       └── WatchConnectivityTransport.swift
├── WatchApp/                # Apple Watch 應用
├── WatchExtension/          # Watch 擴展
│   ├── OpenClawWatchApp.swift
│   ├── WatchInboxStore.swift
│   └── WatchInboxView.swift
├── ActivityWidget/          # 動態島 Widget
│   ├── OpenClawLiveActivity.swift
│   └── OpenClawActivityWidgetBundle.swift
├── ShareExtension/          # 分享擴展
└── Tests/                   # 測試
```

> **來源**：`source-repo/apps/ios/Sources/` 和 `source-repo/apps/ios/` 目錄結構

### 3.2 iOS 獨特功能

| 功能 | 檔案 | 說明 |
|------|------|------|
| **Live Activity** | `LiveActivityManager.swift` | iOS 動態島/鎖定畫面即時狀態顯示 |
| **Apple Watch** | `WatchExtension/` | Watch 上查看收件匣、回覆訊息 |
| **分享擴展** | `ShareExtension/` | 從其他 App 分享內容到 OpenClaw |
| **重要位置監控** | `SignificantLocationMonitor.swift` | 背景偵測位置變化 |
| **相機** | `CameraController.swift` | 拍照/錄影功能 |
| **日曆+提醒** | `CalendarService.swift` / `RemindersService.swift` | 整合系統行事曆和提醒事項 |

---

## 4. macOS 應用

### 4.1 關鍵元件

```
apps/macos/Sources/OpenClaw/
├── GeneralSettings.swift            # 一般設定
├── AboutSettings.swift              # 關於設定
├── PermissionsSettings.swift        # 權限設定
├── TalkModeController.swift         # 對話模式控制
├── TalkOverlayView.swift            # 對話覆蓋視圖
├── VoicePushToTalk.swift            # 按鍵通話
├── VoiceWakeOverlayController+Session.swift
├── VoiceWakeForwarder.swift         # 語音喚醒轉發
├── MenuSessionsInjector.swift       # 選單列 Session 注入
├── MenuSessionsHeaderView.swift     # 選單列標題
├── GatewayDiscoveryMenu.swift       # Gateway 發現選單
├── DockIconManager.swift            # Dock 圖示管理
├── AgentWorkspace.swift             # 代理工作空間
├── CLIInstaller.swift               # CLI 安裝器
├── ConnectionModeResolver.swift     # 連線模式解析
├── RemoteTunnelManager.swift        # 遠端通道管理
├── PortGuardian.swift               # 端口守護
├── HostEnvSanitizer.swift           # 主機環境淨化
├── HostEnvSecurityPolicy.generated.swift  # 安全策略（自動產生）
├── ChannelsStore+Config.swift       # 通道配置
└── CommandResolver.swift            # 指令解析
```

> **來源**：`source-repo/apps/macos/Sources/OpenClaw/` 目錄結構

### 4.2 macOS 獨特功能

| 功能 | 檔案 | 說明 |
|------|------|------|
| **Menu Bar 應用** | `MenuSessionsInjector.swift` | 選單列即時顯示 sessions |
| **Push to Talk** | `VoicePushToTalk.swift` | 熱鍵觸發語音 |
| **CLI 安裝** | `CLIInstaller.swift` | 安裝 `openclaw` CLI 到 PATH |
| **遠端通道** | `RemoteTunnelManager.swift` | SSH/Tailscale 通道管理 |
| **安全策略** | `HostEnvSecurityPolicy.generated.swift` | 自動產生的安全策略 |
| **端口守護** | `PortGuardian.swift` | 確保 Gateway 端口不被占用 |

---

## 5. OpenClawKit 共享 SDK

### 5.1 Package 定義

```swift
// swift-tools-version: 6.2
let package = Package(
    name: "OpenClawKit",
    platforms: [.iOS(.v18), .macOS(.v15)],
    products: [
        .library(name: "OpenClawProtocol", targets: ["OpenClawProtocol"]),
        .library(name: "OpenClawKit", targets: ["OpenClawKit"]),
        .library(name: "OpenClawChatUI", targets: ["OpenClawChatUI"]),
    ],
    dependencies: [
        .package(url: "https://github.com/steipete/ElevenLabsKit", exact: "0.1.0"),
        .package(url: "https://github.com/gonzalezreal/textual", exact: "0.3.1"),
    ],
)
```

> **來源**：`source-repo/apps/shared/OpenClawKit/Package.swift:1-20`

### 5.2 三層架構

```
┌──────────────────┐
│  OpenClawChatUI  │  ← SwiftUI 聊天元件
│   依賴: Textual  │     (Markdown 渲染)
├──────────────────┤
│   OpenClawKit    │  ← 核心 SDK 邏輯
│ 依賴: ElevenLabs │     (TTS 整合)
├──────────────────┤
│OpenClawProtocol  │  ← 純協議定義（無依賴）
└──────────────────┘
```

| Library | 依賴 | 用途 |
|---------|------|------|
| **OpenClawProtocol** | 無 | Gateway 通訊協議、資料模型 |
| **OpenClawKit** | OpenClawProtocol + ElevenLabsKit | 核心 SDK（Gateway 客戶端、語音、裝置管理） |
| **OpenClawChatUI** | OpenClawKit + Textual | SwiftUI 聊天介面元件 |

> **來源**：`source-repo/apps/shared/OpenClawKit/Package.swift:21-60`

### 5.3 平台要求

- **iOS 18+**：需要最新的 iOS
- **macOS 15+**：需要 Sequoia 或更新版本
- **Swift 6.2**：使用最新的 Swift concurrency 特性（`StrictConcurrency`）

### 5.4 外部依賴

| 依賴 | 版本 | 用途 |
|------|------|------|
| **ElevenLabsKit** | 0.1.0 | ElevenLabs TTS API 整合 |
| **Textual** | 0.3.1 | Markdown → NSAttributedString（聊天渲染） |

---

## 6. A2UI：Agent-to-User Interface

### 6.1 什麼是 A2UI？

A2UI 是一個開源標準，讓 AI 代理能產生宣告式 JSON 來描述 UI，由客戶端原生渲染。目前版本 v0.8（公開預覽）。

> *"A2UI is an open standard and set of libraries that allows agents to 'speak UI.' Agents send a declarative JSON format describing the intent of the UI. The client application then renders this using its own native component library."*
>
> — `source-repo/vendor/a2ui/README.md:24-27`

### 6.2 核心哲學

| 原則 | 說明 |
|------|------|
| **安全優先** | 宣告式資料格式，非可執行程式碼。客戶端維護受信任的元件目錄 |
| **LLM 友好** | 扁平的元件列表 + ID 引用，易於 LLM 漸進式生成 |
| **框架無關** | 相同 JSON 可被 Web Components、Flutter、React、SwiftUI 渲染 |
| **可擴展** | 開放的 registry pattern，開發者可映射自訂元件 |

> **來源**：`source-repo/vendor/a2ui/README.md:36-69`

### 6.3 安全模型

A2UI 的核心安全洞見：

> *"This approach ensures that agent-generated UIs are safe like data, but expressive like code."*

- AI 代理只能請求渲染預批准目錄中的元件（Card、Button、TextField 等）
- 無法注入任意程式碼
- 客戶端控制「信任階梯」——決定哪些元件可用

> **來源**：`source-repo/vendor/a2ui/README.md:29`

### 6.4 使用案例

1. **動態資料收集**：代理根據對話上下文產生客製表單（日期選擇器、滑桿、輸入框）
2. **遠端子代理**：編排者代理委派任務給遠端專門代理，返回 UI payload 在主聊天視窗中渲染
3. **自適應工作流**：企業代理動態產生審批儀表板或資料視覺化

> **來源**：`source-repo/vendor/a2ui/README.md:73-79`

### 6.5 在 OpenClaw 中的整合

A2UI 已整合到 OpenClaw 的各平台：
- **Android**：`OpenClawCanvasA2UIAction.kt` 處理 A2UI 渲染
- **Web UI**：透過 Lit renderer（vendor/a2ui/ 包含 Lit 實作）
- **iOS/macOS**：透過 OpenClawKit SDK

---

## 7. 跨平台架構總覽

```
                    ┌─────────────────────┐
                    │   OpenClaw Gateway  │
                    │   (WebSocket API)   │
                    └────────┬────────────┘
                             │
             ┌───────────────┼───────────────┐
             │               │               │
      ┌──────▼──────┐ ┌─────▼──────┐ ┌──────▼──────┐
      │ Control UI  │ │  iOS App   │ │ Android App │
      │ (Lit/Vite)  │ │ (Swift)    │ │ (Kotlin)    │
      └─────────────┘ └─────┬──────┘ └─────────────┘
                             │
                    ┌────────▼────────┐
                    │  macOS App      │
                    │  (Swift)        │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  OpenClawKit    │  ← iOS + macOS 共享
                    │  (Swift SDK)    │
                    └─────────────────┘
```

### 7.1 共同特徵

所有平台共享：
- **WebSocket Gateway 通訊**：相同的幀格式和事件系統
- **裝置認證**：Ed25519 簽章
- **Session Key 格式**：`agent:{id}:{rest}`
- **A2UI 渲染**：宣告式 JSON UI

### 7.2 平台獨特能力

| 能力 | Android | iOS | macOS | Web |
|------|---------|-----|-------|-----|
| 語音喚醒 | ✅ | ✅ | ✅ | ❌ |
| Live Activity | ❌ | ✅ | ❌ | ❌ |
| Apple Watch | ❌ | ✅ | ❌ | ❌ |
| Menu Bar | ❌ | ❌ | ✅ | ❌ |
| Push to Talk | ❌ | ❌ | ✅ | ❌ |
| 相機 HUD | ✅ | ✅ | ❌ | ❌ |
| 位置監控 | ✅ | ✅ | ❌ | ❌ |
| CLI 安裝 | ❌ | ❌ | ✅ | ❌ |
| i18n（13 語言） | ❌ | ❌ | ❌ | ✅ |
| 通道完整配置 | ❌ | ❌ | ❌ | ✅ |

---

## 引用來源

| 來源路徑 | 引用內容 |
|----------|----------|
| `source-repo/apps/` | 頂層目錄結構 |
| `source-repo/apps/android/app/build.gradle.kts` | Android 專案配置 |
| `source-repo/apps/android/app/src/main/java/ai/openclaw/app/` | Android 原始碼結構 |
| `source-repo/apps/ios/Sources/` | iOS 原始碼結構 |
| `source-repo/apps/ios/WatchExtension/` | Apple Watch 擴展 |
| `source-repo/apps/ios/ActivityWidget/` | 動態島 Widget |
| `source-repo/apps/macos/Sources/OpenClaw/` | macOS 原始碼結構 |
| `source-repo/apps/shared/OpenClawKit/Package.swift:1-60` | 共享 SDK 定義 |
| `source-repo/vendor/a2ui/README.md:24-79` | A2UI 規格概覽 |
| `source-repo/apps/android/.../OpenClawCanvasA2UIAction.kt` | Android A2UI 整合 |
