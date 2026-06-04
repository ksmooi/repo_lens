---
repo: deepseek-ai/DeepSeek-R1
file: 9-questions
studied_at: 2026-05-28
commit_sha: 0cf7856
---

# DeepSeek-R1 · 未解問題

## 還沒搞懂的設計決策

- [ ] **G=64 的選擇是否有理論依據？**
  - 我目前的推測：[UNVERIFIED] 這可能是 GPU memory 限制下的 trade-off——64 個輸出的 logits 剛好能放進單 GPU（671B MoE 用 expert parallelism 分散後）。更大的 G 理論上能提供更穩定的 advantage 估計，但 memory 需求線性增長
  - 相關程式碼：論文 §2.2.1

- [ ] **Cold-start data 的「數千條」具體是多少？**
  - 我目前的推測：[UNVERIFIED] 從經驗上推測可能是 5,000-20,000 條。如果太少（<1K）不足以為 RL 提供穩定起點；如果太多（>100K）會限縮 RL 的探索空間。但論文中沒有給出具體數字
  - 相關程式碼：[論文 §2.3.1](https://github.com/deepseek-ai/DeepSeek-R1/blob/0cf7856/DeepSeek_R1.pdf#page=9)

- [ ] **Stage 3 的 600K reasoning + 200K non-reasoning 比例是如何決定的？**
  - 我目前的推測：[UNVERIFIED] 可能是根據 validation loss 或 downstream benchmark 的改善幅度來調整。600:200 = 3:1 的比例意味著推理仍是最主要關注點，但非推理資料佔了 25%，足以維持通用能力
  - 相關程式碼：[論文 §2.3.3](https://github.com/deepseek-ai/DeepSeek-R1/blob/0cf7856/DeepSeek_R1.pdf#page=10)

- [ ] **Language consistency reward 的權重如何設定？**
  - 論文提到「slight degradation in model's performance」，但沒有給出量化的 degradation 數據。這個權重應該是在可讀性和推理能力之間的一個 trade-off
  - 我目前的推測：[UNVERIFIED] 可能 weight 很低（0.01-0.1），以免主導 accuracy reward
  - 相關程式碼：[論文 §2.3.2](https://github.com/deepseek-ai/DeepSeek-R1/blob/0cf7856/DeepSeek_R1.pdf#page=10)

## 想驗證的論文 claim

- [ ] **「純 RL 即可激發推理能力」的邊界在哪裡？**
  論文 claim：LLM 可以純靠 RL 發展出推理能力，無需 SFT。但 DeepSeek-V3-Base 本身是經過數兆 token pre-training 的模型，pre-training 階段已經學到了大量的語言與推理模式。所以更精確的問題是：**推理能力是在 pre-training 中被學到後被 RL「激活」，還是 RL 從零「創造」了推理能力？** 這個區別論文沒有討論。
  - 相關程式碼：[論文 §2.2.4](https://github.com/deepseek-ai/DeepSeek-R1/blob/0cf7856/DeepSeek_R1.pdf#page=6)

- [ ] **蒸餾 vs RL 的比較是否公平？**
  論文比較了 DeepSeek-R1-Distill-Qwen-32B（蒸餾 + SFT）和 DeepSeek-R1-Zero-Qwen-32B（直接 RL on base）。但後者只跑了「over 10K steps」，並且是從 base model 開始，沒有 cold-start data。如果給 Zero-Qwen-32B 相同的 cold-start SFT 和 4-stage pipeline，結果是否會不同？
  - 相關程式碼：[論文 §4.1 Table 6](https://github.com/deepseek-ai/DeepSeek-R1/blob/0cf7856/DeepSeek_R1.pdf#page=14)

- [ ] **Generative RM（用 DeepSeek-V3 當 judge）的 reliability 如何？**
  論文在 Stage 3 提到部分 data 使用 generative reward model，但沒有給出這個 judge 的 agreement rate 或 reliability 數據。如果 judge 本身有偏誤（例如偏好長回答），會不會汙染訓練資料？
  - 相關程式碼：[論文 §2.3.3](https://github.com/deepseek-ai/DeepSeek-R1/blob/0cf7856/DeepSeek_R1.pdf#page=10)

## 想問維護者的問題

- 671B MoE 的訓練 infra 用了多少 GPU / 多少天？
- Group size G=64 是怎麼決定的？試過其他的值嗎？
- Cold-start data 的具體 prompt 模板能否分享？
- Stage 2 RL 訓練收斂的判斷標準是什麼？
- DeepSeek-R1-Distill 的小模型如果也跑 RL（不是只做 SFT），能提升多少？

## 下次再看時的待辦

- [ ] 對照 DeepSeek-Math 論文的 GRPO 原始實作，確認演算法細節的一致性
- [ ] 對照 DeepSeek-V3 repo 的 MoE 架構，理解 671B / 37B active 的 routing 機制
- [ ] 試跑 R1-Distill-Qwen-7B 本地推論，觀察實際的 CoT 行為
- [ ] 閱讀 OpenAI o1 的技術報告（若有），比較 RL pipeline 的異同
- [ ] 比較 DeepSeek-R1-Zero 和 DeepSeek-R1 在可讀性上的實際差異（用 sample outputs）

## 跨專案對照備忘

- **Group-based advantage normalization (GRPO)** 與 vLLM 的 PagedAttention 的設計哲學有相似之處：都是「用 group 或 block 層級的統計來避免 per-element 的額外成本」。GRPO 省去 critic model，PagedAttention 省去 per-token memory allocation。→ **可能候選 pattern**：當系統需要對每個元素（token / response / request）分配資源時，group/block 層級的 aggregation 可以大幅降低 overhead。已在：GRPO（DeepSeek-R1）、PagedAttention（vLLM）、page-level KV cache（多個推理引擎）觀察到此模式。此設計已在 3 個以上獨立專案出現，建議資料達到閾值後在 `_patterns/` 新增 pattern。
