# UI 與原生應用程式研究索引

> **層級**：Level 2 — Component Mechanics  
> **版本**：OpenClaw v2026.4.12  
> **範圍**：Control UI（Web）、Android/iOS/macOS 原生應用、共享 SDK、A2UI

## 目錄

| 檔案 | 內容 | 字數估計 |
|------|------|----------|
| [01-control-ui-architecture.md](./01-control-ui-architecture.md) | Web Control UI 架構、WebSocket Gateway、Device Auth | ~8,000 |
| [02-native-apps-and-a2ui.md](./02-native-apps-and-a2ui.md) | Android/iOS/macOS 原生應用、OpenClawKit SDK、A2UI 規格 | ~8,000 |

## 摘要

OpenClaw 的前端由兩大部分構成：（1）以 Vite + Lit 建構的 **Control UI** 網頁應用，透過 WebSocket 與 Gateway 即時通訊；（2）**原生應用程式**（Android Kotlin、iOS/macOS Swift），共享 OpenClawKit SDK，提供語音喚醒、攝影機、位置服務等系統能力。此外，**A2UI**（Agent-to-User Interface）是一個宣告式 JSON UI 標準，讓 AI 代理能安全地產生富互動介面。
