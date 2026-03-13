## MLLM 架构
> BLIP, Flamingo基本不用了，被LLaVA统治了。LLaVA足够简单和暴力，scaling law的胜利，LLM够大，数据够多。
> 所以这里主要回顾讲解LLaVA，BLIP/Flamingo重点强调主要思想，他们的思想现在还是在特定领域应用的。
> 貌似现在闭源的大模型都是end2end了

### LLaVA
#### 1. 模型结构
Vision Encoder + Projector（两层全连接） + LLM。image 过 Vision Encoder 和 Projector，得到 vision tokens 和 text tokens 拼到一起输入到 LLM。 
anyres：原始大图 resize 小图输入 ViT 提供全局信息，大图 crop 局部输入 ViT 提供局部信息，两者的 tokens 拼到一起。但是现在 SigLIP 2 都直接用 NaViT 了，大力出奇迹。

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


### BLIP
#### BLIP1
1. encoder，decoder结构。感觉这个结构太复杂，没人用了，下次忘了可以gpt一下，对着论文看看。不过这个结构还挺有意思的，参数共享后，有一个双塔的粗召回ITC, 又有一个精排的ITM。
2. 论文提出的数据清洗也是重点，CapFilt (Captioning and Filtering)，少量高质量的人工标注数据得到filter和captioner：（1）Filter (过滤器)：训练一个 ITM 模型（看图文匹不匹配）。如果网上的 Caption 和图不匹配，直接扔掉。 （2）- Captioner (生成器)：训练一个 LM 模型（看图说话）。给网上的图重新生成一个高质量的 Caption（合成数据）。 （3）Bootstrap：用清洗后的数据 + 生成的数据，重新训练一个更强的 BLIP。这个数据量大于一开始的高质量的人工标注数据。（ **现在有更好的方法用gpt精标数据，训练student model来清洗数据生成caption，BLIP-1 captioner用coco之类训练的生成风格呆板，但是stutent gpt model因为来自gpt蒸馏，生成风格详细生动。**）

#### BLIP2
提出了 Q-Former，其实就是 DETR 里面的 learnable query，也是 SigLIP 里面的 MAP。两阶段训练 loss 也有点意思，第一阶段还保留了 BLIP1 的 loss，把 Q-Former 当 text encoder，也当 decoder，LLM 完全没参与第一阶段。第二阶段 LLM 参与，训练 Projector 和 Q-Former，这里和 LLaVA 差不多，多了个 Q-Former attention。后面 LLaVA 也证明了不需要阶段 1

### Flamingo
> 是多模态领域的 GPT-3。它不追求“微调后有多强”，而是追求“不微调也能看懂图文交织的长文章” (Few-Shot / Zero-Shot)。

类似 T5 这种 encoder-decoder 结构，encoder 是 ViT，decoder 是 LLM。和 LLaVA 的区别是，它的 vision tokens 没有输入 LLM，而是通过 Perceiver + cross attention 的方式注入，Perceiver 其实就是没有多余 loss 的 Q-Former。为了不破坏 LLM 一开始的参数，cross attention 的注入门控 tanh 初始化都是 0。

论文用了大量interleaved数据，DeepMind 爬了 4300 万个网页。这是 Flamingo 能做 In-Context Learning 的关键。它见惯了“图-文-图-文”的排版，所以你给它几个 `Example Image -> Answer` 的例子，它立马就能学会。
