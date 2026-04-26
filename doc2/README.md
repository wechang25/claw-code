# Claw Code 專案解析文件集

> 本資料夾包含 Claw Code 專案的完整技術解析文件，目的是讓開發者快速理解系統的每一個面向。

## 文件索引

| 文件 | 內容 |
|------|------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 系統架構總覽 — 分層設計、模組依賴、Mermaid 架構圖 |
| [SYSTEM_FLOW.md](./SYSTEM_FLOW.md) | 系統流程圖 — 關鍵流程的 Mermaid 序列圖與流程圖 |
| [DATA_MODEL.md](./DATA_MODEL.md) | 資料模型與 DB 關聯圖 — 所有核心資料結構的 ER 圖 |
| [MODULES.md](./MODULES.md) | 模組逐一解析 — 9 個 crate + 47 個子模組詳細說明 |
| [API_REFERENCE.md](./API_REFERENCE.md) | API 參考 — 多提供商客戶端、請求/回應格式 |
| [TOOLS_REFERENCE.md](./TOOLS_REFERENCE.md) | 工具系統參考 — 40 個內建工具的完整規格 |
| [MCP_ARCHITECTURE.md](./MCP_ARCHITECTURE.md) | MCP 子系統 — Model Context Protocol 完整架構 |
| [SECURITY_MODEL.md](./SECURITY_MODEL.md) | 安全模型 — 權限、沙箱、信任機制 |

## 專案一句話摘要

**Claw Code** 是一個用 Rust 實作的 **AI 代理人 CLI 工具框架**（代號 `claw`），讓 AI 模型能透過終端機直接操作開發環境（讀寫檔案、執行指令、搜尋程式碼），支援 Anthropic Claude、OpenAI、xAI 等多個提供商。

## 數字快覽

| 指標 | 值 |
|------|-----|
| Rust Crate 數量 | 9 |
| 總 Rust 程式碼行數 | ~48,600 LOC |
| 測試程式碼行數 | ~2,568 LOC |
| 內建工具數量 | 40 |
| 斜線命令數量 | 25+ |
| 支援的 AI 提供商 | 4（Anthropic, OpenAI, xAI, DashScope） |
| runtime 子模組數量 | 35+ |

---

*最後更新：2026-04-26*
