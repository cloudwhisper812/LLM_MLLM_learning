## LoRA

### 原理
- ΔW = B⋅A，训练时只训练 A, B。A 用高斯分布初始化，B 是 0。优点：显存占用低，无推理延迟（训练完把 B⋅A 直接加回 W₀），便于切换，比如 100 个任务有一百个 LoRA A/B 权重（一个写诗，一个写代码，一个做客服)。
- 如果 A, B 都用高斯分布？→ 模型一开始就“偏移”了预训练权重，Loss 会瞬间飙升，很难训练。如果 A, B 都用 0？→ 梯度无法回传（Symmetry Breaking 问题），A, B 永远学不到东西。
- 目前 LoRA 针对的是 Transformer 里面的各种全连接层


### Q-LoRA
https://github.com/cloudwhisper812/LLM_MLLM_learning/blob/main/%E7%BB%8F%E5%85%B8%E8%AE%BA%E6%96%87/Qlora.md
