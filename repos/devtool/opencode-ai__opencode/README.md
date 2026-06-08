---
repo: opencode-ai/opencode
file: README
studied_at: 2026-05-31
commit_sha: 73ee493
type: devtool
language: Go
framework: Bubble Tea / Cobra
stars: 12.8k
status: archived
---

# OpenCode · 概覽

## 解決什麼問題

OpenCode 是一個純 CLI 的 AI coding agent，直接在終端機中提供智慧輔助——程式碼生成、除錯、檔案操作、命令執行等。它不需要離開 terminal 就能完成大多數開發任務，類似 Claude Code、Codex CLI 等工具的定位。

## 為什麼值得研究

OpenCode 是少數**用 Go 寫的 AI coding agent**。這讓它跟 Python 生態的同類工具（OpenHands、Codex CLI）有根本不同的技術取捨。值得注意的點：

- **Go 的單一二進位部署**——不會有 Python 環境、套件依賴問題，`go install` 或一個 curl 腳本就裝好
- **Bubble Tea TUI**——charmbracelet 生態的 TUI 框架，終端 UI 互動流暢
- **Provider 抽象層用 Go generics**——用泛型 Type Parameter 實現型別安全的 Provider 切換，在 Go 1.18+ 很好的實踐
- **9 個 LLM Provider**——支援 Anthropic、OpenAI、Gemini、Copilot、Groq、Bedrock、Azure、OpenRouter、XAI、Local 等，是同類工具中 provider 覆蓋最廣的之一
- **專案已封存**——origin 作者後續在 Charm 團隊接手開發了 Crush，所以封存前的架構反而是一個穩定的學習標的

## API 風格一句話

Cobra CLI entry + Bubble Tea TUI Model 驅動的 event-driven application

## 技術棧一句話

`Go` + `Bubble Tea` (TUI) + `Cobra` (CLI) + `SQLite/sqlc` (persistence) + `Go generics` (provider abstraction)

## 健康度訊號

- ⭐ Stars: ~12.8k
- 📅 最後 commit: 2025-09-17
- 📦 最新版本: 0.x (early development)
- 💀 狀態: **archived**（已由原作者 Charm 團隊接手成 [Crush](https://github.com/charmbracelet/crush)）

## 競品比較

| 面向 | OpenCode | Claude Code | Codex CLI | OpenHands |
|------|----------|-------------|-----------|-----------|
| 語言 | Go | TypeScript | Python | Python |
| TUI | Bubble Tea | Full TUI | Full TUI | Web UI |
| Provider 數量 | 9+ | Anthropic only | OpenAI only | 6+ |
| 部署方式 | 單一二進位 | npm install | pip install | Docker |
| 主要抽象 | Agent + Tool 迴圈 | Agent + Tool 迴圈 | Agent + Tool 迴圈 | Agent + Tool + Sandbox |
| 是否封存 | 是 | 否 | 否 | 否 |

## 我會在後續筆記中回答的問題

- 為什麼 Go 是 AI coding agent 一個合理但少見的選擇？
- Provider 抽象層的 Go generics 設計如何做到型別安全且易於擴充？
- Agent 迴圈在 Go 的 goroutine + channel 模型下如何實作流式串流？
- 權限系統（Permission Service）的設計如何在 CLI 環境做互動式安全管控？
- Auto-compact 機制如何解決 context window 限制？
