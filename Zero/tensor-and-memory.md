# 张量与内存

前一篇用张量表示计算图节点，但没有解释张量本身。在真实训练中，一个张量不只是一组数值，还需要记录形状、步长、数据类型、设备以及自动微分状态。

Zero 将“数据存储”与“数据解释”分开，让不同张量可以以不同形状或步长解释同一块内存。这是 `View` 和 `Transpose` 能够零拷贝实现的基础。

## 多维数组

向量、矩阵和更高维数组可以用统一的 `Tensor` 接口表示：

```cpp
Tensor a = Tensor::Zeros({2, 3});
Tensor b = Tensor::Ones({2, 3});
Tensor c = Tensor::Randn({2, 3});

Tensor flat = a.View({6});
Tensor transposed = a.T();
Tensor product = a.MatMul(b.T());
Tensor shifted = a + 5.0f;
```

`{2, 3}` 表示两行三列，但底层仍是线性存储的六个元素。从线性地址得到多维坐标，需要依靠形状和步长。

## 形状与步长

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

## 零拷贝视图

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

## 数据存储

`TensorNode` 只负责一块原始内存：

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

## 内存分配器

频繁创建临时张量时，直接为每个张量调用系统分配函数会产生明显开销。Zero 通过 `TensorContext` 抽象屏蔽具体分配策略：

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

## 张量的组成

当前 `Tensor` 由四部分组成：

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

视图则不同：它与原张量共享 `node_`，却有自己的 `meta_`、`back_` 和 `grad_holder_`。这组看似细碎的共享语义，决定了零拷贝视图和自动微分能否同时正确工作。

## 设备与类型

`TensorOption` 用于创建张量时指定设备、数据类型和是否需要梯度：

```cpp
TensorOption option;
option.SetDevice(Device::CUDA())
      .SetDataType(F32)
      .RequiresGrad(true);

Tensor x({2, 3}, option);
```

Zero 目前支持 `F32` 和 `I32`，设备支持 CPU 和 CUDA。`DefaultTensorContext::Get(device)` 为两类设备分别返回默认分配器，`Tensor::To(device)` 则通过 `DeviceTransfer` 搬运数据及已有梯度。

这种设计使张量的上层接口不需要区分 CPU 和 CUDA 存储。具体算子如何根据设备选择 kernel，则是下一篇的内容。
