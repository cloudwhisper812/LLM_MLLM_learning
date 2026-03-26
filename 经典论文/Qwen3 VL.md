# Qwen3 VL

## 模型结构
结构上基本和2.5的一样，主要做了下面这些改动
1. 使用 SigLip2. 并且用CoMP的方法来修改positional embedding （CoMP这个论文引用很少， 有机会看一下）
2. enhanced interleaved MRoPE: 2.5的时候MRoPE是把token切成3部分分别对应t，h，w，这样做造成t只有高频，w只有低频。interleaved后保证每个都有高中低频。
3. DeepStack Integration: 除了siglip的输出token经过mlp进入llm。中间有两层的feature也会经过mlp，然后add到llm vision token对应的第二三层的hidden state上，mlp的最后一层用0初始化。这样做的好处是底层中层数据特征也进入llm。
4. text-based time alignment for video： 每个frame都用text提前告诉llm对应的timestamp。
5. square root normalized per-token loss

## 预训练

训练流程：
从简单到难，sequence length从短到长，并且保证加入文本不会灾难性遗忘。
- stage 0: 只训练merger，做alignment。
- stage 1: 防止灾难性遗忘，在多模态数据上加入纯文本。8K上下文，图文交错，vqa，stem，少量视频。
- stage 2: long context pre training： 增加纯文本长文的比例。视觉上引入大量视频和agent指令数据。 32k的上下文，视频本来就很长。
- stage 3: ultra-long-context adaption： 这个阶段目的不是学习新知识，适应256k的上下文。

数据：
-  model as data factory：造数据，洗数据。老生常谈了。
-  对图片采用0 - 1000的坐标归一化。
-  保证长尾分布的采样。对图片特征聚类，找到稀疏区域，增强采样。
-  对于stem难的分布，拆解两步。先纯视觉感知，用代码生成大量集合图，让模型做定位点（找交点，重心）和基础vqa。第二部把llm和上面能力结合。

## post training


1. sft - 激活和分化
   -  120w数据。1/3文本，2/3视频和图文。先32k上下文sft，然后256k。
   -  一部分数据时一问一答（no thinking模式）。一部分数据时长篇大论的思维链模式。多模态模型也要有think模式。

2. strong-to-weak distillation 纯文本蒸馏
   -  用强大llm蒸馏llm backbone。因为多模态训练后，llm变笨，我们要把这个智商补回来。
   -  off policy和online policy。qwen3的distill用过，这里就不赘述。
  
3. rl
   - reasoning rl：先用模型采样，都做错或者都做对的不要。经典的甜点区，rl只在模型跳一跳够得着的地方发挥，太难得话reward 0，提督消失，太简单的学不到东西。死磕有标准答案的推理题（数学，代码，视觉定位）
   - general rl： rule based rewards， model based rewards。如果reasoning是为了拔高智商，general rl就是为了提高情商和纠正坏习惯。
