# Training-Free Activation Sparsity in Large Language Models

## Motivation

**背景**

自回归阶段是memory bound，预训练和预填充是compute bound

解决memory bound的方法：（1）权重量化；（2）权重稀疏化；（3）激活稀疏化（focus）

激活稀疏化：（1）ReLU激活天然的稀疏性（旧模型）；（2）SwiGLU不再保持天然稀疏

**动机**

*   现有的LLM采用SwiGLU，使得传统基于ReLU的方法，如Dejavu，失效
*   现在的一些工作想恢复ReLU的稀疏性，但要进行重新预训练
*   CATS实现无训练稀疏，但稀疏效果有限

总结：现有方法或依赖过时的ReLU架构，或需昂贵再训练，或仅稀疏局部模块

## Key Idea

利用参数分布的密集性，对稠密矩阵的值进行阈值置零

## Design and Solution

目标：无需修改模型和额外训练，在SwiGLU上实现稀疏化激活

*   分布观察
*   TEAL阈值截断（截断概率分布占大头且值较小的参数）
*   块间感知优化
*   硬件感知优化

## Evaluation
