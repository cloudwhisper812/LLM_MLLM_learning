## mllm 架构
> BLIP, Flamingo基本不用了，被llava统治了。llava足够简单和暴力，scaling law的胜利，llm够大，数据够多。
> 所以这里主要回顾讲解llava，blip/flamingo重点强调主要思想，他们的思想现在还是有在特灵领域应用的。
> 貌似现在闭源的大模型都是end2end了

### llava
#### 1. 模型结构
vision encoder + projector（两层全联接） + llm。 image过vision encoder和projector，得到vision tokens和text tokens拼到一起输入到llm。 
anyres：原始大图resize小图输入vit提供全局信息，大图crop局部输入vit提供局部信息，两者的tokens拼到一起。 但是现在siglip2都直接navit了，大力出奇迹。

#### 2. 训练流程
step1: feature alignment，只训练projector。数据用简单的图-文对，caption data。
step2: 指令微调 instruction tuning，全部参数可学习，可用LoRA。数据是复杂的对话数据，详细描述，复杂推理。step1是caption陈述句，step2是qa问答，step1计算caption部分loss， step2只计算answer部分loss，问题不算。

#### 3.数据构成
step1用开源简单的caption数据。step2指令微调用论文生成的数据。
from gpt：数据构造 (Data Generation) —— LLaVA 的灵魂。LLaVA 最牛的地方在于它没有用人工标注，而是用 GPT-4 生成了高质量的指令微调数据。
- 输入：给 GPT-4 (纯文本版) 提供图片的 Caption (标题) 和 Bounding Box (检测框)。
- Prompt：让 GPT-4 假装自己看到了图片，生成三类数据：
  1. 多轮对话：用户问“这张图里有什么？”，GPT-4 编一段对话。
  2. 详细描述：用户问“详细描述这张图”，GPT-4 写一段几百字的小作文。
  3. 复杂推理：用户问“图里的人为什么要带伞？”，GPT-4 根据 Bounding Box 推理出“因为在下雨”。
- 意义：这证明了数据质量 > 模型结构。LLaVA 的成功很大程度上归功于这 158K 高质量数据。
