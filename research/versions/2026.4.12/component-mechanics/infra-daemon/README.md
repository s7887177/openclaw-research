# 基礎設施與 Daemon 服務管理

> **OpenClaw v2026.4.12** — 元件機制：基礎設施層與跨平台 Daemon 管理

本目錄探討 OpenClaw 系統中最底層的兩大支柱：**基礎設施工具庫**（`src/infra/`）與**跨平台 Daemon 服務管理**（`src/daemon/`）。這兩個模組共同構成了 OpenClaw 在任何作業系統上穩定運行的根基——從進程鎖、心跳監控、安全執行到 systemd / launchd / schtasks 的全平台服務生命週期管理。

---

## 章節索引

| 檔案 | 標題 | 說明 |
|------|------|------|
| [01-infra-基礎設施工具庫.md](./01-infra-基礎設施工具庫.md) | 基礎設施工具庫（Infrastructure Utilities） | 深入剖析 `src/infra/` 目錄下 322 個檔案的功能分類、關鍵設計模式與實作細節，涵蓋錯誤韌性、Gateway 鎖、網路發現、心跳監控、安全執行、更新管理等核心子系統。 |
| [02-daemon-跨平台服務管理.md](./02-daemon-跨平台服務管理.md) | 跨平台 Daemon 服務管理（Cross-Platform Daemon Service Management） | 完整解析 `src/daemon/` 的服務註冊表架構，以及 Linux（systemd）、macOS（launchd）、Windows（schtasks）三平台的服務安裝、啟動、監控與稽核機制。 |

---

## 閱讀建議

1. **先讀第一章**——理解基礎設施層提供了哪些原語（Primitives），才能明白 Daemon 層為何需要如此設計。
2. **第二章**可依平台選讀——如果你只用 macOS，可以先跳到 launchd 整合段落，再回頭看共用基礎設施。
3. 兩章末尾的**引用來源表**可作為直接跳入原始碼的索引。
