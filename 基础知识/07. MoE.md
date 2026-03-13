### MoE

### 1. 核心架构 (Router + Experts)
- Experts 通常是多个 FFN (Feed-Forward Network) 层。比如 8 个 FFN，每个结构一模一样，但参数不同。Router 是一个小的线性层 + Softmax。输入 Token x，输出每个专家的权重 G(x)。
- 流程：Token x 进来。Router 计算权重，只选权重最大的 K 个专家（比如 Top-2）。选择到的专家输出加权求和。虽然有 8 个专家，但每次只算 2 个。

### 2. 负载均衡 (Load Balancing)
- 坍塌 (Collapse) / 赢家通吃 (Winner-take-all)。Router 初始化时，某个专家（比如 Expert 1）的权重稍微大了一点点。Router 发现 Expert 1 训练得好，就拼命把 Token 往它那里发。
- 解决方案：辅助损失 (Auxiliary Loss / Load Balancing Loss): 强迫 Router 把 Token 均匀分配 给所有专家。 其实就是统计一些case，比如一个batch。尽量的让多个expert都平均的被使用了


### 专家容量 (Expert Capacity)
- 背景：在分布式训练中，不同的专家可能在不同的 GPU 上。
- Expert Capacity：设定每个专家在一个 Batch 里能处理的最大 Token 数量。
- - Token Dropping (丢弃)：如果 Router 把太多 Token 发给同一个专家（超过 Capacity）。它们不经过 FFN，直接通过残差连接（Residual Connection）流到下一层。会信息丢失，性能下降。

### 架构演进
- Switch Transformer (Google, 2021): top 1: 计算量最小，但训练不稳定（路由抖动）。
- Mixtral 8x7B (Mistral AI, 2023) / DeepSeek-V3：Top-2 Routing, 更平滑：两个专家可以互补，梯度更新更稳定。更强表达：允许专家之间进行“组合”（Expert 1 懂语法，Expert 2 懂代码，合起来懂 Python 语法）。
- Shared Expert: DeepSeek-MoE 的创新：设立一个“共享专家”，所有 Token 必选它。 共享专家负责通用知识（语法、常识），独有专家负责专业知识（代码、数学），进一步提高参数利用率。

