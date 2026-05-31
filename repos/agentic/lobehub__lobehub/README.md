---
repo: lobehub/lobehub
file: README
type: agentic
studied_at: 2026-05-27
commit_sha: bcc31ca
language: TypeScript
framework: 自製 Agent Runtime + Context Engine
agent_style: multi-agent (CAO orchestration)
stars: 77.8k
status: active
---

# LobeHub · 概覽

一個從「ChatGPT 第三方客戶端」進化而來的**完整 Agentic AI 平台**。v2.0 之後的核心定位是 **Chief Agent Operator (CAO)**——不只是讓你跟 LLM 對話，而是讓你「雇用、排程、監控一支 AI agent 團隊」來執行 7×24 的自主運作。

## 解決什麼問題

LobeHub 解決的是「從單一 LLM 對話到多 agent 自主團隊」這個躍進中的基礎設施問題：

- **Agent 生命週期管理**：建立、設定、更新、刪除 agent，跟 DevOps 管 container 一樣
- **跨平台 agent 部署**：同一個 agent 可以部署到 Discord、WeChat、Feishu、QQ、iMessage、LINE、Telegram
- **heterogeneous agent 整合**：不只跑 LobeHub 自己的 agent，還能雇用 Claude Code、Codex 等外部 CLI agent 協同工作
- **MCP 工具生態**：25+ 個內建 tool/skill/knowledge-base 套件，完整 MCP 協定支援
- **人機協作護欄**：五層 tool 權限控制（security blacklist → headless → allow-list → manual → per-tool dynamic）
- **多 agent 群組編排**：Supervisor + Executor 模式，支援群組對話與 Agent Council 協作

## 為什麼值得研究

1. **從產品轉型為平台的架構演化**。LobeHub 從一個 LLM chat UI 開始（類似 ChatGPT 介面），逐步長出 agent runtime、tool system、multi-agent orchestation、chat adapters、heterogeneous agents。這種「由產品需求驅動的架構演進」路徑比其他從零設計的 agent 框架更有參考價值。

2. **CAO 模型是 agent 平台的一個新風格**。有別於 LangGraph 的 graph-based 或 AutoGen 的 conversation-based，LobeHub 採用「雇用 + 排程 + 監控」的組織隱喻，把 agent 視為可被動態建立/管理的員工，而非硬編碼的 workflow 節點。

3. **Context Engine pipeline 設計**。30+ 個 context processor/provider 以 pipeline 方式組裝最終 prompt，每個 processor 負責一個切面（system role / tool list / memory / knowledge / date / platform context 等）。這種設計能讓你在不改 agent loop 的情況下任意組合「這個 session 要帶哪些 context」。

4. **人機協作的安全模型**。InterventionChecker 實作了五層的 tool 執行權限檢查（global blacklist → headless mode → allow-list → manual → per-tool dynamic），這在開源 agent 框架中屬於最完整的實作之一。

## Agent 系統定位

| 面向 | 選擇 |
|---|---|
| Agent 風格 | Plan-Execute loop（指令驅動，非 graph-based） |
| 數量 | single-agent + multi-agent group orchestration + heterogeneous agents |
| 編排方式 | Supervisor+Executor 模式（有限狀態機風格的決策迴圈） |
| Memory | session-based（對話內）+ 長期用戶記憶（UserMemoryInjector） |
| Tool calling | LLM native function calling + MCP 協定 |
| Context 組裝 | Pipeline-based Context Engine（30+ processors） |

## 技術棧一句話

`TypeScript/Next.js 16 + React 19 + Zustand + Drizzle ORM + PostgreSQL + React Router (SPA)`

## 健康度信號

- ⭐ Stars: ~77,750
- 📅 最後 commit: 2026-05-27（極活躍）
- 👥 主要維護者: LobeHub 團隊（<github.com/lobehub>）
- 🔄 commit 頻率: 每日多次
- 🏗️ 專案規模: 11,154 個檔案、6,088 個 .ts + 2,783 個 .tsx

## 我會在後續筆記中回答的問題

- Agent Runtime 的 Plan→Execute loop 怎麼轉動的？
- Context Engine 的 pipeline-based 架構跟傳統 prompt 拼接差在哪？
- 人機協作的安全模型怎麼實作？
- Tool 系統如何做到可插拔？
- Multi-agent group orchestration 是怎麼編排的？

## 比較

| 面向 | LobeHub | LangGraph | AutoGen |
|---|---|---|---|
| 核心隱喻 | CAO（雇用/排程/監控） | Graph（狀態機/邊） | Conversation（對話者） |
| Agent 定義 | Config + Plugin 動態組合 | Node + Edge 靜態定義 | Agent class + 手動 coding |
| 部署型態 | Web app + Electron + Chat adapters | Library（無內建部署） | Library + 範例 script |
| Tool 系統 | 25+ MCP-based built-in tools | 自訂 tool function | 自訂 tool function |
| 人機協作 | 五層 intervention 系統 | 無內建 | 無內建 |
| 跨平台通訊 | 6 種 chat adapters | 無 | 無 |
