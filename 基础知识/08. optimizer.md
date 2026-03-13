## optimizer
SGD的原理就不用说了

### 1. Momentum
SGD 在山谷（峡谷）里震荡，收敛慢。引入动量 (Momentum)，积累之前的梯度方向。像推一个重球下山，有了惯性，遇到小坑（局部极小值）能冲过去，遇到峡谷震荡能被平滑掉。

### 2. 自适应学习率
梯度大的参数步长变小，梯度小的参数步长变大，实现了自适应。


### 3. Adam -> AdamW
- Adam 就是 momentum + adaptive learning rate
- AdamW：Adam 算 weight decay 也用了自适应 lr，梯度大的参数，正则化力度反而小（被除以大数），这完全反了！ 我们通常希望经常更新的参数受到更多约束，防止过拟合。正则化力度恒定。AdamW 把 weight decay 部分解耦，不管梯度大小，大家都按同样的比例衰减。这才是真正的 L2 正则化。

### 4. 梯度裁剪 (Gradient Clipping)
- Transformer 训练容易梯度爆炸 (Gradient Explosion)。
- 做法：在优化器更新之前，计算梯度的范数 (Norm)。如果超过阈值 (比如 1.0)，就整体缩放 (Rescale)。`torch.nn.utils.clip_grad_norm_`
- 作用：保证更新步长不会过大，维持训练稳定。


### 5. 其他还没看的：Adafactor，Lion (Evolved Sign Momentum)
