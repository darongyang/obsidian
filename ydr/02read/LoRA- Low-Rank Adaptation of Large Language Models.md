# LoRA: Low-Rank Adaptation of Large Language Models

## Motivation

模型参数太大，全参数微调变得困难

有很多工作减少微调参数来解决上述挑战，但：

*   <span style="color: rgb(25, 27, 31)"><span style="background-color: rgb(255, 255, 255)">Adapt Tuning：</span></span>增加了推理延迟，引入了额外路径

*   <span style="color: rgb(25, 27, 31)"><span style="background-color: rgb(255, 255, 255)">Prefix Tuning：</span></span>要么减少了模型可用序列长度

*   效率和模型质量上做了trade-off

受启发：<span style="color: rgb(25, 27, 31)"><span style="background-color: rgb(255, 255, 255)">处理一个细分的小任务时，不需要那么复杂的高维空间，可能只需要在某个子空间范围内就可以解决</span></span>

优势：共享 易替换、轻量、

## Key Idea

冻结原参数矩阵，仅训练低秩矩阵，然后注入

## Design and Solution

*   将矩阵的更新做低秩的分解。原稠密矩阵： $R^{d \times k} $ 。分解为两个低秩矩阵： $R^{d \times r}$

和$R^{r \times k}$。

*   仅训练这两个低秩矩阵。
*   将其运用到transformer架构中，同时调整 q 和 v 矩阵效果最好

## Evaluation

Mark：<span style="color: rgba(0, 0, 0, 0.9)"><span style="background-color: rgb(252, 252, 252)">LoRA 的显存节省主要体现在​</span></span>**<span style="color: rgba(0, 0, 0, 0.9)"><span style="background-color: rgb(252, 252, 252)">​可训练参数减少​</span></span>**<span style="color: rgba(0, 0, 0, 0.9)"><span style="background-color: rgb(252, 252, 252)">​，但冻结参数仍需驻留显存</span></span>

参考解读

*   <https://zhuanlan.zhihu.com/p/650197598>
