# Topic 5: 自訂 Skill 開發（Custom Skill Development）

本章涵蓋 OpenClaw 自訂 Skill 的完整開發指南。

## 章節索引

| 檔案 | 內容 |
|------|------|
| [01-Skill架構與格式.md](./01-Skill架構與格式.md) | SKILL.md 格式、目錄結構、Frontmatter、需求定義 |
| [02-開發與發佈.md](./02-開發與發佈.md) | 開發流程、測試、CLI 管理、ClawHub 發佈 |

## 摘要

Skill 是 OpenClaw 的能力擴展單元。每個 Skill 由一個 `SKILL.md` 檔案定義，包含 YAML frontmatter（中繼資料）和 Markdown 指引（提供給 AI 的上下文）。Skill 可以宣告對外部工具的依賴（requires）、提供安裝指引、包含腳本和參考資料。操作者可以透過 CLI 管理 Skill，也可以將自訂 Skill 發佈到 ClawHub 供社群使用。
