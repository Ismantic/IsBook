# Optimizer 参数更新

## 引言

反向传播只负责计算参数梯度，并不会自动修改参数。训练框架还需要决定怎样使用梯度，以及每次更新采用多大的步长，这些工作由优化器完成。

优化器不需要理解模型的层级结构，只需遍历 `Module::Parameters()` 返回的扁平参数列表。除了执行具体的更新公式，它还要清理上一轮梯度，并为 AdamW 这类算法保存跨越多个训练步骤的状态。参数更新本身不能进入自动微分图，否则不同训练步骤的计算图会被连接起来。

完整训练流程可以概括为：

```text
清空梯度 → 前向传播 → 计算损失 → 反向传播 → 更新参数
```

## 实现

参数更新并不只是把某个优化公式翻译成代码。优化器需要为参数保存跨越多个训练步骤的状态，训练循环还可能根据当前进度调整学习率。二者共同决定参数在这一轮发生多大变化。

Zero 用 `Optimizer` 统一管理参数和梯度清理，并使用 `AdamW` 完成参数更新。`AdamW` 为每个参数保存一阶、二阶动量，其计算由张量算子实现，因此同一份代码可以运行在 CPU 和 CUDA 上。`SetLr` 则让训练循环能够在每一步改变学习率。

下面先从统一接口开始，再说明 AdamW 的更新过程和学习率变化。

**Optimizer**

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

**AdamW**

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

**学习率调度**

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

## 示例

MNIST 可以用一个很小的模型验证参数更新是否真正有效：如果前向计算、反向传播或优化器中的任意一环出错，损失就无法稳定下降。这个示例不依赖复杂网络，能够把注意力集中在一轮训练如何完成。

**数据**

MNIST 的每张图像包含 \\(28\times28\\) 个灰度像素。数据加载器读取 IDX 二进制文件，将其中的大端整数转换为主机字节序，再把像素从 `[0, 255]` 归一化到 `[0, 1]`。一个 Batch 最终形成形状为 `[B, 784]` 的 `F32` 张量，标签则使用形状为 `[B]` 的 `I32` 张量。

```text
图像：[B, 28, 28] → 展平 → [B, 784]
标签：[B]
```

`F32` 图像参与矩阵计算，`I32` 标签用于在交叉熵中索引正确类别。两者分开表示，也说明张量的数据类型是算子语义的一部分。

**模型**

示例使用一个单隐藏层 MLP：

```text
[B, 784] → Linear(784, 512) → ReLU
         → Linear(512, 10) → Logits [B, 10]
```

它直接复用上一篇的 `Module` 和 `Linear`：

```cpp
class SimpleMLP : public Module {
public:
    SimpleMLP(int64_t input_size,
              int64_t hidden_size,
              int64_t output_size) {
        fc1_ = std::make_shared<Linear>(input_size, hidden_size);
        fc2_ = std::make_shared<Linear>(hidden_size, output_size);
        RegisterModule("fc1", fc1_);
        RegisterModule("fc2", fc2_);
    }

    Tensor OpValue(const Tensor& input) {
        return fc2_->OpValue(ReLU(fc1_->OpValue(input)));
    }

private:
    std::shared_ptr<Linear> fc1_;
    std::shared_ptr<Linear> fc2_;
};
```

`model->Parameters()` 会递归收集两个线性层的权重和 Bias，并将同一份参数列表交给设备迁移与优化器：

```cpp
auto all_params = model->Parameters();
```

**训练循环**

训练循环的核心代码只有几步：

```cpp
optimizer->ZeroGrad();

auto logits = model->OpValue(images);
auto loss = CrossEntropyLoss(logits, targets);

loss.Backward();
optimizer->Step();
```

`ZeroGrad()` 清除上一个 Batch 的梯度，前向过程将 `[B, 784]` 的输入转换成 `[B, 10]` 的 Logits，`CrossEntropyLoss` 根据标签产生损失。`Backward()` 将梯度传回四个参数，`Step()` 再根据当前优化算法更新它们。下一个 Batch 会使用更新后的参数重新开始这条链路。

**设备选择**

CPU 和 GPU 使用相同的 `AdamW`。如果选择 CUDA，需要先将模型参数迁移到 GPU，再创建优化器状态；每个 Batch 的输入也要迁移到同一设备：

```cpp
if (use_cuda) {
    for (auto& [name, parameter] : all_params) {
        parameter->Cuda();
    }
}

auto optimizer = std::make_unique<AdamW>(0.001f);
optimizer->InsertParameters(all_params);

if (use_cuda) images.Cuda();
```

`AdamW::InsertParameters()` 会为每个参数创建一阶和二阶动量，因此应在参数迁移之后调用，保证优化器状态与参数位于同一设备。除此之外，模型结构、损失函数和训练循环不需要因设备不同而改变。

训练过程中可以记录每个 Batch 的损失和分类准确率。损失持续下降说明参数更新方向有效，准确率上升则说明这些更新确实改善了模型输出。这个示例的价值不在于模型规模，而在于它打通了数据、模型、损失、自动微分和优化器之间的完整链路。

这条训练链路不依赖具体设备，下一篇将说明相同算子如何派发到 CUDA，并在 GPU 上高效执行。
