# 模型训练

## 引言
前面实现了自动微分机制与带有自动微分机制的多维数组。接下来能够在这个基础上继续构建深度学习框架的组件：参数管理、网络模块和优化器。通过实际的 MNIST 手写数组识别任务，深入阐述这些组件的设计理念和实现细节。

深度学习框架的模块化需要理清的关键问题：
1. **参数与数据的区分**：参数需要梯度计算和更新，普通数据不需要
2. **模块的组合性**：复杂的神经网络由简单的基础模块组合而成
3. **参数的统一管理**：所有参数需要被优化器统一更新
4. **设备的透明性**：参数在不同设备间的迁移应该自动化

下面就通过项目中的实际代码来展示这些设计怎样在实践中发挥作用。

## 概览

深入具体实现之前，通过示例图先理解下这个系统的架构关系：

```
┌─────────────────────────────────────────────────────────────┐
│                      训练流程                                │
├─────────────────────────────────────────────────────────────┤
│  数据加载 → 前向传播 → 损失计算 → 反向传播 → 参数更新        │
└─────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────┐
│                    核心组件关系                              │
├─────────────────────────────────────────────────────────────┤
│  Module (网络层)                                            │
│    ├── Parameter (可训练参数)                               │
│    │     └── Tensor (数据载体)                              │
│    └── 子Module (嵌套结构)                                  │
│                                    ↓                        │
│  Optimizer (优化器)                                         │
│    └── 管理所有Parameter的更新                               │
└─────────────────────────────────────────────────────────────┘
```

## Parameter

Parameter 类是对 Tensor 的简单包装，其核心作用是将普通的张量标记为"可训练参数"。


```cpp
class Parameter : public Tensor {
public:
    Parameter() : Tensor() {} 
    Parameter(const Tensor& tensor) 
        : Tensor(tensor) {
        RequiresGrad(true);  // 核心：自动启用梯度计算
    }
};
```


**为什么 Parameter 继承 Tensor 而不是组合？**

继承方案使得 Parameter 可以无缝地在任何需要 Tensor 的地方使用，避免了大量的接口转发代码。在深度学习框架中，参数的使用频率极高，这种设计提供了最佳的性能和易用性。

**自动梯度启用的重要性**

构造函数中的 `RequiresGrad(true)` 确保了任何 Parameter 对象都会参与梯度计算。这是一个重要的安全机制，防止开发者忘记启用梯度而导致训练失败。

## Module

Module 类是整个神经网络架构的基础，它解决了参数管理和模块组合的核心问题。

```cpp
class Module {
public:
    using NameParameter = std::pair<std::string, Parameter*>;

    virtual ~Module() = default;

    // 注册参数到当前模块
    void RegisterParameter(const std::string& name, 
                           std::unique_ptr<Parameter> param) {
        parameters_[name] = std::move(param);
    }

    // 注册子模块
    void RegisterModule(const std::string& name, std::shared_ptr<Module> module) {
        modules_[name] = module;
    }

    // 递归收集所有参数 - 核心功能
    std::vector<NameParameter> Parameters(const std::string& n = "") {
        std::vector<NameParameter> params;

        // 收集当前模块的参数
        for (auto& [name, param] : parameters_) {
            std::string m = n.empty() ? name : n + "." + name;
            params.push_back({m, param.get()});
        }

        // 递归收集子模块的参数
        for (auto& [name, module] : modules_) {
            std::string p = n.empty() ? name : n +"." + name;
            auto sub_params = module->Parameters(p);
            params.insert(params.end(), sub_params.begin(), sub_params.end());
        }

        return params;
    } 

protected:
    std::unordered_map<std::string, std::unique_ptr<Parameter>> parameters_;
    std::unordered_map<std::string, std::shared_ptr<Module>> modules_;
};
```

**参数的递归收集**：`Parameters()` 方法通过递归遍历所有子模块，收集整个网络的所有参数。这种设计使得优化器可以轻松地获取到所有需要更新的参数。

**内存管理策略**：
- 参数使用 `unique_ptr` 管理，确保清晰的所有权
- 子模块使用 `shared_ptr` 管理，允许模块复用

### Linear

```cpp
class Linear : public Module {
public:
    Linear(int64_t in_features, int64_t out_features) {
        // He初始化 - 专为ReLU激活函数设计

        // 初始化weight
        auto temp = Tensor::Randn({out_features, in_features});
        auto weight_tensor = temp * std::sqrt(2.0/in_features);
        auto weight_param = std::make_unique<Parameter>(weight_tensor);
        weight_ = weight_param.get();
        RegisterParameter("weight", std::move(weight_param));
        
        // 初始化bias
        auto bias_tensor = Tensor::Zeros({out_features});
        auto bias_param = std::make_unique<Parameter>(bias_tensor);
        bias_ = bias_param.get();
        RegisterParameter("bias", std::move(bias_param));
    }

    Tensor OpValue(const Tensor& input) {
        return input.MatMul(weight_->T()) + *bias_; // 使用->和*访问成员
    }

private:
    Parameter* weight_; // 原始指针
    Parameter* bias_;
};
```

**权重初始化**：使用 He 初始化，专为 ReLU 激活函数设计，标准差为 `sqrt(2.0 / fan_in)`，这有助于保持梯度的方差稳定，避免梯度消失或爆炸。

**内存管理模式**：
1. 创建 `unique_ptr<Parameter>` 管理参数的生命周期
2. 通过 `RegisterParameter()` 将参数注册到模块中
3. 保存原始指针 `Parameter*` 用于高效访问 (只有访问权限，不需要管理生命周期)

**前向传播**：`input.MatMul(weight_->T()) + *bias_` 实现了标准的线性变换 $y = xW^T + b$。

### MLP

以下是一个三成MLP的实现：

```cpp
class MLP : public Module {
private:
    std::shared_ptr<Linear> fc1_; // 第一个全连接层
    std::shared_ptr<Linear> fc2_; // 第二个全连接层
public:
    MLP(int64_t input_size, int64_t hidden_size, int64_t output_size) {
        fc1_ = std::make_shared<Linear>(input_size, hidden_size);
        fc2_ = std::make_shared<Linear>(hidden_size, output_size);

        RegisterModule("fc1", fc1_);
        RegisterModule("fc2", fc2_);
    }

    // 前向传播：输入 -> FC1 -> ReLU -> FC2 -> 输出
    Tensor Process(const Tensor& input) {
        auto h1 = fc1_->Process(input);
        h1 = ReLU(h1); // 激活函数
        auto output = fc2_->Process(h1);
        return output;
    }
};
```

## Optimizer

Optimizer 类定义了所有优化器的通用接口：

```cpp
class Optimizer {
public:
    virtual ~Optimizer() = default;

    // 接收并管理参数列表
    virtual void InsertParameters(
        const std::vector<Module::NameParameter>& parameters) {
        parameters_.insert(parameters_.end(),
                           parameters.begin(),
                           parameters.end());
    }

    // 清零所有参数的梯度
    virtual void ZeroGrad() {
        for (auto& [_, param] : parameters_) {
            param->ZeroGrad();
        }
    }

    // 纯虚函数，具体优化算法由子类实现
    virtual void Step() = 0;

protected:
    std::vector<Module::NameParameter> parameters_;
};
```

**设计要点**：
- `InsertParameters()` 接收模块的参数列表
- `ZeroGrad()` 清零所有参数的梯度
- `Step()` 是纯虚函数，由具体优化器实现

### SGD

让我们看看项目中的 SGD Optimizer实现，它带有一个梯度裁剪功能：

```cpp
class SGD : public Optimizer {
public:
    SGD(float lr = 0.01) : lr_(lr) {}

    void Step() override {
        for (const auto& [name, param] : parameters_) {
            // 跳过不需要梯度的参数
            if (!param->RequiresGrad() || !param->Grad().Node()) {
                continue;
            }
            
            auto grad = param->Grad();
            
            // 梯度裁剪 - 防止梯度爆炸
            auto grad_data = grad.Data<float>();
            float max_grad = 0.1f;  // 梯度裁剪阈值
            for (int64_t i = 0; i < grad.NumElements(); i++) {
                if (std::abs(grad_data[i]) > max_grad) {
                    grad_data[i] = (grad_data[i] > 0) ? max_grad : -max_grad;
                }
            }
            
            // 参数更新：param = param - lr * clipped_grad
            auto lr_update = grad * lr_;
            param->Update(lr_update);
        }
    }

private:
    float lr_;  // 学习率
};
```

**梯度裁剪**：通过限制梯度的最大值（0.1）来防止梯度爆炸，这是训练稳定性的重要保障。实现方式是直接修改梯度数据，将超出阈值的梯度值截断到阈值范围内。


## MNIST

接下来是一个完整的模型训练示例。

### 数据处理

```cpp
class MNISTLoader {
public:
    struct DataSet {
        std::vector<std::vector<float>> images;
        std::vector<int> labels;
    };
    
    static std::vector<std::vector<float>> ReadImages(const std::string& filename, int max_samples = -1) {
        std::ifstream file(filename, std::ios::binary);
        if (!file) {
            throw std::runtime_error("Cannot open " + filename);
        }
        
        // 读取 MNIST 文件头
        uint32_t magic_number, num_images, rows, cols;
        file.read(reinterpret_cast<char*>(&magic_number), 4);
        file.read(reinterpret_cast<char*>(&num_images), 4);
        file.read(reinterpret_cast<char*>(&rows), 4);
        file.read(reinterpret_cast<char*>(&cols), 4);
        
        // 处理字节序问题
        magic_number = swap_endian(magic_number);
        num_images = swap_endian(num_images);
        rows = swap_endian(rows);
        cols = swap_endian(cols);
        
        // 读取图像数据并归一化
        std::vector<std::vector<float>> images(num_images);
        for (uint32_t i = 0; i < num_images; ++i) {
            images[i].resize(rows * cols);
            for (uint32_t j = 0; j < rows * cols; ++j) {
                unsigned char pixel;
                file.read(reinterpret_cast<char*>(&pixel), 1);
                images[i][j] = static_cast<float>(pixel) / 255.0f;  // 归一化到 [0,1]
            }
        }
        return images;
    }
    
    static DataSet LoadTrainingData(int max_samples = -1) {
        DataSet dataset;
        dataset.images = ReadImages("/home/tfbao/now/Zero/san/data/train-images-idx3-ubyte", max_samples);
        dataset.labels = ReadLabels("/home/tfbao/now/Zero/san/data/train-labels-idx1-ubyte", max_samples);
        return dataset;
    }
};
```

**处理要点**：
- 处理二进制文件格式和字节序问题
- 将像素值从 [0,255] 归一化到 [0,1]
- 支持限制加载的样本数量（便于调试）

### 训练循环


```cpp
void train_mnist_mlp_sgd() {
    std::cout << "=== Training MNIST MLP with SGD ===\n" << std::endl;
    
    // 1. 数据准备
    auto train_data = MNISTLoader::LoadTrainingData(20000);
    auto test_data = MNISTLoader::LoadTestData(2000);
    
    // 2. 超参数设置
    const int64_t input_size = 784;     // 28x28 像素
    const int64_t hidden_size = 512;    // 隐藏层大小
    const int64_t num_classes = 10;     // 10个数字类别
    const float learning_rate = 0.05f;  // 学习率
    const int batch_size = 50;          // 批次大小
    const int epochs = 20;              // 训练轮数
    
    // 3. 模型和优化器初始化
    auto model = std::make_unique<SimpleMLP>(input_size, hidden_size, num_classes);
    auto optimizer = std::make_unique<ClippedSGD>(learning_rate);
    
    // 4. 参数管理
    auto all_params = model->Parameters();
    for (auto& param_pair : all_params) {
        param_pair.second->RequiresGrad(true);  // 确保所有参数都需要梯度
    }
    optimizer->InsertParameters(all_params);
    
    // 5. 训练循环
    for (int epoch = 0; epoch < epochs; epoch++) {
        // 学习率衰减策略
        if (epoch == 1) {
            float new_lr = learning_rate / 2.0f;
            optimizer = std::make_unique<ClippedSGD>(new_lr);
            optimizer->InsertParameters(all_params);
            std::cout << "Learning rate decayed to: " << std::fixed << std::setprecision(6) << new_lr << std::endl;
        } else if (epoch == 2) {
            float new_lr = learning_rate / 4.0f;
            optimizer = std::make_unique<ClippedSGD>(new_lr);
            optimizer->InsertParameters(all_params);
            std::cout << "Learning rate decayed to: " << std::fixed << std::setprecision(6) << new_lr << std::endl;
        }
        
        float total_loss = 0.0f;
        float total_accuracy = 0.0f;
        int num_batches = 0;
        
        // 批次训练
        for (size_t i = 0; i + batch_size <= train_data.images.size(); i += batch_size) {
            // 准备批次数据
            std::vector<float> batch_images;
            std::vector<int> batch_labels;
            
            for (int j = 0; j < batch_size; j++) {
                batch_images.insert(batch_images.end(), 
                                  train_data.images[i + j].begin(), 
                                  train_data.images[i + j].end());
                batch_labels.push_back(train_data.labels[i + j]);
            }
            
            // 创建张量
            auto images = Tensor::From<float>(batch_images, {batch_size, input_size});
            
            // 标准训练步骤
            optimizer->ZeroGrad();                              // 1. 清零梯度
            auto predictions = model->OpValue(images);          // 2. 前向传播
            auto loss = ImprovedCrossEntropyLoss(predictions, batch_labels);  // 3. 计算损失
            loss.Backward();                                    // 4. 反向传播
            optimizer->Step();                                  // 5. 更新参数
            
            // 统计信息
            total_loss += loss.Data<float>()[0];
            total_accuracy += CalculateAccuracy(predictions, batch_labels);
            num_batches++;
        }
        
        float avg_loss = total_loss / num_batches;
        float avg_accuracy = total_accuracy / num_batches;
        
        std::cout << "Epoch [" << epoch + 1 << "/" << epochs 
                 << "] Summary: Avg Loss: " << avg_loss
                 << ", Avg Accuracy: " << avg_accuracy * 100 << "%" << std::endl;
    }
}
```

**1. 数据准备**：每个批次将多个样本的数据拼接成一个大的张量，标签保持为整数向量。

**2. 标准训练循环**：
- `optimizer->ZeroGrad()` - 清零上一步的梯度
- `model->Process(images)` - 前向传播计算预测值
- `CrossEntropyLoss()` - 计算损失函数
- `loss.Backward()` - 反向传播计算梯度
- `optimizer->Step()` - 根据梯度更新参数

**3. 学习率衰减**：通过重新创建优化器来实现学习率衰减，这是一种简单但有效的策略。

**4. 性能监控**：实时计算损失和准确率，便于监控训练进度。

### 准确率

准确率计算函数：

```cpp
// nas/apps/mnist_mlp_sgd.cc:204-227
float CalculateAccuracy(const Tensor& predictions, const std::vector<int>& true_labels) {
    auto pred_data = predictions.Data<float>();
    int correct = 0;
    int batch_size = true_labels.size();
    int num_classes = predictions.NumElements() / batch_size;
    
    for (int i = 0; i < batch_size; i++) {
        // 找到预测概率最大的类别
        int predicted_class = 0;
        float max_score = pred_data[i * num_classes];
        
        for (int j = 1; j < num_classes; j++) {
            if (pred_data[i * num_classes + j] > max_score) {
                max_score = pred_data[i * num_classes + j];
                predicted_class = j;
            }
        }
        
        if (predicted_class == true_labels[i]) {
            correct++;
        }
    }
    
    return static_cast<float>(correct) / batch_size;
}
```

## 总结

本文通过实际的 MNIST 训练代码，深入阐述了深度学习框架的核心组件：

### 核心组件

**Parameter 类**：
- 将普通张量标记为可训练参数，自动启用梯度计算
- 通过继承而非组合提供最佳性能和易用性
- 确保参数安全地参与训练过程

**Module 系统**：
- 提供模块化的网络构建能力，支持参数的递归收集和设备管理
- 通过智能指针实现清晰的内存管理和所有权语义
- 支持灵活的模块组合和嵌套结构

**Optimizer 系统**：
- 抽象了参数优化的通用接口，支持各种优化算法
- 提供统一的参数管理和梯度更新机制
- 实现了梯度裁剪等训练稳定性技术

### 设计哲学

这些组件通过精心设计的接口和内存管理策略，实现了：

1. **高效性**：最小化内存拷贝，优化访问模式
2. **安全性**：通过 RAII 和智能指针保证内存安全
3. **易扩展性**：清晰的继承层次和抽象接口
4. **易维护性**：模块化设计降低耦合度

### 实践价值

项目中的实际代码展示了这些设计原则在实践中的应用：

- **完整的训练流程**：从数据加载到模型训练的端到端实现
- **真实的工程考虑**：错误处理、内存管理、性能优化
- **实用的训练技巧**：梯度裁剪、学习率衰减、权重初始化

通过 MNIST 手写数字识别任务的完整实现，我们看到了这些组件如何协同工作，形成一个完整的深度学习训练流程。这种模块化的设计不仅提高了代码的可维护性，也为后续的功能扩展提供了清晰的架构基础。

