# BytePieceCounter

## 1. 引言

BytePieceCounter是BytePieceTokenizer的**训练组件**，能够从原始文本语料中学习出高质量的词汇表和概率值。

注：该算法来自苏剑林(https://kexue.fm/archives/9752)， 本项目只是 1）Python 改写成 C++;2）UTF-8约束的改进。

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

## 2. 统计

### 2.1 N-Gram语言模型

N-Gram语言模型基于马尔科夫假设：当前词的概率只依赖于前面的N-1个词。

**传统N-Gram模型**：
```
P(w₁w₂...wₘ) = ∏ P(wᵢ|wᵢ₋ₙ₊₁...wᵢ₋₁)
```

**BytePiece的创新**：
- **统计层面**：基于字节级N-Gram统计模式 (Byte)
- **建模层面**：最终构建字符级的Unigram模型 (Piece)

### 2.2 核心数据结构

BytePieceCounter使用一个关键的数据结构来存储不同长度的N-Gram统计：

```Cpp
std::vector<std::unordered_map<std::string, float_t>> N_;
```

**结构解释**：
- `N_[i]`：存储长度为i的所有子串及其统计值
- `N_[0]`：空字符串（用于归一化）
- `N_[1]`：所有1-gram（单字节/字符）
- `N_[2]`：所有2-gram（字节对/字符对）
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


### 2.3 具体实现

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

N_[3] (3-Gram/字符级)：
{E58D97:1, E4BAAC:1, E5B882:1, E995BF:1, E6B19F:1, E5A4A7:1, E6A1A5:1}
# 注意：这些正好对应UTF-8字符"南","京","市","长","江","大","桥"

N_[4] (4-Gram)：
{E58D97E4:1, E4BAACE5:1, E5B882E9:1, E995BFE6:1, E6B19FE5:1, E5A4A7E6:1}

N_[5] (5-Gram)：
{E58D97E4BA:1, E4BAACE5B8:1, E5B882E995:1, E995BFE6B1:1, E6B19FE5A4:1}

N_[6] (6-Gram)：
{E58D97E4BAAC:1, E4BAACE5B882:1, E5B882E995BF:1, E995BFE6B19F:1, E6B19FE5A4A7:1}
# 注意：这些对应字符对"南京","京市","市长","长江","江大","大桥"
```

`CountRaw`给出的是联合概率，还需要继续计算条件概率：


**目标**：计算P(C|AB) = P(ABC) / P(AB)

**对数形式**：log P(C|AB) = log P(ABC) - log P(AB)

```cpp
void PruneRaw() {
    // 确保所有256个字节都被包含
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
                pruned[k] = std::log(v);  // log P(k)
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
{E5: log(3/21), E4: log(1/21), E6: log(2/21), E9: log(1/21), ...}

N_[2]: 包含log P(byte₂|byte₁)  
{E58D: log P(8D|E5), 8D97: log P(97|8D), ...}

N_[3]: 包含log P(byte₃|byte₁byte₂)
{E58D97: log P(97|E58D), E4BAAC: log P(AC|E4BA), ...}
# 这一层特别重要，对应完整UTF-8字符的条件概率

N_[4]: 包含log P(byte₄|byte₁byte₂byte₃)
...
```

## 3. 标注

### 3.1 状态空间

**关键区别**：与BytePieceTokenizer不同，这里的状态空间更复杂：

**BytePieceTokenizer的状态**：
- 只有2个状态：0（字符边界）和1（字符内部）
- 状态转移简单：0→1（开始新字符），1→0（完成字符）

**BytePieceCounter的状态**：
- N个状态：0, 1, 2, ..., max-1
- 状态i表示：当前token的长度为i
- 状态转移复杂：受UTF-8边界和N-Gram长度限制

### 3.2 状态转移

```cpp
void InitT() {
    int num_ = max;
    T_.resize(num_, std::vector<float_t>(num_, -INF));
    
    for (int i = 0; i < num_; ++i) {
        // 从任何状态都可以转移到状态0（开始新token）
        T_[i][0] = 0;
        
        // 从状态i只能转移到状态i+1（token继续增长）
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
- **T[i][0] = 0**：任何时候都可以"切分"，结束当前token
- **T[i][i+1] = 0**：当前token可以继续增长
- **T[max-1][max-1] = 0**：★ **生成任意长度pieces的关键**


#### 自环机制：支持任意长度pieces

**数学表示**：
设max = 6，则状态转移允许：
```
状态5 → 状态5 （自环）
```

这意味着一旦token长度达到6，可以无限期保持在状态5，从而生成任意长度的piece。

**概率计算公式**：
对于长度为L (L ≥ 6) 的piece：
```
P(piece) = P(p₁) × P(p₂|p₁) × ... × P(p₆|p_{L-6})
```

其中最后的条件概率使用N_[6]中的6-gram统计：
```
P(suffix₆|prefix_{L-6}) = N_[6][last_6_chars]
```

**实际示例**：
```
假设piece = "ABCDEFGH"（长度8字节）

P(ABCDEFGH) = P(A)P(B|A)P(C|AB)P(D|ABC)P(E|ABCD)P(F|ABCDE)P(G|BCDEF)P(H|CDEFG)

分解过程：
1. 'A' → 状态0→状态1，使用N_[1]["A"]
2. 'B' → 状态1→状态2，使用N_[2]["AB"] - N_[1]["A"] 
3. 'C' → 状态2→状态3，使用N_[3]["ABC"] - N_[2]["AB"]
4. 'D' → 状态3→状态4，使用N_[4]["ABCD"] - N_[3]["ABC"]  
5. 'E' → 状态4→状态5，使用N_[5]["ABCDE"] - N_[4]["ABCD"]
6. 'F' → 状态5→状态5，使用N_[6]["ABCDEF"] - N_[5]["ABCDE"] ★ 自环
7. 'G' → 状态5→状态5，使用N_[6]["BCDEFG"] - N_[5]["BCDEF"] ★ 自环  
8. 'H' → 状态5→状态5，使用N_[6]["CDEFGH"] - N_[5]["CDEFG"] ★ 自环
```

**关键洞察**：
- 虽然N-Gram统计只到6-Gram，但通过状态5的自环机制
- 算法可以使用最后6个字符的条件概率持续评估更长的piece
- 这实现了在有限统计基础上支持无限长度token的生成

**状态转移表示例**（max=6）：
```
T矩阵（简化表示，0表示允许转移，-∞表示不允许）：

    →  0  1  2  3  4  5
从 ↓ 
 0     -∞ 0  -∞ -∞ -∞ -∞
 1     0  -∞ 0  -∞ -∞ -∞  
 2     0  -∞ -∞ 0  -∞ -∞
 3     0  -∞ -∞ -∞ 0  -∞
 4     0  -∞ -∞ -∞ -∞ 0
 5     0  -∞ -∞ -∞ -∞ 0  ★ 自环允许无限增长
 ```

### 3.3 具体实现

BytePieceCounter的核心是一个复杂的动态规划算法，它需要在字节级统计基础上实现字符级分词，因而要引入UTF-8边界约束。


#### UTF-8位置预处理

虽然N-Gram统计是字节级的，但最终分词必须是字符级的。算法首先检测UTF-8字符边界：

```Cpp
// UTF-8位置预处理：标记每个字节在UTF-8字符中的位置
std::vector<int> utf8_position(num, 0);
int i = 0;
while (i < num) {
    unsigned char c = static_cast<unsigned char>(text[i]);
    int char_length = SizeUTF8(c);
    
    // 标记UTF-8字符的每个字节位置
    for (int j = 0; j < char_length && i + j < num; ++j) {
        utf8_position[i + j] = j;  // 0=首字节, 1=第二字节, 2=第三字节
    }
    i += char_length;
}
```

**UTF-8位置标记示例**：
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


#### 约束机制总体设计

基于utf8_position数组，算法采用了以下约束策略：

**1. 状态有效性约束**
- 在UTF-8字符的非首字节位置，禁止进入较小的状态
- 具体措施：跳过不符合条件的状态，不填充对应的scores[i][j]

**举例说明**：
```
在位置1（字节8D，utf8_position[1]=1）：
- 状态0：禁止 ✗ （状态0 < 1，意味着从UTF-8字符中间开始新token）
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

**3. N-gram边界约束**
- 确保N-gram的起始位置是UTF-8字符的首字节
- 具体措施：检查ngram_start位置，如果不是首字节则跳过该转移

**举例说明**：
```
在位置3，考虑状态2（当前token长度=3）：
ngram_start = 3 - 2 = 1

检查：utf8_position[1] = 1 ≠ 0
结论：N-gram起始位置不是UTF-8首字节，跳过此状态

原因：如果允许，会产生N-gram "8D97E4"，这跨越了UTF-8字符边界
正确的应该是从位置0开始的"E58D97"或从位置3开始的"E4BA..."
```

**4. 切分点约束**
- 最终的token边界必须在UTF-8字符边界上
- 具体措施：在回溯时只在utf8_position[i]==0的位置设置切分点

**举例说明**：
```
回溯得到的状态序列：opt_route = [1, 2, 0, 1, 2, 0]
位置：                           [0, 1, 2, 3, 4, 5]
utf8_position：                  [0, 1, 2, 0, 1, 2]

切分点候选：
- 位置2：状态=0 但 utf8_position[2]=2 ✗ （UTF-8字符中间，禁止切分）
- 位置3：状态=1 ✗ （状态非0，不是token边界）
- 位置5：状态=0 但 utf8_position[5]=2 ✗ （UTF-8字符中间，禁止切分）

实际切分点：只有位置0和文本结束位置6
结果：整个"南京"作为一个token
```

这些约束通过**跳过无效状态和转移**来实现，确保算法只考虑合法的分词方案。


#### 动态规划框架

基于上述约束机制，动态规划算法的整体结构如下：

```Cpp
std::vector<std::string> Tokenize(const std::string& text) const {
    const int num = text.length();
    if (num == 0) return {};
    
    // 1. UTF-8位置预处理（已完成）
    std::vector<int> utf8_position = PreprocessUTF8(text);
    
    // 2. 节点评分矩阵：scores[i][j] = 在字节位置i处于状态j的得分
    std::vector<std::vector<float_t>> scores(num, 
        std::vector<float_t>(max, -INF));
    
    // 3. 路径记录矩阵
    std::vector<std::vector<int>> routes(num - 1, 
        std::vector<int>(max, 0));
```

**核心思想**：寻找一条穿越状态空间的最优路径，使得总概率最大化。

#### 节点评分填充（应用约束1）

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
                    scores[i][j] = it->second;  // 使用N-Gram概率
                }
            }
        }
    }
```

**约束1效果**：
- 位置1（UTF-8第二字节）：只填充状态≥1的scores，状态0保持-INF
- 位置2（UTF-8第三字节）：只填充状态≥2的scores
- 位置3（新字符开始）：可以填充任何状态的scores

其实这一步也可以看成是把合理的N-Gram概率取出来。

#### 动态规划状态转移（应用约束2和3）

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
                
                // 约束3：N-gram边界检查
                int ngram_start = i - curr_j;
                if (ngram_start > 0 && utf8_position[ngram_start] > 0) {
                    continue;  // 跳过从UTF-8字符中间开始的N-gram
                }
                
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

#### 关键约束详解

**约束1 - 状态有效性**：`curr_j < utf8_position[i]`
```
含义：如果当前字节是UTF-8的第k字节，那么状态必须≥k
原因：状态j表示当前token长度为j+1，如果j<k，意味着token长度小于当前UTF-8字符的字节位置，
      这会导致token从UTF-8字符中间开始，违反字符完整性
```

**约束2 - 转移路径**：`prev_j < utf8_position[i-1]`
```
含义：前一位置的状态也必须满足相同的UTF-8约束
原因：确保状态转移路径的连续性和合法性
```

**约束3 - N-Gram边界**：`utf8_position[ngram_start] > 0`
```
含义：N-Gram的起始位置必须是UTF-8字符的首字节（utf8_position=0）
原因：防止N-Gram跨越字符边界，确保语义完整性
```

#### 最优路径回溯（应用约束4）

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
    
    // 7. 根据路径提取tokens（应用约束4）
    std::vector<int> split_points;
    split_points.push_back(0);
    
    for (int i = 1; i < opt_route.size(); ++i) {
        // 约束4：只在UTF-8首字节处切分
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

## 4. 裁剪

### 4.1 迭代策略

```Cpp
Str2Int PrunePieces(Str2Int& pieces) {
    Str2Int keep, drop;
    
    // 第一轮过滤：按长度和频率
    for (const auto& [str, cnt] : pieces) {
        if (str.length() == 1 ||  // 保留所有单字符（保证完备性）
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
    
    // 迭代直到收敛
    while (true) {
        size_t n = keep.size();
        auto entire_keep_as_drop = keep;
        keep = SplitPieces(keep, entire_keep_as_drop);  // 自我重分词
        
        if (keep.size() == n) break;  // 收敛：词汇表大小不再变化
    }
    
    return FinalSelection(keep);  // 最终筛选到目标大小
}
```

### 4.2 核心机制

```Cpp
Str2Int SplitPieces(const Str2Int& keep, const Str2Int& drop) {
    // 1. 基于keep构建临时Unigram分词器
    std::unordered_map<std::string, float_t> dict;
    for (const auto& p : keep) {
        dict.emplace(p.first, static_cast<float_t>(p.second));
    }
    BytePieceTokenizer tokenizer(dict);  // 使用完整的分词器！
    
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

### 4.3 收敛性

**为什么算法会收敛？**

1. **单调性**：每次迭代词汇表大小单调递减或保持不变
2. **下界**：至少保留256个字节pieces和单字符pieces
3. **稳定性**：当SplitPieces的输入输出相同时，算法收敛

**收敛过程**：
```
迭代0：pieces = {所有N-gram}，大小=100,000
迭代1：剪枝低频 → 大小=50,000
迭代2：重分词后剪枝 → 大小=30,000
迭代3：继续剪枝 → 大小=25,000
迭代4：大小=25,000（不变）→ 收敛！