# Scaling Up On-Device LLMs via Active-Weight Swapping Between DRAM and Flash

## Motivation

传统基于CNN和Bert的DRAM-Flash交换方法不适用于LLM

现有的LLM利用稀疏化的方法要么基于ReLU，或者基于ReLU重新训练；要么过时，要么开销大

新兴的无训练稀疏化方法TEAL没法预测活跃权重

观察1：层与层之间的输入权重有相似性

观察2：在上下文场景下，热权重被激活的概率大于0.7

## Key Idea

实现一个用户透明的模型DRAM使用限制

## Design and Solution

*   跨层的主动权重预取

一次预取N层，N取决于加载和计算延迟，按需预取

调整数据布局，\[写的很抽象]，类似于一个channel内存多层数据，一次能读更多数据，\[更好奇怎么做到的]

*   交换pipeline

pipeline阶段：计算、top-k，按需加载，预加载

目标：最大化overlap，最大化缓存命中率

各种参数调整：LLM稀疏性，预测层数N，缓存的大小

缓存装：kvcache内存固定，跨层权重和热权重内存动态调整

使用贪心的方法来确定上述参数

利用LRU来更新缓存

*   添加补偿来对齐稠密模型的分布

通过蒸馏+训练来弥补因top-k稀疏化带来的损失下降

## Evaluation
