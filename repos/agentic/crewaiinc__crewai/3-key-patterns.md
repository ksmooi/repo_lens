---
repo: crewAIInc/crewAI
file: 3-key-patterns
studied_at: 2026-06-03
commit_sha: ee70702
---

# CrewAI · 值得偷學的設計

## Pattern 1: 宣告式 Agent-Crew-Task 三段式

**是什麼**: 將 agent 系統拆成三個正交的宣告式元件——Agent（誰做）、Task（做什麼）、Crew（怎麼編排）。使用者只需宣告這三者的屬性，執行引擎自動處理協作細節。

**為什麼有效**: 多數 agent 框架要求使用者理解控制流的運作原理（StateGraph 的節點與邊、ReAct loop 的細節）。CrewAI 的三段式把「能力宣告」跟「執行邏輯」徹底分離：

- Agent 只回答：我是誰（role）、我要達到什麼（goal）、我的背景（backstory）、我有什麼工具（tools）
- Task 只回答：要做什麼（description）、產出什麼（expected_output）、誰來做（agent）、可用的工具（tools）
- Crew 只回答：用哪種流程（process）、agent 有誰（agents）、task 有誰（tasks）

**程式碼位置**: [`crew.py:159-392`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/crew.py#L159-L392)（Crew 的宣告式欄位）、[`agent/core.py`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/agent/core.py)（Agent 類別）、[`task.py`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/task.py)（Task 類別）

**何時可以借用**: 任何需要讓「非專家使用者」定義 agent 流程的場景。如果你的 target audience 是應用開發者而非 AI 研究者，這個抽象層次恰到好處。

**注意事項**: 宣告式模型對於需要細粒度控制的高階場景會顯得綁手綁腳。CrewAI 的解法是用 Flow DSL 補上這個缺口——但這也意味著使用者需要學習第二套抽象。此外，三個元件的邊界並非絕對清晰：Task 可以有獨立的 tools（覆蓋 Agent 的 tools）、Crew 可以有獨立的 memory 設定，這在某些情境下會造成「同一個設定可以在三處指定」的困惑。

**替代方案**:
- LangGraph: 用 StateGraph 程式化定義節點與邊，提供最大靈活性但入門門檻高
- AutoGen: 用 ConversationAgent + GroupChat 模型，agent 之間自由對話，無宣告式結構

---

## Pattern 2: 雙軌 Agent Loop（Native Function Calling + ReAct Fallback）

**是什麼**: Agent executor 根據 LLM provider 的能力自動選擇執行模式：若 provider 支援 function calling，使用 native tool call 模式；否則回退到傳統的 ReAct 文字模式。

**為什麼有效**: 不是所有 LLM provider 都支援 native function calling（或支援程度不同）。雙軌設計讓 CrewAI 的 agent 可以在任何 LLM 上執行——從 GPT-4 到本地運行的 Ollama 模型。選擇邏輯集中在一處（`_invoke_loop()`），維護成本可控。

**程式碼位置**: [`crew_agent_executor.py:306-325`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/agents/crew_agent_executor.py#L306-L325) — 選擇邏輯；[`crew_agent_executor.py:327-465`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/agents/crew_agent_executor.py#L327-L465) — ReAct loop；[`crew_agent_executor.py:467-575`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/agents/crew_agent_executor.py#L467-L575) — Native tool loop

**何時可以借用**: 當你的 agent 系統需要支援多種 LLM provider，且不希望使用者因為換 provider 而需要改寫 agent loop 時。

**注意事項**:
- 這個 pattern 有個明顯的歷史包袱：CrewAgentExecutor 已被標記為 deprecated（L142-148），正逐步遷移到新的 `AgentExecutor`。雙軌維護成本在長期來看可能不值得。
- `_is_tool_call_list()`（L614-645）需要識別至少 5 種不同的 tool call 格式（OpenAI、Anthropic、自訂 dict 等），這是跨 provider 相容性的主要瓶頸。
- 若大部分使用者只用 GPT-4 或 Claude（都支援 function calling），ReAct 分支的程式碼就成了 dead code。

**替代方案**:
- 統一使用 function calling，不支援的 provider 就報錯——簡單但限制多
- 全部走 ReAct 文字模式——相容性最好但效能較差，且無法利用 function calling 的結構化優勢
- 抽像出一個中介層，將所有 provider 的 tool call 統一成一個格式——這個方案理論上最乾淨，但需要對每個 provider 寫 adapter

---

## Pattern 3: Event Bus 作為內部通訊與可觀測性的統一介面

**是什麼**: 整個 CrewAI 內部使用一個全域的 Event Bus（`crewai_event_bus`），所有重要事件（agent 開始/完成、task 開始/完成/失敗、knowledge 查詢等）都透過這個 bus 發送，listener 可以註冊並回應事件。

**為什麼有效**: 傳統的 agent 框架把 logging、tracing、callback 分散在不同的機制中。CrewAI 統一用事件匯流排解決三個問題：

1. **Observability**: `TraceCollectionListener` 監聽所有事件，產出 OpenTelemetry spans
2. **State management**: Event record 是 checkpoint 的核心資料結構——只要重播事件就能還原執行狀態
3. **Extensibility**: 開發者可以註冊自訂 listener，不需要修改核心程式碼

**程式碼位置**: [`events/event_bus.py`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/events/event_bus.py)；事件類型定義在 [`events/event_types.py`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/events/event_types.py) 與 `events/types/` 目錄下的多個檔案

**何時可以借用**: 任何需要「多個元件相互通訊但不想產生直接依賴」的系統。特別適合 agent 框架、工作流引擎、或任何需要 tracing + state management 的系統。

**注意事項**:
- 全域 bus 在測試時需要特別處理——多個測試共用一個 bus 會造成事件干擾。CrewAI 的測試用 `vcr.py` 錄製 LLM 回放避免了這個問題，但自訂 listener 的測試仍需注意。
- 事件類型的爆炸：每個 subsystem 都有自己的事件類型（agent events、task events、crew events、knowledge events、memory events），事件層級結構如果不夠清晰，會變成維護負擔。

**替代方案**:
- 直接用 callback function — 簡單但會造成強耦合，擴充需要改 core
- OpenTelemetry 專用 tracing — 只解決 observability，不解決 state management
- Observer pattern (class-level) — 比 event bus 更結構化，但靈活性較低

---

## Pattern 4: RuntimeState Checkpoint + Event Replay

**是什麼**: 透過序列化執行期間的 RuntimeState（含 event record、task outputs、agent messages），實現 crew 的中斷續跑。Crew 可以從 checkpoint 還原、跳過已完成 task、從中斷點繼續執行，甚至 fork 出不同的執行分支。

**為什麼有效**: Agent 任務通常很耗時（多個 LLM 呼叫、tool 執行），而 LLM provider 可能會 timeout、token 限制可能被觸發、tool 可能失敗。如果每次失敗都要從頭開始，對使用者體驗是災難性的。Checkpoint 機制讓長時間執行的 crew 有了容錯能力。

**程式碼位置**: [`state/checkpoint_config.py`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/state/checkpoint_config.py)、[`state/runtime.py`](https://github.com/crewAIInc/crewai/blob/ee70702/lib/crewai/src/crewai/state/runtime.py)、[`state/event_record.py`](https://github.com/crewAIInc/crewai/blob/ee70702/lib/crewai/src/crewai/state/event_record.py)

**何時可以借用**: 任何需要長時間執行的 agent 工作流。如果你的 agent 平均執行時間超過 30 秒或涉及多個外部 API 呼叫，checkpoint 就值得考慮。

**注意事項**:
- 序列化 agent message list 可能很大（特別是包含長 context 的 ReAct loop），會影響 checkpoint 的寫入/讀取效能
- Tool 的 side effect 無法回滾——如果一個 tool 已經寫入了資料庫，checkpoint restore 不會自動回復那個狀態
- Checkpoint fork 功能雖然強大，但目前的使用場景還不明確

**替代方案**:
- 不做 checkpoint，每次失敗重跑——適合快速、低成本的 agent 任務
- 只保存 task outputs（不保存 agent message）——體積小但只能重跑完整 task，無法在 agent loop 中間恢復
- 外部 DB 儲存狀態（如 LangGraph 的 postgres checkpoint）——提供持久化但增加了依賴

---

## Pattern 5: Pydantic Circular Reference Resolution via model_rebuild()

**是什麼**: CrewAI 使用 Pydantic v2 的 `model_rebuild()` 機制來解決元件之間的雙向引用問題。`Agent` 引用 `Crew`，`Crew` 引用 `Agent`，`Task` 引用 `Agent` 和 `Crew`——這些在靜態定義時會造成 forward reference 問題。CrewAI 在 `__init__.py` 的模組載入後半段，統一對所有關鍵 model 呼叫 `model_rebuild(force=True, _types_namespace=...)`，傳入完整的 namespace 來解析所有 forward reference。

**為什麼有效**: Agent 框架天生就充滿了雙向引用：Agent 需要知道它屬於哪個 Crew，Crew 需要知道它有哪些 Agent。Pydantic v2 的 `model_rebuild()` 允許在 schema 定義完成後再次解析 type annotations，這是目前最乾淨的解法。

**程式碼位置**: [`__init__.py:128-176`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/__init__.py#L128-L176) — 核心 rebuild 邏輯；每個類別定義中的 `TYPE_CHECKING` block 和 `Annotated` forward reference

**何時可以借用**: 任何使用 Pydantic v2 且有雙向/循環引用的大型專案。特別適合 framework 類專案（元件之間必然有複雜的引用關係）。

**注意事項**:
- `model_rebuild()` 需要小心順序：先定義所有 model，再統一 rebuild。若順序出錯會造成 `PydanticUserError`。
- `_types_namespace` 必須包含所有會被 forward reference 的類型，遺漏任何一個都會導致重建失敗。CrewAI 用了兩層 namespace（`_base_namespace` 和 `_full_namespace`）來處理這個問題。
- CrewAI 的實作中有一些脆弱的 `try/except PydanticUserError`（L173-176），表示某些 rebuild 路徑在特定情境下會失敗——這是一個危險信號。

**替代方案**:
- 完全不使用 forward reference，改用 `PrivateAttr` 或 `Optional` 來弱化引用——但會喪失類型安全
- 使用 `typing.TYPE_CHECKING` + 字串 annotation（`"Crew"` 而非 `Crew`）——Pydantic v2 原生支援，但需要每個 annotation 都手動改
- 把雙向引用拆成 unidirectional：例如 Agent 持有一個 crew_id 字串而非 Crew 物件——需要額外的 lookup

---

## Agent 設計的哲學觀察

從程式碼中可以看出 CrewAI 團隊對 agent 設計的一些根本假設：

1. **宣告式優於程式化** — 團隊偏好讓使用者透過設定（Agent.role、Task.description 等）來定義行為，而不是寫程式碼控制流程。這讓 CrewAI 的 API 看起來乾淨且容易上手。

2. **Agent 應該被約束而非被信任** — max_iter 限制、RPM controller、security fingerprint、guardrail 機制——CrewAI 對 agent 的行為有許多預設的限制，而不是讓 agent 自由發揮直到出錯。

3. **事件驅動是內建需求而非外加功能** — Event bus 不是後來加上去的 tracing 工具，而是從架構底層就存在的核心機制。這從 checkpoint 依賴 event record 的設計可以看出來。

4. **商業化意圖明顯但克制** — Telemetry（`share_crew` 參數）、CrewAI AMP Suite、Crew Control Plane 等商業整合存在於核心程式碼中（`plus_api.py`、`telemetry/`），但沒有干擾核心 agent loop 的設計。

## 跟其他 agent 框架比較

| 面向 | CrewAI | LangGraph | AutoGen |
|---|---|---|---|
| 核心抽象 | Crew + Agent + Task | StateGraph + Node + Edge | Agent + GroupChat + Message |
| 宣告程度 | 高度宣告式 | 中度（圖結構宣告，節點邏輯程式化） | 低度（大量程式化） |
| Checkpoint | ✅ v1.14+ 完整支援 | ✅ 內建（postgres/SQLite） | ❌ 無 |
| 學習曲線 | 低 | 中 | 中高 |
| LLM 綁定 | litellm（鬆散） | LangChain（緊密） | 自建（鬆散） |
| 工具生態 | 自建 + MCP | LangChain 生態 | 自建 |
| 架構複雜度 | 中（monorepo 6 packages） | 高（LangChain 整體架構） | 中 |
