# 第三部分：系統架構深度解析

在本書前兩部分中，我們已經對 Claw Code 的基本概念、設計哲學以及快速入門流程進行了全面的介紹。讀者至此應當已經能夠在自己的開發環境中成功安裝、設定並運行 Claw Code，體驗其基本的對話式開發功能。然而，要真正理解 Claw Code 的能力邊界、擴充潛力以及在各種複雜場景下的行為模式，僅僅停留在使用者介面層面是遠遠不夠的。我們需要深入系統的內部，探索其架構設計的每一個層面，理解每一個元件的職責邊界、互動模式以及設計決策背後的考量。

本部分將帶領讀者進行一場徹底的架構巡禮，從最高層的 Cargo Workspace 組織結構開始，逐層深入到核心引擎的三十五個以上子模組，再到通訊層的多提供商 API 抽象。通過這趟旅程，讀者將獲得對 Claw Code 系統架構的完整認知，為後續章節中對各個子系統的詳細分析奠定堅實的基礎。

---

## 第七章：分層架構總覽

### 7.1 Cargo Workspace Monorepo 設計

Claw Code 的原始碼組織採用了 Rust 生態系統中最為成熟的大型專案管理方式——Cargo Workspace Monorepo 架構。這意味著整個專案被組織為一個單一的 Git 儲存庫，其中包含多個相互協作但各自獨立的 crate（Rust 的套件單位）。所有這些 crate 共享同一個 Cargo.lock 檔案，確保依賴版本的一致性，同時透過工作區層級的 Cargo.toml 統一管理建構設定、程式碼品質標準以及共用的外部依賴。

這種 Monorepo 架構的選擇並非偶然，而是經過深思熟慮的設計決策。在軟體工程實踐中，Monorepo 與 Polyrepo（多儲存庫）兩種策略各有優劣。Monorepo 的核心優勢在於：所有程式碼都在同一個版本控制歷史中演進，跨 crate 的重構可以在單一提交中完成，依賴版本永遠保持同步，而且 CI/CD 管線只需要維護一套。對於 Claw Code 這種內部高度耦合的系統——CLI 層、核心引擎層、API 通訊層、工具分派層之間有著大量的型別共享和介面依賴——Monorepo 架構能夠極大地降低版本不匹配、介面不一致等常見問題的發生機率。

具體而言，Claw Code 的工作區包含九個 crate，它們分別是：rusty-claude-cli、runtime、api、tools、commands、plugins、telemetry、compat-harness 以及 mock-anthropic-service。這九個 crate 各自承擔明確的職責，形成了一個清晰的分層結構。在整個工作區層級，有兩個極為重要的全域設定：第一是 unsafe_code 被設定為 forbid，這意味著在整個工作區的所有 crate 中，任何使用 unsafe 關鍵字的程式碼都將導致編譯失敗。這是一個極為激進但深思熟慮的安全決策。在一個需要執行不可信 AI 模型指令的系統中，消除記憶體安全漏洞的風險是至關重要的。第二是啟用了 clippy 的 pedantic 等級警告，這是 Rust 靜態分析工具 clippy 中最嚴格的檢查等級。它不僅檢查潛在的錯誤和效能問題，還強制執行慣用的 Rust 程式碼風格，確保整個程式碼庫保持高度一致的品質標準。

工作區的 Cargo.toml 還統一管理了所有 crate 共用的外部依賴版本。例如 tokio（非同步執行時期）、serde 和 serde_json（序列化框架）、reqwest（HTTP 客戶端）等核心依賴，都在工作區層級指定版本，各個 crate 透過 workspace = true 的語法來引用。這種做法確保了所有 crate 使用完全相同版本的外部依賴，避免了依賴衝突和重複編譯的問題。

### 7.2 五層架構模型

Claw Code 的九個 crate 按照職責和依賴關係，自然形成了五個清晰的架構層次。這種分層設計遵循了軟體工程中經典的關注點分離原則——每一層只負責特定的職責領域，層與層之間透過明確定義的介面進行通訊，上層可以依賴下層，但下層絕不能依賴上層。

第一層是使用者介面層，由 rusty-claude-cli 這個 crate 獨自承擔。這個 crate 編譯產出的二進位檔案名為 claw，是使用者與整個系統互動的唯一入口。它負責所有使用者可見的互動體驗，包括三種核心運作模式的實現：互動式 REPL（Read-Eval-Print Loop）模式讓使用者能夠持續地與 AI 模型進行多輪對話；一次性提示模式允許使用者透過命令列參數直接發送單一提示詞並等待回應，適合腳本化和自動化場景；子命令模式則提供了各種管理功能，如 claw init、claw doctor、claw status 等。在終端渲染方面，rusty-claude-cli 使用了 pulldown-cmark 函式庫將 Markdown 格式的 AI 回應轉換為帶有 ANSI 轉義序列的終端輸出，再透過 syntect 函式庫為其中的程式碼區塊添加語法高亮。crossterm 函式庫則提供了跨平台的終端控制能力，使得游標移動、清屏、顏色設定等操作在 Linux、macOS 和 Windows 上都能一致地運作。rustyline 函式庫為 REPL 模式提供了類似 GNU readline 的編輯體驗，包括歷史記錄瀏覽、行內編輯、Tab 補全等功能。此外，這個 crate 還定義了一個名為 ModelProvenance 的資料結構，用於追蹤最終使用的模型名稱是從哪裡決定的——是來自命令列旗標、環境變數、設定檔還是系統預設值。這種來源追蹤機制在除錯和日誌記錄時非常有用。

第二層是協調層，包含 commands 和 compat-harness 兩個 crate。commands crate 負責管理和分派二十四個以上的 REPL 斜線命令。在互動式 REPL 模式中，使用者可以透過以斜線開頭的命令來執行各種管理操作——從顯示幫助資訊和系統狀態，到切換模型和權限模式，再到執行 Git 操作和程式碼審查。每個斜線命令都被註冊為一個獨立的處理器，commands crate 負責解析使用者的輸入、匹配對應的命令以及執行命令邏輯。compat-harness crate 則扮演著一個獨特的角色：它從上游 TypeScript 原始碼中提取工具清單、命令清單以及啟動清單，產生一個名為 ExtractedManifest 的資料結構，供 Rust 端進行功能對等性檢查。這個機制確保了 Claw Code 的 Rust 實作在功能覆蓋度上能夠與上游 TypeScript 版本保持一致。

第三層是核心引擎層，由 runtime 和 tools 兩個 crate 組成。這是整個系統最為龐大和複雜的部分。runtime crate 是系統的「大腦」，包含三十五個以上的子模組，涵蓋了對話迴圈管理、工作階段持久化、設定載入與驗證、權限控制、沙箱隔離、MCP 子系統、工作流程引擎、以及各種輔助功能。tools crate 則是統一的工具分派中樞，定義了四十個內建工具的規格，並負責將 AI 模型的工具呼叫請求路由到正確的執行器。關於核心引擎層的詳細剖析，我們將在第八章中進行深入探討。

第四層是通訊層，由 api crate 獨自承擔。api crate 實現了一個統一的多提供商 AI API 客戶端，能夠與 Anthropic、OpenAI、xAI 以及 DashScope 等多家 AI 服務提供商進行通訊。它透過一個名為 ProviderClient 的列舉型別統一封裝了所有提供商的客戶端實作，並提供了 SSE（Server-Sent Events）串流解析、提示詞快取、HTTP 代理配置等共用功能。關於通訊層的詳細架構，我們將在第九章中進行全面分析。

第五層是支援層，包含 plugins、telemetry 和 mock-anthropic-service 三個 crate。plugins crate 管理外掛的完整生命週期——從發現、載入、初始化到關閉。它定義了三種外掛類型（內建、隨附、外部）和三種外掛權限（讀取、寫入、執行），並實現了工具前後勾子機制，讓外掛能夠在工具執行的前後介入處理流程。telemetry crate 負責事件追蹤與分析，定義了五種遙測事件類型，並提供了記憶體和 JSONL 兩種事件接收器實作。mock-anthropic-service crate 是一個專門用於測試的 Anthropic API 模擬服務，內建了十個測試場景，涵蓋了從基本的串流文字回應到複雜的多工具回合往返等各種使用案例。

### 7.3 Crate 依賴矩陣

理解九個 crate 之間的依賴關係是掌握整個系統架構的關鍵。這些依賴關係形成了一個嚴格的有向無環圖（DAG），確保了系統不存在循環依賴的問題。

在最頂層，rusty-claude-cli 依賴於幾乎所有其他 crate：api、commands、compat-harness、runtime、plugins 以及 tools。這是合理的，因為作為使用者介面層，它需要整合所有下層元件的功能來提供完整的使用體驗。

tools crate 依賴於 api、runtime 和 plugins。它需要 api 來進行網路相關的工具呼叫（如 web_fetch 和 web_search），需要 runtime 來執行核心的檔案操作、Shell 命令和搜尋功能，需要 plugins 來處理外掛提供的動態工具。

commands crate 依賴於 runtime 和 tools。它需要 runtime 來存取工作階段狀態、設定資訊和系統健康狀況，需要 tools 來獲取工具清單等資訊。

api crate 依賴於 runtime 和 telemetry。它需要 runtime 中定義的一些共用型別和設定結構，需要 telemetry 來記錄 HTTP 請求的遙測事件。

runtime crate 依賴於 plugins 和 telemetry。它需要 plugins 來整合外掛系統的勾子機制，需要 telemetry 來進行工作階段追蹤。

compat-harness crate 依賴於 commands、runtime 和 tools。作為功能對等性檢查工具，它需要存取這三個 crate 的元資料來進行比較。

plugins 和 telemetry 兩個 crate 位於依賴鏈的最底部，不依賴工作區中的任何其他 crate。這種設計使得它們能夠被任何其他 crate 自由使用而不會引入循環依賴。mock-anthropic-service 僅在開發時依賴 api crate，用於測試目的。

### 7.4 跨層互動模式

在分層架構中，層與層之間的互動模式決定了系統的靈活性、可測試性以及擴充能力。Claw Code 採用了三種核心的跨層互動模式，每一種都解決了特定的架構需求。

第一種是泛型 Runtime 模式。在 runtime crate 的核心模組 conversation.rs 中，對話迴圈引擎被定義為一個泛型結構 ConversationRuntime，它接受兩個型別參數——C 必須實作 ApiClient trait，T 必須實作 ToolExecutor trait。ApiClient trait 定義了與 AI 提供商通訊的介面，包括發送訊息、串流回應等操作；ToolExecutor trait 定義了工具執行的介面，包括工具呼叫的分派和結果收集。這種泛型設計的最大優勢在於可測試性：在生產環境中，C 被實例化為真實的 ProviderClient，T 被實例化為真實的工具執行器；而在測試環境中，C 可以被替換為 mock-anthropic-service 提供的模擬客戶端，T 可以被替換為預設回應的虛擬工具。這意味著整個對話引擎的核心邏輯——包括多輪對話管理、工具呼叫串接、壓縮觸發等——都可以在不需要真實 API 金鑰和網路連線的情況下進行完整的單元測試和整合測試。

第二種是全域 Registry 模式。Claw Code 使用了六個全域單例 Registry 來管理不同類型的動態資源：TaskRegistry 管理任務的生命週期、TeamRegistry 管理團隊、CronRegistry 管理排程任務、LspRegistry 管理 LSP（Language Server Protocol）連線、McpToolRegistry 管理 MCP 伺服器的工具橋接、WorkerRegistry 管理 Worker 執行個體。這些 Registry 都使用 Rust 標準函式庫中的 OnceLock 機制實現懶初始化的全域單例——第一次存取時創建，之後所有存取都返回同一個實例的參考。OnceLock 保證了初始化的執行緒安全性，而 Registry 內部使用適當的同步原語（如 RwLock 或 Mutex）來保護併發存取。這種全域 Registry 模式的好處是簡化了資源的生命週期管理——Registry 的生命週期與整個程序相同，不需要在函數呼叫鏈中傳遞 Registry 的參考。然而，它的缺點是增加了全域狀態，使得某些測試場景更加複雜。Claw Code 透過在每個 Registry 中提供清除方法來緩解這個問題，允許測試在需要時重置全域狀態。

第三種是設定合併鏈模式。Claw Code 的設定系統支援五個層級的設定來源，優先級從低到高依序為：使用者全域設定（位於家目錄的 .claw.json）、使用者設定目錄中的設定（位於 .config/claw/settings.json）、專案根目錄的設定（.claw.json）、專案設定目錄中的設定（.claw/settings.json）、以及專案本地覆蓋設定（.claw/settings.local.json）。ConfigLoader 的 discover 方法會按照優先級順序載入所有存在的設定檔案，然後進行深度合併。高優先級的設定會覆蓋低優先級的同名欄位，但不會刪除低優先級中存在而高優先級中不存在的欄位。合併完成後，config_validate 模組會對最終的設定進行驗證，產生可能的診斷報告。這種多層設定合併鏈讓使用者能夠在不同的粒度上定制 Claw Code 的行為——全域設定適用於所有專案，專案設定適用於團隊共享，本地覆蓋設定適用於個人偏好和敏感資訊。

---

## 第八章：核心引擎架構——runtime crate 全解

### 8.1 三十五個以上子模組分群

runtime crate 是 Claw Code 中最為龐大和複雜的元件，其三十五個以上的子模組涵蓋了從對話管理到沙箱隔離、從 MCP 子系統到工作流程引擎的幾乎所有核心功能。為了便於理解，我們將這些子模組按照功能領域分為七個群組，每個群組中的模組在功能上緊密相關，共同完成一組特定的職責。

第一個群組是對話核心群組，包含四個模組：conversation.rs、session.rs、compact.rs 以及 prompt.rs。這四個模組共同構成了 Claw Code 的對話引擎——系統最核心的運作邏輯所在。conversation.rs 定義了前述的泛型 ConversationRuntime 結構，它包含 ApiClient trait（定義 AI 提供商通訊介面）、ToolExecutor trait（定義工具執行介面）、TurnSummary（對話回合摘要）以及 AssistantEvent（助手事件，包括 TextDelta、ToolUse、MessageStop 等變體）。ConversationRuntime 的核心迴圈可以概括為：接收使用者訊息、建構 API 請求（包含系統提示詞、歷史訊息和工具定義）、透過 SSE 串流接收 AI 回應、處理回應中的事件（文字增量即時渲染、工具呼叫觸發權限檢查和執行）、將工具結果回饋給 API 繼續對話、直到收到 message_stop 事件。session.rs 負責工作階段的持久化管理，它定義了 Session（工作階段）、ConversationMessage（對話訊息）、ContentBlock（內容區塊，有 Text、ToolUse、ToolResult 三種變體）、MessageRole（訊息角色，有 System、User、Assistant、Tool 四種）以及 SessionCompaction（工作階段壓縮記錄）等核心型別。工作階段使用 JSONL（JSON Lines）格式進行持久化存儲，每則訊息作為一行 JSON 追加寫入。compact.rs 處理工作階段壓縮，當對話的 token 數量超過預設閾值（十萬個 token）時，自動觸發壓縮流程，保留最近四則訊息和壓縮摘要。prompt.rs 則負責系統提示詞的組裝，它定義了 SystemPromptBuilder、ProjectContext 和 ContextFile 等型別，能夠將 Git 狀態、專案設定、使用者指令檔等上下文資訊整合進系統提示詞中。

第二個群組是安全子系統群組，包含五個模組：permissions.rs、permission_enforcer.rs、sandbox.rs、bash_validation.rs 以及 trust_resolver.rs。這五個模組共同構成了 Claw Code 的多層安全防禦體系。permissions.rs 定義了五種權限模式——ReadOnly（唯讀）、WorkspaceWrite（工作區寫入）、DangerFullAccess（完全存取）、Prompt（每次詢問）和 Allow（自動允許），以及 PermissionPolicy（權限策略）和 PermissionOutcome（權限結果）等型別。permission_enforcer.rs 實現了工具級的權限強制執行，每個工具都有一個 required_permission 屬性指定其所需的最低權限級別，PermissionEnforcer 會將工具的需求與當前的權限模式進行比較。sandbox.rs 負責 Linux namespace 沙箱隔離，包括 PID 命名空間（程序隔離）、NET 命名空間（網路隔離）和檔案系統隔離三種機制，以及容器環境偵測功能。bash_validation.rs 對 AI 模型請求執行的每一條 Bash 命令進行語意分析，將其分類為八種意圖類型：ReadOnly、Write、Destructive、Network、ProcessManagement、PackageManagement、SystemAdmin 和 Unknown。trust_resolver.rs 管理 Worker 的信任閘門，定義了三種信任策略：AutoTrust（自動信任）、RequireApproval（需要人工核准）和 Deny（拒絕）。

第三個群組是工具執行群組，包含三個模組：bash.rs、file_ops.rs 以及 git_context.rs。bash.rs 負責在沙箱環境中安全地執行 Shell 命令，它定義了 BashCommandInput（包含命令字串、逾時設定、描述、背景執行旗標等）和 BashCommandOutput（包含標準輸出、標準錯誤、中斷狀態、背景任務 ID 等）。file_ops.rs 是一個功能密集的模組，實現了五種檔案操作：讀取檔案（ReadFile）、寫入檔案（WriteFile）、編輯檔案（EditFile）、glob 模式搜尋（GlobSearch）和正規表達式搜尋（GrepSearch）。每種操作都內建了安全檢查，包括十 MB 的大小限制、二進位檔案偵測（透過 NUL 位元組檢查）、工作區邊界驗證（防止路徑逃逸）和符號連結追蹤（防止 symlink 攻擊）。git_context.rs 負責偵測和收集 Git 相關的上下文資訊，包括當前分支名稱、最近的提交歷史和暫存區狀態。

第四個群組是 MCP（Model Context Protocol）子系統群組，包含六個模組：mcp.rs、mcp_client.rs、mcp_stdio.rs、mcp_server.rs、mcp_tool_bridge.rs 以及 mcp_lifecycle_hardened.rs。MCP 是 Claw Code 中用於與外部工具伺服器通訊的協定子系統，讓 AI 模型能夠存取外部服務提供的工具和資源。mcp.rs 提供 MCP 工具命名的規範化函式，定義了雙下劃線分隔的命名格式。mcp_client.rs 抽象了六種傳輸方式：Stdio（透過子程序的 stdin/stdout 通訊）、SSE（HTTP Server-Sent Events 串流）、HTTP（標準 HTTP 請求/回應）、WebSocket（雙向即時通訊）、SDK（保留）和 ManagedProxy（託管代理轉發）。mcp_stdio.rs 實現了最常用的 Stdio 傳輸，包括子程序管理和 JSON-RPC 2.0 通訊。mcp_server.rs 實現了本地 MCP 伺服器，遵循 Protocol v2024-11-05 版本規格。mcp_tool_bridge.rs 是 MCP 伺服器到內部工具系統的橋接層，維護了 McpToolRegistry。mcp_lifecycle_hardened.rs 定義了一個精密的十一階段生命週期狀態機，涵蓋從 ConfigLoad 到 Cleanup 的完整生命週期。

第五個群組是設定與狀態群組，包含六個模組：config.rs、config_validate.rs、usage.rs、hooks.rs、oauth.rs 以及 session_control.rs。config.rs 實現了前述的五層設定合併鏈，定義了 RuntimeConfig、ConfigLoader、RuntimeFeatureConfig、McpServerConfig 等核心設定型別。config_validate.rs 負責對合併後的設定進行驗證，產生 ConfigDiagnostic 診斷報告。usage.rs 追蹤 token 用量和成本估算，定義了 ModelPricing（模型定價表）、TokenUsage（token 用量）、UsageTracker（用量追蹤器）和 UsageCostEstimate（成本估算）等型別。hooks.rs 實現了工具前後勾子機制，定義了 HookRunner、HookEvent（PreToolUse、PostToolUse、PostToolUseFailure 三種事件）和 HookAbortSignal 等型別，允許外部腳本在工具執行的前後介入處理流程並可能覆蓋權限決策。oauth.rs 實現了 PKCE S256 OAuth 認證流程。session_control.rs 管理工作階段的存儲和工作區指紋機制。

第六個群組是工作流程引擎群組，包含十一個模組：lane_events.rs、worker_boot.rs、policy_engine.rs、recovery_recipes.rs、task_registry.rs、team_cron_registry.rs、stale_branch.rs、stale_base.rs、branch_lock.rs、green_contract.rs 以及 task_packet.rs。這個群組實現了 Claw Code 的 CI/CD 工作流程自動化能力。lane_events.rs 定義了二十個以上的 Lane 事件類型和十二種失敗分類。worker_boot.rs 管理 Worker 的生命週期狀態機（Spawning → TrustRequired → ReadyForPrompt → Running → Finished/Failed）。policy_engine.rs 實現了 Lane 策略評估引擎，根據 PolicyRule、PolicyCondition 和 PolicyAction 來決定工作流程的行為。recovery_recipes.rs 定義了七種失敗場景的自動化復原方案。task_registry.rs 是記憶體中的任務生命週期管理器。team_cron_registry.rs 管理團隊和排程任務。stale_branch.rs 偵測分支的新鮮度並根據策略採取行動。stale_base.rs 追蹤基底提交的新鮮度。branch_lock.rs 偵測分支鎖定衝突。green_contract.rs 定義了四級測試品質閘門。task_packet.rs 定義了任務封包的結構和驗證邏輯。

第七個群組包含其餘的輔助模組：remote.rs（遠端會話和上游代理）、sse.rs（增量 SSE 幀解析器）、lsp_client.rs（LSP 客戶端整合，支援七種動作：diagnostics、hover、definition、references、completion、symbols 和 format）、plugin_lifecycle.rs（外掛健康狀態追蹤，定義了八種狀態）、summary_compression.rs（摘要文字壓縮，上限一千二百個字元或二十四行）、bootstrap.rs（啟動計畫管理）以及 json.rs（JSON 工具輔助函式）。

### 8.2 ConversationRuntime 泛型設計

ConversationRuntime 的泛型設計是 Claw Code 架構中最為精妙的部分之一，值得我們進行更加深入的分析。在傳統的軟體設計中，核心業務邏輯往往與具體的基礎設施實作緊密耦合——對話引擎直接呼叫特定提供商的 API 客戶端，工具執行器直接存取檔案系統和程序管理。這種緊耦合使得程式碼難以測試、難以擴充、也難以在不同的環境中復用。

Claw Code 透過 Rust 的 trait 系統和泛型機制優雅地解決了這個問題。ConversationRuntime 被定義為 ConversationRuntime<C: ApiClient, T: ToolExecutor>，其中 C 和 T 是型別參數，分別要求實作 ApiClient 和 ToolExecutor 兩個 trait。ApiClient trait 定義了與 AI 提供商通訊的所有必要介面方法，包括建立串流連線、發送訊息請求等；ToolExecutor trait 定義了工具執行的所有必要介面方法，包括工具分派、結果收集和錯誤處理。

在生產環境中，ConversationRuntime 被實例化為 ConversationRuntime<ProviderClient, ToolDispatcher>，其中 ProviderClient 是真實的多提供商 API 客戶端，ToolDispatcher 是真實的四十個工具的分派器。而在測試環境中，它可以被實例化為 ConversationRuntime<MockAnthropicClient, MockToolExecutor>，其中 MockAnthropicClient 預先定義了特定的 SSE 事件序列，MockToolExecutor 預先定義了特定的工具呼叫回應。這種替換是在編譯時期靜態決定的（Rust 的泛型是單態化的），沒有任何執行時期的效能開銷。

ConversationRuntime 的核心對話迴圈遵循一個精確的事件驅動模型。迴圈的每一個回合（turn）包含以下步驟：首先，接收使用者的輸入訊息，將其封裝為 ConversationMessage 並添加到工作階段的訊息歷史中。然後，呼叫 build_api_request 方法，將系統提示詞、完整的訊息歷史以及所有可用工具的定義組合成一個 API 請求。接著，透過 ApiClient 的串流介面發送請求，並開始接收 SSE 事件。在 SSE 事件迴圈中，每當收到 TextDelta 事件時，增量的文字內容會被即時傳遞到 CLI 層進行終端渲染，讓使用者能夠看到 AI 的回應正在生成。當收到 ToolUse 事件時，系統首先透過 HookRunner 執行工具前勾子（pre_tool_use），然後透過 PermissionPolicy 進行權限檢查。如果權限允許，則透過 ToolExecutor 執行工具呼叫，收集結果，再透過 HookRunner 執行工具後勾子（post_tool_use）。工具結果被封裝為 ToolResult 並添加到訊息歷史中，然後系統會自動發起新一輪的 API 請求，將工具結果包含在內。這個過程會持續進行，直到 AI 模型決定不再呼叫工具（收到 message_stop 事件且 stop_reason 為 end_turn）。在每個回合結束後，系統會更新 UsageTracker 記錄 token 用量，檢查是否需要觸發自動壓縮（token 總量超過十萬的閾值），並將工作階段存儲到磁碟。

---

## 第九章：通訊層與 API 架構

### 9.1 ProviderClient 多提供商統一

Claw Code 的通訊層面臨一個核心挑戰：如何在支援多家 AI 提供商的同時，保持上層程式碼的簡潔性和一致性。不同的 AI 提供商使用不同的 API 端點、不同的請求格式、不同的認證機制、甚至不同的 SSE 事件結構。如果為每個提供商編寫專門的處理邏輯，上層的對話引擎將充滿條件分支，維護成本會急劇上升。

api crate 透過一個名為 ProviderClient 的列舉型別優雅地解決了這個問題。ProviderClient 有三個變體：Anthropic（包裝 AnthropicClient）、Xai（包裝 OpenAiCompatClient）和 OpenAi（也包裝 OpenAiCompatClient）。AnthropicClient 是專門為 Anthropic Messages API 設計的客戶端，使用 x-api-key 標頭進行認證，處理 Anthropic 特有的 SSE 事件格式。OpenAiCompatClient 則是一個通用的 OpenAI Chat Completions API 相容客戶端，透過可配置的基礎 URL（base_url）來支援多個不同的提供商——xAI 使用 api.x.ai 作為端點，DashScope 使用 dashscope.aliyuncs.com 作為端點，Ollama 使用 localhost:11434 作為端點，OpenRouter 使用 openrouter.ai 作為端點。這種設計的精妙之處在於，OpenAI Chat Completions API 已經成為事實上的行業標準，許多提供商都選擇相容這個 API 格式，因此一個通用的相容客戶端就能覆蓋大部分提供商。

提供商的選擇是透過模型名稱的前綴自動路由的。detect_provider_kind 函數接收模型名稱作為參數，根據前綴判斷應該使用哪個提供商：以 claude- 開頭的模型名稱路由到 Anthropic，以 grok- 開頭的模型名稱路由到 xAI，以 qwen- 開頭的模型名稱路由到 DashScope，以 gpt- 開頭的模型名稱路由到 OpenAI。如果模型名稱不匹配任何已知前綴，系統會檢查環境變數來決定——如果設定了 ANTHROPIC_API_KEY 或 ANTHROPIC_AUTH_TOKEN 就使用 Anthropic，如果設定了 OPENAI_API_KEY 就使用 OpenAI，如果設定了 XAI_API_KEY 就使用 xAI，如果設定了 DASHSCOPE_API_KEY 就使用 DashScope。如果都沒有設定，則預設使用 Anthropic。

此外，api crate 還支援模型別名機制，讓使用者可以用簡短的名稱指定模型。例如，opus 解析為 claude-opus-4-6，sonnet 解析為 claude-sonnet-4-6，haiku 解析為 claude-haiku-4-5-20251213，grok 解析為 grok-3，grok-mini 解析為 grok-3-mini。resolve_model_alias 函數在 detect_provider_kind 之前被呼叫，確保別名能夠正確地被路由到對應的提供商。

### 9.2 SSE 串流解析

SSE（Server-Sent Events）是 AI API 最常用的串流回應機制。與傳統的請求-回應模式不同，SSE 允許伺服器在單一 HTTP 連線上持續地推送事件到客戶端，讓使用者能夠看到 AI 回應的逐步生成過程。然而，不同提供商的 SSE 事件格式存在顯著差異，api crate 需要能夠正確地解析兩種主要格式。

Anthropic 使用自定義的 SSE 事件序列，一次完整的回應包含以下事件：首先是 message_start 事件，攜帶訊息的元資料（ID、模型、角色等）；接著是一個或多個 content_block_start 事件，每個標記一個新的內容區塊的開始；在每個內容區塊內，會有多個 content_block_delta 事件，攜帶增量的內容更新。content_block_delta 有四種變體：TextDelta 攜帶增量的文字內容，InputJsonDelta 攜帶增量的 JSON 資料（用於工具呼叫的輸入參數），ThinkingDelta 攜帶增量的思考過程（extended thinking 功能），SignatureDelta 攜帶思考簽章。每個內容區塊結束時會有一個 content_block_stop 事件。最後是 message_delta 事件（攜帶停止原因和最終的 token 用量）和 message_stop 事件（標記整個回應的結束）。

OpenAI Chat Completions API 使用的 SSE 格式相對簡單。每個事件的 data 欄位是一個 JSON 物件，包含 choices 陣列，每個 choice 有一個 delta 物件攜帶增量的 content 或 tool_calls。api crate 的 SseParser 模組能夠根據提供商類型自動選擇正確的解析邏輯。

特別值得一提的是增量 JSON 組裝機制。當 AI 模型決定呼叫一個工具時，它需要提供工具呼叫的輸入參數，這些參數是以 JSON 格式表示的。然而，在 SSE 串流中，這個 JSON 不是一次性發送的，而是被分割成多個 InputJsonDelta 事件逐步傳送。api crate 需要在收集到所有增量的 JSON 片段後，將它們正確地組裝成完整的 JSON 物件。這個過程需要處理各種邊緣情況，包括 JSON 字串中的轉義字元、嵌套的物件和陣列結構、以及片段之間的正確拼接。

所有這些提供商特定的 SSE 解析邏輯都被封裝在 api crate 內部，對上層的 ConversationRuntime 來說，它只看到統一的 StreamEvent 列舉——MessageStart、MessageDelta、ContentBlockStart、ContentBlockDelta、ContentBlockStop 和 MessageStop。這種抽象確保了對話引擎的核心邏輯不受提供商差異的影響。

### 9.3 認證與 HTTP 代理

api crate 支援多種認證方式，適應不同提供商和不同使用場景的需求。對於 Anthropic，支援兩種認證方式：API Key 認證（透過 x-api-key HTTP 標頭傳遞以 sk-ant- 開頭的金鑰）和 OAuth Bearer Token 認證（透過 Authorization: Bearer 標頭傳遞 OAuth token）。對於 OpenAI 相容的提供商（包括 xAI、DashScope、Ollama 等），統一使用 Authorization: Bearer 標頭傳遞 API 金鑰。api crate 還實現了完整的 PKCE S256 OAuth 認證流程，支援從瀏覽器授權回調中獲取 token，並自動管理 token 的刷新。

在企業環境中，HTTP 代理配置是不可或缺的功能。api crate 定義了 ProxyConfig 結構，支援四種代理設定：統一代理 URL（proxy_url）、HTTP 代理（http_proxy）、HTTPS 代理（https_proxy）以及代理排除清單（no_proxy）。這些設定既可以透過設定檔指定，也可以透過環境變數（HTTP_PROXY、HTTPS_PROXY、NO_PROXY，大小寫均支援）自動偵測。build_http_client 函數在建構 reqwest HTTP 客戶端時，會自動應用代理配置，確保所有的 API 呼叫都能正確地透過企業代理進行。

### 9.4 提示詞快取

提示詞快取是 Anthropic 提供的一項獨特功能（利用 prompt-caching-scope-2026-01-05 beta 功能），能夠顯著減少重複系統提示詞的 token 消耗。在 Claw Code 的典型使用場景中，系統提示詞（包含專案上下文、工具定義、指令等）通常佔據了每次 API 請求的大部分 token 量，而這些提示詞在同一個工作階段中幾乎不會改變。如果每次請求都重新傳送完整的系統提示詞，不僅浪費了大量的 token（進而增加成本），還增加了不必要的延遲。

api crate 的 PromptCache 模組實現了與 Anthropic 提示詞快取 API 的整合。當系統發出第一次 API 請求時，系統提示詞會被標記為可快取的內容，Anthropic 的伺服器會將其儲存在快取中。在後續的請求中，如果系統提示詞沒有改變，快取的內容可以被直接重用，只需要支付快取讀取的 token 成本（遠低於正常的輸入 token 成本）。以 Claude Opus 模型為例，快取讀取的成本是每百萬 token 一點五美元，而正常輸入的成本是每百萬 token 十五美元，節省了百分之九十的成本。PromptCacheStats 結構追蹤快取的命中次數、未命中次數以及累計節省的 token 數量，讓使用者能夠了解快取的效益。

### 9.5 錯誤處理策略

api crate 定義了一個全面的錯誤型別 ApiError，涵蓋了 API 通訊中可能遇到的所有錯誤場景：Http 變體包裝了底層 reqwest 函式庫的 HTTP 錯誤；Json 變體處理 JSON 序列化和反序列化的錯誤；Authentication 變體表示認證失敗（如 API 金鑰無效或 OAuth token 過期）；RateLimit 變體表示速率限制被觸發（通常包含重試等待時間的資訊）；ServerError 變體表示 AI 提供商的伺服器端錯誤（如內部錯誤或服務暫時不可用）；InvalidResponse 變體表示收到了格式不正確的回應；Configuration 變體表示設定相關的錯誤（如缺少必要的認證資訊）。

Claw Code 的上層對話引擎會根據不同的錯誤類型採取不同的處理策略。對於速率限制錯誤，系統會自動等待指定的時間後重試。對於認證失敗，系統會立即終止並提示使用者檢查 API 金鑰設定。對於伺服器錯誤，系統會進行有限次數的指數退避重試。對於其他錯誤，系統會將錯誤資訊完整地呈現給使用者，讓使用者能夠根據錯誤訊息進行排除。

---

本部分到此完成了對 Claw Code 系統架構的全面剖析。我們從最高層的 Cargo Workspace 組織結構開始，深入到九個 crate 的分層設計、依賴關係和互動模式，然後詳細分析了核心引擎 runtime crate 中三十五個以上子模組的功能分群，最後全面探討了通訊層的多提供商統一架構、SSE 串流解析、認證機制以及提示詞快取。

在接下來的第四部分中，我們將把焦點集中在核心運行時期的對話引擎上，深入分析 ConversationRuntime 的每一個執行步驟、工作階段管理的完整生命週期，以及設定系統的合併與驗證機制。


### 7.5 建構系統與 CI/CD 架構

Claw Code 的建構系統充分利用了 Cargo Workspace 提供的統一建構能力。在工作區根目錄執行 cargo build --workspace 命令會同時建構所有九個 crate，Cargo 會自動處理 crate 之間的依賴順序——先建構沒有內部依賴的 plugins 和 telemetry，然後建構依賴它們的 runtime，接著建構依賴 runtime 的 api 和 tools，最後建構最頂層的 rusty-claude-cli。

建構過程中，rusty-claude-cli 的 build.rs 建構腳本會在編譯時期注入版本元資料。這些元資料包括：Git 提交雜湊值（讓使用者能夠精確地知道二進位檔對應的原始碼版本）、建構時間戳記（讓使用者知道二進位檔的建構時間）、Rust 編譯器版本（用於除錯依賴問題）。這些元資料在 claw --version 命令的輸出中可見，在 /status 斜線命令的輸出中也可見。

Cargo 的增量編譯（incremental compilation）功能在日常開發中非常有用。當開發者只修改了一個 crate 的程式碼時，Cargo 只需要重新編譯受影響的 crate 及其依賴者，而不需要從頭建構整個工作區。由於依賴圖是有向無環的，這種增量編譯通常能夠顯著減少重新建構的時間。

GitHub Actions CI 管線定義了三個主要的驗證步驟。首先是格式檢查：cargo fmt --check 驗證所有原始碼檔案是否符合 rustfmt 的格式標準。其次是靜態分析：cargo clippy --workspace --all-targets -- -D warnings 使用最嚴格的 clippy 等級檢查所有程式碼，並將所有警告視為錯誤。最後是測試：cargo test --workspace 執行工作區中所有 crate 的完整測試套件。只有三個步驟都成功通過，程式碼變更才能被合併到主分支。

容器化建構是另一個重要的部署選項。專案根目錄的 Containerfile 定義了一個多階段建構流程：第一個階段使用 Rust 官方的建構映像進行編譯，產生靜態連結的二進位檔；第二個階段使用精簡的基礎映像，只包含最終的二進位檔和必要的執行時依賴。這種多階段建構產生的容器映像非常小（通常只有幾十 MB），適合在 CI/CD 管線和雲端環境中使用。

### 8.3 對話核心的詳細工作機制

讓我們更深入地分析對話核心四個模組之間的協作方式。當一個新的對話回合開始時，ConversationRuntime 首先呼叫 session.rs 提供的介面，將使用者的輸入封裝為一則 ConversationMessage 並添加到工作階段的訊息歷史中。ConversationMessage 的 role 欄位設定為 User，content 欄位包含一個 Text 變體的 ContentBlock。

接下來，ConversationRuntime 呼叫 prompt.rs 的 SystemPromptBuilder 來組裝系統提示詞。SystemPromptBuilder 需要收集多種上下文資訊：ProjectContext 包含專案目錄結構和技術偵測結果；GitContext 包含當前分支和最近提交；ContextFile 列表包含所有需要注入到系統提示詞中的檔案內容。SystemPromptBuilder 將這些資訊按照特定的模板格式拼接成完整的系統提示詞字串。

然後，ConversationRuntime 建構 MessageRequest。這個請求包含：系統提示詞、工作階段中的完整訊息歷史、所有可用工具的定義清單、模型名稱、最大輸出 token 數量（通常設定為一個足夠大的值以避免截斷）、串流設定（stream: true）以及可選的溫度和其他生成參數。

API 請求被發送到提供商的伺服器後，ConversationRuntime 進入 SSE 事件接收迴圈。在這個迴圈中，它持續地從串流中讀取事件，根據事件類型進行不同的處理。當迴圈結束時（收到 message_stop 事件），ConversationRuntime 會計算本次回合的 token 用量，更新 UsageTracker。

最後，ConversationRuntime 檢查是否需要觸發壓縮。如果工作階段的 token 總量超過了 compact.rs 中定義的閾值，compact_session 方法會被呼叫，移除較舊的訊息並產生摘要。壓縮完成後（或者不需要壓縮時），session.rs 的 save 方法會將更新後的工作階段追加寫入到 JSONL 檔案中。

### 8.4 安全子系統的互動流程

安全子系統的五個模組形成了一個多層過濾器。當一個工具呼叫請求到達時，它首先經過 permissions.rs 的策略評估。PermissionPolicy 會根據以下順序進行檢查：檢查工具是否在 deny 清單中（如果是，立即拒絕）；檢查工具是否在 allow 清單中（如果是，立即允許）；檢查工具是否在 ask 清單中（如果是，顯示使用者確認提示）；最後，根據工具的 required_permission 和當前的 PermissionMode 進行比較。

如果策略評估的結果是允許，請求會被傳遞到 permission_enforcer.rs 進行工具級的強制執行。PermissionEnforcer 會再次驗證工具的 required_permission 是否被當前模式滿足（這是一個雙重檢查，增加了安全性）。

對於 bash 工具的呼叫，還會經過 bash_validation.rs 的額外驗證。classify_command_intent 函數會分析命令的語意意圖，根據當前的權限模式決定是否允許。即使在 DangerFullAccess 模式下，Destructive 意圖的命令仍然會觸發警告。

最後，如果命令需要在沙箱中執行，sandbox.rs 的 build_linux_sandbox_command 函數會將命令包裝在 unshare 命令中，應用 PID、NET 和 FS 隔離。沙箱的配置（是否啟用網路隔離、檔案系統隔離模式等）可以在全域設定中定義，也可以在單個 bash 工具呼叫中透過參數覆蓋。

### 8.5 MCP 子系統的詳細工作流

MCP 子系統的六個模組協同工作，實現了從設定讀取到工具呼叫的完整流程。

啟動階段：config.rs 讀取設定中的 mcpServers 區段，為每個伺服器建立 McpServerConfig。mcp_lifecycle_hardened.rs 的狀態機從 ConfigLoad 階段開始驅動整個啟動流程。

連線階段：mcp_client.rs 根據 McpClientTransport 的類型選擇連線方式。對於 Stdio 傳輸，mcp_stdio.rs 的 spawn_mcp_stdio_process 函數啟動子程序。狀態機進入 SpawnConnect 階段。

握手階段：系統透過選定的傳輸方式發送 initialize JSON-RPC 請求，包含客戶端的能力聲明（支援的協定版本、支援的功能等）。伺服器回覆自己的能力聲明。狀態機進入 InitializeHandshake 階段。

發現階段：系統發送 tools/list 請求，收集伺服器提供的所有工具定義。然後發送 resources/list 請求，收集伺服器提供的所有資源定義。mcp_tool_bridge.rs 的 McpToolRegistry 將這些工具和資源註冊到全域的工具清單中。狀態機依序經過 ToolDiscovery 和 ResourceDiscovery 階段。

就緒階段：所有步驟成功完成後，狀態機進入 Ready 階段。此時，MCP 伺服器的工具可以像內建工具一樣被 AI 模型呼叫。

呼叫階段：當 AI 模型呼叫一個 MCP 工具時，tools crate 的 execute_tool 函數識別出 mcp__ 前綴，從 McpToolRegistry 中找到對應的伺服器連線，透過 mcp_stdio.rs（或其他傳輸模組）發送 tools/call JSON-RPC 請求，等待回應，解析結果。狀態機在 Ready 和 Invocation 之間切換。

### 9.5 請求與回應的資料流

讓我們追蹤一次完整的 API 請求-回應的資料流，理解 api crate 內部各模組的協作方式。

建構階段：ConversationRuntime 建構 MessageRequest 結構，包含模型名稱、系統提示詞、訊息歷史、工具定義和生成參數。

路由階段：ProviderClient 的 stream 方法接收 MessageRequest。根據內部的提供商類型（Anthropic、Xai 或 OpenAi），請求會被路由到對應的客戶端實作。

轉換階段：對於 Anthropic，MessageRequest 會被轉換為 Anthropic Messages API 的原生格式；對於 OpenAI 相容的提供商，MessageRequest 會被轉換為 OpenAI Chat Completions API 的格式。這個轉換涉及角色名稱的映射、內容格式的調整以及工具定義結構的轉換。

發送階段：http_client.rs 的 build_http_client 函數建構的 reqwest 客戶端被用於發送 HTTP POST 請求。請求包含適當的認證標頭（x-api-key 或 Authorization: Bearer）和內容類型標頭。如果配置了 HTTP 代理，代理設定會被自動應用。如果啟用了提示詞快取（Anthropic 專屬），快取標記會被添加到請求中。

接收階段：HTTP 回應以 SSE 格式到達。sse.rs 的 SseParser 模組根據提供商類型選擇正確的解析邏輯。對於 Anthropic，解析器處理 message_start、content_block_start、content_block_delta、content_block_stop、message_delta 和 message_stop 六種事件。對於 OpenAI 相容格式，解析器處理 data: 行，解析 JSON 中的 choices 陣列。

轉換回階段：提供商特定的事件被轉換為統一的 StreamEvent 列舉，回傳給 ConversationRuntime。ToolUse 事件中的 InputJsonDelta 片段會被自動組裝成完整的 JSON 物件。Usage 資訊（token 用量）會從 message_delta 事件中提取。

錯誤處理階段：如果在任何步驟中發生錯誤，error.rs 的 ApiError 列舉會捕獲並分類錯誤。速率限制錯誤會觸發自動重試（帶有指數退避），認證錯誤會立即終止，其他錯誤會根據嚴重程度進行處理。


### 9.6 MessageRequest 與 MessageResponse 的完整結構

深入理解 API 通訊層的資料結構對於掌握 Claw Code 的運作機制至關重要。讓我們完整地解析 MessageRequest 和 MessageResponse 的每一個欄位。

MessageRequest 是 Claw Code 發送給 AI 提供商的請求結構。model 欄位指定要使用的模型名稱，經過別名解析後是完整的模型識別符（如 claude-opus-4-6）。max_tokens 欄位指定單次回應的最大輸出 token 數量，這個值通常設定得足夠大以避免回應被截斷。messages 欄位是一個有序的 InputMessage 列表，代表完整的對話歷史。system 欄位是可選的系統提示詞字串，它被放在對話歷史之外，作為 AI 模型的全域指導。

tools 欄位是一個可選的 ToolDefinition 列表。每個 ToolDefinition 包含工具名稱（name）、可選的工具描述（description）和輸入參數的 JSON Schema（input_schema）。JSON Schema 精確地定義了工具接受什麼格式的輸入——哪些欄位是必填的、哪些是選填的、每個欄位的型別和約束是什麼。AI 模型會根據 JSON Schema 來產生格式正確的工具輸入。

tool_choice 欄位控制 AI 模型使用工具的策略。Auto 讓模型自行決定是否使用工具；Any 要求模型必須使用某個工具（但可以選擇任何一個）；Tool 指定模型必須使用某個特定的工具。在大多數情況下，Claw Code 使用 Auto 策略，讓 AI 模型根據任務需求自主決定。

stream 欄位控制是否啟用 SSE 串流。Claw Code 幾乎總是啟用串流，以提供即時的回應渲染體驗。temperature 和 top_p 欄位控制生成的隨機性——較低的值產生更確定性的輸出，較高的值產生更多樣化的輸出。frequency_penalty 和 presence_penalty 控制重複內容的懲罰程度。stop 欄位是可選的停止序列列表——當模型生成了其中任何一個序列時，生成會立即停止。reasoning_effort 欄位控制模型的推理深度，接受 low、medium 或 high 三個值。

MessageResponse 是 AI 提供商回傳的回應結構。id 是回應的唯一標識符。kind 固定為 message。role 固定為 assistant。content 是 OutputContentBlock 的列表，包含回應的所有內容。model 是實際使用的模型名稱（可能與請求中指定的不同，例如提供商可能使用了更新的模型版本）。stop_reason 指示回應結束的原因——end_turn 表示模型主動結束回應，tool_use 表示模型需要等待工具結果後繼續。usage 包含本次回應的 token 用量統計。request_id 是提供商分配的請求標識符，用於除錯和追蹤。

OutputContentBlock 有四種變體。Text 包含純文字回應。ToolUse 包含工具呼叫請求（id、name 和 input）。Thinking 包含模型的思考過程（thinking 文字和可選的 signature）。RedactedThinking 包含被遮蔽的思考內容（某些敏感的推理過程可能被提供商遮蔽）。

InputMessage 結構在請求中代表一則對話訊息。role 欄位可以是 user 或 assistant。content 欄位是 InputContentBlock 的列表。InputContentBlock 同樣有三種變體：Text 包含純文字；ToolUse 包含之前的工具呼叫記錄；ToolResult 包含工具呼叫的結果（tool_use_id 將結果與對應的呼叫關聯起來，content 包含結果的內容，is_error 標記結果是否為錯誤）。

### 9.7 多提供商格式轉換的細節

api crate 的一個核心職責是在 Claw Code 的統一格式和各提供商的原生格式之間進行轉換。這個轉換過程涉及多個層面的映射。

對於 Anthropic Messages API，格式轉換相對直接，因為 Claw Code 的內部格式很大程度上參考了 Anthropic 的 API 設計。system 提示詞直接映射為 Anthropic API 的 system 參數。messages 列表中的每個 InputMessage 直接映射為 Anthropic 的 messages 陣列元素。ToolDefinition 直接映射為 Anthropic 的 tools 參數。

對於 OpenAI Chat Completions API，格式轉換需要處理幾個關鍵差異。首先，OpenAI 的系統提示詞是作為 messages 陣列中的第一個元素（role 為 system），而不是獨立的 system 參數。其次，OpenAI 的工具定義使用不同的結構——工具被包裝在一個 functions 或 tools 物件中，input_schema 對應 parameters。最後，OpenAI 的工具呼叫格式（function_call 或 tool_calls）與 Anthropic 的 content 區塊格式不同。

api crate 的 SseParser 需要處理兩種截然不同的 SSE 事件結構。Anthropic 的 SSE 使用自訂的事件類型（message_start、content_block_delta 等），每個事件有特定的 JSON 結構。OpenAI 的 SSE 使用統一的 data: 前綴行，JSON 結構中透過 choices 陣列傳遞增量內容。SseParser 根據提供商類型選擇正確的解析分支，最終都轉換為統一的 StreamEvent 列舉。

### 9.8 HTTP 客戶端的建構與配置

http_client.rs 模組負責建構和配置所有 API 通訊使用的 HTTP 客戶端。build_http_client 函數接受 ProxyConfig 和其他配置參數，回傳一個配置好的 reqwest Client 實例。

HTTP 客戶端的配置包含多個面向。連線池管理：客戶端維護一個持久的 HTTP 連線池，避免每次請求都建立新的 TCP 連線和 TLS 握手。連線池的大小和空閒連線的逾時時間都可以配置。

TLS 配置：客戶端使用系統的 CA 憑證庫來驗證伺服器憑證。在某些企業環境中，可能需要配置自訂的 CA 憑證或停用憑證驗證（不推薦）。

逾時配置：客戶端有多個逾時參數——連線逾時（建立 TCP 連線的最大等待時間）、讀取逾時（等待伺服器回應的最大時間）和總體逾時（整個請求的最大持續時間）。對於 SSE 串流請求，總體逾時通常被設定為非常長的時間，因為串流可能持續數分鐘。

使用者代理：客戶端的 User-Agent 標頭包含 Claw Code 的版本資訊，幫助提供商追蹤客戶端的使用情況。

重試邏輯：雖然基本的 HTTP 客戶端不包含重試邏輯，但 api crate 的上層程式碼會根據回應的狀態碼決定是否重試。429（Too Many Requests）回應會觸發帶有退避時間的自動重試。5xx（伺服器錯誤）回應會觸發有限次數的指數退避重試。


### 7.6 Crate 的公開介面設計

每個 crate 的公開介面（即 pub 標記的型別、函式和 trait）經過精心設計，以最小化層與層之間的耦合。一個好的公開介面應該只暴露使用者真正需要的功能，隱藏內部的實作細節。

runtime crate 的公開介面是最為複雜的，因為它被多個 crate 依賴。它暴露的核心型別包括：ConversationRuntime（對話引擎的入口）、Session 及其相關的訊息型別（用於工作階段管理）、PermissionMode 和 PermissionPolicy（用於權限控制）、SandboxConfig（用於沙箱配置）、RuntimeConfig 和 ConfigLoader（用於設定管理）。每個公開型別都有詳細的 Rust doc 註解，說明其用途、欄位含義和使用範例。

api crate 的公開介面相對簡潔：ProviderClient 和 MessageRequest/MessageResponse 是最重要的公開型別。detect_provider_kind 和 resolve_model_alias 是最重要的公開函式。StreamEvent 列舉讓上層程式碼能夠處理 SSE 事件。ApiError 讓上層能夠根據錯誤類型做出適當的處理決策。

tools crate 的公開介面圍繞 execute_tool 函式展開。它還暴露了工具規格列表（用於生成工具定義）和全域 Registry 的存取函式。

這種謹慎的介面設計確保了修改一個 crate 的內部實作不會影響其他 crate——只要公開介面保持不變。這是 Monorepo 架構中避免波紋效應（ripple effect）的關鍵策略。

### 8.6 工作流程引擎的架構理念

工作流程引擎是 Claw Code 中相對較新的子系統，其架構理念反映了對 AI 驅動的 CI/CD 自動化的深入思考。傳統的 CI/CD 工具（如 Jenkins、GitHub Actions）使用預定義的管線——每個步驟的行為在執行前就已經確定。而 Claw Code 的工作流程引擎採用了事件驅動的自適應模型——系統根據每個步驟的實際結果動態地決定下一步的行為。

這種自適應模型的核心元件是 Lane 事件系統和策略引擎。Lane 事件系統定義了工作流程中可能發生的所有事件類型——從 Started 到 Green（測試通過）到 Red（測試失敗）到 Finished。策略引擎定義了在不同事件發生時系統應該採取什麼行動——是繼續下一步、暫停等待、觸發復原還是放棄。

recovery_recipes.rs 是自適應模型的關鍵創新。當測試失敗或建構出錯時，系統不是簡單地標記為失敗，而是嘗試自動復原。復原方案由 AI 模型驅動——系統將失敗的上下文（錯誤訊息、相關的程式碼、最近的修改歷史）提供給 AI 模型，讓模型分析原因並產生修復建議。然後，系統根據修復建議自動執行修復操作（如修改程式碼、更新配置），最後重新執行失敗的步驟來驗證修復。

這種 AI 驅動的自動復原在某些場景下非常有效——例如，簡單的語法錯誤、缺少的 import 語句、型別不匹配等常見錯誤通常可以被 AI 模型準確地診斷和修復。然而，對於複雜的邏輯錯誤或設計缺陷，自動復原可能不足以解決問題，此時系統會回退到人工介入模式。

### 9.9 Usage 追蹤的完整流程

UsageTracker 的工作流程貫穿整個對話回合。讓我們追蹤一次完整的 token 用量追蹤流程。

在對話回合開始時，build_api_request 方法會估算請求的輸入 token 數量。這個估算是基於訊息歷史的總長度、系統提示詞的長度和工具定義的總長度。估算值用於判斷是否即將超過模型的上下文視窗限制。

當 API 回應到達時，SSE 事件中的 message_delta 事件攜帶了本次回應的精確 token 用量。Usage 結構包含四個欄位：input_tokens（輸入 token 數量，即請求中所有文字的 token 數量）、output_tokens（輸出 token 數量，即回應中所有文字的 token 數量）、cache_creation_input_tokens（用於建立提示詞快取的 token 數量）和 cache_read_input_tokens（從提示詞快取中讀取的 token 數量）。

UsageTracker 在收到每次 API 回應的 Usage 後，將各個欄位的值累加到工作階段的總計中。同時，UsageTracker 使用 ModelPricing 來計算本次回應的成本，也累加到工作階段的總成本中。

UsageCostEstimate 結構提供了成本的分項明細：input_cost_usd（輸入 token 的成本）、output_cost_usd（輸出 token 的成本）、cache_creation_cost_usd（建立快取的成本）和 cache_read_cost_usd（讀取快取的成本）。這些分項讓使用者能夠了解成本的分布——如果輸入成本佔比很高，可能需要精簡系統提示詞或減少工具定義；如果快取讀取比例很低，可能需要最佳化快取策略。

/cost 斜線命令從 UsageTracker 取得累計資料，格式化後顯示給使用者。典型的輸出包括：各類型的 token 數量、各類型的成本估算、快取命中率、以及總成本。
