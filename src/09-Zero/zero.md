# Zero

## 引言
接下来会详细的阐述深度学习引擎怎么实现自动梯度计算（或者说自动微分）的原理，以及通过一个最小化的C++程序来阐述实现细节。不过，一个完整的深度学习引擎的实现，还会涉及到更多的概念，比如Tensor的广播处理，Gpu上的运算等，不过最核心的还是这个梯度计算过程。

### 链式法则

自动微分的数学基础是链式法则。对复合函数 $f(g(x))$ ，其导数为：

$$\frac{d}{dx}[f(g(x))] = f'(g(x)) \cdot g'(x)$$

更一般地，对多层复合函数 $f_n(f_{n-1}(...f(_1(x)...)))$ ，链式法则表明：

$$\frac{df_n}{dx} = \frac{df_n}{df_{n-1}} \cdot \frac{df_{n-1}}{df_{n-2}}
                    \cdot ... \cdot \frac{df_1}{dx}$$

这意味着复杂函数的导数可以通过一系列简单函数导数的乘积来计算。每个简单函数的导数我们都知道怎么计算，那么整个复合函数的导数就变得可计算了。

**具体例子：**

通过一个具体例子来理解链式法则： $f(x) = \sin(x^2 + 1)$

这是一个复合函数，我们可以将它分解为：
- $u(x) = x^2 + 1$ （内层函数）
- $f(u) = \sin(u)$ (外层函数)

因此，$f(x) = \sin(u(x))$

第一步：计算各部分的导数
- $$\frac{du}{dx} = \frac{d}{dx}{x^2+1} = 2x$$
- $$\frac{df}{du} = \frac{d}{du}\sin(u) = \cos(u) = \cos(x^2+1)$$

第二步：应用链式法则

$$\frac{df}{dx} = \frac{df}{du} \cdot \frac{du}{dx}
               = \cos(x^2 + 1) \cdot 2x = 2x\cos(x^2+1)$$


验证具体点的计算：

在 $x = 1$ 处：
- $$u(1) = 1^2 + 1 = 2$$
- $$f(1) = \sin(2) \approx 0.909$$
- $$\frac{du}{dx}\big|_{x=1} = 2 \cdot 1 = 2$$
- $$\frac{df}{du}\big|_{u=2} = \cos(2) \approx -0.416$$
- $$\frac{df}{dx}\big|_{x=1} = \frac{df}{du}\big|_{u=2} \cdot \frac{du}{dx}\big|_{x=1} 
                             = \cos(2) \cdot 2 \approx -0.832$$

### 计算图
通过计算图能更直观的理解梯度计算过程。计算图是一种表示数学表达式的有向无环图（DAG），其中：
- **节点**表示变量或者操作
- **边**表示数据流动的方向

对每个复杂函数都可以分解为基本操作的组合。比如，函数 $f(x, y) = (x + y) \times x$ 可以表示为：

```
输入节点:
x ────┐
      ├─→ Add (a = x + y) ────┐
y ────┘                       |
                              ↓
x ───────────────────────────→ Multiply (f = a × x) → 输出 f
```

对梯度计算，也就是反向过程，也能通过计算图表达：

```
输出节点:
f (∂f/∂f = 1)
│
Multiply [反向传播]
├──→ a (∂f/∂a = x) ────→ Add [反向传播]
|                         ├──→ x (∂a/∂x = 1) → 梯度贡献: ∂f/∂x = x
|                         └──→ y (∂a/∂y = 1) → ∂f/∂y = x
└──→ x (∂f/∂x = a = x + y) → 梯度贡献: ∂f/∂x = x + y
```

**梯度计算逻辑**

1. **乘法节点**（Multiply）的反向传播：
   - $\frac{\partial f}{\partial a} = x$
     （乘法规则：梯度 = 另一个输入的值）
   - $\frac{\partial f}{\partial x} = a = x + y$
     （乘法规则：梯度 = 另一个输入的值）

2. **加法节点**（Add）的反向传播：
   - $\frac{\partial a}{\partial x} = 1$ , $\frac{\partial a}{\partial y} = 1$
     （加法规则：梯度均匀分配）

3. **链式法则应用**：
   - 对 `x` 的梯度需累加两条路径的贡献：
     - **路径1**（通过乘法节点）： $\frac{\partial f}{\partial x} = x + y$
     - **路径2**（通过加法节点 → 乘法节点）： $\frac{\partial f}{\partial a} \cdot \frac{\partial a}{\partial x} = x \cdot 1 = x$
     - **总梯度**： $\frac{\partial f}{\partial x} = (x + y) + x = 2x + y$
   - 对 `y` 的梯度仅来自加法节点：
     - $\frac{\partial f}{\partial y} = \frac{\partial f}{\partial a} \cdot \frac{\partial a}{\partial y} = x \cdot 1 = x$


**注意：反向传播节点的输入/输出规则**
1. **反向过程的输入：操作节点（Op）的输入 = 前向过程的输入值 + 上游传递的梯度值**
  - 乘法节点 $f = a \times b$ 的反向传播需要知道前向的输入值 $a$ 和 $b$，以及上游梯度 $\frac{\partial L}{\partial f}$

2. **反向过程的输出：操作节点的输出梯度数 = 前向过程该节点的输入数**
  - 加法节点前向有 2 个输入（ $x, y$ ），反向时会输出 2 个梯度（ $\frac{\partial L}{\partial x}, \frac{\partial L}{\partial y}$ ）
  - 乘法节点同理，输出 2 个梯度

#### 排序
在反向传播过程中，计算梯度必须遵循 **依赖顺序**：
- 每个节点的梯度计算依赖于其后继节点的梯度（链式法则）。
- 拓扑排序保证：当计算节点 $v$ 的梯度时，所有依赖 $v$ 的后续节点梯度已计算完成。

以 \( f(x, y) = (x + y) \times x \) 为例：

| **阶段**       | **拓扑序**          | **操作类型**       | **目的**                     |
|----------------|---------------------|--------------------|------------------------------|
| **前向计算**   |  $x, y → \text{Add} → a → \text{Multiply} → f$  | 数据流动           | 计算输出值                   |
| **反向传播**   |  $f → \text{Multiply} → a → \text{Add} → x, y$  | 梯度回传           | 计算梯度（需逆向前向拓扑序） |

描述算法：

```
拓扑排序（深度优先搜索）:
1. 初始：维护已访问节点集合，避免重复处理
2. 循环：由目标节点开始深度优先遍历
   2.1 对每个节点，先递归访问其所有输入依赖
   2.2 所有依赖访问完成后，将当前节点加入排序结果
3. 最终得到的序列保证：依赖节点总是排在被依赖节点之前
```

### 梯度计算

对深度学习中来说，参数（输入）多，仅有一个损失函数值（输出）。非常适合计算图一节中讲述的方法来计算参数的梯度。本节再举一例，以及也介绍下另外一种对偶方法求解梯度。

梯度计算分为两个阶段：
1. **前向传播**：计算并存储所有中间结果
2. **反向传播**：从输出开始，反向计算所有梯度


假设要计算 $\frac{\partial f}{\partial x_i}$，其中 $f$ 是最终输出。根据链式法则：

$$\frac{\partial f}{\partial x_i} = \sum_j \frac{\partial f}{\partial y_j} \cdot \frac{\partial y_j}{\partial x_i}$$

这里 $y_j$ 是所有直接依赖于 $x_i$ 的中间变量。

举例：考虑函数 $f(x, y) = xy + \sin(x)$：

**前向传播**（计算函数值）：
1. $v_1 = xy$
2. $v_2 = \sin(x)$
3. $f = v_1 + v_2$

**反向传播**（计算梯度）：
1. $\frac{\partial f}{\partial f} = 1$（种子梯度）
2. $\frac{\partial f}{\partial v_1} = 1$，$\frac{\partial f}{\partial v_2} = 1$
3. $\frac{\partial f}{\partial x} = \frac{\partial f}{\partial v_1} \cdot \frac{\partial v_1}{\partial x} + \frac{\partial f}{\partial v_2} \cdot \frac{\partial v_2}{\partial x} = 1 \cdot y + 1 \cdot \cos(x) = y + \cos(x)$
4. $\frac{\partial f}{\partial y} = \frac{\partial f}{\partial v_1} \cdot \frac{\partial v_1}{\partial y} = 1 \cdot x = x$

#### 对偶方法

对偶数的形式为：$(a, a')$，其中 $a$ 是函数值，$a'$ 是导数值。

**对偶数的运算规则**

- 加法：$(u, u') + (v, v') = (u + v, u' + v')$
- 乘法：$(u, u') \times (v, v') = (uv, u'v + uv')$
- 幂运算：$(u, u')^n = (u^n, nu^{n-1}u')$

**计算过程**

考虑函数 $f(x) = x^2 + 2x$ 在 $x = 3$ 处的导数计算：

1. **初始化**：$x = (3, 1)$，其中 $1$ 表示 $\frac{dx}{dx} = 1$
2. **计算 $x^2$**：$(3, 1)^2 = (9, 6)$，因为 $\frac{d(x^2)}{dx} = 2x = 6$
3. **计算 $2x$**：$2 \times (3, 1) = (6, 2)$
4. **相加**：$(9, 6) + (6, 2) = (15, 8)$

结果：$f(3) = 15$，$f'(3) = 8$

注：该方法只是觉得比较有趣做下记录，不是本文重点。

## 实现

接下来来看一个完整的反向自动微分系统的C++实现。该实现由以下三个核心部分组成：
- **Tensor类**：表示计算图中的节点，存储数值和梯度
- **TensorOp抽象类**：表示计算图中的操作，定义前向和反向计算接口
- **具体操作类**：实现特定的数学运算（如加法、乘法）

### Tensor类

```cpp
class Tensor {
public:
    struct Back {
        const TensorOp* op = nullptr;
        std::vector<std::shared_ptr<Tensor>> inputs;  // 使用shared_ptr存储输入
    };

    Tensor(float value = 0.0, bool requires_grad = false)
        : data_(value), requires_grad_(requires_grad) {
    }
    
    // 数据访问
    float Data() const { return data_; }
    float Grad() const { return grad_; }
    bool RequiresGrad() const { return requires_grad_; }
    
    // 梯度管理
    void ZeroGrad() { grad_ = 0.0; }
    
    void Backward();  // 反向传播实现
    
private:
    float data_;                        // 存储数值
    float grad_ = 0.0;                 // 存储梯度
    bool requires_grad_;               // 是否需要梯度
    std::unique_ptr<Back> back_;       // 反向传播信息
    
    friend class TensorOp;
};
```

#### 数据成员
- **data_**：存储Tensor的数值，对应计算图中节点的值
- **grad_**：存储梯度值，在反向传播时计算得出
- **requires_grad_**：标记是否需要计算梯度，只有参数节点才需要
- **back_**：存储反向传播所需的信息，包括创建此节点的操作和输入节点

#### 梯度接口

```cpp
// 梯度相关
bool RequiresGrad() const { return requires_grad_; }
float Grad() const { return grad_; }
void ZeroGrad() { grad_ = 0.0; }
```
- **requires_grad()**：判断是否是计算图中需要求导的节点
- **grad()**：获取反向传播计算出的梯度值
- **zero_grad()**：清零梯度，用于新一轮的梯度计算

### 反向传播

```cpp
void Tensor::Backward() {
    if (!requires_grad_) {
        throw std::runtime_error("not require grad");
    }

    std::unordered_map<Tensor*, float> gradients;
    std::vector<Tensor*> topo_order;
    std::unordered_set<Tensor*> visited;

    // 拓扑排序
    std::function<void(Tensor*)> build_topo = [&](Tensor* v) {
        if (visited.count(v)) return;
        visited.insert(v);
        
        if (v->back_) {
            for (auto& input : v->back_->inputs) {
                if (input->RequiresGrad()) {
                    build_topo(input.get());  // 通过shared_ptr获取原始指针
                }
            }
        }
        topo_order.push_back(v);
    };

    build_topo(this);
    
    // 初始化根节点梯度
    gradients[this] = 1.0;
    
    // 反向传播
    for (auto it = topo_order.rbegin(); it != topo_order.rend(); ++it) {
        Tensor* v = *it;
        
        if (v->back_ && v->back_->op && gradients.count(v)) {
            auto input_grads = v->back_->op->Backward(gradients[v], v->back_->inputs);
            
            for (size_t i = 0; i < v->back_->inputs.size(); ++i) {
                if (v->back_->inputs[i]->RequiresGrad()) {
                    Tensor* input_ptr = v->back_->inputs[i].get();  // 获取原始指针
                    gradients[input_ptr] += input_grads[i];  // 梯度累积到原始对象
                }
            }
        }
    }
    
    // 将计算出的梯度设置到各个Tensor中
    for (auto& [tensor_ptr, grad_value] : gradients) {
        tensor_ptr->grad_ = grad_value;
    }
}
```

```cpp
void Backward() {
    if (!requires_grad_) {
        throw std::runtime_error("tensor doesn't require grad");
    }

    // 1. 拓扑排序 - 确保正确的计算顺序
    std::vector<Tensor*> topo_order;
    std::unordered_set<Tensor*> visited;
    
    std::function<void(Tensor*)> build_topo = [&](Tensor* t) {
        if (!t || visited.count(t)) return;
        visited.insert(t);
        
        if (t->back_) {
            for (auto& input : t->back_->inputs) {
                if (input.requires_grad()) {
                    build_topo(&input);
                }
            }
        }
        topo_order.push_back(t);
    };
    
    build_topo(this);

    // 2. 初始化根节点梯度
    this->grad_ = 1.0;

    // 3. 反向传播 - 按拓扑顺序计算梯度
    for (auto it = topo_order.rbegin(); it != topo_order.rend(); ++it) {
        Tensor* current = *it;
        if (current->back_ && current->back_->op) {
            auto input_grads = current->back_->op->backward(
                current->grad_, current->back_->inputs);
            
            for (size_t i = 0; i < current->back_->inputs.size(); ++i) {
                if (current->back_->inputs[i].requires_grad()) {
                    current->back_->inputs[i].grad_ += input_grads[i];
                }
            }
        }
    }
}
```

#### 种子梯度

```cpp
gradients[this] = 1.0;
```

这行代码设置了种子梯度。对于输出节点，$\frac{\partial f}{\partial f} = 1$，这是链式法则计算的起点。

#### 梯度累计


```cpp
current->back_->inputs[i].grad_ += input_grads[i];
```

使用 `+=` 而不是 `=` 是因为一个节点可能被多个后续节点使用。根据链式法则，总梯度是所有路径梯度的和：

$$\frac{\partial f}{\partial x} = \sum_i \frac{\partial f}{\partial y_i} \cdot \frac{\partial y_i}{\partial x}$$

### Op类

```cpp
class TensorOp {
public:
    virtual ~TensorOp() = default;
    
    // 统一调用接口：注意参数和返回值都是shared_ptr
    std::shared_ptr<Tensor> operator()(const std::vector<std::shared_ptr<Tensor>>& inputs) {
        Tensor result_tensor = Process(inputs);
        auto result = std::make_shared<Tensor>(result_tensor.Data());
        
        // 检查是否需要梯度
        bool need_grad = false;
        for (const auto& input : inputs) {
            if (input->RequiresGrad()) {
                need_grad = true;
                break;
            }
        }
        
        if (need_grad) {
            result->requires_grad_ = true;
            result->back_ = std::make_unique<Tensor::Back>();
            result->back_->op = this;
            result->back_->inputs = inputs;  // 存储shared_ptr，不会拷贝Tensor对象
        }
        
        return result;
    }
    
    virtual Tensor Process(const std::vector<std::shared_ptr<Tensor>>& inputs) const = 0;
    virtual std::vector<float> Backward(float grad_output, 
                                       const std::vector<std::shared_ptr<Tensor>>& inputs) const = 0;
};
```

- **Process()**：计算操作的前向结果，对应计算图中从输入到输出的数据流
- **Backward()**：计算局部梯度，对应链式法则中的 $\frac{\partial y}{\partial x_i}$
- **operator()**：Functor接口，构建计算图的连接关系，注意只调用Process不会建图

**梯度传播机制：** 只有当至少一个输入需要梯度时，输出才需要梯度。这是一个重要的优化：如果所有输入都是常数，那么输出也可以看作常数，不需要构建计算图。

### Op实现

#### 加法操作

```cpp
class AddOp : public TensorOp {
public:
    Tensor Process(const std::vector<std::shared_ptr<Tensor>>& inputs) const override {
        if (inputs.size() != 2) {
            throw std::runtime_error("Add requires 2 inputs");
        }
        return Tensor(inputs[0]->Data() + inputs[1]->Data());
    }
    
    std::vector<float> Backward(float grad_output, 
                               const std::vector<std::shared_ptr<Tensor>>& inputs) const override {
        return {grad_output, grad_output};
    }
};
```

**数学原理**：对于 $z = x + y$，有 $\frac{\partial z}{\partial x} = 1$， $\frac{\partial z}{\partial y} = 1$ 

根据链式法则： $\frac{\partial f}{\partial x} = \frac{\partial f}{\partial z} \cdot \frac{\partial z}{\partial x} = \text{grad\_output} \cdot 1$ 

#### 乘法操作

```cpp
class MulOp : public TensorOp {
public:
    Tensor Process(const std::vector<std::shared_ptr<Tensor>>& inputs) const override {
        if (inputs.size() != 2) {
            throw std::runtime_error("Mul requires 2 inputs");
        }
        return Tensor(inputs[0]->Data() * inputs[1]->Data());
    }
    
    std::vector<float> Backward(float grad_output, 
                               const std::vector<std::shared_ptr<Tensor>>& inputs) const override {
        return {grad_output * inputs[1]->Data(), grad_output * inputs[0]->Data()};
    }
};
```

**数学原理**：对于 $z = x \times y$，有 $\frac{\partial z}{\partial x} = y$，$\frac{\partial z}{\partial y} = x$

根据链式法则：
- $\frac{\partial f}{\partial x} = \frac{\partial f}{\partial z} \cdot \frac{\partial z}{\partial x} = \text{grad\_output} \cdot y$
- $\frac{\partial f}{\partial y} = \frac{\partial f}{\partial z} \cdot \frac{\partial z}{\partial y} = \text{grad\_output} \cdot x$

### 运算符重载

```cpp
// 全局操作实例
AddOp add_op;
MulOp mul_op;

// 便捷函数 - 运算符重载
std::shared_ptr<Tensor> operator+(std::shared_ptr<Tensor> a, std::shared_ptr<Tensor> b) {
    return add_op({a, b});
}

std::shared_ptr<Tensor> operator*(std::shared_ptr<Tensor> a, std::shared_ptr<Tensor> b) {
    return mul_op({a, b});
}
```

运算符重载使得我们可以用自然的数学表达式来构建计算图：`z = x * y + x`

注意：这里的参数直接使用 `std::shared_ptr<T>`（其中 `T = Tensor`）而不是引用，这样可以避免额外的拷贝和转换。


### 完整示例


让我们分析一个完整的计算过程：


```cpp
void test_auto() {
    // 创建输入张量
    auto x = std::make_shared<Tensor>(3.0, true);  // x = 3, requires_grad=True
    auto y = std::make_shared<Tensor>(2.0, true);  // y = 2, requires_grad=True

    // 前向计算: z = x * y + x
    auto z = x * y + x;

    std::cout << "x = " << x->Data() << std::endl;     // 输出: 3
    std::cout << "y = " << y->Data() << std::endl;     // 输出: 2
    std::cout << "z = " << z->Data() << std::endl;     // 输出: 9

    // 反向传播
    z->Backward();

    std::cout << "Dz/Dx = " << x->Grad() << std::endl;  // 输出: 3
    std::cout << "Dz/Dy = " << y->Grad() << std::endl;  // 输出: 3
}
```

#### 前向传播过程

1. `x * y`：创建一个新的 `shared_ptr` 节点，值为 6，连接到 `x` 和 `y`
2. `(x * y) + x`：创建最终的 `shared_ptr` 节点 `z`，值为 9，连接到前面的结果和 `x`

计算图结构：
```
x ──┬──→ × ──┬──→ + ──→ z
    │        │     ↑
y ──┘        │     │
             │     │
x ───────────┴─────┘
```

#### 反向传播过程

1. **拓扑排序**：[x, y, x*y, z]
2. **初始化**：z.grad = 1.0
3. **反向传播**：
   - z → (x*y) 和 x：梯度都是1.0
   - (x*y) → x 和 y：梯度分别是y=2和x=3
   - 最终：x.grad = 1.0 + 2.0 = 3.0，y.grad = 3.0

这个结果符合数学推导：
- $\frac{\partial z}{\partial x} = \frac{\partial}{\partial x}(xy + x) = y + 1 = 2 + 1 = 3$
- $\frac{\partial z}{\partial y} = \frac{\partial}{\partial y}(xy + x) = x = 3$

## 问题

**为什么使用std::shared_ptr存储？**

答案：如果使用值类型存储输入节点，会导致严重的对拷拷贝问题：

```cpp
// 错误的设计
struct Back {
    std::vector<Tensor> inputs;  // 存储拷贝！
};

Tensor operator()(const std::vector<Tensor>& inputs)
```

**问题分析**：
```
1. 创建原始变量
   x @ 0x1000 (data=3.0, grad=0.0)
   y @ 0x1100 (data=2.0, grad=0.0)

2. 执行 x + y 时会发生：
   operator+ 调用 add_op({x, y})
   ↓
   计算图中存储：{x_copy @ 0x2000, y_copy @ 0x2100}

3. 反向传播时：
   梯度累积到拷贝对象：
   x_copy @ 0x2000 (grad=1.0)  ← 梯度在这里！
   y_copy @ 0x2100 (grad=1.0)  ← 梯度在这里！
   
   而原始变量仍然是：
   x @ 0x1000 (grad=0.0)  ← 用户看到的是0！
   y @ 0x1100 (grad=0.0)  ← 用户看到的是0！
```

**使用shared_ptr的解决方案**：
```cpp
struct Back {
    std::vector<std::shared_ptr<Tensor>> inputs;  // 存储指针，指向原始对象
};

std::shared_ptr<Tensor> operator()(const std::vector<std::shared_ptr<Tensor>>& inputs)
```

**解决后的数据流**：
```
1. 创建原始变量
   x @ 0x1000 (data=3.0, grad=0.0)
   y @ 0x1100 (data=2.0, grad=0.0)

2. 执行 x + y 时：
   计算图中存储：{shared_ptr→0x1000, shared_ptr→0x1100}

3. 反向传播时：
   梯度正确累积到原始对象：
   x @ 0x1000 (grad=1.0)  ← 梯度正确！
   y @ 0x1100 (grad=1.0)  ← 梯度正确！
```

不像Python那样的语言一切都是对象，C++中务必要注意这些临时变量的生成机制。
