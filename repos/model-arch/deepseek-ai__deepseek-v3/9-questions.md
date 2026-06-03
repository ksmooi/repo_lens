---
repo: deepseek-ai/DeepSeek-V3
file: 9-questions
---

# DeepSeek-V3 · 未解問題

## 還沒搞懂的設計決策

- [ ] 為什麼 bias 只在 dim=7168（完整模型）時啟用？而小模型（config_16B、config_236B）的 Gate 沒有 bias？
  - 我目前的推測: 小模型的專家數量少或負載平衡較容易，bias 的效益不明顯。也可能是 bias 的初始化和更新依賴特定 training setup，小模型不一定適用 [UNVERIFIED]
  - 相關程式碼: [`model.py:564`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L564)

- [ ] `route_scale = 2.5` 這個值怎麼來的？跟 experts 數量（256）和 activated（8）的關係是什麼？
  - 我目前的推測: `route_scale` 放大 sigmoid 的輸出，應是為了在 softmax 替代方案中維持 routing weights 的方差。2.5 可能來自 256/8 的比例經驗值 [UNVERIFIED]
  - 相關程式碼: [`model.py:597`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L597)

- [ ] Group routing 中的 group scores 計算為什麼用 `scores.topk(2, dim=-1)[0].sum(dim=-1)`（取 top-2 加總）而非直接取 max 或 mean？
  - 我目前的推測: 用 top-2 sum 比 max 更穩健（避免 outlier），比 mean 更有鑑別力（鼓勵組內有至少 2 個強 expert） [UNVERIFIED]
  - 相關程式碼: [`model.py:588-589`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L588-L589)

## 想驗證的論文 claim

- [ ] **Auxiliary-loss-free load balancing 真的不需要 aux loss 嗎？**
  - 論文宣稱 sigmoid gating + learnable bias 完全取代了 auxiliary loss [§2.2.1]
  - code 中 bias 的梯度會透過 training 流回，但 repo 無訓練程式碼
  - 推測: 可能仍有一個極輕量的輔助機制（如 bias 的 L2 regularization），在論文中未提及

- [ ] **MTP 模組對訓練的實際貢獻**
  - 論文宣稱 MTP 提升了模型效能且可做 speculative decoding [§2.2.2]
  - 但推論 demo 完全沒用 MTP，speculative decoding 僅在第三方實作（如 SGLang）中
  - 需對照論文實驗結果來驗證 MTP vs standard next-token prediction 的差異

- [ ] **FP8 training 的穩定性 claim**
  - 論文說「整個訓練過程沒有任何 irrecoverable loss spike or rollback」
  - 這是極強的 claim——已知 FP8 training 在 overflow/underflow 邊界敏感
  - 推測: 128x128 block scaling 的細粒度大幅提供了動態範圍，加上 per-token-per-128-channel 的 online quantization 補償

## 想問維護者的問題

- 前 3 層用 dense MLP 而非 MoE 的設計理由是什麼？論文沒有提及這個設計選擇的實驗依據
- MoE 的 bias 更新策略是什麼？learning rate 與主模型是否不同？初始化值是什麼？
- 是否有實驗顯示 group routing 的 `topk_groups=4` 和 `n_groups=8` 是經過 sweep 找到的最優值？

## 下次再看時的待辦

- [ ] 對照 SGLang 的 DeepSeek-V3 MLA 實作，看 absorb mode 在不同 framework 實作上的差異
- [ ] 對照 vLLM 的 DeepSeek-V3 support（FP8/BF16），比較二者對 FP8 GEMM 的處理
- [ ] 追蹤 DeepSeek-V4 / DeepSeek-R1 的架構演進，看 MLA 是否被保留

## 跨專案對照備忘

- **低秩 KV 壓縮（MLA）**: 這跟 KV cache quantization（如 KIVI, 2024）不同，MLA 是壓縮維度而非量化精度。跟 MQA/GQA 的關係是「垂直壓縮 vs 水平壓縮」。目前已知只有 DeepSeek 系列使用 MLA
- **Auxiliary-loss-free MoE**: 傳統 MoE aux loss（如 switch transformer 的 load balancing loss）是主流。DeepSeek-V3 是第一個（目前唯一）在 production 規模驗證 auxiliary-loss-free 的。如果這個做法在 3 個以上不同 repo 出現，可考慮收錄進 `_patterns/moe-load-balancing-strategies.md`
- **FP8 Triton GEMM**: 跟 vLLM 的 FP8 支援（透過 cutlass/Marlin kernel）是不同路線。vLLM 使用更成熟的 cutlass 而不是 Triton。這跟 DeepSeek-V3 repo 的輕量級定位一致——用最小依賴達成目標
- **Tensor + Expert 混合平行**: 這個模式在 MoE 模型中常見（GShard, Mixtral 8x7B 都有類似設計），但不是顯著的跨 repo pattern（因為這類實作多藏在訓練框架中而非 model repo）

## 候選 pattern

- **MTP as auxiliary training objective**: 目前只在此 repo 看到。如果後續在其他框架（如 Megatron-LM、OpenMoE）也看到類似做法，可考慮收錄
