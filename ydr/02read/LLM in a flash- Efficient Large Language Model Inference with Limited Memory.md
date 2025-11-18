# LLM in a flash: Efficient Large Language Model Inference with Limited Memory

## Motivation

DRAM受限的设备

异构存储的带宽/容量不同：（1）如果DRAM装不下，频繁闪存访问开销很大；（2）即使DRAM能装下，加载速度也会影响TTFT

闪存的读性能随块大小和线程数越大而越快：（1）增大读的块能够减少启动延迟；（2）线程数越多能够充分利用SSD的内部并行

## Key Idea

利用稀疏激活的特性，加载必要的权重，实现减少数据传输，增大传输吞吐

## Design and Solution

*   减少数据传输

加载必要的权重：注意力的全部权重，FFN的非稀疏部分

使用预测器预测非稀疏部分

滑动窗口缓存历史激活神经元

*   提升数据传输吞吐

目标：一次可以读更多数据

将某个神经元计算所需要的行和列捆绑一起存储

神经元之间的激活有伙伴关系 -> 失败，因为带来冗余多次load

*   高效内存管理

预先分配内存

行列感知的神经元驱逐和追加

## Evaluation
