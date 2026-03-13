## KV cache
> 我的理解：在 inference 的时候，自回归，新 token 作为 Q 和之前的 token 的 K 算 attention map，然后用它们和自己的 V 作为加权求和得到自己的新 value。每次新 token 前面的 K、V 都一样，所以不需要重复计算，我们可以保存下来重复利用。

### 1. 显存计算
Size=2 × Batch × Layers × Heads × Seq_Len × Head_Dim × Precision。FP16 是 2 Bytes，FP32 是 4 Bytes。

### 2. MHA/MQA/GQA
每个head不同q，k，v -> 每个head共享k，v -> 折中方案。把 Head 分组（比如 8 组），每组共享 K,V.效果接近 MHA，

### 3. PagedAttention (vLLM)
- 传统：传统框架（如 HuggingFace）会预先申请一个连续的显存块存kv cache（比如最大长度 2048）。 如果用户只生成了 10 个 Token，剩下的 2038 个位置都空着（Internal Fragmentation）。如果显存不够连续的大块，即使总剩余空间够，也申请不下来（External Fragmentation）。浪费率：通常高达 60%-80%！
- PagedAttention：借鉴了操作系统的虚拟内存管理。它把 KV Cache 切成小块（Blocks），允许这些块在物理显存中不连续。这样就能把零散的显存利用起来，极大提升了显存利用率和吞吐量。让显存利用率接近 100%，吞吐量提升了 2-4 倍。
