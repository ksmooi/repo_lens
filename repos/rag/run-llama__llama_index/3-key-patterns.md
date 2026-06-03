---
repo: run-llama/llama_index
file: 3-key-patterns
studied_at: 2026-05-27
commit_sha: f027669
---

# LlamaIndex · 值得偷學的設計

## Pattern 1: Plugin 生態的介面隔離策略

**是什麼**: LlamaIndex 將所有外部相依（LLM、vector store、reader 等）定義為核心層的抽象介面，然後以獨立 PyPI 套件的方式實作這些介面。每個整合套件在 `llama-index-integrations/` 下有自己的目錄，有自己的 `pyproject.toml`、版本號、和 CI。

**為什麼有效**: 這種設計解決了 SDK 生態中最常見的問題——依賴衝突。假設你同時用 OpenAI (需 `openai>=1.0`) 和 Anthropic (需 `anthropic>=0.30`)，如果它們都在一個大包裡，版本管理會是一場惡夢。分離後，使用者只安裝需要的套件，每個套件聲明自己的相依，`pip` 的 resolver 只需要解決小型依賴樹。

**程式碼位置**:
- 核心介面定義: [`llama-index-core/llama_index/core/llms/llm.py`](https://github.com/run-llama/llama_index/blob/f027669/llama-index-core/llama_index/core/llms/llm.py)（以 LLM 為例）
- 整合套件實作: [`llama-index-integrations/llms/llama-index-llms-openai/`](https://github.com/run-llama/llama_index/blob/f027669/llama-index-integrations/llms/llama-index-llms-openai/)（以 OpenAI 為例）
- 發布腳本: [`scripts/publish_packages.sh`](https://github.com/run-llama/llama_index/blob/f027669/scripts/publish_packages.sh)

**何時可以借用**: 當你的 SDK/framework 需要支援多個外部服務提供者（LLM、vector store、embedding 等）時。

**注意事項**:
- monorepo 工具需要選對 — LlamaIndex 用簡單的 shell 腳本管理套件發布，這在套件數量成長到 300+ 後開始吃力。如果你的目標是 50+ 套件，考慮用 `changesets` 或 `release-please` 等自動化工具。
- 介面需要穩定 — 核心層的 `BaseLLM`、`BaseRetriever` 等抽象一旦發布，修改就是 breaking change。LlamaIndex 使用了 `<3.0` 的版本範圍來保留空間。

---

## Pattern 2: Event-Driven Workflow 作為 Agent Runtime

**是什麼**: LlamaIndex 的 agent 系統不是基於傳統的 ReAct loop 實作，而是建構在一個完整的事件驅動 workflow 引擎之上。Agent 的每一步（接收輸入、呼叫 LLM、執行工具、產生輸出）都是一個 `@step` 裝飾的事件處理器，透過 `Context` 物件傳遞資料。

```python
# agent/workflow/base_agent.py — BaseWorkflowAgent
class BaseWorkflowAgent(Workflow, BaseModel, PromptMixin):
    name: str
    system_prompt: Optional[str]
    tools: List[AsyncBaseTool]
```

每個 agent 本身是一個 workflow，執行時的事件流是:
```
StartEvent → AgentInput → [LLM step] → [Tool step 循環] → StopEvent
```

**為什麼有效**: 這種設計帶來三個優勢：
1. **可觀測性**: 每個 `@step` 自動被 workflow 引擎追蹤，可以掛載 `Context.write_event_to_stream()` 做 streaming
2. **可組合性**: 多個 agent 可以透過 `AgentWorkflow` 組合，agent 之間的 handoff 只是 workflow 的事件路由
3. **可擴充性**: 使用者在步驟之間插入自訂 step（例如 logging、validation、human-in-the-loop）而不需要 fork agent 程式碼

**程式碼位置**:
- Workflow 引擎: [`llama-index-workflows`](https://github.com/run-llama/llama_index/blob/f027669/llama-index-core/llama_index/core/workflow/)（外部套件，核心內轉發）
- Agent 基底: [`agent/workflow/base_agent.py:87`](https://github.com/run-llama/llama_index/blob/f027669/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L87)
- Multi-agent handoff: [`agent/workflow/multi_agent_workflow.py:98`](https://github.com/run-llama/llama_index/blob/f027669/llama-index-core/llama_index/core/agent/workflow/multi_agent_workflow.py#L98)

**何時可以借用**: 當你需要一個可插拔的 agent runtime，且預期未來需要支援 streaming、human-in-the-loop、或 multi-agent 協作時。

**注意事項**:
- 事件驅動模型的除錯比序列程式碼困難——你無法在 step 之間下中斷點，因為 step 之間的調度由 workflow 引擎控制
- 學習曲線: 使用者需要理解 `Event`、`Context`、`@step` 的 dispatch 語義

**替代方案**:
- **LangGraph**: 用 graph-based 取代 event-based，節點之間的邊定義控制流。適合需要精確控制 agent 行為路徑的場景。
- **CrewAI**: 用 class-based Agent/Task 模型，更接近傳統程式設計的直覺。但擴充性較差。

---

## Pattern 3: Stateful Index + StorageContext 的持久化策略

**是什麼**: LlamaIndex 的每個 `BaseIndex` 在初始化時會建立 `IndexStruct`（包含 mapping 資訊）並自動寫入 `StorageContext`（包含 docstore、index_store、vector_store）。Query 時不需要重建 index，只需要從持久化儲存載入。

```python
# indices/base.py:76-84
index_struct = self.build_index_from_nodes(nodes)
self._index_struct = index_struct
self._storage_context.index_store.add_index_struct(self._index_struct)
```

**為什麼有效**: 這種設計將「運算結果」視為「可持久化的狀態」——embedding 只需要計算一次，之後可以序列化到磁碟或存在外部 vector store 中反覆查詢。對比 stateless 方案（每次啟動重新 embedding），這節省了大量時間和金錢。

**程式碼位置**:
- `StorageContext`: [`storage/storage_context.py:52`](https://github.com/run-llama/llama_index/blob/f027669/llama-index-core/llama_index/core/storage/storage_context.py#L52)
- `BaseIndex`: [`indices/base.py:25`](https://github.com/run-llama/llama_index/blob/f027669/llama-index-core/llama_index/core/indices/base.py#L25)

**何時可以借用**: 任何需要「一次建構、多次查詢」的資料 pipeline。特別是 embedding/vector search 場景。

**注意事項**:
- Stateful 物件在 serverless 環境不友好——每次冷啟動都需要從儲存載入狀態
- `SimpleDocumentStore` 和 `SimpleIndexStore` 預設用 JSON 序列化，檔案會隨節點數量線性增長。大規模使用需要換成外部資料庫實作

**替代方案**:
- **LangChain VectorStore**: Stateless，不自動管理 docstore。使用者需要自行管理文件與 embedding 的對應關係。
- **Haystack DocumentStore**: 類似 LlamaIndex 的 stateful 設計，但更偏向資料庫抽象而非記憶體中的序列化物件。

---

## Pattern 4: Prompt 系統的 Mixin 架構

**是什麼**: 所有需要自訂 prompt 的元件的都繼承 `PromptMixin`，這個 mixin 提供統一的 prompt 註冊、取得、更新機制。

```
PromptMixin
  ├── BaseQueryEngine
  ├── BaseWorkflowAgent
  ├── BaseSynthesizer
  └── ...
```

每個元件透過 `_get_prompt_modules()` 回傳子模組的 prompt，形成樹狀結構。修改 prompt 時不需要知道元件的內部實作。

**程式碼位置**: [`prompts/mixin.py`](https://github.com/run-llama/llama_index/blob/f027669/llama-index-core/llama_index/core/prompts/mixin.py)

**為什麼有效**: 統一 prompt 管理路徑讓框架層級的 prompt 覆蓋（如 observability、logging）成為可能——你可以在 `PromptMixin` 層攔截所有 prompt 的讀寫，而不需要修改每個元件。

**何時可以借用**: 當你的系統有大量元件各自需要自訂 prompt，且你希望提供統一的 prompt 管理介面時。

---

## Pattern 5: CallbackManager + DispatcherSpanMixin 的觀測性框架

**是什麼**: LlamaIndex 的 instrumentation 分為兩層：
1. `CallbackManager` — 傳統的 callback/listener 模式，用於 tracing 關鍵事件（index_construction、query、llm_call 等）
2. `DispatcherSpanMixin` — 基於 `instrumentation` 模組的 decoration-based tracing，類似 OpenTelemetry 的 span

**程式碼位置**:
- CallbackManager: [`callbacks/base.py`](https://github.com/llama_index/blob/f027669/llama-index-core/llama_index/core/callbacks/base.py)
- DispatcherSpanMixin: 分散在核心類別中，每個 class 透過 `instrument.get_dispatcher(__name__)` 註冊

**為什麼有效**: 雙層設計讓 LlamaIndex 同時支援輕量級的 callback（純同步、低開銷）和重量級的 instrumentation（非同步、rich event payload）。使用者可以根據需求選擇觀測層級，而不必為簡單場景付出完整的 tracing overhead。

**何時可以借用**: 當你的系統需要支援多種觀測性層級（開發階段的全量 tracing 與生產階段的 minimal logging）時。

---

## 跟其他 RAG 框架比較

| 面向 | LlamaIndex | LangChain | Haystack |
|---|---|---|---|
| 核心抽象 | Document → Node → Index → Retriever → QueryEngine | Chain + Runnable + Tool | Pipeline + DocumentStore |
| Index 管理 | Stateful (StorageContext) | Stateless | Stateless (DocumentStore 管理) |
| Agent 系統 | Workflow-based (event-driven) | LangGraph (graph-based) | 有限（Pipeline 層） |
| 插件生態 | 300+ monorepo managed | langchain-community (community maintained) | 中度整合 |
| 部署模式 | 偏 library (在應用程式中 import) | 偏 framework (LCEL chain) | 偏 pipeline (components) |
| 學習曲線 | 中（Index 抽象容易理解） | 高（LCEL + Runnable 組合多） | 低（Pipeline 直覺） |
| 主要 trade-off | Index stateful 簡化使用但 serverless 不利 | Chain 靈活但版本升級易壞 | Pipeline 穩定但擴充性受限 |
