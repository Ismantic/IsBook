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

接下来的一步是要让模型的参数尽量多的能为零，这么做的好处是能让占用的内存更好，运行的速度更快。这就要引入L1 正则化方法。

**梯度公式**

对前文推导出的梯度公式增加对正则项的求导：

$$
\frac{\partial L}{\partial \lambda&#95;k} = \sum&#95;s \left[\sum&#95;i f&#95;k(y&#95;{i-1}^{(s)}, y&#95;i^{(s)}, x^{(s)}, i)\right] - \sum&#95;s E&#95;{P(y|x^{(s)})}\left[\sum&#95;i f&#95;k(y&#95;{i-1}, y&#95;i, x^{(s)}, i)\right] - \frac{\partial R(\lambda)}{\partial \lambda&#95;k}
$$


**标准 L1 正则化方法**

**L1 正则化的定义**

L1 正则化项定义为：
$$
R(\lambda) = C \sum&#95;k |\lambda&#95;k|
$$

其中 \\(C\\) 是正则化强度参数。

**L1 正则化的梯度**

L1 正则化项在非零点可导，但在零点不可导。绝对值函数的次微分为：

$$
\partial |\lambda&#95;k| = \begin{cases}
\{1\} & \text{if } \lambda&#95;k > 0 \\\\
\{-1\} & \text{if } \lambda&#95;k < 0 \\\\
[-1,1] & \text{if } \lambda&#95;k = 0
\end{cases}
$$

在零点取 0，可以写成下面这个具体的次梯度选择：
$$
\text{sign}(\lambda&#95;k) = \begin{cases}
1 & \text{if } \lambda&#95;k > 0 \\\\
-1 & \text{if } \lambda&#95;k < 0 \\\\
0 & \text{if } \lambda&#95;k = 0
\end{cases}
$$

当采用这一选择时，正则项的一个次梯度为：
$$
g&#95;k^{(R)} = C \cdot \text{sign}(\lambda&#95;k)
$$

**批量次梯度法的标准更新**

批量次梯度上升中，可以写出下面的更新：

$$
\lambda&#95;k^{t+1} = \lambda&#95;k^t + \eta \left[\text{经验期望} - \text{模型期望} - C \cdot \text{sign}(\lambda&#95;k)\right]
$$

这种更新并不保证参数恰好停在零点；实际实现通常需要截断、近端更新，或者后文介绍的累计惩罚方法。

不过，实际场景中，批量梯度下降收敛的效率比较低，更多的是使用随机梯度下降。

**随机梯度下降中的问题**

**SGD 的基本更新：** 只使用一个样本来计算梯度：

$$
\lambda&#95;k^{t+1} = \lambda&#95;k^t + \eta \left[\sum&#95;i f&#95;k(y&#95;{i-1}^{(s)}, y&#95;i^{(s)}, x^{(s)}, i) - E&#95;{P(y|x^{(s)})}\left[\sum&#95;i f&#95;k(y&#95;{i-1}, y&#95;i, x^{(s)}, i)\right] - \frac{C}{N} \cdot \text{sign}(\lambda&#95;k)\right]
$$

注意L1 项系数变成了 \\(\frac{C}{N}\\)，以保持与批量方法的期望等价性。

- **问题1：** 计算效率低下
  - **问题描述**：更新需要对**全部特征**都应用 L1 惩罚，即使当前样本中没有使用这些特征。
    - CRF 中，特征空间会有数百万维（ 一元特征 x 标签数 + 二元特征 x 标签数 x 标签数）
    - 通常一个样本（也就是一个句子）只包含很少的特征
    - 对全部特征计算 \\(\text{sign}(\lambda_k)\\) 极其低效
- **问题2：** 稀疏效果差
  - **问题描述：** 由于SGD计算的梯度含噪声，权重很难真正变为0
  - **具体例子：**
    - 当前权重：\\(\lambda_k = 0.001\\)
    - 噪声梯度更新：\\(+0.002\\)
    - L1 惩罚：\\(-\frac{C}{N}\eta = -0.001\\)
    - 最终结果：\\(\lambda_k = 0.001 + 0.002 - 0.001 = 0.002\\)（没变成0！）
  - **根本原因：**
    - SGD 的梯度噪声大，权重容易被噪声"推离"零点
    - 难以在零点稳定下来

**累计惩罚方法**

**引入理论累计惩罚强度值**

首先定义每个权重**理论上应该**接受的总 L1 惩罚强度值：

$$
u^t = \frac{C}{N}\sum&#95;{\tau=1}^t \eta&#95;\tau
$$

**含义**：
- \\(u^t\\) 对所有权重都相同
- 代表"如果没有噪声，每个权重应该接受的总惩罚强度"
- 这是一个全局累积量

**理论依据**：在标准 SGD 中，每次迭代权重应该接受的 L1 惩罚是 \\(\frac{C}{N}\eta_\tau\\)，累积起来就是 \\(u^t\\)。

**跟踪实际累计惩罚**

接着定义每个权重**实际接受**的L1 惩罚：
$$
q&#95;k^t = \sum&#95;{\tau=1}^t (\lambda&#95;k^{\tau+1} - \lambda&#95;k^{\tau+\frac{1}{2}})
$$

**含义**：
- \\(\lambda_k^{\tau+\frac{1}{2}}\\) 是应用梯度更新后、应用 L1 惩罚前的权重
- \\(\lambda_k^{\tau+1}\\) 是最终的权重值
- 差值记录了这次实际应用的L1 惩罚

**累计惩罚更新规则**

当特征 \\(k\\) 在第 \\(t\\) 次迭代中被使用时：

1. **梯度更新**：
   $$
\lambda&#95;k^{t+\frac{1}{2}} = \lambda&#95;k^t + \eta&#95;t \left[\sum&#95;i f&#95;k(y&#95;{i-1}^{(s)}, y&#95;i^{(s)}, x^{(s)}, i) - E&#95;{P(y|x^{(s)})}\left[\sum&#95;i f&#95;k(y&#95;{i-1}, y&#95;i, x^{(s)}, i)\right]\right]
$$

2. **应用累计惩罚**：

   \\(q_k^{t-1}\\) 是有符号的累计修正量，因此正权重和负权重使用的补偿项不同：

   $$
\lambda&#95;k^{t+1} = \begin{cases}
\max(0, \lambda&#95;k^{t+\frac{1}{2}} - (u^t + q&#95;k^{t-1})) & \text{if } \lambda&#95;k^{t+\frac{1}{2}} > 0 \\\\
\min(0, \lambda&#95;k^{t+\frac{1}{2}} + (u^t - q&#95;k^{t-1})) & \text{if } \lambda&#95;k^{t+\frac{1}{2}} < 0
\end{cases}
$$

3. **更新实际惩罚记录**：
   $$
q&#95;k^t = q&#95;k^{t-1} + (\lambda&#95;k^{t+1} - \lambda&#95;k^{t+\frac{1}{2}})
$$

如果梯度更新后的权重恰好为零，则保持为零，再按同样方式更新 \\(q_k\\)。

**注意：** 相比普通 SGD，新方法需要维护一个全局标量 \\(u\\) 和一个与参数等长的向量 \\(q\\)。因此优化器会额外占用约一个参数向量的空间。

**合理性解释**

**核心思想**：累积惩罚方法实际上是一种**惰性更新（Lazy Update）**策略。

**关键洞察**：
- **惩罚不会遗漏**：特征未出现期间积累的 L1 惩罚会在它下次活跃时补齐
- **应用时机延迟**：惩罚不是每次都应用，而是延迟到该特征在当前样本中出现时才应用
- **能够精确归零**：\\(\max(0,\cdot)\\) 和 \\(\min(0,\cdot)\\) 会在权重跨过零点时把它截断为零

**类比理解**：
想象L1 惩罚是每个权重需要缴纳的"税款"：
- **标准方法**：每次迭代都要交税，不管这个权重是否"工作"
- **累积惩罚**：只有当权重"工作"（在样本中出现）时才交税，但要把之前欠的税一起补齐


**好处 1：计算速度大幅提升**

```
标准 SGD每次迭代：
for k in all_features:  # 数百万特征
    weights[k] -= C/N * sign(weights[k])  # O(特征总数)

累积惩罚每次迭代：
for k in active_features:  # 只有几十个特征
    ApplyPenalty(k)  # O(活跃特征数)

单次更新从遍历全部特征变为只遍历活跃特征
```

累计惩罚并不会消除随机梯度中的噪声。它解决的是惰性更新导致的惩罚遗漏，并通过截断操作产生真正的零权重。

## L-BFGS

L-BFGS（Limited-memory Broyden-Fletcher-Goldfarb-Shanno）是一种高效的拟牛顿方法，CRF训练中，L-BFGS通常比SGD具有更快的收敛速度和更好的数值稳定性，不过缺点是占用更多的存储空间，实现起来也会更麻烦些。

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

**梯度下降法**

最基本的优化方法是梯度下降：

$$
\lambda^{(k+1)} = \lambda^{(k)} -\alpha&#95;k \nabla F(\lambda^{(k)})
$$

其中：
- \\(\alpha_k\\) 是学习率（步长）
- \\(\nabla F(\lambda^{(k)})\\) 是损失函数在第 k 次迭代处的梯度

**梯度下降的问题:**
- 对学习率敏感，需要精心调整
- 只利用了一阶导数信息，忽略了函数的曲率信息

**牛顿法**

牛顿法利用二阶导数信息来加速收敛：

$$
\lambda^{(k+1)} = \lambda^{(k)} - 
[\nabla^2 F(\lambda^{(k)})]^{-1} \nabla F(\lambda^{(k)})
$$

其中 \\(\nabla^2 F(\lambda^{(k)})\\) 是 Hessian 矩阵。

**牛顿方向：** \\(-[\nabla^2 F(\lambda^{(k)})]^{-1}\nabla F(\lambda^{(k)})\\) 使用 Hessian 的逆对梯度进行曲率修正。它是一个搜索方向，不是另一种“高级梯度”。

**普通梯度 vs 牛顿方向**

**普通梯度**（梯度下降）：
$$
d^{GD} = -\nabla F(\lambda^{(k)})
$$
- 只告诉我们"哪个方向函数值下降最快"
- 就像只看到脚下的坡度

**牛顿方向**（牛顿法）：
$$
d^{Newton} = -[\nabla^2 F(\lambda^{(k)})]^{-1} \nabla F(\lambda^{(k)})
$$
- 不仅考虑了当前的坡度（一阶导数）
- 还考虑了坡度的变化率（二阶导数）
- 对当前点附近的二次近似而言，它直接指向该近似模型的极小点

**每个分量的直观含义**

牛顿方向的第 \\(i\\) 个分量：
$$
[d^{Newton}]&#95;i
= -\sum&#95;{j=1}^n
  \left([\nabla^2F(\lambda^{(k)})]^{-1}\right)&#95;{ij}
  \frac{\partial F}{\partial \lambda&#95;j}
$$

这意味着：
- 参数 \\(\lambda_i\\) 的更新不仅依赖于 \\(\frac{\partial F}{\partial \lambda_i}\\)
- 还依赖于**所有其他参数的梯度** \\(\frac{\partial F}{\partial \lambda_j}\\)
- 权重由 Hessian 逆矩阵的第 \\((i,j)\\) 个元素决定，反映了参数间的局部耦合

**直观理解：参数间的“协调”**

假设有两个参数 \\(\lambda_1, \lambda_2\\)：

**梯度下降**：
```
λ₁方向：只看 ∂F/∂λ₁，独立更新
λ₂方向：只看 ∂F/∂λ₂，独立更新
```

**牛顿法**：
```
λ₁方向：不仅看 ∂F/∂λ₁，还要考虑：
- λ₁和λ₂如何相互影响？(∂²L/∂λ₁∂λ₂)
- λ₂的梯度信息是什么？(∂F/∂λ₂)
- 如何协调两个参数的更新？
```

**牛顿方向怎样修正梯度？**

这个向量可以理解为：
$$
\text{牛顿方向}
= -\text{Hessian 逆矩阵}\times\text{梯度}
$$

其中 Hessian 逆矩阵起到了"修正器"的作用：
- **放大**某些方向的梯度（在那些方向上函数变化缓慢）
- **缩小**某些方向的梯度（在那些方向上函数变化剧烈）
- **重新分配**不同方向的重要性

牛顿法利用二阶信息修正了一阶搜索方向，但效果依赖局部二次近似和 Hessian 的数值性质。

**牛顿法的问题：**
- 需要计算和存储 Hessian 矩阵，对于 CRF 这样的大规模问题不现实
- 需要求解线性方程组 \\(H^{(k)} d^{(k)} = -\nabla F(\lambda^{(k)})\\)，计算代价高
- Hessian 矩阵可能奇异或病态，直接求解会带来数值问题

**拟牛顿法**

拟牛顿法试图在梯度下降和牛顿法之间找到平衡：
- **核心思想：** 用正定矩阵 \\(B_k\\) 近似 Hessian 矩阵，或用 \\(H_k\\) 近似 Hessian 逆矩阵
- **更新公式：** \\(\lambda^{(k+1)} = \lambda^{(k)} - \alpha_k H_k \nabla F(\lambda^{(k)})\\)

**正定性**

**定义：** 矩阵 \\(H\\) 是正定的，当且仅当对于任意非零向量 \\(v\\)，都有 \\(v^T H v > 0\\)。

**为什么需要正定矩阵？**

答案：因为能保证始终是下降方向。

**下降方向**指的是能够使目标函数值 \\(F(\lambda)\\) 减少的搜索方向。

如果我们从点 \\(\lambda^{(k)}\\) 沿着方向 \\(d_k\\) 走一小步 \\(\alpha\\)（\\(\alpha > 0\\) 且很小），那么新的点是：
$$
\lambda^{new} = \lambda^{(k)} + \alpha d&#95;k
$$

下降方向意味着：
$$
F(\lambda^{new}) < F(\lambda^{(k)})
$$


使用泰勒展开的一阶近似：
$$
F(\lambda^{(k)} + \alpha d&#95;k) \approx F(\lambda^{(k)}) + \alpha \nabla F(\lambda^{(k)})^T d&#95;k
$$

要让函数值下降，需要：
$$
F(\lambda^{(k)} + \alpha d&#95;k) < F(\lambda^{(k)})
$$

将泰勒展开代入：
$$
F(\lambda^{(k)}) + \alpha \nabla F(\lambda^{(k)})^T d&#95;k < F(\lambda^{(k)})
$$

简化得到：
$$
\alpha \nabla F(\lambda^{(k)})^T d&#95;k < 0
$$

因为步长 \\(\alpha > 0\\)，所以下降条件等价于：
$$
\nabla F(\lambda^{(k)})^T d&#95;k < 0
$$


**拟牛顿法的搜索方向：**
$$
d&#95;k = -H&#95;k \nabla F(\lambda^{(k)})
$$

**验证下降条件：**
$$
\nabla F(\lambda^{(k)})^T d&#95;k = \nabla F(\lambda^{(k)})^T \cdot (-H&#95;k \nabla F(\lambda^{(k)}))
$$
$$
= -\nabla F(\lambda^{(k)})^T H&#95;k \nabla F(\lambda^{(k)})
$$

因为 \\(H_k\\) 是正定矩阵，对于任何非零向量 \\(v\\)，都有 \\(v^T H_k v > 0\\)。

当 \\(\nabla F(\lambda^{(k)}) \neq 0\\) 时：
$$
\nabla F(\lambda^{(k)})^T H&#95;k \nabla F(\lambda^{(k)}) > 0
$$

因此：
$$
\nabla F(\lambda^{(k)})^T d&#95;k = -\nabla F(\lambda^{(k)})^T H&#95;k \nabla F(\lambda^{(k)}) < 0
$$

正定矩阵 \\(H_k\\) 保证了搜索方向 \\(d_k = -H_k \nabla F(\lambda^{(k)})\\) 总是满足下降条件，因而确保算法朝着减少目标函数值的方向前进。

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
H&#95;{k+1}
=
(I-\rho&#95;k s&#95;k y&#95;k^T)
H&#95;k
(I-\rho&#95;k y&#95;k s&#95;k^T)
+\rho&#95;k s&#95;k s&#95;k^T
$$

其中：

$$
\rho&#95;k=\frac{1}{y&#95;k^T s&#95;k}
$$

需要特别区分，下面这个公式是 DFP 的逆矩阵更新，而不是 BFGS：

$$
H&#95;{k+1}
=H&#95;k
-\frac{H&#95;k y&#95;k y&#95;k^T H&#95;k}{y&#95;k^T H&#95;k y&#95;k}
+\frac{s&#95;k s&#95;k^T}{y&#95;k^T s&#95;k}
$$

二者都属于拟牛顿方法，但不能视为同一个更新公式。

**曲率条件**

如果 \\(H_k\\) 正定，并且满足：

$$
y&#95;k^T s&#95;k>0
$$

那么 BFGS 更新得到的 \\(H_{k+1}\\) 仍然正定，搜索方向：

$$
d&#95;k=-H&#95;k\nabla F(\lambda^{(k)})
$$

就是下降方向。实践中通常使用满足 Wolfe 条件的线搜索来维持曲率条件；如果 \\(y_k^Ts_k\\) 太小或非正，可以跳过该次更新或采用阻尼策略。

**L-BFGS**

标准 BFGS 需要存储 \\(n \times n\\) 的矩阵，对于大规模问题（\\(n\\) 很大）并不现实。L-BFGS 不显式存储 Hessian 逆矩阵，而是保存最近的 \\(m\\) 组向量对 \\(\{s_k,y_k\}\\)，用它们隐式表示曲率信息。


**核心思想：**
- **内存需求：** 从\\(O(n^2)\\)降低到\\(O(mn)\\)，其中\\(m\\)通常取5-20
- **计算复杂度：** 每次迭代从\\(O(n^2)\\)降低到\\(O(mn)\\)

**算法步骤**

1. **初始化：** 选择初始点\\(\lambda_0\\)，记忆深度\\(m\\)，收敛阈值\\(\varepsilon\\)

2. **计算初始梯度：** \\(g_0 = \nabla F(\lambda_0)\\)

3. **主循环：** 对于\\(k=0,1,2,...\\)直到收敛：
   
   a. **计算搜索方向：** \\(d_k = -H_k g_k\\)（使用两步循环算法计算\\(H_k g_k\\)）
   
   b. **线搜索：** 确定步长\\(\alpha_k\\)
   
   c. **更新参数：** \\(\lambda_{k+1} = \lambda_k + \alpha_k d_k\\)
   
   d. **计算新梯度：** \\(g_{k+1} = \nabla F(\lambda_{k+1})\\)
   
   e. **收敛检查：** 如果\\(\|g_{k+1}\| < \varepsilon\\)，终止
   
   f. **计算差分：** \\(s_k = \lambda_{k+1} - \lambda_k\\)，\\(y_k = g_{k+1} - g_k\\)
   
   g. **更新历史：** 存储\\(\{s_k, y_k\}\\)对，丢弃最旧的超出记忆\\(m\\)的对

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

**算法解释**

**为什么是两步？**

**第一步的作用：**
- 从当前梯度开始，逐步"去除"历史信息中的某些分量
- 每次减去 \\(\alpha_i y_i\\)，相当于在梯度空间中做投影变换
- 得到的 \\(q\\) 是经过"预处理"的梯度

**第二步的作用：**
- 应用初始 Hessian 逆矩阵近似 \\(H_k^0\\)
- 然后逐步"添加回"经过修正的历史信息
- 每次加上 \\(s_i (\alpha_i - \beta)\\)，恢复并修正之前去除的信息

**数学原理**

每一对 \\((s_i,y_i)\\) 都对应一次 BFGS 逆矩阵更新。连续使用最近 \\(m\\) 个向量对时，矩阵中出现的是这些更新变换的有序乘积，不能把它们简单写成若干秩一矩阵的和。两步循环按照与这些更新相同的顺序执行向量运算，因此能直接得到 \\(H_kg_k\\)，而不需要显式构造 \\(H_k\\)。

**详细示例**

假设我们有 \\(m=2\\) 个历史对：\\(\{s_1, y_1\}, \{s_2, y_2\}\\)，当前梯度为 \\(g\\)。

**第一步：反向循环**

**迭代1（i=2）：**
\\(\alpha_2 = \frac{s_2^T g}{y_2^T s_2}\\)
\\(q = g - \alpha_2 y_2\\)

**迭代2（i=1）：**
\\(\alpha_1 = \frac{s_1^T q}{y_1^T s_1}\\)
\\(q = q - \alpha_1 y_1\\)

此时 \\(q = g - \alpha_2 y_2 - \alpha_1 y_1\\)

**第二步：正向循环**

**初始化：**
\\(r = H_0^0 q = \gamma q \quad \text{// 假设 } H_0^0 = \gamma I\\)

**迭代1（i=1）：**
\\(\beta_1 = \frac{y_1^T r}{y_1^T s_1}\\)
\\(r = r + s_1 (\alpha_1 - \beta_1)\\)

**迭代2（i=2）：**
\\(\beta_2 = \frac{y_2^T r}{y_2^T s_2}\\)
\\(r = r + s_2 (\alpha_2 - \beta_2)\\)

**最终结果：**
\\(d = -r\\)

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

相比标准 BFGS的 \\(O(n^2)\\)，这是巨大的改进！

**关键洞察**

1. **不需要矩阵存储：** 整个过程只涉及向量操作
2. **隐式矩阵乘法：** 两步循环等价于一个复杂的矩阵-向量乘法
3. **历史信息利用：** 巧妙地利用了最近 \\(m\\) 步的梯度和参数变化信息
4. **数值稳定：** 避免了显式矩阵求逆，提高了数值稳定性

**初始 Hessian 逆矩阵的选择**

常用的 \\(H_k^0\\) 选择：

**1. 单位矩阵：**
\\(H_k^0 = I\\)
\\(r = q \quad \text{// 直接复制}\\)

**2. 标量矩阵：**
\\(\gamma = \frac{y_{k-1}^T s_{k-1}}{y_{k-1}^T y_{k-1}}\\)
\\(H_k^0 = \gamma I\\)
\\(r = \gamma \cdot q \quad \text{// 标量乘法}\\)

第二种选择通常效果更好，因为它考虑了函数的局部尺度信息。

## OWL-QN

**L1 正则化的挑战**

令 \\(f(\lambda)\\) 表示需要最小化的光滑负对数似然，OWL-QN 处理的目标函数为：
$$
F(\lambda) = f(\lambda) + C \sum&#95;k |\lambda&#95;k|
$$

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

这确保了搜索方向指向能够减少目标函数值的方向。

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

**修改的线搜索条件**

OWL-QN使用修改的Armijo条件而非标准Wolfe条件：
$$
F(\lambda&#95;{\text{projected}})
\leq F(\lambda^{(k)})
{}+ \gamma\widetilde{\nabla}F(\lambda^{(k)})^T
  (\lambda&#95;{\text{projected}}-\lambda^{(k)})
$$

其中 \\(\gamma\\) 是Armijo参数，典型值为 \\(10^{-4}\\)。

**算法流程**

**OWL-QN 的完整算法流程：**

1. **初始化：** 设置初始参数 \\(\lambda^{(0)}\\)，记忆深度 \\(m\\)，正则化参数 \\(C\\)

2. **主循环：** 对于 \\(k = 0, 1, 2, \ldots\\)
   
   a. **计算伪梯度：** \\(g_k=\widetilde{\nabla}F(\lambda^{(k)})\\)
   
   b. **检查收敛：** 如果 \\(\|g_k\|\\) 足够小，则停止
   
   c. **计算搜索方向：** 使用两步循环算法计算 \\(d_k = -H_k g_k\\)
   
   d. **象限约束：** 删除不与负伪梯度同向的搜索方向分量
   
   e. **线搜索：** 使用修改的 Armijo 条件找到合适步长 \\(\alpha_k\\)
   
   f. **象限投影：** \\(\lambda^{(k+1)} = \text{project}(\lambda^{(k)} + \alpha_k d_k, \xi)\\)
   
   g. **更新历史：** 令 \\(s_k=\lambda^{(k+1)}-\lambda^{(k)}\\)，并用光滑部分的梯度差 \\(y_k=\nabla f(\lambda^{(k+1)})-\nabla f(\lambda^{(k)})\\) 更新历史信息

**收敛与实践表现**

OWL-QN 是针对“光滑凸损失加 L1 正则项”的工程算法，实践中能够利用 L-BFGS 的曲率信息并产生稀疏解。不过，不能直接把光滑 L-BFGS 的超线性收敛结论搬到原始 OWL-QN 上；原始算法的严格全局收敛证明也存在已知缺口。因此，这里只把它作为实践有效的优化方法，而不声称无条件的理论收敛速度。

**参数调优建议**

**正则化参数 C：**
- 通过交叉验证选择
- 起始范围：\\([10^{-3}, 10^{1}]\\)
- 较大的C产生更稀疏的模型

**记忆深度 m：**
- 对于L1 正则化，\\(m = 5-10\\) 通常足够
- 过大的m可能导致数值不稳定

**线搜索参数：**
- Armijo参数：\\(\gamma = 10^{-4}\\)
- 最大线搜索步数：20-50
- 初始步长：1.0
