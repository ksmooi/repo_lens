---
repo: run-llama/llama_index
file: README
studied_at: 2026-05-27
commit_sha: f027669
type: rag
language: Python
framework: LlamaIndex
stars: 49.7k
status: active
---

# LlamaIndex · 概覽

LlamaIndex 是當前最成熟的 LLM data framework，提供從文件解析、向量索引、檢索增廣到 agent 編排的完整 pipeline。它不只是一套 RAG 函式庫，而是一個以「資料」為核心的模組化生態系統——核心套件支援 300+ 外部整合，從 LLM providers、向量資料庫到文件 parser 一應俱全。

## 解決什麼問題

LLM 的知識截止於訓練資料，而真實世界的應用需要模型能存取私有、即時、專屬的資料。LlamaIndex 把這個「將資料餵給 LLM」的過程標準化為一套可組合的 pipeline：

- **資料進**: 130+ 檔案格式的文件解析（PDF、DOCX、HTML...），透過 LlamaParse 雲端服務或本地 parser
- **資料建索引**: 支援向量索引、關鍵字索引、知識圖譜、屬性圖等多種索引結構
- **資料查詢**: 從簡單的 retrieve-then-synthesize 到多步驟 sub-question query engine
- **Agent 編排**: 基於 workflow 引擎的 function calling agent、ReAct agent、multi-agent handoff

## 為什麼值得研究

1. **模組化架構的極致實踐**: 核心 (`llama-index-core`) 只有基本的抽象與 pipeline，所有外部相依（LLM、vector store、reader）都以獨立套件形式發布。這種 plugin 機制的設計值得任何要建立 SDK 生態的專案參考。

2. **從 RAG 到 Agent 的演化路徑**: LlamaIndex 最初是純 RAG 框架，後來加入了 `workflow` 事件驅動引擎，再在上面建了 `AgentWorkflow`。這段演化比其他框架（如 LangChain）更清楚地展示了「如何在不破壞向後相容的前提下加入新抽象層」。

3. **300+ 整合套件的套件管理策略**: 維護近百個獨立 PyPI 套件（每個 vector store 一個、每個 LLM provider 一個）是一項工程挑戰。它的發布腳本、版本管理、CI/CD 管道本身就有學習價值。

## 技術定位

| 面向 | LlamaIndex 的選擇 |
|---|---|
| 核心抽象 | Document → Node → Index → Retriever → QueryEngine |
| 編排引擎 | 事件驅動 Workflow（`@step` + `Context`） |
| Agent 風格 | Function calling / ReAct / Multi-agent handoff |
| 插件機制 | 獨立 PyPI 套件（`llama-index-{category}-{name}`） |
| 擴充點 | Reader / LLM / Embedding / VectorStore / Retriever / Tool |

## 健康度信號

- ⭐ Stars: ~49.7k
- 📅 最後 commit: 2026-05-26（每日活躍）
- 👥 主要維護者: LlamaIndex Inc.（全職團隊）
- 🔄 生態規模: 300+ PyPI 套件、100+ LLM providers、80+ vector stores

## 我會在後續筆記中回答的問題

- LlamaIndex 的「三層抽象」（Index → Retriever → QueryEngine）跟 LangChain 的「LCEL」路線有何根本差異？
- `workflow` 引擎的 event-driven 設計如何解決 agent 的 state management 問題？
- 為什麼維護 300+ 獨立套件而不是一個大包？背後的成本與取捨？
