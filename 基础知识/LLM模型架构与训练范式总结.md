#### LLM 基础模型架构与训练范式总结
> ### 🚀 核心观点：架构选型与“大力出奇迹”
>
> *   **同等规模下的架构特长**： 在中小参数规模（如 <10B）下，架构决定上限。
> *   **当下的“大一统”趋势**：：Decoder-only 一统江湖，通过堆砌算力、数据和参数量（>100B），Decoder-only 模型涌现出了强大的通用能力，用“高智商”弥补了架构上的“偏科”。目前开源社区（HuggingFace, vLLM, Ollama）对 Decoder-only 的支持最完善，微调和部署成本最低，主流基座Qwen，DeepSeek，Llama，Gemma。


##### 1. Encoder-only 架构
- 代表模型：BERT, RoBERTa, BGE (Embedding 模型)
- 核心特点：双向注意力 (Bidirectional Attention)。每个 token 都能看到上下文所有 token，适合需要全局理解的任务（如分类、实体识别、语义匹配）。
- BERT 训练 (MLM)：
  - Masked Language Modeling：随机 Mask 15% 的 token（如 `A [MASK] C D`）。
  - 目标：利用上下文预测被 Mask 的词（预测 `B`）。
  - 输出：在 Encoder 的输出层，直接对 Mask 位置的向量做分类预测。
  - 优势：双向视野带来了极强的上下文理解能力，是判别式任务（Discriminative Tasks）的王者。
- BGE 训练 (RetroMAE)：
  - 动机：BERT 的 `[CLS]` 向量原生质量不高，需要特化训练以适用于检索。
  - 预训练 (RetroMAE)：引入一个轻量级 Decoder（仅用于预训练，推理时丢弃）。
    - Encoder：输入被中度 Mask 的句子，输出 `[CLS]` 向量 h。
    - Decoder：输入 h + 被重度 Mask 的句子，尝试还原原句。
    - 关键点：Decoder 只能通过 h 获取语义信息，强迫 Encoder 将全句语义极致压缩进 h。
  - 微调 (Contrastive Learning)：使用有监督数据（Query-Passage 对）做对比学习，并且找小模型粗召+reranker大模型打高分的case作为hard negative，也可理解为reranker来蒸馏了。
##### 2. Encoder-Decoder 架构
- 代表模型：T5, BART, UL2, BLIP (早期的多模态)
- 核心特点：Cross-Attention (交叉注意力)。Encoder 负责理解（双向），Decoder 负责生成（单向）。Decoder 的每一层都会通过 Cross-Attention “回头看” Encoder 的完整输出。
- T5 训练 (Span Corruption)：
  - Mask 策略：随机 Mask 掉 15% 的连续片段 (Span)（而非独立的 token），用哨兵符（如 `<extra_id_0>`）占位。
  - 目标：Decoder 自回归地生成被 Mask 掉的内容（如 `<extra_id_0> content <extra_id_1>`）。
  - 优势：
    - Seq2Seq 任务：在翻译、摘要、文本纠错等任务上，由于 Cross-Attention 的存在，Decoder 能时刻对齐源文本，生成内容的忠实度 (Faithfulness) 极高。
    - 多模态 (BLIP)：Vision Encoder 提取图像特征，Text Decoder 通过 Cross-Attention 注入图像信息，实现 Image Captioning。
- 对比 LLaVA (Decoder-only)：
  - T5/BLIP：层层交互。Decoder 每一层都在看 Encoder 的输出。
  - LLaVA：一次性注入。Vision Encoder 的特征仅在输入层（Input Embedding）拼接到 Text Embedding 前面，后续全是 Self-Attention。
  - 区别：T5 结构对细节控制更精准（适合 OCR/定位），LLaVA 结构泛化性更强（适合聊天/推理）。
##### 3. Decoder-only 架构
- 代表模型：GPT 系列, Llama, Qwen, DeepSeek
- 核心特点：Causal Attention (因果注意力)。每个 token 只能看到自己之前的 token（单向）。
- 训练 (CLM)：
  - Causal Language Modeling：预测下一个 token（Next Token Prediction）。
  - 优势：
    - Scaling Law：结构简单，并行效率高，最适合堆算力和数据。
    - 涌现能力：当参数量和数据量达到一定规模，展现出强大的通用推理和零样本能力。
    - 统一范式：通过 Prompt Engineering，可以用生成任务解决几乎所有 NLP 问题（分类、翻译、摘要等）。


> ### 💡 进阶思考与架构辨析
>
> #### 1.Encoder-Decoder进化史：家族的三部曲
> *   **BART (探索者)**：**“试错与验证”**
>     *   **核心贡献**：证明了 Encoder-Decoder 架构在生成任务（如摘要、翻译）上的巨大潜力。
>     *   **训练策略**：**去噪自编码 (Denoising Autoencoder)**。尝试了“句子打乱、删除、旋转、遮罩”等多种噪声还原任务。
>     *   **局限**：任务虽多但杂，后续证明除了 **Span Masking** 外，其他花哨操作（如旋转）收益甚微。
>
> *   **T5 (精简者)**：**“做减法，极致专一”**
>     *   **核心贡献**：确立了 **Text-to-Text** 的统一范式，成为 NLP 领域的瑞士军刀。
>     *   **训练策略**：**Span Corruption (连续片段遮罩)**。吸取 BART 的教训，只保留最有效的“填空”任务，并做到极致。
>     *   **侧重**：理解与短生成能力极强，但在长文本续写（GPT 模式）上略显吃力。
>
> *   **UL2 (集大成者)**：**“做加法，全能统一”**
>     *   **核心贡献**：解决了“T5 不擅长续写，GPT 不擅长理解”的矛盾，是 Google 在 PaLM 之前的终极优化。
>     *   **训练策略**：**Mixture of Denoisers (MoD)**。
>         1.  **R-Denoiser** (像 T5)：练理解。
>         2.  **S-Denoiser** (像 GPT)：练续写。
>         3.  **X-Denoiser** (极端遮罩)：练长文。
>     *   **侧重**：通过 **Mode Token** 灵活切换任务模式，实现了理解与生成的完美统一。
>
> #### 2. 数据对比：Mask 比例与 Decoder 规模
> *   **Mask 比例**：
>     *   **BERT**：**15%** (Token-level)。其中 80% 替换为 `[MASK]`，10% 随机词，10% 保持不变。
>     *   **T5**：**15%** (Span-level)。虽然遮罩的总 Token 数约占 15%，但它是按片段（平均长度 3）来遮罩的。
>     *   **BGE (RetroMAE)**：
>         *   **Encoder 端**：**30%** (Mask Ratio)。为了强迫 Encoder 提取语义，遮罩比例比 BERT 高。
>         *   **Decoder 端**：**50% - 70%** (Aggressive Masking)。Decoder 只能看到极少的词，必须依赖 Encoder 的 `[CLS]` 向量来还原句子。
> *   **Decoder 层数**：
>     *   **T5**：**与 Encoder 层数相同**（例如 T5-Base 是 12 层 Encoder + 12 层 Decoder）。它是“对称”的。
>     *   **BGE (RetroMAE)**：**极浅 (Shallow Decoder)**。通常只有 **1 层** 或 **2 层**。
>         *   **原因**：Decoder 只是为了辅助 Encoder 训练，推理时直接丢弃。如果 Decoder 太强（层数太多），它自己就能把句子补全了，Encoder 就“偷懒”不学语义了。
>
> #### 3. 深度思考：Scaling Law 的偏爱
> *   **结论**：Scaling Law **理论上适用所有架构**（T5 也有 Scaling Law 论文），但 **Decoder-only 的“性价比”最高**。
> *   **原因**：
>     1.  **训练效率**：Decoder-only (GPT) 是单向注意力，计算 Attention 矩阵时可以利用 **Causal Mask** 极大地优化并行计算（FlashAttention 对其优化最极致）。Encoder-Decoder 涉及 Cross-Attention，计算量和显存占用都更大（KV Cache 翻倍）。
>     2.  **数据利用率**：GPT 的任务是 **Next Token Prediction**，这意味着**每一个 Token 都是训练样本**（预测下一个）。BERT 的 MLM 只能利用 15% 的 Token 进行反向传播。T5 虽然也是生成，但输入部分的 Token 不产生 Loss。
>     3.  **工程上限**：在千亿参数（100B+）级别，Decoder-only 架构的**稳定性**最好，不容易梯度爆炸或坍塌。Google 的 PaLM (Decoder-only) 证明了这一点，而 T5 到了 11B 之后再往上扩就很难了。
>
> #### 4. 观点辨析：涌现能力 (Emergent Abilities) 的归属
> *   **事实**：涌现能力（如 In-Context Learning, Chain-of-Thought）确实**主要在 Decoder-only 架构的大模型（>10B/60B）上被观察到**。
> *   **T5/BERT 为什么很少提？**
>     *   **BERT**：它是判别式模型，天生不会“生成”，更别提“思维链推理”了。它只能做分类、抽取。
>     *   **T5**：虽然是生成式，但 T5 的训练目标（Span Corruption）让它更倾向于“填空”和“翻译”，而不是“自由推理”。
>     *   **关键点**：涌现能力似乎与 **Causal Language Modeling (Next Token Prediction)** 这个训练目标强相关。模型为了预测下一个词，被迫学会了逻辑推理、算术、常识等“世界知识”。
> *   **特例**：**UL2** 和 **Flan-T5** 在经过大规模指令微调（Instruction Tuning）后，也展现出了类似的涌现能力（如 CoT），证明了**架构不是绝对壁垒，数据和训练目标才是关键**。但 Decoder-only 依然是目前挖掘这种能力最高效的容器。
