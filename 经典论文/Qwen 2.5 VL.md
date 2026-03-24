# Qwen 2.5 VL

## 模型结构
#### LLM
1. LLM用Qwen2.5 LLM pre-trained weights.
2. Multimodal Rotary Position Embedding
    - LLM原本的1D RoPE 改成 M RoPE.
    -  For textual inputs, all three components use identical position IDs, making M-RoPE functionally equivalent to traditional 1D RoPE. For static images, the temporal ID remains constant (e.g., 0) across all visual tokens, while unique IDs are assigned to the height and width components based on each token’s 2D spatial position. When processing videos, the spatial components (height and width) follow the same assignment pattern as static images. However, instead of simply incrementing temporal IDs frame-by-frame, Qwen2.5-VL aligns the temporal ID with absolute physical time (i.e., actual timestamps). This allows the model to capture the true speed of content changes and maintain consistent temporal alignment regardless of the dynamic FPS sampling rates.
    - 我的理解其实就是把token切成三份，每份对应一个1d-rope，他们的index分别是time，h，w。然后对text，image，video分别采用上面提到的处理方式。
3. Dynamic FPS: 针对长视频，模型不再采用固定的帧率进行抽取，而是可以根据视频的动态变化自适应调整采样帧率。Vision Encoder 只是一个“无情”的特征提取机器。LLM 通过 M-RoPE 获取了这些视觉 Token的1D 时间坐标。
4. 特殊 Token 的插入： Vision Encoder 压缩后的 Token 序列，在输入给 LLM 之前，会在序列的两端和内部插入特定的控制 Token。例如单张图片使用 <|image_pad|>，视频使用 <|video_pad|>，并用 <|vision_start|> 和 <|vision_end|> 包裹。

#### Vision Encoder
1. 没有使用开源的siglip之类的模型，重新设计了一个ViT. 采用2D-RoPE。并且用RMSNorm, SwiGLU，对齐了LLM。
2. 采用window attention，window size是 112 x 112， 正好对应8 x 8个patch。ViT的大部分层只在window内做attention，只有部分层做所有patch的global attention。这样大量减少了计算量。
3. MLP-based Vision-Language Merger，其实就是token compression。这里每2 x 2个  ViT的输出token concatenate乘一个，然后过两层MLP。这里就是每4个tokens，压缩成一个token，大大减少计算量。
4. Native Dynamic Resolution：这个老生常谈了，就不复述了。
5. 输入的size是[T=2, H, W, C], 然后用2x14x14xd 的3d卷积，stride是（2，14， 14），这里其实等于全连接mapping成token输入ViT.对于video，所以是每两个frame输入一次。对于单张图，会被copy一次。

## 预训练
#### 数据 4T
1. 图文穿插数据 (Interleaved Data) 的“去噪与清洗”：网上的图文穿插数据通常毫无逻辑（图片仅仅是网页装饰），缺乏真正的关联。自研了一个基于内部模型的 4 阶段打分系统。不仅看“文本质量”和“图文相关性”，还特别引入了图文互补性和信息密度平衡。这意味着模型只吃那些“缺了图看不懂文字，缺了字看不懂图”的高质量关联数据。
2. Grounding 数据：全面拥抱“绝对坐标”：训练时直接使用基于输入图像真实像素尺寸的绝对坐标。并且为了提升泛化性，大量使用 Copy-Paste、Grounding DINO 和 SAM 合成了海量的罕见类别（10k+ 类）甚至“不存在的物体”，强迫模型去真正理解视觉特征而不是死记硬背常见物体。
3. 文档解析 (Document Omni-Parsing)：万物皆可 HTML。传统文档 OCR 是一条长长的 Pipeline（版面分析 -> 文本提取 -> 图表模型 -> 公式模型），极易产生误差级联。解法：极其粗暴且优雅，直接把所有元素（文本、段落、表格、图表、甚至乐谱和化学式）全部统一格式化为带有 data-bbox 坐标的 HTML 标签。让模型在预训练阶段就把排版逻辑和视觉内容通过 HTML 树状结构强绑定。
4. 视频数据 (Video Data)：配合架构的动态采样。在训练时动态采样 FPS，让模型见识各种帧率。同时针对半小时以上的长视频，专门合成了长视频 Caption。时间戳不仅有秒级的，还有 hmsf（时分秒帧）格式，强迫模型学会各种时间表达。
5. Agent 数据：强迫模型“先思考，再行动”。把移动端、Web 端、桌面的 UI 操作全部统一为 Function Call。最关键的是，没有简单地让模型去拟合 $(x,y)$ 点击坐标，而是引入了人工和模型生成的 Reasoning（推理过程）。给模型看操作前后的截图，让它解释“为什么要点这里”。这有效防止了模型对特定 UI 布局的过拟合。

#### 训练流程
    ViT做了clip一样的预训练，原文一笔带过，没有做siglip这种精细化训练原因是它把这种精细化的复杂任务融合在了后续的end2end训练中。
    
    step 1: 冻结 LLM，只训 ViT以及 MLP Merger。图像 Caption、OCR、视觉基础知识。让视觉特征先“翻译”成 LLM 能听懂的语言。
    
    step 2: 全参数解冻。图文穿插、VQA、多模态数学、纯文本等。
    
    step3: 全参数解冻。 视频数据、Agent 数据、长序列数据。


    step2,3为什么分开： 一方面从简单到难，另一方面序列长度从短到长，节省里训练资源。
    
    这里的step1类似llava的alignment。2和3仍然是pretraining，因为他们的loss是所有text token上的，而不是只有answer。


## 个人一些补充总结
1. 关于 Grounding DINO 和 SAM 怎么拿到 rare/non-existent objects 的 label
   - Grounding DINO 的机制： Grounding DINO 不是传统的闭集检测器，它是语言驱动的零样本检测器。它的输入不仅是一张图，还有一段 Text Prompt。模型通过对比学习对齐了图像特征和文本特征。只要你的 Text 包含这个罕见词汇，它就能通过跨模态注意力机制把对应的区域框出来。
   - 文本生成图像 (T2I/Diffusion)： 先用类似 Midjourney/Stable Diffusion 这样的文生图模型，输入 Prompt（比如：“一只长着三只翅膀的紫色机械猫”）。这就凭空创造了一个“不存在的物体”。
   - 获取精确 Mask： 把这张图和 Prompt 输入给 Grounding DINO，DINO 依靠其 Open-Vocabulary 能力框出这个大体位置；然后把这个 Bounding Box 作为 Prompt 喂给 SAM，SAM 输出像素级完美的 Mask。
   - Copy-Paste 增强： 把 SAM 抠出来的这个“紫猫”抠图（Foreground），随机粘贴（Copy-Paste）到各种真实的背景图片上。
   - Label 的来源： 因为是我们主动把这个带有明确语义（紫猫）的像素块贴到新图的 $(x, y)$ 位置的，所以它的 Class Label 和 Bounding Box 坐标在合成的这一瞬间就是100% 确定且已知的
   - 启示：自然界的数据是长尾的。Qwen 团队为了拉满 Open-vocabulary 的能力，故意合成“不存在的类别”塞进 Query 里，并在单图里疯狂塞入 multiple instances。这种在 Pre-training 阶段就人为制造“困难场景”的 Data Synthesis 策略，是提升基础底座鲁棒性的不二法门。
2. Agent 数据中的“强迫思考（Reasoning）”是什么意思？
    - 传统做法的弊端： 以前的训练数据是 (输入：当前屏幕截图 + 任务 "买咖啡", 输出：点击坐标 (100, 200))。模型根本不知道为什么要点这里，它只是通过海量数据死记硬背了“当屏幕长这样，且看到‘买’字，就点右下角的某个形状”。一旦按钮换了位置，或者 UI 换了皮肤，模型直接崩溃。
    - Qwen2.5-VL 的“先思考，再行动”数据构造（引入 Chain-of-Thought）：团队在造数据时，没有直接让模型去拟合坐标。他们把操作前的截图、操作后的截图以及用户的真实点击动作拿出来，雇佣数据标注员（或使用更强大的模型，如 GPT-4V）来写推理过程（Reasoning）。
    - 具体的输入输出形式：
    
        User 输入： “帮我在美团买一杯星巴克拿铁。” + 当前屏幕截图。
        
        Assistant 输出（必须先输出推理，再输出动作）：
        “(思考过程) 用户想要买拿铁，当前页面是星巴克主页，我需要先找到搜索框或者咖啡分类。我观察到屏幕中间偏上有一个‘咖啡’的类别图标，位于坐标 [x1, y1, x2, y2]。我应该点击这个图标进入列表。(执行动作) Function Call: Click([x1, y1, x2, y2])。”
