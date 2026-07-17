# 番外篇：词向量W2V

## 模型定义

W2V是一种高效的词向量学习模型，它能够把词汇映射到低维稠密向量空间中，使得语义相似的词能在向量空间中距离较近。W2V包含两种模型架构：Skip-Gram 和 CBOW，以及负采样和分层 Softmax 两种常用的训练方法。本文专注于 CBOW 架构结合基于霍夫曼树的分层 Softmax 实现。

**CBOW**

CBOW模型的核心思想是：**通过上下文词汇来预测中心词**。给定一个词序列，CBOW模型将目标词周围的上下文词作为输入，预测中间的目标词。

具体来说，对于句子中的词 \\(w_t\\) ，我们使用其前后各 \\(c\\) 个词 \\(\{w_{t-c}, ..., w_{t-1}, w_{t+1}, ..., w_{t+c}\}\\) 作为上下文，来预测 \\(w_t\\) 。

**Softmax**

标准的神经网络语言模型中，输出层通常使用 Softmax 函数来计算词汇表中每个词的概率：

$$
P(w_o|context) =
    \frac{\exp(v_{w_o}^T \cdot v_c)}
         {\sum_{w=1}^{V} \exp(v_w^T \cdot v_c)}
$$

其中：
- \\(v_{w_o}\\) 是输出词 \\(w_o\\) 的向量表示
- \\(v_c\\) 是跟context相关的隐藏层的输出
- \\(V\\) 是词汇表大小

这种方法的计算**时间复杂度为 \\(O(V)\\)**，当词汇表包含数十万甚至数百万个词时，计算成本变得极其昂贵。

**Context**

CBOW模型中，给定上下文词集合 \\(C = \{w_{c_1}, w_{c_2}, ..., w_{c_t}\}\\) ，隐藏层输出通过平均上下文词向量得到：

$$
v_c = \frac{1}{|C|} \sum_{c \in C} v_{w_c}
$$

其中 \\(v_{w_c}\\) 是上下文词 \\(w_c\\) 的输入向量表示。


## 霍夫曼树

W2V引入了**分层Softmax** (**Hierarchical Softmax**) 技术，通过霍夫曼树把计算复杂度由 \\(O(V)\\) 降低到 \\(O(\log V)\\)。

**霍夫曼树的定义**

**霍夫曼树是一种二叉树，具有以下性质**
- 叶子节点代表词汇表中的词，全部叶子节点对应词汇表中的全部词
- 根节点到叶子节点的固定路径能一一对应到具体的词
- 词的概率计算能转化为沿着这个路径的决策序列
- 内部节点对应着进行一次二分类决策

**霍夫曼树的Softmax计算**

词 \\(w\\) 对应一个叶子节点。设从根节点到该叶子节点的路径依次为
\\(n_1,n_2,\ldots,n_{L(w)}\\)，其中 \\(L(w)\\) 表示路径包含的节点数。因此，这条路径包含 \\(L(w)-1\\) 条边，也就是 \\(L(w)-1\\) 次二分类决策。除最后的叶子节点 \\(n_{L(w)}\\) 外，每个内部节点 \\(n_j\\) 都有一个参数向量 \\(\theta_{n_j}\\)。

**概率计算公式**
$$
P(w|context)
= \prod_{j=1}^{L(w)-1}
\sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T \cdot v_c)
$$

其中：
- \\(\sigma(x) = \frac{1}{1+\exp(-x)}\\) 是 Sigmoid 函数
- \\(I(n_j,n_{j+1})\\) 是路径指示函数（左分支为1,右分支为-1）
- \\(v_c\\) 是隐藏层输出

注：最后参与计算的是内部节点 \\(n_{L(w)-1}\\)，叶子节点 \\(n_{L(w)}\\) 不包含参数，也不会参与点积计算。

**内部节点的物理意义**

霍夫曼树中的每个内部节点实际上在进行一次**二分类决策**：
- \\(\sigma(\theta_{n_j}^T v_c)\\) 表示选择左分支的概率
- \\(1 - \sigma(\theta_{n_j}^T v_c) = \sigma(-\theta_{n_j}^T v_c)\\) 表示选择右分支的概率

因此，到达目标词 \\(w\\) 的概率就是沿路径进行所有正确决策的概率乘积。


**时间复杂度分析**

**传统Softmax的复杂度**：
- 需要计算所有 \\(V\\) 个词的 \\(\exp(v_w^T \cdot v_c)\\)
- 需要对所有 \\(V\\) 个值求和作为分母
- 总计算量：\\(O(V \cdot m)\\)，其中 \\(m\\) 是向量维度

**霍夫曼树Softmax的复杂度**：
- 只需要沿着一条路径计算 \\(L(w)-1\\) 个sigmoid函数
- 霍夫曼树不保证每个词的路径长度都是 \\(O(\log V)\\)；极端情况下，个别低频词的路径可能很长
- 霍夫曼编码最小化的是按照词频加权的平均路径长度，其平均编码长度接近词频分布的信息熵
- 对常见的词频分布，平均路径长度通常可近似看作 \\(O(\log V)\\)，因此平均计算量通常写作 \\(O(\log V \cdot m)\\)

**霍夫曼树的构建**

1. **初始化**：将词汇表中的每个词作为叶子节点，节点权重设为词频
2. **迭代合并**：重复选择两个权重最小的节点进行合并，新节点权重为两个子节点权重之和
3. **路径编码**：为每条从根节点到叶子节点的路径分配二进制编码
   - 左分支编码为1，右分支编码为-1
   - 高频词自然获得较短的路径，低频词获得较长的路径

这种构建方式会让高频词更晚加入到树中，距离根更近，确保了计算效率的最优化：高频词由于路径短，计算快速；低频词虽然路径长，但由于出现频率低，总体上仍然高效。

以下给出一个示例：


**基本设定**
- **词汇表**：`{A, B, C, D}`
- **词频统计**：`A:8`, `B:6`, `C:3`, `D:1`
- **编码规则**：
  - 左分支 = `1`（正类）
  - 右分支 = `-1`（负类）

**霍夫曼树构建全流程**

**初始化节点**
| 节点 | 词 | 频次 | 类型   |
|------|----|------|--------|
| n1   | A  | 8    | 叶子节点 |
| n2   | B  | 6    | 叶子节点 |
| n3   | C  | 3    | 叶子节点 |
| n4   | D  | 1    | 叶子节点 |


**迭代合并过程**
1. **第一轮合并**  
   合并最低频的 `C(3)` 和 `D(1)` → 创建内部节点 `Node2(4)`  

```
Node2 (频次=4)
/ \
C D
```

2. **第二轮合并**  
合并 `Node2(4)` 和 `B(6)` → 创建内部节点 `Node1(10)`  

```
 Node1 (频次=10)
 /   \
B     Node2
    /   \
   C     D
```


3. **最终合并**  
合并 `Node1(10)` 和 `A(8)` → 创建根节点 `Root(18)`  

```
    Root (频次=18)
   /    \
  A     Node1
       /    \
      B     Node2
           /    \
          C      D
```

**完整路径编码表**
| 词 | 路径                 | 编码序列   | 决策次数 | 计算示例 |
|----|----------------------|------------|----------|----------|
| A  | Root → A             | 左(1)      | 1        | `σ(θ_Root·v_c)` |
| B  | Root → Node1 → B     | 右(-1)→左(1) | 2        | `σ(-θ_Root·v_c) × σ(θ_Node1·v_c)` |
| C  | Root → Node1 → Node2 → C | 右→右→左 | 3        | `σ(-θ_Root·v_c) × σ(-θ_Node1·v_c) × σ(θ_Node2·v_c)` |
| D  | Root → Node1 → Node2 → D | 右→右→右 | 3        | `σ(-θ_Root·v_c) × σ(-θ_Node1·v_c) × σ(-θ_Node2·v_c)` |

**霍夫曼树的证明**

接下来还要证明以下公式，是霍夫曼树Softmax能取代标准Softmax的基本要求：

$$
\sum_{w=1}^V P(w|context)
=
\sum_{w=1}^V \prod_{j=1}^{L(w)-1}
\sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T v_c)
= 1
$$


**基本定义**

**霍夫曼树结构**
- **内部节点**：每个内部节点 \\(n\\) 包含参数向量 \\(\theta_n\\)，用于二元决策
- **叶子节点**：代表词汇表中的单词 \\(w\\)，总数为 \\(V\\)

**概率计算**
对于隐藏层输出 \\(h\\) 和内部节点 \\(n\\)：
- 向左子节点移动概率：

$$
p(\text{left}|n)
= \sigma(\theta_n \cdot v_c)
= \frac{1}{1+e^{-\theta_n \cdot v_c}}
$$

- 向右子节点移动概率：

$$
p(\text{right}|n)
= \sigma(-\theta_n \cdot v_c)
= 1 - \sigma(\theta_n \cdot v_c)
$$

其中 \\(\sigma(x)\\) 为 sigmoid 函数，满足：
$$
\sigma(x) + \sigma(-x) = 1
$$

**归纳法证明**

**基础证明（树高度=1）**
单层霍夫曼树含1个内部节点和2个叶子节点：

$$
\begin{aligned}
p(w_1) &= \sigma(\theta \cdot v_c) \\\\
p(w_2) &= \sigma(-\theta \cdot v_c) \\\\
\sum_{i=1}^2 p(w_i) &= \sigma(\theta \cdot v_c) + \sigma(-\theta \cdot v_c) = 1
\end{aligned}
$$

**归纳假设**
假设对于高度 \\(=k\\) 的霍夫曼树，所有叶子节点概率和为1：

$$
\sum_{w \in \text{Leaves}_k} p(w) = 1
$$

**归纳步骤（高度=k+1）**
考虑根节点及其左右子树：
1. 根节点决策概率：

$$
\begin{cases}
p_L = \sigma(\theta_{root} \cdot v_c) \\\\
p_R = \sigma(-\theta_{root} \cdot v_c)
\end{cases}
$$

2. 根据归纳假设：
   - 左子树叶子概率和 \\(= 1\\) → 贡献 \\(p_L \times 1\\)
   - 右子树叶子概率和 \\(= 1\\) → 贡献 \\(p_R \times 1\\)
3. 整体概率和：

$$
\begin{aligned}
p_L + p_R
&= \sigma(\theta_{root} \cdot v_c) + \sigma(-\theta_{root} \cdot v_c) \\\\
&= 1
\end{aligned}
$$

**递归性质证明**

这里需要区分两种概率：从整棵树的根节点出发计算的全局概率，以及已经到达某个子树根节点 \\(n\\) 后的条件概率。对于以 \\(n\\) 为根的任意子树，都有：

$$
\sum_{w \in \operatorname{Leaves}(n)} P(w\mid n,context)=1
$$

如果 \\(n\\) 是内部节点，设其左右孩子分别为 \\(n_L\\) 和 \\(n_R\\)，那么：

$$
\begin{aligned}
\sum_{w \in \operatorname{Leaves}(n)}P(w\mid n,context)
&=P(n_L\mid n,context)
  \sum_{w\in\operatorname{Leaves}(n_L)}P(w\mid n_L,context) \\\\
&\quad+P(n_R\mid n,context)
  \sum_{w\in\operatorname{Leaves}(n_R)}P(w\mid n_R,context) \\\\
&=P(n_L\mid n,context)+P(n_R\mid n,context) \\\\
&=1
\end{aligned}
$$

从根节点应用这一递归关系，就能得到所有叶子节点的全局概率之和为 1。相应地，如果使用从整棵树根节点出发的全局概率，那么某个子树中所有叶子的概率之和等于到达该子树根节点的概率，而不一定等于 1。

**示例验证**

3个叶子节点的霍夫曼树：
```
   Root
  /    \
 A      w3
/  \
w1 w2
```


概率计算：

$$
\begin{aligned}
p(w_1) &= p(A) \times p(\text{left}|A) \\\\
p(w_2) &= p(A) \times p(\text{right}|A) \\\\
p(w_3) &= p(\text{right}|Root) \\\\
\sum_{i=1}^3 p(w_i) &= p(A)[p(\text{left}|A)+p(\text{right}|A)] + p(w_3) \\\\
&= p(A)\times 1 + p(w_3) \\\\
&= \sigma(\theta_{Root}\cdot v_c) + \sigma(-\theta_{Root}\cdot v_c) \\\\
&= 1
\end{aligned}
$$

**关键结论**

通过以下性质保证归一化：
1. **局部归一化**： \\(\forall n,\ p(\text{left}|n)+p(\text{right}|n)=1\\)
2. **递归累乘**： \\(p(w) = \prod_{\text{path to }w} p(\text{branch})\\)
3. **树结构完备性**：每个样本必被分配到唯一叶子节点

因此分层 Softmax 满足：

$$
\sum_{w=1}^V p(w|v_c) = 1
$$

## 目标函数


对于训练样本 \\((context, w)\\)，我们希望最大化条件概率 \\(P(w|context)\\)。采用最大似然估计，目标函数为：

$$
\mathcal{L}
= \log P(w|context)
= \log \prod_{j=1}^{L(w)-1}
\sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T v_c)
$$

$$
\mathcal{L}
= \sum_{j=1}^{L(w)-1}
\log \sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T v_c)
$$


对于整个训练语料库，总的目标函数为：

$$
\mathcal{L}&#95;{\mathrm{total}}
= \sum_{(context, w) \in D}
\sum_{j=1}^{L(w)-1}
\log \sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T v_{\mathrm{context}})
$$

其中 \\(D\\) 是训练数据集， \\(v_{context}\\) 是对应上下文的隐藏层输出。

## 梯度推导

CBOW + 霍夫曼树模型中，需要推导的参数包括：
- **输入词向量** \\(v_{w_c}\\) ：每个词作为上下文时的向量表示
- **霍夫曼树节点向量** \\(\theta_{n_j}\\) : 霍夫曼树内部节点的参数向量

**对霍夫曼树节点参数的梯度**

对于路径上的节点 \\(n_j\\)，我们需要计算目标函数对 \\(\theta_{n_j}\\) 的梯度。

**单个样本的梯度计算**：

$$
\frac{\partial \mathcal{L}}{\partial \theta_{n_j}}
= \frac{\partial}{\partial \theta_{n_j}}
\log \sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T \cdot v_c)
$$

由Sigmoid函数的导数性质 \\(\frac{d}{dx}\log \sigma(x) = 1 - \sigma(x)\\)：

$$
\frac{\partial}{\partial \theta_{n_j}}
\log \sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T \cdot v_c)
= [1 - \sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T v_c)]
\cdot I(n_j, n_{j+1}) \cdot v_c
$$

**梯度的物理意义**：
- 当 \\(\sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T \cdot v_c) \approx 1\\) 时（预测正确），梯度较小
- 当 \\(\sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T \cdot v_c) \approx 0\\) 时（预测错误），梯度较大

**对隐藏层输出的梯度**

隐藏层输出 \\(v_c\\) 连接到路径上的所有节点，因此其梯度是所有节点梯度的累加：

$$
\frac{\partial \mathcal{L}}{\partial v_c}
= \sum_{j=1}^{L(w)-1}
\frac{\partial}{\partial v_c}
\log \sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T \cdot v_c)
$$

$$
\frac{\partial \mathcal{L}}{\partial v_c}
= \sum_{j=1}^{L(w)-1}
[1 - \sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T \cdot v_c)]
\cdot I(n_j, n_{j+1}) \cdot \theta_{n_j}
$$


**对输入词向量的梯度**

由于隐藏层输出是上下文词向量的平均：

$$
v_c = \frac{1}{|C|} \sum_{c \in C} v_{w_c}
$$

因此：

$$
\frac{\partial v_c}{\partial v_{w_c}} = \frac{1}{|C|}
$$

由链式法则，对每个上下文词向量的梯度为：

$$
\frac{\partial \mathcal{L}}{\partial v_{w_c}}
= \frac{\partial \mathcal{L}}{\partial v_c}
\cdot \frac{\partial v_c}{\partial v_{w_c}}
= \frac{1}{|C|} \cdot \frac{\partial \mathcal{L}}{\partial v_c}
$$

**完整的梯度表达式**

综合以上推导，完整的梯度表达式为：

**霍夫曼树节点参数梯度**：
$$
\frac{\partial \mathcal{L}}{\partial \theta_{n_j}}
= [1 - \sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T v_c)]
\cdot I(n_j, n_{j+1}) \cdot v_c
$$

**上下文词向量梯度**：
$$
\frac{\partial \mathcal{L}}{\partial v_{w_c}}
= \frac{1}{|C|}
\sum_{j=1}^{L(w)-1}
[1 - \sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T \cdot v_c)]
\cdot I(n_j, n_{j+1}) \cdot \theta_{n_j}
$$


**参数更新**

使用梯度上升法（因为我们要最大化似然函数）更新参数：

**霍夫曼树节点参数更新**：
$$
\theta_{n_j}
\leftarrow \theta_{n_j}
{}+ \alpha
\cdot [1 - \sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T v_c)]
\cdot I(n_j, n_{j+1}) \cdot v_c
$$

**输入词向量更新**：
$$
v_{w_c}
\leftarrow v_{w_c}
{}+ \frac{\alpha}{|C|}
\sum_{j=1}^{L(w)-1}
[1 - \sigma(I(n_j, n_{j+1}) \cdot \theta_{n_j}^T v_c)]
\cdot I(n_j, n_{j+1}) \cdot \theta_{n_j}
$$

其中 \\(\alpha\\) 是学习率。 

实际实现时，必须先使用更新前的内部节点向量 \\(\theta_{n_j}\\) 累加
\\(\partial \mathcal{L}/\partial v_c\\)，再更新上下文词向量。内部节点参数也应当根据同一次前向计算得到的梯度更新。如果一边修改 \\(\theta_{n_j}\\)，一边使用修改后的值继续计算输入词向量梯度，代码执行的就不再是上面推导出的同一次梯度更新。常见实现会先把隐藏层梯度完整累加到一个临时向量中，然后统一更新所有上下文词向量。
