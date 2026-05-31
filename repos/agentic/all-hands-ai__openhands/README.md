---
repo: All-Hands-AI/OpenHands
type: agentic
studied_at: 2026-05-27
commit_sha: 7ea2aed
language: Python
framework: openhands-sdk (自製 SDK)
agent_style: single-agent (可擴展至 multi-agent)
stars: 75k
status: active
---

# OpenHands · 概覽

OpenHands 是一個 AI-driven development 平台，前身為 OpenDevin。不同於多數 agent 框架只做「控制流抽象」，OpenHands 涵蓋從 SDK、CLI、Web UI 到 Enterprise 的完整堆疊，讓開發者可以用多種介面與 coding agent 互動。

## 解決什麼問題

OpenHands 不只是一套 agent 函式庫，而是一個**完整運行的 agent 平台**：

- **SDK** — 提供 Agent、Conversation、Tool 等核心抽象，開發者可以在 Python code 中定義 agent、註冊工具、在 Workspace 中執行
- **CLI** — 終端機 TUI，類似 Claude Code / Codex 的體驗，但支援多種 LLM provider
- **Web UI / App Server** — 瀏覽器中的 agent 互動介面，含 conversation 管理、sandbox 編排
- **Enterprise** — 整合 GitHub/GitLab/Jira/Slack，支援 RBAC、webhooks、多用戶

## 為什麼值得研究

1. **Namespace package 架構罕見** — 用 `pkgutil.extend_path` 把 `openhands` 這個頂層 package 拆到 4 個獨立 pip 套件（sdk、tools、agent-server、app-server），在 Python 生態中是少見的模組化策略
2. **沙箱執行的深層設計** — agent 不在主機執行，而是啟動完整 Docker 容器跑 agent-server，透過 HTTP API 互動，這跟 LangGraph / AutoGen 的 in-process agent 完全不同
3. **事件系統作為基石** — 所有 agent 行為（LLM 呼叫、tool 執行結果、state 變化）都記為 Event，支援 SSE streaming、持久化、搜尋

## Agent 系統定位

| 面向 | 選擇 |
|---|---|
| Agent 風格 | ReAct + iterative refinement |
| 數量 | 預設 single-agent，支援 subagent delegation |
| 編排方式 | LLM-decided（agent 自行選擇工具與順序） |
| Memory | file-based EventLog（可序列化、搜尋、condensation） |
| Tool calling | OpenAI Responses API / LiteLLM native function calling |

## 技術棧一句話

`Python 3.12+` + `openhands-sdk`（自製） + `LiteLLM`（統一 LLM provider 介面） + `FastAPI`（app server）+ `Docker`（sandbox）+ `React`（frontend）

## 健康度信號

- ⭐ Stars: ~75k
- 📅 最後 commit: 2026-05-27（每日活躍）
- 👥 主要維護者: OpenHands 社群（All-Hands.dev）
- 🔄 commit 頻率: 每日 10-30 commits（2026/05）
- 🏆 SWE-bench: 77.6%（據 README badge）

## 與主要競品比較

| 面向 | OpenHands | Claude Code / Codex | LangGraph / AutoGen |
|---|---|---|---|
| 主要抽象 | Agent + Conversation + Tool | CLI-only | Graph / Multi-agent |
| 部署方式 | Docker sandbox 隔離 | 本地 process | 一般 import |
| 執行環境 | 容器內 agent-server | 直接執行 shell | Python process |
| SDK 封裝程度 | 完整 Python SDK | 無 SDK | Python library |
| 雲端支援 | 有（SaaS 與 Enterprise） | 無 | 框架層無 |
| Enterprise 功能 | 整合 Slack/Jira/RBAC | 無 | 無 |

## 我會在後續筆記中回答的問題

- 為什麼選擇 namespace package 而非單一套件？
- Docker sandbox 架構如何管理 agent-server 的生命週期？
- Event 系統如何支援 SSE streaming、持久化、搜尋？
- Tool 系統的註冊與 dispatch 機制如何設計？
- OpenHands 的各層（SDK / CLI / Web / Enterprise）是如何分工的？
