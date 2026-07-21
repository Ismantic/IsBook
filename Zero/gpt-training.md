# GPT 训练

GPT 是一种自回归语言模型。给定一段 Token 序列，模型根据已经出现的上下文预测下一个 Token。Zero 的 `stone_gpt` 使用字符级词表，将文本中每个 Unicode 码点映射为一个整数 ID，并使用一个小型 GPT 学习字符序列。

这个示例将前面的张量、自动微分、模块、优化器和 GPU kernel 连成一条完整训练链路。

## 自回归目标

对序列 \\(x_1,x_2,\ldots,x_T\\)，联合概率可以按条件概率展开：

$$
P(x_1,x_2,\ldots,x_T) = \prod_{t=1}^{T}P(x_t\mid x_1,\ldots,x_{t-1})
$$

训练数据通过错位一个 Token 构造输入和目标：

```text
原序列：  [x0, x1, x2, x3, x4]
输入 x：   [x0, x1, x2, x3]
目标 y：   [x1, x2, x3, x4]
```

模型在每个位置输出词表上的 logits，再使用交叉熵最小化正确下一 Token 的负对数似然：

$$
\begin{aligned}
\mathcal{L}
&= -\frac{1}{BT}
\sum_{b=1}^{B}\sum_{t=1}^{T}
\log P(y_{b,t}\mid x_{b,1:t})
\end{aligned}
$$

\\(B\\) 是 batch 大小，\\(T\\) 是序列长度。一次前向计算可以同时训练所有位置，但必须防止某个位置看到未来的 Token。

## 模型结构

Zero 实现的 GPT 使用 Transformer Decoder 结构：

```text
Token ID
  → Token Embedding + Position Embedding
  → Dropout
  → TransformerBlock × N
  → LayerNorm
  → Token Embedding 权重的转置
  → logits
```

输入张量形状为 `[B, T]`，经过嵌入后变为 `[B, T, C]`，其中 \\(C\\) 是隐藏维度。最终 logits 形状为 `[B, T, V]`，\\(V\\) 是词表大小。

## 嵌入

Token ID 是离散整数，不能直接参与神经网络运算。`Embedding` 使用索引从参数矩阵中选取对应行：

$$
H_0=E_{\mathrm{token}}(x)+E_{\mathrm{position}}(0,1,\ldots,T-1)
$$

Token 嵌入矩阵形状为 `[V, C]`，位置嵌入矩阵形状为 `[T_max, C]`。自注意力本身不区分 Token 的先后顺序，因此必须额外加入位置信息。

```cpp
auto token_embeddings = token_embedding_->OpValue(tokens);
auto pos_ids = GetPositionIds(batch_size, seq_len, option);
auto position_embeddings = position_embedding_->OpValue(pos_ids);
auto x = dropout_emb_.OpValue(
    token_embeddings + position_embeddings);
```

嵌入权重初始化为标准差 `0.02` 的正态分布。较小的初始尺度能避免与输出层权重共享时 logits 过大，从而使 Softmax 过早饱和。

## 多头因果注意力

对输入 \\(X\in\mathbb{R}^{B\times T\times C}\\)，首先做三次线性投影：

$$
Q=XW^Q+b^Q,
\qquad
K=XW^K+b^K,
\qquad
V=XW^V+b^V
$$

为了让投影转化为一次大矩阵乘法，Zero 先将 `[B, T, C]` 展平为 `[B*T, C]`，再使用融合的 `OpLinearBias`。得到 Q、K、V 后，将它们重塑为多头布局：

```text
[B*T, C]
  → View [B, T, H, D]
  → Transpose [B, H, T, D]
```

其中 \\(C=H\times D\\)，\\(H\\) 是注意力头数，\\(D\\) 是单头维度。每个头的缩放点积注意力为：

$$
\operatorname{Attention}(Q,K,V) = \operatorname{softmax}\left(
\frac{QK^T}{\sqrt{D}}+M
\right)V
$$

\\(M\\) 是因果掩码。对于位置 \\(i\\) 和 \\(j\\)：

$$
M_{ij}=
\begin{cases}
0, & j\le i \\\\
-10^4, & j>i
\end{cases}
$$

对角线上方的大负数经 Softmax 后接近零，使第 \\(i\\) 个位置只能使用自己和之前的上下文：

```text
       x0  x1  x2  x3
x0     ✓   ×   ×   ×
x1     ✓   ✓   ×   ×
x2     ✓   ✓   ✓   ×
x3     ✓   ✓   ✓   ✓
```

有掩码时，Zero 使用 `OpFlashAttention` 将 `Q @ K^T`、缩放掩码 Softmax 和 `probs @ V` 组织成一个自动微分算子。它不在计算图中长期保留 `[B, H, T, T]` 的 scores 和 probabilities，而在反向传播时重新计算，用计算换取显存。

## Transformer Block

Zero 使用 Pre-LN 结构，两个子层都在输入端做 LayerNorm，再通过残差连接加回主干：

$$
\begin{aligned}
H' &= H+\operatorname{Dropout}(
       \operatorname{Attention}(\operatorname{LN}_1(H))) \\\\
H'' &= H'+\operatorname{Dropout}(
        \operatorname{FFN}(\operatorname{LN}_2(H')))
\end{aligned}
$$

残差连接为梯度提供直接路径。LayerNorm 放在子层之前，可以在较深的网络中改善训练稳定性。

FFN 对每个 Token 位置独立执行两次线性变换，中间维度是隐藏维度的四倍：

$$
\operatorname{FFN}(x) = \operatorname{GeLU}(xW_1+b_1)W_2+b_2
$$

`OpFFN` 将两个线性层、bias 和 GeLU 放在同一个算子内，减少中间张量和自动微分节点。Dropout 分别用在嵌入、注意力残差分支和 FFN 残差分支上。

## 输出层

所有 Transformer Block 之后，先应用最终 LayerNorm，再将隐藏向量投影到词表：

$$
\operatorname{logits}=H_{\mathrm{final}}E_{\mathrm{token}}^T
$$

Zero 直接复用 Token Embedding 的权重，不为输出层另外创建 `[V, C]` 参数。这种权重共享不仅减少参数量，也使输入和输出使用同一个 Token 表示空间。

普通 `OpValue` 返回 `[B, T, V]` logits，适合推理。训练使用 `OpValueLoss`，将输出投影与交叉熵融合：

```cpp
auto x_2d = x.View({B * T, C});
return RunOp<OpLinearCrossEntropy>(
    {x_2d, *token_weight, target_flat});
```

词表较大时，`[B*T, V]` logits 会占用大量显存。融合算子不让这个张量进入长期保留的自动微分图，可以显著降低训练峰值显存。

## 数据准备

字符级分词器首先收集语料中出现的所有 Unicode 字符，排序后建立字符到 ID 的映射：

```python
vocab = sorted(set(text))
stoi = {ch: i for i, ch in enumerate(vocab)}
ids = [stoi[ch] for ch in text]
```

整数 ID 按 `int32` 二进制格式写入 `train.bin` 和 `val.bin`，前 \\(90\%\\) 用于训练，后 \\(10\%\\) 用于验证。训练时从整个 ID 序列中随机选择 \\(B\\) 个起点，每个起点连续取 \\(T+1\\) 个 ID，再错位生成输入和目标。

```bash
python3 prepare_stone.py
```

这种字符级建模方式不需要额外 Tokenizer，便于将重点放在训练引擎本身；代价是序列较长，并且字符级模型难以直接学习更大粒度的语义单元。

## 训练配置

`stone_gpt` 的主要超参与 nanoGPT `shakespeare_char` 配置对齐：

| 超参 | 值 |
|---|---:|
| 序列长度 | 256 |
| batch size | 64 |
| 隐藏维度 | 384 |
| Transformer Block | 6 |
| 注意力头 | 6 |
| Dropout | 0.2 |
| AdamW \\(\beta_1,\beta_2\\) | 0.9, 0.99 |
| weight decay | 0.1 |
| 梯度范数阈值 | 1.0 |
| 最大/最小学习率 | \\(10^{-3},10^{-4}\\) |
| warmup | 100 steps |

每一步的核心训练过程为：

```cpp
opt->ZeroGrad();

auto per_token_loss = model->OpValueLoss(x, mask, y);
auto loss = per_token_loss.Mean(0, false);
loss.Backward();

float grad_norm = ClipGradNorm(all_params, 1.0f);
opt->SetLr(lr_for_iter(iter));
opt->Step();
```

学习率前 `100` 步线性增长，之后余弦衰减。每 `500` 步在验证集上估计损失，验证和采样都使用 `EvalGuard` 关闭 Dropout。

运行方式为：

```bash
# CPU
./build/apps/stone_gpt

# GPU，训练 5000 步，使用 Shakespeare 数据
./build/apps/stone_gpt cuda 5000 data/shakespeare
```

## 生成

生成时取最近 \\(T\\) 个 Token 作为上下文，对最后一个位置的 logits 应用温度缩放：

$$
p_i = \frac{\exp(z_i/\tau)}{\sum_j\exp(z_j/\tau)}
$$

实现中会先减去最大 logit，避免指数溢出，再用 `std::discrete_distribution` 根据概率采样下一 Token。温度 \\(\tau\\) 越小，分布越尖锐；温度越大，生成结果越随机。

采样前使用 `EvalGuard` 关闭 Dropout，并暂时关闭所有参数的梯度跟踪，避免为每一个生成步骤构建计算图。训练 batch 与单样本采样的张量形状差异较大，因此采样前后还会调用 `TrimFreePool()` 清理无法复用的 CUDA 缓存桶。

当前生成路径没有 KV Cache，每产生一个 Token 都会重新计算整个上下文窗口。这个实现足以验证模型的训练结果，但不适合高吞吐推理。
