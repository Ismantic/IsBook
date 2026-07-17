# Tokenizer: BytePiece

## 引言

BytePieceCounter 是 PieceTokenizer 的训练组件，用于从原始语料中生成候选 piece、估计分数并裁剪词表。

本文的基本算法参考了[苏剑林的实现](https://kexue.fm/archives/9752)，本项目将其改写为 C++，并增加 UTF-8 切分边界约束。

**核心任务：**

**输入**：大量原始文本语料
**输出**：训练好的Unigram语言模型，包含：
- 词汇表：各种粒度的subword pieces
- 概率值：每个piece的出现概率


**基本思路：**
- **统计阶段**：收集文本中所有字节级N-gram的统计信息
- **标注阶段**：使用动态规划找到最优分词方案
- **剪枝阶段**：移除低频或冗余的词汇，优化词汇表大小
- **迭代收敛**：重复上述过程直到词汇表稳定

注：BytePieceCounter在训练过程中的剪枝阶段需要使用BytePieceTokenizer。

## 统计

**N-Gram 统计**

N-Gram 模型基于有限阶马尔可夫假设：当前符号的概率只依赖于前面的 \\(N-1\\) 个符号。这里的符号是字节，不是词。

**传统N-Gram模型**：
```
P(w₁w₂...wₘ) = ∏ P(wᵢ|wᵢ₋ₙ₊₁...wᵢ₋₁)
```

BytePieceCounter 先统计字节级 N-gram，再利用这些局部条件概率给候选 piece 评分。最终词表中的 piece 必须在 UTF-8 字符边界切分，但 piece 内部的滑动 N-gram 可以从任意字节位置开始。

**核心数据结构**

BytePieceCounter使用一个关键的数据结构来存储不同长度的N-Gram统计：

```Cpp
std::vector<std::unordered_map<std::string, float_t>> N_;
```

**结构解释**：
- `N_[i]`：存储长度为 `i` 字节的所有子串及其统计值
- `N_[0]`：空字符串（用于归一化）
- `N_[1]`：所有 1-gram（单字节）
- `N_[2]`：所有 2-gram（字节对）
- `N_[3]`：所有3-gram（三字节组合）
- ...
- `N_[max]`：最长统计的N-gram（通常max=6）

**示例初始化**：
```Cpp
N_.clear();
N_.resize(max + 1);  // 通常max = 6
N_[0][""] = 0;  // 空字符串初始化
```

**为什么选择字节级N-Gram?**
1. **语言无关性**：任何UTF-8文本都能统一处理
2. **完备性**：保证100%覆盖，不存在未知字符
3. **细粒度模式**：能够发现字符内部和跨字符的统计规律


**具体实现**

```Cpp
void CountRaw(const std::vector<std::string>& sentences) {
    // 初始化N_数组
    N_.clear();
    N_.resize(max + 1);
    N_[0][""] = 0;  // 空字符串计数

    // 对每个文本的每个位置，统计所有可能长度的子串
    for (const auto& text : sentences) {
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
UTF-8字节序列：[E5,8D,97,E4,BA,AC,E5,B8,82,E9,95,BF,E6,B1,9F,E5,A4,A7,E6,A1,A5]
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

`CountRaw` 得到的是出现次数，不是概率。随后用相邻阶的计数比估计条件概率：


**目标**：计算P(C|AB) = P(ABC) / P(AB)

**对数形式**：log P(C|AB) = log P(ABC) - log P(AB)

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

    // 从最长N-gram开始向下处理
    for (int i = N_.size() - 1; i >= 0; --i) {
        std::unordered_map<std::string, float_t> pruned;

        // 1. 频率过滤 + 对数概率转换
        for (const auto& [k, v] : N_[i]) {
            if (k.length() == i && v >= (i > 1 ? min_count_ : 0)) {
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
# 这一层特别重要，对应完整UTF-8 字符的条件概率

N_[4]: 包含log P(byte₄|byte₁byte₂byte₃)
...
```

## 标注

**状态空间**

**关键区别**：与BytePieceTokenizer不同，这里的状态空间更复杂：

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

**关键洞察**：
- 虽然N-Gram统计只到6-Gram，但通过状态5的自环机制
- 算法可以使用最近 6 个字节的条件概率持续评估更长的 piece
- 这实现了在有限统计基础上支持无限长度token的生成

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

BytePieceCounter 的核心是一个复杂的动态规划算法，它需要在字节级统计基础上实现字符级分词，因而要引入UTF-8边界约束。


**UTF-8 位置预处理**

虽然N-Gram统计是字节级的，但最终分词必须是字符级的。算法首先检测UTF-8 字符边界：

```Cpp
// UTF-8 位置预处理：标记每个字节在UTF-8 字符中的位置
std::vector<int> utf8_position(num, 0);
int i = 0;
while (i < num) {
    unsigned char c = static_cast<unsigned char>(text[i]);
    int char_length = SizeUTF8(c);

    // 标记UTF-8 字符的每个字节位置
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


**约束机制总体设计**

基于utf8_position数组，算法采用了以下约束策略：

**1. 状态有效性约束**
- 在UTF-8 字符的非首字节位置，禁止进入较小的状态
- 具体措施：跳过不符合条件的状态，不填充对应的scores[i][j]

**举例说明**：
```
在位置1（字节8D，utf8_position[1]=1）：
- 状态0：禁止 ✗ （状态0 < 1，意味着从UTF-8 字符中间开始新token）
- 状态1：允许 ✓ （状态1 ≥ 1，当前token长度=2，覆盖了首字节E5和当前字节8D）
- 状态2：允许 ✓ （状态2 ≥ 1，当前token长度=3，可能包含更多前缀）

在位置3（字节E4，utf8_position[3]=0）：
- 状态0,1,2,3...：都允许 ✓ （新字符开始，任何状态都合法）
```

**2. 转移路径约束**
- 在状态转移时，检查前后位置的UTF-8约束
- 具体措施：在DP转移循环中continue跳过不合法的转移

**举例说明**：
```
从位置2到位置3的状态转移：
位置2：utf8_position[2]=2 → 位置3：utf8_position[3]=0

合法转移：
- 状态2 → 状态0：允许 ✓ （状态2≥2，且可以转移到任何状态）
- 状态2 → 状态3：允许 ✓ （状态2≥2，token继续增长）

非法转移（会被跳过）：
- 状态1 → 任何状态：禁止 ✗ （位置2的状态1<2，不满足UTF-8约束）
- 状态0 → 任何状态：禁止 ✗ （位置2的状态0<2，不满足UTF-8约束）
```

**3. 切分点约束**
- 最终的token边界必须在UTF-8 字符边界上
- 具体措施：在回溯时只在utf8_position[i]==0的位置设置切分点

注意，UTF-8 约束只限制 token 的起点和终点，不限制 piece 内部滑动 N-gram 的起点。跨越字符边界的字节窗口仍然是合法的统计上下文。


**动态规划框架**

基于上述约束机制，动态规划算法的整体结构如下：

```Cpp
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

**节点评分填充（应用约束1）**

```Cpp
    // 3. 填充节点评分（基于N-Gram统计）
    for (int j = 0; j < max; ++j) {
        for (int i = j; i < num; ++i) {
            // 约束1：状态有效性约束
            if (j < utf8_position[i]) continue;  // 跳过无效状态

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

**约束1效果**：
- 位置1（UTF-8第二字节）：只填充状态≥1的scores，状态0保持-INF
- 位置2（UTF-8第三字节）：只填充状态≥2的scores
- 位置3（新字符开始）：可以填充任何状态的scores

其实这一步也可以看成是把合理的N-gram 概率取出来。

**动态规划状态转移（应用约束2）**

关键是过滤掉不合理的转移 （某些状态不需要转移以及某些状态之间不能转移）。

```Cpp
    // 4. 动态规划核心：寻找最优路径
    for (int i = 1; i < num; ++i) {
        for (int curr_j = 0; curr_j < max; ++curr_j) {
            // 约束1：当前状态的UTF-8约束检查
            if (curr_j < utf8_position[i]) continue;

            int best_prev_j = -1;
            float_t best_score = -INF;

            for (int prev_j = 0; prev_j < max; ++prev_j) {
                // 约束2：前一位置的UTF-8约束
                if (prev_j < utf8_position[i-1]) continue;

                // 状态转移约束（基于T矩阵）
                if (T_[prev_j][curr_j] == -INF) continue;
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

**关键约束详解**

**约束1 - 状态有效性**：`curr_j < utf8_position[i]`
```
含义：如果当前字节是UTF-8的第k字节，那么状态必须≥k
原因：状态j表示当前token长度为j+1，如果j<k，意味着token长度小于当前UTF-8 字符的字节位置，
      这会导致token从UTF-8 字符中间开始，违反字符完整性
```

**约束2 - 转移路径**：`prev_j < utf8_position[i-1]`
```
含义：前一位置的状态也必须满足相同的UTF-8约束
原因：确保状态转移路径的连续性和合法性
```

**最优路径回溯（应用约束3）**

```Cpp
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

    // 7. 根据路径提取tokens（应用约束3）
    std::vector<int> split_points;
    split_points.push_back(0);

    for (int i = 1; i < opt_route.size(); ++i) {
        // 约束3：只在UTF-8首字节处切分
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

**迭代策略**

```Cpp
Str2Int PrunePieces(Str2Int& pieces) {
    Str2Int keep, drop;

    // 第一轮过滤：按长度和频率
    for (const auto& [str, cnt] : pieces) {
        if (str.length() == 1 ||  // 保留全部单字节piece（保证字节回退）
            (str.length() <= max_piece_size_ && cnt >= min_count_)) {
            keep[str] = cnt;
        } else {
            drop[str] = cnt;  // 标记为丢弃
        }
    }

    // 重分词被丢弃的pieces
    auto new_counter = SplitPieces(keep, drop);
    for (const auto& [str, cnt] : new_counter) {
        keep[str] += cnt;  // 更新保留pieces的频率
    }

    // 迭代到达不动点，并设置上限避免异常振荡
    for (int iteration = 0; iteration < max_iterations_; ++iteration) {
        auto entire_keep_as_drop = keep;
        auto next = SplitPieces(keep, entire_keep_as_drop);
        if (next == keep) break;
        keep = std::move(next);
    }

    return FinalSelection(keep);  // 最终筛选到目标大小
}
```

**核心机制**

```Cpp
Str2Int SplitPieces(const Str2Int& keep, const Str2Int& drop) {
    // 1. 基于keep构建临时Unigram分词器
    std::unordered_map<std::string, float_t> dict;
    double total = 0.0;
    for (const auto& [piece, cnt] : keep) {
        total += cnt;
    }
    if (total <= 0.0) return {};

    for (const auto& p : keep) {
        dict.emplace(
            p.first,
            static_cast<float_t>(p.second / total)
        );
    }
    BytePieceTokenizer tokenizer(dict);

    // 2. 用临时分词器重新分词drop中的内容
    Str2Int counter;
    for (const auto& [str, cnt] : drop) {
        auto tokens = tokenizer.Tokenize(str);  // 基于词汇表的分词
        for (const auto& token : tokens) {
            counter[token] += cnt;  // 统计重分词后的频率
        }
    }

    return counter;
}
```

**示例**：
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

**收敛性**

每轮重分词产生的 piece 都来自当前词表，因此候选集合不会扩张。但仅比较词表大小并不能判断收敛：词条集合或计数可能在大小不变时继续变化。实现应比较完整结果是否达到不动点，同时设置最大迭代次数作为保护。

**收敛过程**：
```
迭代0：pieces = {所有N-gram}，大小=100,000
迭代1：剪枝低频 → 大小=50,000
迭代2：重分词后剪枝 → 大小=30,000
迭代3：继续剪枝 → 大小=25,000
迭代4：词条及计数均不再变化 → 收敛
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

若 `min_count = 2`，则“南京市”和“京市”进入待裁剪集合。临时 Unigram 分词器使用保留词表重新切分它们：

```text
南京市 → 南京 / 市
京市   → 京 / 市
```

相应计数被转移到仍然保留的 piece。完成所有重分词并达到不动点后，使用完整保留词表的计数和 `Z` 归一化：

```text
P(piece) = count(piece) / Z
Z = Σ count(piece)
```

最终写入模型的是 piece 及其概率；256 个单字节 piece 始终保留，负责处理词表没有覆盖的输入。至此，字节 N-gram 统计、最优路径标注、词表裁剪和 Unigram 概率生成形成了完整闭环。
