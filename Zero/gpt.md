# GPT 训练

## 引言

GPT 是一种自回归语言模型。给定一段 Token 序列，模型根据已经出现的上下文预测下一个 Token。Token 可以表示字符、子词或其他离散单元；模型只接收整数 ID，并不依赖词表采用哪种构造方式。

这一篇将前面的张量、自动微分、模块、优化器和 GPU Kernel 连成一条完整训练链路。

**自回归目标**

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

模型在每个位置输出词表上的 Logits，再使用交叉熵最小化正确下一 Token 的负对数似然：

$$
\begin{aligned}
\mathcal{L}
&= -\frac{1}{BT}
\sum_{b=1}^{B}\sum_{t=1}^{T}
\log P(y_{b,t}\mid x_{b,1:t})
\end{aligned}
$$

\\(B\\) 是 Batch 大小，\\(T\\) 是序列长度。一次前向计算可以同时训练所有位置，但必须防止某个位置看到未来的 Token。

## 模型

GPT 接收一批 Token ID，输出每个位置对下一 Token 的概率预测。要完成这个转换，模型首先需要把离散 ID 映射为连续向量，再让每个位置从已有上下文中提取信息，最后把隐藏表示投影回词表空间。三个阶段分别对应 Embedding、Transformer Block 和输出层。

上下文建模是其中的核心。单个 Token 的向量只包含自身和位置信息，经过多头注意力后才能聚合前文；FFN 再对每个位置的表示进行非线性变换。多个 Block 重复这一过程，使当前位置逐层形成对已有序列的表示。因果掩码贯穿所有注意力层，保证这种表示始终只依赖当前位置及其之前的 Token。

**整体结构**

Zero 使用 Transformer Decoder 结构，将这些阶段连接起来：

```text
Token ID
  → Token Embedding + Position Embedding
  → Dropout
  → TransformerBlock × N
  → LayerNorm
  → Token Embedding 权重的转置
  → Logits
```

输入张量形状为 `[B, T]`，其中每个元素是一个 Token ID。经过嵌入后，张量变为 `[B, T, C]`，\\(C\\) 是隐藏维度；所有 Transformer Block 都保持这个形状，因此残差分支可以直接相加。最终输出层将隐藏维度 \\(C\\) 投影到词表维度 \\(V\\)，得到形状为 `[B, T, V]` 的 Logits。

**Embedding**

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

嵌入权重初始化为标准差 `0.02` 的正态分布。较小的初始尺度能避免与输出层权重共享时 Logits 过大，从而使 Softmax 过早饱和。

**Attention**

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

有掩码时，Zero 使用 `OpFlashAttention` 将 `Q @ K^T`、缩放掩码 Softmax 和 `probs @ V` 组织成一个自动微分算子。它不在计算图中长期保留 `[B, H, T, T]` 的 Scores 和 Probabilities，而在反向传播时重新计算，用计算换取显存。

**Transformer Block**

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

`OpFFN` 将两个线性层、Bias 和 GeLU 放在同一个算子内，减少中间张量和自动微分节点。Dropout 分别用在嵌入、注意力残差分支和 FFN 残差分支上。

**输出层**

所有 Transformer Block 之后，先应用最终 LayerNorm，再将隐藏向量投影到词表：

$$
\operatorname{logits}=H_{\mathrm{final}}E_{\mathrm{token}}^T
$$

Zero 直接复用 Token Embedding 的权重，不为输出层另外创建 `[V, C]` 参数。这种权重共享不仅减少参数量，也使输入和输出使用同一个 Token 表示空间。

普通 `OpValue` 返回 `[B, T, V]` Logits，适合推理。训练使用 `OpValueLoss`，将输出投影与交叉熵融合：

```cpp
auto x_2d = x.View({B * T, C});
return RunOp<OpLinearCrossEntropy>(
    {x_2d, *token_weight, target_flat});
```

词表较大时，`[B*T, V]` Logits 会占用大量显存。融合算子不让这个张量进入长期保留的自动微分图，可以显著降低训练峰值显存。

## 训练

训练阶段需要先将原始文本转换为整数序列，再确定模型规模和更新策略。数据批次经过模型得到每个位置的损失，反向传播后还要完成梯度裁剪、学习率调整和参数更新。

**数据准备**

原始文本首先由 Tokenizer 转换成整数 ID。训练程序不需要知道一个 ID 对应字符还是子词，只要求训练集与验证集使用同一份词表，并能把 ID 还原成 Token。

整数 ID 按 `int32` 二进制格式写入 `train.bin` 和 `val.bin`。训练时从整个 ID 序列中随机选择 \\(B\\) 个起点，每个起点连续取 \\(T+1\\) 个 ID，再错位生成输入和目标。随机起点让不同 Batch 覆盖语料中的不同位置，连续片段则保留了语言模型需要学习的上下文关系。

**训练配置**

训练程序使用一组规模较小的 GPT 配置：

| Hyperparameter | Value |
|---|---:|
| Sequence Length | 256 |
| Batch Size | 64 |
| Hidden Size | 384 |
| Transformer Blocks | 6 |
| Attention Heads | 6 |
| Dropout | 0.2 |
| AdamW \\(\beta_1,\beta_2\\) | 0.9, 0.99 |
| Weight Decay | 0.1 |
| Gradient Norm Threshold | 1.0 |
| Max / Min Learning Rate | \\(10^{-3},10^{-4}\\) |
| Warmup Steps | 100 |

**训练循环**

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

GPT 的计算图较深，个别 Batch 可能产生异常大的梯度。`ClipGradNorm` 先计算所有参数梯度共同组成的 L2 范数；当范数超过阈值 \\(c\\) 时，将所有梯度按同一比例缩小：

$$
g_p\leftarrow
g_p\cdot\frac{c}{\lVert g\rVert_2+10^{-6}}
$$

统一缩放不会改变整体梯度方向。这里将阈值设为 `1.0`，裁剪完成后再由 AdamW 读取梯度并更新参数。

学习率前 `100` 步线性增长，之后余弦衰减。每 `500` 步在验证集上估计损失，验证和采样都使用 `EvalGuard` 关闭 Dropout。

## 推理

**生成过程**

生成时取最近 \\(T\\) 个 Token 作为上下文，对最后一个位置的 Logits 应用温度缩放：

$$
p_i = \frac{\exp(z_i/\tau)}{\sum_j\exp(z_j/\tau)}
$$

实现中会先减去最大 logit，避免指数溢出，再用 `std::discrete_distribution` 根据概率采样下一 Token。温度 \\(\tau\\) 越小，分布越尖锐；温度越大，生成结果越随机。

采样前使用 `EvalGuard` 关闭 Dropout，并暂时关闭所有参数的梯度跟踪，避免为每一个生成步骤构建计算图。训练 Batch 与单样本采样的张量形状差异较大，因此采样前后还会调用 `TrimFreePool()` 清理无法复用的 CUDA 缓存桶。

当前生成路径没有 KV Cache，每产生一个 Token 都会重新计算整个上下文窗口。这个实现足以验证模型的训练结果，但不适合高吞吐推理。

## 示例

**训练 StoneGPT**

仓库中的 `prepare_stone.py` 将 UTF-8 文本转换成字符级训练数据。把语料保存为根目录下的 `FourBooks.txt`，再执行：

```bash
python3 prepare_stone.py
```

脚本收集语料中出现的所有 Unicode 字符，排序后将每个字符映射为一个整数 ID。这种方式不需要额外的子词 Tokenizer，便于直接观察训练流程；代价是序列较长，也难以直接使用更大粒度的语义单元。

脚本生成 `data/stone/vocab.txt`，并按照九比一划分训练集和验证集：

```text
data/stone/
├── vocab.txt
├── train.bin
└── val.bin
```

构建 Zero 后，可以先在 CPU 上运行少量步骤，确认数据读取、前向计算和反向传播能够连通：

```bash
./build/apps/stone_gpt
```

默认只训练 `30` 步。完整训练更适合使用 GPU，并通过命令行指定训练步数和数据目录：

```bash
./build/apps/stone_gpt cuda 5000 data/stone
```

程序每隔固定步数输出训练损失，并定期在验证集上估计损失和生成文本。训练损失反映模型对当前 Batch 的拟合程度，验证损失用于观察模型对未参与更新的数据是否同样改善，生成结果则提供更直观的检查。三者需要结合观察：训练损失下降并不必然意味着模型已经学会稳定生成，也可能只是对训练语料拟合得更好。

如果希望使用其他语料，只需生成相同格式的 `vocab.txt`、`train.bin` 和 `val.bin`，再将目录作为第三个参数传给 `stone_gpt`。模型结构和训练流程不需要改变。
