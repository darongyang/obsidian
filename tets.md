以下是用 LaTeX 格式呈现的公式：

### 数学本质

将联合策略分解为：

```
\pi(\mathbf{a}|\mathbf{s}) = \prod_{i} \pi_{\text{role}(i)}(a_i | \phi_i(\mathbf{s}))
```

其中角色函数 `\text{role}(i)` 由注意力机制动态分配，状态编码 `\phi_i(\mathbf{s})` 过滤无关博弈信息。

### 关键创新

```
\begin{aligned}
&\textbf{竞争感知}:\quad & &\min_{\pi} \max_{\pi_{\text{opp}}} Q_i \\
&\textbf{协作涌现}:\quad & &\arg\max_{\mathbf{a}} \sum_{j \in \text{team}} U_j \\
&\textbf{熵控平衡}:\quad & &\pi^* = \arg\max_{\pi} \mathbb{E}[Q(\mathbf{s},\mathbf{a})] + \tau \mathcal{H}(\pi)
\end{aligned}
```

### 完整公式环境

```
\documentclass{article}
\usepackage{amsmath,amssymb}

\begin{document}

\section*{数学本质}
将联合策略分解为：
$$
\pi(\mathbf{a}|\mathbf{s}) = \prod_{i} \pi_{\text{role}(i)}(a_i | \phi_i(\mathbf{s}))
$$
其中角色函数 $\text{role}(i)$ 由注意力机制动态分配，状态编码 $\phi_i(\mathbf{s})$ 过滤无关博弈信息。

\section*{关键创新}
\begin{align*}
&\text{竞争感知}:\quad & &\min_{\pi} \max_{\pi_{\text{opp}}} Q_i \\
&\text{协作涌现}:\quad & &\arg\max_{\mathbf{a}} \sum_{j \in \text{team}} U_j \\
&\text{熵控平衡}:\quad & &\pi^* = \arg\max_{\pi} \mathbb{E}[Q(\mathbf{s},\mathbf{a})] + \tau \mathcal{H}(\pi)
\end{align*}

\end{document}
```

### 符号说明

- `\pi(\mathbf{a}|\mathbf{s})`：状态 `\mathbf{s}` 下采取联合动作 `\mathbf{a}` 的策略
- `\text{role}(i)`：智能体 `i` 的角色函数（由注意力机制学习）
- `\phi_i(\mathbf{s})`：智能体 `i` 的状态编码（过滤无关博弈信息）
- `Q_i`：智能体 `i` 的动作价值函数
- `U_j`：团队智能体 `j` 的效用函数
- `\mathcal{H}(\pi)`：策略 `\pi` 的熵（衡量策略多样性）
- `\tau`：温度参数（调节竞争强度）