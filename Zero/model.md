# 模型与参数

自动微分能够计算任意可导表达式的梯度，但构建模型还需要区分可学习参数，并将基础网络层组合成更大的结构。Zero 用 `Parameter` 表示可训练张量，用 `Module` 管理参数和子模块。

## 可训练参数

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

## 模块组合

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

## 线性层

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

## 从线性层到模型

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

## 训练与评估

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

模型完成前向计算并收集参数以后，还需要由优化器统一读取梯度并更新这些参数。
