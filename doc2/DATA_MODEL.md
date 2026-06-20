# Claw Code 資料模型與關聯圖

Claw Code **不使用傳統資料庫**，所有狀態以記憶體資料結構 + 檔案系統持久化管理。本文件以 Mermaid ER Diagram 呈現所有核心資料模型之間的關係。

---

## 1. 完整資料關聯圖

```mermaid
erDiagram
    Session ||--o{ ConversationMessage : "包含多則訊息"
    Session ||--o| SessionCompaction : "可能有壓縮紀錄"
    Session ||--o| SessionFork : "可能有分支來源"
    Session ||--o{ SessionPromptEntry : "使用者提示歷史"
    Session {
        string session_id PK
        u64 created_at_ms
        u64 updated_at_ms
        string workspace_root
        string model
        u64 last_health_check_ms
    }

    ConversationMessage ||--o{ ContentBlock : "包含多個內容區塊"
    ConversationMessage ||--o| TokenUsage : "可能有 token 用量"
    ConversationMessage {
        MessageRole role
        Vec_ContentBlock blocks
        Option_TokenUsage usage
    }

    ContentBlock {
        string variant "Text | ToolUse | ToolResult"
        string text "Text 時"
        string id "ToolUse 時"
        string name "ToolUse 時"
        string input "ToolUse 時"
        string tool_use_id "ToolResult 時"
        string tool_name "ToolResult 時"
        string output "ToolResult 時"
        bool is_error "ToolResult 時"
    }

    SessionCompaction {
        u32 count
        usize removed_message_count
        string summary
    }

    SessionFork {
        string parent_session_id FK
        string branch_name
    }

    SessionPromptEntry {
        u64 timestamp_ms
        string text
    }

    TokenUsage {
        u32 input_tokens
        u32 output_tokens
        u32 cache_creation_input_tokens
        u32 cache_read_input_tokens
    }

    MessageRole {
        string value "System | User | Assistant | Tool"
    }
```

---

## 2. API 請求/回應資料模型

```mermaid
erDiagram
    MessageRequest ||--o{ InputMessage : "包含訊息"
    MessageRequest ||--o{ ToolDefinition : "攜帶工具定義"
    MessageRequest ||--o| ToolChoice : "工具選擇策略"
    MessageRequest {
        string model
        u32 max_tokens
        bool stream
        Option_string system
        Option_f64 temperature
        Option_f64 top_p
        Option_string reasoning_effort
    }

    InputMessage ||--o{ InputContentBlock : "訊息內容"
    InputMessage {
        string role
    }

    InputContentBlock {
        string variant "Text | ToolUse | ToolResult"
        string text "Text 時"
        string id "ToolUse 時"
        string name "ToolUse 時"
        Value input "ToolUse 時"
        string tool_use_id "ToolResult 時"
        bool is_error "ToolResult 時"
    }

    ToolDefinition {
        string name PK
        Option_string description
        Value input_schema
    }

    ToolChoice {
        string variant "Auto | Any | Tool"
        string name "Tool 時"
    }

    MessageResponse ||--o{ OutputContentBlock : "回應內容"
    MessageResponse ||--|| Usage : "token 用量"
    MessageResponse {
        string id PK
        string kind
        string role
        string model
        Option_string stop_reason
        Option_string request_id
    }

    OutputContentBlock {
        string variant "Text | ToolUse | Thinking | RedactedThinking"
        string text "Text 時"
        string id "ToolUse 時"
        string name "ToolUse 時"
        Value input "ToolUse 時"
        string thinking "Thinking 時"
    }

    Usage {
        u32 input_tokens
        u32 output_tokens
        u32 cache_creation_input_tokens
        u32 cache_read_input_tokens
    }
```

---

## 3. 設定資料模型

```mermaid
erDiagram
    RuntimeConfig ||--o{ ConfigEntry : "合併來源"
    RuntimeConfig ||--|| RuntimeFeatureConfig : "解析結果"
    RuntimeConfig {
        BTreeMap merged
    }

    ConfigEntry {
        ConfigSource source "User | Project | Local"
        PathBuf path
    }

    RuntimeFeatureConfig ||--|| RuntimeHookConfig : "勾子設定"
    RuntimeFeatureConfig ||--|| RuntimePluginConfig : "外掛設定"
    RuntimeFeatureConfig ||--|| McpConfigCollection : "MCP 設定"
    RuntimeFeatureConfig ||--o| OAuthConfig : "OAuth 設定"
    RuntimeFeatureConfig ||--|| RuntimePermissionRuleConfig : "權限規則"
    RuntimeFeatureConfig ||--|| SandboxConfig : "沙箱設定"
    RuntimeFeatureConfig ||--|| ProviderFallbackConfig : "回退設定"
    RuntimeFeatureConfig {
        Option_string model
        BTreeMap aliases
        Option_ResolvedPermissionMode permission_mode
        Vec_string trusted_roots
    }

    RuntimeHookConfig {
        Vec_string pre_tool_use
        Vec_string post_tool_use
        Vec_string post_tool_use_failure
    }

    RuntimePluginConfig {
        BTreeMap enabled_plugins
        Vec_string external_directories
        Option_string install_root
        Option_string registry_path
    }

    McpConfigCollection ||--o{ ScopedMcpServerConfig : "伺服器設定"
    ScopedMcpServerConfig ||--|| McpServerConfig : "傳輸設定"

    McpServerConfig {
        string variant "Stdio | Sse | Http | Ws | Sdk | ManagedProxy"
    }

    SandboxConfig {
        bool enabled
        bool namespace_restrictions
        bool network_isolation
        FilesystemIsolationMode filesystem_mode
        Vec_string allowed_mounts
    }

    ProviderFallbackConfig {
        Option_string primary
        Vec_string fallbacks
    }
```

---

## 4. 權限系統資料模型

```mermaid
erDiagram
    PermissionPolicy ||--|| PermissionMode : "目前模式"
    PermissionPolicy ||--o{ PermissionRequest : "評估請求"

    PermissionMode {
        string value "ReadOnly | WorkspaceWrite | DangerFullAccess | Prompt | Allow"
    }

    PermissionRequest {
        string tool_name
        string input
        PermissionMode current_mode
        PermissionMode required_mode
        Option_string reason
    }

    PermissionRequest ||--|| PermissionOutcome : "評估結果"
    PermissionOutcome {
        string variant "Allow | Deny"
        string reason "Deny 時"
    }

    PermissionContext ||--o| PermissionOverride : "勾子覆蓋"
    PermissionOverride {
        string variant "Allow | Deny | Ask"
    }

    PermissionEnforcer ||--|| PermissionPolicy : "使用"
    PermissionEnforcer {
        string tool_name
        PermissionMode required_permission
    }

    PermissionEnforcer ||--|| EnforcementResult : "結果"
    EnforcementResult {
        bool allowed
        string reason
    }
```

---

## 5. 工具系統資料模型

```mermaid
erDiagram
    ToolRegistry ||--o{ ToolManifestEntry : "工具清單"
    ToolManifestEntry {
        string name PK
        ToolSource source "Base | Conditional"
    }

    GlobalToolRegistry ||--o{ RuntimeToolDefinition : "執行時工具"
    GlobalToolRegistry ||--o{ PluginTool : "外掛工具"
    GlobalToolRegistry ||--|| PermissionEnforcer : "權限執行器"

    ToolSpec {
        string name PK
        string description
        Value input_schema
        PermissionMode required_permission
    }

    BashCommandInput {
        string command
        Option_u64 timeout
        Option_string description
        Option_bool run_in_background
        Option_bool dangerously_disable_sandbox
        Option_FilesystemIsolationMode filesystem_mode
    }

    BashCommandOutput {
        string stdout
        string stderr
        bool interrupted
        Option_string background_task_id
        Option_bool is_image
    }

    ReadFileOutput {
        TextFilePayload file
    }

    TextFilePayload {
        string file_path
        string content
        usize num_lines
        usize start_line
        usize total_lines
    }

    WriteFileOutput {
        string file_path
        string content
        Vec_StructuredPatchHunk structured_patch
    }

    GrepSearchInput {
        string pattern
        Option_string path
        Option_string glob_pattern
        Option_string output_mode
    }
```

---

## 6. MCP 系統資料模型

```mermaid
erDiagram
    McpServerManager ||--o{ McpStdioProcess : "管理程序"
    McpServerManager ||--o{ McpTool : "已發現工具"
    McpServerManager ||--o{ McpResource : "已發現資源"

    McpStdioProcess {
        string server_name PK
        string command
        Vec_string args
    }

    McpTool {
        string name PK
        string description
        Value input_schema
    }

    McpResource {
        string uri PK
        string name
        Option_string description
        Option_string mime_type
    }

    McpToolCallParams {
        string name
        Value arguments
    }

    McpToolCallResult ||--o{ McpToolCallContent : "回傳內容"
    McpToolCallContent {
        string variant "Text | Json | Image | Resource"
    }

    McpLifecycleState ||--|| McpLifecyclePhase : "目前階段"
    McpLifecycleState ||--o{ McpFailedServer : "失敗伺服器"

    McpLifecyclePhase {
        string value "ConfigLoad | ServerRegistration | SpawnConnect | InitializeHandshake | ToolDiscovery | ResourceDiscovery | Ready | Invocation | ErrorSurfacing | Shutdown | Cleanup"
    }

    McpToolRegistry ||--o{ McpTool : "橋接工具"
    McpToolRegistry ||--o{ McpResource : "橋接資源"

    JsonRpcRequest {
        string jsonrpc "2.0"
        JsonRpcId id
        string method
        Option_Value params
    }

    JsonRpcResponse {
        string jsonrpc "2.0"
        JsonRpcId id
        Option_Value result
        Option_JsonRpcError error
    }
```

---

## 7. 工作流程資料模型

```mermaid
erDiagram
    TaskRegistry ||--o{ Task : "管理任務"
    Task ||--o{ TaskMessage : "任務訊息"
    Task {
        string task_id PK
        string prompt
        string description
        Option_TaskPacket task_packet
        TaskStatus status
        u64 created_at
        u64 updated_at
        Option_string team_id FK
    }

    TaskStatus {
        string value "Created | Running | Completed | Failed | Stopped"
    }

    TaskMessage {
        string role
        string content
        u64 timestamp
    }

    TeamRegistry ||--o{ Team : "管理團隊"
    Team ||--o{ Task : "包含任務"
    Team {
        string team_id PK
        string name
        TeamStatus status
        u64 created_at
        u64 updated_at
    }

    TeamStatus {
        string value "Created | Running | Completed | Deleted"
    }

    CronRegistry ||--o{ CronTask : "管理排程"
    CronTask {
        string cron_id PK
        string name
        string schedule
        CronStatus status
        u64 created_at
    }

    WorkerRegistry ||--o{ Worker : "管理 Worker"
    Worker {
        string worker_id PK
        WorkerStatus status
        Option_WorkerFailure failure
        u64 created_at
    }

    WorkerStatus {
        string value "Spawning | TrustRequired | ReadyForPrompt | Running | Finished | Failed"
    }

    TaskPacket {
        string objective
        TaskScope scope
        Option_string scope_path
        Option_string repo
        Option_string worktree
    }

    LaneEvent {
        LaneEventName name
        LaneEventStatus status
        EventProvenance provenance
        u64 timestamp_ms
    }
```

---

## 8. 外掛系統資料模型

```mermaid
erDiagram
    PluginManager ||--o{ PluginMetadata : "管理外掛"
    PluginManager ||--|| PluginManagerConfig : "管理設定"

    PluginMetadata ||--o{ PluginTool : "提供工具"
    PluginMetadata ||--|| PluginHooks : "註冊勾子"
    PluginMetadata ||--o| PluginLifecycle : "生命週期"
    PluginMetadata {
        string id PK
        string name
        string version
        Option_string description
        PluginKind kind
        PathBuf root
        bool default_enabled
    }

    PluginKind {
        string value "Builtin | Bundled | External"
    }

    PluginTool {
        string name PK
        string description
        Value input_schema
        Vec_PluginPermission permissions
    }

    PluginPermission {
        string value "Read | Write | Execute"
    }

    PluginHooks {
        Vec_string pre_tool_use
        Vec_string post_tool_use
        Vec_string post_tool_use_failure
    }

    PluginLifecycle {
        Vec_string init
        Vec_string shutdown
    }

    PluginRegistry ||--o{ PluginMetadata : "儲存"
    PluginState {
        string value "Unconfigured | Validated | Starting | Healthy | Degraded | Failed | ShuttingDown | Stopped"
    }
```

---

## 9. 遙測系統資料模型

```mermaid
erDiagram
    SessionTracer ||--o{ TelemetryEvent : "記錄事件"
    SessionTracer {
        string session_id PK
        Arc_AtomicU64 sequence
    }

    TelemetryEvent {
        string variant "HttpRequestStarted | HttpRequestSucceeded | HttpRequestFailed | Analytics | SessionTrace"
    }

    SessionTraceRecord {
        string session_id FK
        u64 sequence
        string name
        u64 timestamp_ms
        Map attributes
    }

    AnalyticsEvent {
        string namespace
        string action
        Map properties
    }

    ClientIdentity {
        string app_name
        string app_version
        string runtime
    }

    AnthropicRequestProfile {
        string anthropic_version
        ClientIdentity client_identity
        Vec_string betas
        Map extra_body
    }

    ModelPricing {
        f64 input_cost_per_million
        f64 output_cost_per_million
        f64 cache_creation_cost_per_million
        f64 cache_read_cost_per_million
    }

    UsageCostEstimate {
        f64 input_cost_usd
        f64 output_cost_usd
        f64 cache_creation_cost_usd
        f64 cache_read_cost_usd
    }

    UsageTracker ||--o{ TokenUsage : "累計用量"
```

---

## 10. 持久化策略摘要

| 資料 | 格式 | 路徑 | 旋轉策略 |
|------|------|------|----------|
| 工作階段 | JSONL | `.claw/sessions/<id>.jsonl` | 256KB 旋轉，保留 3 份 |
| Worker 狀態 | JSON | `.claw/worker-state.json` | 覆蓋寫入 |
| 設定 | JSON | `.claw.json` / `.claw/settings.json` | 手動管理 |
| 遙測 | JSONL | 遙測路徑 | 追加寫入 |
| OAuth 憑證 | JSON | credentials 路徑 | 覆蓋寫入 |
| 任務/團隊/排程 | 記憶體 | 無持久化 | Session 結束時消失 |
| MCP 連線 | 記憶體 | 無持久化 | Session 結束時關閉 |
| LSP 連線 | 記憶體 | 無持久化 | Session 結束時關閉 |

---

*本文件基於 claw-code 專案原始碼分析產生 — 2026-04-26*
