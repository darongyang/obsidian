# DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving

## Motivation

【场景】

*   TTFT和TPOT的延迟问题
*   P/D两阶段有非常不同的计算特征和延迟要求

P：并行单步，但长；计算密集型

D：串行多步，但短；访存密集型

【现有方法】

由于KV cache的共享，现有的方法都是基于一个gpu内耦合的方式做，batch P/D。

一种优化方法，分块prefill：

块小无法饱和GPU，prefill时延增加，但decode batch的机会多

块大decode batch更少，但能饱和GPU

此外分块导致访存增加O(N) -> O(\$N^2\$)

【问题】

*   （1）P/D两阶段的干扰

D要等P，增加TPOT；

P/D资源争夺，相互干扰；难以平衡两种延迟，同时满足两者；

无论是串行调度、batch处理还是优先级调度方式

*   （2）耦合资源分配和并行

两种延迟的需求不同；资源耦合分配不符合不同需求

## Key Idea

将P/D分配到不同GPU，在满足TTFT和TPOT

## Design and Solution

（1）一对多。P/D使用不用的instance，一个D instance对应多个P instance来提高D instance的batch

（2）如何定制P/D instance策略

如何对P instance定制并行策略？阈值、算子/模型并行

如何对D instance定制并行策略？考虑mem-bound（限制batch大小，page attention），com-bound阶段（和P instance类似）

（3）考虑变长prefill和通信开销的问题？集群放置策略等

节点间高速

节点间低速：P/D instance要放一个node内，用NVLNK

## Result

实验配置：4节点，每个节点8张A100
