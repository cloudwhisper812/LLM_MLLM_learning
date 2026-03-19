## DCA（Dual Chunk Attention）


核心作用：DCA 是一种免训练（Training-Free）的推理期技术，专门用于解决模型在处理超出其训练长度的超长序列时遇到的位置编码外推问题。在 Qwen2.5-1M 中，正是 DCA 将原本 256K 训练长度的模型硬生生扩展到了 100 万（1M）Token 的处理能力。
工作原理：当输入序列过长时，RoPE 会遇到训练时未见过的巨大相对位置距离。DCA 的核心思想是将超长序列分割成多个较小的块（Chunks），并将相对位置重新映射到较小的值，确保任意两个 Token 之间的距离都不会超过模型预训练时的最大长度。
-三种注意力模式：DCA 在底层将注意力计算分为三种模式来维持全局连贯性：
1. 块内注意力 (Intra-Chunk Attention)：处理同一个块内 Token 之间的注意力，保留原始的相对位置。
2. 块间注意力 (Inter-Chunk Attention)：处理不同块之间 Token 的注意力，使用重复的序列作为相对位置。
3. 相邻块注意力 (Successive-Chunk Attention)：处理相邻两个块之间的注意力，在局部窗口内保留原始相对位置，较远距离则采用块间注意力的处理方式。


三个不同的 Query 变体（`Q_intra`, `Q_inter`, `Q_succ`）是通过对同一个原始 Query 向量应用不同的位置索引（Position Index）进行 RoPE 旋转得到的。

核心思想是：欺骗模型，让它以为所有的 Token 都在它预训练见过的长度（比如 256K 或 32K）以内。
我们假设模型的预训练窗口大小是 ccc（比如 32K），我们将超长文本切分成大小为 sss 的块（Chunk，通常 s=cs = cs=c）。对于第 iii 个 Token（它的真实绝对位置是 iii），它的原始 Query 向量是 qiq_iqi​。
以下是这三个变体是如何通过修改位置索引算出来的：

#### 1. Q_intra (用于块内注意力)
- 目标：处理同一个 Chunk 内的 Token。因为它们都在同一个块里，距离很近，模型完全能处理。
- 位置索引分配：保持真实的绝对位置不变。
- 计算方式：直接用真实的绝对位置 iii 对 qiq_iqi​ 进行 RoPE 旋转。
  - `Q_intra` = RoPE(qi,position=i)\text{RoPE}(q_i, \text{position}=i)RoPE(qi​,position=i)
- 效果：当它和同一个块内的 KjK_jKj​（位置为 jjj）计算时，相对距离就是真实的 i−ji - ji−j。
#### 2. Q_inter (用于跨块远距离注意力)
- 目标：处理相隔很远的 Chunk 之间的 Token。如果直接算，相对距离 i−ji - ji−j 可能会达到几十万，模型会崩溃。
- 位置索引分配：把所有块的索引“折叠”或“重置”。 DCA 强制让每个 Chunk 内部的 Token 重新从 0 开始编号（或者映射到一个固定的局部范围）。
- 计算方式：假设 Token iii 在当前块内的局部偏移量是 ilocal=i(mods)i_{\text{local}} = i \pmod silocal​=i(mods)。DCA 会用这个局部偏移量作为位置索引进行旋转。
  - `Q_inter` = RoPE(qi,position=ilocal)\text{RoPE}(q_i, \text{position}=i_{\text{local}})RoPE(qi​,position=ilocal​)
- 效果：当它和远处的 KjK_jKj​（假设 jjj 的局部偏移是 jlocalj_{\text{local}}jlocal​）计算时，模型看到的相对距离变成了 ilocal−jlocali_{\text{local}} - j_{\text{local}}ilocal​−jlocal​。这相当于把所有相隔十万八千里的块，在位置编码上强行“叠”在了一起，距离永远不会超过块的大小 sss。
#### 3. Q_succ (用于相邻块注意力)
- 目标：处理相邻两个 Chunk 的边界。比如块 1 的末尾和块 2 的开头，物理距离很近，如果用 `Q_inter` 的折叠方式，会破坏这种局部连贯性。
- 位置索引分配：平移一个块的长度。
- 计算方式：为了让当前块的 Token 能够平滑地连接到前一个块，DCA 会给当前 Token 的局部偏移量加上一个块的长度 sss。
  - `Q_succ` = RoPE(qi,position=ilocal+s)\text{RoPE}(q_i, \text{position}=i_{\text{local}} + s)RoPE(qi​,position=ilocal​+s)
- 效果：当它和前一个块的 KjK_jKj​ 计算时，相对距离被巧妙地控制在了 [0,2s][0, 2s][0,2s] 的范围内，既保留了局部的相对顺序，又没有超出模型预训练的最大长度限制。
#### 总结
这三个 Q 的本质区别，仅仅在于传给 RoPE 函数的那个 `position` 参数不同：
1. `Q_intra` 传入的是真实全局位置。
2. `Q_inter` 传入的是块内局部位置（折叠距离）。
3. `Q_succ` 传入的是块内局部位置 + 块长度（平滑过渡）
