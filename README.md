# LLM-MLLM-learning

主要分为两块：**基础知识**（架构、训练范式、位置编码等）和**经典论文**。学习方式主要是看论文 + 和大模型交互问答，这里是自己总结的一些关键点。
---

## 一、基础知识

| 主题 | 内容 |
|------|------|
| [01. LLM模型架构与训练范式总结](基础知识/01.%20LLM模型架构与训练范式总结.md) | Encoder-only (BERT, BGE), Encoder-Decoder (T5, BART, UL2), Decoder-only (GPT, Llama) |
| [02. MLLM架构](基础知识/02.%20MLLM架构(llava,%20blip,%20flamingo,%20end2end).md) | LLaVA, BLIP (Q-Former, CapFilt), Flamingo (Perceiver, interleaved) |
| [03. activation function](基础知识/03.%20activation%20function.md) | sigmoid, tanh, relu, GeLU, Swish, SwiGLU |
| [04. positional embedding](基础知识/04.%20positional%20embedding.md) | Sinusoidal PE, Learnable PE, RoPE, Alibi |
| [05. normalization](基础知识/05.%20normalization.md) | LN vs BN, Pre-Norm vs Post-Norm, RMSNorm |
| [06. scaling law](基础知识/06.%20scaling%20law.md) | Power Law, Chinchilla, D≈20N, C≈6ND |
| [07. MoE](基础知识/07.%20MoE.md) | Router, Experts, Load Balancing, Top-K, Expert Capacity |

---

## 二、经典论文

| 主题 | 内容 |
|------|------|
| [stable diffusion + DIT](经典论文/stable%20diffusion%20+%20DIT.md) | LDM, VAE, U-Net, DiT |
| [siglip 1 and 2](经典论文/siglip%201%20and%202.md) | 二分类, MAP, NaViT, 自蒸馏, LocCa |
