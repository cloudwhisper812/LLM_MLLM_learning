
#### PPO (Proximal Policy Optimization) 
需要4 个模型（假设都是 7B 大小）。它们的输入输出和状态如下：
- actor model：可训练，输入prompt，输出下一个token的logtis。
- reference model：冻结，sft阶段的模型副本。作为“锚点”。在 Actor 生成每一个 Token 时，Reference 也会算一下概率。如果 Actor 的概率分布偏离 Reference 太远，就会受到 KL 散度惩罚。这是为了防止模型“灾难性遗忘”或变成只会迎合奖励的复读机。
- Reward Model：冻结 (Frozen)（提前用人类偏好数据训练好的）。输入Prompt + Actor 生成的完整 Response。输出：一个标量（Scalar），比如 `+2.5` 或 `-1.2`，代表这句话的整体得分。
- Critic Model (价值模型 / Value Model)：可训练（通常用 Reward Model 初始化，然后跟着 Actor 一起训）。输入：Prompt + Actor 生成的部分 Response（即当前 State）。 输出：一个标量。它预测的是：“在当前这个状态下，未来我预期能拿到多少总奖励？”
