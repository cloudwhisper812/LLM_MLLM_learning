## positional embedding
> Transformer对位置不敏感（Self-Attention是集合运算，打乱顺序结果不变），所以必须手动注入位置信息。

### 1. 绝对位置编码
#### 1.1 Sinusoidal PE （attention is all you need 原文）
- sin，cos function，变量是pos index，以及每个token中单个数值的index
- 可以外推(Extrapolation)，但是效果一般
#### 1.1 Learnable PE （BERT, GPT-2, ViT）
- 在固定长度任务上表现好，比如原始vit的输入图像尺寸固定。完全没法外推

### 2.相对位置编码 Relative PE 
#### 2.1 RoPE (Rotary Positional Embedding) 现在的主流
- 在计算attention之前，对Q,K进行旋转。位置 m就把向量逆时针旋转 mθ 角度。位置 n就把向量逆时针旋转 nθ角度。当计算两者点积的时候，结果只跟m-n的相对角度有关，跟绝对位置无关。
- 乘法注入：相比于加法（Add），旋转（Multiply）保留了向量的模长，信息损失更少。
- 个人理解：直接根据位置旋转q和k，旋转角度和位置以及token里的单个数值的index有关。经过点乘后，相邻的token的角度差距小会被放大内积，远的两个token的角度相差大会被缩小内积。
- 理论上的无限外推(NTK-Aware)：
  - 相对位置不变性：不管绝对位置多大（哪怕是 100 万），只要 相对距离 没变（还是相邻），点积结果就 完全一样。   
  - 拉伸 (Scaling)：强行把 8192 的长度 压缩 回 4096 的范围内。
  - 做法：把所有频率 θi​ 除以一个系数 s（比如 s=2）。
  - 效果：原本转 8192θ 的角度，现在只转 4096θ。
  - RoPE Base (基频) 决定了外推的上限，Llama 3 把这个基数从 10000 改成了 500000 (50万)。HunyuanVideo 甚至用到了 更大的基数。
- 3d rope（后面看一下）
#### 2.2 Alibi
- 直接对attention map上的attention score 减去一个线性偏置。相对距离。距离越远，减得越多。
- 无限外推。适合超长序列
- 为了让模型能关注不同范围的上下文，ALiBi 给每个 Head 分配了 不同的斜率 m（乘法力度，线性偏执的权重。）
