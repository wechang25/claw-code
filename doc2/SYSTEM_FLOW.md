# Claw Code 系統流程圖

本文件以 Mermaid 圖表呈現 Claw Code 的所有關鍵系統流程。

---

## 1. 應用程式啟動流程

```mermaid
flowchart TD
    START["🚀 使用者執行 claw"] --> PARSE["解析 CLI 參數<br/>main.rs"]
    PARSE --> MODE{"模式判斷"}

    MODE -->|"claw prompt ..."| ONESHOT["一次性模式"]
    MODE -->|"claw（無參數）"| REPL["互動式 REPL"]
    MODE -->|"claw doctor/status/init/..."| SUBCMD["子命令"]

    ONESHOT --> INIT_COMMON["共用初始化流程"]
    REPL --> INIT_COMMON

    INIT_COMMON --> LOAD_CONFIG["1. ConfigLoader::discover()<br/>載入設定鏈"]
    LOAD_CONFIG --> RESOLVE_MODEL["2. 解析模型<br/>（flag → env → config → default）"]
    RESOLVE_MODEL --> RESOLVE_AUTH["3. 解析認證<br/>（API key → OAuth → Bearer）"]
    RESOLVE_AUTH --> LOAD_PLUGINS["4. PluginManager::load_plugins()<br/>載入外掛"]
    LOAD_PLUGINS --> MCP_BOOT["5. MCP 伺服器啟動<br/>（如果設定中有 MCP）"]
    MCP_BOOT --> BUILD_PROMPT["6. SystemPromptBuilder<br/>組裝系統提示詞"]
    BUILD_PROMPT --> SESSION_INIT["7. Session 初始化/續接<br/>（--resume 或新建）"]
    SESSION_INIT --> PERM_SETUP["8. PermissionPolicy 設定<br/>（模式 + 規則）"]

    PERM_SETUP --> READY["✅ 就緒"]

    READY -->|一次性| SEND_PROMPT["送出使用者提示詞"]
    READY -->|REPL| ENTER_REPL["進入 REPL 迴圈"]

    SUBCMD --> EXEC_SUBCMD["執行子命令邏輯"]
    EXEC_SUBCMD --> EXIT["退出"]

    style START fill:#e8f5e9
    style READY fill:#c8e6c9
```

---

## 2. 對話迴圈（ConversationRuntime 核心）

```mermaid
sequenceDiagram
    actor User as 👤 使用者
    participant CLI as CLI (main.rs)
    participant RT as ConversationRuntime
    participant Hook as HookRunner
    participant API as ProviderClient
    participant SSE as SSE Parser
    participant Provider as AI Provider
    participant Perm as PermissionPolicy
    participant Tool as execute_tool()
    participant FS as 本地環境

    User->>CLI: 輸入提示詞
    CLI->>RT: add_user_message(text)
    RT->>RT: build_api_request()
    Note over RT: 組合 system_prompt + session.messages

    RT->>API: stream(ApiRequest)
    API->>Provider: POST /v1/messages (SSE)

    loop SSE 串流事件迴圈
        Provider-->>SSE: event: content_block_delta
        SSE-->>CLI: AssistantEvent::TextDelta
        CLI-->>User: 即時渲染到終端
    end

    alt 模型請求工具呼叫
        Provider-->>SSE: event: tool_use {name, input}
        SSE-->>RT: AssistantEvent::ToolUse

        RT->>Hook: run_pre_tool_use(name, input)
        Hook-->>RT: HookRunResult (allow/deny/ask)

        RT->>Perm: authorize(tool_name, mode)
        Perm-->>RT: PermissionOutcome

        alt 權限允許
            RT->>Tool: execute_tool(name, input)
            Tool->>FS: 執行操作
            FS-->>Tool: 結果
            Tool-->>RT: ToolResult(output)

            RT->>Hook: run_post_tool_use(name, input, output)
        else 權限拒絕
            RT-->>RT: ToolResult(error: "permission denied")
        end

        RT->>RT: add_tool_result_to_session()
        RT->>API: 繼續對話（含工具結果）
        Note over RT: 回到 SSE 串流迴圈
    end

    Provider-->>SSE: event: message_stop
    SSE-->>RT: AssistantEvent::MessageStop
    RT->>RT: usage_tracker.add(usage)

    opt 自動壓縮檢查
        RT->>RT: should_compact(threshold: 100k tokens)
        RT->>RT: compact_session()
    end

    RT->>RT: session.save()
    RT-->>CLI: TurnSummary
    CLI-->>User: 顯示完成
```

---

## 3. 工具執行分派流程

```mermaid
flowchart TD
    CALL["模型發出工具呼叫<br/>ToolUse {name, input}"] --> DISPATCH{"execute_tool()<br/>tools/src/lib.rs"}

    DISPATCH -->|"bash"| BASH["execute_bash()<br/>runtime/bash.rs"]
    DISPATCH -->|"read_file"| READ["read_file()<br/>runtime/file_ops.rs"]
    DISPATCH -->|"write_file"| WRITE["write_file()<br/>runtime/file_ops.rs"]
    DISPATCH -->|"edit_file"| EDIT["edit_file()<br/>runtime/file_ops.rs"]
    DISPATCH -->|"glob_search"| GLOB["glob_search()<br/>runtime/file_ops.rs"]
    DISPATCH -->|"grep_search"| GREP["grep_search()<br/>runtime/file_ops.rs"]
    DISPATCH -->|"web_fetch/web_search"| WEB["HTTP 工具"]
    DISPATCH -->|"task_*"| TASK["TaskRegistry"]
    DISPATCH -->|"team_*"| TEAM["TeamRegistry"]
    DISPATCH -->|"cron_*"| CRON["CronRegistry"]
    DISPATCH -->|"lsp"| LSP["LspRegistry"]
    DISPATCH -->|"mcp__*__*"| MCP["McpToolRegistry"]
    DISPATCH -->|"外掛工具"| PLUGIN["PluginTool::execute()"]
    DISPATCH -->|"其他"| OTHER["Agent/Skill/Todo 等"]

    BASH --> SANDBOX{"沙箱檢查"}
    SANDBOX -->|"啟用"| UNSHARE["build_linux_sandbox_command()<br/>namespace 隔離"]
    SANDBOX -->|"停用"| PLAIN["直接執行 Command"]
    UNSHARE --> EXEC["tokio::process::Command"]
    PLAIN --> EXEC
    EXEC --> TIMEOUT["timeout 控制<br/>（預設 120s）"]
    TIMEOUT --> OUTPUT["BashCommandOutput<br/>{stdout, stderr, interrupted}"]

    READ --> VALIDATE["validate_workspace_boundary()"]
    VALIDATE --> CHECK_BIN["is_binary_file()"]
    CHECK_BIN --> CHECK_SIZE["MAX_READ_SIZE (10MB)"]
    CHECK_SIZE --> RESULT["ReadFileOutput"]

    style DISPATCH fill:#e3f2fd
    style BASH fill:#fff3e0
    style READ fill:#e8f5e9
```

---

## 4. 設定載入流程

```mermaid
flowchart LR
    subgraph "設定來源（優先級由低到高）"
        U["~/.claw.json<br/>使用者全域"]
        UC["~/.config/claw/settings.json<br/>使用者設定"]
        P[".claw.json<br/>專案根"]
        PS[".claw/settings.json<br/>專案設定"]
        PL[".claw/settings.local.json<br/>本地覆蓋"]
    end

    U --> MERGE["ConfigLoader::discover()"]
    UC --> MERGE
    P --> MERGE
    PS --> MERGE
    PL --> MERGE

    MERGE --> VALIDATE["config_validate.rs<br/>驗證設定"]
    VALIDATE --> RC["RuntimeConfig"]

    RC --> FC["RuntimeFeatureConfig"]
    FC --> HOOKS_CFG["RuntimeHookConfig"]
    FC --> PLUGIN_CFG["RuntimePluginConfig"]
    FC --> MCP_CFG["McpConfigCollection"]
    FC --> OAUTH_CFG["OAuthConfig"]
    FC --> PERM_CFG["RuntimePermissionRuleConfig"]
    FC --> SANDBOX_CFG["SandboxConfig"]
    FC --> ALIAS_CFG["aliases: BTreeMap"]
    FC --> FALLBACK_CFG["ProviderFallbackConfig"]
```

---

## 5. 工作階段（Session）生命週期

```mermaid
stateDiagram-v2
    [*] --> Created: Session::new()
    Created --> Active: 第一則訊息
    Active --> Active: add_message()
    Active --> Compacted: compact_session()<br/>token 超過 100k
    Compacted --> Active: 繼續對話
    Active --> Saved: session.save()<br/>JSONL 持久化
    Saved --> Resumed: --resume latest
    Resumed --> Active: 載入歷史訊息

    Active --> Forked: fork()<br/>分支會話
    Forked --> Active: 在新分支繼續

    Saved --> Rotated: 超過 256KB
    Rotated --> Saved: 保留最新 3 個檔案

    Active --> [*]: 使用者退出
```

---

## 6. 提供商路由流程

```mermaid
flowchart TD
    INPUT["使用者指定模型<br/>--model X"] --> ALIAS["resolve_model_alias(X)"]
    ALIAS --> DETECT{"detect_provider_kind()"}

    DETECT -->|"claude-*"| ANTHROPIC["ProviderKind::Anthropic"]
    DETECT -->|"grok-*"| XAI["ProviderKind::Xai"]
    DETECT -->|"qwen-* / qwen/*"| DASHSCOPE["ProviderKind::OpenAi<br/>（DashScope 路由）"]
    DETECT -->|"openai/* / gpt-*"| OPENAI["ProviderKind::OpenAi"]
    DETECT -->|"其他"| FALLBACK{"檢查環境變數"}

    FALLBACK -->|"ANTHROPIC_API_KEY"| ANTHROPIC
    FALLBACK -->|"OPENAI_API_KEY"| OPENAI
    FALLBACK -->|"XAI_API_KEY"| XAI
    FALLBACK -->|"DASHSCOPE_API_KEY"| DASHSCOPE
    FALLBACK -->|"都沒有"| ANTHROPIC_DEFAULT["預設 Anthropic"]

    ANTHROPIC --> AC["AnthropicClient"]
    XAI --> XC["OpenAiCompatClient<br/>base_url: api.x.ai"]
    OPENAI --> OC["OpenAiCompatClient<br/>base_url: api.openai.com"]
    DASHSCOPE --> DC["OpenAiCompatClient<br/>base_url: dashscope.aliyuncs.com"]
    ANTHROPIC_DEFAULT --> AC

    AC --> STREAM["stream_message()<br/>Anthropic SSE 格式"]
    XC --> STREAM_OAI["stream_message()<br/>OpenAI SSE 格式"]
    OC --> STREAM_OAI
    DC --> STREAM_OAI

    style INPUT fill:#e3f2fd
    style DETECT fill:#fff9c4
```

---

## 7. 沙箱執行流程

```mermaid
flowchart TD
    CMD["bash 指令"] --> CHECK{"resolve_sandbox_status()"}

    CHECK -->|"Linux + 非容器"| SANDBOX["建構沙箱指令"]
    CHECK -->|"容器內/macOS/Windows"| DIRECT["直接執行"]
    CHECK -->|"dangerouslyDisableSandbox"| DIRECT

    SANDBOX --> UNSHARE["unshare 指令建構"]
    UNSHARE --> NS_PID["--pid：PID 命名空間"]
    UNSHARE --> NS_NET{"isolateNetwork?"}
    NS_NET -->|"是"| NET_ISO["--net：網路隔離"]
    NS_NET -->|"否"| NET_PASS["保留網路存取"]
    UNSHARE --> NS_FS{"filesystemMode?"}
    NS_FS -->|"WorkspaceOnly"| FS_WS["掛載限制在工作區"]
    NS_FS -->|"AllowList"| FS_AL["依 allowedMounts 掛載"]
    NS_FS -->|"Off"| FS_OFF["無檔案系統限制"]

    NET_ISO --> EXEC["執行隔離指令"]
    NET_PASS --> EXEC
    FS_WS --> EXEC
    FS_AL --> EXEC
    FS_OFF --> EXEC
    DIRECT --> EXEC

    EXEC --> TIMEOUT["timeout<br/>（預設 120s）"]
    TIMEOUT --> OUTPUT["BashCommandOutput"]
```

---

## 8. MCP 伺服器生命週期

```mermaid
stateDiagram-v2
    [*] --> ConfigLoad: 讀取 MCP 設定
    ConfigLoad --> ServerRegistration: 註冊伺服器規格
    ServerRegistration --> SpawnConnect: 啟動/連接程序

    SpawnConnect --> InitializeHandshake: 發送 initialize
    InitializeHandshake --> ToolDiscovery: tools/list
    ToolDiscovery --> ResourceDiscovery: resources/list

    ResourceDiscovery --> Ready: ✅ 就緒
    Ready --> Invocation: 執行工具呼叫
    Invocation --> Ready: 回傳結果

    Ready --> ErrorSurfacing: 錯誤發生
    ErrorSurfacing --> Ready: 復原成功
    ErrorSurfacing --> Shutdown: 嚴重錯誤

    Ready --> Shutdown: 使用者退出
    Shutdown --> Cleanup: 清理資源
    Cleanup --> [*]

    note right of SpawnConnect
        支援傳輸方式：
        - Stdio（子程序）
        - SSE（HTTP 串流）
        - WebSocket
        - HTTP
        - ManagedProxy
    end note
```

---

## 9. 勾子（Hook）執行流程

```mermaid
sequenceDiagram
    participant RT as ConversationRuntime
    participant HR as HookRunner
    participant Script as 使用者腳本

    Note over RT: 工具呼叫前
    RT->>HR: run_pre_tool_use(name, input)
    HR->>Script: 執行 pre_tool_use 命令
    Script-->>HR: exit code + stdout
    HR-->>RT: HookRunResult<br/>(allow/deny/override)

    alt Hook 返回 deny
        RT->>RT: 跳過工具執行
    else Hook 返回 allow
        RT->>RT: 執行工具
        Note over RT: 工具呼叫後
        RT->>HR: run_post_tool_use(name, input, output)
        HR->>Script: 執行 post_tool_use 命令
        Script-->>HR: 結果
    end
```

---

## 10. Worker 啟動狀態機

```mermaid
stateDiagram-v2
    [*] --> Spawning: 啟動
    Spawning --> TrustRequired: 需要信任驗證
    TrustRequired --> ReadyForPrompt: 信任通過<br/>（AutoAllowlisted / ManualApproval）
    TrustRequired --> Failed: 信任拒絕

    Spawning --> ReadyForPrompt: 自動信任

    ReadyForPrompt --> Running: 接收到提示詞
    Running --> Finished: 完成
    Running --> Failed: 錯誤

    Failed --> Spawning: 重試

    note right of TrustRequired
        信任策略：
        - AutoTrust
        - RequireApproval
        - Deny
    end note
```

---

## 11. Lane 工作流程事件流

```mermaid
flowchart LR
    START["Started"] --> READY["Ready"]
    READY --> RUNNING["Running"]

    RUNNING --> GREEN["Green<br/>（測試通過）"]
    RUNNING --> RED["Red<br/>（測試失敗）"]
    RUNNING --> BLOCKED["Blocked<br/>（等待）"]

    GREEN --> COMMIT["CommitCreated"]
    COMMIT --> PR["PrOpened"]
    PR --> MERGE_READY["MergeReady"]

    MERGE_READY --> SHIP["ShipPrepared"]
    SHIP --> MERGED["ShipMerged"]
    MERGED --> FINISHED["Finished"]

    RED --> RECOVERY["Recovery"]
    RECOVERY --> RUNNING

    BLOCKED --> RUNNING

    RED --> FAILED["Failed"]
    BLOCKED --> FAILED

    style GREEN fill:#c8e6c9
    style RED fill:#ffcdd2
    style FINISHED fill:#e8f5e9
    style FAILED fill:#ffebee
```

---

*本文件基於 claw-code 專案原始碼分析產生 — 2026-04-26*
