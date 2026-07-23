# Tokenizer：BytePiece

## 引言

BytePieceCounter 是 PieceTokenizer 的训练组件，用于从原始语料中生成候选 piece、统计权重并裁剪词表。

本文的基本算法参考了[苏剑林的实现](https://kexue.fm/archives/9752)，本项目将其改写为 C++，并增加 UTF-8 切分边界约束。

**核心任务：**

**输入**：大量原始文本语料
**输出**：训练好的 BytePiece 模型，包含：
- 词汇表：不同粒度的 subword pieces
- 计数权重：每个 piece 经重分词后得到的频次


**基本思路：**
- **统计阶段**：收集文本中所有字节级 N-gram 的统计信息
- **标注阶段**：使用动态规划找到最优分词方案
- **剪枝阶段**：移除低频或冗余的词汇，优化词汇表大小
- **重分词阶段**：用保留词表重新切分候选，回收被裁剪 piece 的计数

BytePieceCounter 在训练过程中的剪枝阶段需要使用 BytePieceTokenizer 对被裁剪片段重新切分。

## 统计

**N-gram 统计**

N-gram 模型基于有限阶马尔可夫假设：当前符号的概率只依赖于前面的 $N-1$ 个符号。这里的符号是字节，不是词。

$$
\begin{aligned}
P(w_1w_2\ldots w_m)
&= \prod_i P(w_i\mid w_{i-N+1}\ldots w_{i-1})
\end{aligned}
$$

BytePieceCounter 先统计字节级 N-gram，再利用这些局部条件概率给候选 piece 评分。最终词表中的 piece 必须在 UTF-8 字符边界切分，但 piece 内部的滑动 N-gram 可以从任意字节位置开始。

**核心数据结构**

BytePieceCounter 使用一组哈希表存储不同长度的 N-gram 统计：

```cpp
std::vector<std::unordered_map<std::string, float_t>> N_;
```

**结构解释**：
- `N_[i]`：存储长度为 `i` 字节的所有子串及其统计值
- `N_[0]`：空字符串（用于归一化）
- `N_[1]`：所有 1-gram（单字节）
- `N_[2]`：所有 2-gram（字节对）
- `N_[3]`：所有 3-gram（三字节组合）
- ...
- `N_[6]`：最长统计的 N-gram

**示例初始化**：
```cpp
N_.clear();
N_.resize(max_piece_count_ + 1);  // max_piece_count_ = 6
N_[0][""] = 0;  // 空字符串初始化
```

**为什么选择字节级 N-gram？**
1. **语言无关性**：任何 UTF-8 文本都能统一处理
2. **完备性**：保证 100% 覆盖，不存在未知字符
3. **细粒度模式**：能够发现字符内部和跨字符的统计规律


**具体实现**

```cpp
void CountRawSegments(const std::vector<std::string>& segments) {
    // 初始化N_数组
    N_.clear();
    N_.resize(max + 1);
    N_[0][""] = 0;  // 空字符串计数

    // 对每个文本的每个位置，统计所有可能长度的子串
    for (const auto& text : segments) {
        for (size_t i = 0; i < text.length(); ++i) {
            for (size_t j = 0; j <= max; ++j) {
                if (i + j <= text.length()) {
                    std::string k = text.substr(i, j);
                    N_[j][k] += 1;  // 长度为j的子串k的计数+1
                }
            }
        }
    }
}
```

**统计示例**：
```
文本："南京市长江大桥"
UTF-8 字节序列：[E5,8D,97,E4,BA,AC,E5,B8,82,E9,95,BF,E6,B1,9F,E5,A4,A7,E6,A1,A5]
总字节数：21

填充N_数组：
N_[0]: {"": 21}  # 在21个字节的文本中，空字符串在每个位置都出现一次

N_[1] (1-Gram/单字节)：
{E5:3, 8D:1, 97:1, E4:1, BA:1, AC:1, B8:1, 82:1, E9:1, 95:1, BF:1,
 E6:2, B1:1, 9F:1, A4:1, A7:1, A1:1, A5:1}

N_[2] (2-Gram/字节对)：
{E58D:1, 8D97:1, 97E4:1, E4BA:1, BAAC:1, ACE5:1, E5B8:1, B882:1,
 82E9:1, E995:1, 95BF:1, BFE6:1, E6B1:1, B19F:1, 9FE5:1, E5A4:1,
 A4A7:1, A7E6:1, E6A1:1, A1A5:1}

N_[3]（3-Gram，以下只列从字符边界开始的部分）：
{E58D97:1, E4BAAC:1, E5B882:1, E995BF:1, E6B19F:1, E5A4A7:1, E6A1A5:1}
# 这些条目对应"南","京","市","长","江","大","桥"
# 实际统计还包含8D97E4、97E4BA等从字符内部开始的滑动窗口

N_[4]（4-Gram，节选）：
{E58D97E4:1, E4BAACE5:1, E5B882E9:1, E995BFE6:1, E6B19FE5:1, E5A4A7E6:1}

N_[5]（5-Gram，节选）：
{E58D97E4BA:1, E4BAACE5B8:1, E5B882E995:1, E995BFE6B1:1, E6B19FE5A4:1}

N_[6]（6-Gram，节选）：
{E58D97E4BAAC:1, E4BAACE5B882:1, E5B882E995BF:1, E995BFE6B19F:1, E6B19FE5A4A7:1}
# 这里列出的条目对应字符对"南京","京市","市长","长江","江大","大桥"
# 实际统计同样包含从字符内部开始的6字节滑动窗口
```

实际的 `StreamCountRaw` 会分批读取语料，先经过 PreTokenizer，再由多个线程执行上面的统计核心。得到的是出现次数，不是概率；随后用相邻阶的计数比估计条件概率：


**目标**：计算 $P(C\mid AB)=P(ABC)/P(AB)$

**对数形式**：$\log P(C\mid AB)=\log P(ABC)-\log P(AB)$

```cpp
void PruneRaw() {
    // 为语料中未出现的字节加入回退伪计数
    for (int i = 0; i < 256; ++i) {
        std::string byte_str(1, static_cast<char>(i));
        if (N_[1].find(byte_str) == N_[1].end()) {
            N_[1][byte_str] = 1;
            N_[0][""] += 1;
        }
    }

    // 从最长 N-gram 开始向下处理
    for (int i = N_.size() - 1; i >= 0; --i) {
        std::unordered_map<std::string, float_t> pruned;

        // 1. 频率过滤 + 对数概率转换
        for (const auto& [k, v] : N_[i]) {
            if (k.length() == i &&
                v >= (i > 1 ? counter_spec_.min_count() : 0)) {
                pruned[k] = std::log(v);  // 先保存对数计数
            }
        }

        // 2. 计算条件概率
        if (i < N_.size() - 1) {
            std::unordered_map<std::string, float_t> next_pruned;
            for (const auto& [k, v] : N_[i + 1]) {
                std::string prefix = k.substr(0, i);  // 前i个字符
                auto it = pruned.find(prefix);
                if (it != pruned.end()) {
                    // log P(k|prefix) = log P(k) - log P(prefix)
                    next_pruned[k] = v - it->second;
                }
            }
            N_[i + 1] = std::move(next_pruned);
        }

        N_[i] = std::move(pruned);
    }
}
```

**结果示例**：
```
修剪后的N_数组（对数概率形式）：

N_[1]: 包含log P(byte)
{E5: log(3/259), E4: log(1/259), E6: log(2/259), E9: log(1/259), ...}
# 本例有18种已出现字节，另外238种字节各加入1次伪计数，因此分母为21+238=259

N_[2]: 包含log P(byte₂|byte₁)
{E58D: log P(8D|E5), 8D97: log P(97|8D), ...}

N_[3]: 包含log P(byte₃|byte₁byte₂)
{E58D97: log P(97|E58D), E4BAAC: log P(AC|E4BA), ...}
# 这一层特别重要，对应完整 UTF-8 字符的条件概率

N_[4]: 包含log P(byte₄|byte₁byte₂byte₃)
...
```

## 标注

**状态空间**

这里的状态空间服务于模型训练，与最终 BytePieceTokenizer 使用的词图动态规划不同：

**BytePieceCounter 的状态**：
- 共有 `max` 个状态：0, 1, 2, ..., max-1
- 状态 `j < max-1` 表示当前 token 已连续包含 `j+1` 个字节
- 状态 `max-1` 是饱和状态，表示当前 token 至少包含 `max` 个字节
- 从任意状态转移到状态 0，表示在当前字节之前切分，并以当前字节开始新 token

**状态转移**

```cpp
void InitT() {
    int num_ = max;
    T_.resize(num_, std::vector<float_t>(num_, -INF));

    for (int i = 0; i < num_; ++i) {
        // 转移到状态0：在下一个字节开始新token
        T_[i][0] = 0;

        // 转移到状态i+1：当前token继续增长
        if (i + 1 < num_) {
            T_[i][i + 1] = 0;
        }

        // 最高状态可以自环（保持最大长度）★ 关键设计
        if (i == num_ - 1) {
            T_[i][i] = 0;
        }
    }
}
```

**转移规则解释**：
- **T[i][0] = 0**：在下一个字节开始新 token
- **T[i][i+1] = 0**：当前 token 继续增长
- **T[max-1][max-1] = 0**：在饱和状态使用滑动 N-gram 继续给长 token 评分


**自环机制：支持任意长度 piece**

**数学表示**：
设max = 6，则状态转移允许：
```
状态5 → 状态5 （自环）
```

这意味着 token 长度达到 6 字节后可以保持在状态 5，并使用长度为 6 的滑动窗口继续评分。

**概率计算公式**：
对于字节序列 \\(x_1,\ldots,x_L\\)，当 \\(L\ge 6\\) 时：
```
P(piece) ≈ P(x₁)P(x₂|x₁)...P(x₆|x₁...x₅)
           × ∏ₜ₌₇ᴸ P(xₜ|xₜ₋₅...xₜ₋₁)
```

长度超过 6 后，每一步都用最近 6 个字节的计数除以其 5 字节前缀计数，估计下一字节的条件概率。

**实际示例**：
```
假设piece = "ABCDEFGH"（长度8字节）

P(ABCDEFGH) = P(A)P(B|A)P(C|AB)P(D|ABC)P(E|ABCD)P(F|ABCDE)P(G|BCDEF)P(H|CDEFG)

分解过程：
1. 'A' → 初始状态0，使用N_[1]["A"]
2. 'B' → 状态0→状态1，使用N_[2]["AB"] - N_[1]["A"]
3. 'C' → 状态1→状态2，使用N_[3]["ABC"] - N_[2]["AB"]
4. 'D' → 状态2→状态3，使用N_[4]["ABCD"] - N_[3]["ABC"]
5. 'E' → 状态3→状态4，使用N_[5]["ABCDE"] - N_[4]["ABCD"]
6. 'F' → 状态4→状态5，使用N_[6]["ABCDEF"] - N_[5]["ABCDE"]
7. 'G' → 状态5→状态5，使用N_[6]["BCDEFG"] - N_[5]["BCDEF"] ★ 自环
8. 'H' → 状态5→状态5，使用N_[6]["CDEFGH"] - N_[5]["CDEFG"] ★ 自环
```

虽然 N-gram 统计只到 6-gram，但状态 5 的自环可以使用滑动窗口继续评估更长的 piece。piece 最终仍会受到 `max_piece_size` 的裁剪约束，默认不超过 18 字节。

**状态转移表示例**（max=6）：
```
T矩阵（简化表示，0表示允许转移，-∞表示不允许）：

    →  0  1  2  3  4  5
从 ↓
 0     0  0  -∞ -∞ -∞ -∞
 1     0  -∞ 0  -∞ -∞ -∞
 2     0  -∞ -∞ 0  -∞ -∞
 3     0  -∞ -∞ -∞ 0  -∞
 4     0  -∞ -∞ -∞ -∞ 0
 5     0  -∞ -∞ -∞ -∞ 0  ★ 自环允许无限增长
```

**具体实现**

BytePieceCounter 的核心是一个动态规划算法：在字节级统计的基础上实现字符级切分，因此需要引入 UTF-8 边界约束。


**UTF-8 位置预处理**

虽然 N-gram 统计是字节级的，但最终切分必须保持字符完整。算法首先检测 UTF-8 字符边界：

```cpp
// UTF-8 位置预处理：标记每个字节在 UTF-8 字符中的位置
std::vector<int> utf8_position(num, 0);
int i = 0;
while (i < num) {
    int char_length = ustr::OneUTF8Size(text.data() + i);

    // 标记 UTF-8 字符的每个字节位置
    for (int j = 0; j < char_length && i + j < num; ++j) {
        utf8_position[i + j] = j;  // 0=首字节, 1=第二字节, 2=第三字节
    }
    i += char_length;
}
```

**UTF-8 位置标记示例**：
```
文本："南京"
字节：[E5, 8D, 97, E4, BA, AC]
位置： 0   1   2   3   4   5
utf8_position: [0, 1, 2, 0, 1, 2]
                ↑     ↑
              字符边界  字符边界

解释：
- 位置0,1,2：属于字符"南"，分别是第1,2,3字节
- 位置3,4,5：属于字符"京"，分别是第1,2,3字节
- 只有utf8_position[i]==0的位置是字符边界，可以作为切分点
```


**UTF-8 约束**

`utf8_position[i]` 记录字节 `i` 在当前 UTF-8 字符中的偏移。状态转移需要满足三条规则：

1. 状态 0 只能出现在字符首字节，避免从字符内部开始 token。
2. 当前状态和前一状态不能小于对应的字符内偏移，保证路径覆盖完整字符。
3. 普通 N-gram 的起点必须位于字符边界；状态 5 自环使用滑动 6-gram 时不受此限制。

因此，piece 的起止位置始终落在 UTF-8 字符边界，而长 piece 内部的滑动窗口仍可跨越字符边界。

**转移示例**

仍以“南京”为例。状态从 0 开始，表示当前 token 已包含一个字节：

```text
位置：       0   1   2   3   4   5
字节：      E5  8D  97  E4  BA  AC
字符内偏移： 0   1   2   0   1   2
```

处理“南”的三个字节时，合法路径依次为：

```text
位置 0：状态 0，token 从 E5 开始
位置 1：状态 0 → 1，继续读入 8D
位置 2：状态 1 → 2，继续读入 97
```

位置 1 不能进入状态 0，否则 token 会从续字节 `8D` 开始；位置 2 也不能处于状态 0 或 1，因为这样的 token 无法覆盖“南”的完整三个字节。

来到位置 3，即“京”的首字节 `E4`，有两种合法选择：

```text
状态 2 → 0：在“南”和“京”之间切分
状态 2 → 3：不切分，让当前 token 继续增长
```

第一条路径产生“南 / 京”，第二条路径则可能产生“南京”。两条路径都会进入动态规划，由累计 N-gram 得分决定最终结果。反过来，若一个普通窗口的起点落在 `8D` 或 `97` 这样的续字节上，即使状态编号能够连接，也会被 `ngram_start` 检查排除。

当 token 达到 6 字节后，状态进入 5。若继续读取第三个汉字，状态 5 可以自环，此时 6-gram 窗口会向前滑动，起点允许落在字符内部；这只是评分窗口移动，token 本身仍从原来的字符边界开始。


**动态规划框架**

基于上述约束机制，动态规划算法的整体结构如下：

```cpp
std::vector<std::string> Tokenize(const std::string& text) const {
    const int num = text.length();
    if (num == 0) return {};

    // 1. UTF-8 位置预处理（已完成）
    std::vector<int> utf8_position = PreprocessUTF8(text);

    // 2. 节点评分矩阵：scores[i][j] = 在字节位置i处于状态j的得分
    std::vector<std::vector<float_t>> scores(num,
        std::vector<float_t>(max, -INF));

    // 3. 路径记录矩阵
    std::vector<std::vector<int>> routes(num - 1,
        std::vector<int>(max, 0));
```

**核心思想**：寻找一条穿越状态空间的最优路径，使得总概率最大化。

**节点评分填充**

```cpp
// 3. 填充节点评分（基于 N-gram 统计）
    for (int j = 0; j < max; ++j) {
        for (int i = j; i < num; ++i) {
            // 状态 0 只能从 UTF-8 字符边界开始
            if (j == 0 && utf8_position[i] > 0) continue;

            std::string piece = text.substr(i - j, j + 1);
            if (j + 1 < N_.size()) {
                auto it = N_[j + 1].find(piece);
                if (it != N_[j + 1].end()) {
                    scores[i][j] = it->second;  // 使用N-gram 概率
                }
            }
        }
    }
```

这一步提取可用的 N-gram 得分，并排除从 UTF-8 字符内部开始的新 token。更完整的状态合法性在随后的转移阶段检查。

**动态规划状态转移**

关键是过滤掉不合理的转移 （某些状态不需要转移以及某些状态之间不能转移）。

```cpp
// 4. 动态规划核心：寻找最优路径
    for (int i = 1; i < num; ++i) {
        for (int curr_j = 0; curr_j < max; ++curr_j) {
            // 当前状态的 UTF-8 约束检查
            if (curr_j < utf8_position[i]) continue;

            int best_prev_j = -1;
            float_t best_score = -INF;

            for (int prev_j = 0; prev_j < max; ++prev_j) {
                // 前一位置的 UTF-8 约束
                if (prev_j < utf8_position[i-1]) continue;

                // 状态转移约束（基于T矩阵）
                if (T_[prev_j][curr_j] == -INF) continue;

                // 普通窗口必须从字符边界开始；饱和状态自环除外
                bool sliding = prev_j == max - 1 && curr_j == max - 1;
                int ngram_start = i - curr_j;
                if (!sliding && ngram_start > 0 &&
                    utf8_position[ngram_start] > 0) continue;

                // 计算转移得分
                float_t score = scores[i-1][prev_j] + T_[prev_j][curr_j] + scores[i][curr_j];

                if (score > best_score) {
                    best_score = score;
                    best_prev_j = prev_j;
                }
            }

            if (best_prev_j != -1) {
                routes[i-1][curr_j] = best_prev_j;
                scores[i][curr_j] = best_score;
            } else {
                scores[i][curr_j] = -INF;  // 无有效转移路径
            }
        }
    }
```

`curr_j` 和 `prev_j` 的检查保证状态覆盖当前字符已经经过的字节数；`ngram_start` 的检查则让非滑动窗口从字符边界开始。这两类约束分别负责路径合法性和候选 piece 的起点合法性。

**最优路径回溯**

```cpp
// 5. 找到最后位置的最佳状态
    int best_last_state = 0;
    float_t best_score = -INF;
    for (int j = 0; j < max; ++j) {
        if (j >= utf8_position[num - 1] && scores[num - 1][j] > best_score) {
            best_score = scores[num - 1][j];
            best_last_state = j;
        }
    }

    // 6. 回溯构建最优路径
    std::vector<int> opt_route(num);
    int curr_pos = num - 1;
    int curr_state = best_last_state;

    while (curr_pos >= 0) {
        opt_route[curr_pos] = curr_state;
        if (curr_pos > 0) {
            curr_state = routes[curr_pos-1][curr_state];
            curr_pos--;
        } else {
            break;
        }
    }

    // 7. 根据路径提取tokens
    std::vector<int> split_points;
    split_points.push_back(0);

    for (int i = 1; i < opt_route.size(); ++i) {
        // 只在 UTF-8 首字节处切分
        if (opt_route[i] == 0 && utf8_position[i] == 0) {
            split_points.push_back(i);
        }
    }
    split_points.push_back(num);

    // 8. 构建最终token序列
    std::vector<std::string> tokens;
    for (size_t i = 0; i < split_points.size() - 1; ++i) {
        tokens.push_back(text.substr(split_points[i],
                                   split_points[i + 1] - split_points[i]));
    }

    return tokens;
}
```

## 裁剪

第一次标注会产生大量候选 piece。`PrunePieces` 先按 `max_piece_size` 和 `min_count` 分成保留集合与裁剪集合，再把被裁剪 piece 的计数重新分配给保留词表：

```cpp
for (const auto& [piece, count] : pieces) {
    if (piece.length() <= counter_spec_.max_piece_size() &&
        count >= counter_spec_.min_count()) {
        keep[piece] = count;
    } else {
        drop[piece] = count;
    }
}

for (const auto& [piece, count] : SplitPieces(keep, drop)) {
    keep[piece] += count;
}
```

`SplitPieces` 用保留集合构建临时 BytePieceTokenizer。构造函数会把计数归一化为对数概率，然后用最大概率路径重新切分待裁剪内容：

```cpp
Str2Int SplitPieces(const Str2Int& keep, const Str2Int& drop) {
    std::unordered_map<std::string, float_t> dict;
    for (const auto& [piece, count] : keep) {
        dict.emplace(piece, static_cast<float_t>(count));
    }
    BytePieceTokenizer tokenizer(dict);

    Str2Int counter;
    for (const auto& [piece, count] : drop) {
        for (const auto& token : tokenizer.Tokenize(piece)) {
            counter[token] += count;
        }
    }
    return counter;
}
```

随后，算法反复用当前词表切分其全部 piece，直到词表大小不再变化。最后若候选数仍超过目标 `vocab_size`，则排序截断：单字节候选优先，其余候选主要按计数降序排列。这里的单字节候选是普通候选；真正保证任意输入可编码的是模型初始化时单独加入的 256 个 `BYTE` 类型元词条。

**计数再分配示例**

```
假设keep = {"南京", "市", "长江", "大桥"}
     drop = {"南京市", "市长江", "长江大桥"}

重分词过程：
"南京市" → tokenizer.Tokenize("南京市") → ["南京", "市"]
"市长江" → tokenizer.Tokenize("市长江") → ["市", "长江"]
"长江大桥" → tokenizer.Tokenize("长江大桥") → ["长江", "大桥"]

结果统计：
counter = {"南京":1, "市":2, "长江":2, "大桥":1}

最终更新：
keep["南京"] += 1  # 原频率 + 重分词贡献
keep["市"] += 2
keep["长江"] += 2
keep["大桥"] += 1
```

## 示例

下面用“南京”串联统计、标注和裁剪过程。为便于展示，令 `max = 3`，状态 0、1、2 分别表示当前 token 已包含 1、2、至少 3 个字节。状态 2 是饱和状态，可以通过自环继续扩展 token。

**统计结果**

“南京”的 UTF-8 字节序列为：

```text
位置：  0   1   2   3   4   5
字节： E5  8D  97  E4  BA  AC
边界：  ✓           ✓
```

假设语料统计得到以下对数条件概率：

```text
log P(E5)          = -1.0
log P(8D | E5)     = -0.2
log P(97 | E5 8D)  = -0.2
log P(E4)          = -1.1
log P(BA | E4)     = -0.2
log P(AC | E4 BA)  = -0.2

log P(E4 | 8D 97)  = -0.1
log P(BA | 97 E4)  = -0.1
```

前三项描述“南”，中间三项可以独立描述“京”，最后两项用于状态 2 自环后跨越两个字符的滑动窗口。窗口可以从字符内部开始，但 token 只能在位置 0、3、6 切分。

**比较候选路径**

路径一将两个汉字分别作为 piece：

```text
切分：  南 | 京
状态：  0 1 2 | 0 1 2
得分：(-1.0 - 0.2 - 0.2) + (-1.1 - 0.2 - 0.2)
     = -2.9
```

路径二让状态 2 保持自环，将“南京”作为一个 piece：

```text
切分：  南京
状态：  0 1 2 2 2 2
窗口： E5 8D 97
       8D 97 E4
       97 E4 BA
       E4 BA AC
得分：-1.0 - 0.2 - 0.2 - 0.1 - 0.1 - 0.2
     = -1.8
```

因为 `-1.8 > -2.9`，动态规划选择“南京”。回溯得到的状态序列是：

```text
位置： 0 1 2 3 4 5
状态： 0 1 2 2 2 2
```

序列中没有再次出现状态 0，因此位置 0 到文本末尾构成一个完整 token。

**裁剪与计数再分配**

假设对小型语料完成标注后得到：

```text
南京    8
京      4
市      5
南京市  1
京市    1
```

若 `min_count = 2`，则“南京市”和“京市”进入待裁剪集合。临时 BytePieceTokenizer 使用保留词表重新切分它们：

```text
南京市 → 南京 / 市
京市   → 京 / 市
```

相应计数被转移到仍然保留的 piece。重分词完成后，模型保存 piece 及其计数权重；BytePieceTokenizer 加载词表时再进行归一化：

```text
P(piece) = count(piece) / Z
Z = Σ count(piece)
```

此外，模型会单独保存 256 个 `BYTE` 类型元词条。普通词表未覆盖某段输入时，编码过程会回退到这些字节词条，从而保证任意字节序列都可表示。至此，字节 N-gram 统计、最优路径标注、词表裁剪和字节回退形成了完整闭环。

配套实现：[Ismantic/PieceTokenizer](https://github.com/Ismantic/PieceTokenizer)
