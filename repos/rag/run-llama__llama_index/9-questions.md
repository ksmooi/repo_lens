---
repo: run-llama/llama_index
file: 9-questions
studied_at: 2026-05-27
commit_sha: f027669
---

# LlamaIndex · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 workflow 引擎要獨立成 `llama-index-workflows` 套件，而不是放在 core 內部？**
  - 推測: 讓其他專案（不只是 LlamaIndex）也能使用這個 workflow 引擎。但目前在 PyPI 上 `llama-index-workflows` 的使用者數量遠少於 `llama-index-core`，實質上仍然是 LlamaIndex 的內部相依。
  - 相關程式碼: [`llama-index-core/pyproject.toml:83`](https://github.com/run-llama/llama_index/blob/f027669/llama-index-core/pyproject.toml#L83)

- [ ] **`BaseWorkflowAgent` 的多重繼承（`Workflow`, `BaseModel`, `PromptMixin`）的 metaclass 衝突是如何解決的？**
  - 推測: 使用了自訂的 `BaseWorkflowAgentMeta` 合併 `WorkflowMeta` 和 `ModelMetaclass`。這個 metaclass 合併的方式值得深究，因為 Python 的 metaclass 衝突通常是開發者最大的痛點。
  - 相關程式碼: [`agent/workflow/base_agent.py:82-87`](https://github.com/run-llama/llama_index/blob/f027669/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L82-L87)

- [ ] **[UNVERIFIED] LlamaIndex 的 AgentWorkflow 和 LangGraph 的核心差異到底是什麼？**
  - 我的推測: 兩者都在解決「non-linear agent 控制流」的問題。LangGraph 用 `StateGraph` + 節點邊的圖形模型，LlamaIndex 用 event-driven `@step` 模型。從語意上來說，graph 模型更適合「固定結構的流程」，event model 更適合「動態分派」。但缺乏實際的比較基準。
  - 相關程式碼: [`agent/workflow/multi_agent_workflow.py:98`](https://github.com/run-llama/llama_index/blob/f027669/llama-index-core/llama_index/core/agent/workflow/multi_agent_workflow.py#L98)

- [ ] **300+ 整合套件的發布流程是怎麼確保一致性的？**
  - 觀察: [`scripts/publish_packages.sh`](https://github.com/run-llama/llama_index/blob/f027669/scripts/publish_packages.sh) 是一個 shell 腳本，對每個套件逐一執行建置和發布。但這個腳本在 CI 中跑一次要多久？如果某個套件發布失敗，是整個 pipeline 重跑還是只重跑失敗的套件？
  - 推測: [UNVERIFIED] 很可能用了某種 lock file 或 CI cache 來加速。腳本中未見明顯的平行化處理，發布 300+ 套件應該需要一段時間。

- [ ] **`SimpleVectorStore` 以 JSON 持久化時，大規模資料的效能瓶頸在哪？**
  - 推測: 每次查詢都需要載入全部 embedding 到記憶體中進行 brute-force 相似度計算。這在文件級別（<10k nodes）還可以，到了 chunk 級別（>100k nodes）就會變得很慢。這是為什麼大部分使用者在生產環境會切換到 Qdrant/Pinecone 等專用 vector store。

## 想問維護者的問題

- LlamaIndex 的 roadmap 上，`workflow` 引擎和 `agent` 系統這兩個方向哪個是主軸？文件解析（LlamaParse）正在朝向商業雲端服務走，開源框架的定位會不會越來越偏向 agent 編排？
- 為什麼選擇維持 300+ 獨立套件的策略，而不是像 LangChain 那樣將不常用的整合放在 community 套件中？套件數量的管理成本是否值得？

## 下次再看時的待辦

- [ ] 深入追蹤 `AgentWorkflow` 的 handoff 機制——`handoff()` 函數如何將控制權從一個 agent 轉移到另一個。特別留意 `Context.store` 在 handoff 中的角色。
- [ ] 跑一個生產級別的 benchmark: 10k 文件的 ingestion 速度、query latency p50/p99、以及不同 `ResponseMode` 的 latency 差異。
- [ ] 對照 `AgentWorkflow` 和 LangGraph 的 `StateGraph`，用同一個 multi-agent 場景（如「文檔 QA + web search + summarization」）在兩個框架上實作，比較開發體驗和 runtime 行為。
- [ ] 研究 `llama-index-instrumentation` 的 `DispatcherSpanMixin` 實作，理解它跟 OpenTelemetry 的整合程度。

## 跨專案對照備忘

- **Plugin 介面隔離（Pattern 1）**: LangChain 也有類似做法（`langchain-openai`、`langchain-community`），但 LangChain 的 community 套件允許外部貢獻者提交新元件，品質控制不如 LlamaIndex 的 monorepo 管理嚴格。→ 值得追蹤是否在其他 SDK 專案（如 Vercel AI SDK、ModelContextProtocol）中觀察到類似的策略，形成 pattern。

- **Event-driven agent runtime（Pattern 2）**: 類似設計在 `crewai` 的 Process 類型和 `autogen` 的 AssistantAgent 中都有雛形，但完整程度不如 LlamaIndex 的 workflow 引擎。→ 候選 pattern tracking：event-driven 或 step-based 的 agent runtime 正在成為新興做法。

- **Singleton Settings 模式**: LlamaIndex 使用 `Settings.llm`、`Settings.embed_model` 等全域設定物件。這在 vLLM（`AsyncLLMEngine`）、LangChain（`RunnableConfig`）等專案中也有類似設計，但各有取捨。→ 候選 pattern：集中式設定管理器在 ML/SDK 框架中的常見實踐。
