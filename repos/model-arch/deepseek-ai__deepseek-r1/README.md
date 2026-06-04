---
repo: deepseek-ai/DeepSeek-R1
file: README
studied_at: 2026-05-28
commit_sha: 0cf7856
type: model-arch
language: (paper/model release; base model DeepSeek-V3 uses Python + CUDA)
framework: DeepSeek-V3-Base (MoE Transformer, 671B total / 37B activated)
task: reasoning (RL-based post-training for Chain-of-Thought)
paper: DeepSeek-R1 (https://arxiv.org/abs/2501.12948)
stars: ~92.0k
status: maintenance
---

# DeepSeek-R1 · 概覽

## 解決什麼問題

DeepSeek-R1 是第一個透過**純強化學習（RL）** 來激發 LLM 推理能力、且達到 OpenAI-o1 等級表現的開源方案。它回答了兩個核心問題：

1. **推理能力能否不靠人工標註的監督資料，僅靠 RL 自發湧現？**（→ DeepSeek-R1-Zero）
2. **如何讓 RL 訓練出來的推理模型同時具備可讀性與通用能力？**（→ DeepSeek-R1）

## 為什麼值得研究

- **里程碑意義**：首次驗證「純 RL 即可激發推理能力，無需 SFT 冷啟動」——DeepSeek-R1-Zero 在 AIME 2024 從 15.6% 提升到 71.0%，全部靠 RL 自我演化
- **管線設計**：四階段訓練管線（Cold Start SFT → Reasoning RL → Rejection Sampling SFT → All-Scenario RL）是 post-training 領域的典範設計
- **蒸餾策略**：證明了「從大模型蒸餾推理模式」遠比「在小模型上跑 RL」更有效（32B 蒸餾版超越 32B 自訓練版 25+ 個百分點）
- **失敗記錄**：論文誠實分享了 PRM 與 MCTS 兩條走不通的路，這些教訓比成功經驗有時更有價值

## 任務與資料

| 面向 | 值 |
|------|-----|
| 任務類型 | Reinforcement Learning (GRPO) + Supervised Fine-Tuning |
| 輸入 | 數學 / 程式 / 科學推理提示（含 Chain-of-Thought） |
| 輸出 | `<think>...</think><answer>...</answer>` 格式的回應 |
| 主要訓練資料 | Cold-start data（數千條）+ 800K SFT samples（600K reasoning + 200K general） |
| 評估指標 | AIME 2024 / MATH-500 / GPQA Diamond / LiveCodeBench / Codeforces / MMLU |

## 模型一句話

DeepSeek-R1 基於 DeepSeek-V3-Base（671B MoE, 37B activated），透過 GRPO 演算法進行大規模 RL 訓練，使模型自發發展出 reflection、self-verification、長 Chain-of-Thought 等推理行為，最終達到與 OpenAI-o1-1217 可比的推理表現。

## 對應論文

- **論文**：[DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning](https://arxiv.org/abs/2501.12948) ([PDF](https://github.com/deepseek-ai/DeepSeek-R1/blob/0cf7856/DeepSeek_R1.pdf))
- **這份 implementation 跟論文的偏離**：此 repo 為 paper/model release，原始訓練程式碼未開源。開源的內容為模型權重（HuggingFace）與 README 中的使用指引。實際訓練實作細節需參考 DeepSeek-V3 repo。

## 與競品比較

| 面向 | DeepSeek-R1 | OpenAI o1 | QwQ-32B-Preview |
|------|-------------|-----------|-----------------|
| 訓練方法 | GRPO-based RL (no critic model) | 未公開（推測含 PRM + RL） | 推測為 SFT + RL |
| 開源程度 | 完整開源（MIT, weights on HF） | 封閉 | 僅模型權重 |
| AIME 2024 Pass@1 | **79.8%** | 79.2% | 50.0% |
| MATH-500 Pass@1 | **97.3%** | 96.4% | 90.6% |
| 模型架構 | MoE (671B, 37B active) | 未公開（推測 dense） | Dense (32B) |
| 主要 trade-off | 671B MoE 推論成本高；蒸餾版顯著壓低成本（1.5B 即超越 GPT-4o） | 封閉生態，API 成本高 | 單一大小，無蒸餾系列 |

## 健康度信號

- ⭐ Stars: ~92.0k
- 📅 最後 commit: 2025-06-27
- 👥 主要維護者: DeepSeek-AI（約 150+ 貢獻者）
- 🔬 已被新方法部分取代（DeepSeek-R2 等後續版本），但此 repo 的訓練管線仍是推理模型的重要參考

## 我會在後續筆記中回答的問題

- GRPO 如何省去 critic model 仍能產出穩定梯度？
- 四階段管線中各階段的設計理由與取捨？
- 為什麼蒸餾比在小模型上直接跑 RL 更有效？
- PRM 和 MCTS 為什麼在 scaling 上失敗？
