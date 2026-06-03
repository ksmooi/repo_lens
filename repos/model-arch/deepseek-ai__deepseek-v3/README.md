---
repo: deepseek-ai/DeepSeek-V3
type: model-arch
studied_at: 2026-05-27
commit_sha: 9b4e978
language: Python (CUDA Triton kernels)
framework: PyTorch 2.4 + Triton 3.0
task: next-token prediction (MoE language model)
paper: DeepSeek-V3 Technical Report (arXiv 2412.19437)
stars: ~104k
status: maintenance
---

# DeepSeek-V3 · 概覽

## 解決什麼問題

DeepSeek-V3 是一個 671B 參數的 Mixture-of-Experts (MoE) 語言模型，以僅 2.788M H800 GPU hours 的訓練成本，達到了與 GPT-4o、Claude-3.5-Sonnet 相當的效能。這是目前開源模型在「訓練效率」與「模型品質」之間最極端的取捨點——證明了 MoE 架構 + FP8 訓練可以讓超大模型的訓練成本降到跟密集小模型同級。

## 為什麼值得研究

這不是 typical 的論文復現 repo。它比較像「模型架構的 reference implementation + weight distribution hub」，但程式碼非常濃縮（~1400 行 Python，5 個檔案），卻包含了三個重大的架構創新：

1. **MLA（Multi-head Latent Attention）** — 把 KV cache 壓到每個 token 僅 512 維（而不是 n_heads × head_dim = 128 × 192 = 24,576 維），讓推論的 KV cache 記憶體大幅降低
2. **Auxiliary-loss-free load balancing** — 用 sigmoid gating + bias correction 取代傳統的 auxiliary balancing loss，完全消除負載平衡對模型效能的干擾
3. **Multi-Token Prediction (MTP)** — 同時預測多個未來 token，在訓練時提供更強的學習訊號，推論時可做 speculative decoding

## 模型一句話

671B MoE decoder-only transformer，61 層、128 heads、256 routed experts（8 activated/token）、MLA 注意力、auxiliary-loss-free load balancing、原生 FP8 權重。

## 對應論文

- **論文**: DeepSeek-V3 Technical Report ([arXiv 2412.19437](https://arxiv.org/abs/2412.19437))
- **這份 implementation 跟論文的偏離**: 此 repo 只包含推論 demo 和 weight 轉換腳本，不包含訓練程式碼。MTP 模組僅含 weights 載入，推論中未啟用 speculative decoding 功能。

## 健康度信號

- ⭐ Stars: ~104k
- 📅 最後 commit: 2025-08-28（穩定維護狀態，非活躍開發）
- 👥 主要維護者: deepseek-ai（組織），主要貢獻者包含 GeeeekExplorer、youkaichao
- 🔬 論文（2024/12）仍為 SOTA-level 開源 MoE 模型，已被 DeepSeek-R1/V4 系列取代

## 我會在後續筆記中回答的問題

- MLA 的 KV cache 壓縮機制如何具體實作？
- Auxiliary-loss-free load balancing 跟傳統做法差在哪？
- FP8 權重的量化規格與 dequantization 流程？
- Triton kernel 怎麼做 FP8 GEMM？
- MoE 的 distributed inference 怎麼切分 experts？

## 比較表格

| 面向 | DeepSeek-V3 | LLaMA 3.1 405B | Qwen2.5 72B |
|------|------------|----------------|-------------|
| 總參數 | **671B** (MoE, 37B active) | 405B (Dense, all active) | 72B (Dense) |
| 推論激活參數 | **37B** | 405B | 72B |
| 訓練成本 | **2.788M** H800 hrs | 30.8M GPU hrs | 推估 ~3M GPU hrs |
| 注意力機制 | **MLA**（壓縮 KV cache） | GQA (8 KV heads) | GQA (8 KV heads) |
| Context | **128K** | 128K | 128K |
| MMLU (Chat) | **88.5** | **88.6** | 85.3 |
| HumanEval-Mul | **82.6** | 77.2 | 77.3 |
| 權重精度 | 原生 FP8 | BF16 | BF16 |
