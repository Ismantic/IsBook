# Python 前端

## 引言

前面的章节已经在 C++ 中实现了张量、自动微分、算子和参数更新。这些能力适合承担数值计算，却不一定适合直接描述模型和训练流程。C++ 需要显式处理类型、所有权和编译过程；Python 则更适合快速组合网络层、准备输入并控制训练循环。

训练代码经常需要反复调整模型结构、损失函数、学习率和数据处理方式。如果这些变化都写在 C++ 中，每次修改后都要重新编译，再运行程序观察结果。Python 可以直接执行新的模型与训练逻辑，也便于在运行过程中打印张量形状、损失和梯度，缩短发现问题、修改代码和重新验证之间的周期。只有底层算子或绑定发生变化时，才需要重新编译 C++ 扩展。

Python 前端并不是重新实现一套训练框架，而是在 C++ 核心之上提供另一种表达方式。两层之间的关系可以概括为：

```text
Python：模型组合与训练流程
              ↓
PyBind11：类型与调用转换
              ↓
C++：Tensor、Autograd 与 Operator
```

Python 创建张量并调用算子，实际的数值计算和计算图仍由 C++ 完成。这样既保留底层实现的性能和统一语义，又让上层训练代码保持简洁。

## 实现

Python 前端需要解决两个问题：首先将 C++ 的张量和算子安全地暴露给 Python，然后用这些基础接口组织模型与参数更新。绑定层只负责跨越语言边界，`Module`、`Linear` 和 `SGD` 则使用普通 Python 代码完成组合。

**PyBind11**

Zero 使用 PyBind11 将 C++ 类型封装为 `_zero` 扩展模块。绑定层暴露 `Tensor`、`Parameter` 和常用算子，并负责在两种语言之间转换参数与返回值：

```cpp
py::class_<Tensor>(m, "Tensor")
    .def("backward", &Tensor::Backward)
    .def_property_readonly("grad", [](const Tensor& tensor) {
        return tensor.Grad();
    })
    .def("__matmul__", [](const Tensor& lhs, const Tensor& rhs) {
        return lhs.MatMul(rhs);
    });
```

当 Python 执行 `loss.backward()` 时，调用会直接进入 C++ 的 `Tensor::Backward()`。矩阵乘法、加减乘、转置、变形、ReLU 和交叉熵也沿用同一套 C++ 算子，因此绑定前后共享相同的前向与反向规则。

Python 列表创建张量时，绑定层先检查嵌套列表是否具有规则形状，再将数值展开为连续数组。张量返回 Python 时，则根据 `shape` 重新构造嵌套列表。这个边界只负责数据表示的转换，不参与数值计算。

**Module**

Python 层的 `Module` 保持得很小。`__call__` 将调用转发给 `forward`，`parameters()` 则递归收集对象属性中的 `Parameter` 和子模块：

```python
class Module:
    def parameters(self):
        result = []
        for value in vars(self).values():
            if isinstance(value, Parameter):
                result.append(value)
            elif isinstance(value, Module):
                result.extend(value.parameters())
        return result

    def __call__(self, *args, **kwargs):
        return self.forward(*args, **kwargs)
```

**Linear**

`Linear` 由参数和已有张量运算组成：

```python
class Linear(Module):
    def __init__(self, in_features, out_features):
        scale = math.sqrt(2.0 / in_features)
        self.weight = Parameter(
            randn([out_features, in_features]) * scale
        )
        self.bias = Parameter(randn([out_features]) * 0.0)

    def forward(self, x):
        return x @ self.weight.T + self.bias
```

前向过程不需要特殊的模型执行器。每一次 Python 运算都会调用对应的 C++ 算子，并动态建立计算图。

**SGD**

Python 版 `SGD` 保存 `model.parameters()` 返回的参数。`zero_grad()` 清除上一轮梯度，`step()` 按学习率生成更新量：

```python
def step(self):
    for parameter in self.parameters:
        if parameter.grad is not None:
            parameter.update(parameter.grad * self.lr)
```

`Parameter.update()` 最终仍在 C++ 中完成无梯度的减法。这样，Python 可以控制训练循环，而参数存储、梯度和更新语义仍由底层统一管理。

## 示例

下面的线性回归同时经过 Python 模型、C++ 算子、自动微分和参数更新：

```python
import zero

model = zero.nn.Linear(2, 1)
optimizer = zero.optim.SGD(model.parameters(), lr=0.05)

x = zero.tensor([[1.0, 2.0], [2.0, 1.0]])
target = zero.tensor([[5.0], [4.0]])

for _ in range(100):
    prediction = model(x)
    difference = prediction - target
    loss = (difference * difference).mean()

    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
```

`model(x)` 从 Python 进入绑定层，并调用 C++ 的矩阵乘法、转置和加法；`loss.backward()` 沿 C++ 计算图生成参数梯度；`optimizer.step()` 再通过 `Parameter.update()` 修改底层参数。循环仍由 Python 控制，但每一步张量计算都复用了同一套 C++ 实现。

Python 前端由此展示了训练框架的一条边界：底层提供稳定的计算能力，上层提供灵活的表达方式。接口不需要覆盖完整的 PyTorch，只要足以把前面实现的组件组合成一次可运行的训练即可。
