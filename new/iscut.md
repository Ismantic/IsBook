# 中文分词：基础篇

## 方法概述

本篇介绍基于 Unigram 模型的分词方法。BPE 按照预先学习到的合并规则构造切分，而 Unigram 模型会为每个候选 piece 分配概率，并在所有可行切分中选择联合概率最高的一条路径。这里的概率反映 piece 的整体使用频率，并不直接建模上下文，因此它能够处理一部分由词频造成的切分歧义，但不能替代真正的上下文语言模型。

算法需要一个训练好的 Unigram 词表。下面只列出完整词表中的一小部分：

```Cpp
// 词汇表：包含各种粒度的subword pieces
std::unordered_map<std::string, float> vocab = {
    // 地名和常用词汇
    {"南京", 0.0751},
    {"市", 0.1003},
    {"长江", 0.0602},
    {"大桥", 0.0399},

    // 可能的歧义词汇
    {"市长", 0.0150},
    {"江大桥", 0.0025},

    // 单字符（作为回退）
    {"南", 0.0148},
    {"京", 0.0081},
    {"长", 0.0181},
    {"江", 0.0200},
    {"大", 0.0296},
    {"桥", 0.0050},

    // 其他常用符号
    {"▁", 0.05},         // 5%的概率（空格标记）
    {"。", 0.02},        // 2%的概率
    {"的", 0.03},        // 3%的概率
};
```

完整词表中的概率已经由训练过程归一化，不能再除以当前示例条目的概率和。实现时只需转换到对数域：

```Cpp
// 转换为对数概率（避免数值下溢）
std::unordered_map<std::string, float> log_probs;
for (const auto& [piece, prob] : vocab) {
    log_probs[piece] = std::log(prob);
}

// 结果示例：
// "南京": log(0.0751) ≈ -2.59
// "市": log(0.1003) ≈ -2.30
// "长江": log(0.0602) ≈ -2.81
// "大桥": log(0.0399) ≈ -3.22
// "市长": log(0.0150) ≈ -4.20
// "江大桥": log(0.0025) ≈ -5.99
```

**分词目标**：对给定输入文本，找到使得整体概率最大的分词方案：

```
tokens* = argmax P(tokens) = argmax ∏ P(tokenᵢ)
```

由于概率连乘容易数值下溢，实际使用对数概率：
```
tokens* = argmax ∑ log P(tokenᵢ)
```


**具体例子**：
对于文本"南京市长江大桥"，可能的分词方案：

1. **正确分词**：`["南京", "市", "长江", "大桥"]`
   → score = -2.59 + (-2.30) + (-2.81) + (-3.22) = **-10.92**

2. **歧义分词**：`["南京", "市长", "江大桥"]`
   → score = -2.59 + (-4.20) + (-5.99) = **-12.78**

3. **字符级**：`["南", "京", "市", "长", "江", "大", "桥"]`
   → score = -4.21 + (-4.81) + (-2.30) + (-4.01) + (-3.91) + (-3.52) + (-5.30) = **-28.06**

显然方案1得分最高（-10.92），因此算法会自动选择正确的分词`["南京", "市", "长江", "大桥"]`，成功避免了"市长"的歧义陷阱。

## 动态规划

具体的分词算法是动态规划，把分词问题转换为**最佳路径搜索**问题：

- **状态定义**：`scores[i]` 表示文本前 `i` 个字节的最优分词得分；只有 UTF-8 字符边界才是有效状态
- **转移方程**：`scores[i] = max(scores[j] + log P(S[j:i]))` for all valid j < i
- **目标**：求解 `scores[num]` 对应的最佳分词路径


**详细示例**

用经典的歧义例子"南京市长江大桥"来详细演示DP如何自动选择最佳分词：

**准备工作：词汇表设置**

```Cpp
// 包含多种分词可能的词汇表
词汇表及对数概率：
"南京": -2.59     // 地名，高频
"市": -2.30       // 常用字，高频
"长江": -2.81     // 地名，高频
"大桥": -3.22     // 建筑物，较高频
"市长": -4.20     // 职位，较低频
"江大桥": -5.99   // 罕见组合，低频

// 单字符
"南": -4.21, "京": -4.81, "长": -4.01, "江": -3.91, "大": -3.52, "桥": -5.30
```

**文本分析**

```
输入文本: "南京市长江大桥"
UTF-8字符边界: 南(3) 京(3) 市(3) 长(3) 江(3) 大(3) 桥(3)
字节位置: 0-3-6-9-12-15-18-21
位置索引: 0  3  6  9  12 15 18 21
```

**第一步：初始化DP数组**

```Cpp
const int num = 21;  // 文本字节长度
std::vector<float> scores(num + 1, -INF);
std::vector<int> routes(num + 1);

初始状态：
位置:   0    3    6    9   12   15   18   21
scores: 0.0 -∞   -∞   -∞   -∞   -∞   -∞   -∞
routes:  0    3    6    9   12   15   18   21
字符:   [   南   京   市   长   江   大   桥   ]
```

**第二步：获取全部候选片段**

```Cpp
候选片段列表：
// 位置0开始
Match{start=0, end=3, score=-4.21}   // "南"
Match{start=0, end=6, score=-2.59}   // "南京"

// 位置3开始
Match{start=3, end=6, score=-4.81}   // "京"

// 位置6开始
Match{start=6, end=9, score=-2.30}   // "市"
Match{start=6, end=12, score=-4.20}  // "市长"

// 位置9开始
Match{start=9, end=12, score=-4.01}  // "长"
Match{start=9, end=15, score=-2.81}  // "长江"

// 位置12开始
Match{start=12, end=15, score=-3.91} // "江"
Match{start=12, end=21, score=-5.99} // "江大桥"

// 位置15开始
Match{start=15, end=18, score=-3.52} // "大"
Match{start=15, end=21, score=-3.22} // "大桥"

// 位置18开始
Match{start=18, end=21, score=-5.30} // "桥"
```

**第三步：逐步动态规划状态转移**

**处理位置0的匹配**

```Cpp
// 处理"南" Match{start=0, end=3, score=-4.21}
新得分 = scores[0] + (-4.21) = 0.0 + (-4.21) = -4.21
scores[3] = -4.21, routes[3] = 0

// 处理"南京" Match{start=0, end=6, score=-2.59}
新得分 = scores[0] + (-2.59) = 0.0 + (-2.59) = -2.59
scores[6] = -2.59, routes[6] = 0

状态更新：
位置:   0    3    6    9   12   15   18   21
scores: 0.0 -4.21 -2.59  -∞   -∞   -∞   -∞   -∞
routes:  0    0    0    9   12   15   18   21
来源:   [   南   南京   ]
```

**处理位置3和6的匹配**

```Cpp
// 处理"京" Match{start=3, end=6, score=-4.81}
新得分 = scores[3] + (-4.81) = -4.21 + (-4.81) = -9.02
-9.02 < scores[6](-2.59) ✗ 不更新（南京路径更好）

// 处理"市" Match{start=6, end=9, score=-2.30}
新得分 = scores[6] + (-2.30) = -2.59 + (-2.30) = -4.89
scores[9] = -4.89, routes[9] = 6

// 处理"市长" Match{start=6, end=12, score=-4.20}
新得分 = scores[6] + (-4.20) = -2.59 + (-4.20) = -6.79
scores[12] = -6.79, routes[12] = 6

状态更新：
位置:   0    3    6    9   12   15   18   21
scores: 0.0 -4.21 -2.59 -4.89 -6.79  -∞   -∞   -∞
routes:  0    0    0    6    6   15   18   21
来源:   [   南   南京  市   市长  ]
```

**处理位置9的匹配**

```Cpp
// 处理"长" Match{start=9, end=12, score=-4.01}
新得分 = scores[9] + (-4.01) = -4.89 + (-4.01) = -8.90
-8.90 < scores[12](-6.79) ✗ 不更新（市长路径暂时更好）

// 处理"长江" Match{start=9, end=15, score=-2.81}
新得分 = scores[9] + (-2.81) = -4.89 + (-2.81) = -7.70
scores[15] = -7.70, routes[15] = 9

状态更新：
位置:   0    3    6    9   12   15   18   21
scores: 0.0 -4.21 -2.59 -4.89 -6.79 -7.70  -∞   -∞
routes:  0    0    0    6    6    9   18   21
来源:   [   南   南京  市   市长  长江  ]
```

**处理位置12和15的匹配**

```Cpp
// 处理"江" Match{start=12, end=15, score=-3.91}
新得分 = scores[12] + (-3.91) = -6.79 + (-3.91) = -10.70
-10.70 < scores[15](-7.70) ✗ 不更新

// 处理"江大桥" Match{start=12, end=21, score=-5.99}
新得分 = scores[12] + (-5.99) = -6.79 + (-5.99) = -12.78
scores[21] = -12.78, routes[21] = 12  // 临时设置

// 处理"大" Match{start=15, end=18, score=-3.52}
新得分 = scores[15] + (-3.52) = -7.70 + (-3.52) = -11.22
scores[18] = -11.22, routes[18] = 15

// 处理"大桥" Match{start=15, end=21, score=-3.22}  ⭐ 关键匹配
新得分 = scores[15] + (-3.22) = -7.70 + (-3.22) = -10.92
-10.92 > scores[21](-12.78) ✓ 更新！

更新后状态：
位置:   0    3    6    9   12   15   18   21
scores: 0.0 -4.21 -2.59 -4.89 -6.79 -7.70 -11.22 -10.92
routes:  0    0    0    6    6    9    15   15
来源:   [   南   南京  市   市长  长江  大   大桥]
```

**处理位置18的匹配**

```Cpp
// 处理"桥" Match{start=18, end=21, score=-5.30}
新得分 = scores[18] + (-5.30) = -11.22 + (-5.30) = -16.52
-16.52 < scores[21](-10.92) ✗ 不更新

最终状态：
位置:   0    3    6    9   12   15   18   21
scores: 0.0 -4.21 -2.59 -4.89 -6.79 -7.70 -11.22 -10.92
routes:  0    0    0    6    6    9    15   15
```

**第四步：回溯最优路径**

```Cpp
std::vector<std::string> tokens;
int e = 21;  // 从文本末尾开始

回溯过程：
e = 21 → start = routes[21] = 15
       → token = text.substr(15, 21-15) = "大桥" (位置15-21)
       → tokens = ["大桥"]
       → e = 15

e = 15 → start = routes[15] = 9
       → token = text.substr(9, 15-9) = "长江" (位置9-15)
       → tokens = ["长江", "大桥"]
       → e = 9

e = 9  → start = routes[9] = 6
       → token = text.substr(6, 9-6) = "市" (位置6-9)
       → tokens = ["市", "长江", "大桥"]
       → e = 6

e = 6  → start = routes[6] = 0
       → token = text.substr(0, 6-0) = "南京" (位置0-6)
       → tokens = ["南京", "市", "长江", "大桥"]
       → e = 0

e = 0  → 结束

正确分词结果：["南京", "市", "长江", "大桥"]
总得分：-2.59 + (-2.30) + (-2.81) + (-3.22) = -10.92
```

**歧义消解分析**

让我们对比两种主要的分词路径：

**路径1（正确）："南京" + "市" + "长江" + "大桥"**
```
得分：-2.59 + (-2.30) + (-2.81) + (-3.22) = -10.92
```

**路径2（歧义）："南京" + "市长" + "江大桥"**
```
得分：-2.59 + (-4.20) + (-5.99) = -12.78
```

**比较**：-10.92 > -12.78，算法正确选择了路径1。

**关键洞察**：
- 虽然"市长"是一个完整词汇，但其概率较低（-4.20）
- "市"（-2.30）+ "长江"的开头"长"比"市长"有更好的后续发展
- 全局动态规划考虑了完整路径，而非局部贪心选择

**具体实现**

```Cpp
function Tokenize(text):
    scores[0] = 0
    for i in 1 to text.length():
        scores[i] = -∞

    matches = GetMatches(text)  // Trie搜索获取所有候选

    for match in matches:  // 必须按 start 从小到大处理
        start = match.start
        end = match.end + 1

        new_score = scores[start] + match.score
        if new_score > scores[end]:
            scores[end] = new_score
            routes[end] = start

    return Backtrack(routes, text)
```


**第一步：初始化**

```Cpp
const int num = sentence.length();
std::vector<float_t> scores(num + 1, -INF);  // 分词得分
std::vector<int> routes(num + 1);            // 路径记录
scores[0] = 0;  // 空字符串得分为0
for (int i = 0; i <= num; i++) {
    routes[i] = i;  // 初始路径
}
```

**第二步：获取候选片段**

这是算法的关键步骤。对于每个位置，需要找到所有以该位置为起点的有效词汇片段。

```Cpp
struct Match {
    int e;      // 片段结束位置
    int n;      // 片段长度
    float_t w;  // 片段得分（对数概率，通常小于零）
};

auto matches = GetMatches(sentence);
```

**Trie 树的作用**：Trie 能从指定位置沿字节逐步匹配，并返回所有有效的前缀 piece。单次状态转移可以在常数时间内完成，但完整查询仍与最长候选长度及返回结果数有关，并不是 \\(O(1)\\)。

**第三步：动态规划状态转移**

```Cpp
for (const auto& m : matches) {
    int start = m.e - m.n + 1;  // 片段起始位置
    int end = m.e + 1;          // 片段结束位置

    if (start < 0 || start >= scores.size() || end >= scores.size()) {
        continue;  // 边界检查
    }

    float_t score = scores[start] + m.w;
    if (score > scores[end]) {
        scores[end] = score;
        routes[end] = start;
    }
}
```

**核心思想**：对于每个候选片段`sentence[start:end]`，我们检查是否通过这个片段能获得更好的分词得分。

**第四步：回溯最佳路径**

```Cpp
std::vector<std::string> tokens;
int e = num;
while (e > 0) {
    int start = routes[e];
    tokens.push_back(sentence.substr(start, e - start));
    e = start;
}
std::reverse(tokens.begin(), tokens.end());
return tokens;
```

## Trie 应用

**前缀匹配**

Trie树（具体实现为DoubleArray Trie）的核心功能是**批量前缀搜索**：

```cpp
std::vector<Match> GetMatches(const std::string& sentence) const {
    std::vector<Match> matches;
    int num = sentence.length();
    int pos = 0;

    while (pos < num) {
        // 使用Trie树进行前缀搜索
        // 同一起点至多产生 max_piece_bytes_ 个不同长度的前缀
        std::vector<darts::DoubleArray<int>::ResultPair> results(
            max_piece_bytes_ + 1
        );

        size_t num_results = trie_.GetUpPieces(
            sentence.c_str() + pos,    // 从当前位置开始
            results.data(),            // 结果数组
            results.size(),            // 不截断合法候选
            num - pos                 // 剩余字符数
        );

        // 处理所有匹配的前缀
        for (size_t i = 0; i < num_results; ++i) {
            if (pos + results[i].length - 1 < num) {
                matches.emplace_back(
                    pos + results[i].length - 1,        // 结束位置
                    results[i].length,                   // 长度
                    value_map_.at(results[i].value)      // 对数概率
                );
            }
        }

        pos += SizeUTF8(sentence[pos]);  // 移动到下一个UTF-8字符
    }

    return matches;
}
```

注意：通过 `GetUpPieces` 调用一次性返回全部有效前缀。

**Trie树创建**

```Cpp
void InitFromDict(const std::unordered_map<std::string, float_t>& dict) {
    std::vector<std::pair<std::string, float_t>> entries(
        dict.begin(), dict.end()
    );
    std::sort(entries.begin(), entries.end(),
              [](const auto& a, const auto& b) {
                  return a.first < b.first;
              });

    // Double-Array Trie 通常要求键按字典序排列。
    std::vector<std::string> keys;
    keys.reserve(entries.size());
    for (const auto& [piece, prob] : entries) {
        keys.push_back(piece);
        max_piece_bytes_ = std::max(max_piece_bytes_, piece.size());
    }

    std::vector<const char*> strs;
    std::vector<int> values;
    strs.reserve(keys.size());
    values.reserve(keys.size());
    int next_value = 1;

    for (size_t i = 0; i < entries.size(); ++i) {
        strs.push_back(keys[i].c_str());
        value_map_[next_value] = std::log(entries[i].second);
        values.push_back(next_value);
        next_value++;
    }

    // 构建DoubleArray Trie
    trie_.build(strs.size(), strs.data(), nullptr, values.data());
}
```

## 回退机制

当最终得到的 token 不在输出词表中时，BytePieceTokenizer 采用**字节级回退**策略。只要词表包含全部 256 个字节 piece，这套实现就不需要输出 UNK：

```Cpp
EncodeResult output;
for (const auto& token : tokens) {
    int i = PieceID(token);

    if (i == unk_id_) {
        // 字节级回退：将token拆分为单个字节
        for (size_t i = 0; i < token.size(); ++i) {
            const uint8_t byte = static_cast<uint8_t>(token[i]);
            std::string byte_piece = ustr::ByteToPiece(byte);  // 如"<byte_65>"
            int byte_id = PieceID(byte_piece);
            output.emplace_back(byte_piece, byte_id);
        }
    } else {
        output.emplace_back(std::string(token), i);
    }
}
```

这是 BytePieceTokenizer 的设计目标，并不是所有 Tokenizer 都必须满足的通用要求：
- **完备性**：任何输入都能被编码（通过256个字节pieces）
- **可逆性**：编码结果可以完全解码回原文本
- **可恢复性**：编码阶段可能拆开 UTF-8 字符的字节，但解码后仍能无损恢复原文本

## UTF-8

在GetMatches过程中，算法会添加UTF-8字符级的候选：

```Cpp
while (pos < num) {
    // 为每个UTF-8字符添加候选
    int n = SizeUTF8(sentence[pos]);
    if (pos + n - 1 < num) {
        matches.emplace_back(pos + n - 1, n, -10.0);  // UTF-8字符惩罚
    }

    // ... Trie搜索 ...

    pos += n;  // 按UTF-8字符边界移动
}
```

这个候选保证动态规划至少能沿 UTF-8 字符边界走到句末，属于字符级回退。如果得到的字符 token 不在输出词表中，再由上一节的字节级回退保证可逆编码。因此，词表不必覆盖所有汉字和符号。

TODO：补充训练环节
