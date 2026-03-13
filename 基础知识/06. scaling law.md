## scaling law

### 核心
1. Scaling Laws for Neural Language Models 它告诉大家“大力出奇迹”是有数学依据的。结论：模型参数量最重要，数据量次之。建议优先把模型做大。
2. Training Compute-Optimal Large Language Models 它指出了 1 的错误，数据量 (D) 和参数量 (N) 同样重要：
- 模型的最终性能（Loss），与计算量 (C)、参数量 (N)、数据量 (D) 之间呈现精确的“幂律关系” (Power Law)。L(N)≈ A/(N^α)+B/(D^β)+E
- 架构不重要：在足够大的规模下，你是用 LSTM 还是 Transformer，层数深一点还是宽一点，对 Loss 的影响微乎其微。规模 (Scale) 才是决定性因素。
- 可预测性：你可以用小模型的 Loss 曲线，精准拟合出大模型的 Loss。这意味着在训练 GPT-4 之前，OpenAI 就已经知道它的效果会是多少了（De-risking）。
- 适用范围：LLM，MLLM，ViT，diffusion 等。
-  C≈6×N×D：Transformer 模型，每训练一个 Token，大约需要约 6N 次浮点运算（FLOPs）。前向传播 2N，反向传播 4N。


### 对我们的指导意义
- 别瞎改结构：不要花时间去研究“把 LayerNorm 换个位置能不能涨点”。在 Scaling Law 面前，这些微调的收益不如多喂点数据。
- 黄金法则：训练数据量应该是参数量的 20 倍 (D≈20ND)。比如 10B 的模型，应该用 200B Tokens 训练。
- 推理最优 (Inference Optimal)：如果你要在端侧部署（推理成本敏感），那就把模型做小 (N 小)，拼命堆数据 (D 大)。
- 可预测性工程：在跑大模型之前，先跑几个小规模实验（1B, 7B），拟合曲线，预测 70B 的效果。如果曲线不符合预期，立刻停下来检查数据质量，别浪费钱。


### 对业界的影响
- 既然数据这么重要，大家开始疯狂清洗 CommonCrawl，买版权数据，搞合成数据。因为 Scaling Law 告诉我们：没有数据，光有卡也没用。
- MoE (混合专家模型)，为了打破 Scaling Law 的计算瓶颈，用稀疏激活（Sparse）来骗过 Scaling Law：用更少的 FLOPs 达到大参数量的效果。
- 我们正在接近 Scaling Law 的边际效应区，这也是为什么最近大家都在搞 System 2 (推理侧 Scaling)，比如 o1/DeepSeek-R1，试图换一条赛道继续 Scale。
  - 核心思想：我不增加模型大小了，我增加推理时间：Chain of Thought，Search & Verification)，自我修正。
  - 未来可能不需要人人都在预训练（Pre-training）上卷参数量，而是在后训练（Post-training）和推理框架上卷“思考深度”。这就是 System 2 Scaling。
