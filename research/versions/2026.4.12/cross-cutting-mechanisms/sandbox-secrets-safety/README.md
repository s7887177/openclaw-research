# 沙箱、秘密管理與安全邊界（Sandbox, Secrets & Safety）

本目錄剖析 OpenClaw 的安全三角：**沙箱隔離**（限制 agent 執行環境）、**秘密管理**（安全注入敏感資料）、以及**安全邊界**（SSRF 防護、執行政策、信任模型）。這些機制共同構成 OpenClaw 的縱深防禦體系。

## 章節索引

| 檔案 | 主題 | 字數概估 |
|------|------|----------|
| [01-sandbox-and-exec-policy.md](./01-sandbox-and-exec-policy.md) | 沙箱模式、Docker 容器設定、執行政策 | ~8,000 |
| [02-secrets-ssrf-trust.md](./02-secrets-ssrf-trust.md) | 秘密解析、SSRF 防護、信任模型、安全稽核 | ~8,000 |

## 核心概念速覽

- **Sandbox Mode**：`"off"` / `"non-main"` / `"all"` 三級沙箱策略
- **Docker Sandbox**：基於容器的程序隔離（readOnly root、tmpfs、capability drop）
- **Exec Policy**：`"yolo"` / `"cautious"` / `"deny-all"` 三種執行政策預設
- **Secret Resolution**：解析 `$SECRET_REF` 引用為實際值的安全管道
- **SSRF Protection**：防止伺服器端請求偽造的 DNS pinning + IP 黑名單
- **Trust Model**：基於群組曝光度 × 工具權限的動態信任評估

## 關鍵原始碼路徑

| 路徑 | 說明 |
|------|------|
| `src/config/types.agents-shared.ts` | 沙箱模式定義 |
| `src/config/types.sandbox.ts` | Docker 沙箱配置型別 |
| `Dockerfile.sandbox*` | 沙箱容器映像 |
| `src/secrets/` | 秘密管理完整實作 |
| `src/infra/net/ssrf.ts` | SSRF 防護 |
| `src/infra/exec-approvals.ts` | 執行審批型別 |
| `src/cli/exec-policy-cli.ts` | 執行政策 CLI |
| `src/infra/exec-safe-bin-trust.ts` | 安全二進位信任目錄 |
| `src/security/audit.types.ts` | 安全稽核型別 |
