---
repo: opencode-ai/opencode
file: 3-key-patterns
studied_at: 2026-05-31
commit_sha: 73ee493
---

# OpenCode · 值得偷學的設計

## Pattern 1: Generic Provider Abstraction（泛型 Provider 抽象）

**是什麼**：用 Go 1.18+ 的 Type Parameter 實現 LLM provider 的抽象層，每個 provider client 保有自己型別特有的 methods，由泛型基底 `baseProvider[C ProviderClient]` 提供共用的 `SendMessages` 與 `StreamResponse`。

**程式碼位置**：[`internal/llm/provider/provider.go:81-84`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/llm/provider/provider.go#L81-L84)

```go
type baseProvider[C ProviderClient] struct {
    options providerClientOptions
    client  C
}

func (p *baseProvider[C]) SendMessages(ctx context.Context, ...) (*ProviderResponse, error) {
    messages = p.cleanMessages(messages)
    return p.client.send(ctx, messages, tools)  // delegates to the typed client
}
```

不同的 provider clients（`AnthropicClient`、`OpenAIClient`、`GeminiClient`）各自實作自己的 `send()` 和 `stream()` 方法，底層使用不同的 HTTP client 和 auth 邏輯。建立 provider 時用 switch-case 分別實例化（[`provider.go:86-168`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/llm/provider/provider.go#L86-L168)），例如 `Groq` 與 `OpenRouter` 實際上都使用 `OpenAIClient` 但換 base URL 與 headers。

**為什麼有效**：
- 避免了 `interface{}` 型別斷言，每個 provider client 保有完整的型別
- `baseProvider` 只依賴 `ProviderClient` interface（`send()` / `stream()`），遵循依賴反轉原則（DIP）
- 新 provider 加入時只需實作 `ProviderClient` 介面 + 一行 `NewProvider` switch case

**替代方案**：
- **傳統 interface 方式**：直接讓 `Provider` 是 interface，每個 provider 直接實作 `SendMessages` 與 `StreamResponse`。這在 Go 1.18 前是唯一選擇，缺點是每個 provider 必須重複實作 `cleanMessages()` 等共用邏輯
- **Embedding + 方法覆寫**：base struct 嵌入，子 struct 覆寫特定方法。缺點是子 struct 可以選擇不呼叫基底方法，違反 Liskov 替代原則
- **策略模式（Strategy Pattern）**：用 factory function 回傳 interface。這在沒有 generics 的語言是主流做法，但需要更多 boilerplate

**何時可以借用**：當你在 Go 中有「多方實體」實作同一介面，且有 3+ 個共用方法時。泛型 Type Parameter 特別適合「一個基底 + 多個特化 client」的場景。

**不適用的情境**：如果各 provider 的串流行為差異太大（例如一個是 SSE、一個是 WebSocket、一個是 polling），共用 `baseProvider` 反而會變成限制。

---

## Pattern 2: Sync Blocking Permission via Channel（同步阻塞權限系統）

**是什麼**：當 agent 需要執行一個需要許可的工具，它在 goroutine 內部用 channel 同步等待 TUI 的使用者回應。TUI 透過 `Grant()` / `Deny()` 向 channel 發送結果解鎖。

**程式碼位置**：[`internal/permission/permission.go:74-100`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/permission/permission.go#L74-L100)

```go
func (s *permissionService) Request(opts CreatePermissionRequest) bool {
    // 1. Check auto-approve sessions
    if slices.Contains(s.autoApproveSessions, opts.SessionID) {
        return true
    }
    // 2. Check persistent permissions
    for _, p := range s.sessionPermissions { ... }
    // 3. Publish event to TUI, block on channel
    respCh := make(chan bool, 1)
    s.pendingRequests.Store(permission.ID, respCh)
    s.Publish(pubsub.CreatedEvent, permission)  // TUI shows dialog
    granted := <-respCh                         // BLOCK until TUI responds
    return granted
}
```

TUI 端收到事件後顯示 dialog，使用者按下 Allow 後呼叫（[`tui.go:275-289`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/tui/tui.go#L275-L289)）：

```go
case dialog.PermissionAllow:
    a.app.Permissions.Grant(msg.Permission)  // respCh <- true
```

**為什麼有效**：
- 非常自然的 Go idiomatic pattern——channel 作為同步原語
- agent 不需要知道 TUI 的存在，它只是在一個 goroutine 裡 blocking wait
- 非互動模式透過 `AutoApproveSession()` 繞過 blocking，不影響 UX
- `sync.Map` 搭配 `PermissionRequest.ID` (UUID) 確保每一個請求可追蹤

**替代方案**：
- **Callback 模式**（JavaScript 常用）：傳入 callback function，TUI 完成後呼叫 callback。在 Go 中不如 channel 自然，且 callback 難以取消
- **Promise/Future**：需要自行實作。Go 的 channel 已經是語言級別的 future
- **Polling**：agent 輪詢「這個 permission 被回應了嗎？」——浪費資源且延遲高

**何時可以借用**：當你有一個後台 goroutine 需要等待 UI thread 的使用者決策時。Channel-based blocking 是 Go 特有的語言特性，其他語言可能需要 Condition Variable 或 CompletableFuture。

---

## Pattern 3: PubSub Event Bus 作為 TUI-Agent 橋梁

**是什麼**：泛型 `Broker[T]` 實作 publish-subscribe 模式，agent 與其他服務發布事件（streaming text、permission request、session 變更），TUI 消費事件更新畫面。

**程式碼位置**：[`internal/pubsub/broker.go`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/pubsub/broker.go)

使用方式（[`cmd/root.go:249-282`](https://github.com/opencode-ai/opencode/blob/73ee493/cmd/root.go#L249-L282)）：

```go
setupSubscriber(ctx, &wg, "logging", logging.Subscribe, ch)
setupSubscriber(ctx, &wg, "sessions", app.Sessions.Subscribe, ch)
setupSubscriber(ctx, &wg, "messages", app.Messages.Subscribe, ch)
setupSubscriber(ctx, &wg, "permissions", app.Permissions.Subscribe, ch)
setupSubscriber(ctx, &wg, "coderAgent", app.CoderAgent.Subscribe, ch)
```

每個 `Suscriber` interface 透過 `Subscribe(ctx) <-chan Event[T]` 回傳一個唯讀 channel，`cmd/root.go` 的 `setupSubscriber` 將這些 channel merge 到一個共同的 `chan tea.Msg`。TUI 的 Bubble Tea 框架接收 `tea.Msg` 做 Update。

**為什麼有效**：
- 鬆散耦合——agent 不知道 TUI 的存在，它只是發布事件
- 回壓處理——`setupSubscriber` 有 2 秒 timeout，slow consumer 不會阻塞 producer（[`cmd/root.go:234-236`](https://github.com/opencode-ai/opencode/blob/73ee493/cmd/root.go#L234-L236)）
- 優雅關閉——`context.WithCancel` 統一管理所有 goroutine 生命週期

**替代方案**：
- **直接 callback**：agent 直接呼叫 TUI 的方法。破壞了分層，agent 需要 import TUI package
- **共享 mutable state + mutex**：agent 寫入、TUI 輪詢。輪詢浪費 CPU，且 race condition 風險高
- **Event 全域變數**：常見的 Go anti-pattern，難以單元測試

**何時可以借用**：當你有「多個 producer → 單一 consumer（UI）」的架構時。如果 producer 數量 > 5，pubsub 的結構化優勢更明顯。

---

## Pattern 4: Multi-Agent 分工（Coder + Task + Title + Summarizer）

**是什麼**：OpenCode 不是單一 agent，而是四個不同的 agent 角色，每個有自己的 model 設定與 tool set：

| Agent 角色 | config key | 用途 | Tools |
|---|---|---|---|
| Coder | `agents.coder` | 主要 coding agent | 11 個完整工具（bash, edit, write, patch 等） |
| Task | `agents.task` | `agent` tool 的子任務 | 唯讀工具（glob, grep, ls, view, sourcegraph） |
| Title | `agents.title` | 自動產生 session title | 無工具 |
| Summarizer | `agents.summarizer` | context window 滿時自動摘要 | 無工具 |

**程式碼位置**：Coder agent 建立於 [`app.go:63-78`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/app/app.go#L63-L78)，Task agent 動態建立於 [`agent-tool.go:57`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/llm/agent/agent-tool.go#L57)，Title provider 建立於 [`agent.go:85-90`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/llm/agent/agent.go#L85-L90)。

**為什麼有效**：
- **安全隔離**：Task agent 只有唯讀工具，即使 prompt injection 也不能修改檔案
- **成本控制**：Title 和 Summarizer 用較小的 model 與 `maxTokens: 80`，避免浪費
- **各自 context**：Task agent 有自己的 session context，不干擾主 session

**替代方案**：
- **單一 agent 動態切換 tool set**：同一 agent 根據任務類型決定可用工具，但容易讓 model 困惑
- **無 agent 區分**：全部用同一個 model，但成本與安全性不可控
- **外部 API 呼叫**：子任務透過 HTTP API 呼叫另一個服務，Overkill

**何時可以借用**：當你的 agent 系統需要執行多種不同安全等級或成本考量的任務時。Task agent 特化為「搜尋 + 唯讀」是比「限制 Coder agent 不要寫檔案」更可靠的設計。

---

## Pattern 5: Tool Interface with Clean Contract

**是什麼**：工具系統定義了清晰的 `BaseTool` interface + `ToolInfo` metadata + `ToolCall`/`ToolResponse` 資料結構，讓加入新工具只需實作三個方法。

**程式碼位置**：[`internal/llm/tools/tools.go`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/llm/tools/tools.go)

關鍵 interface：

```go
type BaseTool interface {
    Info() ToolInfo                    // name, description, parameters (JSON schema)
    Run(ctx context.Context, call ToolCall) (ToolResponse, error)
}
```

`ToolInfo` 的 `Parameters` 欄位是 `map[string]any`，直接對應 LLM 的 function calling schema 格式。每個 tool 實作自己的參數解析與執行邏輯。

**為什麼有效**：
- 新工具只需要實作兩個方法，非常低的加入門檻
- `ToolInfo` 同時是 LLM 的 function calling schema 來源與自動生成文件
- `ToolResponse` 可以包含 `IsError`、`Metadata`，豐富錯誤處理
- `Run()` 接受 context，支援 cancellation 傳遞

**替代方案**：
- **字串式 tool 定義**：用 JSON/YAML 定義 tool，共用一個 executor。靈活性低，難以處理不同 tool 的特殊邏輯
- **Plugin 系統**：動態載入外部 plugin（如 gRPC），overkill for CLI tool
- **繼承體系**：OOP 風格的 tool base class + 子 class 覆寫。Go 沒有繼承，composition 是合適的替代

**何時可以借用**：任何需要「AI agent 可以執行外部操作」的系統。這個 interface 設計足夠通用，且可以平行對應到 OpenAI / Anthropic / Gemini 的 function calling 規格。

---

## Pattern 6: Auto-Compact Context Management

**是什麼**：當 session 的 token 使用量達到 model context window 的 95% 時，自動啟動 summarization——用較小的 model 總結對話歷史，建立新 session 讓使用者可以繼續提問。

**程式碼位置**：[`internal/tui/tui.go:306-344`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/tui/tui.go#L306-L344)

```go
// In TUI event handler:
case pubsub.Event[agent.AgentEvent]:
    if payload.Done && payload.Type == agent.AgentEventTypeResponse {
        tokens := a.selectedSession.CompletionTokens + a.selectedSession.PromptTokens
        if (tokens >= int64(float64(contextWindow)*0.95)) && config.Get().AutoCompact {
            return a, util.CmdHandler(startCompactSessionMsg{})
        }
    }
```

啟動後，`Summarize(ctx, sessionID)` 會用 `summarizeProvider` 產生摘要，建立一個新 session 並將摘要作為首則訊息。舊 session 保留 `SummaryMessageID` 指向摘要點。

**為什麼有效**：
- 使用者完全無感——不需要手動按下「新對話」，也不會突然遺失 context
- 95% threshold 留了 buffer，避免趕在 token 爆掉的那一刻
- 透過 `config.AutoCompact` 可以關閉，使用者保有控制權
- 摘要後的歷史截斷透過 `SummaryMessageID` 實現——從該 message 開始之後的歷史保留下來給新 session 參考

**替代方案**：
- **直接截斷**：丟棄最早的 message。簡單但遺失資訊
- **手動新對話**：使用者自己建立新對話、貼上 context。UX 差
- **依賴 LLM 的 context window**：如果 model context window 很大（如 Gemini 2M），可能根本不需要 auto-compact

**不適用的情境**：如果 model context window 足夠大且使用者對話長度不會超過，auto-compact 是多餘的。也適合使用者明確想要保留完整對話歷史的場景。

---

## Pattern 7: Named Arguments for Custom Commands

**是什麼**：自訂命令（markdown 檔案）中可以使用 `$ARG_NAME` 佔位符，執行時彈出 dialog 讓使用者填入參數，多次出現的相同佔位符會自動填入同一個值。

**程式碼位置**：[`internal/tui/tui.go:417-442`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/tui/tui.go#L417-L442)

自訂命令範例（`~/.config/opencode/commands/fetch-context.md`）：
```markdown
RUN gh issue view $ISSUE_NUMBER --json title,body,comments
RUN git grep --author="$AUTHOR_NAME" -n .
```

執行時，如果內容中有 `$ARG_NAME` 模式（正規表示式 `$[A-Z_]+`），系統會解析所有 unique placeholder，彈出 `MultiArgumentsDialog`。提交後，系統進行字串替換（[`tui.go:431-434`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/tui/tui.go#L431-L434)）：

```go
for name, value := range msg.Args {
    placeholder := "$" + name
    content = strings.ReplaceAll(content, placeholder, value)
}
```

**為什麼有效**：
- 比環境變數或 hardcoded 參數靈活得多
- 同一參數只需填一次，自動替換所有出現
- markdown 格式讓開發者可以寫多行命令、夾雜註解

**替代方案**：
- **硬編碼命令**：沒有參數，每種情境要寫一個命令檔案。重複
- **shell script + 位置參數**：`$1`、`$2`，缺乏語意且無法在 UI 引導
- **JSON 參數描述**：需要額外 schema 檔案，比 markdown 內嵌 $ARG 重

**何時可以借用**：當你為開發者設計可擴展的命令系統時。將參數佔位符直接嵌在內容中，比外部的 schema 描述更接近「dogfooding」——開發者直接用他們熟悉的 shell script 語法。

## API 設計品味的觀察

OpenCode 的 API 設計（指內部 package 之間的 contract）有幾個一貫的做法：

1. **Interface 定義 slim**——`Service` interface 通常只包含真正需要的方法（見 [`agent.go:48-57`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/llm/agent/agent.go#L48-L57)），沒有不必要的 getter/setter
2. **PubSub 作為核心抽象**——service 回傳 read-only channel 而非 callback
3. **Composition over inheritance**——`baseProvider[C]` 透過泛型 Type Parameter 實現，而非 struct embedding 覆寫
4. **明確的錯誤類型**——`ErrRequestCancelled`、`ErrSessionBusy`、`ErrorPermissionDenied` 都是 exported sentinel error

## 對相容性的態度

專案處於早期開發階段（無正式 release），沒有版本相容性的壓力。但架構層的設計（Provider interface、Tool interface、PubSub broker）已經足夠穩定，即使後來接手到 Charm 團隊的 Crush 也能沿用類似架構。
