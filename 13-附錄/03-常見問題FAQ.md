# 常見問題 FAQ

> **說明**：本文收錄 OpenClaw 使用過程中的常見問題和解答，按主題分類。

---

## 目錄

1. [基礎概念](#1-基礎概念)
2. [安裝與配置](#2-安裝與配置)
3. [模型與提供者](#3-模型與提供者)
4. [Agent 與人格](#4-agent-與人格)
5. [記憶系統](#5-記憶系統)
6. [工具與技能](#6-工具與技能)
7. [安全與權限](#7-安全與權限)
8. [通道整合](#8-通道整合)
9. [語音功能](#9-語音功能)
10. [效能與成本](#10-效能與成本)
11. [MCP 整合](#11-mcp-整合)
12. [部署與維運](#12-部署與維運)
13. [疑難排解](#13-疑難排解)

---

## 1. 基礎概念

### Q1.1 OpenClaw 是什麼？
OpenClaw 是一個開源的 AI Agent 框架，專為建構「個人 AI 助手」設計。它提供 Gateway 架構、多 Agent 支援、記憶系統、多通道連接、九層權限模型和 ReAct 推理迴圈。

### Q1.2 OpenClaw 和 LangChain 有什麼不同？
LangChain 是一個通用的 LLM 應用框架，適合各種 LLM 應用場景。OpenClaw 更專注於「個人 AI 助手」這個使用場景，內建了人格系統（SOUL.md）、長期記憶、多通道支援和完整的安全模型。如果你需要建構一個有記憶、有個性、能在多個平台上使用的個人助手，OpenClaw 開箱即用的體驗會更好。

### Q1.3 OpenClaw 適合用來做多人 SaaS 服務嗎？
不太適合。OpenClaw 的設計理念是「一個 Gateway = 一個人的助手」。它的信任模型、記憶系統和權限設計都基於單一操作者。如果需要多租戶服務，建議使用專門的 SaaS 框架。

### Q1.4 OpenClaw 可以離線使用嗎？
可以。透過 Ollama Provider 使用本地模型（如 Llama 3），所有推理都在本地完成。但某些技能（如 search_web）仍需要網路連接。

### Q1.5 OpenClaw 支援哪些程式語言？
OpenClaw 核心使用 TypeScript/Node.js 開發。Skill 可以用 TypeScript、JavaScript 編寫。透過 MCP 協議，其他語言（Python、Go 等）也可以與 OpenClaw 整合。

---

## 2. 安裝與配置

### Q2.1 安裝的系統需求是什麼？
- Node.js 20+
- npm 或 pnpm
- 至少 512 MB RAM
- Linux（推薦）、macOS 或 Windows（WSL）

### Q2.2 如何快速開始？
```bash
npm install -g @openclaw/cli
openclaw config init
# 填寫 API Key 等基本資訊
openclaw gateway start
openclaw run  # 互動模式
```

### Q2.3 配置檔用 JSON5 有什麼好處？
JSON5 支援註解（方便文件化配置）、尾逗號（方便新增項目）和更寬鬆的語法（單引號、多行字串等），大幅提升配置檔的可讀性和維護性。

### Q2.4 如何安全地管理 API Key？
推薦使用環境變數：
1. 在 `.env` 檔案中定義 API Key
2. 在 `openclaw.json5` 中用 `${ENV_VAR}` 引用
3. 確保 `.env` 加入 `.gitignore`
4. 不要將 API Key 直接寫在配置檔中

### Q2.5 可以同時執行多個 Gateway 嗎？
可以，只要使用不同的 Port 和配置檔。例如一個用於日常助手，另一個用於開發測試。

---

## 3. 模型與提供者

### Q3.1 推薦使用哪個模型？
- **日常對話和通用任務**：GPT-4o 或 Claude 3 Sonnet（性價比好）
- **程式碼生成**：Claude 3 Sonnet 或 GPT-4-turbo
- **語音對話**：GPT-4o（回應快，支援串流）
- **本地/離線**：Llama 3 via Ollama
- **低成本**：GPT-3.5-turbo

### Q3.2 可以在不同任務中使用不同模型嗎？
可以。每個 Agent 可以配置不同的模型。例如社交 Agent 使用 GPT-4o，程式碼 Agent 使用 Claude 3 Sonnet。

### Q3.3 模型 API 費用大概多少？
以 GPT-4o 為例，個人日常使用（每天 50-100 次對話）大約每月 $20-50 美元。使用本地模型（Ollama）完全免費。

### Q3.4 如何設定模型 fallback？
目前 OpenClaw 不內建自動 fallback，但可以透過 Hook 機制實現：在 postInference hook 中檢測錯誤，切換到備用 Provider 重新推理。

---

## 4. Agent 與人格

### Q4.1 一個 Gateway 可以有幾個 Agent？
沒有硬限制，但建議不超過 10 個，以控制資源使用和配置複雜度。

### Q4.2 SOUL.md 寫多長比較好？
建議 500-2000 字。太短會導致人格不夠鮮明，太長會佔用過多上下文窗口。核心規則應該簡潔明確，可以用範例來補充說明。

### Q4.3 如何讓 Agent 不再說「作為一個 AI」？
在 SOUL.md 中明確寫上：「不要說『作為一個 AI』、『我是語言模型』之類的話。」同時定義 Agent 應該如何回答關於自身身份的問題。

### Q4.4 不同 Agent 之間可以通訊嗎？
目前版本不支援 Agent 間直接通訊。但可以透過共享記憶間接傳遞資訊。未來版本計劃加入 Agent-to-Agent 通訊協議。

### Q4.5 Agent 的回應太長/太短怎麼調？
- **太長**：降低 `max_tokens`，在 SOUL.md 中強調簡潔
- **太短**：提高 `max_tokens`，在 SOUL.md 中鼓勵詳細回答
- **語音場景**：`max_tokens` 設為 200-500，SOUL.md 中要求簡短口語化

---

## 5. 記憶系統

### Q5.1 記憶儲存在哪裡？
預設使用 SQLite，儲存在 `data/memories/` 目錄。也支援 PostgreSQL。

### Q5.2 記憶會佔多少空間？
文字記憶本身很小（幾 KB），但嵌入向量會較大（每條約 6KB，使用 1536 維度）。1000 條記憶大約佔 6-10 MB。

### Q5.3 自動記憶會記住不該記的東西嗎？
可能會。建議在 MEMORY.md 中明確定義「不應記住」的類別（如密碼、token、臨時的抱怨等），並定期檢查記憶內容。

### Q5.4 如何備份記憶？
```bash
# 手動備份
openclaw memory export --format json > memories-backup.json

# 自動備份（在 openclaw.json5 中配置）
memory.backup.enabled = true
memory.backup.schedule = "0 3 * * *"  # 每天凌晨 3 點
```

### Q5.5 記憶可以跨 Agent 共享嗎？
預設所有 Agent 共享同一個記憶儲存。可以透過 `memory.channel_scoped` 配置讓某些類別的記憶只對特定通道可見。

### Q5.6 記憶上限是多少？
沒有硬限制，但建議定期整理。超過 10,000 條記憶時，搜尋效能可能受影響（取決於嵌入搜尋的實作）。

---

## 6. 工具與技能

### Q6.1 內建工具有哪些？
search_web（搜尋）、memory_read/write（記憶）、exec（執行命令）、file_read/write（檔案）等。完整列表請參考 CLI 指令：`openclaw skill list`。

### Q6.2 如何建立自訂技能？
```bash
openclaw skill create my-skill --type function
# 編輯 skills/my-skill/index.ts
# 在 openclaw.json5 中註冊
openclaw skill test my-skill --args '{"key": "value"}'
```

### Q6.3 技能可以呼叫外部 API 嗎？
可以。技能本質上是 TypeScript 函數，可以使用 `fetch` 呼叫任何 HTTP API。建議在沙箱外執行需要網路存取的技能。

### Q6.4 AI 會不會亂用工具？
OpenClaw 的九層權限模型就是為此設計的。你可以控制每個 Agent 能使用哪些工具、哪些參數範圍、是否需要人工批准。此外，模型本身的工具選擇通常是合理的。

---

## 7. 安全與權限

### Q7.1 個人使用需要關心安全嗎？
需要，但程度較低。建議至少：
- 不讓 Agent 直接使用 `exec`（除非在沙箱中）
- API Key 使用環境變數管理
- 啟用記憶加密
- 定期執行 `openclaw security audit`

### Q7.2 什麼時候需要沙箱？
當 Agent 需要執行外部命令（exec）或寫入檔案（file_write）時，建議啟用沙箱。純對話和記憶操作不需要。

### Q7.3 如何防止 Prompt Injection？
- 輸入清理：過濾已知的注入模式
- 角色隔離：系統提示與使用者輸入明確分離
- 工具權限：限制 Agent 能調用的工具
- 參數約束：對工具參數設定白名單和範圍
- 監控：記錄並審查異常的工具調用

### Q7.4 九層權限的順序重要嗎？
非常重要。權限檢查從第一層開始，任何一層拒絕就立即終止。設計時應遵循「最早拒絕」原則，將最嚴格的限制放在前面的層級。

---

## 8. 通道整合

### Q8.1 支援哪些通道？
目前支援：Discord（文字+語音）、Telegram、Matrix、API（HTTP+WebSocket）、CLI。透過插件系統可以擴展更多通道。

### Q8.2 一個 Agent 可以同時在多個通道上嗎？
可以。例如 "claw" Agent 可以同時在 Discord 和 Telegram 上接收訊息，且共享記憶和人格。

### Q8.3 Discord Bot 需要哪些權限？
最小權限：Send Messages、Read Message History。語音功能額外需要：Connect、Speak、Use Voice Activity。建議也加上 Message Content Intent。

### Q8.4 如何處理 Discord 訊息長度限制？
Discord 單條訊息限制 2000 字元。OpenClaw 會自動將超長回應分割成多條訊息。

---

## 9. 語音功能

### Q9.1 語音對話的延遲大概多少？
理想情況下 1.5-3 秒（STT 500ms + LLM 500-2000ms + TTS 500ms）。使用串流模式可以讓使用者更早聽到回應。

### Q9.2 支援哪些語言的語音識別？
取決於 STT 引擎。Whisper 支援 99 種語言，Azure Speech 支援 80+ 種語言。繁體中文（zh-TW）在兩者都有良好支援。

### Q9.3 TTS 語音自然嗎？
Azure Neural TTS 的中文語音品質相當高，特別是「聊天」風格。ElevenLabs 的語音克隆效果更自然但費用較高。

### Q9.4 多人同時在語音頻道說話會怎樣？
Discord 為每個使用者提供獨立的音頻流，OpenClaw 可以識別每個說話者。預設只回應直接對 Bot 說話（呼叫名字）或一對一場景。

### Q9.5 語音功能的費用高嗎？
- Whisper STT：約 $0.006/分鐘
- Azure TTS：免費額度 500,000 字元/月，之後 $16/100 萬字元
- Edge TTS：完全免費（但品質略低）

---

## 10. 效能與成本

### Q10.1 Gateway 需要多少系統資源？
最低：1 CPU 核心、256 MB RAM。建議：2 核心、1 GB RAM。語音功能額外需要更多 CPU。

### Q10.2 如何降低 API 費用？
- 使用較便宜的模型處理簡單任務（GPT-3.5-turbo）
- 限制 `max_tokens` 減少回應長度
- 啟用工具結果快取
- 使用 Token 壓縮（摘要舊對話）
- 本地模型（Ollama）處理低風險任務

### Q10.3 上下文太長怎麼辦？
OpenClaw 會自動進行 Token 壓縮：
1. 摘要舊對話（保留最近 10 輪）
2. 截斷長的工具結果
3. 按相關性選擇記憶（不是全部載入）

### Q10.4 如何監控使用量？
```bash
openclaw usage show --period today     # 今日統計
openclaw usage cost --period month     # 本月費用估算
```

---

## 11. MCP 整合

### Q11.1 MCP 和 REST API 有什麼不同？
MCP 是專為 AI 工具互動設計的協議，支援工具發現、資源讀取和提示模板。REST API 是通用的 HTTP API。MCP 更適合 AI-to-AI 的整合場景。

### Q11.2 如何讓 Copilot CLI 使用 OpenClaw 的記憶？
1. 啟動 OpenClaw MCP Server
2. 在 Copilot CLI 的 MCP 配置中註冊
3. Copilot 會自動發現 `memory_search` 工具
4. 在對話中自然地使用記憶功能

### Q11.3 MCP Server 支援認證嗎？
stdio 模式依賴作業系統的進程安全。SSE 模式支援 Bearer Token 認證。

---

## 12. 部署與維運

### Q12.1 推薦的部署方式？
個人使用推薦：
- **本地**：直接 `openclaw gateway start --daemon`
- **VPS**：Docker Compose + systemd
- **進階**：Docker + 反向代理（Nginx）+ Let's Encrypt

### Q12.2 如何設定開機自動啟動？
```bash
# systemd 服務（Linux）
sudo systemctl enable openclaw-gateway

# 或使用 pm2
pm2 start openclaw -- gateway start
pm2 save
pm2 startup
```

### Q12.3 如何更新 OpenClaw？
```bash
npm update -g @openclaw/cli
openclaw gateway stop
openclaw gateway start
```

### Q12.4 日誌太大怎麼辦？
在配置檔中設定日誌輪轉：
```json5
logging: {
  rotation: {
    max_size: "50M",
    max_files: 10
  }
}
```

---

## 13. 疑難排解

### Q13.1 Gateway 啟動失敗
```
常見原因：
1. Port 被佔用 → 換一個 Port 或關閉佔用的進程
2. API Key 無效 → 檢查 .env 和環境變數
3. 配置檔語法錯誤 → openclaw config validate
4. Node.js 版本太低 → 升級到 20+
```

### Q13.2 Agent 不回應
```
檢查步驟：
1. openclaw gateway status → 確認 Gateway 運行中
2. openclaw log tail → 查看錯誤日誌
3. 確認通道已連接（channel list）
4. 確認 Agent 已啟用（agent list）
5. 測試推理：openclaw run --agent <agent-id>
```

### Q13.3 記憶搜尋結果不相關
```
可能原因：
1. 記憶數量太少，嵌入空間不夠密集
2. 搜尋查詢太模糊 → 使用更具體的關鍵字
3. 嵌入模型不適合 → 嘗試不同的嵌入模型
4. 相似度閾值太高 → 降低 threshold
```

### Q13.4 工具調用被拒絕
```bash
# 使用 probe 命令診斷
openclaw security probe tool <tool-name> --agent <agent-id>
# 會顯示在哪一層被拒絕以及原因
```

### Q13.5 Discord Bot 離線
```
檢查步驟：
1. 確認 Bot Token 正確
2. 確認 Bot 已被邀請到伺服器
3. 檢查 Intents 設定
4. openclaw channel connect discord
5. 查看日誌中的 Discord 相關錯誤
```

### Q13.6 語音品質差
```
改善方法：
1. 確認網路連線穩定（延遲 < 100ms）
2. 使用更好的 STT 引擎（Azure > Whisper API）
3. 調整 VAD 參數（降低 threshold 提高靈敏度）
4. 使用更自然的 TTS 語音（HsiaoChenNeural + chat style）
5. 確認 Opus 編解碼器正確安裝
```

### Q13.7 費用超出預期
```
控制措施：
1. 設定 rate_limiting（每分鐘/每小時上限）
2. 使用 openclaw usage cost 監控
3. 簡單任務切換到便宜模型
4. 限制 max_tokens
5. 啟用工具結果快取
6. 考慮使用本地模型處理部分任務
```

---

> **返回**：[README](../README.md)
