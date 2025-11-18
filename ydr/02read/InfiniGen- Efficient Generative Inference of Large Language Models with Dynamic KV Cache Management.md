# InfiniGen: Efficient Generative Inference of Large Language Models with Dynamic KV Cache Management

## Motivation

H2O按照top-k给kvcache打分，驱逐不必要kvcache

H2O固定top-k存在的问题

*   C1：注意力模式的动态性——<span style="color: rgb(64, 64, 64)"><span style="background-color: rgb(255, 255, 255)">永久丢弃token会导致模型“遗忘”潜在重要的历史信息</span></span>

*   C2：不同层对KV缓存的需求不同

*   C3：不同查询对KV缓存的需求不同

观察：跨层的输入具有相似性

## Key Idea

kvcache offload，仅将必要的部分预取到 GPU

利用前一层 Transformer层的注意力输入来推测和预取当前层重要标记的键和值张量

## Design and Solution

*   利用层间相似性预测——提前知道下一层kvcache的重要性

*   SVD分解——<span style="color: rgb(64, 64, 64)"><span style="background-color: rgb(255, 255, 255)">直接使用原始Q和K的数值分布较分散，难以快速定位关键token。通过倾斜变换，强化少数关键通道的权重</span></span>

*   分层按需窗口预算

*   kvcache缓存池管理

（SVD分解如何做到不影响模型精度的？）

## Evaluation

todo
