## SigLIP

### 1. SigLIP 1：二分类替换多分类
CLIP 相似度矩阵计算量太大，SigLIP 把图文匹配变成二分类问题，
没有了 Softmax 的分母，各个 GPU 节点不再需要同步完整的特征矩阵，可以毫无压力地把 Batch Size 飙到百万级别。

没有cls token，转而使用 MAP (Multihead Attention Pooling)。直接拿所有的 Patch Tokens 算一个 Attention 权重，然后把它们加权求和（汇聚）成一个单独的全局向量。

这么直观的方法，为什么之前的人没试过？
> ### Sigmoid 的致命陷阱：被负样本的汪洋大海淹没
> 想象一下你现在的一个 Batch Size 是 $32,000$。
> 在这 $32,000$ 对图文中，对于任何一张图片，只有 **1 个** 文本是它的正样本（Ground Truth），剩下的 **31,999 个** 全是负样本。
>
> * **如果你用 Sigmoid（独立二分类）**：
>   模型会对这 $32,000$ 个配对分别做判断。它会收到 1 个正向梯度（告诉它“拉近距离”）和 31,999 个负向梯度（告诉它“推远距离”）。
> * **结果是什么？** >   负向梯度的总和会瞬间把正向梯度彻底碾压。模型为了快速降低 Loss，会直接“摆烂”，学到一个极其偷懒的捷径：**把所有图文特征全部推远，预测所有的 Pair 都是不匹配的。** 这样它的准确率依然高达 99.99%（因为绝大多数确实是不匹配的），但模型彻底废了，什么表征也没学到。
>
>
>
> ### 为什么 Softmax (InfoNCE) 能活下来？
> Softmax 的公式天生自带一种**“强迫竞争” (Competitive) 的机制**。
> Softmax 的分母把所有的相似度加在一起做归一化，这就意味着概率之和必须为 1。
>
> * **机制优势**：不管那 31,999 个负样本有多想把距离推远，Softmax 强迫模型必须在里面**挑出一个最像的**（分配最高的概率）。
> * **隐式难负样本挖掘 (Implicit Hard Negative Mining)**：Softmax 算梯度时，那些长得极其不像的负样本（Easy Negatives），算出来的概率接近 0，梯度也接近 0，模型不会在它们身上浪费精力；而只有那些长得很像的正样本，或者极具迷惑性的负样本（Hard Negatives），才会产生巨大的梯度推动模型更新。
>
> 这就是为什么从 SimCLR 到 MoCo 再到 CLIP，整个业界长达好几年都把 Softmax 奉为圭臬的原因。它完美解决了不平衡问题。

#### SigLIP 是怎么让 Sigmoid 起死回生的？

既然 Sigmoid 有这么大的缺陷，SigLIP 的作者（Google 团队）是怎么把它捞回来的？秘密全在你刚才贴的那行公式的尾巴上。$$L_{SigLIP} = -\dots \log \left( \frac{1}{1 + e^{-s \cdot (x_i \cdot y_j) + b}} \right)$$ 仔细看，里面不仅有温度系数 $s$，还有一个可学习的偏置项 (Learnable Bias) $b$。正是这个看似不起眼的 $b$，成为了扭转乾坤的关键。精准的初始化魔法：在做内容安全或者推荐系统这种极度不平衡的任务时，你一定知道 Prior Probability（先验概率）的初始化有多重要。SigLIP 在训练的最开始，把这个偏置 $b$ 初始化为一个负数（通常是 $-\log(N)$ 的近似值）。抵消负样本洪流：因为 $b$ 一上来就是一个很大的负数，所以在训练初期，模型默认所有对的相似度输出都极低。这就相当于提前给模型打了一剂预防针：“我知道负样本极多，我已经把整体水位降下来了”。这样，那 31,999 个负样本在初期产生的梯度就会变得极小，再也不会淹没掉那唯一的 1 个正样本的梯度了。

### 2. SigLIP 2
#### 2.1 引入了 NaViT (Native Resolution ViT) Patch n' Pack 的机制。
它保留了图像的原始长宽比，把不同分辨率的图像切成不同数量 Patch 后，像 NLP 里的 Token 一样展平。因为 Transformer 设定的最大序列长度（Sequence Length）是固定的，所以多张图的tokens可以拼到一起，巧妙利用attention 来解决。
#### 2.2 除了map后 token的对比学习，增加额外loss
1. mask tokens prediction loss，类似mae
2. 自蒸馏的Global-Local Loss，类似dino。架构被拆分成 Teacher（冻结/动量更新）和 Student（正在训练）两个网络。
老师看完整的原图，提取出一个全局特征。学生只看原图被随机裁切出来的一小块局部（Crop）。
Loss 要求：学生算出来的局部特征，必须去拟合/对齐老师的全局特征。除了map后的embedding对齐，还要求mask的输出 token也对齐。
作用：让每一个局部的 Patch 都具备“见微知著”的能力，增强特征的鲁棒性。
3. LocCa loss，增加一个text decoder

    全局描述 (Captioning)：看着图，生成整张图的描述。

    区域描述 (Region-specific Caption)：给模型一个边界框 (Bounding Box) 的坐标，让它描述框里是什么。

    指代定位 (Referring Expression)：给模型一段文字（比如“左边穿红衣服的女孩”），让它预测出对应的 Bounding Box 坐标。