---
repo: crewAIInc/crewAI
type: agentic
studied_at: 2026-06-03
commit_sha: ee70702
language: Python
framework: CrewAI
agent_style: multi-agent (orchestrator+workers)
stars: ~52.7k
status: active
---

# CrewAI · 概覽

CrewAI 是一個完全從頭打造的 Python multi-agent 編排框架，**不依賴 LangChain 或其他 agent 框架**。它讓開發者用宣告式的方式定義一群 AI agent，並以 sequential 或 hierarchical 流程協作完成任務。作者 Joao Moura（@joaomdmoura）在 2024 年初以一己之力開啟了這個專案，至今已累積 52.7k stars，成為 agent 生態系中最有影響力的 Python 框架之一。

## 解決什麼問題

CrewAI 解決的是「**如何讓多個 AI agent 有結構地協作**」這個問題。具體來說：

- 把 agent 定義簡化成角色、目標、背景故事三個維度，降低心智負擔
- 提供兩種開箱即用的協作模式（sequential / hierarchical），不需要自行設計 message passing
- 讓開發者可以專注在「每個 agent 該做什麼」，而不是「agent 們怎麼溝通」
- Flow 系統進一步提供事件驅動的細粒度控制，適合生產級部署

## 為什麼值得研究

1. **純 Python agent 框架的典範** — 不依賴 LangChain，所有設計從零自建。這讓它的程式碼是「agent 框架該怎麼設計」的純粹案例，沒有 legacy 包袱
2. **從宣告式到事件驅動的進化** — Crew 系統提供高層級宣告式編排，Flow 系統提供低層級事件驅動控制，兩者在同一個框架內共存。這種 dual-layer 設計在 agent 框架中少見
3. **生產就緒的 checkpoint 系統** — v1.14 引入的 checkpoint 機制讓長時間執行的 crew 可以中斷續跑，這是其他開源 agent 框架（AutoGen、LangGraph 等）尚未完整支援的能力

## Agent 系統定位

| 面向 | 選擇 |
|---|---|
| Agent 風格 | 混合：Crew 層用 Workflow（sequential/hierarchical），Flow 層用事件驅動 |
| 數量 | Multi-agent |
| 編排方式 | 宣告式（Crew + Task 定義），底層可自訂 Flow |
| Memory | 可選：short-term（對話內）+ long-term（跨對話，支援多種後端） |
| Tool calling | 雙軌：LLM native function calling / ReAct text pattern（fallback）|

## 技術棧一句話

`Python` + `Pydantic` + `litellm` + `OpenTelemetry`

## 健康度信號

| 指標 | 值 |
|---|---|
| ⭐ Stars | ~52.7k |
| 📅 最後 commit | 2026-06-03（非常活躍） |
| 👥 主要維護者 | Joao Moura + core team（crewAI Inc.） |
| 🔄 commit 頻率 | 每日活躍 |
| 🏢 贊助方 | CrewAI Inc.（商業化：CrewAI AMP Suite + Crew Control Plane） |
| 📦 版本 | v1.14.6 |
| 🧪 測試工具 | pytest + asyncio + vcr.py（錄製回放）+ ruff + mypy strict |
| 🔗 Docs | [docs.crewai.com](https://docs.crewai.com) |

## 競品對照

| 面向 | CrewAI | LangGraph | AutoGen (Microsoft) |
|---|---|---|---|
| 核心抽象 | Crew + Agent + Task（宣告式） | StateGraph（圖形化） | Agent + GroupChat（對話式） |
| 協作模式 | sequential / hierarchical + Flow DSL | graph edge traversal | 自訂 conversation pattern |
| 底層依賴 | 自建（無 LangChain） | LangChain | 依賴套件少，自建 |
| Checkpoint | ✅ 完整（v1.14+） | ✅（內建） | ❌ 無 |
| Flow DSL | ✅ 事件驅動 | ❌ 圖形為主 | ❌ |
| 學習曲線 | 低（宣告式設定檔） | 中（需理解 StateGraph） | 中（需理解 conversation pattern） |

## 我會在後續筆記中回答的問題

- CrewAI 的 agent loop 到底怎麼跑？ReAct vs native function calling 的雙軌設計怎麼切換？
- Crew 如何在 sequential / hierarchical 模式下編排任務？manager agent 怎麼工作？
- Flow 系統的事件驅動模型跟 Crew 系統怎麼共存？
- Checkpoint 機制如何做到中斷續跑？序列化哪些狀態？
- Tool 系統的設計：base_tool → structured_tool → tool_calling 的抽象層次？
