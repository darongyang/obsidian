### 5.27 等式约束最小二乘问题：

![[770f7ff82cb3ef1a22278195c665e148.jpeg]]

考虑如下等式约束最小二乘问题：
$$
\min \|Ax - b\|_2^2
$$
使得 $Gx = h$，

其中 $A \in \mathbb{R}^{m \times n}$，秩为 $n$，$G \in \mathbb{R}^{p \times n}$，秩为 $p$。

给出 KKT 条件，并推导出原问题解 $x^*$ 和对偶解 $\nu^*$ 的表达式。

**解答：**

(a) 拉格朗日函数为
$$
L(x, \nu) = \|Ax - b\|_2^2 + \nu^T (Gx - h)
$$
$$
= x^T A^T A x + (G^T \nu - 2A^T b)^T x - \nu^T h，
$$
其最小值点为 $x = -(1/2)(A^T A)^{-1}(G^T \nu - 2A^T b)$。对偶函数为
$$
g(\nu) = -(1/4)(G^T \nu - 2A^T b)^T (A^T A)^{-1}(G^T \nu - 2A^T b) - \nu^T h。
$$

(b) 最优条件为
$$
2A^T (Ax^* - b) + G^T \nu^* = 0， \quad Gx^* = h。
$$

(c) 从第一个方程可以得出
$$
x^* = (A^T A)^{-1}(A^T b - (1/2)G^T \nu^*)。
$$
将此表达式代入第二个方程，得到
$$
G(A^T A)^{-1} A^T b - (1/2)G(A^T A)^{-1} G^T \nu^* = h，
$$
即
$$
\nu^* = -2(G(A^T A)^{-1}G^T)^{-1}(h - G(A^T A)^{-1}A^T b)。
$$
将此表达式代入第一个表达式，可以得到 $x^*$ 的解析解。

---

### 4.5 等价的凸问题

![[e620b3fb0f1776d29a767a3f7e3ce869.jpeg]]
![[bfee8acd3686ce09e02a981d3c8b0c6e.jpeg]]

展示以下三个凸问题是等价的。仔细说明每个问题的解是如何从其他问题的解中得到的。问题的数据包括矩阵 $A \in \mathbb{R}^{m \times n}$（行向量为 $a_i^T$）、向量 $b \in \mathbb{R}^m$，以及常数 $M > 0$。

1. **鲁棒最小二乘问题**
   $$
   \min \sum_{i=1}^m \phi(a_i^T x - b_i)
   $$
   其中变量 $x \in \mathbb{R}^n$，$\phi: \mathbb{R} \rightarrow \mathbb{R}$ 定义为
   $$
   \phi(u) = \begin{cases} 
   u^2 & |u| \le M \\
   M(2|u| - M) & |u| > M 
   \end{cases}
   $$
   （该函数称为Huber惩罚函数）

2. **带变量权重的最小二乘问题**
   $$
   \min \sum_{i=1}^m \frac{(a_i^T x - b_i)^2}{w_i + 1} + M^2 \mathbf{1}^T w
   $$
   满足 $w \succeq 0$，其中变量 $x \in \mathbb{R}^n$ 和 $w \in \mathbb{R}^m$，定义域 $D = \{(x, w) \in \mathbb{R}^n \times \mathbb{R}^m | w \succeq -1\}$。
   
   提示：对 $x$ 固定，优化 $w$，建立与问题 (a) 的关系。

3. **二次规划问题**
   $$
   \min \sum_{i=1}^m (u_i^2 + 2M v_i)
   $$
   满足
   $$
   -u - v \preceq Ax - b \preceq u + v, \quad 0 \preceq u \preceq M \mathbf{1}, \quad v \succeq 0.
   $$

---

**解答**

(a) **问题 (a) 和 (b)**。对于固定的 $u$，最小化问题
   $$
   \min_{w \succeq 0} \frac{u^2}{w + 1} + M^2 w
   $$
   的解是
   $$
   w = \begin{cases} 
   |u| / M - 1 & |u| \ge M \\
   0 & |u| < M 
   \end{cases}
   $$

   当 $|u| < M$ 时，最优值为 0；否则，$w = 0$ 是最优解。

因此，对于固定的 $x$，问题 (b) 中变量 $w$ 的最优解 $w_i$ 给出如下：
$$
w_i = \begin{cases} 
\frac{|a_i^T x - b_i|}{M} - 1 & |a_i^T x - b_i| \ge M \\
0 & |a_i^T x - b_i| < M 
\end{cases}
$$

通过将 $w_i$ 的表达式代入，可以得到问题 (b) 与问题 (a) 的等价关系。

---

(b) **问题 (a) 和 (c)**。假设在问题 (c) 中固定 $x$。

首先我们注意到，在最优情况下，必须有 $u_i + v_i = |a_i^T x - b_i|$。如果 $u_i$ 和 $v_i$ 满足 $u_i + v_i > |a_i^T x - b_i|$ 且 $0 \le u_i \le M$ 和 $v_i \ge 0$，那么由于 $u_i$ 和 $v_i$ 不同时为零，可以减少 $u_i$ 和/或 $v_i$ 而不违反约束。这也会使目标函数值下降。

因此，在最优情况下，我们有
$$
v_i = |a_i^T x - b_i| - u_i
$$

通过消去 $v$，可以将问题转化为等价的最小化问题
$$
\min \sum_{i=1}^m (u_i^2 - 2M u_i + 2M|a_i^T x - b_i|)
$$
满足
$$
0 \le u_i \le \min\{M, |a_i^T x - b_i|\}
$$

如果 $|a_i^T x - b_i| \le M$，则 $u_i$ 的最优选择是 $u_i = |a_i^T x - b_i|$。在这种情况下，目标函数中对应的第 $i$ 项变为 $(a_i^T x - b_i)^2$。如果 $|a_i^T x - b_i| > M$，我们选择 $u_i = M$，那么目标函数中对应的第 $i$ 项变为 $2M |a_i^T x - b_i| - M^2$。

因此，对于固定的 $x$，问题 (c) 的最优值给出为
$$
\sum_{i=1}^m \phi(a_i^T x - b_i)
$$
这与问题 (a) 的目标函数相同，从而说明了问题 (a) 和问题 (c) 的等价性。

---

综上所述，三个问题 (a)、(b)、(c) 是等价的，即它们具有相同的最优值，并且可以相互转换。


