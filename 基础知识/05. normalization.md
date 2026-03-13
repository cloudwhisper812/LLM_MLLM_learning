## normalization

### 1. 为什么 LLM 用 LN 不用 BN
- 句子长度不一（Padding 问题）。
- Batch Size 通常较小（显存限制）。
- 对 RNN/Transformer 这种序列模型，LN 能保证每个时间步的分布一致。

### 2. Pre-Norm vs Post-Norm
- Post-Norm：
  - 优点：最后一层输出直接归一化，分布稳定。
  - 缺点：深层网络梯度容易消失/爆炸，训练难收敛（需要 Warmup）。
- Pre-Norm：
  - 优点：梯度直接通过残差路径传回输入，训练极稳定，不需要 Warmup 也能收敛。
  - 缺点：深层表示能力可能略弱（因为 Norm 在残差分支里）。
- 现状：现在的 LLM (Llama, PaLM, GLM) 几乎全用 Pre-Norm。训练稳定性压倒一切，深层网络的收敛比那一点点表达能力更重要。

### 3. RMSNorm
- 公式区别：
  - Layer Norm。需要算均值 μ 和方差 σ。 做了 减均值 (Re-centering) 和 除标准差 (Re-scaling)。
  - RMSNorm (Root Mean Square Norm): 不减均值！ 只做除均方根（RMS）。假设均值 μ≈0。对于高维向量通常成立）。
- 优势：
  - 计算更快：少算一个均值，少做一次减法。
  - 效果相当：实验证明，减均值对 Transformer 的性能影响微乎其微，RMSNorm 效果几乎一样好。
  - 简单：去掉了 Bias 项 β，参数更少。既然没用，layer norm也可以去掉了？
