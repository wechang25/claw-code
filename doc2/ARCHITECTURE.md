# Claw Code 系統架構深度解析

## 1. 高層架構總覽

Claw Code 採用**分層式 Cargo Workspace Monorepo** 架構，共 9 個 crate，嚴格遵循由上到下的依賴方向。

```mermaid
graph TB
    subgraph "使用者介面層"
        CLI["🖥️ rusty-claude-cli<br/>（CLI 二進位：claw）<br/>REPL・一次性指令・子命令"]
    end

    subgraph "協調層"
        CMD["📋 commands<br/>斜線命令註冊與分派"]
        CH["🔗 compat-harness<br/>上游 TS 相容橋接"]
    end

    subgraph "核心引擎層"
        RT["⚙️ runtime<br/>對話迴圈・工作階段・設定<br/>權限・MCP・沙箱<br/>35+ 子模組"]
        TOOLS["🔧 tools<br/>工具分派中樞<br/>40 個工具規格"]
    end

    subgraph "通訊層"
        API["🌐 api<br/>多提供商 API 客戶端<br/>SSE 串流・認證"]
    end

    subgraph "支援層"
        PLG["🧩 plugins<br/>外掛管理・勾子系統"]
        TEL["📊 telemetry<br/>遙測・追蹤・分析"]
        MOCK["🧪 mock-anthropic-service<br/>測試用模擬服務"]
    end

    subgraph "外部服務"
        ANTHROPIC["☁️ Anthropic API"]
        OPENAI["☁️ OpenAI API"]
        XAI["☁️ xAI API"]
        DASHSCOPE["☁️ DashScope API"]
        MCPSERV["🔌 MCP Servers"]
    end

    CLI --> CMD
    CLI --> CH
    CLI --> RT
    CLI --> TOOLS
    CLI --> API
    CLI --> PLG

    CMD --> RT
    CMD --> TOOLS

    TOOLS --> API
    TOOLS --> RT
    TOOLS --> PLG

    API --> RT
    API --> TEL

    RT --> PLG
    RT --> TEL

    API --> ANTHROPIC
    API --> OPENAI
    API --> XAI
    API --> DASHSCOPE

    RT --> MCPSERV

    MOCK -.->|測試時替代| API

    style CLI fill:#e3f2fd,stroke:#1565c0
    style RT fill:#fff3e0,stroke:#e65100
    style TOOLS fill:#fce4ec,stroke:#c62828
    style API fill:#e8f5e9,stroke:#2e7d32
    style PLG fill:#f3e5f5,stroke:#6a1b9a
    style TEL fill:#e0f7fa,stroke:#00838f
```

## 2. Crate 依賴矩陣

| Crate | 依賴 |
|-------|------|
| `rusty-claude-cli` | api, commands, compat-harness, runtime, plugins, tools |
| `tools` | api, runtime, plugins |
| `commands` | runtime, tools |
| `api` | runtime, telemetry |
| `runtime` | plugins, telemetry |
| `compat-harness` | commands, runtime, tools |
| `plugins` | (無內部依賴) |
| `telemetry` | (無內部依賴) |
| `mock-anthropic-service` | api (dev 依賴) |

## 3. 分層詳解

### 3.1 使用者介面層（`rusty-claude-cli`）

**職責**：所有使用者可見的互動面。

| 模組 | 檔案 | 職責 |
|------|------|------|
| 進入點 | `src/main.rs` | CLI 參數解析、模式判斷、工作階段管理 |
| REPL 輸入 | `src/input.rs` | rustyline 整合、Tab 補全、快捷鍵 |
| 終端渲染 | `src/render.rs` | Markdown → ANSI、語法高亮、Spinner 動畫 |
| 專案初始化 | `src/init.rs` | `claw init` 指令 |
| Build 腳本 | `build.rs` | 編譯時期元資料注入 |

**關鍵設計決策**：
- 使用 `crossterm` 跨平台終端控制
- `pulldown-cmark` + `syntect` 實現 Markdown 渲染 + 語法高亮
- `rustyline` 提供 GNU readline 相容的 REPL 體驗
- `ModelProvenance` 結構追蹤模型來源（flag → env → config → default）

### 3.2 核心引擎層（`runtime`）

**職責**：整個系統的「大腦」，包含 35+ 子模組。

```mermaid
graph LR
    subgraph runtime["runtime crate（35+ 模組）"]
        direction TB

        subgraph 對話核心
            CONV["conversation.rs<br/>對話迴圈"]
            SESSION["session.rs<br/>工作階段"]
            COMPACT["compact.rs<br/>壓縮"]
            PROMPT["prompt.rs<br/>提示詞建構"]
        end

        subgraph 安全
            PERM["permissions.rs<br/>權限模式"]
            ENFORCER["permission_enforcer.rs<br/>強制執行"]
            SANDBOX["sandbox.rs<br/>沙箱"]
            BASH_VAL["bash_validation.rs<br/>指令驗證"]
        end

        subgraph 工具執行
            BASH["bash.rs<br/>Shell 執行"]
            FILE_OPS["file_ops.rs<br/>檔案操作"]
            GIT_CTX["git_context.rs<br/>Git 上下文"]
        end

        subgraph MCP 子系統
            MCP["mcp.rs<br/>工具函式"]
            MCP_CLI["mcp_client.rs<br/>客戶端"]
            MCP_STDIO["mcp_stdio.rs<br/>Stdio 傳輸"]
            MCP_SRV["mcp_server.rs<br/>伺服器"]
            MCP_BRIDGE["mcp_tool_bridge.rs<br/>工具橋接"]
            MCP_LIFE["mcp_lifecycle_hardened.rs<br/>生命週期"]
        end

        subgraph 設定與狀態
            CONFIG["config.rs<br/>設定載入"]
            CONFIG_VAL["config_validate.rs<br/>設定驗證"]
            USAGE["usage.rs<br/>用量追蹤"]
            HOOKS["hooks.rs<br/>勾子系統"]
        end

        subgraph 工作流程
            LANE["lane_events.rs<br/>Lane 事件"]
            WORKER["worker_boot.rs<br/>Worker 啟動"]
            POLICY["policy_engine.rs<br/>策略引擎"]
            RECOVERY["recovery_recipes.rs<br/>錯誤復原"]
            TASK_REG["task_registry.rs<br/>任務註冊"]
            TEAM_CRON["team_cron_registry.rs<br/>團隊/排程"]
        end

        subgraph 其他
            OAUTH["oauth.rs<br/>OAuth 認證"]
            REMOTE["remote.rs<br/>遠端會話"]
            SSE_RT["sse.rs<br/>SSE 解析"]
            LSP["lsp_client.rs<br/>LSP 客戶端"]
            TRUST["trust_resolver.rs<br/>信任解析"]
            STALE_BR["stale_branch.rs<br/>分支新鮮度"]
            STALE_BASE["stale_base.rs<br/>基底提交"]
            BRANCH_LOCK["branch_lock.rs<br/>分支鎖"]
            GREEN["green_contract.rs<br/>綠燈契約"]
            TASK_PKT["task_packet.rs<br/>任務封包"]
            PLUGIN_LIFE["plugin_lifecycle.rs<br/>外掛生命週期"]
            SESSION_CTL["session_control.rs<br/>會話控制"]
            SUMMARY["summary_compression.rs<br/>摘要壓縮"]
            BOOTSTRAP["bootstrap.rs<br/>啟動計畫"]
            JSON["json.rs<br/>JSON 工具"]
        end
    end
```

### 3.3 工具層（`tools`）

**職責**：統一工具分派中樞，橋接 40 個工具到各自的執行器。

| 工具類別 | 工具名稱 | 執行器 |
|----------|----------|--------|
| 核心 I/O | `bash`, `read_file`, `write_file`, `edit_file` | runtime (bash.rs, file_ops.rs) |
| 搜尋 | `glob_search`, `grep_search` | runtime (file_ops.rs) |
| 網路 | `web_fetch`, `web_search` | tools (內建) |
| 工作流 | `task_*`, `team_*`, `cron_*` | runtime registries |
| MCP | `mcp__*__*` | runtime MCP bridge |
| LSP | `lsp` | runtime LSP registry |
| 外掛 | 動態載入 | plugins crate |
| 其他 | `agent`, `skill`, `todo_write`, `notebook_edit`, `sleep` 等 | 各自實作 |

### 3.4 通訊層（`api`）

```mermaid
graph TD
    PC["ProviderClient"]

    PC -->|"claude-*"| AC["AnthropicClient<br/>Anthropic Messages API"]
    PC -->|"grok-*"| XC["OpenAiCompatClient<br/>xAI endpoint"]
    PC -->|"qwen-*"| DC["OpenAiCompatClient<br/>DashScope endpoint"]
    PC -->|"其他"| OC["OpenAiCompatClient<br/>OpenAI/Ollama endpoint"]

    AC --> SSE_A["SSE 串流解析<br/>（Anthropic 格式）"]
    XC --> SSE_O["SSE 串流解析<br/>（OpenAI 格式）"]
    DC --> SSE_O
    OC --> SSE_O

    AC --> CACHE["PromptCache<br/>提示詞快取"]

    style PC fill:#e8f5e9
    style AC fill:#fff3e0
    style XC fill:#e3f2fd
    style DC fill:#fce4ec
    style OC fill:#f3e5f5
```

## 4. 跨層互動模式

### 4.1 泛型 Runtime 模式

```rust
// runtime/src/conversation.rs
pub struct ConversationRuntime<C: ApiClient, T: ToolExecutor> { ... }
```

- `C: ApiClient` — 可注入真實客戶端或 mock
- `T: ToolExecutor` — 可注入真實工具或 mock
- 使得整個對話引擎可獨立測試

### 4.2 全域 Registry 模式

```rust
// tools/src/lib.rs — 使用 OnceLock 的懶初始化全域單例
fn global_task_registry() -> &'static TaskRegistry {
    static REGISTRY: OnceLock<TaskRegistry> = OnceLock::new();
    REGISTRY.get_or_init(TaskRegistry::new)
}
```

6 個全域 Registry：LSP、MCP、Task、Team、Cron、Worker。

### 4.3 設定合併鏈

```
優先級（低 → 高）：
~/.claw.json → ~/.config/claw/settings.json → <repo>/.claw.json → <repo>/.claw/settings.json → <repo>/.claw/settings.local.json
```

## 5. 建構與部署架構

```mermaid
graph LR
    SRC["原始碼<br/>rust/crates/*"] --> CARGO["cargo build<br/>--workspace"]
    CARGO --> BIN["claw 二進位<br/>target/debug/claw"]
    BIN --> INSTALL["安裝方式"]
    INSTALL --> SYM["符號連結<br/>/usr/local/bin/claw"]
    INSTALL --> CARGO_INSTALL["cargo install<br/>--path ."]
    INSTALL --> CONTAINER["容器<br/>Containerfile"]

    SRC --> CI["GitHub Actions"]
    CI --> FMT["cargo fmt"]
    CI --> CLIPPY["cargo clippy"]
    CI --> TEST["cargo test --workspace"]
    CI --> DOC_CHECK["文件一致性檢查"]
```

## 6. 關鍵設計原則

1. **unsafe_code = "forbid"** — 整個工作區禁止 unsafe 程式碼
2. **clippy::pedantic** — 啟用最嚴格的 clippy 檢查
3. **Trait-based 多態** — 所有主要擴充點都用 trait 定義
4. **沙箱優先** — 預設啟用 Linux namespace 隔離
5. **多提供商不鎖定** — 模型名稱前綴路由，不綁定單一 AI 服務
6. **JSONL 持久化** — 工作階段使用追加寫入的 JSONL 格式，支援旋轉
7. **零外部資料庫** — 所有狀態以檔案系統管理

---

*本文件基於 claw-code 專案原始碼分析產生 — 2026-04-26*
