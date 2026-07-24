# Operator 算子实现

## 引言

张量解决了数据如何表示和存储，但它本身并不知道两个张量能否相加、矩阵乘法会产生什么形状，也不知道计算结果应当怎样参与反向传播。这些规则由算子定义。一次看似简单的乘法，既要检查输入形状、分配输出并完成数值计算，也要记录输入之间的依赖，并在反向阶段根据上游梯度计算每个输入的梯度。

如果把这些工作全部放进 `Tensor`，数据结构会同时承担存储、计算和设备实现；如果每个算子分别处理 CPU 与 CUDA，形状规则和反向逻辑又容易重复。Zero 因此将一次算子调用分成三层：

```text
Tensor 接口
    ↓
TensorOp：形状检查、建图、前向/反向规则
    ↓
Ops：按 Device 派发到 CPU 或 CUDA Kernel
```

`Tensor` 提供面向使用者的运算接口，`TensorOp` 定义算子的形状语义、前向与反向规则，并决定是否记录计算图；最底层的 Ops 只负责把数值计算派发到对应设备。三层之间各自处理一种变化：用户表达式保持不变，算子规则与设备实现可以独立扩展。

因此，同一段 `Tensor` 表达式既可以运行在 CPU 上，也可以迁移到 GPU，而自动微分仍然沿用相同的计算图和反向规则。下面先从连接这三层的 `TensorOp` 开始。

## 实现

**算子抽象**

`TensorOp` 只要求子类实现两个核心函数：

```cpp
class TensorOp {
public:
    static Tensor Apply(
        std::unique_ptr<TensorOp> op,
        std::vector<Tensor> inputs,
        bool no_grad = false);

    std::vector<Tensor> Backward(
        const Tensor& grad_output,
        const std::vector<Tensor>& inputs) const;

protected:
    virtual Tensor OpValue_(
        const std::vector<Tensor>& inputs) const = 0;

    virtual std::vector<Tensor> Backward_(
        const Tensor& grad_output,
        const std::vector<Tensor>& inputs) const = 0;
};
```

`OpValue_` 计算前向结果，`Backward_` 接收结果的上游梯度，返回与输入一一对应的梯度张量。`Apply` 在前向完成后判断是否需要建图，而公开的 `Backward` 还会包装算子性能计时。

用户接口通过 `RunOp` 为每次调用创建独立算子：

```cpp
template <typename Op, typename... Args>
Tensor RunOp(std::vector<Tensor> inputs, Args&&... args) {
    return TensorOp::Apply(
        std::make_unique<Op>(std::forward<Args>(args)...),
        std::move(inputs));
}
```

Zero 不设置全局算子注册表，而是为每次前向调用创建独立的算子对象。算子只保存少量参数，构造成本相比张量计算可以忽略；算子的状态与生命周期则明确归属于它产生的计算图节点。

**设备派发**

`TensorOp` 不直接实现数值循环，而是调用 `ops` 命名空间中的统一入口。例如逐元素乘法：

```cpp
inline void mul(const Tensor& a, const Tensor& b, Tensor& out) {
    on_gpu(a) ? mul_cuda(a, b, out)
              : mul_cpu(a, b, out);
}

inline void mul_backward(
    const Tensor& grad, const Tensor& a, const Tensor& b,
    Tensor& grad_a, Tensor& grad_b) {
    on_gpu(grad)
        ? mul_backward_cuda(grad, a, b, grad_a, grad_b)
        : mul_backward_cpu(grad, a, b, grad_a, grad_b);
}
```

算子层只处理形状、参数和自动微分，CPU 与 CUDA 的具体算法分别放在 `tensor_ops_cpu.cc` 和 `tensor_ops_gpu.cu`。这样新增算子时，边界非常明确：

1. 在 `ops.h` 和 `ops.cc` 定义前向、反向及形状语义。
2. 在 `tensor_ops_cpu.*` 实现 CPU Kernel。
3. 在 `tensor_ops_gpu.*` 实现 CUDA Kernel。
4. 在 `tensor_ops.h` 增加设备派发入口。

**逐元素算子**

加、减、乘、除的直接情况是两个输入形状相同。以乘法为例，前向计算为：

$$
z_i=x_i y_i
$$

反向计算为：

$$
\begin{aligned}
\frac{\partial L}{\partial x_i}
&= \frac{\partial L}{\partial z_i}y_i,
\qquad
\frac{\partial L}{\partial y_i}
&= \frac{\partial L}{\partial z_i}x_i
\end{aligned}
$$

相应的算子实现只需要分配结果与梯度，再调用设备无关的派发入口：

```cpp
Tensor OpMul::OpValue_(const std::vector<Tensor>& inputs) const {
    Tensor out(inputs[0].Sizes(), Tensor::LikeOption(inputs[0]));
    ops::mul(inputs[0], inputs[1], out);
    return out;
}

std::vector<Tensor> OpMul::Backward_(
    const Tensor& grad, const std::vector<Tensor>& inputs) const {
    Tensor grad_a(inputs[0].Sizes(), Tensor::LikeOption(grad));
    Tensor grad_b(inputs[1].Sizes(), Tensor::LikeOption(grad));
    ops::mul_backward(grad, inputs[0], inputs[1], grad_a, grad_b);
    return {grad_a, grad_b};
}
```

实际代码还会检查输入数量和广播关系。这些检查属于算子语义，不应散落在 CPU 和 CUDA Kernel 中。

**广播**

当两个张量的形状不同时，逐元素算子可以按广播规则对齐它们。规则从最后一维向前比较，每一维需要满足以下条件之一：

- 两个维度相等。
- 其中一个维度为 `1`。
- 其中一个维度不存在，视为 `1`。

```text
A: [2, 3, 4]
B: [   3, 1]
             → [2, 3, 4]

A: [2, 3, 4]
B: [2, 4, 5]
             → 不兼容
```

Zero 当前会将较小的输入显式广播到目标形状，再执行形状相同的 Kernel。广播 Kernel 将输出线性下标还原为多维坐标，再映射回输入；输入中大小为 `1` 的维度，坐标始终为 `0`。

例如将 `[10, 20, 30]` 从 `[3]` 广播到 `[2, 3]`：

```text
[[10, 20, 30],
 [10, 20, 30]]
```

前向过程中的重复意味着反向过程中的求和。如果原始元素 \\(x_j\\) 被映射到多个输出位置，它的梯度为：

$$
\begin{aligned}
\frac{\partial L}{\partial x_j}
&= \sum_{i\mapsto j}
   \frac{\partial L}{\partial y_i}
\end{aligned}
$$

`OpSumToSize` 使用 `broadcast_backward` 将大形状的梯度收缩回原输入形状。因此所有二元算子都可以共用同一组流程：

```text
前向：广播小张量 → 执行算子
反向：计算大形状梯度 → SumToSize 回原形状
```

线性层的 Bias 加法是一个高频特例。对于最后一维为 \\(C\\) 的输入和形状 `[C]` 的 Bias，`OpAdd` 会走专用路径，避免先生成一个完整的 Bias 广播中间张量。

**矩阵乘法**

对于矩阵 \\(A\in\mathbb{R}^{M\times K}\\) 和 \\(B\in\mathbb{R}^{K\times N}\\)，前向计算为：

$$
C_{mn}=\sum_{k=1}^{K}A_{mk}B_{kn}
$$

其反向公式为：

$$
\begin{aligned}
\frac{\partial L}{\partial A}
&= \frac{\partial L}{\partial C}B^T,
\qquad
\frac{\partial L}{\partial B}
&= A^T\frac{\partial L}{\partial C}
\end{aligned}
$$

Zero 同时支持普通矩阵乘法、批量矩阵乘法和 batch 维度广播：

```text
[M, K]       @ [K, N]       → [M, N]
[B, M, K]    @ [B, K, N]    → [B, M, N]
[1, M, K]    @ [B, K, N]    → [B, M, N]
```

当 batch 维度需要广播时，前向先对齐两个输入；反向计算完成后，再将梯度收缩回各自的原始形状。

**步长处理**

线性层常用 `x.MatMul(weight.T())`。`T()` 是零拷贝视图，所以 `weight.T()` 不是默认行优先布局。如果 Kernel 用 `m * K + k` 这类连续下标读取它，就会把转置前的存储误当成转置后的矩阵。

正确的读取方式必须使用张量步长：

$$
\begin{aligned}
\operatorname{offset}&#95;A(m,k)
&= m\cdot s&#95;{A,-2}+k\cdot s&#95;{A,-1}
\end{aligned}
$$

CPU 和 CUDA 的 MatMul Kernel 都必须遵守这条规则。这不是一个可选优化，而是计算转置视图时的正确性要求。

**视图反向**

`View` 和 `Transpose` 虽然只改变张量的形状或步长，不拷贝前向数据，但它们仍然是计算图中的算子。反向传播时，需要将视图形状下的梯度转换回输入的形状和布局。

`Transpose` 的反向过程需要将梯度的对应维度再交换一次：

```cpp
auto grad_view = grad_output.ViewWithMeta(
    SwapDims(grad_output.Meta(), dim0_, dim1_));

return {OpBroadcastTo(grad_view, inputs[0].Sizes())};
```

这里的 `OpBroadcastTo` 不是在逻辑上扩展形状，而是利用通用的 stride-aware 拷贝将视图梯度实体化为连续存储。

`View` 在输入连续时只重新解释形状；如果输入来自 `Transpose` 等非连续视图，Zero 会先生成行优先的连续副本，再改变形状。这样才能保证 Attention 中的 `Transpose(...).View(...)` 按正确顺序解释底层数据。

## 融合

基础算子让模型能够自由组合，但组合得越细，执行时需要保存的中间张量和计算图节点就越多。在 GPU 上，每个小算子通常还对应一次 Kernel 启动和一次显存读写；计算本身很少时，这些固定开销会变得格外明显。

**减少调度**

以线性变换为例，普通实现需要先完成矩阵乘法，再单独执行 Bias 加法：

```text
x @ Wᵀ → 中间结果 → Bias Add → y
```

`OpLinearBias` 使用 cuBLASLt 的 Bias Epilogue，在矩阵乘法结束时直接完成加法：

```text
x、W、b → LinearBias → y
```

这样可以减少一次 Kernel 启动，也不必把矩阵乘法结果完整写回显存后再读取。`OpLayerNorm` 采用相同思路，将均值、方差、归一化和仿射变换放进一个算子，避免由多个基础算子反复产生临时结果。

**控制存储**

融合还可以改变中间结果的保存方式。语言模型输出层会先计算形状为 `[N, V]` 的 Logits，再计算交叉熵；当词表较大时，这个张量会占用大量显存。`OpLinearCrossEntropy` 在前向阶段只返回每个 Token 的损失，不让 Logits 长期留在计算图中，反向传播时再根据输入和权重重新计算。

`OpFFN` 和 `OpFlashAttention` 也采用这种策略。它们在反向阶段重新计算部分激活，以减少需要跨越整个前向过程保存的中间张量。这是一种计算与显存之间的取舍：反向计算有所增加，但训练能够使用更大的 Batch 或序列长度。

**融合边界**

Zero 在保留基础算子的同时，只为高频且边界稳定的组合提供融合实现：

- `OpLayerNorm`
- `OpLinearBias`
- `OpFFN`
- `OpLinearCrossEntropy`
- `OpFlashAttention`

融合算子仍然遵守 `TensorOp` 的统一接口：前向接收张量并返回结果，反向接收上游梯度并返回各输入的梯度。它没有改变模型的数学定义，只是把多个小算子的计算和存储安排合并到更粗的执行单元中。

如果所有表达式都改写为专用融合算子，框架会失去组合能力，代码也难以维护。因此，基础算子负责表达完整语义，融合算子只处理已经成为性能或显存瓶颈的关键路径。张量和算子由此构成计算基础，下一篇将在这套接口之上组织参数与模型层。
