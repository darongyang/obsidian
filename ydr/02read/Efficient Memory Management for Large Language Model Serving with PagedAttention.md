# Efficient Memory Management for Large Language Model Serving with PagedAttention

## Motivation

现有的方法：

（1）静态的固定分配一个显存上限

这进一步导致内部/外部碎片化问题，和预留空间问题

（2）不能很好的支持共享

## Key Idea

打破kv cache的连续存储，离散申请逻辑页，实现共享

## Design and Solusion

（1）公式变换

计算公式的token对齐 -> block对齐

当前token和前面所有的token的分数计算是并行的

从q\_token \* k\_token 的注意力计算，变成以q\_token \* (k\_token)s 的注意力计算

（2）块映射表

逻辑地址 | 分配地址 | 装填token数

（3）\*场景讨论

单个sequence

多个sequence

不同解码算法

系统提示词

请求调度

驱逐方式

恢复方式：swap / 重计算

并行

## Result

测试指标：请求的吞吐量、单个token延迟、内存利用率
