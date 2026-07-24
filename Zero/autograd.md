# Autograd 自动微分

## 引言

深度学习训练的目标，是找到一组使损失尽可能小的模型参数。梯度给出了损失相对于各个参数的变化率，优化器据此更新参数。为了得到这些梯度，训练框架必须解决两个问题：如何记录前向计算，以及如何沿着计算过程反向传播梯度。

Zero 使用反向模式自动微分。它在前向计算时动态构建计算图；调用 `Backward()` 后，从损失的梯度开始，按照与前向相反的顺序依次执行各个算子的反向规则。每个算子接收输出端传来的梯度，计算各个输入的梯度并继续向前传递；如果同一个张量参与了多条计算路径，来自不同路径的梯度还需要累加。这个过程一直进行到模型参数，得到参数更新所需的梯度。

**链式法则**

考虑复合函数：

$$
y=f(g(x))
$$

它对 \\(x\\) 的导数可以拆成两个局部导数的乘积：

$$
\begin{aligned}
\frac{\partial y}{\partial x}
&= \frac{\partial y}{\partial g}
   \frac{\partial g}{\partial x}
\end{aligned}
$$

训练引擎不需要一次求出整个模型的导函数。每个算子只需知道自己的局部导数，然后将上游梯度乘上局部导数即可。

例如 \\(z=x\cdot y\\) 的上游梯度为 \\(g\\)，乘法算子返回：

$$
\frac{\partial L}{\partial x}=g\cdot y,
\qquad
\frac{\partial L}{\partial y}=g\cdot x
$$

如果一个张量被多个后续算子使用，那么它的总梯度是每条路径贡献之和：

$$
\begin{aligned}
\frac{\partial L}{\partial x}
&= \sum_i
   \frac{\partial L}{\partial y_i}
   \frac{\partial y_i}{\partial x}
\end{aligned}
$$

这也是反向传播中梯度必须累加，而不能直接覆盖的原因。

**计算图**

表达式 \\(z=x\cdot y+x\\) 可以写成一张有向无环图：

```text
x ──┐
    ├── Mul ──┐
y ──┘          ├── Add ── z
x ────────────┘
```

图中的张量保存数值，算子保存局部的前向和反向规则。前向计算得到 `z` 时，还需要记住 `Add` 的输入；否则反向阶段无法继续找到 `Mul`、`x` 和 `y`。

设 \\(x=3\\)、\\(y=2\\)，前向计算得到 \\(x\cdot y=6\\)、\\(z=9\\)。反向计算从 \\(\partial z/\partial z=1\\) 开始：

```text
                     ∂z/∂z = 1
                          │
                    Add::Backward
                    ┌─────┴─────┐
              传给 Mul：1    传给 x：1
                    │
                    │
               Mul::Backward
               ┌────┴────┐
       传给 x：1 × y   传给 y：1 × x
               │
               └──→ x 的两条梯度相加
```

`Add` 将上游梯度分别传给两个输入，`Mul` 再根据局部导数计算 \\(x\\) 和 \\(y\\) 的梯度。由于 \\(x\\) 同时经过 `Mul` 和 `Add` 到达 \\(z\\)，它需要累加两条路径的贡献。

最终得到：

$$
\frac{\partial z}{\partial x}=y+1=3,
\qquad
\frac{\partial z}{\partial y}=x=3
$$

这个例子同时说明了两件事：算子只计算局部梯度，训练引擎负责按照计算图组合并累加它们。

Zero 在 `Tensor` 中使用 `Back` 保存这条连接：

```cpp
struct Back {
    std::unique_ptr<TensorOp> op;
    std::vector<Tensor> inputs;
};
```

`op` 是产生当前结果的算子，`inputs` 是该算子的输入。`Back` 被当前逻辑张量的所有副本共享，算子的生命周期则由 `unique_ptr` 明确管理。

## 实现

**动态建图**

每次运算都会创建一个新的算子实例，再通过 `TensorOp::Apply` 执行：

```cpp
Tensor Tensor::operator*(const Tensor& other) const {
    return RunOp<OpMul>({*this, other});
}
```

`Apply` 先计算结果，然后检查输入是否需要梯度：

```cpp
Tensor result = op->OpValue_(inputs);

bool need_grad = std::any_of(
    inputs.begin(), inputs.end(),
    [](const Tensor& t) { return t.RequiresGrad(); });

if (need_grad) {
    result.RequiresGrad(true);
    result.back_->op = std::move(op);
    result.back_->inputs = std::move(inputs);
}
```

只有至少一个输入需要梯度时，结果才会记录计算图。推理计算和普通数据处理因此不会携带不必要的反向状态。优化器更新参数时使用 `RunOpNoGrad`，也是为了防止更新过程自身进入计算图。

**反向传播**

一个节点的梯度可能来自多条路径。在执行该节点的反向函数之前，必须先收集完所有下游贡献。Zero 首先从损失张量出发进行深度优先遍历，得到拓扑序，再逆序执行每个算子的 `Backward`。

对上面的例子，前向的依赖顺序是：

```text
x, y → Mul → Add → z
```

反向传播则按照相反顺序处理：

```text
z → Add → Mul → x, y
```

拓扑排序的意义不只是“倒着遍历”，而是保证处理某个节点时，它的所有下游梯度已经准备完成。

根节点的起始梯度为全 `1` 张量，对应：

$$
\frac{\partial L}{\partial L}=1
$$

反向遍历的核心过程可以概括为：

```cpp
for (auto it = topo.rbegin(); it != topo.rend(); ++it) {
    Tensor* current = *it;
    if (!current->back_ || !current->back_->op) continue;

    Tensor grad = RowMajorGradOf(*current);
    auto input_grads = current->back_->op->Backward(
        grad, current->back_->inputs);

    for (size_t i = 0; i < current->back_->inputs.size(); ++i) {
        auto& input = current->back_->inputs[i];
        if (input.RequiresGrad()) {
            AccumulateGrad(input, input_grads[i]);
        }
    }
}
```

第一次写入某个梯度槽时，`AccumulateGrad` 直接接管算子产生的存储；如果该梯度已经存在，则就地累加。这既满足多路径的链式法则，又避免了每次累加都分配新张量。

**张量身份**

`Back::inputs` 保存的是 `Tensor` 值副本。如果单纯使用对象地址识别计算图节点，同一个逻辑张量的不同副本就会被当成多个节点，导致上游梯度重复计算。

Zero 为每个逻辑张量设置一个共享的梯度槽：

```cpp
struct GradHolder {
    std::shared_ptr<TensorNode> grad_node;
};
```

普通 `Tensor` 副本共享 `GradHolder`，因此反向传播通过图中副本写入的梯度，用户持有的原张量也能立即看到。拓扑排序也以 `GradHolder` 的身份去重，而不是使用 `Tensor*` 或底层数据地址。

视图算子是一个特例。`Transpose` 和 `View` 可以共享数据存储，但形状和步长已经改变，因此必须使用独立的 `GradHolder`。张量的存储共享与梯度共享并不是同一件事，这一点会在下一篇详细讨论。

## 示例

有了动态计算图和反向传播，最小训练循环已经可以表达：

```cpp
auto w = Parameter(Tensor::From<float>({0.1f}, {1}));
auto b = Parameter(Tensor::From<float>({0.0f}, {1}));

for (int step = 0; step < 200; ++step) {
    w.ZeroGrad();
    b.ZeroGrad();

    auto pred = w * x + b;
    auto diff = pred - y;
    auto loss = diff * diff;
    loss.Backward();

    w.Update(w.Grad() * learning_rate);
    b.Update(b.Grad() * learning_rate);
}
```

前向表达式在计算数值的同时建立计算图，`Backward()` 将损失梯度传回 `w` 和 `b`，最后再由优化器或参数更新函数修改数值。这条链路是训练引擎的核心，后续各篇将在它之上逐步扩展数据规模、模型结构和执行设备。
