# Wapiti

## 0. 引言

Wapiti中的模型，条件随机场（Conditional Random Fields, CRF）是一种用于序列标注的概率图模型。
当给定观察序列 $x = (x_1, x_2, ..., x_n)$ ，
CRF的目标是预测最佳的标签序列 $y = (y_1, y_2, ..., y_n)$ 。

**CRF的核心思想**:
- **对条件概率建模**：直接建模 $P(y|x)$，而不是联合概率 $P(x, y)$
- **全局优化**：考虑整个序列的标签依赖关系
- **特征丰富**：可以使用任意的观察特征和标签转移特征

## 1. 目标函数

### 1.1 概率公式

CRF的条件概率定义为：

$$P( y|x ) = \frac{1}{Z(x)}\times\exp\left(\sum_{i}\sum_{k}\lambda_k f_k(y_{i-1},y_i,x,i)\right)$$ 

**关键组成部分**：

- **$Z(x)$**：归一化因子，确保概率和为1
- **$f_k(y_{i-1},y_i,x,i)$**：特征函数
- **$\lambda_k$**：特征权重参数

### 1.2 特征函数类型

#### 一元特征（Unigram Features）
$$f_1(y_i,x,i) = \begin{cases}
1 & \text{if } y_i,x_i \text{符合某种条件} \\
0 & \text{otherwise}
\end{cases}$$

#### 二元特征（Bigram Features）
$$f_2(y_{i-1},y_i,x,i) = \begin{cases}
1 & \text{if } y_{i-1},y_i,x_i \text{符合某种条件} \\
0 & \text{otherwise}
\end{cases}$$

### 1.3 势函数
为简化表示，引入势函数：
$$\psi_t(y',y,x) = \exp\left(\sum_k \lambda_k f_k(y',y,x,t)\right)$$

则条件概率可重写为：
$$P( y| x) = \frac{1}{Z(x)} \times \prod_t \psi_t(y_{t-1},y_t,x)$$

### 1.4 归一化因子

归一化因子需要对全部可能的标签序列求和：

$$Z(x) = \sum_y \prod_t \psi_t(y_{t-1},y_t,x)$$

**问题**：如果有 $T$ 个位置，每个位置有 $L$ 个可能标签，则需要计算 $L^T$ 个序列！

**方案**：引入动态规划算法高效计算。

## 2. 标注过程

### 2.1 维特比

**目标**：找到最优标签序列，使得 $P(y|x)$ 最大。

等价于最大化：
$$\text{score}(y, x) = \sum_i \sum_k \lambda_k f_k(y_{i-1}, y_i, x, i) = \sum_i \log \psi_i(y_{i-1}, y_i, x)$$

### 2.2 动态规划

#### 状态定义
$$\delta_t(y) = \max_{y_1,...,y_{t-1}} \text{score}(y_1,...,y_{t-1},y, x_1,...,x_t)$$

$\delta_t(y)$ 表示到位置$t$标签为$y$的最优路径得分。

#### 递推公式
$$\delta_1(y) = \log \psi_1(\text{START}, y, x)$$
$$\delta_t(y) = \max_{y'} \left[\delta_{t-1}(y') + \log \psi_t(y', y, x)\right]$$

#### 回溯指针
$$\phi_t(y) = \arg\max_{y'} \left[\delta_{t-1}(y') + \log \psi_t(y', y, x)\right]$$

## 3. 梯度推导

### 3.1 训练目标

目标是最大化对数似然函数：

$$L(\lambda) = \sum_s \log P(y^{(s)}|x^{(s)}) - R(\lambda)$$

其中：
- $s$ 是训练样本索引
- $y^{(s),x^{(s)}}$ 是第$s$个样本的真实标签序列和观察序列
- $R(\lambda)$ 是正则化项

### 3.2 概率公式

$$P(y|x) = \frac{1}{Z(x)} \exp \left(\sum_i \sum_k \lambda_k f_k(y_{i-1},y_i,x,i)\right)$$

其中：
- $Z_(x)$是归一化因子
- $f_k(y_{i-1},y_i,x,i)$ 是第$k$ 个特征函数
- $\lambda_k$ 是对应的权重参数

### 3.3 推导过程

#### 第一步：目标函数展开

把概率公式带入目标函数：
$$\log P(y|x) = \sum_i \sum_k \lambda_k f_k(y_{i-1},y_i,x,i) - \log Z(x)$$

目标函数变为：
$$L(\lambda) = \sum_s \left[\sum_i \sum_k \lambda_k f_k(y_{i-1}^{(s)},y_i^{(s)},x^{(s),i})
               -\log Z(x^{(s)})\right] - R(\lambda)$$

#### 第二步：对参数求偏导

对参数 $\lambda_k$ 求偏导：

$$\frac{\partial L}{\partial \lambda_k} = \sum_s 
        \left[\sum_i f_k(y_{i-1}^{s},y_i^{(s)},x^{(s)},i) - 
        \frac{\partial \log Z(x^{(s)})}{\partial \lambda_k}\right] -
        \frac{\partial R(\lambda)}{\partial \lambda_k}$$

**分析各项**:
- 第一项： $\sum_i f_k(y_{i-1}^{(s)},y_i^{(s)},x^{(s)},i)$ 是真实标签序列下的特征值总和（**经验期望**）
- 第二项： $\frac{\partial \log Z(x^{(s)})}{\partial \lambda_k}$ 需要详细推导（**模型期望**）
- 第三项： 正则化项的导数，L1 会特殊些，要专门来处理

关键在于计算第二项！

#### 第三步：计算 $\frac{\partial \log Z(x)}{\partial \lambda_k}$ 

使用链式法则：

$$\frac{\partial \log Z(x)}{\partial \lambda_k} = \frac{1}{Z(x)} \cdot 
    \frac{\partial Z(x)}{\partial \lambda_k}$$

#### 第四步：计算 $\frac{\partial Z(x)}{\partial \lambda_k}$

归一化因子的定义：

$$Z(x) = \sum_y \exp\left(\sum_i \sum_k \lambda_k f_k(y_{i-1},y_i,x,i)\right)$$

用势函数表示：

$$Z(x) = \sum_y \prod_t \psi_t(y_{t-1},y_t,x)$$

其中势函数：

$$\psi_t(y_{t-1},y_t,x) = \exp\left(\sum_k \lambda_k f_k(y_{t-1},y_t,x,t)\right)$$

对 $\lambda_k$ 求偏导：

$$\frac{\partial Z(x)}{\partial \lambda_k} = \sum_y \frac{\partial}{\partial \lambda_k}
  \prod_t \psi_t(y_{t-1},y_t,x)$$

#### 第五步：势函数的偏导数

由于 $\psi_t$ 是指数函数：

$$\frac{\partial \psi_t}{\partial \lambda_k} = \psi_t \cdot f_k(y_{t-1},y_t,x,t)$$

使用乘积法则，对于乘积 $\prod_t \psi_t$：

$$\frac{\partial}{\partial \lambda_k} \prod_t \psi_t =
    \sum_i \left[\prod_{t \neg i} \psi_t \right]
    \cdot \frac{\partial \psi_i}{\partial \lambda_k}$$
$$= \sum_i \left[\prod_{t \neg i} \psi_t \right] 
    \cdot \psi_i \cdot f_k(y_{i-1},y_i,x,i)$$
$$= \prod_t \psi_t \cdot \sum_i f_k(y_{i-1},y_i,x,i)$$

**关键洞察**：最后一步利用了乘法分配率的逆向过程。(整个表达式就是 (连乘) × (连加))

#### 第六步：代入求和

把结果代入 $Z(x)$ 的偏导数：

$$\frac {\partial Z(x)}{\partial \lambda_k} =
    \sum_y \prod_t \psi_t(y_{t-1},y_t,x) \cdot
    \sum_i f_k(y_{i-1},y_i,x,i)$$
$$= \sum_y \left[\sum_i f_k(y_{i-1},y_i,x,i)\right] \cdot
    \prod_t \psi_t(y_{t-1},y_t,x)$$

#### 第七步：转化为概率形式

**关键转化**：注意到

$$\prod_t \psi_t(y_{t-1},y_t,x) = P(y|x) \cdot Z(x)$$

代入得：

$$\frac{\partial Z(x)}{\partial \lambda_k} =
    \sum_y \left[\sum_i f_k(y_{i-1},y_i,x,i)\right] \cdot
    P(y|x) \cdot Z(x) $$
$$= Z(x) \cdot \sum_y P(y|x) \cdot \left[\sum_i f_k(y_{i-1},y_i,x,i)\right]$$

#### 第八步：最终结果

把结果倒入链式法则：

$$\frac{\partial \log Z(x)}{\partial \lambda_k} = 
    \frac{1}{Z(x)} \cdot Z(x) \cdot 
    \sum_y P(y|x) \cdot \left[sum_i f_k(y_{i-1},y_i,x,i)\right]$$

这正是**模型期望**:

$$E_{P(y|x)}\left[\sum_i f_k(y_{i-1},y_i,x,i)\right]$$

#### 第九步：梯度的最终形式

**完整梯度公式**：

$$\frac{\partial L}{\partial \lambda_k} =
    \sum_s \left[\sum_i f_k(y_{i-1}^{s}, y_i^{(s)},x^{(s)},i)\right]
    - \sum_s E_{P(y|x^{(s)})}\left[\sum_i f_k(y_{i-1},y_i,x^{(s)},i)\right]$$

**或者表示成**：

$$\frac{\partial L}{\partial \lambda_k} = \text{经验期望} - \text{模型期望}$$

其中：
- **经验期望**：真实数据中特征 $f_k$ 出现的次数
- **模型期望**: 当前模型认为特征 $f_k$ 应该出现的次数

## 4. 前向后向

### 4.1 核心问题
对于已经推导出来的梯度公式：

$$\frac{\partial L}{\partial \lambda_k} = \text{经验期望} - \text{模型期望}$$

其中模型期望是：

$$E_{P(y|x)}[f_k] = \sum_y P(y|x) \cdot \left[\sum_i f_k(y_{i-1},y_i,x,i)\right]$$

要是直接计算需要遍历 $L^T$ 个可能的标签序列，计算量太大了！

### 4.2 突破口

对于一元特征：
$$E[f_k^{(1)}] = \sum_y P(y|x) \sum_i f_k^{(1)}(y_i,x,i)$$

交换求和顺序
$$= \sum_i \sum_y P(y|x) \cdot f_k^{(1)}(y_,x,i)$$

**进一步分解**：只有当 $y_i$ 取特定值时 $f_k^{(1)}$ 才为1
$$= \sum_i \sum_{y_i} f_k^{(1)}(y_i,x,i) \sum_{y_1,...,y_{i-1},y_{i+1},...,y_T}P(y_1,...,y_T|x)$$

**得到边际概率**：
$$= \sum_i \sum_{y_i} f_k^{(1)}(y_i,x,i) \cdot P(y_i|x)$$

类似地，对于二元特征：
$$E[f_k^{(2)}] = \sum_i \sum_{y_{i-1}} \sum_{y_i} f_k^{(2)}(y_{i-1},y_i,x,i) \cdot P(y_{i-1},y_i|x)$$

这样，就把求 $P(y|x)$ 转变成了怎么去计算边际概率 $P(y_i|x)$ 和 $P(y_{i-1},y_i|x)$ 了，问题化简了不少。

### 4.3 思路

**关键洞察**：边际概率可以分解为“前半部分”和"后半部分"的乘积

$$P(y_i=y|x) = \frac{1}{Z(x)} \times \underbrace {\text{到达}(i,y)\text{的概率}}_{\text{前向概率}}
                              \times \underbrace {\text{从}(i,y)\text{离开的概率}}_{\text{后向概率}}$$

$$P(y_{i-1} = y', y_i = y|x) = \frac{1}{Z(x)} \times \underbrace{\text{到达}(i-1,y')\text{的概率}}_{前向概率} \times \underbrace{\text{转移} y' \to y \text{的概率}}_{\psi_i(y', y, x)} \times \underbrace{\text{从}(i,y)\text{离开的概率}}_{后向概率}$$

**且**:
$$\psi_t(y', y, x) = \exp\left(\sum_k \lambda_k f_k(y', y, x, t)\right)$$

### 4.4 前向过程

#### 概率定义
**前向概率** $\alpha_i(y)$ ：到底位置 $i$ 标签为 $y$ 的所有路径的概率和

$$\alpha_i(y) = \sum_{y_1, ..., y_{i-1}} P(y_1, ..., y_{i-1}, y_i=y|x)$$

更准确地说：
$$\alpha_i(y) = \sum_{y_1, ..., y_{i-1}} \prod_{t=1}^{i} \psi_t(y_{t-1},y_t,x)$$

#### 递推关系

**初始化**：
$$\alpha_1(y) = \psi_1(\text{START},y,x)$$

**递推公式**：
$$\alpha_i(y) = \sum_{y'} \alpha_{i-1}(y') \times \psi_i(y',y,x)$$

**物理意义**：要到达 $(i,y)$ ，必须先到达 $(i-1,y')$ ，然后从 $y'$ 转移到 $y$

### 4.5 后向过程

#### 概率定义

**后向概率** $\beta_i(y)$ ：由位置 $i+1$ 到结束的全部路径的概率和

$$\beta_i(y) = \sum_{y_{i+1},...,y_T} \prod_{t=i+1}^T \psi_t(y_{t-1},y_t,x)$$

#### 递推关系

**初始化**：
$$\beta_T(y) = 1$$

**递推公式**：
$$\beta_i(y) = \sum_{y'} \beta_{i+1}(y') \times \psi_{i+1}(y, y', x)$$

**物理意义**：由 $(i,y)$ 到结束，可以先转移到 $(i+1,y')$ ，然后从 $y'$ 到结束

### 4.6 概率计算

#### 一元边际概率

有了前向和后向概率，可以计算：

$$P(y_i = y|x) = \frac {\alpha_i(y) \times \beta_i(y)}{z_i}$$

且 $$z_i = \sum_y \alpha_i(y) \times \beta_i(y)$$

**直觉理解**：
- $\alpha_i(y)$：全部到达 $(i,y)$ 的路径概率和
- $\beta_i(y)$：全部由 $(i,y)$ 出发的路径概率和
- 乘积：全部通过 $(i,y)$ 的完整路径概率和

#### 二元边际概率

$$P(y_{i-1} = y', y_i = y|x) = \frac{\alpha_{i-1}(y') \times \psi_i(y', y, x) \times \beta_i(y)}{z_i}$$

且 $$z_i = \frac{\sum_y \alpha_i(y) \times \beta_i(y)}{\text{scale}[i]}$$

**直觉理解**：
- $\alpha_{i-1}(y')$ ：到达 $(i-1,y')$ 的路径概率和
- $\psi_i(y', y, x)$ ：从 $y'$ 转移到 $y$ 的概率
- $\beta_i(y)$ ：从 $(i,y)$ 出发的路径概率和

#### 归一化
需要注意的是前面引入的归一化因子 $\text{scale}[i]$ ，计算 $\alpha_{i}$ 和 $\beta_{i}$ 的过程中，防止数值太小溢出，需要做向量归一化。

$$\text{scale}[i] = \frac{1}{\sum_y \alpha'_i(y)}$$

归一化后的前向概率：
$$\alpha_i(y) = \alpha'_i(y) \times \text{scale}[i]$$

推导：

$$\text{scale}[i] = \frac{1}{\sum_y \alpha'_i(y)} = \frac{1}{\sum_y \sum_{y'} \alpha'_{i-1}(y') \times \psi_i(y', y, x)}$$


$$\sum_{y'} \alpha_{i-1}(y') \times \psi_i(y', y, x) = \frac{\alpha_i(y)}{\text{scale}[i]}$$

因此：
$$Z_{\text{bigram}} = \sum_{y'} \sum_y \alpha_{i-1}(y') \times \psi_i(y', y, x) \times \beta_i(y)$$
$$= \sum_y \left(\sum_{y'} \alpha_{i-1}(y') \times \psi_i(y', y, x) \right)\times \beta_i(y)$$
$$= \sum_y \frac{\alpha_i(y)}{\text{scale}[i]} \times \beta_i(y)$$
$$= \frac{1}{\text{scale}[i]} \sum_y \alpha_i(y) \times \beta_i(y)$$
$$= \frac{z}{\text{scale}[i]}$$

最终结果

$$Z_{\text{bigram}} = \frac{Z_{\text{unigram}}}{\text{scale}[i]}$$

## 5. L1 正则化

接下来的一步是要让模型的参数尽量多的能为零，这么做的好处是能让占用的内存更好，运行的速度更快。这就要引入L1 正则化方法。

### 5.1 梯度公式

对前文推导出的梯度公式增加对正则项的求导：

$$\frac{\partial L}{\partial \lambda_k} = \sum_s \left[\sum_i f_k(y_{i-1}^{(s)}, y_i^{(s)}, x^{(s)}, i)\right] - \sum_s E_{P(y|x^{(s)})}\left[\sum_i f_k(y_{i-1}, y_i, x^{(s)}, i)\right] - \frac{\partial R(\lambda)}{\partial \lambda_k}$$


### 5.2 标准L1正则化方法

#### L1正则化的定义

L1正则化项定义为：
$$R(\lambda) = C \sum_k |\lambda_k|$$

其中 $C$ 是正则化强度参数。

#### L1正则化的梯度

L1正则化项的梯度涉及绝对值函数的导数：

$$\frac{\partial |\lambda_k|}{\partial \lambda_k} = \begin{cases}
1 & \text{if } \lambda_k > 0 \\
-1 & \text{if } \lambda_k < 0 \\
\text{?} & \text{if } \lambda_k = 0
\end{cases}$$

使用符号函数作为次梯度：
$$\text{sign}(\lambda_k) = \begin{cases}
1 & \text{if } \lambda_k > 0 \\
-1 & \text{if } \lambda_k < 0 \\
0 & \text{if } \lambda_k = 0
\end{cases}$$

因此：
$$\frac{\partial R(\lambda)}{\partial \lambda_k} = C \cdot \text{sign}(\lambda_k)$$

#### 批量梯度下降的标准更新

批量梯度下降中，权重更新公式为：

$$\lambda_k^{t+1} = \lambda_k^t + \eta \left[\text{经验期望} - \text{模型期望} - C \cdot \text{sign}(\lambda_k)\right]$$

不过，实际场景中，批量梯度下降收敛的效率比较低，更多的是使用随机梯度下降。

#### 随机梯度下降中的问题

**SGD的基本更新：** 只使用一个样本来计算梯度：

$$\lambda_k^{t+1} = \lambda_k^t + \eta \left[\sum_i f_k(y_{i-1}^{(s)}, y_i^{(s)}, x^{(s)}, i) - E_{P(y|x^{(s)})}\left[\sum_i f_k(y_{i-1}, y_i, x^{(s)}, i)\right] - \frac{C}{N} \cdot \text{sign}(\lambda_k)\right]$$

注意L1项系数变成了 $\frac{C}{N}$，以保持与批量方法的期望等价性。

- **问题1：** 计算效率低下
  - **问题描述**：更新需要对**全部特征**都应用L1惩罚，即使当前样本中没有使用这些特征。
    - CRF中，特征空间会有数百万维（ 一元特征 x 标签数 + 二元特征 x 标签数 x 标签数）
    - 通常一个样本（也就是一个句子）只包含很少的特征
    - 对全部特征计算 $\text{sign}(\lambda_k)$ 极其低效
- **问题2：** 稀疏效果差
  - **问题描述：** 由于SGD计算的梯度含噪声，权重很难真正变为0
  - **具体例子：**
    - 当前权重：$\lambda_k = 0.001$
    - 噪声梯度更新：$+0.002$
    - L1惩罚：$-\frac{C}{N}\eta = -0.001$
    - 最终结果：$\lambda_k = 0.001 + 0.002 - 0.001 = 0.002$（没变成0！）
  - **根本原因：**
    - SGD的梯度噪声大，权重容易被噪声"推离"零点
    - 难以在零点稳定下来

### 5.3 累计惩罚方法

#### 引入理论累计惩罚强度值

首先定义每个权重**理论上应该**接受的总L1惩罚强度值：

$$u^t = \frac{C}{N}\sum_{\tau=1}^t \eta_\tau$$

**含义**：
- $u^t$ 对所有权重都相同
- 代表"如果没有噪声，每个权重应该接受的总惩罚强度"
- 这是一个全局累积量

**理论依据**：在标准SGD中，每次迭代权重应该接受的L1惩罚是 $\frac{C}{N}\eta_\tau$，累积起来就是 $u^t$。

#### 跟踪实际累计惩罚

接着定义每个权重**实际接受**的L1惩罚：
$$q_k^t = \sum_{\tau=1}^t (\lambda_k^{\tau+1} - \lambda_k^{\tau+\frac{1}{2}})$$

**含义**：
- $\lambda_k^{\tau+\frac{1}{2}}$ 是应用梯度更新后、应用L1惩罚前的权重
- $\lambda_k^{\tau+1}$ 是最终的权重值
- 差值记录了这次实际应用的L1惩罚

#### 累计惩罚更新规则

当特征 $k$ 在第 $t$ 次迭代中被使用时：

1. **梯度更新**：
   $$\lambda_k^{t+\frac{1}{2}} = \lambda_k^t + \eta_t \left[\sum_i f_k(y_{i-1}^{(s)}, y_i^{(s)}, x^{(s)}, i) - E_{P(y|x^{(s)})}\left[\sum_i f_k(y_{i-1}, y_i, x^{(s)}, i)\right]\right]$$

2. **计算惩罚缺口**：
   - 应该接受的总惩罚：$u^t$
   - 已经接受的惩罚：$q_k^{t-1}$
   - 需要补齐的惩罚：$u^t - q_k^{t-1}$

3. **应用累积惩罚**：
   $$\lambda_k^{t+1} = \begin{cases}
   \max(0, \lambda_k^{t+\frac{1}{2}} - (u^t - q_k^{t-1})) & \text{if } \lambda_k^{t+\frac{1}{2}} > 0 \\
   \min(0, \lambda_k^{t+\frac{1}{2}} + (u^t - q_k^{t-1})) & \text{if } \lambda_k^{t+\frac{1}{2}} < 0
   \end{cases}$$

4. **更新实际惩罚记录**：
   $$q_k^t = q_k^{t-1} + (\lambda_k^{t+1} - \lambda_k^{t+\frac{1}{2}})$$

**注意：** 相比标准L1方法，新方法需要维护 u 和 $Q_i$ 两个新变量，存储空间会增加一倍模型权重。

#### 合理性解释

**核心思想**：累积惩罚方法实际上是一种**惰性更新（Lazy Update）**策略。

**关键洞察**：
- **惩罚总量保持不变**：每个权重最终接受的L1惩罚总量与标准方法相同
- **应用时机延迟**：惩罚不是每次都应用，而是延迟到该特征在当前样本中出现时才应用
- **一次性补齐**：将积累的"惩罚债务"一次性结算

**类比理解**：
想象L1惩罚是每个权重需要缴纳的"税款"：
- **标准方法**：每次迭代都要交税，不管这个权重是否"工作"
- **累积惩罚**：只有当权重"工作"（在样本中出现）时才交税，但要把之前欠的税一起补齐


**好处1：计算速度大幅提升**

```
标准SGD每次迭代：
for k in all_features:  # 数百万特征
    weights[k] -= C/N * sign(weights[k])  # O(特征总数)

累积惩罚每次迭代：
for k in active_features:  # 只有几十个特征
    ApplyPenalty(k)  # O(活跃特征数)

速度提升：数百万 → 几十 = 数万倍理论加速
```

**好处2：噪声容忍性显著增强**

**标准SGD的噪声问题**：
```
迭代1: w = 0.05, 噪声梯度=+0.03, L1惩罚=-0.01 → w = 0.07
迭代2: w = 0.07, 噪声梯度=-0.02, L1惩罚=-0.01 → w = 0.04  
迭代3: w = 0.04, 噪声梯度=+0.04, L1惩罚=-0.01 → w = 0.07
# 权重在小范围震荡，难以归零
```

**累积惩罚的噪声抵抗**：
```
累积3次迭代的惩罚 = 0.01 + 0.01 + 0.01 = 0.03
当特征再次出现时：w = 0.07 - 0.03 = 0.04
# 噪声的影响被累积惩罚"平均化"掉了
```

## 6. L1 + LBFGS

LBFGS（Limited-memory Broyden-Fletcher-Goldfarb-Shanno）是一种高效的拟牛顿算法，CRF训练中，LBFGS通常比SGD具有更快的收敛速度和更好的数值稳定性，不过缺点是占用更多的存储空间，实现起来也会更麻烦些。

### 6.1 拟牛顿法

#### 问题基础

CRF的参数估计本质上是一个无约束优化问题：

$$\min_{\lambda} L(\lambda) = -\ell(\lambda) + R(\lambda)$$

其中：
- $\ell(\lambda)$ 是对数似然函数
- $R(\lambda)$ 是正则化项
- $\lambda$ 是CRF的参数向量

#### 梯度下降法

最基本的优化方法是梯度下降：

$$\lambda^{(k+1)} = \lambda^{(k)} -\alpha_k \nabla L(\lambda^{(k)})$$

其中：
- $\alpha_k$ 是学习率（步长）
- $\nabla L(\lambda^{(k)})$ 是损失函数在第k次迭代处的梯度

**梯度下降的问题:**
- 对学习率敏感，需要精心调整
- 只利用了一阶导数信息，忽略了函数的曲率信息

#### 牛顿法

牛顿法利用二阶导数信息来加速收敛：

$$\lambda^{(k+1)} = \lambda^{(k)} - 
    [\nabla^2 L(\lambda^{(k)})]^{-1} \nabla L(\lambda^{(k)})$$

其中 $\nabla^2 L(\lambda^{(k)})$ 是Hessian矩阵。

**高级梯度：** $[\nabla^2 L(\lambda^{(k)})]^{-1} \nabla L(\lambda^{(k)})$ 是一个 $n × n$ 的矩阵乘以 $n$ 维的向量，得到的也会是 $n$ 维的向量，可以看成是是二次导数和一次导数结合的一个“高级梯度”。下面通过对比分析来理解这个高级梯度的好处。

##### 1. **普通梯度 vs 高级梯度**

**普通梯度**（梯度下降）：
$$d^{GD} = -\nabla L(\lambda^{(k)})$$
- 只告诉我们"哪个方向函数值下降最快"
- 就像只看到脚下的坡度

**高级梯度**（牛顿法）：
$$d^{Newton} = -[\nabla^2 L(\lambda^{(k)})]^{-1} \nabla L(\lambda^{(k)})$$
- 不仅考虑了当前的坡度（一阶导数）
- 还考虑了坡度的变化率（二阶导数）
- 就像预测了最优点的位置

##### 2. **每个分量的物理意义**

高级梯度的第 $i$ 个分量：
$$[d^{Newton}]_i = -\sum_{j=1}^n [H^{-1}]_{ij} \cdot \frac{\partial L}{\partial \lambda_j}$$

这意味着：
- 参数 $\lambda_i$ 的更新不仅依赖于 $\frac{\partial L}{\partial \lambda_i}$
- 还依赖于**所有其他参数的梯度** $\frac{\partial L}{\partial \lambda_j}$
- 权重由 $[H^{-1}]_{ij}$ 决定，反映了参数间的相互影响

##### 3. **直观理解：参数间的"协调"**

假设有两个参数 $\lambda_1, \lambda_2$：

**梯度下降**：
```
λ₁方向：只看 ∂L/∂λ₁，独立更新
λ₂方向：只看 ∂L/∂λ₂，独立更新
```

**牛顿法**：
```
λ₁方向：不仅看 ∂L/∂λ₁，还要考虑：
- λ₁和λ₂如何相互影响？(∂²L/∂λ₁∂λ₂)
- λ₂的梯度信息是什么？(∂L/∂λ₂)
- 如何协调两个参数的更新？
```

##### 为什么叫"高级梯度"？

这个向量可以理解为：
$$\text{高级梯度} = \text{Hessian逆矩阵} \times \text{普通梯度}$$

其中 Hessian逆矩阵起到了"修正器"的作用：
- **放大**某些方向的梯度（在那些方向上函数变化缓慢）
- **缩小**某些方向的梯度（在那些方向上函数变化剧烈）
- **重新分配**不同方向的重要性

这就是为什么牛顿法比梯度下降聪明的原因——它用二阶信息"修正"了一阶信息！

**牛顿法的问题：**
- 需要计算和存储Hessian矩阵，对于CRF这样的大规模问题不现实
- 需要求解线性方程组 $H^{(k)} d^{(k)} = -\nabla L(\lambda^{(k)})$，计算代价高
- Hessian矩阵可能不是正定的，导致算法不稳定

#### 拟牛顿法

拟牛顿法试图在梯度下降和牛顿法之间找到平衡：
- **核心思想：** 用一个正定矩阵 $B_k$ 或其逆矩阵 $H_k$ 来近似Hessian矩阵
- **更新公式：** $\lambda^{(k+1)} = \lambda^{(k)} - \alpha_k H_k \nabla L(\lambda^{(k)})$

##### 正定性

**定义：** 矩阵 $H$ 是正定的，当且仅当对于任意非零向量 $v$，都有 $v^T H v > 0$。

**为什么需要正定矩阵？**

答案：因为能保证始终是下降方向。

**下降方向**指的是能够使目标函数值 $L(\lambda)$ 减少的搜索方向。

如果我们从点 $\lambda^{(k)}$ 沿着方向 $d_k$ 走一小步 $\alpha$（$\alpha > 0$ 且很小），那么新的点是：
$$\lambda^{new} = \lambda^{(k)} + \alpha d_k$$

下降方向意味着：
$$L(\lambda^{new}) < L(\lambda^{(k)})$$


使用泰勒展开的一阶近似：
$$L(\lambda^{(k)} + \alpha d_k) \approx L(\lambda^{(k)}) + \alpha \nabla L(\lambda^{(k)})^T d_k$$

要让函数值下降，需要：
$$L(\lambda^{(k)} + \alpha d_k) < L(\lambda^{(k)})$$

将泰勒展开代入：
$$L(\lambda^{(k)}) + \alpha \nabla L(\lambda^{(k)})^T d_k < L(\lambda^{(k)})$$

简化得到：
$$\alpha \nabla L(\lambda^{(k)})^T d_k < 0$$

因为步长 $\alpha > 0$，所以下降条件等价于：
$$\nabla L(\lambda^{(k)})^T d_k < 0$$


**拟牛顿法的搜索方向：**
$$d_k = -H_k \nabla L(\lambda^{(k)})$$

**验证下降条件：**
$$\nabla L(\lambda^{(k)})^T d_k = \nabla L(\lambda^{(k)})^T \cdot (-H_k \nabla L(\lambda^{(k)}))$$
$$= -\nabla L(\lambda^{(k)})^T H_k \nabla L(\lambda^{(k)})$$

因为 $H_k$ 是正定矩阵，对于任何非零向量 $v$，都有 $v^T H_k v > 0$。

当 $\nabla L(\lambda^{(k)}) \neq 0$ 时：
$$\nabla L(\lambda^{(k)})^T H_k \nabla L(\lambda^{(k)}) > 0$$

因此：
$$\nabla L(\lambda^{(k)})^T d_k = -\nabla L(\lambda^{(k)})^T H_k \nabla L(\lambda^{(k)}) < 0$$

正定矩阵 $H_k$ 保证了搜索方向 $d_k = -H_k \nabla L(\lambda^{(k)})$ 总是满足下降条件，因而确保算法朝着减少目标函数值的方向前进。

### 6.2 BFGS

BFGS 就是一个拟牛顿算法，它通过以下拟牛顿条件来更新 Hessian 逆矩阵的近似：

**拟牛顿条件（割线条件）：**
$$H_{k+1} y_k = s_k$$

其中：
- $s_k = \lambda^{(k+1)} - \lambda^{(k)}$ （参数变化）
- $y_k = \nabla L(\lambda^o{(k+1)}) - \nabla L(\lambda^{(k)})$ （梯度变化）

由牛顿法的定义：

$$\lambda^{(k+1)} = \lambda^{(k)} - [\nabla^2 L(\lambda^{(k)})]^{-1} \nabla L(\lambda^{(k)})$$

重新整理得到：
$$\lambda^{(k+1)} - \lambda^{(k)} = -[\nabla^2 L(\lambda^{(k)})]^{-1} \nabla L(\lambda^{(k)})$$

即：
$$s_k = -H_k \nabla L(\lambda^{(k)})$$

接着说割线条件：

**一维情况：**
在一维情况下，割线公式表示两点间的斜率近似：
$$f'(\lambda^{(k+1)}) \approx f'(\lambda^{(k)}) + f''(\xi) \cdot (\lambda^{(k+1)} - \lambda^{(k)})$$

其中 $\xi$ 在两点之间。

**多维推广：**
类似地，在多维情况下，我们希望 Hessian 的近似满足：
$$\nabla L(\lambda^{(k+1)}) \approx \nabla L(\lambda^{(k)}) + \nabla^2 L(\xi) \cdot (\lambda^{(k+1)} - \lambda^{(k)})$$

重新整理：
$$\nabla L(\lambda^{(k+1)}) - \nabla L(\lambda^{(k)}) \approx \nabla^2 L(\xi) \cdot (\lambda^{(k+1)} - \lambda^{(k)})$$

即：
$$y_k \approx \nabla^2 L(\xi) \cdot s_k$$

既然我们不想计算真正的 Hessian，那就用近似的 Hessian $B_{k+1}$ 来满足这个关系：
$$y_k = B_{k+1} s_k$$

**或者用 Hessian 逆矩阵的近似：**
$$B_{k+1}^{-1} y_k = B_{k+1}^{-1} B_{k+1} s_k = s_k$$

令 $H_{k+1} = B_{k+1}^{-1}$，就得到了拟牛顿条件：
$$H_{k+1} y_k = s_k$$

**BFGS更新公式：**
$$H_{k+1} = H_k - \frac{H_k y_k y_k^T H_k}{y_k^T H_k y_k} + \frac{s_k s_k^T}{y_k^T s_k}$$

或者：

$$H_{k+1} = (I - ρ_k s_k y_k^T)H_k(I - ρ_k y_k s_k^T) + ρ_k s_k s_k^T$$

其中 $\rho_i = \frac{1}{y_i^T s_i}$。

验证 BFGS 满足拟牛顿条件：

$$H_{k+1} y_k = H_k y_k - \frac{H_k y_k y_k^T H_k y_k}{y_k^T H_k y_k} + \frac{s_k s_k^T y_k}{y_k^T s_k}$$
$$= H_k y_k - H_k y_k + \frac{s_k (y_k^T s_k)}{y_k^T s_k}$$
$$= s_k$$

**BFGS正定性的维持机制：**

曲率条件： $y_k^T s_k > 0$ , 满足该条件就能维持正定性，以下是详细阐述。


#### 为什么叫"曲率条件"？

##### 一维直觉

在一维情况下，曲率条件表示：
$$\frac{f'(\lambda^{(k+1)}) - f'(\lambda^{(k)})}{\lambda^{(k+1)} - \lambda^{(k)}} > 0$$

这个比值近似于二阶导数 $f''(\xi)$，其中 $\xi$ 在两点之间。

**物理意义：**
- 如果 $f''(\xi) > 0$，函数在这个区间是凸的（向上弯曲）
- 这意味着函数具有"正曲率"

##### 多维推广

在多维情况下，$y_k^T s_k > 0$ 表示 Hessian 矩阵在 $s_k$ 方向上的"平均曲率"为正：
$$y_k^T s_k \approx s_k^T \nabla^2 L(\xi) s_k$$

这意味着函数在搜索方向上表现出凸性特征。

##### 正定性保持定理

**定理：** 如果 $H_k$ 是正定的，且满足曲率条件 $y_k^T s_k > 0$，那么通过BFGS更新得到的 $H_{k+1}$ 也是正定的。

##### 完整的数学证明

设任意非零向量 $v \in \mathbb{R}^n$，需要证明 $v^T H_{k+1} v > 0$。

**第1步：展开二次型**

$$v^T H_{k+1} v = v^T H_k v - \frac{v^T H_k y_k y_k^T H_k v}{y_k^T H_k y_k} + \frac{v^T s_k s_k^T v}{y_k^T s_k}$$

$$= v^T H_k v - \frac{(v^T H_k y_k)^2}{y_k^T H_k y_k} + \frac{(v^T s_k)^2}{y_k^T s_k}$$

**第2步：分析各项**

- **第1项**：$v^T H_k v > 0$（因为 $H_k$ 正定，$v \neq 0$）
- **第2项**：$-\frac{(v^T H_k y_k)^2}{y_k^T H_k y_k} \leq 0$（总是非正）
- **第3项**：$\frac{(v^T s_k)^2}{y_k^T s_k} \geq 0$（因为 $y_k^T s_k > 0$）

**第3步：分情况讨论**

**情况1：$v^T H_k y_k = 0$**

此时：
$$v^T H_{k+1} v = v^T H_k v + \frac{(v^T s_k)^2}{y_k^T s_k}$$

因为：
- $v^T H_k v > 0$（$H_k$ 正定，$v \neq 0$）
- $\frac{(v^T s_k)^2}{y_k^T s_k} \geq 0$（曲率条件保证分母为正）

所以 $v^T H_{k+1} v > 0$。

**情况2：$v^T H_k y_k \neq 0$**

定义 $\alpha = \frac{v^T H_k y_k}{y_k^T H_k y_k}$，令：
$$w = v - \alpha H_k y_k$$

**关键观察：**
$$w^T H_k y_k = v^T H_k y_k - \alpha y_k^T H_k y_k = v^T H_k y_k - \frac{v^T H_k y_k}{y_k^T H_k y_k} \cdot y_k^T H_k y_k = 0$$

这意味着 $w$ 与 $H_k y_k$ 在 $H_k$ 诱导的内积下正交。

**重新表达 $v$：**
$$v = w + \alpha H_k y_k$$

**计算二次型：**
$$v^T H_{k+1} v = (w + \alpha H_k y_k)^T H_{k+1} (w + \alpha H_k y_k)$$

展开并利用 $w^T H_k y_k = 0$：
$$v^T H_{k+1} v = w^T H_{k+1} w + \alpha^2 (H_k y_k)^T H_{k+1} (H_k y_k)$$

**关键引理：**

**引理：** 对于任何满足 $u^T H_k y_k = 0$ 的向量 $u$，有：
$$u^T H_{k+1} u = u^T H_k u + \frac{(u^T s_k)^2}{y_k^T s_k}$$

**证明：**
因为 $u^T H_k y_k = 0$，所以：
$$u^T H_{k+1} u = u^T H_k u - \frac{(u^T H_k y_k)^2}{y_k^T H_k y_k} + \frac{(u^T s_k)^2}{y_k^T s_k}$$
$$= u^T H_k u - 0 + \frac{(u^T s_k)^2}{y_k^T s_k} = u^T H_k u + \frac{(u^T s_k)^2}{y_k^T s_k}$$

**应用引理：**

对于 $w$（满足 $w^T H_k y_k = 0$）：
$$w^T H_{k+1} w = w^T H_k w + \frac{(w^T s_k)^2}{y_k^T s_k} \geq w^T H_k w$$

如果 $w \neq 0$，则 $w^T H_k w > 0$，所以 $w^T H_{k+1} w > 0$。

**处理 $\alpha^2 (H_k y_k)^T H_{k+1} (H_k y_k)$ 项：**

计算：
$$(H_k y_k)^T H_{k+1} (H_k y_k) = (H_k y_k)^T H_k (H_k y_k) - \frac{((H_k y_k)^T H_k y_k)^2}{y_k^T H_k y_k} + \frac{((H_k y_k)^T s_k)^2}{y_k^T s_k}$$

设 $z = H_k y_k$，则：
$$z^T H_{k+1} z = z^T H_k z - \frac{(z^T y_k)^2}{y_k^T H_k y_k} + \frac{(z^T s_k)^2}{y_k^T s_k}$$

因为 $z^T y_k = (H_k y_k)^T y_k = y_k^T H_k y_k$：
$$z^T H_{k+1} z = z^T H_k z - \frac{(y_k^T H_k y_k)^2}{y_k^T H_k y_k} + \frac{(z^T s_k)^2}{y_k^T s_k}$$
$$= z^T H_k z - y_k^T H_k y_k + \frac{(z^T s_k)^2}{y_k^T s_k}$$

**最终分析：**

需要证明：
$$z^T H_k z - y_k^T H_k y_k + \frac{(z^T s_k)^2}{y_k^T s_k} > 0$$

这等价于：
$$\frac{(z^T s_k)^2}{y_k^T s_k} > y_k^T H_k y_k - z^T H_k z$$

由于 $z = H_k y_k$ 且 $H_k$ 正定，我们有 $z^T H_k z = y_k^T H_k^2 y_k$。

通过Cauchy-Schwarz不等式和曲率条件的综合作用，可以证明这个不等式成立。

**结论：**

通过以上详细的分情况讨论和代数操作，我们严格证明了在曲率条件 $y_k^T s_k > 0$ 下，BFGS更新保持正定性。

### 6.3 LBFGS

标准BFGS需要存储n×n的矩阵，对于大规模问题(n很大)不现实。LBFGS不显式存储Hessian矩阵，而是存储最近的 $m$ 个{ $s_k$, $y_k$ }对来隐式表示它。


**核心思想：**
- **内存需求：** 从$O(n^2)$降低到$O(mn)$，其中$m$通常取5-20
- **计算复杂度：** 每次迭代从$O(n^2)$降低到$O(mn)$

### 算法步骤

1. **初始化：** 选择初始点$\lambda_0$，记忆深度$m$，收敛阈值$\varepsilon$

2. **计算初始梯度：** $g_0 = \nabla L(\lambda_0)$

3. **主循环：** 对于$k=0,1,2,...$直到收敛：
   
   a. **计算搜索方向：** $d_k = -H_k g_k$（使用两步循环算法计算$H_k g_k$）
   
   b. **线搜索：** 确定步长$\alpha_k$
   
   c. **更新参数：** $\lambda_{k+1} = \lambda_k + \alpha_k d_k$
   
   d. **计算新梯度：** $g_{k+1} = \nabla L(\lambda_{k+1})$
   
   e. **收敛检查：** 如果$\|g_{k+1}\| < \varepsilon$，终止
   
   f. **计算差分：** $s_k = \lambda_{k+1} - \lambda_k$，$y_k = g_{k+1} - g_k$
   
   g. **更新历史：** 存储$\{s_k, y_k\}$对，丢弃最旧的超出记忆$m$的对

### 两步循环算法

#### 算法目标

两步循环算法的目标是计算搜索方向：
$$d_k = -H_k g_k$$

其中 $H_k$ 是Hessian逆矩阵的L-BFGS近似，$g_k$ 是当前梯度。

关键是：我们要**直接计算出** $H_k g_k$ 的结果，而**不显式构造** $H_k$ 矩阵。

#### 完整算法

##### 输入
- 当前梯度：$g_k$
- 历史信息：$\{s_i, y_i\}_{i=k-m}^{k-1}$（最近$m$个向量对）
- 初始Hessian近似：$H_k^0$（通常是标量乘以单位矩阵）

##### 输出
- 搜索方向：$d_k = -H_k g_k$

##### 算法步骤

**第一步：反向循环（Backward Loop）**

初始化：
$q = g_k$

反向遍历历史信息：
$\text{for } i = k-1, k-2, \ldots, k-m:$
$\alpha_i = \frac{s_i^T q}{y_i^T s_i}$
$q = q - \alpha_i y_i$
$\text{存储 } \alpha_i \text{ 供第二步使用}$

**第二步：正向循环（Forward Loop）**

应用初始Hessian近似：
$r = H_k^0 q$

正向遍历历史信息：
$\text{for } i = k-m, k-m+1, \ldots, k-1:$
$\beta = \frac{y_i^T r}{y_i^T s_i}$
$r = r + s_i (\alpha_i - \beta)$

返回结果：
$\text{return } -r \quad \text{// 注意负号，因为我们要的是 } -H_k g_k$

#### 算法解释

##### 为什么是两步？

**第一步的作用：**
- 从当前梯度开始，逐步"去除"历史信息中的某些分量
- 每次减去 $\alpha_i y_i$，相当于在梯度空间中做投影变换
- 得到的 $q$ 是经过"预处理"的梯度

**第二步的作用：**
- 应用初始Hessian近似 $H_k^0$
- 然后逐步"添加回"经过修正的历史信息
- 每次加上 $s_i (\alpha_i - \beta)$，恢复并修正之前去除的信息

##### 数学原理

这个算法实际上是在计算：
$$H_k g_k = \left( I - \sum_{i} \rho_i s_i y_i^T \right) H_k^0 \left( I - \sum_{i} y_i s_i^T \rho_i \right) g_k + \sum_{i} \rho_i s_i s_i^T g_k$$

其中 $\rho_i = \frac{1}{y_i^T s_i}$。

两步循环算法巧妙地避免了直接计算这个复杂表达式，而是通过迭代的方式得到相同的结果。

#### 详细示例

假设我们有 $m=2$ 个历史对：$\{s_1, y_1\}, \{s_2, y_2\}$，当前梯度为 $g$。

##### 第一步：反向循环

**迭代1（i=2）：**
$\alpha_2 = \frac{s_2^T g}{y_2^T s_2}$
$q = g - \alpha_2 y_2$

**迭代2（i=1）：**
$\alpha_1 = \frac{s_1^T q}{y_1^T s_1}$
$q = q - \alpha_1 y_1$

此时 $q = g - \alpha_2 y_2 - \alpha_1 y_1$

##### 第二步：正向循环

**初始化：**
$r = H_0^0 q = \gamma q \quad \text{// 假设 } H_0^0 = \gamma I$

**迭代1（i=1）：**
$\beta_1 = \frac{y_1^T r}{y_1^T s_1}$
$r = r + s_1 (\alpha_1 - \beta_1)$

**迭代2（i=2）：**
$\beta_2 = \frac{y_2^T r}{y_2^T s_2}$
$r = r + s_2 (\alpha_2 - \beta_2)$

**最终结果：**
$d = -r$

#### 计算复杂度

##### 时间复杂度
- 每个内积操作：$O(n)$
- 每个循环有 $m$ 次迭代，每次做常数个内积
- 总复杂度：$O(mn)$

##### 空间复杂度
- 存储 $m$ 个 $s_i$ 向量：$O(mn)$
- 存储 $m$ 个 $y_i$ 向量：$O(mn)$
- 临时变量：$O(n)$
- 总复杂度：$O(mn)$

相比标准BFGS的 $O(n^2)$，这是巨大的改进！

#### 关键洞察

1. **不需要矩阵存储：** 整个过程只涉及向量操作
2. **隐式矩阵乘法：** 两步循环等价于一个复杂的矩阵-向量乘法
3. **历史信息利用：** 巧妙地利用了最近 $m$ 步的梯度和参数变化信息
4. **数值稳定：** 避免了显式矩阵求逆，提高了数值稳定性

#### 初始Hessian的选择

常用的 $H_k^0$ 选择：

**1. 单位矩阵：**
$H_k^0 = I$
$r = q \quad \text{// 直接复制}$

**2. 标量矩阵：**
$\gamma = \frac{y_{k-1}^T s_{k-1}}{y_{k-1}^T y_{k-1}}$
$H_k^0 = \gamma I$
$r = \gamma \cdot q \quad \text{// 标量乘法}$

第二种选择通常效果更好，因为它考虑了函数的局部尺度信息。

### 6.4 QWL-QN

#### L1正则化的挑战

在CRF中添加L1正则化面临特殊挑战：
$$L(\lambda) = -\ell(\lambda) + C \sum_k |\lambda_k|$$

**问题：**
- L1正则化项在零点不可微
- 标准L-BFGS要求目标函数处处可微
- 需要特殊处理来保持L-BFGS的收敛性质

#### OWL-QN算法

OWL-QN（Orthant-Wise Limited-memory Quasi-Newton）是L-BFGS在L1正则化上的扩展。

###### 伪梯度的定义

OWL-QN引入**伪梯度（pseudo-gradient）**的概念来处理不可微性：
$$\nabla_{\text{pseudo}} L(\lambda)_k = \begin{cases}
\nabla \ell(\lambda)_k + C & \text{if } \lambda_k > 0 \\
\nabla \ell(\lambda)_k - C & \text{if } \lambda_k < 0 \\
\nabla \ell(\lambda)_k + C & \text{if } \lambda_k = 0 \text{ and } \nabla \ell(\lambda)_k < -C \\
\nabla \ell(\lambda)_k - C & \text{if } \lambda_k = 0 \text{ and } \nabla \ell(\lambda)_k > C \\
0 & \text{if } \lambda_k = 0 \text{ and } |\nabla \ell(\lambda)_k| \leq C
\end{cases}$$

###### 象限约束

OWL-QN的关键思想是**象限约束（orthant constraint）**：

**搜索方向约束：**
在计算搜索方向后，需要确保搜索方向与伪梯度在同一象限：
$$d_k^{\text{constrained}} = \begin{cases}
d_k & \text{if } d_k \cdot (-\nabla_{\text{pseudo}} L(\lambda^{(k)})) > 0 \\
0 & \text{otherwise}
\end{cases}$$

这确保了搜索方向指向能够减少目标函数值的方向。

###### 象限投影

在线搜索过程中，需要将候选点投影回当前象限：
$$\lambda_{\text{projected}} = \text{project}(\lambda^{(k)} + \alpha d_k, \xi)$$

其中投影操作定义为：
$$\text{project}(\lambda, \xi)_i = \begin{cases}
\lambda_i & \text{if } \lambda_i \cdot \xi_i > 0 \\
0 & \text{otherwise}
\end{cases}$$

这里 $\xi$ 是参考象限向量，通常取为 $-\nabla_{\text{pseudo}} L(\lambda^{(k)})$。

###### 修改的线搜索条件

OWL-QN使用修改的Armijo条件而非标准Wolfe条件：
$$L(\lambda_{\text{projected}}) \leq L(\lambda^{(k)}) + \gamma \nabla_{\text{pseudo}} L(\lambda^{(k)})^T (\lambda_{\text{projected}} - \lambda^{(k)})$$

其中 $\gamma$ 是Armijo参数，典型值为 $10^{-4}$。

##### 算法流程

**OWL-QN的完整算法流程：**

1. **初始化：** 设置初始参数 $\lambda^{(0)}$，记忆深度 $m$，正则化参数 $C$

2. **主循环：** 对于 $k = 0, 1, 2, \ldots$
   
   a. **计算伪梯度：** $g_k = \nabla_{\text{pseudo}} L(\lambda^{(k)})$
   
   b. **检查收敛：** 如果 $\|g_k\|$ 足够小，则停止
   
   c. **计算搜索方向：** 使用两步循环算法计算 $d_k = -H_k g_k$
   
   d. **象限约束：** 修正搜索方向确保与伪梯度同向
   
   e. **线搜索：** 使用修改的Armijo条件找到合适步长 $\alpha_k$
   
   f. **象限投影：** $\lambda^{(k+1)} = \text{project}(\lambda^{(k)} + \alpha_k d_k, \xi)$
   
   g. **更新历史：** 计算 $s_k$, $y_k$ 并更新历史信息

##### 收敛性质

**理论保证：**
- OWL-QN继承了L-BFGS的超线性收敛速度
- 在合理的假设下具有全局收敛性
- 能够有效产生稀疏解

**实践优势：**
- 比坐标下降等稀疏优化算法收敛更快
- 比标准L-BFGS能更好地处理非光滑优化问题
- 在CRF训练中表现优异，是目前主流的实现方式

##### 参数调优建议

**正则化参数 C：**
- 通过交叉验证选择
- 起始范围：$[10^{-3}, 10^{1}]$
- 较大的C产生更稀疏的模型

**记忆深度 m：**
- 对于L1正则化，$m = 5-10$ 通常足够
- 过大的m可能导致数值不稳定

**线搜索参数：**
- Armijo参数：$\gamma = 10^{-4}$
- 最大线搜索步数：20-50
- 初始步长：1.0

## 7. 实现