# Tensor 张量存储

## 引言

前一篇使用张量保存计算结果，并将它们连接成计算图，但张量本身还没有展开。最简单的实现似乎只需要一个 `std::vector<float>`：申请一段连续内存，再把数值依次放进去。然而模型处理的不只是向量，还包括批量输入、权重矩阵和多维激活。同一组数值可能需要被变形或转置，也可能位于 CPU 或 GPU，并在反向传播时拥有对应的梯度。单独一段内存无法表达这些信息。

因此，张量不只是一组数值。它还需要记录形状、步长、数据类型、设备和自动微分状态，并明确这些信息在复制与变形时怎样共享。Zero 将“数据存储”与“数据解释”分开：底层内存只保存数值，元数据决定如何把它解释成多维数组。理解这个区别，是理解零拷贝视图和张量共享语义的起点。

**多维数组**

考虑一个两行三列的矩阵：

```text
[[1, 2, 3],
 [4, 5, 6]]
```

从数学上看，它有两个维度；从内存角度看，它仍然只是连续排列的六个数值。张量需要把这两种视角连接起来。Zero 使用统一的 `Tensor` 接口表示向量、矩阵和更高维数组：

```cpp
Tensor a = Tensor::Zeros({2, 3});
Tensor b = Tensor::Ones({2, 3});
Tensor c = Tensor::Randn({2, 3});

Tensor flat = a.View({6});
Tensor transposed = a.T();
Tensor product = a.MatMul(b.T());
Tensor shifted = a + 5.0f;
```

`{2, 3}` 表示两行三列。`a`、`b` 和 `c` 具有相同形状，却可以采用不同的初始化方式；`View` 将六个元素重新解释成一维数组，`T()` 交换两个维度，`MatMul` 和加法则在这些结构之上执行计算。

这些接口背后需要处理不同的关系：`View` 和 `Transpose` 应该复用已有数据，计算结果需要拥有新的存储，普通张量副本又必须共享正确的计算图和梯度。如果这些边界没有定义清楚，减少一次数据拷贝就可能换来错误的布局或反向传播。张量设计的关键，正是确定哪些状态应该共享，哪些状态必须彼此独立。

## 实现

无论张量具有多少个维度，底层内存最终都是一段一维的连续地址。多维坐标不能直接用于访问这段内存，框架必须先把它转换成相对于起始位置的下标。以一个矩阵为例，沿列移动通常只需访问下一个元素，沿行移动则要跨过一整行；转置以后，两个方向的移动方式还会互换。形状描述每个维度包含多少个元素，步长描述沿各个维度移动一次需要跨过多少个内存位置，二者共同建立多维坐标与线性存储之间的对应关系。

**形状与步长**

`TensorMeta` 保存张量的元数据：

```cpp
struct TensorMeta {
    std::vector<int64_t> sizes;
    std::vector<int64_t> strides;
    Device device;
    DataType datatype;
};
```

`sizes` 定义每个维度的长度，`strides` 定义对应坐标增加 \\(1\\) 时，内存下标需要跨过多少个元素。

对一个行优先存储的 \\(2\times3\\) 矩阵：

```text
矩阵：[[1, 2, 3],
      [4, 5, 6]]

内存：[1, 2, 3, 4, 5, 6]
sizes:   [2, 3]
strides: [3, 1]
```

位置 \\((i,j)\\) 对应的线性下标为：

$$
\operatorname{offset}(i,j)=i\cdot3+j\cdot1
$$

例如 \\((1,0)\\) 对应下标 \\(3\\)，所以读到数值 \\(4\\)。

`TensorMeta::InitStrides` 从最后一维开始计算默认步长：

```cpp
void InitStrides() {
    strides.resize(sizes.size());
    int64_t stride = 1;
    for (int i = sizes.size() - 1; i >= 0; --i) {
        strides[i] = stride;
        stride *= sizes[i];
    }
}
```

对形状 `[2, 3, 4]`，结果是 `[12, 4, 1]`。`NumElements()` 将所有维度相乘，`NumBytes()` 再乘以单个元素的字节数，得到需要分配的内存大小。

形状回答“每个维度有多长”，步长回答“沿某个维度移动时怎样访问内存”。二者共同决定张量的布局，也使同一块内存可以拥有不止一种解释方式。

**零拷贝视图**

转置一个 \\(2\times3\\) 矩阵时，不需要移动数据，只需交换形状和步长：

```text
转置前：sizes=[2, 3], strides=[3, 1]
转置后：sizes=[3, 2], strides=[1, 3]
```

转置后的 \\((0,1)\\) 对应内存下标：

$$
0\cdot1+1\cdot3=3
$$

因此读到原矩阵 \\((1,0)\\) 位置的数值 \\(4\\)。底层数据没有改变，改变的只是访问方式。

Zero 的 `Transpose` 通过 `ViewWithMeta` 创建这种视图：

```cpp
Tensor Tensor::ViewWithMeta(TensorMeta new_meta) const {
    Tensor view;
    view.node_ = node_;
    view.meta_ = std::move(new_meta);
    view.grad_holder_ = std::make_shared<GradHolder>();
    return view;
}
```

视图共享 `node_`，因此指向同一块数据；但它拥有自己的 `meta_` 和 `GradHolder`。原张量和转置视图的梯度形状不同，如果共享梯度槽，反向传播就会以错误的步长解释梯度。

`View` 同样可以共享数据，但它有一个限制：当前布局必须能够用新形状合法表示。这样的设计避免了形状操作中不易察觉的数据拷贝。

**数据存储**

视图能够共享数据，是因为原始内存没有直接放在某个 `Tensor` 对象中，而是由独立的 `TensorNode` 管理。它只负责一块原始内存：

```cpp
class TensorNode {
public:
    TensorNode(TensorContext* ctx, size_t num_bytes)
        : ctx_(ctx), data_ptr_(ctx_->NewMem(num_bytes)) {}

    ~TensorNode() {
        if (data_ptr_) ctx_->DeleteMem(data_ptr_);
    }

private:
    TensorContext* ctx_;
    std::byte* data_ptr_;
};
```

它在构造时申请内存，析构时归还内存，并禁止自身拷贝和移动，避免多个对象重复管理同一指针。`Tensor` 通过 `shared_ptr<TensorNode>` 共享它，最后一个引用消失后才真正归还内存。

反向传播还会在某个中间结果的数据不再被后续算子使用时，调用 `ReleaseData()` 提前归还存储。这能避免所有前向激活都一直保留到整张计算图被销毁。

**内存分配器**

一次前向和反向计算会产生大量中间张量。矩阵乘法、激活函数和梯度计算都需要临时结果，如果每次都向系统申请并立即释放内存，分配成本可能超过一些小算子本身的计算成本。Zero 通过 `TensorContext` 抽象屏蔽具体分配策略：

```cpp
class TensorContext {
public:
    virtual std::byte* NewMem(size_t size) = 0;
    virtual void DeleteMem(std::byte* ptr) = 0;
};
```

**CPU 分配器**

`CPUTensorContext` 使用空闲链表保存已释放的内存块。新请求先按 \\(16\\) 字节对齐，再查找能容纳请求的最小空闲块。如果找到的块过大，则重新分配，避免一个小张量长期占用大块内存。空闲块数量过多时，多余内存会真正返回系统。

**CUDA 分配器**

`cudaMalloc` 不仅是分配操作，还会引入驱动层同步开销。`CUDATensorContext` 因此将请求向上对齐到 \\(512\\) 字节，并按大小放入不同的缓存桶：

```text
申请内存 → 对齐到桶大小 → 命中空闲块？
                              ├─ 是：直接复用
                              └─ 否：cudaMalloc

释放内存 → 放回对应的空闲桶
```

缓存的块默认不会立即返回 CUDA。当训练从大 batch 切换到单样本生成时，张量形状发生大幅改变，旧桶往往无法复用。此时可以调用 `TrimFreePool()` 将空闲块真正返回设备，为新阶段腾出显存。

**张量组成**

至此，数值由 `TensorNode` 保存，形状与步长由 `TensorMeta` 解释，计算图和梯度又有各自的生命周期。它们不能简单地绑在同一个对象中，否则一次浅拷贝或视图变换就会同时改变所有状态。

当前 `Tensor` 因此由四部分组成：

```cpp
class Tensor {
    std::shared_ptr<TensorNode> node_;
    TensorMeta meta_;
    std::shared_ptr<Back> back_;
    std::shared_ptr<GradHolder> grad_holder_;
};
```

它们分别遵循不同的共享规则：

| 成员 | 作用 | 拷贝 `Tensor` 时 |
|---|---|---|
| `node_` | 底层数据 | 共享存储 |
| `meta_` | 形状、步长、设备和类型 | 按值拷贝 |
| `back_` | 产生当前结果的算子与输入 | 共享计算图记录 |
| `grad_holder_` | 梯度存储槽 | 共享梯度 |

因此 `Tensor b = a` 是浅拷贝：`a` 和 `b` 代表同一个逻辑张量，共享数据、计算图位置和梯度槽。这不仅避免了大块数据拷贝，也保证计算图中的副本写入梯度后，用户持有的张量可以读到相同结果。

视图则不同：它与原张量共享 `node_`，却有自己的 `meta_`、`back_` 和 `grad_holder_`。回到开头的 \\(2\times3\\) 矩阵，转置视图和原张量读取的是同样六个数值，却拥有不同的形状、步长和梯度关系。这组共享语义决定了零拷贝视图和自动微分能否同时正确工作。

**设备与类型**

`TensorOption` 用于创建张量时指定设备、数据类型和是否需要梯度：

```cpp
TensorOption option;
option.SetDevice(Device::CUDA())
      .SetDataType(F32)
      .RequiresGrad(true);

Tensor x({2, 3}, option);
```

Zero 目前支持 `F32` 和 `I32`，设备支持 CPU 和 CUDA。`DefaultTensorContext::Get(device)` 为两类设备分别返回默认分配器，`Tensor::To(device)` 则通过 `DeviceTransfer` 搬运数据及已有梯度。

这种设计使张量的上层接口不需要区分 CPU 和 CUDA 存储。无论数据位于哪种设备，张量都使用形状和步长解释内存，并通过相同的共享规则参与计算图。

张量至此解决了“数据是什么”以及“数据怎样存放”的问题。下一篇将继续处理“数据怎样计算”：算子如何读取不同布局的张量，并根据设备选择对应的 Kernel。
