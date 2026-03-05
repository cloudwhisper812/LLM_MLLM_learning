# 图像与视频生成：从 Stable Diffusion 到 DiT

> **学习目标**：掌握扩散模型从 Pixel Space 到 Latent Space 的跨越，理解 Transformer 如何取代 U-Net 成为新一代生成底座。

---

## Ⅰ. Stable Diffusion (LDM 架构)

### 1. Latent Diffusion Model (LDM)
* **核心痛点**：传统的 Diffusion（如 DDPM）在像素空间计算，分辨率越高，计算量呈平方级增长。
* **解决方案**：引入 **VAE (Variational Autoencoder)**。
    * **Encoder**：将图像从 $512 \times 512 \times 3$ 压缩到 $64 \times 64 \times 4$ 的隐空间（Latent Space）。
    * **Decoder**：将去噪后的 Latent 还原回像素图像。
* **面试考点**：为什么不在 Pixel Space 练？答：节省计算资源，同时 Latent 表达了更高层的语义，去噪更高效。



### 2. U-Net 与 Cross-Attention
* **U-Net 角色**：作为噪声预测器（Noise Predictor），输入带噪 Latent $z_t$ 和步数 $t$，输出预测噪声 $\epsilon$。
* **Attention 机制**：
    * **Self-Attention**：用于建模图像内部的全局空间关系。
    * **Cross-Attention**：将文本 Embedding（由 CLIP 提取）注入 U-Net，实现 Text-to-Image 的控制。
* **公式理解**：$Attention(Q, K, V) = softmax(\frac{QK^T}{\sqrt{d}})V$，其中 $Q$ 来自图像特征，$K, V$ 来自文本特征。



### 3. 采样 (Sampling / Scheduler)
* **本质**：求解随机微分方程（SDE/ODE）的过程。
* **常用采样器**：
    * **DDPM**：随机性强，步数多（1000步）。
    * **DDIM**：确定性采样，支持更少步数（20-50步），且支持 Inversion（反推噪声）。
    * **DPM++ / Euler a**：目前工业界兼顾速度与质量的首选。

---

## Ⅱ. DiT (Diffusion Transformer)

### 1. 从 U-Net 到 Transformer 的演进
* **背景**：Sora 和 SD3 的成功证明了 Scaling Law 在生成领域依然成立。U-Net 虽好，但参数量扩展（Scaling）不如 Transformer 顺滑。
* **DiT 核心结构**：
    * **Patchify**：效仿 ViT，将 $64 \times 64$ 的 Latent 切成一个个 Patch。
    * **DiT Block**：取代卷积层，使用标准 Transformer Block 处理 Patch。
    * **Adaptive Layer Norm (adaLN)**：将时间步 $t$ 和条件标签 $c$ 注入模型的核心手段。



### 2. Scaling Law 与视频生成
* **DiT 的优势**：模型深度和宽度更容易增加，更适合处理 **视频序列 (3D Attention)**。
* **应用案例**：Sora、SD3、Flux。

---

## Ⅲ. 算法工程师视角：TikTok 业务思考
* **内容安全**：如何利用 SD 的 Inpainting（局部重绘）能力进行违规内容遮盖或修复？
* **多模态增强**：SigLIP 2 能否作为更好的 Vision Tower 取代 CLIP，提供更精准的 Prompt 引导？

---
**Next Step**: [ ] 实现一个简单的 DDIM Sampler 代码 | [ ] 比较 DiT 与 U-Net 的 FLOPs 差异