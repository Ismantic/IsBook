# Gpt

## 引言

Gpt (Generative Pre-trained Transformer) 是一个**自回归语言模型**，它能够根据给定的文本上下文生成后续文本。数学角度看，Gpt 学习的是一个条件概率分布：

$$P(x_n|x_1,x_x, \ldots, x_{n-1})$$

其中 $x_1, x_2, \ldots, x_{n-1}$ 是已知的上下文， $x_n$ 是要预测的下一个词。


Gpt 基于 Transformer 的**解码器**架构，由以下几个关键部分组成：

```
输入文本 → 嵌入层 → 位置编码 → Transformer块 × N → 输出层 → 概率分布
```

### 嵌入层

对于输入序列 $\mathbf{x} = [x_1, x_2, \ldots, x_n]$，首先将每个token转换为稠密向量：

$$\mathbf{h}_0 = \mathbf{E}_{token}(\mathbf{x}) + \mathbf{E}_{pos}([1, 2, \ldots, n])$$

其中：
- $\mathbf{E}_{token} \in \mathbb{R}^{V \times d}$ 是词汇表嵌入矩阵
- $\mathbf{E}_{pos} \in \mathbb{R}^{L \times d}$ 是位置嵌入矩阵
- $V$ 是词汇表大小，$d$ 是隐藏维度，$L$ 是最大序列长度

### 位置编码

**问题**：自注意力机制本身是位置无关的，但序列的顺序信息对语言理解至关重要。

**解决方案**：GPT使用**可学习的位置嵌入**而非固定的正弦位置编码。

对于位置 $p \in [0, L-1]$，位置嵌入向量为：
$$\mathbf{p}_p = \mathbf{E}_{pos}[p, :] \in \mathbb{R}^d$$

**与原始Transformer的区别**：
- **原始Transformer**：使用固定的正弦/余弦位置编码
- **GPT**：使用可学习嵌入，能够更好地适应特定任务和数据分布

### 层归一化

Layer Normalization是GPT稳定训练的关键组件：

$$\text{LayerNorm}(\mathbf{x}) = \gamma \odot \frac{\mathbf{x} - \boldsymbol{\mu}}{\sqrt{\boldsymbol{\sigma}^2 + \epsilon}} + \boldsymbol{\beta}$$

其中：
- $\boldsymbol{\mu} = \frac{1}{d}\sum_{i=1}^{d} x_i$ （均值）
- $\boldsymbol{\sigma}^2 = \frac{1}{d}\sum_{i=1}^{d} (x_i - \mu)^2$ （方差）
- $\gamma, \boldsymbol{\beta} \in \mathbb{R}^d$ 是可学习的缩放和偏移参数
- $\epsilon$ 是防止除零的小常数（通常为 $10^{-5}$）

**作用**：
- 稳定训练，减少内部协变量偏移
- 加速收敛，提高训练效率
- 减少对初始化的敏感性

### 多头自注意力


#### 缩放点积注意力

$$\text{Attention}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{softmax}\left(\frac{\mathbf{Q}\mathbf{K}^T}{\sqrt{d_k}}\right)\mathbf{V}$$

#### 因果掩码

为了确保模型在预测位置$i$时只能看到位置$1$到$i-1$的信息：

$$\text{mask}_{i,j} = \begin{cases}
0 & \text{if } j \leq i \\
-\infty & \text{if } j > i
\end{cases}$$

**实现**：
$$\text{Attention}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{softmax}\left(\frac{\mathbf{Q}\mathbf{K}^T + \mathbf{M}}{\sqrt{d_k}}\right)\mathbf{V}$$

#### 多头机制

**原理**：将表示空间分解为多个子空间， $d_{model} = h \times d_k$ 

每个头 $i$ 独立计算：
$$\text{head}_i = \text{Attention}(\mathbf{Q}_i, \mathbf{K}_i, \mathbf{V}_i)$$

其中：
- $\mathbf{Q}_i = \mathbf{X}\mathbf{W}_i^Q$
- $\mathbf{K}_i = \mathbf{X}\mathbf{W}_i^K$ 
- $\mathbf{V}_i = \mathbf{X}\mathbf{W}_i^V$

**拼接和投影**：
$$\text{MultiHead}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{Concat}(\text{head}_1, \ldots, \text{head}_h)\mathbf{W}^O$$


### 前馈网络

$$\text{FFN}(\mathbf{x}) = \text{GELU}(\mathbf{x}\mathbf{W}_1 + \mathbf{b}_1)\mathbf{W}_2 + \mathbf{b}_2$$

**GELU激活函数**

Gpt使用GELU（Gaussian Error Linear Unit）：

$$\text{GELU}(x) = x \cdot \Phi(x) = x \cdot P(X \leq x), \quad X \sim \mathcal{N}(0,1)$$

**近似计算**：
$$\text{GELU}(x) \approx 0.5x\left(1 + \tanh\left(\sqrt{\frac{2}{\pi}}\left(x + 0.044715x^3\right)\right)\right)$$

**优势**：
- 平滑性：相比ReLU在所有点都可导
- 非单调性：在负值区域有非零输出
- 概率解释：基于高斯分布

### 残差连接

每个子层都使用残差连接：
$$\mathbf{h}_{l+1} = \mathbf{h}_l + F(\mathbf{h}_l)$$

**梯度流分析**：
$$\frac{\partial \mathcal{L}}{\partial \mathbf{h}_l} = \frac{\partial \mathcal{L}}{\partial \mathbf{h}_{l+1}} \left(1 + \frac{\partial F(\mathbf{h}_l)}{\partial \mathbf{h}_l}\right)$$

残差连接确保梯度至少有一条直接路径，缓解梯度消失问题。

另外，Gpt使用**Pre-LN**方式，即在子层之前应用LayerNorm：

$$\begin{align}
\mathbf{h}_l^{(1)} &= \text{LayerNorm}(\mathbf{h}_{l-1}) \\
\mathbf{h}_l^{(2)} &= \mathbf{h}_{l-1} + \text{MultiHead}(\mathbf{h}_l^{(1)}) \\
\mathbf{h}_l^{(3)} &= \text{LayerNorm}(\mathbf{h}_l^{(2)}) \\
\mathbf{h}_l &= \mathbf{h}_l^{(2)} + \text{FFN}(\mathbf{h}_l^{(3)})
\end{align}$$

**Pre-LN的优势**：
- 训练稳定性：梯度更稳定，不易爆炸
- 支持更深网络
- 学习率调度更容易

### 以图示例

```
输入序列: ["The", "cat", "sat", "on", "mat"]
    ↓
┌─────────────────────────────────────────────────┐
│ 嵌入层 (Embedding Layer)                        │
│ Token Embedding + Position Embedding           │
│ "The" + pos_0 → combined_vector                │
│ "cat" + pos_1 → combined_vector                │
│ ...                                            │
└─────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────┐
│ Transformer Block 1                           │
│                                                │
│ ┌─ LayerNorm ──→ Multi-Head Attention ──────┐   │
│ │   • 8个注意力头并行计算                    │   │
│ │   • 因果掩码确保单向信息流                  │   │
│ │   • 拼接后通过输出投影                      │   │
│ └─────────────────────────────────────────────┘   │
│ │                     ↓                          │
│ └─────→ [+] ← 残差连接                           │
│         ↓                                      │
│ ┌─ LayerNorm ──→ Feed Forward Network ──────┐   │
│ │   • GELU激活函数                           │   │
│ │   • 两层线性变换                            │   │
│ └─────────────────────────────────────────────┘   │
│ │                     ↓                          │
│ └─────→ [+] ← 残差连接                           │
└─────────────────────────────────────────────────┘
    ↓
    ... (重复N层)
    ↓
┌─────────────────────────────────────────────────┐
│ 最终层归一化 → 输出投影                          │
│ logits = LayerNorm(h_N) · W_token^T             │
│ 形状: [batch, seq_len, vocab_size]              │
└─────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────┐
│ Softmax (推理时)                                │
│ P(next_token|context) = softmax(logits[:,-1,:]) │
└─────────────────────────────────────────────────┘
```

#### 因果注意力

对于序列 ["The", "cat", "sat"]，因果掩码确保：

```
注意力矩阵:
        The   cat   sat
The   [ ✓     ✗     ✗  ]  ← "The" 只能看到自己
cat   [ ✓     ✓     ✗  ]  ← "cat" 能看到 "The" 和自己  
sat   [ ✓     ✓     ✓  ]  ← "sat" 能看到前面所有词

其中: ✓ = 可以关注, ✗ = 被掩码 (-∞)
```

#### 多头注意力

假设有4个注意力头处理句子 "The big cat sat on the mat"：

```
Head 1 (语法关系):
cat ←→ big     (形容词修饰名词)
sat ←→ cat     (主谓关系)
sat ←→ mat     (动宾关系)

Head 2 (长距离依赖):
The ←→ cat     (定冠词指向名词)
the ←→ mat     (另一个定冠词)

Head 3 (语义相似性):
cat ←→ mat     (都是名词，物理实体)
big ←→ on      (都是修饰词)

Head 4 (位置关系):
sat ←→ on      (动词和介词的搭配)
on ←→ mat      (介词和其宾语)
```

## 实现

### Op

#### MatMul

矩阵乘法是注意力机制的核心操作：

```cpp
// 自注意力中的关键计算基于项目实现

// 计算 Q、K、V 投影
auto query = input.MatMul(weight_q_expanded) + *bias_q_;
auto key = input.MatMul(weight_k_expanded) + *bias_k_;
auto value = input.MatMul(weight_v_expanded) + *bias_v_;

// 注意力分数计算
auto attention_scores = query.MatMul(key.Transpose(-2,-1));
```

#### Softmax

数值稳定的softmax实现：

```cpp
auto attention_probs = Softmax(attention_scores);
```

#### GELU

```cpp
auto activated = GeLU(hidden);
```

#### Lookup

```cpp
// 基于项目中的Embedding实现
class Embedding : public Module {
    Tensor OpValue(const Tensor& indices) {
        return Lookup(*weight_, indices);
    }
};
```

### Module

基于项目中的实际实现，每个模块都继承自 `Module` 基类，并使用 `OpRegistry` 来创建操作。

首先让我们看看 Linear 层的实际实现：

```cpp
class Linear : public Module {
public:
    Linear(int64_t in_features, int64_t out_features) {
        // 初始化weight
        auto temp = Tensor::Randn({out_features, in_features});
        auto weight_tensor = temp * std::sqrt(2.0/(in_features+out_features));
        auto weight_param = std::make_unique<Parameter>(weight_tensor);
        weight_ = weight_param.get();
        RegisterParameter("weight", std::move(weight_param));
        
        // 初始化bias
        auto bias_tensor = Tensor::Zeros({out_features});
        auto bias_param = std::make_unique<Parameter>(bias_tensor);
        bias_ = bias_param.get();
        RegisterParameter("bias", std::move(bias_param));
    }

    Tensor Process(const Tensor& input) {
        return input.MatMul(weight_->T()) + *bias_;
    }

private:
    Parameter* weight_;
    Parameter* bias_;
};
```

#### MultiHeadAttention


```cpp
class MultiHeadAttention : public Module {
public:
    MultiHeadAttention(int64_t n, int64_t s)
        : n_(n), head_num_(s), head_size_(n/s) {
        if (n % s != 0) {
            throw std::runtime_error("hidden_size must be divisible by num_heads");
        }

        // 初始化参数（仅修改存储方式）
        const int64_t hidden_size = n_;

        // 辅助注册函数
        auto register_param = [&](const std::string& name, Tensor tensor) {
            auto param = std::make_unique<Parameter>(tensor);
            Parameter* ptr = param.get();
            RegisterParameter(name, std::move(param));
            return ptr;
        };

        // 初始化权重
        auto init_weight = [&] {
            auto temp = Tensor::Randn({hidden_size, hidden_size});
            return temp * std::sqrt(2.0 / (hidden_size + hidden_size));
        };
        weight_q_ = register_param("weight_q", init_weight());
        weight_k_ = register_param("weight_k", init_weight());
        weight_v_ = register_param("weight_v", init_weight());
        weight_o_ = register_param("weight_o", init_weight());

        // 初始化偏置
        auto init_bias = [&] { 
            return Tensor::Zeros(std::vector<int64_t>{hidden_size}); 
        };
        bias_q_ = register_param("bias_q", init_bias());
        bias_k_ = register_param("bias_k", init_bias());
        bias_v_ = register_param("bias_v", init_bias());
        bias_o_ = register_param("bias_o", init_bias());
    }

    Tensor OpValue(const Tensor& input, const Tensor& mask = Tensor()) {
        int64_t hidden_size_ = n_;
        int64_t num_heads_ = head_num_;

        auto shape = input.Sizes();
        int64_t batch_size = shape[0];
        int64_t seq_len = shape[1]; 

        // 创建广播的目标形状 
        const std::vector<int64_t> target_sizes = {batch_size, hidden_size_, hidden_size_};

        // 1. 计算 Q、K、V，先广播权重矩阵（仅修改参数访问方式）
        auto weight_q_expanded = BroadcastTo(weight_q_->T(), target_sizes);
        auto weight_k_expanded = BroadcastTo(weight_k_->T(), target_sizes);
        auto weight_v_expanded = BroadcastTo(weight_v_->T(), target_sizes);

        auto query = input.MatMul(weight_q_expanded) + *bias_q_;
        auto key = input.MatMul(weight_k_expanded) + *bias_k_;
        auto value = input.MatMul(weight_v_expanded) + *bias_v_;

        // 2. 分离多个头
        query = query.View({batch_size, seq_len, num_heads_, head_size_}).Transpose(1, 2);
        key = key.View({batch_size, seq_len, num_heads_, head_size_}).Transpose(1, 2);
        value = value.View({batch_size, seq_len, num_heads_, head_size_}).Transpose(1, 2);

        // 3. 计算注意力分数和缩放
        auto attention_scores = query.MatMul(key.Transpose(-2,-1));
        attention_scores = attention_scores / std::sqrt(head_size_);

        // 4. 应用 mask（如果有）
        if (mask.Node()) {
            attention_scores = attention_scores + mask;
        }

        // 5. Softmax
        auto attention_probs = Softmax(attention_scores);

        // 6. 与 V 相乘并重组头
        auto context = attention_probs.MatMul(value);
        context = context.Transpose(1, 2).View({batch_size, seq_len, hidden_size_});

        // 7. 输出投影（仅修改参数访问方式）
        auto weight_o_expanded = BroadcastTo(weight_o_->T(), target_sizes);
        auto output = context.MatMul(weight_o_expanded) + *bias_o_;
        
        return output;
    }

private:
    // 维度配置
    const int64_t n_;
    const int64_t head_num_;
    const int64_t head_size_;

    // 参数指针（只有访问权限）
    Parameter* weight_q_;
    Parameter* weight_k_;
    Parameter* weight_v_;
    Parameter* weight_o_;
    Parameter* bias_q_;
    Parameter* bias_k_;
    Parameter* bias_v_;
    Parameter* bias_o_;
};
```

#### FFN

```cpp
class FFN : public Module {
public:
    FFN(int64_t input_dim, int64_t hidden_dim) {
        fc1_ = std::make_shared<Linear>(input_dim, hidden_dim);
        fc2_ = std::make_shared<Linear>(hidden_dim, input_dim);

        RegisterModule("fc1", fc1_);
        RegisterModule("fc2", fc2_);
    }

    Tensor OpValue(const Tensor& input) {
        // 获取输入的形状
        auto shape = input.Sizes();
        int64_t batch_size = shape[0];
        int64_t seq_len = shape[1];
        int64_t hidden_size = shape[2];

        // 重塑为 2D: [batch_size * seq_len, hidden_size]
        auto reshaped = input.View({batch_size * seq_len, hidden_size});

        // 前向传播
        auto hidden = fc1_->OpValue(reshaped);
        hidden = GeLU(hidden);
        auto output = fc2_->OpValue(hidden);

        // 恢复原始形状 [batch_size, seq_len, hidden_size]
        return output.View({batch_size, seq_len, hidden_size});
    }

private:
    std::shared_ptr<Linear> fc1_;
    std::shared_ptr<Linear> fc2_;
};
```

#### TransformerBlock

```cpp
class TransformerBlock : public Module {
public:
    TransformerBlock(int64_t hidden_size, int64_t num_heads) {
        // 初始化多头注意力
        attention_ = std::make_shared<MultiHeadAttention>(hidden_size, num_heads);
        
        // 初始化 FFN，隐藏层维度是输入维度的4倍
        ffn_ = std::make_shared<FFN>(hidden_size, hidden_size * 4);
        
        // 两个 Layer Normalization
        ln1_ = std::make_shared<LayerNorm>(hidden_size);
        ln2_ = std::make_shared<LayerNorm>(hidden_size);

        // 注册所有子模块
        RegisterModule("attention", attention_);
        RegisterModule("ffn", ffn_);
        RegisterModule("ln1", ln1_);
        RegisterModule("ln2", ln2_);
    }

    Tensor OpValue(const Tensor& x, const Tensor& mask = Tensor()) {
        // 1. 第一个残差连接: Attention + LayerNorm
        auto h = ln1_->OpValue(x);
        h = attention_->OpValue(h, mask);
        auto out1 = x + h;

        // 2. 第二个残差连接: FFN + LayerNorm
        h = ln2_->OpValue(out1);
        h = ffn_->OpValue(h);
        auto out2 = out1 + h;

        return out2;
    }

private:
    std::shared_ptr<MultiHeadAttention> attention_;
    std::shared_ptr<FFN> ffn_;
    std::shared_ptr<LayerNorm> ln1_;
    std::shared_ptr<LayerNorm> ln2_;
};
```

#### Gpt

```cpp
class Gpt : public Module {
public:
    Gpt(int64_t vocab_size,
        int64_t max_seq_len,
        int64_t hidden_size,
        int64_t num_layers,
        int64_t num_heads) : max_seq_len_(max_seq_len) {
        
        token_embedding_ = std::make_shared<Embedding>(vocab_size, hidden_size);

        position_embedding_ = std::make_shared<Embedding>(max_seq_len, hidden_size);

        for (int i = 0; i < num_layers; i++) {
            auto block = std::make_shared<TransformerBlock>(hidden_size, num_heads);
            transformer_blocks_.push_back(block);
            RegisterModule("block" + std::to_string(i), block);
        }

        ln_final_ = std::make_shared<LayerNorm>(hidden_size);

        RegisterModule("token_embedding", token_embedding_);
        RegisterModule("position_embedding", position_embedding_);
        RegisterModule("ln_final", ln_final_);
    } 

    Tensor OpValue(const Tensor& tokens, const Tensor& mask = Tensor()) {
        TensorOption option;
        option.SetDataType(F32);

        auto token_embeddings = token_embedding_->OpValue(tokens);
        auto shape = tokens.Sizes();
        auto pos_ids = GetPositionIds(shape[0], shape[1], option);
        auto position_embeddings = position_embedding_->OpValue(pos_ids);
        auto x = token_embeddings + position_embeddings;

        for (auto& block : transformer_blocks_) {
            x = block->OpValue(x, mask);
        }
        x = ln_final_->OpValue(x);

        auto token_weight = token_embedding_->Parameters("weight").front().second;
        auto weight_t = token_weight->T();
        auto logits = x.MatMul(weight_t);

        return logits;
    }

private:
    Tensor GetPositionIds(int64_t batch_size, int64_t seq_len, 
                          TensorOption opt) {
        std::vector<int32_t> ids(batch_size * seq_len);
        for (int64_t i = 0; i < batch_size; i++) {
            for (int64_t j = 0; j < seq_len; j++) {
                ids[i*seq_len + j] = j;
            }
        }

        return Tensor::From<int32_t>(ids, {batch_size, seq_len}, opt);
    }

    int64_t max_seq_len_;
    std::shared_ptr<Embedding> token_embedding_;
    std::shared_ptr<Embedding> position_embedding_;
    std::vector<std::shared_ptr<TransformerBlock>> transformer_blocks_;
    std::shared_ptr<LayerNorm> ln_final_;
    std::shared_ptr<Linear> lm_head_;
};
```

### 测试


```cpp
void test_gpt() {
   // 创建一个小型 GPT
    // vocab_size = 10
    // max_seq_len = 8
    // hidden_size = 6
    // num_layers = 2
    // num_heads = 2

    Gpt gpt(10, 8, 6, 2, 2);
    gpt.To(Device::CUDA());

    // 创建输入数据
    // batch_size = 2, seq_len = 4
    std::vector<int32_t> token_data = {
        0, 1, 2, 3,  // 第一个序列
        4, 5, 6, 7   // 第二个序列
    };

    TensorOption opt;
    opt.SetDataType(F32).SetDevice(Device::CUDA());

    // 创建输入张量 [2, 4]
    auto tokens = Tensor::From<int32_t>(token_data, {2, 4}, opt);
    std::cout << "Input ";
    PrintDevice(tokens.GetDevice());

    // 前向传播
    auto output = gpt.OpValue(tokens);

    std::cout << "Output" << "\n";
    // 检查输出形状 [batch_size, seq_len, vocab_size]
    for (auto s : output.Sizes()) {
        std::cout << s << " ";
    }
    std::cout << "\n\n";

    output.Cpu();
    std::cout << "Gpt forward result:\n";
    auto out_data = output.Data<float>();
    for (int b = 0; b < 2; b++) {
        std::cout << "batch " << b << ":\n";
        for (int s = 0; s < 4; s++) {
            std::cout << "  seq " << s << " (showing first 5 logits): ";
            for (int v = 0; v < 5; v++) {
                std::cout << out_data[b*40 + s*10 + v] << " ";
            }
            std::cout << "\n";
        }
    }

    // 反向传播
    output.Cuda();
    output.Backward();
    std::cout << "Backward Done\n";

    // 打印所有参数的形状
    gpt.To(Device::CPU());
    std::cout << "\nParameters:\n";
    for (const auto& [name, param] : gpt.Parameters()) {
        std::cout << name << " shape: ";
        for (auto s : param->Sizes()) {
            std::cout << s << " ";
        }
        std::cout << "\n";
    }
}
```

### 总结

Chat with Gpt 自从2022年12月推出之后，世界就改变了，至少是对我的改变，也间接造成了我23年11月的离职，这两年要说我做了什么，也许就是这个指北录了，把22年当时的一个接手任务做了，写一本NLP的书，没有离职的时候我也动手写过，却发现无从下手吧，没有自己写的代码，能写什么呢。也曾搁置过一段时间，离职后确实做了起来，但其实也进展缓慢，分析SentencePiece连CMake一关都很难过去，折腾了一个月好像就是能拆分出一些代码来，底层的Protobuf都脱离不了，也就是Craft Compiler那本书能让我进展较大。还好24年10月我买了Claude的会员，做起事来就顺利多了，两个大项目Matx和Zero都顺利完成，年后到现在也终于断断续续的把UTF-8->Regex->Trie->Tokenizer->Wapiti->LDA->W2V->Zero->Gpt这条线彻底串通了，到今天也基本上完成了初稿，虽然还有很多问题，但往后确实是小问题了。指北录的内容覆盖我想也就到此为止了，该走向下一步了，小模型的训练也该做起来了。


Zero完了是Gpt,Gpt完了呢，不管怎么样，指北录到此为止，该往南看一看了，也希望今天以后我就脱胎换骨了。
