# Module 模型组织

## 引言

自动微分能够沿计算图求出梯度，却不知道哪些张量代表训练数据，哪些张量需要在每轮训练后更新。对于只有权重和 Bias 的线性模型，可以手动保存两个张量；当模型包含多个线性层、归一化、注意力和重复堆叠的 Block 时，逐个管理参数很快就会变得困难。

模型还需要保留自身的层级关系。一个参数不仅是一块需要梯度的存储，还属于某个具体的层；优化器需要遍历所有参数，模型内部则需要稳定地持有参数和子模块。如果参数散落在普通 C++ 成员中，每增加一层，都要同步修改参数收集和生命周期管理代码。

Zero 用 `Parameter` 区分可训练张量，用 `Module` 注册参数和子模块。小模块可以继续组合成更大的模型，而 `Module` 能够递归展开整棵结构，为优化器提供统一的参数列表。模型的前向计算仍然由张量算子完成，自动微分也不需要理解模型层级。

## 实现

模型组织从单个可训练张量开始。首先需要让参数具有明确的梯度和更新语义，再由模块接管参数的生命周期，并递归管理更小的子模块。在此基础上，线性层等具体网络层只需定义自己的参数和前向过程；训练模式则负责控制 Dropout 这类在训练与评估阶段行为不同的模块。

**Parameter**

`Parameter` 是一种默认需要梯度的 `Tensor`：

```cpp
class Parameter : public Tensor {
public:
    Parameter(const Tensor& tensor) : Tensor(tensor) {
        RequiresGrad(true);
    }

    void Update(const Tensor& delta) {
        *this = RunOpNoGrad<OpSub>({*this, delta});
        RequiresGrad(true);
    }
};
```

训练数据和模型参数都用 `Tensor` 存储，但语义不同：数据通常不需要梯度，参数则需要在每轮反向传播后更新。`Parameter` 在构造时开启梯度跟踪，让这个区别成为类型自身的不变式。

`Update` 在 `no_grad` 模式下执行减法。如果参数更新也记录自动微分图，后一轮训练就会继续引用前一轮的图，导致计算图和内存不断增长。

**Module**

`Module` 同时保存直接参数和子模块：

```cpp
class Module {
protected:
    std::unordered_map<std::string,
        std::unique_ptr<Parameter>> parameters_;

    std::unordered_map<std::string,
        std::shared_ptr<Module>> modules_;
};
```

参数由所属模块独占，因此使用 `unique_ptr`；子模块可以被组合和复用，使用 `shared_ptr`。`RegisterParameter` 和 `RegisterModule` 建立所有权关系，`Parameters()` 则递归遍历整棵模块树：

```text
model
├── block0.attention.weight_q
├── block0.attention.bias_q
├── block0.ffn.fc1.weight
├── block0.ffn.fc1.bias
└── ...
```

返回值中同时包含层级化名称和 `Parameter*`。优化器不需要理解模型结构，只需接收这个扁平参数列表。

**Linear**

线性层实现仿射变换：

$$
y=xW^T+b
$$

Zero 将权重存储为 `[out_features, in_features]`，使用时通过零拷贝转置视图参与矩阵乘法：

```cpp
class Linear : public Module {
public:
    Tensor OpValue(const Tensor& input) {
        return input.MatMul(weight_->T()) + *bias_;
    }

private:
    Parameter* weight_;
    Parameter* bias_;
};
```

`weight_` 和 `bias_` 是不拥有对象的原始指针，只用于快速访问；参数的生命周期由 `Module::parameters_` 中的 `unique_ptr` 管理。

权重使用 He 初始化：

$$
W_{ij}\sim\mathcal{N}\left(0,\frac{2}{\operatorname{fan\_in}}\right)
$$

这种初始化适合后续使用 ReLU 或 GeLU 的网络，能减少层数增加时激活和梯度尺度的剧烈变化。

**训练模式**

部分模块在训练和推理时行为不同。例如 Dropout 在训练时随机丢弃激活，在评估时应该直接返回输入：

```cpp
Tensor Dropout::OpValue(const Tensor& x) {
    if (p_ == 0.0f || !IsTraining()) return x;
    return RunOp<OpDropout>({x}, p_);
}
```

Zero 默认处于训练模式。验证或生成时可以在局部作用域中创建 `EvalGuard`：

```cpp
{
    EvalGuard guard;
    auto output = model.OpValue(input);
} // 离开作用域后恢复原模式
```

RAII 保证即使中途返回或抛出异常，训练状态也能正确恢复。

## 示例

复杂模型由小模块组合而成。例如一个单隐藏层 MLP：

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

同样的组合方式也用于 `FFN`、`MultiHeadAttention` 和 `TransformerBlock`。各层只提供自己的前向表达式，参数收集和自动微分由统一机制完成。

调用 `model.Parameters()` 会递归得到：

```text
fc1.weight
fc1.bias
fc2.weight
fc2.bias
```

模型内部仍然保留由子模块构成的层级结构，对外则提供一个带名称的扁平参数列表。下一篇中的优化器只需要遍历这个列表，无须了解参数属于线性层、注意力还是其他模块。

模型完成前向计算并收集参数以后，还需要由优化器统一读取梯度并更新这些参数。
