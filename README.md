# LLM/MLLM learning笔记

[![Commit Activity (week)](https://img.shields.io/github/commit-activity/w/cloudwhisper812/LLM-MLLM-learning/main?label=commits%2Fweek)](https://github.com/cloudwhisper812/LLM-MLLM-learning)
[![Commit Activity (month)](https://img.shields.io/github/commit-activity/m/cloudwhisper812/LLM-MLLM-learning/main?label=commits%2Fmonth)](https://github.com/cloudwhisper812/LLM-MLLM-learning)


目前笔记主要分为三块：**个人总结**、**基础知识**（和**经典论文**（后续计划增加大模型 coding 部分）。学习方式主要是看论文 + 和大模型交互问答，以个人理解为主，不拘泥于公式推导，旨在遗忘时能快速召回、温故知新。


## 一、个人总结

| 主题 | 内容 |
|------|------|
| [01. LLM模型架构与训练范式总结](个人总结/LLM模型架构与训练范式总结.md) | Encoder-only (BERT, BGE), Encoder-Decoder (T5, BART, UL2), Decoder-only (GPT, Llama) |
| [02. MLLM架构](个人总结/MLLM架构(LLaVA,%20BLIP,%20Flamingo,%20end2end).md) | LLaVA, BLIP (Q-Former, CapFilt), Flamingo (Perceiver, interleaved) |
| [03. LLM/MLLM数据收集和训练流程总结](个人总结/LLM_MLLM数据收集和训练流程.md) | LLM/MLLM Data Collection and Training Pipeline |

---

## 二、基础知识

| 主题 | 内容 |
|------|------|
| [01. Activation Function](基础知识/Activation%20Function.md) | sigmoid, tanh, relu, GeLU, Swish, SwiGLU |
| [02. Positional Embedding](基础知识/Positional%20Embedding.md) | Sinusoidal PE, Learnable PE, RoPE, Alibi |
| [03. Normalization](基础知识/Normalization.md) | LN vs BN, Pre-Norm vs Post-Norm, RMSNorm |
| [04. Scaling Law](基础知识/Scaling%20Law.md) | Power Law, Chinchilla, D≈20N, C≈6ND |
| [05. MoE](基础知识/MoE.md) | Router, Experts, Load Balancing, Top-K, Expert Capacity |
| [06. Optimizer](基础知识/Optimizer.md) | Momentum, adaptive learning rate, adam, adamW |
| [07. KV Cache](基础知识/KV%20cache.md) | 显存计算, MHA/MQA/GQA, PagedAttention |
| [08. Flash Attention](基础知识/FlashAttention.md) | IO bound 问题, 块化到 SRAM, online softmax |
| [09. LoRA](基础知识/LoRA.md) | Low-rank adaptation, 参数高效微调, 及 Q-LoRA 相关笔记 |
| [10. RAG](基础知识/RAG.md) | 幻觉与时效性, chunking→embedding→索引→ANN+重排→生成 |
| [11. Decoding Strategy](基础知识/Decoding%20Strategy.md) | Greedy, Beam Search, Sampling, Top-k/top-p, Temperature |
| [12. Tokenizer](基础知识/Tokenizer.md) | subword 动机, BPE 工作机制, special tokens 与 embedding |
| [13. Quantization](基础知识/Quantization.md) | NF4, QLoRA, GPTQ/AWQ 等主流量化方案 |
| [14. Distributed Training](基础知识/Distributed%20Training.md) | 数据并行/张量并行/流水线并行, 通信与显存 |
| [15. RL](基础知识/RL.md) | 大模型对齐中的 PPO, DPO, GRPO |

---

## 三、经典论文
参考：https://github.com/Hannibal046/Awesome-LLM https://github.com/friedrichor/Awesome-Multimodal-Papers
| 主题 | 内容 |
|:-----|:-----|
| **1. Milestone Papers** | |
| 1. [QLoRA](经典论文/QLoRA.md) | QLoRA: 量化后的高效微调, NF4+LoRA 的整体设计 |
| 2. [ReAct](经典论文/ReAct.md) | Synergizing Reasoning and Acting |
| 3. [AWQ(Activation-aware Weight Quantization)](经典论文/AWQ(Activation-aware%20Weight%20Quantization).md) | Activation-aware Weight Quantization |
| **2. MLLM Papers** | |
| 1. [Janus and Janus-Pro](经典论文/Janus%20%20and%20Janus-Pro.md) | 视觉理解/生成双编码器解耦 + 统一 LLM 融合（SigLIP + VQ Tokenizer） |
| 2. [Stable Diffusion + DiT](经典论文/Stable%20Diffusion%20+%20DiT.md) | LDM, VAE, U-Net, DiT |
| 3. [SigLIP 1 and 2](经典论文/SigLIP%201%20and%202.md) | 二分类, MAP, NaViT, 自蒸馏, LocCa |



