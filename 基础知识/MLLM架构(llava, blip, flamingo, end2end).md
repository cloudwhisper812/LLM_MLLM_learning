## mllm 架构
> BLIP, Flamingo基本不用了，被llava统治了。llava足够简单和暴力，scaling law的胜利，llm够大，数据够多。
> 所以这里主要回顾讲解llava，blip/flamingo重点强调主要思想，他们的思想现在还是在特定领域应用的。
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


### blip
#### blip1
1. encoder，decoder结构。感觉这个结构太复杂，没人用了，下次忘了可以gpt一下，对着论文看看。不过这个结构还挺有意思的，参数共享后，有一个双塔的粗召回ITC, 又有一个精排的ITM。
2. 论文提出的数据清洗也是重点，CapFilt (Captioning and Filtering)，少量高质量的人工标注数据得到filter和captioner：（1）Filter (过滤器)：训练一个 ITM 模型（看图文匹不匹配）。如果网上的 Caption 和图不匹配，直接扔掉。 （2）- Captioner (生成器)：训练一个 LM 模型（看图说话）。给网上的图重新生成一个高质量的 Caption（合成数据）。 （3）Bootstrap：用清洗后的数据 + 生成的数据，重新训练一个更强的 BLIP。这个数据量大于一开始的高质量的人工标注数据。（ **现在有更好的方法用gpt精标数据，训练student model来清洗数据生成caption，BLIP-1 captioner用coco之类训练的生成风格呆板，但是stutent gpt model因为来自gpt蒸馏，生成风格详细生动。**）

#### blip2
提出了q-former，其实就是detr里面的learnable query，也是siglip里面的map。两阶段训练loss也有点意思，第一阶段还保留了blip1的loss，把q-former当text encoder，也当decoder，llm完全没参与第一阶段。第二阶段llm参与，训练projector和q-former，这里和llava差不多，多了个q-former attention。后面llava也证明了不需要阶段1

### flamingo
> 是多模态领域的 GPT-3。它不追求“微调后有多强”，而是追求“不微调也能看懂图文交织的长文章” (Few-Shot / Zero-Shot)。

类似t5这种encoder，decoder结构，encoder是vit，decoder是llm。和llava的区别是，他的vision tokens没有输入llm，而是通过perceiver + cross attention的方式注入，perceiver其实就是没有多余loss的q-former，为了不破坏llm一开始的参数，cross attention的注入门控tanh 初始化都是0.

论文用了大量interleaved数据，DeepMind 爬了 4300 万个网页。这是 Flamingo 能做 In-Context Learning 的关键。它见惯了“图-文-图-文”的排版，所以你给它几个 `Example Image -> Answer` 的例子，它立马就能学会。
