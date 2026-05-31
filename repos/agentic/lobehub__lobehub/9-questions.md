---
repo: lobehub/lobehub
file: 9-questions
studied_at: 2026-05-27
commit_sha: bcc31ca
---

# LobeHub · 未解問題

## 還沒搞懂的設計決策

- [ ] **Context Engine pipeline 的執行順序如何影響最終 prompt？**
  - 30+ 個 processor 的順序定義了 context 的累積規則，但對一個特定 session，最終執行了哪些 processor？順序如何決定？
  - 我目前的推測：順序由 runtime config 的 `pipeline` 陣列決定，但某些 provider 是條件式啟動的（只有特定 agent 類型或特定 plugin 啟用時才執行）[UNVERIFIED]
  - 相關程式碼：[`packages/context-engine/src/pipeline.ts:19`](https://github.com/lobehub/lobehub/blob/bcc31ca/packages/context-engine/src/pipeline.ts#L19)

- [ ] **heterogeneous-agents 的 Claude Code/Codex 整合在實際部署中如何運作？**
  - 從檔案結構看，`packages/heterogeneous-agents/src/spawn/spawnAgent.ts` 是透過子行程（subprocess）執行的。但在生產環境（Docker / Kubernetes）中，Claude Code 需要 terminal 互動、API key、網絡訪問——這些在容器化環境中如何配置？
  - 我目前的推測：這主要用在 desktop app 或開發環境，production 部署可能透過 cloud-sandbox 或限制使用場景 [UNVERIFIED]
  - 相關程式碼：[`packages/heterogeneous-agents/src/spawn/spawnAgent.ts`](https://github.com/lobehub/lobehub/blob/bcc31ca/packages/heterogeneous-agents/src/spawn/spawnAgent.ts)

- [ ] **SSRF-safe-fetch 與 web-crawler 的關係是什麼？**
  - 有獨立的 `packages/ssrf-safe-fetch` 和 `packages/web-crawler` 套件。ssrf-safe-fetch 是網絡請求的安全封裝，web-crawler 是網頁爬取工具。兩者是競爭關係（web-crawler 繞過了 ssrf-safe-fetch？）還是合作關係（web-crawler 用 ssrf-safe-fetch？）
  - 我目前的推測：web-crawler 是 builtin-tool 背後的實作，ssrf-safe-fetch 是更底層的網絡請求安全層，但兩者的 package 依賴關係不明 [UNVERIFIED]

- [ ] **Session 的「forceFinish」機制在實務中是否會產生不一致的 state？**
  - `forceFinish = true` 時，tool 仍然允許執行完畢，但下一個 LLM call 的 tool 會被 strip。這表示在 forceFinish 的 step 中，被執行的 tool 的結果不會被 LLM「看到」就結束了。這對 user experience 的影響是什麼？使用者看到 tool 執行完了但 agent 沒有處理結果？
  - 相關程式碼：[`packages/agent-runtime/src/core/runtime.ts:90-98`](https://github.com/lobehub/lobehub/blob/bcc31ca/packages/agent-runtime/src/core/runtime.ts#L90-L98)

- [ ] **Agent Council 模式的實際行為是什麼？**
  - `packages/context-engine/src/processors/AgentCouncilFlatten.ts` 暗示存在一種多 agent 會議模式，但不能從程式碼確認這個 group conversation 的具體行為。這是一種「多個 agent 獨立產生回應 → flatten → 同時顯示」的展示模式，還是真的有「多個 agent 互相對話」的協作模式？
  - 相關程式碼：[`packages/context-engine/src/processors/AgentCouncilFlatten.ts`](https://github.com/lobehub/lobehub/blob/bcc31ca/packages/context-engine/src/processors/AgentCouncilFlatten.ts)

## 想問維護者的問題

- LobeHub 從 chat UI 轉型為 CAO 平台過程中，最大的架構重構是什麼時候發生的？是 v2.0 一次大改，還是逐步演進？
- AgentRuntime 跟 ContextEngine 之間的分界線（誰決定 call LLM，誰決定組裝什麼進 context）在實務上會不會 overlap？
- 為什麼選擇 MCP 作為 tool 協定而非直接用 OpenAI function calling format？MCP 在 TypeScript 生態中的支援度如何？
- heterogeneous-agents 的 Claude Code adapter 在什麼場景下被實際使用？是開發者調試時的 helper，還是 production 中的標準 agent 工具？

## 下次再看時的待辦

- [ ] 跑 local 開發環境（`pnpm dev:spa`），實際追蹤一個完整 request 的 frontend→backend→LLM→response 路徑
- [ ] 深入研究 `packages/agent-runtime/src/agents/GraphAgent.ts` — 這是 graph-based agent 的實作，跟 GeneralChatAgent 的比較可能很有價值
- [ ] 閱讀 `packages/memory-user-memory/` 的實作細節，理解長期記憶的存取模式
- [ ] 看 `packages/agent-signal/` 的事件系統設計 — 這是跨 agent 通訊的關鍵
- [ ] 實際建立一個 multi-agent group 並觀察 GroupOrchestrationRuntime 的行為
- [ ] 讀取 `packages/context-engine/src/providers/` 下的所有 provider，理解 context 組裝的全貌

## 跨專案對照備忘

- **Context Engine pipeline pattern**：Context Engine 的 pipeline-based context assembly 跟 LangGraph 的 node-based 方法不同，值得後續觀察是否在其他 agent 框架也出現類似模式 → 候選 pattern
- **Executor Pattern**：指令型別的 executor 註冊表設計與微服務架構中常見的 strategy pattern 類似，但在 agent 框架中 LobeHub 是第一個採用這種方式的開源專案
- **五層 intervention system**：這種多層級的人機干預系統在其他 agent 框架中尚未看到完整實作（其他框架多為 binary: 要嘛全部 auto-run，要嘛全部 manual）。若在 3 個獨立 repo 都觀察到此設計，可收錄為 pattern
- **Supervisor + Executor 編排**：GroupOrchestrationRuntime 的 Supervisor-Decide → Executor-Execute 迴圈與 microservice orchestration 中的 saga pattern 類似，但用於 agent 場景。對照 LangGraph 的 `send()` edge 和 AutoGen 的 GroupChatManager，三者本質都是中央編排者模式，只是實作方式不同 ([`packages/agent-runtime/src/groupOrchestration/`](https://github.com/lobehub/lobehub/blob/bcc31ca/packages/agent-runtime/src/groupOrchestration/))

### 既有 Pattern 對照

- **agent-state-machine**：LobeHub 的 AgentState.status（idle → running → waiting_for_human → interrupted）構成一个簡單的狀態機，但整體控制流仍是 instruction-dispatch loop，非顯式 graph。對照此 pattern 的「不適用」情境：LobeHub 的 Plan→Execute loop 比 LangGraph 的 graph 輕量，但要支援完整的中斷續跑，可能會需要更多狀態管理 → 目前不建議擴充此 pattern
- **llm-provider-abstraction**：LobeHub 的 `modelRuntime` callback 提供了極簡的 LLM 抽象（`(payload) => AsyncIterable<Chunk>`），比 LangChain 的 `BaseChatModel` 薄很多。這種設計讓測試 mock 非常容易 → 已有 LLM provider abstraction pattern，無需新增
