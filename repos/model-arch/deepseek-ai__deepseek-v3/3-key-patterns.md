---
repo: deepseek-ai/DeepSeek-V3
file: 3-key-patterns
---

# DeepSeek-V3 · 值得偷學的設計

## Pattern 1: MLA Absorb Mode — 把解壓縮延後到 Attention 之後

**是什麼**: 不是先還原多頭 K/V 再做注意力，而是在壓縮後的 latent 空間做 score 計算，等到 weighted sum 後再用剩餘的 V 投影還原。

```python
# 標準做法 (naive mode, model.py:471-478):
kv = self.wkv_b(self.kv_norm(kv))          # 先還原成多頭
k, v = split(kv)                             # (B, S, 128, 192) + (B, S, 128, 128)
scores = q @ k                               # 在高維度做 attention

# Absorb mode (model.py:481-495):
q_nope = q_nope @ wkv_b[:, :128]            # 把 QK 投影融合進 q
scores = q_latent @ kv_cache + q_pe @ pe_cache  # 在低維(512)做 attention
x = scores @ kv_cache                        # weighted sum 仍在 latent 空間
x = x @ wkv_b[:, 128:]                      # 最後才還原成 V 維度
```

**為什麼有效**: 把 `wkv_b`（KV 的升維投影）拆成 QK 部分和 V 部分。QK 部分被absorb進 query，V 部分延後到 weighted sum 之後——這樣 KV cache 只需要存 512 維的 latent 而不是 128×192+128×128 = 40,960 維。**71 倍的 KV cache 壓縮**。

**程式碼位置**: [`model.py:481-495`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L481-L495)

**何時可以借用**:
- 當你的模型 KV cache 是記憶體瓶頸（特別是長 context 推論）
- 當你願意接受一點額外的計算開銷來換取大幅減少的記憶體

**注意事項**:
- 這需要 torch.einsum 的自動最佳化支援；在低階硬體上 naive mode 可能更快
- 權重融合假設 `wkv_b` 在推論時是 static 的
- `weight_dequant` 在 FP8 模式每次 forward 都要執行 [`model.py:481`]，增加了計算量

**替代方案**:
- MQA/GQA（多 query + 少 KV heads）：LLaMA 的做法，KV cache 壓縮比 ~4-8x，但需要修改 attention head structure
- 傳統 KV cache offloading：把 KV cache 放到 CPU，只傳送必要部分到 GPU，加大頻寬需求

---

## Pattern 2: Auxiliary-Loss-Free Load Balancing — 用 Bias 取代 Aux Loss

**是什麼**: 傳統 MoE 使用 auxiliary loss（如 load balancing loss）來鼓勵 router 平均分配 token 到各專家，但這個 loss 會幹擾模型的主要訓練目標（next-token prediction）。DeepSeek-V3 改用 **sigmoid gating + learnable bias** 來達成同樣效果。

**替代方案比較**:

| 面向 | Aux Loss 方法 | Auxiliary-Loss-Free (DeepSeek-V3) |
|------|-------------|----------------------------------|
| 路由函數 | softmax | sigmoid |
| 平衡機制 | 在 loss 上加 regularization term | learnable bias 參數 |
| 對主任務的干擾 | 有（需要調 loss weight） | 無（不參與主要 loss） |
| 程式碼 | 需要額外 loss 計算 + weight 調參 | router 多一個 bias vector |
| 收斂難度 | 高（loss weight 敏感） | 低（bias 自適應） |

**為什麼有效**: Sigmoid 的輸出不是機率分佈（不像 softmax 總和為 1），所以 router 可以對多個 expert 同時給高分。這讓 bias 可以獨立調節每個 expert 的熱度——偏載的 expert 累積更多梯度來校正 bias。

**程式碼位置**:
- Gate class: [`model.py:535-598`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L535-L598)
- sigmoid gating: [`model.py:577-580`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L577-L580)
- bias 只在最大模型使用（dim=7168）: [`model.py:564`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L564)

**何時可以借用**:
- 任何 MoE 場景，特別是你發現 auxiliary loss 難以收斂時
- 當 model quality 是最優先考量、不想加任何 regularization

**注意事項**:
- Bias 只在 dim=7168（即完整 671B 模型）時啟用；小模型（如 config_16B）無 bias
- 由於訓練程式碼未開源，bias 的具體更新策略（learning rate、初始化）只能從論文推測
- Group routing 的 top-2 per group + top-4 groups 邏輯跟 bias 交互 —— bias 對組內 routing 影響較大

**不適用情境**:
- 專家數量極少（<8）：不需要複雜的平衡機制
- 路由需要嚴格 one-hot（每個 token 只去一個 expert）：sigmoid 不適合

---

## Pattern 3: Group-Limited Routing — 地理多樣性 > Top-K 精準度

**是什麼**: 不直接對所有 256 experts 做 top-8 選擇，而是先分 8 組，選 top-4 組，再在每組內取 top-2。總共仍維持 top-8 activated。

```python
# model.py:584-592
scores = scores.view(x.size(0), n_groups, experts_per_group)
group_scores = scores.topk(2, dim=-1)[0].sum(dim=-1)  # 每組取 top-2 加總
indices = group_scores.topk(topk_groups, dim=-1)[1]    # 取 top-4 組
mask = ...                                              # mask 掉未選中的組
scores = scores.masked_fill(mask, -inf).flatten(1)
indices = torch.topk(scores, topk, dim=-1)[1]          # 在選中組內取 top-2
```

**為什麼有效**: 如果只做全局 top-8，可能 8 個 activated experts 都集中在同一組內（因為同一組的 experts 學習了類似的能力）。這會導致 experts 退化為「只有 8 個真的被用，其他 248 個浪費」。Group 限制強迫 experts 跨組參與，增加專家多樣性。

**程式碼位置**: [`model.py:584-592`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L584-L592)

**何時可以借用**:
- 專家數量很大（256+），需要確保專家使用率均勻
- 你的 experts 有明顯的組內相似性時

**注意事項**:
- `topk_groups` 和 `n_groups` 的比例很重要：DeepSeek-V3 設定為 4/8 = 0.5，表示強迫至少 50% 的組參與
- 如果組數太少，group routing 的效益不明顯
- 這個機制跟 bias 共同作用：bias 影響專家分組內的排名，group routing 決定哪些組參與

**替代方案**:
- 全域 top-K（GShard 原始做法）：簡單但可能專家使用不均
- Expert choice routing（Gemini 1.5）：反過來讓專家選 token，梯度計算更複雜

---

## Pattern 4: FP8 Weight + Triton Custom GEMM — 端到端量化推論

**是什麼**: 權重以 FP8（e4m3）格式存儲，使用 128×128 block-wise quantization，並透過自訂 Triton kernel 在推論時做 activation 量化 + FP8 GEMM。

**為什麼有效**:
1. 權重儲存在 FP8 減少了 50% 的模型載入記憶體（比起 BF16）
2. Block-wise scaling（128×128）比 per-tensor scaling 更細粒度，量化誤差更小
3. Triton kernel 自動調優（autotune）不同 block size 的組合，找到硬體最佳配置

**程式碼位置**:
- Activation quantization: [`kernel.py:9-57`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/kernel.py#L9-L57)
- Weight dequantization: [`kernel.py:60-110`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/kernel.py#L60-L110)
- FP8 GEMM with autotune: [`kernel.py:113-196`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/kernel.py#L113-L196)
- 自動調優配置：36 組 (BLOCK_SIZE_M × BLOCK_SIZE_N × num_stages) 組合 [`kernel.py:113-116`]

**替代方案**:
- PyTorch FP8 (torch.fp8)：使用 `torch.compile` 的 FP8 支援，但靈活性低於自訂 kernel
- vLLM FP8：vLLM 內建的 FP8 支援（W8A8），整合進其 PagedAttention 架構
- AWQ/GPTQ（INT4）：較低精度但需要校正集做 post-training quantization

**注意事項**:
- Triton 只支援 NVIDIA GPU（需要 CUDA）
- act_quant kernel 使用 `scale_fmt="ue8m0"` 可選模式（無符號 8-bit 指數），這是論文提到但 code 中有實作的特性 [`kernel.py:29-31`]
- `weight_dequant` 在 FP8 GEMM 之外另有 bf16 fallback 路徑 [`model.py:153-157`]

**不適用情境**:
- 非 NVIDIA GPU（AMD、Apple Silicon）：Triton 不相容
- 需要 BF16 精度的場景：可透過 `fp8_cast_bf16.py` 轉換

---

## Pattern 5: Tensor + Expert 混合平行 — 雙層模型平行策略

**是什麼**: 同時使用 tensor parallelism（ColumnParallelLinear / RowParallelLinear 切分 linear 層）和 expert parallelism（不同 GPU 載入不同 experts），透過 all_reduce 同步中間結果。

**為什麼有效**:
- Tensor parallelism 適合 dense 部分（embedding、attention、shared MLP）
- Expert parallelism 適合稀疏部分：256 experts 平均分給 N 張 GPU，每張只載入 256/N 個 experts
- 兩者組合讓 671B 模型可在 16-32 張 H800 上推論

**程式碼位置**:
- ColumnParallelLinear: [`model.py:208-234`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L208-L234)
- RowParallelLinear: [`model.py:237-267`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L237-L267)
- Expert 分配: [`model.py:658-662`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L658-L662)
- Expert 推論: [`model.py:684-692`](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e978/inference/model.py#L684-L692)

**何時可以借用**:
- 模型大到無法放進單張 GPU（這裡是 671B > 80GB × N cards）
- 已經用了 tensor parallelism 但模型仍然太大

**注意事項**:
- Expert 之間需要 all_reduce 同步 routed expert 輸出 [`model.py:692`]
- 先 tensor-parallel 切分 attention，再 expert-parallel 切分 MoE——但兩者共用 world_size，無法獨立設定
- Expert 的 local 分配是依 rank 順序分割 `[rank*local_n : (rank+1)*local_n)`，沒有負載感知

**替代方案**:
- Pipeline parallelism：不同層在不同 GPU，但 MoE 的專家分佈使 pipeline bubble 問題更嚴重
- Sequence parallelism（DeepSpeed Ulysses/Ring Attention）：適合長 context，但跟 MoE 的 interaction 較複雜

---

## ML 工程品味的觀察

- **依賴極簡**: 只依賴 torch、triton、transformers、safetensors 四個套件。整個推論流程無外部 serving framework
- **Abstraction 扁平**: model.py 只有 ~20 個 class，類別層次不超過 2-3 層。對比 HuggingFace Transformers 的 ~50 層繼承，DeepSeek-V3 的 code 極度直接
- **效能 vs 可讀性**: 在 MLA 上用 absorb mode（減記憶體但增加計算複雜度），在 kernel 上做 autotune（追求極致效能），但在 MoE 的 expert 執行上用簡單的 for-loop（可讀性優先）。這是一個務實的取捨：瓶頸在 attention 和 GEMM，不在 expert routing 的 overhead
- **Config-driven**: 模型架構完全由 JSON config 驅動，從 16B 到 671B 用同一份 code。`n_layers`、`dim`、`n_routed_experts` 等 hyperparameter 調整即可 scale
- **無測試**: 這是 reference implementation 典型特徵——優先保證論文 reproducibility，不追求 engineering robustness。對照 vLLM 的 extensive test suite，兩者的工程階段不同
