# 中文分词：高级篇

## 引入 CRF

前面章节的分词方法容易理解，也有很高的执行效率，但面对歧义、新词和复杂上下文时，仅靠局部匹配往往难以作出稳定判断。更进一步的做法，是把中文分词看成一个**字序列标注问题**：为句子中的每个字预测其在词语中的位置，再根据标签序列恢复词语边界。

条件随机场（Conditional Random Field，CRF）正是解决这类问题的经典模型，也是 Wapiti 使用的核心模型。给定字序列 \\(x = (x_1, x_2, \ldots, x_n)\\)，CRF 不再逐字独立判断，而是为完整的标签序列 \\(y = (y_1, y_2, \ldots, y_n)\\) 计算条件概率，并从中选择整体得分最高的序列。这使模型既能利用当前字及其上下文，也能约束相邻标签之间的组合关系。

**CRF 用于中文分词的关键能力**：

- **条件建模**：直接建模 \\(P(y\mid x)\\)，关注已知句子时标签序列出现的概率
- **全局解码**：联合考虑整句话的标签，不以某个字的局部最优结果代替全局最优结果
- **特征组合**：可以同时使用字、上下文、字符类型和标签转移等特征

举例，“南京市长江大桥”只看局部可能会把“市长”识别成一个词，但结合完整上下文，更合理的切分是“南京市 / 长江大桥”。常用的 BMES 标签体系以 B、M、E、S 分别表示词首、词中、词尾和单字成词。这个切分可以表示为：

```text
南  京  市  长  江  大  桥
B   M   E   B   M   M   E
```

这样，分词问题就变成了为字序列寻找最佳标签序列的问题。


## 目标函数

**概率公式**

CRF 的条件概率定义为：

$$
P( y|x ) = \frac{1}{Z(x)}\times\exp\left(\sum&#95;{i}\sum&#95;{k}\lambda&#95;k f&#95;k(y&#95;{i-1},y&#95;i,x,i)\right)
$$

**关键组成部分**：

- **\\(Z(x)\\)**：归一化因子，确保概率和为1
- **\\(f_k(y_{i-1},y_i,x,i)\\)**：特征函数
- **\\(\lambda_k\\)**：特征权重参数

**特征函数类型**

**一元特征（Unigram Features）**
$$
f&#95;1(y&#95;i,x,i) = \begin{cases}
1 & \text{if } (y&#95;i,x,i) \text{满足指定的一元特征模板} \\\\
0 & \text{otherwise}
\end{cases}
$$

**二元特征（Bigram Features）**
$$
f&#95;2(y&#95;{i-1},y&#95;i,x,i) = \begin{cases}
1 & \text{if } (y&#95;{i-1},y&#95;i,x,i) \text{满足指定的二元特征模板} \\\\
0 & \text{otherwise}
\end{cases}
$$

**势函数**
为简化表示，引入势函数：
$$
\psi&#95;t(y',y,x) = \exp\left(\sum&#95;k \lambda&#95;k f&#95;k(y',y,x,t)\right)
$$

则条件概率可重写为：
$$
P( y| x) = \frac{1}{Z(x)} \times \prod&#95;t \psi&#95;t(y&#95;{t-1},y&#95;t,x)
$$

**归一化因子**

归一化因子需要对全部可能的标签序列求和：

$$
Z(x) = \sum&#95;y \prod&#95;t \psi&#95;t(y&#95;{t-1},y&#95;t,x)
$$

**问题**：如果有 \\(T\\) 个位置，每个位置有 \\(L\\) 个可能标签，则需要计算 \\(L^T\\) 个序列！

**方案**：引入动态规划算法高效计算。

## 标注过程

**维特比**

**目标**：找到最优标签序列，使得 \\(P(y|x)\\) 最大。

等价于最大化：
$$
\text{score}(y, x) = \sum&#95;i \sum&#95;k \lambda&#95;k f&#95;k(y&#95;{i-1}, y&#95;i, x, i) = \sum&#95;i \log \psi&#95;i(y&#95;{i-1}, y&#95;i, x)
$$

**动态规划**

**状态定义**
$$
\delta&#95;t(y) = \max&#95;{y&#95;1,...,y&#95;{t-1}} \text{score}(y&#95;1,...,y&#95;{t-1},y, x&#95;1,...,x&#95;t)
$$

\\(\delta_t(y)\\) 表示到位置\\(t\\)标签为\\(y\\)的最优路径得分。

**递推公式**
$$
\delta&#95;1(y) = \log \psi&#95;1(\text{START}, y, x)
$$
$$
\delta&#95;t(y) = \max&#95;{y'} \left[\delta&#95;{t-1}(y') + \log \psi&#95;t(y', y, x)\right]
$$

**回溯指针**
$$
\phi&#95;t(y) = \arg\max&#95;{y'} \left[\delta&#95;{t-1}(y') + \log \psi&#95;t(y', y, x)\right]
$$

## 梯度推导

**训练目标**

目标是最大化对数似然函数：

$$
L(\lambda) = \sum&#95;s \log P(y^{(s)}|x^{(s)}) - R(\lambda)
$$

其中：
- \\(s\\) 是训练样本索引
- \\(y^{(s)}\\) 和 \\(x^{(s)}\\) 分别是第 \\(s\\) 个样本的真实标签序列和观察序列
- \\(R(\lambda)\\) 是正则化项

**概率公式**

$$
P(y|x) = \frac{1}{Z(x)} \exp \left(\sum&#95;i \sum&#95;k \lambda&#95;k f&#95;k(y&#95;{i-1},y&#95;i,x,i)\right)
$$

其中：
- \\(Z(x)\\) 是归一化因子
- \\(f_k(y_{i-1},y_i,x,i)\\) 是第\\(k\\) 个特征函数
- \\(\lambda_k\\) 是对应的权重参数

**推导过程**

**第一步：目标函数展开**

把概率公式带入目标函数：
$$
\log P(y|x) = \sum&#95;i \sum&#95;k \lambda&#95;k f&#95;k(y&#95;{i-1},y&#95;i,x,i) - \log Z(x)
$$

目标函数变为：
$$
L(\lambda) = \sum&#95;s \left[\sum&#95;i \sum&#95;k \lambda&#95;k f&#95;k(y&#95;{i-1}^{(s)},y&#95;i^{(s)},x^{(s)},i)
-\log Z(x^{(s)})\right] - R(\lambda)
$$

**第二步：对参数求偏导**

对参数 \\(\lambda_k\\) 求偏导：

$$
\frac{\partial L}{\partial \lambda&#95;k} = \sum&#95;s 
\left[\sum&#95;i f&#95;k(y&#95;{i-1}^{(s)},y&#95;i^{(s)},x^{(s)},i) -
\frac{\partial \log Z(x^{(s)})}{\partial \lambda&#95;k}\right] -
\frac{\partial R(\lambda)}{\partial \lambda&#95;k}
$$

**分析各项**:
- 第一项： \\(\sum_i f_k(y_{i-1}^{(s)},y_i^{(s)},x^{(s)},i)\\) 是真实标签序列下的特征值总和（**经验期望**）
- 第二项： \\(\frac{\partial \log Z(x^{(s)})}{\partial \lambda_k}\\) 需要详细推导（**模型期望**）
- 第三项： 正则化项的导数，L1 会特殊些，要专门来处理

关键在于计算第二项！

**第三步：计算 \\(\frac{\partial \log Z(x)}{\partial \lambda_k}\\)**

使用链式法则：

$$
\frac{\partial \log Z(x)}{\partial \lambda&#95;k} = \frac{1}{Z(x)} \cdot 
\frac{\partial Z(x)}{\partial \lambda&#95;k}
$$

**第四步：计算 \\(\frac{\partial Z(x)}{\partial \lambda_k}\\)**

归一化因子的定义：

$$
Z(x) = \sum&#95;y \exp\left(\sum&#95;i \sum&#95;k \lambda&#95;k f&#95;k(y&#95;{i-1},y&#95;i,x,i)\right)
$$

用势函数表示：

$$
Z(x) = \sum&#95;y \prod&#95;t \psi&#95;t(y&#95;{t-1},y&#95;t,x)
$$

其中势函数：

$$
\psi&#95;t(y&#95;{t-1},y&#95;t,x) = \exp\left(\sum&#95;k \lambda&#95;k f&#95;k(y&#95;{t-1},y&#95;t,x,t)\right)
$$

对 \\(\lambda_k\\) 求偏导：

$$
\frac{\partial Z(x)}{\partial \lambda&#95;k} = \sum&#95;y \frac{\partial}{\partial \lambda&#95;k}
\prod&#95;t \psi&#95;t(y&#95;{t-1},y&#95;t,x)
$$

**第五步：势函数的偏导数**

由于 \\(\psi_t\\) 是指数函数：

$$
\frac{\partial \psi&#95;t}{\partial \lambda&#95;k} = \psi&#95;t \cdot f&#95;k(y&#95;{t-1},y&#95;t,x,t)
$$

使用乘积法则，对于乘积 \\(\prod_t \psi_t\\)：

$$
\frac{\partial}{\partial \lambda&#95;k} \prod&#95;t \psi&#95;t =
\sum&#95;i \left[\prod&#95;{t \ne i} \psi&#95;t \right]
\cdot \frac{\partial \psi&#95;i}{\partial \lambda&#95;k}
$$
$$
= \sum&#95;i \left[\prod&#95;{t \ne i} \psi&#95;t \right]
\cdot \psi&#95;i \cdot f&#95;k(y&#95;{i-1},y&#95;i,x,i)
$$
$$
= \prod&#95;t \psi&#95;t \cdot \sum&#95;i f&#95;k(y&#95;{i-1},y&#95;i,x,i)
$$

**关键洞察**：最后一步从各项中提取了公共因子 \\(\prod_t \psi_t\\)，将表达式化为“连乘 × 连加”

**第六步：代入求和**

把结果代入 \\(Z(x)\\) 的偏导数：

$$
\frac {\partial Z(x)}{\partial \lambda&#95;k} =
\sum&#95;y \prod&#95;t \psi&#95;t(y&#95;{t-1},y&#95;t,x) \cdot
\sum&#95;i f&#95;k(y&#95;{i-1},y&#95;i,x,i)
$$
$$
= \sum&#95;y \left[\sum&#95;i f&#95;k(y&#95;{i-1},y&#95;i,x,i)\right] \cdot
\prod&#95;t \psi&#95;t(y&#95;{t-1},y&#95;t,x)
$$

**第七步：转化为概率形式**

**关键转化**：注意到

$$
\prod&#95;t \psi&#95;t(y&#95;{t-1},y&#95;t,x) = P(y|x) \cdot Z(x)
$$

代入得：

$$
\frac{\partial Z(x)}{\partial \lambda&#95;k} =
\sum&#95;y \left[\sum&#95;i f&#95;k(y&#95;{i-1},y&#95;i,x,i)\right] \cdot
P(y|x) \cdot Z(x)
$$
$$
= Z(x) \cdot \sum&#95;y P(y|x) \cdot \left[\sum&#95;i f&#95;k(y&#95;{i-1},y&#95;i,x,i)\right]
$$

**第八步：最终结果**

把结果代入链式法则：

$$
\frac{\partial \log Z(x)}{\partial \lambda&#95;k} = 
\frac{1}{Z(x)} \cdot Z(x) \cdot 
\sum&#95;y P(y|x) \cdot \left[\sum&#95;i f&#95;k(y&#95;{i-1},y&#95;i,x,i)\right]
$$

这正是**模型期望**:

$$
E&#95;{P(y|x)}\left[\sum&#95;i f&#95;k(y&#95;{i-1},y&#95;i,x,i)\right]
$$

**第九步：梯度的最终形式**

**完整梯度公式**：

$$
\frac{\partial L}{\partial \lambda&#95;k} =
\sum&#95;s \left[\sum&#95;i f&#95;k(y&#95;{i-1}^{(s)}, y&#95;i^{(s)},x^{(s)},i)\right]
{}- \sum&#95;s E&#95;{P(y|x^{(s)})}\left[\sum&#95;i f&#95;k(y&#95;{i-1},y&#95;i,x^{(s)},i)\right]
{}- \frac{\partial R(\lambda)}{\partial \lambda&#95;k}
$$

**或者表示成**：

$$
\frac{\partial L}{\partial \lambda&#95;k}
= \text{经验期望} - \text{模型期望}
  {}- \frac{\partial R(\lambda)}{\partial \lambda&#95;k}
$$

其中：
- **经验期望**：真实数据中特征 \\(f_k\\) 出现的次数
- **模型期望**: 当前模型认为特征 \\(f_k\\) 应该出现的次数

## 前向后向

**核心问题**
对于已经推导出来的梯度公式：

$$
\frac{\partial L}{\partial \lambda&#95;k}
= \text{经验期望} - \text{模型期望}
  {}- \frac{\partial R(\lambda)}{\partial \lambda&#95;k}
$$

其中模型期望是：

$$
E&#95;{P(y|x)}[f&#95;k] = \sum&#95;y P(y|x) \cdot \left[\sum&#95;i f&#95;k(y&#95;{i-1},y&#95;i,x,i)\right]
$$

要是直接计算需要遍历 \\(L^T\\) 个可能的标签序列，计算量太大了！

这里与维特比算法的区别只有一个关键运算：维特比在每个状态保留最大值，用于寻找最佳切分；前向后向算法对所有路径求和，用于计算边际概率和模型期望。

**突破口**

对于一元特征：
$$
E[f&#95;k^{(1)}] = \sum&#95;y P(y|x) \sum&#95;i f&#95;k^{(1)}(y&#95;i,x,i)
$$

交换求和顺序
$$
= \sum&#95;i \sum&#95;y P(y|x) \cdot f&#95;k^{(1)}(y&#95;i,x,i)
$$

**进一步分解**：只有当 \\(y_i\\) 取特定值时 \\(f_k^{(1)}\\) 才为1
$$
= \sum&#95;i \sum&#95;{y&#95;i} f&#95;k^{(1)}(y&#95;i,x,i) \sum&#95;{y&#95;1,...,y&#95;{i-1},y&#95;{i+1},...,y&#95;T}P(y&#95;1,...,y&#95;T|x)
$$

**得到边际概率**：
$$
= \sum&#95;i \sum&#95;{y&#95;i} f&#95;k^{(1)}(y&#95;i,x,i) \cdot P(y&#95;i|x)
$$

类似地，对于二元特征：
$$
E[f&#95;k^{(2)}] = \sum&#95;i \sum&#95;{y&#95;{i-1}} \sum&#95;{y&#95;i} f&#95;k^{(2)}(y&#95;{i-1},y&#95;i,x,i) \cdot P(y&#95;{i-1},y&#95;i|x)
$$

这样，就把求 \\(P(y|x)\\) 转变成了怎么去计算边际概率 \\(P(y_i|x)\\) 和 \\(P(y_{i-1},y_i|x)\\) 了，问题化简了不少。

**思路**

**关键洞察**：边际概率可以分解为到达当前位置的前向分数、当前转移的势函数和离开当前位置的后向分数。这里的前向量与后向量都是未归一化分数，不是条件概率。

$$
\psi&#95;t(y',y,x)
= \exp\left(\sum&#95;k \lambda&#95;k f&#95;k(y',y,x,t)\right)
$$

**前向过程**

**分数定义**

前向分数 \\(\alpha_i(y)\\) 是所有以标签 \\(y\\) 到达位置 \\(i\\) 的路径势函数之和：

$$
\alpha&#95;i(y)
= \sum&#95;{y&#95;1,\ldots,y&#95;{i-1}}
  \prod&#95;{t=1}^{i}\psi&#95;t(y&#95;{t-1},y&#95;t,x)
$$

**初始化**：

$$
\alpha&#95;1(y)=\psi&#95;1(\mathrm{START},y,x)
$$

**递推公式**：

$$
\alpha&#95;i(y)
= \sum&#95;{y'}\alpha&#95;{i-1}(y')\psi&#95;i(y',y,x)
$$

最后一个位置的前向分数之和就是配分函数：

$$
Z(x)=\sum&#95;y\alpha&#95;T(y)
$$

**后向过程**

后向分数 \\(\beta_i(y)\\) 是从位置 \\(i\\) 的标签 \\(y\\) 出发，到达序列末尾的所有后缀路径势函数之和：

$$
\beta&#95;i(y)
= \sum&#95;{y&#95;{i+1},\ldots,y&#95;T}
  \prod&#95;{t=i+1}^{T}\psi&#95;t(y&#95;{t-1},y&#95;t,x)
$$

**初始化**：

$$
\beta&#95;T(y)=1
$$

**递推公式**：

$$
\beta&#95;i(y)
= \sum&#95;{y'}\psi&#95;{i+1}(y,y',x)\beta&#95;{i+1}(y')
$$

**边际概率**

**一元边际概率**：

$$
P(y&#95;i=y\mid x)
= \frac{\alpha&#95;i(y)\beta&#95;i(y)}{Z(x)}
$$

**二元边际概率**：

$$
P(y&#95;{i-1}=y',y&#95;i=y\mid x)
= \frac{
    \alpha&#95;{i-1}(y')
    \psi&#95;i(y',y,x)
    \beta&#95;i(y)
  }{Z(x)}
$$

这两个公式使用同一个全局配分函数 \\(Z(x)\\)，不需要分别定义 \\(z_i\\)、\\(Z_{\text{unigram}}\\) 或 \\(Z_{\text{bigram}}\\)。

**数值稳定性**

直接连乘势函数容易上溢或下溢。实现时可以在对数域中保存前向与后向分数，并用 \\(\operatorname{logsumexp}\\) 完成求和。例如：

$$
\log\alpha&#95;i(y)
= \operatorname{logsumexp}&#95;{y'}
  \left(
    \log\alpha&#95;{i-1}(y')
    {}+ \log\psi&#95;i(y',y,x)
  \right)
$$

也可以在每个位置对向量进行缩放，但前向、后向和边际概率必须使用同一组缩放因子并保持一致的定义。

## L1 正则化

CRF 会产生大量特征，其中许多权重接近零，却仍然占用模型空间并参与计算。L1 正则化通过惩罚权重绝对值产生稀疏参数。Wapic 实际最小化的目标同时支持 L1 和 L2：

$$
\begin{aligned}
F(\lambda)
&=-L(\lambda)+r&#95;1\sum&#95;k|\lambda&#95;k| \\\\
&\quad+\frac{r&#95;2}{2}\sum&#95;k\lambda&#95;k^2
\end{aligned}
$$

其中 \\(r_1\\) 对应命令行参数 `--rho1`，控制稀疏程度；\\(r_2\\) 对应 `--rho2`，为光滑部分增加 L2 惩罚。`GradientComputer::RunGradientComputation` 返回这个目标值，并把 \\(r_2\lambda_k\\) 加入普通梯度；L1 项则由优化器单独处理。

**零点不可导**

L1 正则化项在非零点可导，但在零点不可导。绝对值函数的次微分为：

$$
\partial |\lambda&#95;k| = \begin{cases}
\{1\} & \text{if } \lambda&#95;k > 0 \\\\
\{-1\} & \text{if } \lambda&#95;k < 0 \\\\
[-1,1] & \text{if } \lambda&#95;k = 0
\end{cases}
$$

标准 L-BFGS 假设目标函数光滑，不能直接处理零点处的区间次梯度。后面的 OWL-QN 会用伪梯度和象限约束解决这个问题，并让一部分参数精确变为零。

## L-BFGS

L-BFGS（Limited-memory Broyden-Fletcher-Goldfarb-Shanno）是一种拟牛顿方法。它适合优化不含 L1 项的光滑 CRF 目标，并用有限的历史向量近似二阶曲率。

Wapic 用同一个 `LBFGSOptimizer` 承担两种模式：当 \\(r_1=0\\) 时执行标准 L-BFGS；当 \\(r_1\neq0\\) 时启用伪梯度和象限投影，实际执行后文的 OWL-QN。命令行虽然统一写作 `-a l-bfgs`，默认 \\(r_1=0.5\\)，因此默认训练会进入 OWL-QN 分支。

**拟牛顿法**

**问题基础**

CRF 的参数估计本质上是一个无约束优化问题：

$$
\min&#95;{\lambda} F(\lambda) = -\ell(\lambda) + R(\lambda)
$$

其中：
- \\(\ell(\lambda)\\) 是对数似然函数
- \\(R(\lambda)\\) 是正则化项
- \\(\lambda\\) 是 CRF 的参数向量

梯度下降只使用一阶信息：

$$
\lambda^{(k+1)} = \lambda^{(k)} -\alpha&#95;k \nabla F(\lambda^{(k)})
$$

牛顿法则使用 Hessian 修正更新方向：

$$
\lambda^{(k+1)} = \lambda^{(k)} - 
[\nabla^2 F(\lambda^{(k)})]^{-1} \nabla F(\lambda^{(k)})
$$

对于高维 CRF，显式构造和求解 Hessian 的成本过高。拟牛顿法因此不直接计算 Hessian，而是逐步近似它的逆矩阵。

**拟牛顿法**

用 \\(H_k\\) 表示 Hessian 逆矩阵的近似，搜索方向为：

$$
d&#95;k = -H&#95;k \nabla F(\lambda^{(k)})
$$

当 \\(H_k\\) 正定时，\\(\nabla F(\lambda^{(k)})^T d_k<0\\)，因此它是下降方向。线搜索随后决定沿该方向前进多远。

**BFGS**

BFGS 使用相邻两次迭代的参数变化和梯度变化来更新曲率近似。定义：

$$
s&#95;k=\lambda^{(k+1)}-\lambda^{(k)}
$$

$$
y&#95;k=\nabla F(\lambda^{(k+1)})-\nabla F(\lambda^{(k)})
$$

如果 \\(H_k\\) 表示 Hessian 逆矩阵的近似，那么它应满足割线条件：

$$
H&#95;{k+1}y&#95;k=s&#95;k
$$

标准 BFGS 的逆矩阵更新为：

$$
\begin{aligned}
H&#95;{k+1}
&=(I-\rho&#95;k s&#95;k y&#95;k^T)
  H&#95;k
  (I-\rho&#95;k y&#95;k s&#95;k^T) \\\\
&\quad+\rho&#95;k s&#95;k s&#95;k^T
\end{aligned}
$$

其中：

$$
\rho&#95;k=\frac{1}{y&#95;k^T s&#95;k}
$$

**曲率条件**

如果 \\(H_k\\) 正定，并且满足：

$$
y&#95;k^T s&#95;k>0
$$

那么 BFGS 更新得到的 \\(H_{k+1}\\) 仍然正定，搜索方向：

$$
d&#95;k=-H&#95;k\nabla F(\lambda^{(k)})
$$

就是下降方向。Wapic 的光滑分支使用 Wolfe 条件进行线搜索，并在 `UpdateHistory` 中直接记录 \\(s_k\\)、\\(y_k\\) 和 \\(\rho_k=1/(y_k^Ts_k)\\)。

**L-BFGS**

标准 BFGS 需要存储 \\(n \times n\\) 的矩阵，对于大规模问题（\\(n\\) 很大）并不现实。L-BFGS 不显式存储 Hessian 逆矩阵，而是保存最近的 \\(m\\) 组向量对 \\(\{s_k,y_k\}\\)，用它们隐式表示曲率信息。


**核心思想：**
- **内存需求：** 从\\(O(n^2)\\)降低到\\(O(mn)\\)
- **计算复杂度：** 每次迭代从\\(O(n^2)\\)降低到\\(O(mn)\\)
- **Wapic 默认值：** \\(m=5\\)，可通过 `--histsz` 调整

**两步循环算法**

**算法目标**

两步循环算法的目标是计算搜索方向：
$$
d&#95;k = -H&#95;k g&#95;k
$$

其中 \\(H_k\\) 是 Hessian 逆矩阵的 L-BFGS 近似，\\(g_k\\) 是当前梯度。

关键是：我们要**直接计算出** \\(H_k g_k\\) 的结果，而**不显式构造** \\(H_k\\) 矩阵。

**完整算法**

**输入**
- 当前梯度：\\(g_k\\)
- 历史信息：\\(\{s_i, y_i\}_{i=k-m}^{k-1}\\)（最近\\(m\\)个向量对）
- 初始 Hessian 逆矩阵近似：\\(H_k^0\\)（通常是标量乘以单位矩阵）

**输出**
- 搜索方向：\\(d_k = -H_k g_k\\)

**算法步骤**

**第一步：反向循环（Backward Loop）**

初始化：
\\(q = g_k\\)

反向遍历历史信息：
\\(\text{for } i = k-1, k-2, \ldots, k-m:\\)
\\(\alpha_i = \frac{s_i^T q}{y_i^T s_i}\\)
\\(q = q - \alpha_i y_i\\)
\\(\text{存储 } \alpha_i \text{ 供第二步使用}\\)

**第二步：正向循环（Forward Loop）**

应用初始 Hessian 逆矩阵近似：
\\(r = H_k^0 q\\)

正向遍历历史信息：
\\(\text{for } i = k-m, k-m+1, \ldots, k-1:\\)
\\(\beta = \frac{y_i^T r}{y_i^T s_i}\\)
\\(r = r + s_i (\alpha_i - \beta)\\)

返回结果：
\\(\text{return } -r \quad \text{// 注意负号，因为我们要的是 } -H_k g_k\\)

**数学原理**

每一对 \\((s_i,y_i)\\) 都对应一次 BFGS 逆矩阵更新。连续使用最近 \\(m\\) 个向量对时，矩阵中出现的是这些更新变换的有序乘积，不能把它们简单写成若干秩一矩阵的和。两步循环按照与这些更新相同的顺序执行向量运算，因此能直接得到 \\(H_kg_k\\)，而不需要显式构造 \\(H_k\\)。

**计算复杂度**

**时间复杂度**
- 每个内积操作：\\(O(n)\\)
- 每个循环有 \\(m\\) 次迭代，每次做常数个内积
- 总复杂度：\\(O(mn)\\)

**空间复杂度**
- 存储 \\(m\\) 个 \\(s_i\\) 向量：\\(O(mn)\\)
- 存储 \\(m\\) 个 \\(y_i\\) 向量：\\(O(mn)\\)
- 临时变量：\\(O(n)\\)
- 总复杂度：\\(O(mn)\\)

相比标准 BFGS 的 \\(O(n^2)\\) 存储，两步循环只执行向量运算，因而适合高维 CRF 参数。Wapic 用 `s`、`y` 和 `p` 保存循环历史；`s`、`y` 使用 `float` 减少内存，内积仍以 `float_t` 累加。

**初始 Hessian 逆矩阵的选择**

Wapic 使用标量矩阵作为初始近似：
\\(\gamma = \frac{y_{k-1}^T s_{k-1}}{y_{k-1}^T y_{k-1}}\\)
\\(H_k^0 = \gamma I\\)
\\(r = \gamma q\\)

## OWL-QN

**L1 正则化的挑战**

令 \\(f(\lambda)\\) 表示包含负对数似然和 L2 惩罚的光滑部分，OWL-QN 处理的目标函数为：
$$
F(\lambda) = f(\lambda) + C \sum&#95;k |\lambda&#95;k|
$$

在 Wapic 中，\\(C=r_1\\)。

**问题：**
- L1 正则化项在零点不可微
- 标准 L-BFGS 要求目标函数处处可微
- 需要特殊处理来保持L-BFGS的收敛性质

**OWL-QN 算法**

OWL-QN（Orthant-Wise Limited-memory Quasi-Newton）是L-BFGS在L1 正则化上的扩展。

**伪梯度的定义**

OWL-QN 引入**伪梯度（pseudo-gradient）**的概念来处理不可微性：
$$
\widetilde{\nabla}F(\lambda)&#95;k = \begin{cases}
\nabla f(\lambda)&#95;k + C & \text{if } \lambda&#95;k > 0 \\\\
\nabla f(\lambda)&#95;k - C & \text{if } \lambda&#95;k < 0 \\\\
\nabla f(\lambda)&#95;k + C & \text{if } \lambda&#95;k = 0 \text{ and } \nabla f(\lambda)&#95;k < -C \\\\
\nabla f(\lambda)&#95;k - C & \text{if } \lambda&#95;k = 0 \text{ and } \nabla f(\lambda)&#95;k > C \\\\
0 & \text{if } \lambda&#95;k = 0 \text{ and } |\nabla f(\lambda)&#95;k| \leq C
\end{cases}
$$

这对应 `ComputePseudoGradient`：普通梯度保存在 `g`，伪梯度保存在 `pg`。

**象限约束**

OWL-QN 的关键思想是**象限约束（orthant constraint）**：

**搜索方向约束：**
在计算搜索方向后，逐维保留与负伪梯度一致的分量：
$$
\widetilde d&#95;{k,i} = \begin{cases}
d&#95;{k,i} & \text{if } d&#95;{k,i}(-\widetilde{\nabla}F(\lambda^{(k)})&#95;i) > 0 \\\\
0 & \text{otherwise}
\end{cases}
$$

这对应 `ConstrainSearchDirection`：如果某一维满足 `d[f] * pg[f] >= 0`，就把该方向分量清零。

**象限投影**

在线搜索过程中，需要将候选点投影回当前象限：
$$
\lambda&#95;{\text{projected}} = \text{project}(\lambda^{(k)} + \alpha d&#95;k, \xi)
$$

其中投影操作定义为：
$$
\text{project}(\lambda, \xi)&#95;i = \begin{cases}
\lambda&#95;i & \text{if } \lambda&#95;i \cdot \xi&#95;i > 0 \\\\
0 & \text{otherwise}
\end{cases}
$$

参考象限不能一律取负伪梯度。对已经非零的参数，应保持它当前所在的象限；只有参数为零时，才使用负伪梯度决定准备进入的象限：

$$
\xi&#95;i =
\begin{cases}
\operatorname{sign}(\lambda&#95;i^{(k)})
  & \text{if } \lambda&#95;i^{(k)}\neq 0 \\\\
\operatorname{sign}(-\widetilde{\nabla}F(\lambda^{(k)})&#95;i)
  & \text{if } \lambda&#95;i^{(k)}=0
\end{cases}
$$

`ProjectToOrthant` 使用上一次参数 `xp` 确定非零参数的象限；如果原参数为零，则使用 `-pg` 选择准备进入的象限。

**回溯线搜索**

Wapic 沿用原始 Wapiti 的 OWL-QN 回溯判定。候选点经过象限投影后，代码检查：
$$
F(\lambda&#95;{\text{projected}})
< F(\lambda^{(k)})
{}+ \gamma d&#95;k^T
  (\lambda&#95;{\text{projected}}-\lambda^{(k)})
$$

其中 \\(\gamma=10^{-4}\\)。`CheckArmijoRule` 直接使用搜索方向 `d` 与投影后实际位移的内积；这与标准 Armijo 条件使用目标函数方向导数的写法不同，因此这里把它称为 Wapiti 的回溯判定，而不把两者视为完全相同的公式。

**算法流程**

Wapic 的 `LBFGSOptimizer::Optimize` 在 \\(r_1\neq0\\) 时执行以下流程：

1. **初始化：** 设置初始参数 \\(\lambda^{(0)}\\)，记忆深度 \\(m\\)，正则化参数 \\(C\\)

2. **主循环：** 对于 \\(k = 0, 1, 2, \ldots\\)
   
   a. **计算伪梯度：** \\(g_k=\widetilde{\nabla}F(\lambda^{(k)})\\)
   
   b. **检查收敛：** 如果 \\(\|g_k\|\\) 足够小，则停止
   
   c. **计算搜索方向：** 使用两步循环算法计算 \\(d_k = -H_k g_k\\)
   
   d. **象限约束：** 删除不与负伪梯度同向的搜索方向分量
   
   e. **线搜索：** 使用 Wapiti 的回溯判定寻找步长 \\(\alpha_k\\)
   
   f. **象限投影：** \\(\lambda^{(k+1)} = \text{project}(\lambda^{(k)} + \alpha_k d_k, \xi)\\)
   
   g. **更新历史：** 令 \\(s_k=\lambda^{(k+1)}-\lambda^{(k)}\\)，并用光滑部分的梯度差 \\(y_k=\nabla f(\lambda^{(k+1)})-\nabla f(\lambda^{(k)})\\) 更新历史信息

**实现参数**

- `--rho1`：L1 强度，默认 0.5；设为 0 时关闭 OWL-QN 分支
- `--rho2`：L2 强度，默认 0.0001
- `--histsz`：历史向量对数量，默认 5
- `--maxls`：最大线搜索次数，默认 40
- `--maxiter`：最大训练轮数，默认 100

`CheckConvergence` 同时检查伪梯度范数和一段目标函数历史中的相对改善量。线搜索失败、梯度足够小或目标函数改善不足时，训练停止。

三节的关系可以归结为：L1 正则化定义稀疏目标；L-BFGS 高效优化光滑目标；当训练既需要 L-BFGS 的曲率信息，又需要 L1 稀疏性时，使用 OWL-QN。

至此，中文分词可以被看作两类互补方案：词典方法通过词频寻找最大概率路径，CRF 则通过特征与标签转移进行全局序列预测。它们都可以作为预切分阶段，为后续子词模型提供更稳定的输入边界。

配套实现：[Ismantic/Wapic](https://github.com/Ismantic/Wapic)
