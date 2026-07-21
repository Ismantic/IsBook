# GPU 执行

CPU 适合复杂控制流和低延迟任务，GPU 则通过大量计算单元同时处理数据。深度学习中的逐元素运算、归约和矩阵乘法都具有高并行度，因此适合在 GPU 上执行。

Zero 不在模型层区分 CPU 和 GPU。张量记录自己所在的设备，算子再根据设备选择对应 kernel。同一个前向表达式因此可以运行在两种后端上。

## CUDA 执行模型

CUDA kernel 是一个由 CPU 发起、在 GPU 上并行执行的函数：

```cpp
__global__ void scale_kernel(float* data, float scale, size_t size) {
    size_t idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < size) {
        data[idx] *= scale;
    }
}
```

线程按两层结构组织：

```text
Grid
├── Block 0
│   ├── Thread 0
│   ├── Thread 1
│   └── ...
├── Block 1
└── ...
```

Zero 的一维逐元素 kernel 通常使用 `256` 个线程组成一个 Block，Block 数向上取整：

```cpp
const int block = 256;
const int grid = (size + block - 1) / block;
scale_kernel<<<grid, block>>>(data, scale, size);
```

最后一个 Block 可能只有部分线程对应有效元素，因此 kernel 内部必须检查 `idx < size`。GPU 还会以 \\(32\\) 个线程为一个 Warp 调度指令，同一 Warp 中过多的条件分支会降低执行效率。

## 并行模式

**逐元素运算**

如果每个输出元素只依赖相同位置的输入，就可以让每个线程处理一个元素。加法、乘法、GeLU 和梯度缩放都属于这一类：

```cpp
template<typename T>
__global__ void add_kernel(
    const T* a, const T* b, T* out, size_t size) {
    size_t idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < size) out[idx] = a[idx] + b[idx];
}
```

**归约**

均值、方差、Softmax 和梯度范数需要将多个输入合并为更少的输出。归约 kernel 通常让每个线程先计算局部结果，再在 Block 内使用共享内存或 CUB 完成合并。

```text
输入元素
    ↓ 每个线程处理一部分
线程局部结果
    ↓ BlockReduce
Block 结果
    ↓ 必要时再次合并
最终结果
```

广播反向也是归约。多个大张量位置可能映射到同一个小张量元素，Zero 使用原子加法将这些梯度累加到同一位置。

## 设备抽象

`Device` 记录设备类型和设备编号：

```cpp
class Device {
public:
    enum class DeviceType { CPU, CUDA };

    static Device CPU();
    static Device CUDA(int index = 0);

    bool IsCpu() const;
    bool IsCuda() const;
};
```

创建张量时，`TensorOption` 将设备信息传给 `TensorMeta`，再由 `DefaultTensorContext::Get(device)` 选择对应的内存分配器：

```cpp
auto option = TensorOption()
    .SetDevice(Device::CUDA())
    .SetDataType(F32);

Tensor x({1024, 1024}, option);
```

`Tensor::Cuda()` 和 `Tensor::Cpu()` 通过 `Tensor::To` 在设备之间搬运数据。如果梯度已经存在，梯度存储也会一起迁移：

```cpp
auto x = Tensor::Randn({1024});
x.Cuda();
auto y = x * 2.0f;
y.Cpu();
```

CPU 到 CUDA 使用 `cudaMemcpyHostToDevice`，CUDA 到 CPU 使用 `cudaMemcpyDeviceToHost`。同设备之间的迁移由 `Tensor::To` 直接忽略，不进入 `DeviceTransfer`。

## GPU 内存

GPU 上的频繁 `cudaMalloc` 会引入显著的驱动和同步开销。`CUDATensorContext` 因此将申请大小对齐到 \\(512\\) 字节，并按大小缓存已释放的显存块。

```text
NewMem(size)
  → RoundUp(size, 512)
  → 尝试从同尺寸空闲桶取块
  → 未命中时才调用 cudaMalloc

DeleteMem(ptr)
  → 放回对应尺寸的空闲桶
```

分配器同时统计 `live_bytes`、`peak_bytes`、`reserved_bytes` 和缓存命中次数。当训练切换到不同形状的生成阶段时，`TrimFreePool()` 可以释放已无法复用的旧桶。

这部分与前文的张量存储属于同一套机制：`TensorNode` 只负责持有指针，`TensorContext` 决定指针如何分配和复用。

## Kernel 派发

算子通过 `tensor_ops.h` 中的统一入口调用 kernel：

```cpp
inline void add(const Tensor& a, const Tensor& b, Tensor& out) {
    on_gpu(a) ? add_cuda(a, b, out)
              : add_cpu(a, b, out);
}
```

公开的张量接口不包含 CUDA 分支：

```cpp
auto a = Tensor::Ones({1024});
auto b = Tensor::Ones({1024});

auto cpu_result = a + b;

a.Cuda();
b.Cuda();
auto gpu_result = a + b;
```

派发器以第一个输入所在设备为准，因此同一算子的所有输入必须位于同一设备。如果混用 CPU 和 CUDA 张量，算子可能把主机指针传给 CUDA kernel，造成非法内存访问。进入计算前应先显式将输入迁移到相同设备。

广播等高维 kernel 需要获取形状和步长。Zero 将最多八维的定长 `DimsArg` 直接作为 kernel 参数传递，避免每次运算都为元数据额外执行 `cudaMalloc` 和 `cudaMemcpy`。

## 矩阵乘法

逐元素算子适合自行编写 kernel，矩阵乘法则需要更复杂的分块、数据搬运和硬件调度。Zero 使用 cuBLAS 执行 `F32` MatMul：

- 普通矩阵使用 `cublasGemmEx`。
- 批量矩阵使用 `cublasGemmBatchedEx`。
- 线性层使用 cuBLASLt 的 bias epilogue，将矩阵乘法和 bias 加法融合。

cuBLAS 以列优先布局解释矩阵，Zero 的张量默认为行优先。实现通过交换乘数和转置标志，在不拷贝连续数据的前提下完成两种布局之间的对应。

对于内层维度不是单位步长的视图，cuBLAS 无法用一个 leading dimension 表示其布局。Zero 会先将这类输入实体化为连续张量，再交给 cuBLAS。这与前文“算子必须正确处理 stride”的要求一致。

默认情况下，输入、输出和内部计算都使用 FP32。设置 `MX_FAST_MATMUL=1` 后，cuBLAS 可在保持 FP32 输入输出的同时，使用 FP16 Tensor Core 执行内部乘加。这能提高吞吐量，但会引入额外数值误差，因此需要显式开启。

## 异步执行

CUDA kernel 启动默认是异步的。CPU 将 kernel 放入默认 Stream 后即可继续运行，同一 Stream 中的 kernel 会按提交顺序执行。Zero 不在每次 kernel 启动后调用 `cudaDeviceSynchronize()`，否则 CPU 会在每个小算子后等待 GPU，丧失异步执行的优势。

`cudaMemcpy` 在 CPU 和 GPU 之间搬运数据时会形成必要的同步点。因此训练循环应尽量让数据和中间结果停留在 GPU 上，只在需要记录损失或输出结果时复制少量数据回 CPU。

每次 kernel 启动后会调用 `cudaGetLastError()` 检查配置和启动错误。如果需要定位异步执行错误，可以使用：

```bash
CUDA_LAUNCH_BLOCKING=1 ./build/apps/gpu_check
```

该环境变量会强制 kernel 同步执行，便于将报错定位到具体调用，但不应用于性能测试。

## 正确性验证

GPU kernel 能够成功启动，不代表数值一定正确。Zero 将 CPU 实现作为对照，检查前向数值、反向梯度和多步参数更新：

- `gpu_check`：CPU↔CUDA 往返、加法、MatMul、ReLU、Softmax 和交叉熵。
- `gpu_train_check`：最小 GPU 训练循环。
- `gpu_cmp`：对比 CPU 和 GPU 上连续多步 AdamW 的损失与最终参数。

测试不只要覆盖单个 kernel，还需要覆盖 `Transpose + MatMul`、广播反向和优化器这类跨算子路径。这些路径同时涉及形状、步长、设备和梯度，比单纯的逐元素运算更容易暴露错误。

至此，训练引擎已经具备从张量到 GPU 执行的完整链路。最后一篇将用字符级 GPT 把这些组件组合成一个真实的语言模型训练程序。
