# ChunkAttention: Efficient Self-Attention with Prefix-Aware KV Cache and Two-Phase Partition

## Motivation

现有的LLM应用共享系统提示词很常见

*   kvcache随上下文长度增加而线性增加
*   共享的系统提示词带来kvcache冗余

现有的工作：静态不灵活、命中率低、没有优化自注意力

## Key Idea

将KV cache进行前缀感知，运行时动态删除冗余

## Design and Solution

*   将KV的管理组织为前缀感知的PAKV

*   将自回归解码分为两个阶段

    *   chunk-first阶段，处理多个序列的共享部分，以chunk为单位
    *   sequence-first阶段，利用上个阶段的结果，处理特定序列的chunk

## Result

配置：A100(80G)

性能提升达到3.2x-4.8x

## Limitation

*   对system prompt有严格位置感知的要求
*   微调可能不会再带来很长的system prompt
