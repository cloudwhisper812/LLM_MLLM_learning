## RAG（Retrieval-Augmented Generation）

#### 概念和动机
RAG = “检索 + 生成”。解决模型“幻觉”和知识时效性问题；让模型回答“私有域/长尾知识”的问题。

#### RAG最小架构（Pipeline）
典型流程：chunking → Embedding → Index → Retrieve → Prompt → Generate。
- chunking
  - 常见策略：（1）固定长度（如 512/1024 tokens）+ 滑动窗口重叠（如 10–20%）（2）语义/结构切分（按标题、段落、列表；或按 Markdown/HTML节点）
- embedding
  - 开源（BGE, E5, GTE）；闭源（text-embedding-3-large/small），常见维度 512/768/1024
- 索引与向量数据库
  - KNN: 数据量小（几万）时可以用，大了就没戏了。
  - ANN: HNSW (Hierarchical Navigable Small World)。 它通过构建层次化的图结构实现快速检索。
- retrieval
  - Top-k 与多路检索：基础：向量检索 top-k；增强：BM25（关键词）与向量混合（hybrid）
  - 重排：用更强的跨编码器（cross-encoder，如 Cohere/MonoT5/bge-reranker）对初选文档做语义打分重排。
  - 粗检（ANN）→ 精排（重排序）→ 去重与覆盖控制
  - 强检索 = 混合检索 + 重排序；先粗后精，提升相关性与精度
- Generation
  - “请仅根据我提供的上下文回答问题。” (限定信息来源)
  - “如果上下文中没有相关信息，请直接回答‘我不知道’。” (防止幻觉)
  
