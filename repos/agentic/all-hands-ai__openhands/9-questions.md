---
repo: All-Hands-AI/OpenHands
file: 9-questions
---

# OpenHands · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼選擇 namespace package 而非 monorepo 多 packages 命名？**
  - 4 個套件都共用 `openhands.xxx` 的頂層命名，但 PyPI 上實際套件名是 `openhands-sdk`、`openhands-tools`、`openhands-agent-server`、`openhands-ai`。開發者在 import 時用的是 `openhands.sdk`，安裝時卻要找 `openhands-sdk`。這對新開發者來說是無謂的認知負擔。
  - 相關程式碼: [`openhands/__init__.py:4`](https://github.com/All-Hands-AI/OpenHands/blob/7ea2aed/openhands/__init__.py#L4)

- [ ] **sdk、tools、agent-server 之間是否有精確的版本兼容矩陣？**
  - `openhands-sdk` 的版本與 `openhands-tools` / `openhands-agent-server` 的版本之間沒有 pyproject.toml 中的強約束。安裝時只能靠「同一天發布」來保證相容。
  - 相關程式碼: [`pyproject.toml:60-63`](https://github.com/All-Hands-AI/OpenHands/blob/7ea2aed/pyproject.toml#L60-L63)（`openhands-agent-server==1.23.1` 等固定版本）

- [ ] **EventLog 在大量 events（> 100k）時的效能表現？**
  - EventLog 是 file-backed Sequence，iteration 是 O(n)。當單一 conversation 累積到數萬 events 時，`_run_agent` 中遍歷 events 做 init_state 檢查（[`agent.py:374`](https://github.com/All-Hands-AI/OpenHands/blob/7ea2aed/openhands-sdk/openhands/sdk/agent/agent.py#L374)）的每次 O(n) 掃描是否會成為瓶頸？Condensation（對話壓縮）的觸發時機是什麼？
  - [UNVERIFIED]

- [ ] **Process Sandbox 的實際架構？**
  - DockerSandbox 啟動一個完整容器，RemoteSandbox 連到遠端 API，但 ProcessSandbox 的實作是直接在本地跑 agent-server subprocess。在開發環境中，本地 subprocess 與 app server 共享哪些資源？隔離程度到哪？

- [ ] **Hooks 與 Security Analyzer 的互動邊界？**
  - 當一個 tool call 同時被 hook（例如 code review hook）和 security analyzer 審查時，兩者的優先級誰高？Blocked action 的邏輯是 hook 先檢查還是 security analyzer 先？
  - 相關程式碼: [`agent.py:174-176`](https://github.com/All-Hands-AI/OpenHands/blob/7ea2aed/openhands-sdk/openhands/sdk/agent/agent.py#L174-L176)

## 想問維護者的問題

- 為什麼從 OpenDevin 改名 OpenHands？品牌策略上有什麼考量？
- CLI 的 TUI 使用 Textual，這是刻意的選擇嗎？跟 Rich + prompt_toolkit 對比如何？
- EventLog 之所以不用 SQLite 而是 file-based JSON，是為了避免 schema migration 的負擔嗎？

## 下次再看時的待辦

- [ ] 深入研究 `openhands/sdk/event/condenser.py` 的對話壓縮策略
- [ ] 讀懂 `subscription_proxy` 在 `openhands/sdk/llm/auth/` 中的實作 — LLM provider 的 API key proxy 是怎麼設計的
- [ ] 跑跑看 RemoteConversation 模式，觀察 app server ↔ agent server 之間的 HTTP round-trip overhead
- [ ] 對照 AutoGen 的 multi-agent 設計，看 OpenHands 的 subagent delegation 與之的差異
- [ ] 讀 `enterprise/` 中的 RBAC 實作，了解 enterprise-grade 的 agent platform 權限模型
- [ ] 看 `openhands/sdk/plugins/` 的 plugin system — plugin 的加載、sandbox、生命週期管理

## 跨專案對照備忘

- **EventLog 作為唯一 state backend** 跟之前看的 Discord bot 架構（事件溯源）類似 → 候選 pattern
- **Sandbox-isolated execution** 跟 Ollama 的 subprocess 隔離（子 process 而非 library）類似，但隔離層級更高（Docker container vs subprocess）
- **Response classification + dispatch** 在 AutoGen 中也存在（assistant 回覆 → tool call or text），但 OpenHands 的 `REASONING_ONLY` 處理（corrective nudge）是獨特的
- **Parallel tool execution + FinishTool 截斷** — 在其他 agent 框架未見過，需要更多 repo 驗證才能判斷是否為 pattern
