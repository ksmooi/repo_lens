---
repo: opencode-ai/opencode
file: 9-questions
studied_at: 2026-05-31
commit_sha: 73ee493
---

# OpenCode · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼選 Bubble Tea 而非 Bubble Tea 的競爭者（如 tview、termui）？**
  - 我目前的推測：[UNVERIFIED] Bubble Tea 的 Elm Architecture 最接近 React/Redux 開發者的心智模型，且 charmbracelet 生態內的 bubbles（components）豐富。但與 Charm 團隊後續接手 Crush 的事實對照，可能選 Bubble Tea 本身就是因為後續的 Charm 生態佈局
  - 相關程式碼：[`cmd/root.go:120`](https://github.com/opencode-ai/opencode/blob/73ee493/cmd/root.go#L120)

- [ ] **goroutine leak 的風險有多大？**
  - 在 permission blocking channel 的設計中，如果使用者關閉 terminal（而非正常 quit），goroutine 是否會 leak？[`permission.go:100`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/permission/permission.go#L100) 的 `<-respCh` 如果 producer 消失怎麼辦？
  - 我目前的推測：`context.Context` 的 cancellation 應該會處理——agent loop 的 context 取消時，`permission.Request()` 所在的 goroutine 應該會收到 `ctx.Done()`。但[程式碼](https://github.com/opencode-ai/opencode/blob/73ee493/internal/permission/permission.go#L74-L100)中沒有看到 select on `ctx.Done()`。可能是已知但未解決的問題

- [ ] **為什麼將 LSP clients 設計在 app level 而非 tool level？**
  - LSP clients 的建立與關閉是 app 層的責任（`app.initLSPClients()`、`app.Shutdown()`），但實際使用是工具層（view tool、edit tool、diagnostics tool）透過共享的 `map[string]*lsp.Client` 來存取
  - 我目前的推測：[UNVERIFIED] 因為 LSP server process 的生命週期應該跟 app 一致而非跟 tool call 一致（LSP server 啟動成本高），所以放在 app level 合理。但透過 map pointer 傳遞而非 interface 封裝讓 testing 變得困難

- [ ] **MCP tool 初始化的競爭問題**
  - [`cmd/root.go:195-207`](https://github.com/opencode-ai/opencode/blob/73ee493/cmd/root.go#L195-L207) 的 `initMCPTools()` 在 background goroutine 執行，但如果 agent 已經開始處理請求了，MCP tools 才載入完成，會發生什麼？
  - 相關程式碼：`CoderAgentTools()` 在 [`agent/tools.go:22`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/llm/agent/tools.go#L22) 中先呼叫 `GetMcpTools(ctx, permissions)` 建立時就讀取一次。但 MCP 工具的更新是在 background goroutine 中發生的，[`mcp-tools.go`](https://github.com/opencode-ai/opencode/blob/73ee493/internal/llm/agent/mcp-tools.go) 似乎是後續再更新 tool list
  - [UNVERIFIED] 如果一個 MCP 工具在第 3 次 tool call 後才註冊成功，之前的 model prompt 中沒有這個 tool 的 schema，它永遠不會被呼叫到

## 想問維護者的問題

- 為什麼專案後來決定封存並轉到 Charm 團隊的 Crush？是技術棧選擇（Go + Bubble Tea vs 其他）還是團隊規模的考量？
- Auto-compact 的 summarizer 用哪個 model 設定？如果 summarizer 因為 max token 過小而 truncate，compact 後的内容會不會遺失重要細節？
- Provider 層為什麼沒有直接用 OpenAI-compatible API 統一所有 provider？Anthropic 的 message API 與 OpenAI 的 chat completion API 有所不同，是選擇直接支援各自的 native API 而非轉換層？

## 下次再看時的待辦

- [ ] 探索 `mcp-tools.go` 的完整實作，理解 MCP tools 如何在 agent 生命週期中被動態追加
- [ ] 對照 Charm 團隊的 Crush 與 OpenCode 的差異，理解哪些設計被保留下來、哪些被重寫
- [ ] 測試 `permission.Request()` 在 context cancel 時的行為——確認是否有 goroutine leak

## 跨專案對照備忘

- **Generic Provider Abstraction**（Pattern 1）與 vLLM 的 `model_runner` 抽象類似——都用了泛型 Type Parameter 來處理多個特化實作。在 `_patterns/llm-provider-abstraction.md` 中有相關的跨專案模式
- **Tool Interface with Clean Contract**（Pattern 5）與 Claude Code、OpenHands 的 tool registration 設計相似——共用一個 `ToolInfo` / metadata 結構來描述工具能力。這在 Claude Code 的 `Tool.use()` TypeScript interface 有類似的 pattern
- **Multi-Agent 分工**（Pattern 4）在 `_patterns/agent-state-machine.md` 中有討論到不同 agent 各有不同 tool set 的設計，OpenCode 的實作是「安全隔離」這個維度的好範例

### 候選 Pattern

「**Session-based Permission with Blocking Channel**」——這個設計（tool 執行前需要使用者互動式許可）在多個 AI coding agent 中都有類似實作（Claude Code 的 JSON editor commands、Codex CLI 的 permission prompts），但 OpenCode 用 Go channel 的同步阻塞是不同語言下的不同實現。若在其他 Go 實作的 agent 中也看到類似模式，可以抽象為 `interactive-permission-channel.md`。

2 個 repo 觀察到（OpenCode 與潛在其他 Go-based AI tool），未達 3 個門檻，暫不新增到 `_patterns/`。
