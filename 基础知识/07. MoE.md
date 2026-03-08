### MoE

### 1. 核心架构 (Router + Experts)
- Experts通常是多个 FFN (Feed-Forward Network) 层。比如 8 个 FFN，每个结构一模一样，但参数不同。Router一个小的线性层 + Softmax。输入 Token x，输出每个专家的权重 G(x).
- 流程：Token x 进来。Router 计算权重，只选权重最大的 K 个专家（比如 Top-2）。选择到的专家输出加权求和。虽然有 8 个专家，但每次只算 2 个。

### 2. 负载均衡 (Load Balancing)
- 坍塌 (Collapse) / 赢家通吃 (Winner-take-all)。Router 初始化时，某个专家（比如 Expert 1）的权重稍微大了一点点。Router 发现 Expert 1 训练得好，就拼命把 Token 往它那里发。
- 解决方案：辅助损失 (Auxiliary Loss / Load Balancing Loss): 强迫 Router 把 Token 均匀分配 给所有专家。 (这里公式再看一下)



# 下面是gpt生成的还没来得及看


#### 考点三：专家容量 (Expert Capacity)
这是工程落地的考点。
##### Q3: 什么是 Expert Capacity？如果一个专家被塞爆了怎么办？
- 背景：在分布式训练中，不同的专家可能在不同的 GPU 上。
- Expert Capacity：每个专家在一个 Batch 里能处理的最大 Token 数量。
  - 设定为：Capacity=Tokens per BatchNum Experts×Capacity Factor\text{Capacity} = \frac{\text{Tokens per Batch}}{\text{Num Experts}} \times \text{Capacity Factor}Capacity=Num ExpertsTokens per Batch​×Capacity Factor。
  - Capacity Factor 通常设为 1.1 ~ 1.2（允许稍微超载）。
- Token Dropping (丢弃)：
  - 如果 Router 把太多 Token 发给同一个专家（超过 Capacity）。
  - 多出来的 Token 会被直接丢弃 (Dropped)！
  - 它们不经过 FFN，直接通过残差连接（Residual Connection）流到下一层。
  - 后果：信息丢失，性能下降。
- 所以负载均衡非常重要！ 只有均衡了，才不会有专家撑死，也不会有专家饿死。
#### 考点四：Switch Transformer vs Mixtral (架构演进)
这是考察你对 SOTA 的了解。
##### Q4: Switch Transformer 和 Mixtral 8x7B 有什么区别？
- Switch Transformer (Google, 2021)：
  - Top-1 Routing：每个 Token 只选 1 个 专家。
  - 极致稀疏：计算量最小，但训练不稳定（路由抖动）。
- Mixtral 8x7B (Mistral AI, 2023) / DeepSeek-V3：
  - Top-2 Routing：每个 Token 选 2 个 专家。
  - 优势：
    1. 更平滑：两个专家可以互补，梯度更新更稳定。
    2. 更强表达：允许专家之间进行“组合”（Expert 1 懂语法，Expert 2 懂代码，合起来懂 Python 语法）。
  - 现状：Top-2 是目前的主流选择。
#### 考点五：推理优化 (Inference Optimization)
这是加分项。
##### Q5: MoE 推理时怎么加速？显存不够怎么办？
- 问题：MoE 参数量巨大（比如 47B），显存放不下。
- 解决方案：
  1. 量化 (Quantization)：把参数压成 FP8 或 INT4。
    - Mixtral 8x7B 用 4-bit 量化后，单张 24G 显卡（如 3090/4090）就能跑！
  2. 专家卸载 (Expert Offloading)：
    - 把不常用的专家放在 CPU 内存里。
    - 用到的时候再搬到 GPU（虽然慢，但能跑）。
  3. 共享专家 (Shared Expert)：
    - DeepSeek-MoE 的创新：设立一个“共享专家”，所有 Token 必选它。
    - 其他“独有专家”按需选择。
    - 作用：共享专家负责通用知识（语法、常识），独有专家负责专业知识（代码、数学），进一步提高参数利用率。
#### 总结：面试怎么答？
如果面试官问：“讲讲 MoE。”
满分回答逻辑：
1. 定义：MoE 通过 Router 实现 稀疏激活，用小模型的计算量达到大模型的容量。
2. 核心难点：负载均衡。
  - 必须加 Auxiliary Loss 防止专家坍塌。
  - 必须设置 Expert Capacity 防止显存溢出，要注意 Token Dropping 问题。
3. 主流架构：
  - 现在流行 Top-2 Routing（如 Mixtral），比 Top-1（Switch）更稳。
  - DeepSeek 提出了 Shared Expert，进一步分离了通用和专用知识。
4. 推理优势：
  - 吞吐量极高（Token/s），适合高并发服务。
一句话：
MoE 就是“三个臭皮匠顶个诸葛亮”，关键是要有一个好管家（Router）把活儿分匀了，别让一个人累死。
