# vAttention: Dynamic Memory Management for Serving LLMs without PagedAttention

## Motivation

问题：

（1）vLLM的逻辑地址不连续 -> 需要改attention，需要接口适配；

（2）需要维护映射表 -> 性能开销大

块力度大带来内部碎片 -> 内存利用率低

原因：物理地址顺序预分配，逻辑地址动态分配，导致逻辑地址不连续，和OS相反

## Key Idea

见解：逻辑地址保持连续，物理地址动态分配，将不连续下放到物理地址。（本质是增加一个逻辑地址-物理地址的映射）

## Design and Solution

观察1：kv cache分配模式可预测

观察2：kv cache不需要高分配带宽？

机遇：最近cuda提供了VMM API

挑战1：每次物理地址分配延迟高

-> 设计1：异步内存分配，重叠分配与计算

-> 设计2：延迟回收、预分配

挑战2：大页面导致内部碎片

-> 修改驱动的分配粒度

## Result
