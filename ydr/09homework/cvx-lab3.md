## 1.实验内容
K-臂赌博机是在线机器学习中一类重要的受限信息序列决策问题。K-臂赌博机需要最小化 𝑇 回合之后的累计损失。本实验需要完成求解K-臂赌博机的Tsallis-Inf算法的简化版本Tsallis-Inf++算法。其中最关键一步在于如何求解每一轮的$p_{t+1}$。 
**（1）问题建模**
上述问题经过数学公式建模，可以变为求解如下最优化问题：
![[Pasted image 20241121184711.png]]
其中，上述式子中的Bregman散度定义为：
![[Pasted image 20241121185408.png]]
**（2）Tsallis-Inf++算法**
该算法的流程如下伪代码所示：
![[Pasted image 20241121185550.png]]
## 2.实验方法

**（1）拉格朗日乘子法**
上述问题包含等式和不等式约束，引入$\lambda$ 和$v$ 构建该问题的拉格朗日函数如下：
$$L(p,\lambda,v) = \min_{p \in \Delta_K} \langle p, \tilde{c}_t \rangle + \mathcal{B}_{\psi}(p, p_t)+\lambda_{i}p_{i}+v(\sum_{i=1}^N p_i - 1),\quad \forall i \in [K]$$
由KKT条件的互补松弛条件有如下式子成立。进一步假设最优解中$p_{i}$不为0，否则原问题的K可以退化为K-1。因此，$\lambda_{i}=0$。

$$\lambda_{i}p_{i}=0$$

由KKT的稳定性条件，对上述朗格朗日函数变量$p$求导，对$p$的每个分量$p_{i}$，均有：

$$(p_{t,i})^{{1}/{\alpha}-1}+\eta(\tilde{c}_{t,i}-v)=(p_{t+1,i}^{*})^{{1}/{\alpha}-1}$$
最终求得：
$$p_{t+1,i}^{*}=((p_{t,i})^{{1}/{\alpha}-1}+\eta(\tilde{c}_{t,i}-v))^{^{\alpha/1-\alpha}}$$

对应到实现代码中为：
```python
# KKT条件
def get_p_star(v, p_t, c_t_estimate, alpha, eta):
    p_star = np.zeros(len(p_t))
    p_star_sum = 0
    for i in range(len(p_t)):
        p_star[i] = (p_t[i] ** (1/alpha - 1) + eta * (c_t_estimate[i] - v)) ** (alpha / (1-alpha))
        p_star_sum += p_star[i]
    return p_star, p_star_sum
```

**（2）二分搜索求解**
在上一步中，利用拉格朗日乘子法，利用$v$得到了$p_{t+1}$的最优解表达式，将其回代到如下式子中。易知该式子是关于$v$的单调函数，对齐进行二分搜索即可找到$v^*$，进而得到$p_{t+1}$。
$$\sum_{i=1}^N p_i = 1$$
对应代码如下：
```python
    for iter in range(max_iter):
        v = (v_max + v_low) / 2
        # 用v得出p_star
        p_star, p_star_sum = get_p_star(v, p_t, c_t_estimate, alpha, eta)
        if abs(p_star_sum - 1) < tol:
            return p_star
        # 将p_star回代要满足概率和为1
        if p_star_sum > 1 :
            v_max = v
        else:
            v_low = v
```
**（3）Tsallis-Inf++算法**
按照实验内容部分，实现的算法流程如下：
```python
def tsallis_inf_simplify(K, eta, alpha, T, C):
    p_t = np.ones(K)
    for i in range(K):
        p_t[i] = p_t[i] / K
    for t in range(T):
        i_t = select_action(K, p_t)
        c_t = select_loss(C, t, i_t)
        c_t_estimate = get_estiamte_C(c_t, p_t, eta, K, i_t)
        p_t = lagrange_solve(c_t_estimate, p_t, alpha, eta, K)
    return p_t
```
## 3.实验结果和分析

按照实验要求，运行10次，计算平均值，得到的$\sum_{t=1}^{T} c_t, i_t, \left\{ \sum_{t=1}^{T} c_t, i \right\}_{i=1}^{K} \text{和} p_{10001}$的结果平均值，依次如下：
```
ydr@inspur:~/convex/lab3$ python3 lab3.py 
 
c_t_it: 254.80214418640986

c_t_actions: [0.03263533 0.03272318 0.03255904 0.03269706 0.03262143 0.03268828
 0.03258916 0.03265137 0.03265137 0.03258916 0.03268828 0.03262143
 0.03269706 0.03255904 0.03272318 0.03263533 0.00662702 0.01048929
 0.01099677 0.01099733]
 
 pt10001: [0.03485652 0.03728465 0.03771661 0.03415517 0.03596112 0.03683661
 0.03371443 0.03481704 0.03304255 0.03621948 0.03307756 0.03537848
 0.03387519 0.03669859 0.03649855 0.03642755 0.13191944 0.1033681
 0.09740419 0.10074817]
```
可以看到，$\left\{ \sum_{t=1}^{T} c_t, i \right\}_{i=1}^{K}$中越小损失对应的动作（即后四个），在$p_{10001}$中的概率越大。其中上述倒数第四个动作为$i^*$，对应在在$p_{10001}$中的概率最大。
## 4.总结

在本实验中，主要研究了K-臂赌博机问题，并实现了Tsallis-Inf算法的简化版本。通过数学建模，将K-臂赌博机问题转化为一个带有等式和不等式约束的最优化问题。对这个问题，进一步采用了拉格朗日乘子法和二分搜索方法，最后通过Tsallis-Inf++算法求解该问题。
实验结果表明，损失较小的动作在$p_{10001}$中的概率较大，与预期相符。特别是，倒数第四个动作为$i^*$，在$i^*$中的概率最大。Tsallis-Inf++算法能够有效地根据历史损失信息调整动作的概率分布，从而实现最小化累计损失的目标。