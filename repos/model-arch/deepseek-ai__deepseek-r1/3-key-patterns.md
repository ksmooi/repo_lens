---
repo: deepseek-ai/DeepSeek-R1
file: 3-key-patterns
studied_at: 2026-05-28
commit_sha: 0cf7856
---

# DeepSeek-R1 · 值得偷學的設計

## Pattern 1: Group-Based Advantage Normalization（不用 Critic Model）

**是什麼**：GRPO 捨棄了 PPO 依賴的 critic model，改以 group 內 reward 的 mean 和 std 來計算 advantage。

**為什麼有效**：Critic model 需要與 policy model 差不多大小的神經網路來估計 value function，這等於兩倍的訓練成本。GRPO 的洞察是：對每個問題取樣多個輸出（G=64），這 group 本身就提供了該問題的 reward distribution，可以用來估計「這個輸出相對 group 有多好」。當 G 夠大時（64），group 的統計量足以穩定地替換 critic 的估計。

**替代方案**：
- **PPO + critic model**：需要額外一個大模型，記憶體翻倍，且在 RL 訓練中 critic 和 policy 的收斂速度不一致時會不穩定
- **REINFORCE leave-one-out**：用 group 內其他輸出的 reward 平均值代替 advantage，但不做標準差正規化，對 reward scale 敏感

**何時可以借用**：任何需要 RL 訓練的場景，特別是當 reward function 可以在 batch 內計算（不需外部分數）時。如果 reward 來自外部 API（每個調用有成本），group sampling 的成本效益就需要權衡。

**注意事項**：當 group 內 reward 變異很小（例如所有輸出都接近正確或都完全錯誤）時，advantage signal 會接近零，訓練效率下降。

---

## Pattern 2: Rule-Based Reward > Neural Reward Model

**是什麼**：DeepSeek-R1 刻意選擇 rule-based reward（答案正確性比對、格式檢查、語言比例）而非 neural reward model。

**為什麼有效**：
1. **避免 reward hacking**：Neural reward model 在大規模 RL 中會被 policy model 找到漏洞並 exploit；rule-based 的邊界清晰，無法 hacking
2. **簡化管線**：不需要訓練和維護 reward model，不需要處理 reward model 與 policy model 之間的同步問題
3. **Zero cost scaling**：Rule-based reward 在訓練中完全免費（只是字串比對），不會成為瓶頸

**替代方案**：
- **Neural PRM（Process Reward Model）**：可以提供 step-level 的細粒度信號，但論文中指出在一般推理中難以定義「步驟」，且自動標註品質差
- **Outcome neural RM**：只在最後一步給分，但不比 rule-based 更好，且容易 reward hacking

**何時可以借用**：任何任務的結果可以被自動驗證時（確定性數學、程式測試、格式檢查）。對於開放式任務（創意寫作、對話），rule-based 顯然不夠，仍需 neural RM。

**注意事項**：這個 pattern 的適用範圍受限於「可自動驗證的任務」。DeepSeek-R1 的 Stage 4 就是因為需要處理 general data，所以不得不退回到 reward model。純 rule-based 無法處理 need 判斷式的任務。

---

## Pattern 3: Cold Start + Iterative RL/SFT Pipeline

**是什麼**：四階段訓練管線（Cold Start SFT → Reasoning RL → Rejection Sampling SFT → All-Scenario RL），交替使用 RL 和 SFT。

**為什麼有效**：RL 和 SFT 各有優勢與弱點：
- RL 擅長**發現**新模式（"aha moment"），但訓練不穩定、輸出品質控制困難
- SFT 擅長**鞏固**已有模式，但無法超越資料分布

交替使用兩者，RL 發現新模式 → SFT 將模式固化為可擴散的資料 → RL 在更穩定的基礎上繼續探索。這個迭代過程類似 AlphaGo 的 self-play + supervised learning 交替。

**替代方案**：
- **純 RL（R1-Zero 路線）**：不需要 SFT 資料就能達到強推理，但輸出品質差、語言混合嚴重
- **純 SFT（傳統 post-training）**：需要大量高品質標註資料，且模型表現上限受限於資料

**何時可以借用**：任何需要提升模型能力的 post-training 場景，特別是當你有辦法自動驗證結果的正確性時。這個 pattern 特別適合「能力提升」與「品質控制」之間的權衡場景。

**注意事項**：四階段設計的最大代價是**管線複雜度**——每一階段都有 checkpoint 選擇、資料收集、超參數調校的工作，iteration cycle 極長。

---

## Pattern 4: Distillation > Small Model RL

**是什麼**：訓練小模型推理能力時，從大模型蒸餾（SFT on teacher's outputs）遠比在小模型上直接跑 RL 更有效。

**量化證據**：

| 方法 | 32B 模型 AIME 2024 |
|------|-------------------|
| 小模型自訓練 RL（DeepSeek-R1-Zero-Qwen-32B） | 47.0% |
| 從 R1 蒸餾（DeepSeek-R1-Distill-Qwen-32B） | **72.6%** |
| 差距 | +25.6pp |

**為什麼有效**：大模型在 RL 中「發現」的推理模式（reflection、self-verification、長 CoT）是一種知識。小模型自己從零開始 RL，不僅運算成本高（需要大量 sampling），而且由於 capacity 限制，可能永遠無法發現某些模式。蒸餾讓小模型直接「繼承」大模型已經發現的知識。

**替代方案**：
- **小規模 RL（直接用 RL 訓練小模型）**：在論文對比中表現遠不如蒸餾
- **知識蒸餾（logit-level KD）**：更細粒度的蒸餾，但需要 teacher model 在線推理，成本更高

**何時可以借用**：任何「希望在有 capacity 限制的模型上部署某個能力」的場景。蒸餾的成本（一次性的 teacher 推理）遠低於 RL 的迭代成本。

**注意事項**：蒸餾只能轉移已知的模式，無法超越 teacher 的能力。如果想要突破 intelligence 的邊界，最終還是需要大規模 RL。

---

## Pattern 5: 誠實的 Reward Hacking 防禦

**是什麼**：DeepSeek-R1 透過多層 reward 設計來防禦 reward hacking，而非依賴單一 reward signal。

**三層防禦**：
1. **Combined reward**：accuracy + format + language，單一 hacking 無法獲得高分
2. **Group normalization**：如果 group 內所有輸出都 hack 了同一個 reward（例如都輸出 `<think>` 但內容亂寫），advantage 會被 group 的整體分布正規化，gradient 接近零
3. **Rule-based 而非 neural**：規則邊界清晰，無法模糊 exploit

**為什麼有效**：Reward hacking 的本質是 model 找到 reward function 中沒有被設計者預期的「捷徑」。多層、互補的 reward signal 讓找到所有捷徑的難度指數級上升。

**替代方案**：
- **單一 reward + KL penalty**：PPO 的標準做法，但容易被 exploit（KL 的懲罰是軟約束）
- **Adversarial reward model**：動態更新 reward model，但訓練複雜度高

**何時可以借用**：任何 RL 訓練場景都應該有至少兩層 reward 防禦。如果只有單一 reward，一定要加上 group normalization 或 KL 懲罰。

**注意事項**：Language consistency reward 的 ablation 顯示會輕微降低 performance。trade-off 是存在的——更強的 reward 規範 = 更小的探索空間。

---

## ML 工程品味的觀察

- **對 abstraction 的態度**：此 repo 本身幾乎沒有 abstraction——它只是一個 paper release。論文本身對訓練管線的描述是高度結構化的四階段流程，這種清晰的階段劃分本身就是一種良好的 engineering abstraction
- **對誠實度的重視**：論文花了顯著篇幅（§4.2）記錄失敗經驗（PRM、MCTS），這種誠實在學術論文中不常見，對後續研究者非常有價值
- **對簡潔性的偏好**：Rule-based reward > neural RM 的選擇、GRPO 省去 critic model 的設計、蒸餾僅用 SFT 不做 RL——這些都是「盡量簡單」的設計品味。在 DeepSeek 的哲學中，如果簡單方案足夠好，就不追求複雜方案
- **量化驅動**：每個設計決策都有對應的量化證據（AIME/MATH 分數變化），沒有「憑感覺」的設計
