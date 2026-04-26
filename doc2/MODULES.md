# Claw Code 模組逐一解析

本文件詳細解析 9 個 crate 以及 runtime 中的 35+ 子模組。

---

## Crate 1：`rusty-claude-cli`（CLI 二進位）

**二進位名稱**：`claw`
**角色**：所有使用者面向的互動介面

### 子模組

| 模組 | 檔案 | 關鍵型別 | 一句話說明 |
|------|------|----------|-----------|
| 進入點 | `src/main.rs` | `ModelProvenance`, `ModelSource` | CLI 參數解析、模式判斷、REPL/一次性迴圈管理 |
| REPL 輸入 | `src/input.rs` | `ReadOutcome`, `SlashCommandHelper` | rustyline 整合、Tab 補全、快捷鍵處理 |
| 終端渲染 | `src/render.rs` | `TerminalRenderer`, `Spinner`, `ColorTheme` | Markdown → ANSI 渲染、語法高亮、進度動畫 |
| 初始化 | `src/init.rs` | `InitStatus`, `InitArtifact`, `InitReport` | `claw init` 建立 .claw.json 和 .gitignore |
| 建構腳本 | `build.rs` | - | 編譯時期注入版本等元資料 |

### 關鍵常數
- `DEFAULT_MODEL` = `"claude-opus-4-6"`
- `PRIMARY_SESSION_EXTENSION` = `"jsonl"`
- `LATEST_SESSION_REFERENCE` = `"latest"`

### 依賴
`api`, `commands`, `compat-harness`, `runtime`, `plugins`, `tools`, `crossterm`, `pulldown-cmark`, `rustyline`, `syntect`, `tokio`, `serde`, `serde_json`

---

## Crate 2：`runtime`（核心引擎）

**角色**：系統大腦，包含 35+ 子模組

### 對話核心

| 模組 | 關鍵型別 | 說明 |
|------|----------|------|
| `conversation.rs` | `ConversationRuntime<C,T>`, `ApiClient` (trait), `ToolExecutor` (trait), `TurnSummary`, `AssistantEvent` | 泛型對話迴圈引擎，協調 API 呼叫、工具執行、壓縮 |
| `session.rs` | `Session`, `ConversationMessage`, `ContentBlock`, `MessageRole`, `SessionCompaction` | JSONL 持久化的工作階段管理 |
| `compact.rs` | `CompactionConfig`, `CompactionResult` | 工作階段壓縮（預設閾值 100k tokens，保留最近 4 則訊息） |
| `prompt.rs` | `SystemPromptBuilder`, `ProjectContext`, `ContextFile` | 系統提示詞組裝（結合 Git、設定、指令檔） |

### 安全子系統

| 模組 | 關鍵型別 | 說明 |
|------|----------|------|
| `permissions.rs` | `PermissionMode`, `PermissionPolicy`, `PermissionOutcome`, `PermissionPrompter` (trait) | 五級權限模式 + 策略評估 |
| `permission_enforcer.rs` | `PermissionEnforcer`, `EnforcementResult` | 工具級權限強制執行 |
| `sandbox.rs` | `SandboxConfig`, `SandboxStatus`, `FilesystemIsolationMode`, `ContainerEnvironment` | Linux namespace 沙箱（PID/NET/FS 隔離） |
| `bash_validation.rs` | `ValidationResult`, `CommandIntent` | Bash 指令語意分類與驗證（寫入/破壞/網路偵測） |
| `trust_resolver.rs` | `TrustPolicy`, `TrustDecision`, `TrustConfig` | Worker 信任閘門（自動/需核准/拒絕） |

### 工具執行

| 模組 | 關鍵型別 | 說明 |
|------|----------|------|
| `bash.rs` | `BashCommandInput`, `BashCommandOutput` | 沙箱化 Shell 指令執行（含逾時、背景執行） |
| `file_ops.rs` | `ReadFileOutput`, `WriteFileOutput`, `EditFileOutput`, `GlobSearchOutput`, `GrepSearchInput` | 檔案讀寫編輯搜尋（含大小限制、二進位偵測、路徑安全） |
| `git_context.rs` | `GitContext`, `GitCommitEntry` | Git 分支/提交/暫存區偵測 |

### MCP 子系統

| 模組 | 關鍵型別 | 說明 |
|------|----------|------|
| `mcp.rs` | 工具函式 | MCP 命名規範化、簽章、前綴 |
| `mcp_client.rs` | `McpClientTransport`, `McpClientBootstrap`, `McpClientAuth` | 傳輸層抽象（Stdio/SSE/HTTP/WS/SDK/Proxy） |
| `mcp_stdio.rs` | `McpStdioProcess`, `McpServerManager`, `McpTool`, `JsonRpcRequest` | Stdio 傳輸的子程序管理與 JSON-RPC 通訊 |
| `mcp_server.rs` | `McpServer`, `McpServerSpec`, `ToolCallHandler` (trait) | 本地 MCP 伺服器實作 |
| `mcp_tool_bridge.rs` | `McpToolRegistry` | MCP 伺服器 → 工具系統的橋接 |
| `mcp_lifecycle_hardened.rs` | `McpLifecyclePhase`, `McpLifecycleState`, `McpDegradedReport` | 11 階段生命週期狀態機 |

### 設定與狀態

| 模組 | 關鍵型別 | 說明 |
|------|----------|------|
| `config.rs` | `RuntimeConfig`, `ConfigLoader`, `RuntimeFeatureConfig`, `McpServerConfig` | 多來源設定合併（user → project → local） |
| `config_validate.rs` | `ConfigDiagnostic`, `DiagnosticKind` | 設定驗證與診斷 |
| `usage.rs` | `ModelPricing`, `TokenUsage`, `UsageTracker`, `UsageCostEstimate` | Token 用量追蹤與成本估算 |
| `hooks.rs` | `HookRunner`, `HookEvent`, `HookAbortSignal`, `HookProgressReporter` (trait) | 工具前/後勾子命令執行 |
| `oauth.rs` | `OAuthTokenSet`, `PkceCodePair` | PKCE S256 OAuth 流程 |
| `session_control.rs` | `SessionStore` | 工作階段存儲管理與工作區指紋 |

### 工作流程引擎

| 模組 | 關鍵型別 | 說明 |
|------|----------|------|
| `lane_events.rs` | `LaneEvent`, `LaneEventName`(20+ 變體), `LaneFailureClass`(12 類) | CI/CD Lane 工作流程事件 |
| `worker_boot.rs` | `Worker`, `WorkerStatus`, `WorkerRegistry`, `WorkerFailureKind` | Worker 生命週期狀態機 |
| `policy_engine.rs` | `PolicyEngine`, `PolicyRule`, `PolicyCondition`, `PolicyAction` | Lane 策略評估引擎 |
| `recovery_recipes.rs` | `FailureScenario`(7 類), `RecoveryRecipe`, `RecoveryStep` | 自動化錯誤復原方案 |
| `task_registry.rs` | `TaskRegistry`, `Task`, `TaskStatus` | 記憶體任務生命週期管理 |
| `team_cron_registry.rs` | `TeamRegistry`, `CronRegistry`, `Team`, `CronTask` | 團隊與排程管理 |
| `stale_branch.rs` | `BranchFreshness`, `StaleBranchPolicy`, `StaleBranchAction` | 分支新鮮度偵測與策略 |
| `stale_base.rs` | `BaseCommitState`, `BaseCommitSource` | 基底提交新鮮度 |
| `branch_lock.rs` | `BranchLockIntent`, `BranchLockCollision` | 分支鎖定衝突偵測 |
| `green_contract.rs` | `GreenLevel`(4 級), `GreenContract` | 測試品質閘門 |
| `task_packet.rs` | `TaskPacket`, `TaskScope` | 任務封包定義與驗證 |

### 其他

| 模組 | 關鍵型別 | 說明 |
|------|----------|------|
| `remote.rs` | `RemoteSessionContext`, `UpstreamProxyBootstrap` | 遠端會話與上游代理 |
| `sse.rs` | `IncrementalSseParser`, `SseEvent` | 增量 SSE 幀解析 |
| `lsp_client.rs` | `LspRegistry`, `LspAction`(7 種), `LspDiagnostic` | LSP 客戶端整合 |
| `plugin_lifecycle.rs` | `PluginState`(8 種), `ServerHealth` | 外掛健康狀態追蹤 |
| `summary_compression.rs` | `SummaryCompressionBudget`, `SummaryCompressionResult` | 摘要文字壓縮（上限 1200 字元/24 行） |
| `bootstrap.rs` | `BootstrapPhase`, `BootstrapPlan` | 啟動計畫 |
| `json.rs` | `JsonValue` | JSON 工具輔助 |

---

## Crate 3：`api`（API 客戶端）

**角色**：多提供商 AI API 通訊

| 模組 | 關鍵型別 | 說明 |
|------|----------|------|
| `client.rs` | `ProviderClient`(3 variant), `MessageStream` | 統一多提供商客戶端 |
| `providers/mod.rs` | `ProviderKind`, `detect_provider_kind()`, `resolve_model_alias()` | 提供商偵測與模型別名 |
| `providers/anthropic.rs` | `AnthropicClient`, `AuthSource` | Anthropic Messages API |
| `providers/openai_compat.rs` | `OpenAiCompatClient`, `OpenAiCompatConfig` | OpenAI Chat Completions 相容 |
| `types.rs` | `MessageRequest`, `MessageResponse`, `ToolDefinition`, `Usage`, `StreamEvent` | 請求/回應資料結構 |
| `sse.rs` | `SseParser`, `parse_frame()` | 提供商感知的 SSE 解析 |
| `error.rs` | `ApiError` | API 錯誤型別 |
| `http_client.rs` | `ProxyConfig`, `build_http_client()` | HTTP 客戶端建構（含代理） |
| `prompt_cache.rs` | `PromptCache`, `PromptCacheStats` | Anthropic 提示詞快取 |

### 模型別名表

| 別名 | 解析結果 | 提供商 |
|------|----------|--------|
| `opus` | `claude-opus-4-6` | Anthropic |
| `sonnet` | `claude-sonnet-4-6` | Anthropic |
| `haiku` | `claude-haiku-4-5-20251213` | Anthropic |
| `grok` | `grok-3` | xAI |
| `grok-mini` | `grok-3-mini` | xAI |

---

## Crate 4：`tools`（工具分派）

**角色**：40 個工具的統一分派中樞

### 內建工具清單（40 個）

| 類別 | 工具 |
|------|------|
| Shell | `Bash`, `PowerShell`, `REPL` |
| 檔案 | `ReadFile`, `WriteFile`, `EditFile` |
| 搜尋 | `GlobSearch`, `GrepSearch`, `ToolSearch` |
| 網路 | `WebFetch`, `WebSearch` |
| 任務 | `TaskCreate`, `TaskGet`, `TaskList`, `TaskStop`, `TaskUpdate`, `TaskOutput` |
| 團隊 | `TeamCreate`, `TeamDelete` |
| 排程 | `CronCreate`, `CronDelete`, `CronList` |
| MCP | `ListMcpResources`, `ReadMcpResource`, `McpAuth`, `MCP` |
| LSP | `LSP` |
| Agent | `Agent`, `Skill`, `Sleep` |
| 互動 | `AskUserQuestion`, `SendUserMessage` |
| 筆記 | `TodoWrite`, `NotebookEdit` |
| 設定 | `Config`, `StructuredOutput` |
| 規劃 | `EnterPlanMode`, `ExitPlanMode` |
| 其他 | `RemoteTrigger`, `TestingPermission` |

---

## Crate 5：`commands`（斜線命令）

**角色**：REPL 斜線命令

### 已註冊命令

| 命令 | 別名 | 說明 |
|------|------|------|
| `/help` | `/h` | 顯示幫助 |
| `/status` | `/s` | 顯示狀態 |
| `/sandbox` | - | 沙箱狀態 |
| `/compact` | - | 壓縮工作階段 |
| `/model` | - | 切換模型 |
| `/permissions` | `/perm` | 權限模式 |
| `/agents` | - | 代理人清單 |
| `/mcp` | - | MCP 伺服器 |
| `/plugins` | `/marketplace` | 外掛管理 |
| `/skills` | - | 技能清單 |
| `/doctor` | - | 健康檢查 |
| `/resume` | - | 續接工作階段 |
| `/clear` | - | 清除上下文 |
| `/quit` | `/exit` | 退出 |
| `/cost` | - | 費用統計 |
| `/config` | - | 設定顯示 |
| `/session` | - | 工作階段資訊 |
| `/diff` | - | Git diff |
| `/commit` | - | Git commit |
| `/pr` | - | 建立 PR |
| `/review` | - | 程式碼審查 |
| `/subagent` | - | 子代理人管理 |
| `/ultraplan` | - | 深度規劃 |
| `/teleport` | - | 跳轉到檔案/符號 |
| `/bughunter` | - | 掃描 bug |

---

## Crate 6：`plugins`（外掛系統）

**角色**：外掛生命週期管理

### 核心概念

- **PluginKind**：`Builtin`（內建）、`Bundled`（隨附）、`External`（外部）
- **PluginPermission**：`Read`、`Write`、`Execute`
- **Hooks**：`PreToolUse`、`PostToolUse`、`PostToolUseFailure`

### 外掛目錄結構
```
plugins/
├── plugin.json        # 清單檔
├── src/
│   ├── init.sh        # 初始化腳本
│   ├── shutdown.sh    # 關閉腳本
│   └── tools/
│       └── my_tool.sh # 工具實作
└── hooks/
    ├── pre_tool_use.sh
    └── post_tool_use.sh
```

---

## Crate 7：`telemetry`（遙測）

**角色**：事件追蹤與分析

### 事件類型
- `HttpRequestStarted` — HTTP 請求開始
- `HttpRequestSucceeded` — HTTP 請求成功
- `HttpRequestFailed` — HTTP 請求失敗
- `Analytics` — 分析事件
- `SessionTrace` — 工作階段追蹤記錄

### Sink 實作
- `MemoryTelemetrySink` — 記憶體（測試用）
- `JsonlTelemetrySink` — JSONL 檔案

---

## Crate 8：`mock-anthropic-service`（測試模擬）

**角色**：Anthropic API 模擬服務

### 測試場景
1. `StreamingText` — 串流文字回應
2. `ReadFileRoundtrip` — 讀檔往返
3. `GrepChunkAssembly` — Grep 分塊組裝
4. `WriteFileAllowed` — 允許寫檔
5. `WriteFileDenied` — 拒絕寫檔
6. `MultiToolTurnRoundtrip` — 多工具回合
7. `BashStdoutRoundtrip` — Bash 輸出往返
8. `BashPermissionPromptApproved` — Bash 權限核准
9. `BashPermissionPromptDenied` — Bash 權限拒絕
10. `PluginToolRoundtrip` — 外掛工具往返

---

## Crate 9：`compat-harness`（相容橋接）

**角色**：上游 TypeScript 清單提取

### 核心功能
- 從上游 TS 原始碼提取工具/命令/啟動清單
- 產生 `ExtractedManifest` 供 Rust 端對齊
- 用於 parity 檢查（確保功能對等）

---

*本文件基於 claw-code 專案原始碼分析產生 — 2026-04-26*
