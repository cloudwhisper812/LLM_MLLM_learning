## flash attention

### 传统 attention 的问题
GPU 计算很快，bottleneck 在 memory bandwidth，IO issue。SRAM 小，存不下 N x N 的大矩阵。Q × K^T 得到 S 存到 HBM（显存），softmax 后得到 P 存到显存。P × V 得到 O，再写到显存。

### FlashAttention
- 切块，把 Q, K, V 切成很多小块，每次只把一小块搬进 SRAM。在 SRAM 里算完这一小块的 attention。避免了中间结果在 HBM 上的读写。
- 引入的两个新问题如何解决：
  - 向前传播没有存中间矩阵 S和P，反向传播要用到怎么办，再算一遍！
- softmax 需要整行求和的统计量，解决方法 online softmax：它不需要等整行算完。它维护一个局部最大值和局部和。算一块，更新一下局部统计量；再算下一块，修正一下之前的统计量。
> online softmax 还挺有意思，说一下个人理解：前面的 block 可以先计算 o，只存分子部分（加权和），以及到目前为止的 softmax 分母局部总和，以及最大值。当计算到新的 block，发现有了新的最大值，更新最大值，修正过去的 o（分子部分）的尺度，更新分母总和（有了新的局部），这样到最后一个 block 后，我们就有了修正正确的每个 o 的分子，也有了正确的分母总和，一除就得到了正确尺度的每个 o。
