## LLM
### 1. Pre-training 阶段
数据：
- 海量无标签语料（几 T 到几十 T tokens）。包括网页抓取（CommonCrawl）、高质量书籍、论文，以及极高比例的代码 (Code) 和数学 (Math) 数据。
- 数据处理：
  <details>
    <summary>文本提取与清洗 (Text Extraction)</summary>
    将 HTML 网页还原为干净的、带有正确换行逻辑的文本，这是保住代码和数学公式格式的生死线。最主流方法： Resiliparse 或 Trafilatura。过去用简单的正则或 BeautifulSoup 剥离 HTML 标签，会导致段落粘连。现在的主流方案是基于 DOM 树结构的特征（如文本节点密度、标签比例）来识别正文区域。对于包含 pre 或 code 标签的区域，必须保留其原始的空格和缩进，否则模型根本学不会 Python。
  </details>
  <details>
    <summary>语种识别</summary>
    过滤掉不需要的语种，或者为后续的语种配比（Data Mix）打标签。极其轻量级的 $N$-gram 线性分类器。为了极致的性能，通常会用 C++ 直接绑定在数据流处理框架中，单 CPU 核心每秒能处理上万条文档。
  </details>
  <details>
    <summary>启发式规则过滤 (Heuristic Filtering)</summary>
    用计算代价极低的规则，快速筛掉极其劣质的“垃圾堆”（如日志文件、SEO 占位符、乱码）。最主流方法： 借鉴 MassiveText (Gopher) 或 RedPajama 的规则集。核心细节： 常见规则包括：字母/数字字符比例（Alphanumeric ratio）：剔除全是符号的乱码。停用词密度（Stop-word density）：正常的自然语言一定会包含一定比例的 "the", "is", "的", "了"。如果不包含，极大概率是名词堆砌的 SEO 网页或机器生成的垃圾。平均单词/句子长度：过长或过短都直接丢弃。
  </details>
  <details>
    <summary>模糊与精确去重 (Deduplication)</summary>
    模糊去重 MinHash + LSH (Locality Sensitive Hashing)。提取文档的原 N-gram，用多个 Hash 函数计算 MinHash 签名，再通过 LSH 分桶。由于处理的是 PB 级数据，这部分必须在 Spark 或 Ray 集群上做大规模分布式 MapReduce 计算。精确/子串去重 Suffix Array (后缀数组)。不仅去重整篇文档，还要把跨文档的重复片段（如网站底部的版权声明、开源协议模板）挖掉。通过构建全局的 Suffix Array，可以找到语料库中所有长度大于 $k$（如 50 个 Token）的重复子串并将其 Mask 掉。
  </details>
  <details>
    <summary>质量过滤 (Quality Filtering / Model-based)</summary>
    在规则过滤之后，使用模型来判断文章内容的“知识密度”和“逻辑性”。最主流方法： FastText 质量分类器 或 小模型 PPL (Perplexity) 截断。分类器流派： 以维基百科、高质量书籍和论文作为正样本（Label=1），随机 CommonCrawl 网页作为负样本（Label=0），训练一个 FastText 二分类器。给所有网页打分，只保留预测概率大于设定阈值（如 0.6）的数据。PPL 流派： 训练一个非常小（如 100M-1B 参数）的语言模型，或者直接用现成的小模型跑一遍推理，计算整篇文章的困惑度。困惑度太高的（语句不通顺）或太低的（车轱辘话反复说）都会被过滤。
  </details>
  <details>
    <summary>Benchmark 去污染 (De-contamination)</summary>
    防止测试集（如 MMLU, GSM8K, HumanEval）泄漏到训练集中，保证评测的客观性。最主流方法： 精确 N-gram 匹配 + Bloom Filter。通常取 $N=13$ 或 $N=15$。考虑到评测集的 Prompt 可能被各种魔改，还会去掉标点和空格进行归一化匹配。为了加速查找，会先将所有评测集的 N-gram 存入一个巨大的 Bloom Filter 中。数据流过时，一旦在 Bloom Filter 中命中，再进行精确核对，若命中率超过阈值则整篇丢弃或进行截断。
  </details>
  <details>
    <summary>隐私脱敏 (PII Removal)</summary>
    移除敏感个人信息（Personally Identifiable Information）。 最主流方法： 高度优化的正则表达式引擎 (如 RE2) + 局部 NER 模型。核心细节： 大规模正则匹配极其消耗 CPU。工业界通常不用 Python 原生的 re 模块，而是使用 Google 开发的 RE2（基于确定性有限状态自动机 DFA），保证匹配时间的线性复杂度，避免正则回溯引发的 CPU 灾难。对于身份证号、邮箱、电话、IP 地址用正则；对于某些特定的敏感人名或机构，可能会辅以轻量级的命名实体识别（NER）模型，将它们替换为 &lt;|EMAIL|&gt;、&lt;|PHONE|&gt; 等占位符。
  </details>


训练：
- Loss 是在每一个 Token 的位置上都进行计算的，不区分 Question 和 Answer。
- 为了 GPU 利用率，通常会将多篇文章拼接到一个最大上下文窗口（如 4k 或 8k），中间用特殊的 <|endoftext|> token 隔开。这里的关键是Attention Mask 的处理，有时会采用 Document Masking 防止跨文章的注意力污染。
- Data Annealing (数据退火)：在预训练的最后 5%~10% 阶段，大幅降低学习率，并极大地提升高质量数据（如精选维基、教科书、高质量代码）的采样权重。这能让模型性能产生一次跃升。

### 2. SFT/Instruct Tuning(指令微调)
主要目的：将“续写机器”变成“对话助手”，学习指令的格式和拒绝策略（Alignment Tax 往往在这里产生）。

数据：
- 几万到几十万条高质量的 (Prompt, Response) 对。数据量不大，但对质量要求极度苛刻。核心目的是防止模式崩溃（Mode Collapse），即模型只会用一种语气回答问题。
- System Prompt 鲁棒性：训练数据中需要混入各种 System Prompt，甚至刻意加入带有约束条件的复杂指令（Evol-Instruct）。
- 语义聚类与多样性采样 (Diversity Routing)： 用一个 Embedding 模型（如 BGE 或 OpenAI 的 API）把所有指令转成高维向量。然后使用 K-Center Greedy 算法或 HDBSCAN 进行聚类。在每个簇（Cluster）中，根据预先训练的质量打分模型（Reward Model）挑出分数最高的 Top-K 条。这样既保证了质量，又保证了指令在语义空间上的均匀覆盖。
- 格式标准化 (Formatting & Chat Template)：将多轮对话严格转化为 System、User、Assistant 的结构，并注入特殊的控制符号（如 <|im_start|> 和 <|im_end|>）。
- 数据收集流程（模型越强，人工比例越高： 越是头部的模型，越依赖高质量的人工干预来纠正细微的偏置。）：
  - 人工精造 (Human Annotation) —— 占比约 10%-20%
  - 存量语料转换 (Corpus Conversion) —— 占比约 30%： 将现有的高质量结构化数据“翻译”成对话，考试题库、维基百科词条、GitHub 代码段。
  - 模型自生成 (Synthetic Data / Distillation) —— 占比约 50% 以上：
    - 1. 种子指令采样： 从人工精造的 2000 条种子数据中采样 5 条。
    - 2. In-Context Learning 生成： 把这 5 条喂给最强的模型（如 GPT-4o 或自家上一代最强模型），让它模仿风格生成 50 条新指令。
    - 3. Evol-Instruct 变异： 对生成的指令进行上述的“复杂度进化”。
    - 4. Response 生成： 再次调用最强模型对这些进化后的指令给出回答
 
训练：
- Loss Masking (掩码损失)：在计算 Cross-Entropy Loss 时，只对 Response 部分计算梯度，Prompt 部分的 Loss 被 Mask 掉。模型不需要学习去预测用户的提问。

### 3. RL / Alignment (强化学习与对齐)
主要目的：对齐人类偏好，解决 SFT 中“模型幻觉”或“顺从用户产生有害内容”的问题。

数据：
- 偏好数据集。对于一个 Prompt，提供模型的两个不同回答 $y_{chosen}$ 和 $y_{rejected}$。无论是 PPO 还是 DPO，模型学习的上限完全取决于你喂给它的 Chosen（好回答）和 Rejected（差回答）的质量。
- 构建“困难负样本” (Hard Negatives)：故意让模型在一个复杂的推理步骤中错一小步（比如数学计算中间的符号写反，或者代码里的一个变量名用错），将其作为 Rejected。这种高难度的对比数据，能逼迫模型学到真正的逻辑，而不是表面的语法。
- 对抗长度偏见 (Length Bias Mitigation) 痛点：大模型有个臭毛病，喜欢“长篇大论”，往往会认为字数多的回答就是好回答。在构建偏好数据时，刻意筛选出长度相近的 Chosen 和 Rejected 对；或者在 DPO 的 Loss 函数中引入长度惩罚项，强迫模型在相同信息密度下比拼准确率，而不是比拼谁更能“水字数”。
- 对齐（Alignment）”上依赖人，在“能力（Capability）”上脱离人。模型必须符合人类的价值观、语调、安全规范。这部分必须由人（或者代表人类价值观的宪法 RLAIF）来把关。模型 A 生成答案 -> 验证器（基于规则或环境）判断对错 -> 正确路径被加入训练集进行迭代。结果： 这种方式产生的数据质量可以远超任何人类标注员，因为它通过大规模采样（Sampling）和搜索（Search）找到了人类难以想到的最优解。

训练：PPO,DPO,GRPO,etc.

## MLLM （针对LLaVa这种非E2E结构）
### Modality Alignment
这是一个建立映射字典的过程。把图像的 Patch 转换为 LLM 词表空间里的“视觉 Token”。
数据：
- 数亿级别的弱相关图文对（Image-Caption pairs，如 LAION, CC3M）。数据往往较短、较嘈杂。主要是海量的 Web-derived Image-Text Pairs（网络爬取图文对）
- 数据清晰：图文匹配度过滤 (CLIP Score Filtering)。视觉质量与分辨率卡控，去掉低质量和奇怪长宽比。水印和OCR过滤，跑一遍轻量级的 OCR 模型计算文本框面积占比。如果图片上密密麻麻全是字，或者特定区域有高频出现的水印模式，直接过滤。
- 

训练：
- 只训练Projector。
- loss在所有text caption token上。

### Multimodal Instruct Tuning (多模态指令微调)
数据：
- 几十万到上百万的高质量多模态指令数据。包括交错图文（Interleaved Image-Text）、细粒度 OCR 数据、Bounding Box 坐标定位数据（Grounding）。
-自动化高精标注 (The Auto-Labeling Engine)
  - 多专家模型打标： 针对一张图，调用多个专项小模型（检测模型标物体坐标、OCR 模型提文字、深度估计模型提空间关系），然后将这些结构化信息汇总成一段极长的文本，再喂给 GPT-4 或 Gemini，让它总结出一份“金标描述”。
  - Recursive Labeling（递归标注）： 用上一代模型生成描述 -> 人工微调 1% -> 训练分类器过滤掉剩下的 99% 中不合格的描述 -> 循环往复。
- 结构化存量数据转换
  - PDF/文档： 爬取数亿份 PDF，利用解析工具提取图表（Chart）及其对应的说明文字，直接转化为“图表阅读”指令对。
  - 网页数据： 网页中的 HTML 结构自带图片与上下文的关联，利用这种结构化关系自动合成问答。
- 人工众包与“困难样本”打磨
  - 空间关系与计数： 模型最容易数错数或分不清左右。大厂会专门雇人标注这类具有强空间约束的数据。
  - 多轮对话轨迹： 模拟人类在看图时的思维流（例如：“图里那个人拿的是什么？” -> “那这个东西在那个环境下有什么用？”），这种长链条数据目前仍需人工精修。

训练：
- loss只在answer上
