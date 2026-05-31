---
repo: All-Hands-AI/OpenHands
file: 3-key-patterns
---

# OpenHands · 值得偷學的設計

## Pattern 1: Namespace Package 作為模組邊界

**是什麼**:
OpenHands 使用 Python 的 namespace package（`pkgutil.extend_path`）將單一 `openhands` 頂層 package 分散到 4 個獨立 pip 套件。主 repo 的 `openhands/__init__.py` 只有一行：

```python
__path__ = __import__('pkgutil').extend_path(__path__, __name__)
```
[`openhands/__init__.py:4`](https://github.com/All-Hands-AI/OpenHands/blob/7ea2aed/openhands/__init__.py#L4)

安裝後，`openhands/` 目錄下會有來自不同套件的 module:
```
openhands/
├── app_server/    # 來自主 repo (openhands-ai)
├── sdk/           # 來自 openhands-sdk
├── tools/         # 來自 openhands-tools
├── agent_server/  # 來自 openhands-agent-server
```

**為什麼有效**:
- **獨立版本化** — 套件可以各自發行版本，`openhands-tools==1.2.0` 可能跟 `openhands-sdk==1.3.0` 不同步
- **輕量化部署** — agent-server 只需要 `openhands-sdk` + `openhands-tools`，不需要 `openhands-ai` 的 web server
- **關注點分離** — SDK 開發者跟 app server 開發者可以在不同 repo 工作

**替代方案**:
多數專案用 `monorepo`（一個 repo 多個 packages，各自獨立命名）或 `single package`（同一個 pip 套件包含一切）。OpenHands 的做法更激進：套件獨立分發但共用頂層命名空間。

| 方案 | 優點 | 缺點 |
|---|---|---|
| Namespace package | 獨立版本、輕量化 import | 命名衝突風險、除錯困難 |
| Monorepo 多 packages | 版本協調容易、CI 統一 | 安裝需要全部依賴 |
| Single package | 最簡單、保證相容 | 無法部分升級 |

**何時可以借用**:
當你的專案有「核心 SDK」和「平台層」兩種明顯不同的消費者，且核心 SDK 需要被第三方專案 import 時。例如：agent SDK + cloud platform、inference library + serving platform。

**注意事項**:
- 任何一個套件加入同名模組（例如 `openhands/foo.py`）會造成難以追蹤的 import 衝突
- 除錯時需要知道某個 `import openhands.xxx` 是從哪個 pip 套件來的
- Tools 如 `mypy` 可能對 namespace package 支援不佳

---

## Pattern 2: Sandbox-Isolated Agent Execution

**是什麼**:
Agent 不在主機上直接執行，而是啟動一個獨立的 sandbox（Docker 容器、子 process、或遠端服務），在 sandbox 內跑 agent-server 來執行 conversation。App Server 與 Agent Server 之間透過 HTTP API 通訊。

```
App Server (FastAPI)
    │ POST /api/v1/sandbox/start → 啟動容器
    │ POST /api/v1/conversation/start → 開始任務
    │ GET /stream → SSE 事件串流
    ▼
Sandbox Container
    └── Agent Server (FastAPI, port 8000)
         ├── Conversation Service
         ├── Event Service
         └── SDK Agent (LocalConversation)
```

**為什麼有效**:
- **完全隔離** — agent 的 shell 指令、檔案修改都限制在容器內，不影響主機
- **清理簡單** — 任務結束後直接刪除容器即可
- **橫向擴展** — 每個 user/session 可以有自己的 sandbox，不互相干擾
- **支援暫停/恢復** — 容器可以被 pause，保留 state

**替代方案**:
多數 agent 框架（LangGraph、AutoGen）使用 in-process 執行，agent 在主程式的 process 內執行 shell，隔離性差但延遲低。OpenHands 選擇了隔離性優先，犧牲了部分 latency（每次 LLM call 需要 HTTP round-trip）。

**何時可以借用**:
- 你的 agent 需要執行任意 shell 指令
- 你要對外提供 agent 服務（SaaS / multi-tenant）
- 你需要精確的 resource isolation

**何時不要用**:
- 單機開發/testing，in-process 更輕量
- 只需要 LLM chain 不需要 shell 執行
- ping-pong latency 敏感的場景

---

## Pattern 3: EventLog 作為唯一的 State Backend

**是什麼**:
不同於大多數 agent 框架用 dict 或 Pydantic model 管理 state，OpenHands 使用 `EventLog` — 一個 file-backed 的 events list。每一個 agent 行為（user message、LLM response、tool call、tool result）都是一個 Event，append-only 地寫入 EventLog。

```python
class EventLog(Sequence[Event]):
    """File-backed event sequence. O(1) len(), O(n) iteration."""
```

**為什麼有效**:
- **天然 audit trail** — 每個動作都有記錄，方便除錯與 replay
- **Lazy materialization** — 長對話（30k+ events）不需要全部載入記憶體
- **搜尋友善** — 可以按 kind / timestamp 過濾 events
- **序列化簡單** — append-only JSON lines

**替代方案**:
| 方案 | 優點 | 缺點 |
|---|---|---|
| EventLog（OpenHands） | 完整歷史、搜尋、audit | 檔案 I/O overhead |
| In-memory list | 最快 | 不持久、無法搜尋 |
| SQLite / PostgreSQL | 結構化查詢 | schema migration、join 複雜度 |

**何時可以借用**:
當你需要 conversation audit 或 debug 時（例如評估 agent 品質）。

**注意事項**:
- 單一 conversation 可以累積大量 events（數萬個），搜尋需要索引
- EventLog 的 iteration 是 O(n)，not designed for random access patterns

---

## Pattern 4: Parallel Tool Execution with Safety Constraints

**是什麼**:
Agent 可以同時發出多個 tool calls（OpenAI Responses API 原生支援），`_ActionBatch` 處理平行執行：

1. 截斷 — 若 FinishTool 與其他 tools 同時出現，只執行到第一個 FinishTool
2. Blocked action 過濾 — 被 hook 或 security analyzer 阻擋的工具不執行
3. 平行 dispatch — `asyncio.gather` 同時執行多個工具

[`openhands/sdk/agent/agent.py:158-230`](https://github.com/All-Hands-AI/OpenHands/blob/7ea2aed/openhands-sdk/openhands/sdk/agent/agent.py#L158-L230)

**為什麼有效**:
- LLM 可以同時發出「grep 搜尋 → 讀取結果檔案 → 檢查 imports」等 chain，節省 round-trip
- 安全檢查在 dispatch 前做，不會因為平行執行繞過安全控制
- FinishTool 優先處理，避免已完成的任務繼續消耗資源

**替代方案**:
| 方案 | 特點 |
|---|---|
| 序列執行 | 簡單、可控，但慢（3 tools = 3 round-trip） |
| 平行執行（本 pattern） | 快、但需處理 race condition |
| 批次+確認 | 先執行再請使用者確認（OpenHands 也支援）|

**何時可以借用**:
當多個工具之間沒有依賴關係時（例如搜尋多個檔案）。

**注意事項**:
- 若 tools 有 side effect（寫檔案、發 HTTP），平行執行可能造成 race
- `FinishTool` 混在平行 tool calls 中是 OpenAI Responses API 的特性，需要像 `_ActionBatch._truncate_at_finish` 這樣的截斷邏輯

---

## Pattern 5: LLM Response Classification + Dispatch

**是什麼**:
Agent 的 `ResponseDispatchMixin` 實現了「分類→分發」模式。LLM response 被 `classify_response` 分為四類，每類對應一個 handler：

[`openhands/sdk/agent/response_dispatch.py:53-77`](https://github.com/All-Hands-AI/OpenHands/blob/7ea2aed/openhands-sdk/openhands/sdk/agent/response_dispatch.py#L53-L77)

```python
match classify_response(message):
    case TOOL_CALLS:     _handle_tool_calls(...)
    case CONTENT:        _handle_content_response(...)
    case REASONING_ONLY: _handle_no_content_response(...)
    case EMPTY:          _handle_no_content_response(...)
```

**為什麼有效**:
- 分類是純函數（pure function），沒有 side effect，易於測試
- Handler 專注於單一職責：tool call 的 dispatch、content 的終止、reasoning-only 的 corrective nudge
- 新增 response 類型只需要加一個 enum value + handler

**替代方案**:
| 方案 | 描述 |
|---|---|
| 條件鏈（本 pattern） | 先分類再 dispatch，清楚但可能分散邏輯 |
| 單一 handler | 在一個 function 內處理所有情況，可能過長 |
| State machine | 用 state + transition 控制，但複雜度高 |

**何時可以借用**:
當你的 LLM response 類型有限（< 10 種）且彼此互斥時。對於多層次巢狀結構的 response，用 state machine 可能更好。

---

## Agent 設計的哲學觀察

OpenHands 的設計透露出的幾個觀點：

1. **Agent 應該被執行在 sandbox 中** — 這是最根本的設計決定。OpenHands 團隊顯然認為 coding agent 的安全風險需要基礎設施層面的隔離，而非僅靠 prompt 防護
2. **Event sourcing 作為唯一事實來源** — 不像 LangGraph 把 state 當作共享資料結構，OpenHands 把 events 當作唯一真相。這讓 replay、debug、audit 變得自然
3. **SDK-first** — 先設計好 SDK（Agent、Conversation、Tool），然後 CLI 和 web 都是 SDK 的使用者。不是反過來
4. **Parallel tool execution 是預設而非選項** — 架構從一開始就支援平行 tool calls，而不是後續補上
5. **Namespace package 是必要的** — 雖然在 Python 社群中少見，但對 OpenHands 的分發策略（SDK 可獨立使用、agent-server 可打包成 binary）是合理的選擇

## 跟其他 agent 框架比較

| 面向 | OpenHands | LangGraph | AutoGen | Claude Code |
|---|---|---|---|---|
| Agent 風格 | ReAct + Iterative | State Graph | Multi-agent Chat | ReAct CLI |
| 隔離性 | Docker sandbox | In-process | In-process | Subprocess |
| 可編程性 | Python SDK | Python + Graph DSL | Python | 無 SDK |
| Event 系統 | Built-in (EventLog) | LangSmith (外部) | 無內建 | Anthropic Console |
| Multi-agent | Subagent delegation | Native graph | 核心功能 | 無 |
| Tool 框架 | 自製 (Action/Observation) | LangChain tool | 自製 | 內建 tools |
| LLM 抽象 | LiteLLM | LangChain ChatModels | 自製 | Anthropic only |
