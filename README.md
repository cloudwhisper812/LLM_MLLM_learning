# LLM/MLLM learning笔记

[![Commit Activity (week)](https://img.shields.io/github/commit-activity/w/cloudwhisper812/LLM-MLLM-learning/main?label=commits%2Fweek)](https://github.com/cloudwhisper812/LLM-MLLM-learning)
[![Commit Activity (month)](https://img.shields.io/github/commit-activity/m/cloudwhisper812/LLM-MLLM-learning/main?label=commits%2Fmonth)](https://github.com/cloudwhisper812/LLM-MLLM-learning)


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
| [08. Flash Attention](基础知识/10.%20flash%20attention.md) | IO bound 问题, 块化到 SRAM, online softmax |
| [09. LoRA](基础知识/11.%20LoRA.md) | Low-rank adaptation, 参数高效微调, 及 Q-LoRA 相关笔记 |
| [10. RAG](基础知识/12.%20RAG.md) | 幻觉与时效性, chunking→embedding→索引→ANN+重排→生成 |
| [11. decoding strategy](基础知识/13.%20decoding%20strategy.md) | Greedy, Beam Search, Sampling, Top-k/top-p, Temperature |
| [12. tokenizer](基础知识/14.%20tokenizer.md) | subword 动机, BPE 工作机制, special tokens 与 embedding |
| [13. quantization](基础知识/15.%20quantization.md) | NF4, QLoRA, GPTQ/AWQ 等主流量化方案 |
| [14. Distributed Training](基础知识/16.%20Distributed%20Training.md) | 数据并行/张量并行/流水线并行, 通信与显存 |
| [15. RL](基础知识/17.%20RL.md) | 大模型对齐中的 PPO, DPO, GRPO |

---

## 三、经典论文
参考：https://github.com/Hannibal046/Awesome-LLM https://github.com/friedrichor/Awesome-Multimodal-Papers
| 主题 | 内容 |
|------|------|
| [QLoRA](经典论文/Qlora.md) | QLoRA: 量化后的高效微调, NF4+LoRA 的整体设计 |
| [stable diffusion + DIT](经典论文/stable%20diffusion%20+%20DIT.md) | LDM, VAE, U-Net, DiT |
| [SigLIP 1 and 2](经典论文/siglip%201%20and%202.md) | 二分类, MAP, NaViT, 自蒸馏, LocCa |
