# 中文分词：基础篇

## 基本原理

中文句子没有天然的词语分隔符。对于“南京市长江大桥”，至少存在两条表面上合理的切分路径：

```text
南京 | 市 | 长江 | 大桥
南京 | 市长 | 江大桥
```

最长匹配只能做局部选择，无法比较两条完整路径。DictCut 因此把中文分词定义为一个最大概率切分问题。

设句子 `x` 的一种合法切分为：

```text
y = (w₁, w₂, ..., wₙ)
```

其中所有词依次拼接后必须等于原句。Unigram 模型假设各词相互独立，因此这条切分路径的联合概率为：

```text
P(y) = P(w₁) × P(w₂) × ... × P(wₙ)
```

分词的目标是在所有合法切分中选择联合概率最高的一条：

```text
y* = argmax P(y)
```

这里是在固定词典概率下选择最可能的隐藏切分，并不是重新估计模型参数。

词典采用简单的文本格式：

```text
南京    751
市      1003
长江    602
大桥    399
市长    150
江大桥  25
```

第二列是词频，不是已经归一化的概率。`Cutter::Build` 计算完整词典的总频次 `sum`，查询一个已登录词时再得到对数概率：

```cpp
double GetTrieValue(const std::string& word) {
    auto result = da_.GetUnit(word);
    if (!result.found || result.value == 0) {
        return std::log(1.0 / sum_);
    }
    return std::log(
        static_cast<double>(result.value) / sum_);
}
```

为了避免连续相乘造成数值下溢，DictCut 在对数域中计算。对数函数保持大小关系，因此最大化概率乘积等价于最大化对数概率之和：

```text
y* = argmax Σ log P(wᵢ)
```

假设完整词典归一化后得到以下权重：

```text
南京   -2.59
市     -2.30
长江   -2.81
大桥   -3.22
市长   -4.20
江大桥 -5.99
```

两条主要路径的得分分别是：

```text
南京 | 市 | 长江 | 大桥
-2.59 -2.30 -2.81 -3.22 = -10.92

南京 | 市长 | 江大桥
-2.59 -4.20 -5.99 = -12.78
```

因为 `-10.92 > -12.78`，第一条路径胜出。这个模型考虑的是整条切分路径，但每个词的概率仍与上下文无关，因此它不能替代真正的上下文语言模型。

DictCut 在进入中文分词前还会执行一次预切分：

```text
他是英国人Tom，编号123
→ 他是英国人 | Tom | ， | 编号 | 123
```

只有连续汉字段进入 Trie 和动态规划。字母、数字、空格与标点片段直接保留。空格会先被折叠并替换为可见符号 `▁`；这里的 Normalize 只处理空格，不包含 NFKC 等完整 Unicode 规范化。

## 动态规划

DictCut 将一个连续汉字段表示成有向无环图。节点是 UTF-8 字节位置，边是词典中从当前位置开始的候选词。

```text
南京市

0 ──南──> 3 ──京──> 6 ──市──> 9
└────南京────> 6
└──────南京市──────> 9
```

代码中的 `DAG` 使用 `G[i]` 保存从字节位置 `i` 出发的边，其元素是候选词最后一个字节的位置：

```cpp
std::vector<std::set<int>> Cutter::DAG(
    const std::string& sentence) {
    int n = sentence.length();
    std::vector<std::set<int>> G(n);

    for (int i = 0; i < n;) {
        int charlen = ustr::CharLen(
            static_cast<uint8_t>(sentence[i]));

        for (const auto& match :
             da_.PrefixSearch(sentence.substr(i))) {
            size_t end = i + match.length;
            if (match.length > 0 && end <= n) {
                G[i].insert(end - 1);
            }
        }

        G[i].insert(i + charlen - 1);
        i += charlen;
    }
    return G;
}
```

Trie 负责找出词典候选，最后加入的单字符边负责回退。这样图中始终存在一条能够走到句末的路径。

**后向动态规划**

当前实现从句末向句首计算：

```text
route[i].first  = 从位置 i 到句末的最高得分
route[i].second = 最优路径第一条边的结束位置
```

对于 `G[i]` 中的每条边 `i → x`，转移为：

```text
候选得分
= log P(sentence[i : x + 1])
 + route[x + 1].first
```

对应源码是：

```cpp
std::vector<float_i> Cutter::Compute(
    const std::string& sentence,
    const std::vector<std::set<int>>& G) {
    int n = sentence.length();
    const double inf = std::numeric_limits<double>::infinity();
    std::vector<float_i> route(n + 1, {-inf, -1});

    route[n] = {1.0, n};

    for (int i = n - 1; i >= 0; --i) {
        float_i best = {-inf, -1};
        for (int end : G[i]) {
            double score = GetTrieValue(
                sentence.substr(i, end - i + 1));
            score += route[end + 1].first;

            if (score > best.first) {
                best = {score, end};
            }
        }
        route[i] = best;
    }
    return route;
}
```

对数域中空后缀通常初始化为 `0.0`。DictCut 当前使用 `1.0`，相当于为所有完整路径加上同一个常数，不会改变最优路径，也不会改变剪枝时两个路径的得分差。

只有 UTF-8 字符边界上的 `graph[i]` 包含候选边。处于多字节字符内部的位置虽然也存在于数组中，但不会被有效路径访问。

**恢复切分**

动态规划完成后，从句首沿 `route` 向后移动即可恢复结果，不需要再反转：

```cpp
std::vector<std::string> Cutter::CutSegment(
    const std::string& sentence) {
    auto G = DAG(sentence);
    auto route = Compute(sentence, G);

    std::vector<std::string> result;
    for (int start = 0; start < sentence.length();) {
        int end = route[start].second + 1;
        result.push_back(
            sentence.substr(start, end - start));
        start = end;
    }
    return result;
}
```

对于“南京市长江大桥”，`route[0]` 最终指向“南京”，随后依次指向“市”“长江”和“大桥”，得到整条最高分路径。

## Trie 应用

动态规划需要在每个字符位置找到全部词典前缀。如果逐个遍历词典，代价会随词表规模增长。DictCut 使用 Double-Array Trie，将公共前缀压缩在同一条搜索路径上。

构建 Trie 前，词语和频次需要保持一一对应，并按照词语排序：

```cpp
void Cutter::Build(const std::vector<std::string>& words,
                   const std::vector<int>& freqs) {
    sum_ = 0;
    for (int freq : freqs) {
        sum_ += freq;
    }

    std::vector<std::pair<std::string, int>> pairs;
    for (size_t i = 0; i < words.size(); ++i) {
        pairs.emplace_back(words[i], freqs[i]);
    }
    std::sort(pairs.begin(), pairs.end());

    std::vector<std::string> sorted_words;
    std::vector<int> sorted_freqs;
    for (auto& [word, freq] : pairs) {
        sorted_words.push_back(std::move(word));
        sorted_freqs.push_back(freq);
    }

    da_.Build(sorted_words, sorted_freqs);
}
```

Trie 的终止节点直接保存整数频次。切分时，`PrefixSearch` 返回当前位置能够匹配的全部词典前缀及其字节长度：

```cpp
auto matches = da_.PrefixSearch(sentence.substr(i));
for (const auto& match : matches) {
    G[i].insert(i + match.length - 1);
}
```

例如词典中同时存在“南京”“南京市”和“南”，一次前缀搜索会把三条候选边全部加入 DAG。Trie 只负责发现候选；选择最长词还是多个短词，仍由整条路径的概率决定。

搜索成本主要取决于当前位置能够沿 Trie 继续匹配的最长字节数以及返回的候选数量，而不是整个词典的大小。

## 回退机制

词典不可能覆盖所有汉字。为了保证 DAG 始终存在一条从句首到句尾的完整路径，DictCut 会在每个 UTF-8 字符位置加入一条单字符边：

```cpp
int charlen = ustr::CharLen(
    static_cast<uint8_t>(sentence[i]));
graph[i].insert(i + charlen - 1);
```

DAG 使用字节位置，而不是字符编号。普通汉字通常占 3 个字节，扩展区汉字可能占 4 个字节，因此不能简单使用 `i + 1`：

```text
文本：    未  知  𠀀
字节位置：0   3   6    10
回退边：  0 → 3 → 6 → 10
```

单字在词典中时使用它的词频；不在词典中时，使用 `log(1 / sum_)` 作为未知词惩罚。因此已登录词通常优先于未知字，但生僻字仍能参与完整路径。

`Cutter::Cut` 还规定：没有构建词典时，不执行概率计算，而是直接把连续汉字段拆成单个 UTF-8 字符。非汉字段无论是否有词典都保持预切分结果。

## 词频训练

前面的最大概率切分假设词典已经包含可靠的词频。训练阶段解决的是另一个问题：从候选词表和无标注语料中估计这些词频。

```text
候选词表 + 无标注语料
        ↓
估计每个词的使用频次
        ↓
word<TAB>frequency
```

DictCut 首先使用正向最长匹配完成冷启动。`Segmenter` 从当前位置查询 Trie 中的全部前缀并选择最长候选；如果没有候选，就回退到一个 UTF-8 字符：

```cpp
while (i < sentence.size()) {
    auto matches = da_.PrefixSearch(sentence.substr(i));
    size_t best_len = 0;
    for (const auto& match : matches) {
        best_len = std::max(best_len, match.length);
    }

    if (best_len > 0) {
        result.push_back(sentence.substr(i, best_len));
        i += best_len;
    } else {
        int len = ustr::CharLen(
            static_cast<uint8_t>(sentence[i]));
        result.push_back(sentence.substr(i, len));
        i += len;
    }
}
```

最长匹配只负责产生第一版切分。`--count` 对切分结果计数，并用候选词表作为白名单，得到第一份 `word<TAB>frequency` 词典。

接下来重复两个步骤：

```text
当前词频
   ↓ 构建 Trie 与 DAG
Viterbi 最优切分
   ↓ 统计最优路径上的词
新的词频
```

这属于 Viterbi EM，也称 Hard EM：E 步只选择当前概率最高的一条切分路径，M 步统计这条路径上的词频。它没有计算所有可能路径的期望计数，因此比 Soft EM 更简单。

`run_em.sh` 会把语料切成多个分片，并行执行切分和计数，再合并各分片的词频。默认每轮剪枝前执行两次 Hard EM，词表达到目标大小后再执行两次最终重估；次数可以通过 `SUB_ITERS` 调整。

## 词表剪枝

初始候选词表可能包含大量低频词、错误组合和可以被其他词替代的冗余词。DictCut 在若干轮 Hard EM 后计算每个词对最优路径的贡献，再缩小词表。

对于当前最优路径中的一个多字词，`CutWithLoss` 暂时删除它对应的 DAG 边，然后重新运行动态规划：

```text
loss(word)
= 原最优路径得分 - 删除该词后的最优路径得分
```

```cpp
double best_score = route[0].first;

graph[word.start].erase(word.end);
double alternative = Compute(sentence, graph)[0].first;
loss[word.text] += best_score - alternative;
count[word.text]++;
graph[word.start].insert(word.end);
```

loss 越大，说明删除这个词造成的概率损失越大，它越难被其他切分替代。

单字边不能删除，否则可能破坏完整路径。对于单字，代码直接比较它作为已登录词和未知字时的得分：

```cpp
double known = GetTrieValue(word);
double unknown = std::log(1.0 / sum_);
loss[word] += known - unknown;
```

训练脚本会合并所有语料分片上的累计 loss 与出现次数，过滤低于 `MIN_COUNT` 的多字词，再计算剪枝排序分数：

```text
score = loss / sqrt(count) / character_count
```

当前脚本对非 ASCII 词使用 `UTF-8 字节数 / 3` 估计字符数。这适用于常见三字节汉字，但不是通用的 Unicode 字符计数；它在这里仅作为剪枝归一化因子。

全部单字始终保留。对于其余候选，每轮保留排序靠前的 75%，但不会让词表缩到目标大小以下：

```text
new_size
= max(target_size - single_count,
      eligible_count × 75%)
```

完整训练循环为：

```text
正向最长匹配冷启动
        ↓
多轮 Hard EM
        ↓
计算删除 loss 并剪枝
        ↓
词表仍然过大？── 是 ──→ 继续训练
        ↓ 否
最终 Hard EM
        ↓
word<TAB>frequency
```

最终词典既可以直接交给 DictCut 做中文分词，也可以由 PieceTokenizer 在 PreTokenize 阶段加载：DictCut 负责确定中文词边界，SentencePiece 再在每个词内部学习子词。

配套实现：[Ismantic/DictCut](https://github.com/Ismantic/DictCut)
