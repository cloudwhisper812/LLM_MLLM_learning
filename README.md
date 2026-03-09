# LLM/MLLM learning笔记

目前笔记主要分为三块：**个人总结**、**基础知识**（和**经典论文**（后续计划增加大模型 coding 部分）。学习方式主要是看论文 + 和大模型交互问答，以个人理解为主，不拘泥于公式推导，旨在遗忘时能快速召回、温故知新。


## 一、个人总结

| 主题 | 内容 |
|------|------|
| [01. LLM模型架构与训练范式总结](基础知识/01.%20LLM模型架构与训练范式总结.md) | Encoder-only (BERT, BGE), Encoder-Decoder (T5, BART, UL2), Decoder-only (GPT, Llama) |
| [02. MLLM架构](基础知识/02.%20MLLM架构(llava,%20blip,%20flamingo,%20end2end).md) | LLaVA, BLIP (Q-Former, CapFilt), Flamingo (Perceiver, interleaved) |

---

## 二、基础知识

| 主题 | 内容 |
|------|------|
| [01. activation function](基础知识/03.%20activation%20function.md) | sigmoid, tanh, relu, GeLU, Swish, SwiGLU |
| [02. positional embedding](基础知识/04.%20positional%20embedding.md) | Sinusoidal PE, Learnable PE, RoPE, Alibi |
| [03. normalization](基础知识/05.%20normalization.md) | LN vs BN, Pre-Norm vs Post-Norm, RMSNorm |
| [04. scaling law](基础知识/06.%20scaling%20law.md) | Power Law, Chinchilla, D≈20N, C≈6ND |
| [05. MoE](基础知识/07.%20MoE.md) | Router, Experts, Load Balancing, Top-K, Expert Capacity |
| [06. optimizer](基础知识/08.%20optimizer.md) | Momentum, adaptive learning rate, adam, adamW |
| [07. KV cache](基础知识/09.%20kv%20cache.md) | 显存计算, MHA/MQA/GQA, PagedAttention |
| [08. Flash Attention](基础知识/10.%20flash%20attention.md) | 切块, Memory bandwidth, Online softmax |

---

## 三、经典论文

| 主题 | 内容 |
|------|------|
| [stable diffusion + DIT](经典论文/stable%20diffusion%20+%20DIT.md) | LDM, VAE, U-Net, DiT |
| [siglip 1 and 2](经典论文/siglip%201%20and%202.md) | 二分类, MAP, NaViT, 自蒸馏, LocCa |
