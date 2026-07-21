# 优化器

反向传播得到参数梯度后，优化器负责根据梯度修改参数。它不需要理解模型的层级结构，只需遍历 `Module::Parameters()` 返回的扁平参数列表。

完整训练流程可以概括为：

```text
清空梯度 → 前向传播 → 计算损失 → 反向传播 → 裁剪梯度 → 更新参数
```

## 优化器接口

`Optimizer` 基类持有扁平参数列表，并提供统一的梯度清零接口：

```cpp
class Optimizer {
public:
    void InsertParameters(
        const std::vector<Module::NameParameter>& parameters);

    virtual void ZeroGrad();
    virtual void Step() = 0;
};
```

`ZeroGrad()` 遍历参数并清空上一轮的梯度，`Step()` 由具体优化算法实现。因为反向传播会累加梯度，每轮训练开始前必须先调用 `ZeroGrad()`。

## ClippedSGD

MNIST 的 CPU 示例实现了带逐元素裁剪的 SGD。它先将每个梯度限制在 `[-0.1, 0.1]`，再执行：

$$
\theta\leftarrow\theta-\eta g
$$

该实现直接读写 `grad.Data<float>()`，因此只适用于 CPU。GPU 上的 `Data()` 返回设备指针，不能由主机代码逐元素访问。

## AdamW

AdamW 为每个参数维护一阶动量 `m` 和二阶动量 `v`：

$$
\begin{aligned}
m_t &= \beta_1m_{t-1}+(1-\beta_1)g_t \\\\
v_t &= \beta_2v_{t-1}+(1-\beta_2)g_t^2
\end{aligned}
$$

使用偏差修正后的更新量为：

$$
\begin{aligned}
\alpha_t
&= \eta\frac{\sqrt{1-\beta_2^t}}{1-\beta_1^t} \\\\
\Delta\theta
&= \alpha_t\frac{m_t}{\sqrt{v_t+\epsilon}}
   +\eta\lambda\theta \\\\
\theta
&\leftarrow\theta-\Delta\theta
\end{aligned}
$$

权重衰减项 `\eta\lambda\theta` 与梯度动量分开，这是 AdamW 与将 L2 正则项直接加入梯度的 Adam 之间的关键区别。

AdamW 的更新完全由张量算子构成，所以同一份代码可以运行在 CPU 和 CUDA 上。所有更新算子都使用 `RunOpNoGrad`，不记录到模型的自动微分图中。

## 全局梯度裁剪

逐元素裁剪会分别改变过大的梯度分量，而大模型训练更常用全局 L2 范数裁剪。对所有参数梯度，先计算：

$$
\begin{aligned}
\lVert g\rVert_2
&= \sqrt{\sum_p\sum_i g_{p,i}^2}
\end{aligned}
$$

如果该范数超过阈值 \\(c\\)，则统一缩放所有梯度：

$$
g_p\leftarrow
g_p\cdot\frac{c}{\lVert g\rVert_2+10^{-6}}
$$

统一缩放能保留梯度向量的方向。`ClipGradNorm` 在 CPU 上直接累加平方和；在 GPU 上则使用一个设备端累加器统计所有参数，最后只将一个标量复制回 CPU。

## 学习率调度

`AdamW::SetLr` 允许训练循环在每步更新学习率。GPT 训练使用线性 warmup 与余弦衰减：

$$
\eta_t=
\begin{cases}
\eta_{\max}\dfrac{t+1}{T_w}, & t<T_w \\\\
\eta_{\min}+\dfrac{1}{2}(\eta_{\max}-\eta_{\min})
\left(1+\cos(\pi r_t)\right), & t\ge T_w
\end{cases}
$$

其中 \\(T_w\\) 是 warmup 步数，\\(r_t\\) 是衰减阶段当前进度。warmup 避免训练初期在动量统计尚不稳定时使用过大步长，余弦衰减则让后期更新逐渐变小。

## MNIST 训练

MNIST 示例将 \\(28\times28\\) 图像展平为 \\(784\\) 维向量，再使用一个单隐藏层 MLP 分类：

```text
[batch, 784] → Linear(784, 512) → ReLU
             → Linear(512, 10) → logits
```

数据加载器解析 MNIST 的 IDX 二进制格式，处理大端字段，并将像素从 `[0, 255]` 归一化到 `[0, 1]`。标签使用 `I32` 张量，损失由融合的 `CrossEntropyLoss` 计算。

训练循环的核心代码只有几步：

```cpp
optimizer->ZeroGrad();

auto logits = model->OpValue(images);
auto loss = CrossEntropyLoss(logits, targets);

loss.Backward();
optimizer->Step();
```

CPU 路径默认使用 `ClippedSGD`，GPU 路径使用设备无关的 AdamW。这个例子的价值不在于模型规模，而在于它打通了数据、模型、损失、自动微分和优化器之间的完整链路。

这条训练链路不依赖具体设备，下一篇将说明相同算子如何派发到 CUDA，并在 GPU 上高效执行。
