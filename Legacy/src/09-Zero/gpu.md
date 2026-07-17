# MxArray

## CUDA编程基础

### 函数基础

#### 函数声明

在 CUDA 中，`__global__` 是用于声明 GPU Kernel 函数的关键字。这类函数具有以下特点：
- 在 GPU 上执行
- 由 CPU 调用（称为 Host 调用 Device）
- 使用特殊的调用语法 `<<<...>>>`

```cpp
// Kernel 函数声明
__global__ void my_kernel(int* data, int size) {
    // GPU 执行的代码
}
```

#### 调用语法

调用Kernel函数需要使用特殊的三重尖括号语法 `<<<gridDim, blockDim>>>`：

```cpp
my_kernel<<<gridDim, blockDim>>>(device_data, size);
```
其中：
- gridDim：定义网格（Grid）的维度，通常为 `dim3` 类型
- blockDim：定义线程快(Block)的维度，通常为 `dim3` 类型

`dim3` 可以看成是：

```cpp
struct dim3 {
    unsigned int x, y, z;
    // 构造函数
    dim3(unsigned int x = 1, unsigned int y = 1, unsigned int z = 1);
};
```



### 核心概念

CUDA (Compute Unified Device Architecture) 是 Nvidia 为 GPU 并行计算设计的编程模型。理解 CUDA 编程需要掌握几个核心概念：


#### 1. 线程层次结构

CUDA 采用层次化的线程组织方式：

```
Grid（网格）
├── Block 0
│   ├── Thread 0
│   ├── Thread 1
│   └── ...
├── Block 1
│   ├── Thread 0
│   ├── Thread 1
│   └── ...
└── ...
```


**具体例子**：
```cpp
// 处理一个2048x2048的矩阵
dim3 blockSize(16, 16)； // 每个block有16x16=256个线程
dim3 gridSize(128, 128); // 需要128x128=16384个block

// 每个线程处理一个矩阵元素 128x16=2048
__global__ void process_matrix(float* data, int width, int height) {
    int x = blockIdx.x * blockDim.x + threadIdx.x; // 计算全局x坐标
    int y = blockIdx.y * blockDim.y + threadIDx.y; // 计算全局y坐标

    if (x < width && y < height) {
        int idx = y * width + x;
        data[idx] = data[idx] * 2.0; // 简单的元素操作
    }
}

// 调用：process_matrix<<<gridSize, blockSize>>>(data, w, h);
```

说明：Kernei函数中可以使用以下内置变量：
| 变量 | 描述 |
|------|------|
| `gridDim` | Grid 的维度 |
| `blockIdx` | 当前 Block 在 Grid 中的索引 |
| `blockDim` | Block 的维度 |
| `threadIdx` | 当前线程在 Block 中的索引 |


#### 2. 内存层次结构

GPU 有复杂的内存层次结构，不同内存的访问速度差异巨大：

```
内存类型          大小        延迟         带宽        用途
Global Memory     几十GB      200-400周期   ~1TB/s     主要数据存储
Shared Memory     几十KB      1-2周期       ~8TB/s     block内共享
Registers        几十KB      1周期         ~20TB/s    线程私有变量
Constant Memory   64KB       1-2周期       ~1TB/s     只读常量
```

**内存使用例子**：
```
__global__ void optimized_kernel(float* global_data, int size) {
    // 1. 共享内存：block内线程共享
    __shared__ float shared_cache[256];

    // 2. 寄存器：线程私有
    int tid = threadIdx.x;
    int gid = blockIdx.x * blockDim.x + threadIdx.x;

    // 3. 全局内存到共享内存
    if (tid < 256 && gid < size) {
        shared_cache[tid] = global_data[gid];
    }

    __syncthreads(); // 同步所有线程

    // 4. 使用共享内存进行快速计算
    if (tid < 256 && gid < size) {
        float result = shared_cache[tid] * 2.0f;
        global_data[gid] = result;
    }
}
```

#### 3. 执行模型

CUDA的执行模型基于SIMT（Single Instruction, Multiple Thread）：

```cpp
// Wrap: GPU执行的基本单位（32个线程）
__global__ void simt_example(float* data, int size) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;

    if (idx < size) {
        // 所有32个线程（一个Wrap）同时执行相同指令
        data[idx] = sinf(data[idx]); // 32个sin计算并行执行
    }
}

```

**关键点**：
- **Warp**：32个线程为一组，同时执行相同指令，仅idx不同

### 编程模式

#### 模式1：并行

最简单的情况是每个线程独立处理一个数据元素：

```cpp
// 向量加法：C[i] = A[i] + B[i]
__global__ void vector_add(float* A, float* B, float* C, int N) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x；
    if (idx < N) {
        C[idx] = A[idx] + B[idx]; // 完全独立的计算
    }
}

void launch_vector_add(float* A, float* B, float* C, int N) {
    int blockSize = 256;
    int gridSize = (N + blockSize - 1)/blockSize;
    vector_add<<<gridSize, blockSize>>>(A, B, C, N);
}
```

#### 模式2：归约

将大量数据归约为单个值的模式：

```cpp
// 数组求和：使用共享内存优化
__global__ void reduce_sum(float* input, float* output, int N) {
    __shared__ float sdata[256];

    int tid = threadIdx.x;
    int idx = blockIdx.x * blockDim.x + threadIdx.x;

    // 1. 数据加载到共享内存
    sdata[tid] = (idx < N>) ? input[idx] : 0;
    __syncthreads();

    // 2. 归约过程
    for (int s = blockDim.x / 2; s > 0; s >> = 1) {
        if (tid < s) {
            sdata[tid] += sdata[tid + s];
        }
        __syncthreads();
    }

    // 3. 第一个线程写回结果
    if (tid == 0) {
        output[blockIdx.x] = sdata[0];
    }
}

void launch_reduce_sum(float* input, float* output, int N) {
    int blockSize = 256;  // 每个 block 256 个线程
    int gridSize = (N + blockSize - 1) / blockSize; // 计算需要的 block 数量
    
    reduce_sum<<<gridSize, blockSize>>>(input, output, N);
    // 通常还需要第二次归约（例如在 CPU 上）：
    // float total_sum = 0;
    // for (int i = 0; i < gridSize; i++) total_sum += output[i];
}
```

- Block内会把256个数放到共享内存，通过归约方式求和（注意tid<s，折半一次就减少一半线程参与计算）
- 还需要把Grid内的全部Block的局部和再累加，才会得到最终的和（也就是代码注释部分）

#### 模式3：卷积

每个输出依赖于输入的一个邻域：

每个输出依赖于输入的一个邻域：

```cpp
// 2D卷积：每个输出像素依赖于输入的3x3邻域
__global__ void conv2d_kernel(float* input, float* kernel, float* output,
                              int height, int width) {
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    int y = blockIdx.y * blockDim.y + threadIdx.y;
    
    if (x < width && y < height) {
        float sum = 0;
        for (int ky = -1; ky <= 1; ky++) {
            for (int kx = -1; kx <= 1; kx++) {
                int nx = x + kx;
                int ny = y + ky;
                if (nx >= 0 && nx < width && ny >= 0 && ny < height) {
                    sum += input[ny * width + nx] * kernel[(ky+1) * 3 + (kx+1)];
                }
            }
        }
        output[y * width + x] = sum;
    }
}
```

### 内存管理

涉及到内存分配/数据传输以及清理内存，以下通过示例代码来阐述：

```cpp
// CUDA内存管理的基本模式
void cuda_memory_example() {
    int N = 1000000;

    size_t size = N * sizeof(float);

    // 1. Cpu 内存分配
    float* h_data = (float*)malloc(size);

    // 2. Gpu 内存分配
    float* d_data;
    cudaMalloc(&d_data, size);

    // 3. 数据传输：Cpu 到 Gpu
    cudaMemcpy(d_data, h_data, size, cudaMemcpyHostToDevice);

    // 4. 启动kernel
    int blockSize = 256;
    int gridSize = (N + blockSize - 1)/blockSize;
    some_kernel<<<gridSize, blockSize>>>(d_data, N);

    // 5. 数据传输：Gpu 到 Cpu
    cudaMemcpy(h_data, d_data, size, cudaMemcpyDeviceToHost);

    // 6. 清理内存
    free(h_data);
    cudaFree(d_data);
}
```

### 错误检查

CUDA编程中的调试不是很方便（因为不能用Print），因而做好错误检查很关键：

```cpp
// 错误检查宏定义
#define CUDA_CHECK(call) do { \
    cudaError_t err = call; \
    if (err != cudaSuccess) { \
        std::cerr << "CUDA error in " << __FILE__ << ":" << __LINE__ << \
                  ": " << cudaGetErrorString(err) << std::endl; \
        throw std::runtime_error("CUDA error"); \
    } \
} while(0)

// 使用示例
void safe_cuda_code() {
    float* d_data;
    CUDA_CHECK(cudaMalloc(&d_data, 1000 * sizeof(float)));
    
    // kernel启动错误检查
    some_kernel<<<100, 10>>>(d_data, 1000);
    CUDA_CHECK(cudaGetLastError());  // 检查启动错误
    CUDA_CHECK(cudaDeviceSynchronize());  // 检查执行错误
    
    CUDA_CHECK(cudaFree(d_data));
}
```

## Zero + GPU

Zero 引擎引入 GPU 计算，需要做两步，一是对 GPU 做抽象，通过 Device/TensorContext/DeviceTransfer这些类来封装了CPU和GPU之间的差异，提供统一接口，二是Tensor和TensorOp要集成这些Class,使用统一接口完成相应操作，且TensorOp要提供专门的CPU和GPU实现的具体函数。以下做详细介绍。

### Device 类

引入 Device 类作为 CPU/CUDA 设备的抽象，这样的使用的时候可以指定是用Cpu还是Gpu。

```cpp
class Device {
public:
    enum class DeviceType { CPU, CUDA };

    Device(DeviceType t = DeviceType::CPU, int i = -1)
        : t_(t), i_(i) {}
    
    static Device CPU() { return Device(DeviceType::CPU); }
    static Device CUDA(int i = 0) { return Device(DeviceType::CUDA, i); }

    bool IsCpu() const { return t_ == DeviceType::CPU; }
    bool IsCuda() const { return t_ == DeviceType::CUDA; }

    bool operator==(const Device& other) const {
        return t_ == other.t_ && i_ == other.i_;
    }
    
    bool operator!=(const Device& other) const {
        return !(*this == other);
    }

private:
    DeviceType t_;
    int i_; // Gpu可能会大于1个
};
```

### TensorContext 类

不同设备需要不同的内存管理策略。引起通过TensorContext抽象来封装这些差异。

```cpp
class TensorContext {
public:
    virtual ~TensorContext() = default;
    
    virtual std::byte* NewMem(size_t size) = 0;
    virtual void DeleteMem(std::byte* ptr) = 0;
};
```

把之前的TensorContext变成CPUTensorContext。

#### CUDATensorContext

针对GPU内存管理，项目中的CUDATensorContext实现了相应机制，且集成了cuBLAS支持：

```cpp
class CUDATensorContext : public TensorContext {
public:
    explicit CUDATensorContext()
        : use_size_(0) {
        CUBLAS_CHECK(cublasCreate(&cublas_handle_));
    }

    ~CUDATensorContext() override {
        if (cublas_handle_) {
            cublasDestroy(cublas_handle_);
        }
    }

    std::byte* NewMem(size_t size) override {
        void* ptr = nullptr;
        CUDA_CHECK(cudaMalloc(&ptr, size));
        if (ptr) {
            this->use_size_ += size;
        }
        return static_cast<std::byte*>(ptr);
    }

    void DeleteMem(std::byte* ptr) override {
        if (ptr) {
            CUDA_CHECK(cudaFree(ptr));
        }
    }

private:
    size_t use_size_;
    cublasHandle_t cublas_handle_;
};
```

注意这里**集成了cuBLAS句柄**，这使得所有GPU张量都能高效地进行线性代数运算。

#### DefaultTensorTensorContext

现在DefaultTensorContext能按Device类型提供TensorContext。

```cpp
class DefaultTensorContext {
public:
    static TensorContext* Get();
    static TensorContext* Get(Device e);

private:
    static TensorContext* context_cpu_;
    static TensorContext* context_cuda_;
};
```

这个设计的核心思想是**单例模式**：每种设备类型只有一个Context实例，所有该设备上的Tensor都共享相同的内存管理器和资源。

### DeviceTransfer 类

不同设备间的数据传输是深度学习中的常见需求。项目中的DeviceTransfer类实现了这一功能

```cpp
class DeviceTransfer {
public:
    static void Transfer(std::byte* dst, const std::byte* src, 
                        size_t size, Device src_device, Device dst_device) {
        if (src_device.IsCpu() && dst_device.IsCuda()) {
            CUDA_CHECK(cudaMemcpy(dst, src, size, cudaMemcpyHostToDevice));
        } 
        else if (src_device.IsCuda() && dst_device.IsCpu()) {
            CUDA_CHECK(cudaMemcpy(dst, src, size, cudaMemcpyDeviceToHost));
        }
        else if (src_device.IsCpu() && dst_device.IsCpu()) {
            throw std::runtime_error("In the same device.");
        }
        else if (src_device.IsCuda() && dst_device.IsCuda()) {
            throw std::runtime_error("In the same device.");
        }
    }
};
```

注意同设备之间操作会抛出异常，避免了不必要的内存复制。

### Tensor 类

创建Tensor的时候，涉及到内存分配，以及设备转移的时候，需要用到设备统一接口。

```cpp
Tensor Tensor::New(const std::vector<int64_t>& sizes, const TensorOption& option) {
    auto device = option.GetDevice();
    auto datatype = option.GetDataType();
    auto requires_grad = option.RequiresGrad();
    
    // 使用设备对应的Context分配内存
    auto context = DefaultTensorContext::Get(device);
    auto tensor_meta = std::make_shared<TensorMeta>(sizes, datatype, device);
    
    size_t total_bytes = tensor_meta->NumBytes();
    std::byte* data = context->NewMem(total_bytes);
    
    auto tensor_node = std::make_shared<TensorNode>(data, tensor_meta, context);
    return Tensor(tensor_node, requires_grad);
}
```

1. **TensorOption**指定设备类型
2. **DefaultTensorContext::Get()**获取对应的内存管理器
3. **TensorMeta**存储设备信息和尺寸信息
4. **Context->NewMem()**在正确的设备上分配内存

以及DeviceTransfer的应用： Tensor能转移到不同的设备上。

```cpp
Tensor Tensor::To(Device device) const {
    if (this->GetDevice() == device) {
        return *this;  // 已经在目标设备上
    }
    
    auto new_tensor = Tensor::New(this->Sizes(), 
                                 TensorOption().SetDevice(device)
                                              .SetDataType(this->GetDataType()));
    
    DeviceTransfer::Transfer(new_tensor.tensor_node_->data_, 
                           this->tensor_node_->data_,
                           this->NumBytes(),
                           this->GetDevice(), 
                           device);
    return new_tensor;
}
```

以上的封装不涉及到具体Device的代码，不管是CPU还是GPU,都使用相同的接口和流程。

### TensorOp 类

具体来看下`OpAdd`的实现，来说明下一个Op该怎么写，能做到同时集成CPU和GPU计算函数。


```cpp
Tensor OpAdd::OpValue_(const std::vector<Tensor>& inputs) const {
    if (inputs.size() != 2) 
        throw std::runtime_error("AddOp requires 2 inputs");

    const auto& a = inputs[0];
    const auto& b = inputs[1];
    
    if (CanBinaryOp(a, b)) {
        TensorOption option;
        option.SetDevice(a.GetDevice())
              .SetDataType(a.GetDataType());
        
        Tensor result(a.Sizes(), option);

        // 关键的设备分发逻辑
        if (a.GetDevice().IsCuda()) {
            ops::add_cuda(a, b, result);  // GPU实现
        } else {
            ops::add_cpu(a, b, result);   // CPU实现
        }
        
        return result;
    }
    
    // 处理广播情况...
}
```

根据输入的Device信息，Op会调用相应的实现，具体的CUDA实现代码：

```cpp
template<typename T>
__global__ void add_kernel(const T* a, const T* b, T* out, size_t size) {
    size_t idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < size) {
        out[idx] = a[idx] + b[idx];
    }
}
```

```cpp
template<typename T>
void add_impl_cuda(const Tensor& a, const Tensor& b, Tensor& out) {
    auto size = a.NumElements();
    auto a_data = a.Data<T>();
    auto b_data = b.Data<T>();
    auto out_data = out.Data<T>();
    
    const int block_size = 256;
    const int grid_size = (size + block_size - 1) / block_size;
    
    add_kernel<T><<<grid_size, block_size>>>(a_data, b_data, out_data, size);
    CUDA_CHECK(cudaGetLastError());
    CUDA_CHECK(cudaDeviceSynchronize());
}
```

```cpp
void add_cuda(const Tensor& a, const Tensor& b, Tensor& out) {
    switch (a.GetDataType()) {
        case F32:
            add_impl_cuda<float>(a, b, out);
            break;
        case I32:
            add_impl_cuda<int32_t>(a, b, out);
            break;
        default:
            throw std::runtime_error("Unsupported datatype for add.");
    }
}
```

Op实现具有三层架构：
1. **Op层** (`OpAdd`)：负责参数验证/广播处理和设备分发
2. **实现层** `ops::add_cuda`/`ops::add_cpu`）：负责具体的计算实现
3. **Kernel层**（`add_kernel`）：负责底层的并行计算

### 易用性

用户角度看，当前的实现其实用来是很方便的，能一键切换设备：


```cpp
// CPU计算
auto a_cpu = Tensor::New({1000}, TensorOption().SetDevice(Device::CPU()));
auto b_cpu = Tensor::New({1000}, TensorOption().SetDevice(Device::CPU()));
auto c_cpu = a_cpu + b_cpu;  // 自动使用CPU实现

// GPU计算
auto a_gpu = a_cpu.To(Device::CUDA());
auto b_gpu = b_cpu.To(Device::CUDA());
auto c_gpu = a_gpu + b_gpu;  // 自动使用GPU实现
```

用户只需要关心数据在哪个设备上，引擎会自动选择合适的实现。

