---
repo: crewAIInc/crewAI
file: 9-questions
studied_at: 2026-06-03
commit_sha: ee70702
---

# CrewAI · 未解問題

## 還沒搞懂的設計決策

- [ ] **CrewAgentExecutor deprecated 但仍在大量使用**
  - 問題: `CrewAgentExecutor` 已在 L142-148 標記為 deprecated，推薦改用 `experimental/AgentExecutor`。但兩者之間有什麼具體差異？為什麼不直接移除？新舊 executor 在實際行為上有哪些 breaking changes？
  - 我目前的推測: [UNVERIFIED] 舊 executor 需要保留以維持向後相容性，新的 `AgentExecutor` 可能在處理複雜場景（hierarchical delegation、human-in-the-loop）時有不同的控管邏輯
  - 相關程式碼: [`crew_agent_executor.py:95-148`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/agents/crew_agent_executor.py#L95-L148)

- [ ] **Flow DSL 與 Crew 系統的邊界在哪裡？**
  - 問題: Flow 系統有自己的 `@start`、`@listen` 裝飾器和事件驅動模型，而 Crew 系統有自己的一組抽象。使用者到底何時該用哪個？文件說 Flow 是「production architecture」，那 Crew 是什麼——prototype-only 嗎？
  - 我目前的推測: [UNVERIFIED] Flow 是較新的、進化版的抽象層，Crew 的存在是為了維持 v1 使用者的相容性。長期來看 Crew 可能會被 Flow 取代或成為 Flow 內的一個 component。
  - 相關程式碼: [`flow/flow.py`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/flow/flow.py)、[`crew.py:159-392`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/crew.py#L159-L392)

- [ ] **`crewai-core` 與 `crewai` 的關係**
  - 問題: `lib/crewai-core/` 是一個獨立套件，包含 `PrinterColor`、`PRINTER` 等基礎設施類別。但它跟 `lib/crewai/` 的邊界是什麼？為什麼不直接把這些放在 `crewai.utilities` 中？
  - 我目前的推測: [UNVERIFIED] crewai-core 可能是一個抽象的「framework core」層，讓未被納入 monorepo 的其他套件（或外部 plugin）可以依賴最小的核心而不需要拉整個 crewai。
  - 相關程式碼: [lib/crewai-core/](https://github.com/crewAIInc/crewAI/tree/ee70702/lib/crewai-core/)

- [ ] **event bus 在全域單例模式下的測試策略**
  - 問題: `crewai_event_bus` 是一個模組層級的全域變數（在 `events/event_bus.py` 中直接實例化）。在多個測試中共享這個 bus 不會造成測試間的干擾嗎？團隊的解決方案是什麼？（測試用 vcr.py 錄製 LLM 回放，但這跟 event bus 無關。）
  - 我目前的推測: [UNVERIFIED] 可能用 pytest fixture 在每次測試前重置 bus，或者測試框架層級有 `event_bus.reset()` 之類的機制。但我沒在程式碼中找到明顯的測試用 bus 重置邏輯。
  - 相關程式碼: [`events/event_bus.py`](https://github.com/crewAIInc/crewAI/blob/ee70702/lib/crewai/src/crewai/events/event_bus.py)

- [ ] **團隊對 `@` vs `__` 的命名策略不一致**
  - 問題: 在同一個 repo 中，同一個概念有時候用 `crewai_files`（import 名稱）、有時候用 `crewai-files`（PEP 503 package 名稱）、有時候用 `FilesHandler` 的命名。這個不一致在跨套件引用時造成了混淆（例如 `from crewai_files import FileInput` 需要從 pip package `crewai-files` 安裝）。為何不統一？
  - 我目前的推測: [UNVERIFIED] 這是 monorepo 轉移過程中的歷史遺留，uv workspace 的 package 名（`crewai-files`）跟 Python import 名（`crewai_files`）之間的映射是 PEP 503 的慣例，但對開發者來說增加了心智負擔。
  - 相關程式碼: [`pyproject.toml:175-234`](https://github.com/crewAIInc/crewAI/blob/ee70702/pyproject.toml#L175-L234) — uv workspace members 清單

## 想問維護者的問題

- CrewAI 的開發團隊規模多大？目前有哪些全職貢獻者？
- Flow 系統的 roadmap 是什麼—它會完全取代 Crew 嗎？還是兩者會長期並存？
- 為什麼選擇 litellm 而非直接支援每個 provider 的 SDK？litellm 的抽象層有沒有造成過限制或 bug？
- Checkpoint 在生產環境的使用情況如何？有沒有已知的序列化效能瓶頸？

## 下次再看時的待辦

- [ ] 深入研究 `experimental/agent_executor.py`，比較它跟 `CrewAgentExecutor` 的具體差異
- [ ] 跑一個 hierarchical process 的範例，觀察 manager agent 如何透過 `AgentTools` 委派任務
- [ ] 對照 AutoGen 的 GroupChat 模型，比較兩者在 multi-agent 訊息傳遞上的設計哲學差異
- [ ] 實際測試 checkpoint → fork → resume 的完整流程，確認 fork 後的行為是否如預期
- [ ] 研究 `lite_agent.py` 與 `lite_agent_output.py` — 這是輕量版 agent？跟一般 Agent 的差異？

## 跨專案對照備忘

- **Pydantic circular reference resolution via model_rebuild()** — 跟 Hermes Agent 專案處理 Pydantic 循環引用的方式類似。CrewAI 的 `__init__.py:128-176` 是 monorepo 層級的統一 rebuild，比分散在各個 model 檔案中個別 rebuild 更乾淨。→ **候選 pattern**（已在 2 個專案觀察到）
