# MxArray

## 引言

前面的自动微分实现只能处理标量，而实际上的深度学习计算涉及大量的多维数组：向量、矩阵、以及更高维的张量。

接下来需要实现的是多维数组，其能够：

1. **高效存储**：连续内存布局，减少内存碎片
2. **零拷贝操作**：Transpose和View等操作不复制数据
3. **广播支持**：自动处理不同形状数组间的运算
4. **向量化计算**：一次操作处理整个数组
5. **保持自动微分**：与前面的计算图机制无缝集成


以NumPy中的多维数组来举例：


```python
import numpy as np

# 一维数组（向量）
a = np.array([1, 2, 3, 4])
print(f"Shape: {a.shape}, Size: {a.size}")  # Shape: (4,), Size: 4

# 二维数组（矩阵）
b = np.array([[1, 2, 3],
              [4, 5, 6]])
print(f"Shape: {b.shape}, Size: {b.size}")  # Shape: (2, 3), Size: 6

# 高维数组（张量）
c = np.zeros((2, 3, 4))  # 2×3×4 的三维数组
print(f"Shape: {c.shape}, Size: {c.size}")  # Shape: (2, 3, 4), Size: 24

# 形状操作
d = a.reshape(2, 2)      # 改变形状
e = b.T                  # 转置
f = np.broadcast_to(a, (2, 4))  # 广播
```

NumPy的成功很大程度上就在于其强大的多维数组支持：
- **统一的接口**：无论是标量、向量、矩阵还是高维张量，都使用相同的操作方式
- **高效的内存管理**：通过智能的内存布局和零拷贝操作避免不必要的数据复制
- **灵活的广播机制**：让不同形状的数组可以自然地进行运算

相对应，本项目的C++实现的Tensor希望能做到类似功能，同时支持自动微分。

```cpp
// 基本创建
Tensor a = Tensor::Zeros({2, 3});        // 2×3的零矩阵
Tensor b = Tensor::Ones({2, 3});         // 2×3的全1矩阵
Tensor c = Tensor::Randn({2, 3});        // 2×3的随机矩阵

// 形状操作
Tensor d = a.View({6});                  // 重塑为一维
Tensor e = a.T();                        // 转置
Tensor f = a.Transpose(0, 1);            // 指定维度转置

// 基础运算
Tensor g = a + b;                        // 逐元素加法
Tensor h = a * b;                        // 逐元素乘法
Tensor i = a.MatMul(b.T());              // 矩阵乘法

// 广播运算
Tensor j = a + 5.0f;                     // 标量广播
Tensor k = a + Tensor::Ones({3});        // 向量广播

// 自动微分
a.RequiresGrad(true);
Tensor loss = (a * b).Sum();
loss.Backward();
Tensor grad = a.Grad();                  // 获取梯度
```

## Tensor

接下来阐述Tensor的具体实现。关键点是**分离数据存储和数据解释**，这也是NumPy的做法。

- **数据存储**：原始的内存会，不关心如何解释字节
- **数据解释**：如何将内存理解为多维数组（形状/步长/类型）

考虑这样一个场景：
```cpp
Tensor a = Tensor::Randn({2, 6});    // 原始数据：12个浮点数
Tensor b = a.View({3, 4});           // 重塑：仍然是这12个数，但解释为3×4
Tensor c = a.T();                    // 转置：还是这12个数，但访问顺序不同
```

如果每次操作都复制数据，性能将无法接受。我们的解决方案是：
- **TensorMeta**：描述如何解释这些数据（形状、步长等）
- **TensorNode**：管理实际的内存数据
- **TensorContext**：内存池，提供高效的内存管理
- **TensorOption**：提供定制Tensor的选项
- **Tensor**：组合以上，提供用户接口

### TensorMeta

描述数据的“形状”：

```cpp
struct TensorMeta {
    std::vector<int64_t> sizes;     // 形状：[2, 3, 4]
    std::vector<int64_t> strides;   // 步长：[12, 4, 1]
    DataType datatype;              // 数据类型：F32 或 I32
    
    void InitStrides() {
        strides.resize(sizes.size());
        int64_t stride = 1;
        // 从最后一维开始计算步长（行优先存储）
        for (int i = sizes.size() - 1; i >= 0; --i) {
            strides[i] = stride;
            stride *= sizes[i];
        }
    }
    
    size_t NumElements() const {
        size_t n = 1;
        for (int64_t s : sizes) n *= s;
        return n;
    }
    
    size_t NumBytes() const {
        return NumElements() * DataTypeSize(datatype);
    }
};
```

**理解步长(stride)**

步长定义了在内存中访问下一个元素需要跳过多少个元素。考虑一个形状为$(2, 3)$的矩阵：

```
矩阵：[[1, 2, 3],
      [4, 5, 6]]

内存布局：[1, 2, 3, 4, 5, 6]
形状：[2, 3]
步长：[3, 1]

访问公式：memory_index = i * stride[0] + j * stride[1]
- matrix[0,0] → 0*3 + 0*1 = 0 → memory[0] = 1
- matrix[0,1] → 0*3 + 1*1 = 1 → memory[1] = 2  
- matrix[1,0] → 1*3 + 0*1 = 3 → memory[3] = 4
```

步长的威力在于可以实现零拷贝的转置：

```
转置后：
形状：[3, 2]
步长：[1, 3]  // 注意：只是交换了步长！

访问：
- matrix_t[0,0] → 0*1 + 0*3 = 0 → memory[0] = 1
- matrix_t[0,1] → 0*1 + 1*3 = 3 → memory[3] = 4
- matrix_t[1,0] → 1*1 + 0*3 = 1 → memory[1] = 2
```

### TensorNode

管理内存：

```cpp
class TensorNode {
public:
    TensorNode(TensorContext* ctx, size_t num_bytes)
        : ctx_(ctx), data_ptr_(nullptr) {
        data_ptr_ = ctx_->NewMem(num_bytes);
        if (!data_ptr_) {
            throw std::runtime_error("Failed to allocate tensor memory.");
        }
    }
    
    ~TensorNode() {
        if (data_ptr_) {
            ctx_->DeleteMem(data_ptr_);
        }
    }
    
    // 禁用拷贝和移动，避免内存管理问题
    TensorNode(const TensorNode&) = delete;
    TensorNode& operator=(const TensorNode&) = delete;
    TensorNode(TensorNode&&) = delete;
    TensorNode& operator=(TensorNode&&) = delete;
    
    template<typename T>
    T* Data() { return reinterpret_cast<T*>(data_ptr_); }
    
private:
    TensorContext* ctx_;                    // 内存分配器
    std::byte* data_ptr_;                   // 数据指针
    std::shared_ptr<TensorNode> grad_node_; // 梯度存储
    friend class Tensor;
};
```

**注意点：**
1. **RAII管理**：构造时分配内存，析构时自动释放（需通过ctx_）
2. **禁用拷贝**：避免意外的内存重复管理
3. **模板访问**：类型安全的数据访问
4. **梯度存储**：`grad_node_`指向存储梯度的另一个TensorNode


**为什么梯度节点是shared_ptr？**

```cpp
Tensor a = Tensor::Randn({100, 200}).RequiresGrad(true);
Tensor grad_a1 = a.Grad();  // 第一次获取梯度
Tensor grad_a2 = a.Grad();  // 第二次获取梯度
// grad_a1和grad_a2应该指向同一块梯度内存
```

### TensorContext

避免频繁的内存分配/释放，专门实现一个内存池来管理：

```cpp
class TensorContext {
private:
    struct MemBlock {
        std::byte* ptr;
        size_t size;
    };
    
    std::list<MemBlock> free_blocks_;   // 空闲内存块
    std::list<MemBlock> use_blocks_;    // 使用中的内存块
    size_t use_size_;                   // 总使用量
    
public:
    std::byte* NewMem(size_t size) {
        size = GetSize(size);  // 16字节对齐
        
        // 最佳适应算法：找最合适的空闲块
        auto best_fit = free_blocks_.end();
        size_t min_size_diff = std::numeric_limits<size_t>::max();
        
        for (auto it = free_blocks_.begin(); it != free_blocks_.end(); ++it) {
            if (it->size >= size) {
                size_t size_diff = it->size - size;
                if (size_diff < min_size_diff) {
                    min_size_diff = size_diff;
                    best_fit = it;
                }
            }
        }
        
        std::byte* ptr = nullptr;
        if (best_fit != free_blocks_.end()) {
            // 重用空闲块
            ptr = best_fit->ptr;
            use_blocks_.push_back({ptr, size});
            free_blocks_.erase(best_fit);
        } else {
            // 分配新内存
            ptr = new std::byte[size];
            if (ptr) {
                use_blocks_.push_back({ptr, size});
            }
        }
        
        if (ptr) {
            use_size_ += size;
        }
        return ptr;
    }
    
    void DeleteMem(std::byte* ptr) {
        // 查找并释放内存块
        for (auto it = use_blocks_.begin(); it != use_blocks_.end(); ++it) {
            if (it->ptr == ptr) {
                size_t size = it->size;
                use_size_ -= size;
                
                // 根据空闲块数量决定是否回收
                if (free_blocks_.size() > MaxFreeBlocks) {
                    delete[] ptr;  // 直接释放
                } else {
                    free_blocks_.push_back(*it);  // 加入空闲池
                }
                
                use_blocks_.erase(it);
                return;
            }
        }
    }
};
```

**内存对齐：**

```cpp
static size_t GetSize(size_t size, size_t alignment = 16) {
    return (size + alignment - 1) & ~(alignment - 1);
}
```

**内存池的好处**：
- **减少系统调用**：避免频繁的`malloc`/`free`
- **减少内存碎片**：16字节对齐和最佳适应算法
- **提高缓存效率**：对齐的内存访问更快
- **自动管理**：限制空闲块数量防止内存过度占用


### TensorOption

通过TensorOption,能指定Tensor的数据类型，是否需要梯度：

```cpp
class TensorOption {
public:
    TensorOption& SetDataType(DataType t) {
        datatype_ = t;
        return *this;
    }
    
    TensorOption& RequiresGrad(bool r) {
        requires_grad_ = r;
        return *this;
    }
    
    DataType GetDataType() const { return datatype_; }
    bool RequiresGrad() const { return requires_grad_; }
    
private:
    DataType datatype_ = F32;
    bool requires_grad_ = false;
};
```

举例：
```cpp
// 创建需要梯度的int32张量
Tensor a({10, 20}, TensorOption()
                   .SetDataType(DataType::I32)
                   .RequiresGrad(true));
```

### Tensor

接下来是具体的Tensor类：

```cpp
class Tensor {
private:
    std::shared_ptr<TensorNode> node_;    // 共享内存节点
    TensorMeta meta_;                     // 元数据
    std::unique_ptr<Back> back_;          // 自动微分信息
    
public:
    // 构造函数
    Tensor(const std::vector<int64_t>& sizes, const TensorOption& option = {})
        : meta_(sizes, option.GetDataType()) {
        auto ctx = DefaultTensorContext::Get();
        node_ = std::make_shared<TensorNode>(ctx, meta_.NumBytes());
        
        if (option.RequiresGrad()) {
            RequiresGrad(true);
        }
    }
    
    // 基本属性访问
    const std::vector<int64_t>& Sizes() const { return meta_.sizes; }
    const std::vector<int64_t>& Strides() const { return meta_.strides; }
    DataType GetDataType() const { return meta_.datatype; }
    size_t NumElements() const { return meta_.NumElements(); }
    
    // 数据访问
    template<typename T>
    T* Data() { return node_->Data<T>(); }
    
    template<typename T>
    const T* Data() const { return node_->Data<T>(); }
};
```

类似`grad_node_`是`shared_ptr`的问题：
**为什么node_是shared_ptr，而back_是unique_ptr？**

这个设计体现了不同的共享语义：

1. **node_(shared_ptr)**：多个Tensor可以共享同一块数据
   ```cpp
   Tensor a = Tensor::Randn({2, 3});
   Tensor b = a.T();        // b和a共享数据
   Tensor c = a.View({6});  // c也和a共享数据
   ```

2. **back_(unique_ptr)**：每个Tensor的计算图位置是唯一的
   ```cpp
   Tensor a = Tensor::Randn({2, 3});
   Tensor b = a.T();
   // a和b虽然共享数据，但在计算图中是不同的节点
   // a.back_记录a的创建历史，b.back_记录转置操作
   ```

## TensorOps

### View

View操作允许改变Tensor的形状而不复制数据：

```cpp
Tensor Tensor::View(const std::vector<int64_t>& new_sizes) const {
    // 1. 检查元素总数是否匹配
    int64_t new_num_elements = 1;
    for (int64_t s : new_sizes) {
        new_num_elements *= s;
    }
    if (new_num_elements != NumElements()) {
        throw std::runtime_error("View size is invalid");
    }
    
    // 2. 创建新的Tensor，共享底层数据
    auto result = *this;  // 拷贝构造，共享node_
    result.meta_.sizes = new_sizes;
    result.meta_.InitStrides();  // 重新计算步长
    return result;
}
```

**示例说明**：
```cpp
Tensor a = Tensor::Randn({2, 6});     // 原始：2×6 = 12个元素
Tensor b = a.View({3, 4});            // 重塑：3×4 = 12个元素
Tensor c = a.View({12});              // 展平：12个元素

// 验证内存共享
assert(a.Data<float>() == b.Data<float>());  // 相同的内存地址
assert(a.Data<float>() == c.Data<float>());  

// 修改任何一个会影响所有
a.Data<float>()[0] = 999.0f;
// 现在 b 和 c 的第一个元素也变成了 999.0f
```

**View的限制**：只能在连续内存上进行。

### Transpose

转置操作通过交换维度来改变数据的访问模式：

```cpp
Tensor Tensor::Transpose(int64_t dim0, int64_t dim1) const {
    // 处理负索引
    if (dim0 < 0) dim0 += meta_.sizes.size();
    if (dim1 < 0) dim1 += meta_.sizes.size();
    
    auto result = *this;
    // 关键：交换对应维度的大小和步长
    std::swap(result.meta_.sizes[dim0], result.meta_.sizes[dim1]);
    std::swap(result.meta_.strides[dim0], result.meta_.strides[dim1]);
    
    return result;
}

// 矩阵转置的便捷接口
Tensor Tensor::T() const {
    if (meta_.sizes.size() != 2) {
        throw std::runtime_error("T() only supports 2D tensors");
    }
    return Transpose(0, 1);
}
```

**转置的数学原理**：

对于矩阵$A \in \mathbb{R}^{M \times N}$，转置$A^T \in \mathbb{R}^{N \times M}$满足$(A^T)_{ij} = A_{ji}$。

在内存中，原始矩阵按行存储：$[A_{00}, A_{01}, ..., A_{0N}, A_{10}, A_{11}, ..., A_{MN}]$

转置后，我们希望$A^T_{ij} = A_{ji}$，通过改变步长实现：
- 原始：`strides = [N, 1]`，访问$A_{ij}$的索引为$i \times N + j$
- 转置：`strides = [1, N]`，访问$A^T_{ij}$的索引为$i \times 1 + j \times N = i + jN = jN + i$

**内存共享的验证：**

```cpp
// 验证零拷贝操作
Tensor a = Tensor::Randn({2, 6});
Tensor b = a.View({3, 4});
Tensor c = a.T();

std::cout << "a和b共享内存: " << (a.Data<float>() == b.Data<float>()) << std::endl;  // true
std::cout << "a和c共享内存: " << (a.Data<float>() == c.Data<float>()) << std::endl;  // true

// 修改a会影响b和c
a.Data<float>()[0] = 999.0f;
std::cout << "b[0,0] = " << b.Data<float>()[0] << std::endl;  // 999.0
```

View/Transpose/T这些操作只改变meta数据，和原Tensor共享node数据，也被称为零拷贝操作，这些操作还有个特点是没有Backward过程。

### Broadcast

广播(Broadcast)是NumPy引入的重要概念，允许不同形状的数组进行运算。理解广播对于高效的深度学习代码至关重要。

#### 广播规则

广播遵循以下规则：

1. **从右往左比较维度**
2. **兼容条件**：两个维度兼容当且仅当：
   - 它们相等，或者
   - 其中一个为1，或者
   - 其中一个不存在（视为1）

```cpp
// 广播示例
A: (2, 3, 4)
B: (   3, 1)  → 广播为 (2, 3, 4)

A: (2, 1, 4)  
B: (   3, 4)  → 广播为 (2, 3, 4)

A: (2, 3, 4)
B: (2, 4, 5)  → 无法广播（最后一维4 vs 5不兼容）
```

#### 兼容性检查

广播操作之前，需要先做广播兼容性检查：

```cpp
bool CanBroadcast(const Tensor& from, const std::vector<int64_t>& to_sizes) {
    const auto& from_sizes = from.Sizes();
    
    // 目标维度不能少于源维度
    if (from_sizes.size() > to_sizes.size()) {
        return false;
    }
    
    // 从右往左检查每个维度
    for (size_t i = 0; i < from_sizes.size(); i++) {
        size_t from_dim = from_sizes[from_sizes.size()-1-i];
        size_t to_dim = to_sizes[to_sizes.size()-1-i];
        
        // 维度必须相等或源维度为1
        if (from_dim != to_dim && from_dim != 1) {
            return false;
        }
    }
    
    return true;
}
```

#### 具体实现

广播的关键是避免实际复制数据。可以通过巧妙的索引计算实现"虚拟复制"：

```cpp
void Broadcast(const Tensor& input, Tensor& output) {
    const auto& in_sizes = input.Sizes();
    const auto& out_sizes = output.Sizes();
    const auto& in_strides = input.Strides();
    
    float* in_data = input.Data<float>();
    float* out_data = output.Data<float>();
    
    #pragma omp parallel for
    for (size_t out_idx = 0; out_idx < output.NumElements(); ++out_idx) {
        // 1. 将线性索引转换为多维索引
        std::vector<int64_t> out_indices = UnravelIndex(out_idx, out_sizes);
        
        // 2. 映射到输入的索引
        size_t in_idx = 0;
        for (size_t i = 0; i < in_sizes.size(); ++i) {
            size_t out_dim_idx = out_indices[out_sizes.size() - in_sizes.size() + i];
            // 关键：如果输入维度为1，索引始终为0
            size_t in_dim_idx = (in_sizes[i] == 1) ? 0 : out_dim_idx;
            in_idx += in_dim_idx * in_strides[i];
        }
        
        out_data[out_idx] = in_data[in_idx];
    }
}
```

**关键点：**
- **大小为1的维度**：索引始终为0，实现"复制"效果
- **缺失的维度**：视为大小1的维度处理
- **相同大小的维度**：正常索引映射

通过巧妙的索引计算，让大小为1的维度被"无限复制"，而不占用额外内存。

```cpp
// 示例：将 (3,) 广播到 (2, 3)
// 输入：[10, 20, 30]
// 输出：[[10, 20, 30],
//       [10, 20, 30]]

// 输入形状：(3,)，步长：(1,)
// 输出形状：(2, 3)
// 
// 对于输出位置 (i, j)：
// - j 映射到输入的索引 j
// - i 被忽略（因为输入没有这个维度）
// 
// 所以 output[0][j] = input[j]，output[1][j] = input[j]
// 实现了"复制"效果，但没有真正复制数据
```

#### 反向过程

不同于前面的View/Transpose/T操作，Broadcast需要专门的反向处理。即如果前向传播时张量被广播，反向传播时需要将梯度"收缩"回原始形状：

```cpp
// 前向：a(1, 3) + b(2, 3) → c(2, 3)
// 反向：grad_c(2, 3) → grad_a(1, 3), grad_b(2, 3)

Tensor SumToSize(const Tensor& input, const std::vector<int64_t>& target_sizes) {
    const auto& input_sizes = input.Sizes();
    
    if (input_sizes == target_sizes) {
        return input;  // 无需收缩
    }
    
    Tensor result = input;
    
    // 1. 处理维度数不匹配：在前面添加维度进行求和
    while (result.Sizes().size() > target_sizes.size()) {
        result = result.Sum(0, false);  // 沿第0维求和并删除该维度
    }
    
    // 2. 处理大小为1的维度：沿该维度求和但保持维度
    for (size_t i = 0; i < target_sizes.size(); ++i) {
        if (target_sizes[i] == 1 && result.Sizes()[i] > 1) {
            result = result.Sum(i, true);  // 沿第i维求和但保持维度
        }
    }
    
    return result;
}
```

**广播反向传播的数学原理**：

如果前向传播时$a$被广播$n$次到$c$的某个位置，那么反向传播时$c$在这些位置的梯度都应该累加到$a$：

$$\frac{\partial L}{\partial a} = \sum_{i=1}^{n} \frac{\partial L}{\partial c_i}$$

### TensorOp

除View/Transpose操作外，每个张量操作都被抽象为一个TensorOp对象，这样设计的好处：
- **统一接口**：所有操作都有相同的调用方式
- **自动微分**：每个操作都知道如何计算梯度
- **可扩展性**：添加新操作很容易
- **优化机会**：可以对操作进行各种优化

```cpp
class TensorOp {
public:
    virtual ~TensorOp() = default;
    
    Tensor Process(const std::vector<Tensor>& inputs, bool no_grad = false) const {
        // 1. 计算前向结果
        Tensor result = Process_(inputs);

        // 2. 如果不需要梯度，直接返回
        if (no_grad) return result;
        
        // 3. 检查是否需要构建计算图
        bool need_grad = std::any_of(inputs.begin(), inputs.end(),
                            [](const Tensor& t) { return t.RequiresGrad(); });
        
        // 4. 构建计算图
        if (need_grad) {
            result.RequiresGrad(true);
            result.back_->op = this;
            result.back_->inputs = inputs;
        }
        
        return result;
    }
    
    Tensor operator()(const std::vector<Tensor>& inputs, bool no_grad = false) const {
        return Process(inputs, no_grad);
    }
    
protected:
    // 子类需要实现的接口
    virtual Tensor Process_(const std::vector<Tensor>& inputs) const = 0;
    virtual std::vector<Tensor> Backward_(
        const Tensor& grad_output,
        const std::vector<Tensor>& inputs) const = 0;
};
```

注意：计算图的构建，分为两步，前向是给back_赋值，反向是给grad_node_赋值。

```cpp
    Tensor& RequiresGrad(bool requires_grad) {
        if (requires_grad) {
            if (!back_) {
                back_ = std::make_unique<Back>();
            }
        }
        return *this;
    }

```

### BinaryOps

全部二元操作（加法/乘法等）都遵循相同的流程：


```cpp
class OpAdd : public TensorOp {
protected:
    Tensor Process_(const std::vector<Tensor>& inputs) const override {
        const auto& a = inputs[0];
        const auto& b = inputs[1];
        
        // 情况1：形状完全相同，直接计算
        if (CanBinaryOp(a, b)) {
            Tensor out(a.Sizes());
            ops::add(a, b, out);
            return out;
        }
        
        // 情况2：需要广播
        if (CanBroadcast(a, b.Sizes())) {
            return Process_({OpBroadcastTo(a, b.Sizes()), b});
        } else if (CanBroadcast(b, a.Sizes())) {
            return Process_({a, OpBroadcastTo(b, a.Sizes())});
        } else {
            throw std::runtime_error("Tensors cannot be broadcast");
        }
    }
    
    std::vector<Tensor> Backward_(
        const Tensor& grad_output,
        const std::vector<Tensor>& inputs) const override {
        const auto& a = inputs[0];
        const auto& b = inputs[1];
        
        if (CanBinaryOp(a, b)) {
            // 形状相同，梯度直接传递
            return {grad_output, grad_output};
        }
        
        // 处理广播情况：需要将梯度收缩回原始形状
        if (CanBroadcast(a, b.Sizes())) {
            auto grad_inputs = Backward_(grad_output, {OpBroadcastTo(a, b.Sizes()), b});
            auto grad_a = OpSumToSize(grad_inputs[0], a.Sizes());
            return {grad_a, grad_inputs[1]};
        }
        // ... 其他情况
    }
};
```

注意广播的处理，前向BrodcastTo的，反向还有SumToSize回来。以及强调下除了Functor和Process接口，其它地方的Op调用都是不建设计算图的。

### MatMul

矩阵乘法是深度学习中最重要的操作，几乎所有的神经网络层都涉及矩阵乘法。我们需要支持：

1. **基础矩阵乘法**：$(M, K) \times (K, N) → (M, N)$
2. **批量矩阵乘法**：$(B, M, K) \times (B, K, N) → (B, M, N)$
3. **广播批量矩阵乘法**：$(B_1, M, K) \times (B_2, K, N) → (\text{broadcast}(B_1, B_2), M, N)$

#### 兼容性检查


```cpp
bool CanMatMul(const Tensor& a, const Tensor& b) {
    if (a.GetDataType() != b.GetDataType()) {
        return false;
    }
    
    const auto& a_sizes = a.Sizes();
    const auto& b_sizes = b.Sizes();
    
    // 至少是2D
    if (a_sizes.size() < 2 || b_sizes.size() < 2) {
        return false;
    }
    
    // 关键约束：A的最后一维 = B的倒数第二维
    if (a_sizes.back() != b_sizes[b_sizes.size()-2]) {
        return false;
    }
    
    return true;
}

std::vector<int64_t> GetBroadcastMatMulSizes(
    const std::vector<int64_t>& a_sizes,
    const std::vector<int64_t>& b_sizes) {
    
    // 提取批次维度（除了最后两维）
    std::vector<int64_t> a_batch(a_sizes.begin(), a_sizes.end()-2);
    std::vector<int64_t> b_batch(b_sizes.begin(), b_sizes.end()-2);
    
    // 广播批次维度
    std::vector<int64_t> output_shape;
    size_t max_batch_dims = std::max(a_batch.size(), b_batch.size());
    
    for (size_t i = 0; i < max_batch_dims; i++) {
        int64_t a_dim = i < a_batch.size() ? a_batch[i] : 1;
        int64_t b_dim = i < b_batch.size() ? b_batch[i] : 1;
        
        if (a_dim != b_dim && a_dim != 1 && b_dim != 1) {
            return {};  // 无法广播
        }
        output_shape.push_back(std::max(a_dim, b_dim));
    }
    
    // 添加矩阵维度
    output_shape.push_back(a_sizes[a_sizes.size()-2]);  // M
    output_shape.push_back(b_sizes[b_sizes.size()-1]);  // N
    
    return output_shape;
}
```

#### 具体实现

```cpp
class OpMatMul : public TensorOp {
protected:
    Tensor OpValue_(const std::vector<Tensor>& inputs) const override {
        const auto& a = inputs[0];
        const auto& b = inputs[1];
        
        if (!CanMatMul(a, b)) {
            throw std::runtime_error("Tensors cannot be matrix multiplied");
        }
        
        // 计算输出形状
        auto output_sizes = GetBroadcastMatMulSizes(a.Sizes(), b.Sizes());
        Tensor out(output_sizes);
        
        // 调用具体实现
        ops::matmul(a, b, out);
        return out;
    }
    
    std::vector<Tensor> Backward_(
        const Tensor& grad_output,
        const std::vector<Tensor>& inputs) const override {
        const auto& a = inputs[0];
        const auto& b = inputs[1];
        
        // 矩阵乘法的梯度公式：
        // ∂L/∂A = ∂L/∂C × B^T
        // ∂L/∂B = A^T × ∂L/∂C
        Tensor grad_a = grad_output.MatMul(b.T());
        Tensor grad_b = a.T().MatMul(grad_output);
        
        // 处理广播情况：如果输入被广播，需要将梯度求和回原始形状
        if (grad_a.Sizes() != a.Sizes()) {
            grad_a = OpSumToSize(grad_a, a.Sizes());
        }
        if (grad_b.Sizes() != b.Sizes()) {
            grad_b = OpSumToSize(grad_b, b.Sizes());
        }
        
        return {grad_a, grad_b};
    }
};
```


```cpp
template<typename T>
void matmul_impl(const Tensor& a, const Tensor& b, Tensor& out) {
    const auto& a_sizes = a.Sizes();
    const auto& b_sizes = b.Sizes();
    
    // 提取矩阵维度
    int64_t M = a_sizes[a_sizes.size()-2];
    int64_t K = a_sizes[a_sizes.size()-1];
    int64_t N = b_sizes[b_sizes.size()-1];
    
    // 计算批次数量
    int64_t batch_size = 1;
    for (size_t i = 0; i < a_sizes.size()-2; ++i) {
        batch_size *= a_sizes[i];
    }
    
    const T* a_data = a.Data<T>();
    const T* b_data = b.Data<T>();
    T* out_data = out.Data<T>();
    
    // 批量矩阵乘法
    #pragma omp parallel for
    for (int64_t batch = 0; batch < batch_size; ++batch) {
        const T* a_batch = a_data + batch * M * K;
        const T* b_batch = b_data + batch * K * N;
        T* out_batch = out_data + batch * M * N;
        
        // 标准的三重循环矩阵乘法
        for (int64_t m = 0; m < M; ++m) {
            for (int64_t n = 0; n < N; ++n) {
                T sum = 0;
                for (int64_t k = 0; k < K; ++k) {
                    sum += a_batch[m * K + k] * b_batch[k * N + n];
                }
                out_batch[m * N + n] = sum;
            }
        }
    }
}

void matmul(const Tensor& a, const Tensor& b, Tensor& out) {
    if (a.GetDataType() == DataType::F32) {
        matmul_impl<float>(a, b, out);
    } else if (a.GetDataType() == DataType::I32) {
        matmul_impl<int32_t>(a, b, out);
    } else {
        throw std::runtime_error("Unsupported data type for matmul");
    }
}
```

**示例**：
```cpp
// 基本矩阵乘法
Tensor A({2, 3});  // 2x3 矩阵
Tensor B({3, 4});  // 3x4 矩阵
Tensor C = A.MatMul(B);  // 结果：2x4

// 批量矩阵乘法
Tensor A_batch({5, 2, 3});  // 5个 2x3 矩阵
Tensor B_batch({5, 3, 4});  // 5个 3x4 矩阵
Tensor C_batch = A_batch.MatMul(B_batch);  // 结果：5个 2x4 矩阵

// 广播批量矩阵乘法
Tensor A_single({2, 3});     // 单个 2x3 矩阵
Tensor B_multi({5, 3, 4});   // 5个 3x4 矩阵
Tensor C_result = A_single.MatMul(B_multi);  // A被广播：结果是5个 2x4 矩阵
```

### OpRegistry

操作注册和缓存：为了避免重复创建相同的操作对象，我们使用注册表模式：

```cpp
class OpRegistry {
private:
    using OpCreator = std::function<std::unique_ptr<TensorOp>(const OpOption&)>;
    
    std::unordered_map<std::string, std::unique_ptr<TensorOp>> op_instances_;
    std::unordered_map<std::string, OpCreator> creators_;
    
    OpRegistry() {
        // 注册基本操作
        Register("add", [](const OpOption&) { return std::make_unique<OpAdd>(); });
        Register("mul", [](const OpOption&) { return std::make_unique<OpMul>(); });
        Register("matmul", [](const OpOption&) { return std::make_unique<OpMatMul>(); });
        
        // 带参数的操作
        Register("add_scalar", [](const OpOption& args) {
            return std::make_unique<OpAddScalar>(args.scalar);
        });
        Register("mean", [](const OpOption& args) {
            return std::make_unique<OpMean>(args.dim, args.keepdim);
        });
    }
    
public:
    static OpRegistry& Instance() {
        static OpRegistry registry;
        return registry;
    }

    void Register(const std::string& name, OpCreator creator) {
        creators_[name] = std::move(creator);
    }

    
    const TensorOp* Create(const std::string& name, 
                          const OpOption& args = OpOption()) {
        auto key = GetOpKey(name, args);
        
        // 检查缓存
        auto it = op_instances_.find(key);
        if (it != op_instances_.end()) {
            return it->second.get();
        }
        
        // 创建新实例
        auto it_creator = creators_.find(name);
        if (it_creator == creators_.end()) {
            throw std::runtime_error("Op not found: " + name);
        }
        
        auto op = it_creator->second(args);
        const TensorOp* op_ptr = op.get();
        op_instances_[key] = std::move(op);
        
        return op_ptr;
    }
};
```

**使用示例**：

```cpp
// 在Tensor类中的使用
Tensor Tensor::operator+(const Tensor& other) const {
    auto op = OpRegistry::Instance().Create("add");
    return (*op)({*this, other});
}

Tensor Tensor::operator+(float scalar) const {
    auto op = OpRegistry::Instance().Create("add_scalar",
                OpRegistry::OpOption().SetScalar(scalar));
    return (*op)({*this});
}
```

**关键点：两层注册表**
- 第一层：creators_，存储的是Op创建方法，支持带参数的Op初始化
- 第二层：op_instances_，存储的是具体的op，同参数的Op能共享

TODO：该把scalar等Op实现换一种方式


## 自动微分

再回过来看下多维数组的自动微分。再回顾下这句话：
计算图的构建，分为两步，前向是给back_赋值，反向是给grad_node_赋值。

### Back结构

```cpp
struct Back {
    const TensorOp* op = nullptr;           // 产生此张量的操作
    std::vector<Tensor> inputs;             // 输入张量
    
    Back() = default;
    Back(const TensorOp* op_, std::vector<Tensor> inputs_)
        : op(op_), inputs(std::move(inputs_)) {}
};
```

注意，相比前面章节实现，通过技术演化，这里不用智能指针了，又变成了`std::vector<Tensor>`。毕竟这个接口更自然些，更符合NumPy的使用习惯，且内部的数据其实已经用`std::shared_ptr`存储了，不会出现丢数据的问题了，不过也有些代价，就是`meta`需要更多的复制，相比起好处，这个缺点还是能承受的。

### 计算图

```cpp
void Tensor::Backward() {
    // 1. 拓扑排序
    std::vector<Tensor*> topo_order;
    std::unordered_set<const TensorNode*> visited;
    
    std::function<void(Tensor*)> build_topo = [&](Tensor* tensor) {
        if (!tensor || !tensor->node_ || 
             visited.count(tensor->node_.get())) {
            return;
        }
        
        visited.insert(tensor->node_.get());

        if (tensor->back_ && !tensor->back_->inputs.empty()) {
            for (auto& input : tensor->back_->inputs) {
                if (input.RequiresGrad()) {
                    build_topo(&input);
                }
            }
        }
        
        topo_order.push_back(tensor);
    };
    
    build_topo(this);

    // 2. 初始化根节点梯度
    if (!node_->grad_node_) {
        node_->grad_node_ = OnesLike(*this).node_;
    }

    // 3. 反向传播
    for (auto it = topo_order.rbegin(); it != topo_order.rend(); ++it) {
        Tensor* current = *it;
        
        Tensor current_grad(current->node_->grad_node_, current->meta_);
        auto input_grads = current->back_->op->Backward(
            current_grad, 
            current->back_->inputs
        );
        
        for (size_t i = 0; i < current->back_->inputs.size(); ++i) {
            auto& input = current->back_->inputs[i];
            if (!input.node_->grad_node_) {
                input.node_->grad_node_ = input_grads[i].node_;
            } else {
                Tensor grad_tensor(input.node_->grad_node_, input.meta_);
                auto op = OpRegistry::Instance().Create("add"); 
                auto grad_sum = (*op)({grad_tensor, input_grads[i]}, true);
                // 梯度累计
                input.node_->grad_node_ = grad_sum.node_;
            }
        }
    }
}
```

## 问题

**为什么View/Transpose没有Backward过程？**

核心原因：它们是零拷贝的元数据操作

View/Transpose 等操作的特殊性在于：**它们不改变底层数据，只改变数据的解释方式**。

1. 数学角度：恒等变换

从数学上看，这些操作的雅可比矩阵是单位矩阵：

```cpp
// 对于 y = x.view(new_shape)
// 雅可比矩阵 ∂y/∂x = I (单位矩阵)
// 根据链式法则：∂L/∂x = ∂L/∂y · ∂y/∂x = ∂L/∂y · I = ∂L/∂y
```

**含义**：梯度在数值上完全相同，只是形状不同。

2. 实现角度：共享存储

```cpp
Tensor a = Tensor::Randn({2, 3}).RequiresGrad(true);
Tensor b = a.View({6});  // b 和 a 共享底层数据和梯度存储

// 关键设计：
// - 共享相同的 node_（底层数据和梯度）
// - 不同的 meta_（形状、步长等解释信息）
```

3. 梯度获取的"隐式反向传播"

```cpp
Tensor Tensor::Grad() {
    // 使用当前 Tensor 的 meta_ 来解释共享的梯度数据
    return Tensor(node_->grad_node_, meta_);
}

// a.Grad() 返回形状 {2, 3} 的梯度视图
// b.Grad() 返回形状 {6} 的梯度视图  
// 但指向相同的底层梯度数据
```

4. 本质：隐式的元数据反向传播

实际上，View/Transpose 确实有"反向传播"，只是被优化成了隐式实现：

- **前向**：`Meta_original → Meta_transformed`（改变形状解释）
- **反向**：`Meta_transformed → Meta_original`（在 Grad() 中恢复原始形状）

总结

View/Transpose 不需要显式的 Backward 操作是因为：

1. **数学透明性**：雅可比矩阵为单位矩阵，梯度传播是恒等变换
2. **共享存储**：底层数据和梯度存储完全共享
3. **隐式处理**：通过不同的元数据自动处理形状变换
4. **效率优化**：避免不必要的计算图节点和数据复制
