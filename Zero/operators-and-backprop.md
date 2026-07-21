# 算子与反向传播

张量解决了数据如何存储，算子则定义数据如何变换。一个可参与自动微分的算子，必须同时实现前向计算和反向计算；训练引擎负责记录算子之间的连接，并把上游梯度传给每个算子。

Zero 将这个过程分成三层：

```text
Tensor 接口
    ↓
TensorOp：形状检查、建图、前向/反向规则
    ↓
ops：按 Device 派发到 CPU 或 CUDA kernel
```

上层代码因此可以使用相同的 `Tensor` 表达式运行在 CPU 或 GPU 上。

## 算子抽象

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

## 设备派发

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
2. 在 `tensor_ops_cpu.*` 实现 CPU kernel。
3. 在 `tensor_ops_gpu.*` 实现 CUDA kernel。
4. 在 `tensor_ops.h` 增加设备派发入口。

## 逐元素算子

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

实际代码还会检查输入数量和广播关系。这些检查属于算子语义，不应散落在 CPU 和 CUDA kernel 中。

## 广播

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

Zero 当前会将较小的输入显式广播到目标形状，再执行形状相同的 kernel。广播 kernel 将输出线性下标还原为多维坐标，再映射回输入；输入中大小为 `1` 的维度，坐标始终为 `0`。

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

线性层的 bias 加法是一个高频特例。对于最后一维为 \\(C\\) 的输入和形状 `[C]` 的 bias，`OpAdd` 会走专用路径，避免先生成一个完整的 bias 广播中间张量。

## 矩阵乘法

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

**步长不能忽略**

线性层常用 `x.MatMul(weight.T())`。`T()` 是零拷贝视图，所以 `weight.T()` 不是默认行优先布局。如果 kernel 用 `m * K + k` 这类连续下标读取它，就会把转置前的存储误当成转置后的矩阵。

正确的读取方式必须使用张量步长：

$$
\begin{aligned}
\operatorname{offset}&#95;A(m,k)
&= m\cdot s&#95;{A,-2}+k\cdot s&#95;{A,-1}
\end{aligned}
$$

CPU 和 CUDA 的 MatMul kernel 都必须遵守这条规则。这不是一个可选优化，而是计算转置视图时的正确性要求。

## 视图的反向过程

`View` 和 `Transpose` 虽然只改变张量的形状或步长，不拷贝前向数据，但它们仍然是计算图中的算子。反向传播时，需要将视图形状下的梯度转换回输入的形状和布局。

`Transpose` 的反向过程需要将梯度的对应维度再交换一次：

```cpp
auto grad_view = grad_output.ViewWithMeta(
    SwapDims(grad_output.Meta(), dim0_, dim1_));

return {OpBroadcastTo(grad_view, inputs[0].Sizes())};
```

这里的 `OpBroadcastTo` 不是在逻辑上扩展形状，而是利用通用的 stride-aware 拷贝将视图梯度实体化为连续存储。

`View` 在输入连续时只重新解释形状；如果输入来自 `Transpose` 等非连续视图，Zero 会先生成行优先的连续副本，再改变形状。这样才能保证 Attention 中的 `Transpose(...).View(...)` 按正确顺序解释底层数据。

## 算子融合

算子抽象让系统容易组合，但过度细分也会带来中间张量、kernel 启动和计算图节点开销。例如 LayerNorm 可以拆成 `Mean`、`Variance`、`Sub`、`Sqrt`、`Div`、`Mul` 和 `Add`，但训练时这条路径会反复分配临时张量。

Zero 因此在保留基础算子的同时，为高频组合提供融合算子，包括：

- `OpLayerNorm`
- `OpLinearBias`
- `OpFFN`
- `OpLinearCrossEntropy`
- `OpFlashAttention`

融合不改变数学定义，只是把多个小算子的前向和反向合并到更粗粒度的实现中。它是训练引擎从“能计算”走向“能有效训练”必须经过的一步。

张量和算子构成了计算基础，下一篇将在这套接口之上组织参数与模型层。
