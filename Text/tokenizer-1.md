# Tokenizer：SentencePiece

Tokenizer 不只是一个 BPE 合并算法。从原始文本到 token id，中间还要经过规范化、预切分、词表训练和推理编码。PieceTokenizer 中的 `sentencepiece` 方法将这条流程分成四个阶段：

```text
原始文本
  ↓ Normalize
规范化文本
  ↓ PreTokenize
预切分片段
  ↓ Counter
SentencePiece 模型
  ↓ Tokenizer
token pieces / token ids
```

训练和推理必须遵守同一套文本处理规则。否则即使词表相同，相同文本也可能产生不同的 token 序列。

## Normalize

Unicode 中可能存在外观相同、编码不同的字符。例如全角字母、兼容字符和组合字符如果不先统一，会被训练器当成不同符号，浪费词表空间。

`Normalizer` 使用预编译的 Unicode 映射表完成 NFKC 等规范化。映射表被编码为 Double-Array Trie，处理文本时对当前位置执行最长前缀匹配：

```text
输入字节
  ↓ Trie 最长匹配
找到规则 ──→ 输出规范化结果
未找到   ──→ 保留当前 UTF-8 码点
非法字节 ──→ U+FFFD
```

空格也在这一阶段处理。默认情况下，连续空格会被合并，开头和结尾的空格被移除，其余空格替换成可见符号 `▁`：

```text
"  Hello   world  "
        ↓
"Hello▁world"
```

这样空格就能像普通字符一样进入词表，同时仍可在解码时恢复。启用 `reconstruct` 后，Normalizer 会保留全部空格，不再执行合并和裁剪。

规范化规则、空格符号和 `reconstruct` 配置保存在 `PreTokenizerSpec` 中，并随模型一起保存。

## PreTokenize

PreTokenizer 在 BPE 训练之前确定基本边界。PieceTokenizer 将它分成三个维度：

| 维度 | 配置 | 作用 |
| --- | --- | --- |
| Split | `word` / `isolate` | 决定空格、标点和英文缩写怎样切分 |
| Digit | `keep` / `split` | 数字串整体保留或逐码点拆开 |
| Cn | none / char / dict | 连续汉字不切、逐字切或按词典切 |

例如：

```text
输入：Hello, world 你好123

word + keep + none
→ Hello | , | ▁world | ▁ | 你好 | 123

isolate + split + char
→ Hello | , | ▁ | world | ▁ | 你 | 好 | 1 | 2 | 3
```

这些边界非常重要。Counter 只在单个预切分片段内部学习 BPE，不会把两个片段直接合并成一个 piece。

中文词典模式同样发生在这里。`CnCutter` 将词典构造成 Double-Array Trie，并把词频转换成对数概率。切分时，它从每个 UTF-8 码点边界查询所有词典前缀，将候选词看成一条从起点到终点的边：

```text
南京市

0 ──南──> 3 ──京──> 6 ──市──> 9
└────南京────> 6
└──────南京市──────> 9
```

Viterbi 动态规划为每个字节位置保存当前最高分及其前驱位置，最后从文本末尾沿前驱回溯，得到总分最高的切分路径。词典没有覆盖的位置会加入单个 Unicode 码点的 fallback 边，因此生僻字也不会使路径中断。

```cpp
for (const auto& match : GetMatches(han_run)) {
    int start = match.end - match.length + 1;
    int end = match.end + 1;
    float_t score = scores[start] + match.weight;
    if (score > scores[end]) {
        scores[end] = score;
        routes[end] = start;
    }
}
```

中文切词完成后，每个词再独立交给 BPE 学习词内的子词：

```text
南京市长江大桥
  ↓ 中文切词
南京市 | 长江大桥
  ↓ BPE
南京 | 市 | 长江 | 大桥
```

这样可以减少 BPE 仅凭局部共现频率学出跨词片段。这里的中文切词器是 PreTokenizer 内部的独立组件，不依赖后面的 BytePiece 或 SentencePiece 算法。`dict=no` 表示逐字模式；CLI 训练时还会将它组合成 `isolate + digit split`，使中文、数字和标点保持细粒度。

Split、Digit 等配置保存在模型中。外部词典内容不会写进模型，因此词典模式下的训练和推理必须使用同一份词典。

实现上，这三个维度最终收束到同一个入口：

```cpp
std::vector<std::string> PreTokenizer::Split(
    std::string_view normalized) const {
    return ustr::SplitTextCn(
        normalized, space_, cn_cut_fn_, cut_, split_digits_);
}

std::vector<std::string> PreTokenizer::PreTokenize(
    std::string_view text) const {
    return Split(normalizer_.Normalize(text));
}
```

`cn_cut_fn_` 为空时，`SplitTextCn` 仍会执行普通切分和数字拆分；配置中文模式时，它再对连续汉字调用逐字或词典切分。Counter 与 Tokenizer 都调用 `PreTokenize`，从代码层面保证边界一致。

## Counter

`SentencePieceCounter` 负责从预切分语料学习 BPE 词表。它首先加入几类基础 piece：

- `<unk>`、`<s>`、`</s>` 等控制符号；
- 256 个 byte piece，覆盖所有可能的字节；
- 由 `character_coverage` 选出的必备字符。

**字符覆盖率**

Counter 先统计所有预切分片段及其频率，再按片段频率加权统计字符。例如 `low` 出现 5 次，其中的 `l`、`o`、`w` 都贡献 5 次。

字符按照频率从高到低加入必备字符集合，直到累计频率达到 `character_coverage`。覆盖率之外的稀有字符在训练语料中暂时替换成内部未知字符，避免大量罕见字符占满词表。

字符覆盖率解决的是“哪些字符值得作为普通 piece 保留”，byte fallback 解决的是“未保留的字符如何编码”。两者并不冲突：常见字符使用普通词表，罕见字符仍可退回 UTF-8 字节。

**初始化 Symbol**

BPE 从单个 Unicode 码点开始：

```text
片段：lowest
初始：l | o | w | e | s | t
```

每个 `Symbol` 可以表示一个字符，也可以表示左右两个 Symbol 合并后的结果。它还记录频率以及该相邻对在语料中的位置：

```cpp
struct Symbol {
    const Symbol* left;
    const Symbol* right;
    UnicodeText chars;
    size_t byte_size;
    uint64_t fp;
    uint64_t freq;
    std::vector<uint64_t> positions;
};
```

字符 Symbol 会被缓存，同一个字符不需要重复创建对象。两个相邻 Symbol 也通过 fingerprint 缓存成同一个候选 Symbol，`positions` 则记录这个 pair 出现在哪个片段、哪两个位置。

位置本身不保存完整对象，而是压缩成一个 `uint64_t`：

```text
高 32 位：片段编号 sid
中 16 位：左 Symbol 位置
低 16 位：右 Symbol 位置
```

语料中相同 pair 的所有位置因此可以集中到同一个候选上。计算频率时，再把对应片段的出现次数累加起来。

**选择并合并 pair**

考虑经过预切分后的简单语料：

```text
low     5 次
lowest  2 次
newer   3 次
```

初始 pair 频率包括：

```text
l+o = 7
o+w = 7
w+e = 5
n+e = 3
e+w = 3
```

频率相同时，Counter 再按照长度和字符串顺序得到确定结果。假设首先选择 `l+o`：

```text
l | o | w       → lo | w
l | o | w | ... → lo | w | ...
```

下一轮会出现新的候选 `lo+w`，频率为 7：

```text
lo | w → low
```

训练循环就是不断重复“选择最高频 pair、合并全部有效位置、加入新邻居”，直到达到目标词表大小。

**局部更新**

一次合并只会改变它附近的 pair：

```text
l | o | w | e | s | t
    ↓ 合并 l+o
lo | w | e | s | t
      ↓ 更新相邻候选
lo+w
```

右侧位置被置空，只有新 Symbol 左右两侧的候选需要重新加入。其他位置没有变化，不需要重新扫描。

旧候选的 `positions` 中可能仍然保存已经失效的位置。Counter 不会在每次合并后到所有候选中删除它们，而是在 `ComputeFreq` 时检查左右位置是否仍然指向原 Symbol，只累加有效位置，并顺便压缩位置列表。这是一种延迟清理策略。

```cpp
void ComputeFreq(Symbol* symbol) {
    if (symbol->freq > 0) return;

    size_t write = 0;
    for (uint64_t encoded : symbol->positions) {
        Position pos = DecodePos(encoded);
        if (symbol->left == symbols_[pos.sid][pos.left] &&
            symbol->right == symbols_[pos.sid][pos.right]) {
            symbol->freq += freqs_[pos.sid];
            symbol->positions[write++] = encoded;
        }
    }
    symbol->positions.resize(write);
}
```

这里的 `freqs_[pos.sid]` 是预切分片段在语料中的出现次数。一个位置有效时累加的不是 `1`，而是该片段的完整权重。

选出最高频 pair 后，真正修改语料表示的代码只处理命中的位置及其两个邻居：

```cpp
for (uint64_t encoded : best_symbol->positions) {
    Position pos = DecodePos(encoded);
    if (symbols_[pos.sid][pos.left] == nullptr) continue;

    int prev = GetPrevIndex(pos.sid, pos.left);
    int next = GetNextIndex(pos.sid, pos.right);

    ResetFreq(pos.sid, prev, pos.left, best_symbol);
    ResetFreq(pos.sid, pos.right, next, best_symbol);

    symbols_[pos.sid][pos.left] = best_symbol;
    symbols_[pos.sid][pos.right] = nullptr;

    AddNewPair(pos.sid, prev, pos.left);
    AddNewPair(pos.sid, pos.left, next);
}
```

右位置被置为 `nullptr`，左位置指向合并后的 Symbol。旧邻居候选的频率被标记为待重算，新产生的两个相邻 pair 则加入候选集合；整轮没有重新遍历语料。

**控制候选规模**

候选集合也不会始终保存所有 pair。Counter 定期计算候选频率，只保留较高频的一部分作为 active symbols；每隔一段合并再刷新一次。这使大规模语料上的训练仍能控制时间和内存。

pair 的长度还受 `max_piece_size` 限制，避免重复标点或噪声文本产生异常长的 piece。

**生成模型**

每次选中的 piece 按学习顺序获得分数：

```text
第 1 个合并：score =  0
第 2 个合并：score = -1
第 3 个合并：score = -2
```

这里的 score 不是 piece 出现的概率，而是推理时重放 BPE 的优先级。越早学到的合并分数越高。

```cpp
pieces_.emplace_back(
    best_symbol->ToString(),
    -static_cast<float>(pieces_.size()));
```

这行代码把训练顺序直接编码进模型。推理器不需要保存独立的 merge 表，只要读取 piece 的 score 就能恢复相同顺序。

达到目标词表大小后，Counter 补入必备字符，并保存：

```text
CounterSpec
PreTokenizerSpec
Pieces: piece / score / type
```

模型采用可读文本格式。训练配置、预切分配置、普通 piece、byte piece 和控制 token 因而可以一起加载。

## Tokenizer

`SentencePieceTokenizer` 读取模型后，建立 `piece → id` 哈希表。编码时先按照模型中的配置执行 Normalize 和 PreTokenize，再分别编码每个预切分片段。因此，普通切分、数字拆分和中文切分产生的边界都不会被 BPE 跨越，训练与推理遵守同一套文本处理规则。

```cpp
EncodeResult SentencePieceTokenizer::Encode(
    std::string_view text) const {
    EncodeResult result;
    for (const auto& piece : pretokenizer_.PreTokenize(text)) {
        auto sub = EncodeSegment(piece);
        result.insert(result.end(),
                      std::make_move_iterator(sub.begin()),
                      std::make_move_iterator(sub.end()));
    }
    return result;
}
```

`EncodeSegment` 一次只接收一个预切分片段，所以后面的 BPE 合并天然无法跨越片段边界。

**为什么不能只做最长匹配**

BPE 词表不仅记录有哪些字符串，还隐含了它们的学习顺序。假设词表同时存在 `ab`、`bc` 和 `abc`，`abc` 必须先由 `a+b` 或 `b+c` 形成，才能继续参与下一次合并。直接从左到右选择最长字符串，可能得到与训练过程不同的结果。

SentencePieceTokenizer 因而从单个 Unicode 码点开始，按照模型 score 重放 BPE，而不是直接使用 Trie 最长匹配。

**初始化候选队列**

单个片段先被表示成 Symbol 数组，并通过 `prev`、`next` 索引模拟双向链表：

```text
l ⇄ o ⇄ w ⇄ e ⇄ s ⇄ t
```

Tokenizer 检查每一对相邻 Symbol。如果拼接结果存在于词表，就将它放入优先队列：

```text
l | o | w | e | s | t
└ l+o → lo，score=0
    └ o+w → ow，score=-3
```

优先队列先弹出 score 较高的 `lo`，将 `l` 和 `o` 合并：

```text
lo ⇄ w ⇄ e ⇄ s ⇄ t
```

此时只需检查 `lo` 的左右邻居。如果 `low` 也在词表中，就把 `lo+w` 加入队列。其他位置的相邻关系没有改变。

候选只在拼接结果已经存在于模型时入队：

```cpp
auto MaybeAddPair = [&](int left, int right) {
    if (left == -1 || right == -1) return;

    std::string_view piece(
        symbols[left].piece.data(),
        symbols[left].piece.size() + symbols[right].piece.size());
    auto it = pieces_.find(piece);
    if (it == pieces_.end()) return;

    SymbolPair* pair = symbol_pair_allocator.Allocate();
    pair->left = left;
    pair->right = right;
    pair->score = model_->GetPieces(it->second).GetScore();
    pair->size = piece.size();
    agenda.push(pair);
};
```

`SymbolPair` 由对象池统一分配，避免大量候选反复申请小块内存。

**失效候选**

合并 `l+o` 后，队列里原来的 `o+w` 已经失效，因为 `o` 已被合并。为了避免在优先队列中间执行昂贵的删除，Tokenizer 允许旧候选暂时保留，等它被弹出时再检查：

- 左右 Symbol 是否已经被消费；
- 两个 Symbol 的总长度是否仍与候选一致。

无效候选直接丢弃，有效候选才执行合并。这与 Counter 的延迟位置清理采用了相同思路：先保留可能过期的信息，使用时再验证。

```cpp
while (!agenda.empty()) {
    SymbolPair* top = agenda.top();
    agenda.pop();

    if (symbols[top->left].piece.empty() ||
        symbols[top->right].piece.empty() ||
        symbols[top->left].piece.size() +
            symbols[top->right].piece.size() != top->size) {
        continue;
    }

    Symbol& left = symbols[top->left];
    Symbol& right = symbols[top->right];
    left.piece = std::string_view(
        left.piece.data(), left.piece.size() + right.piece.size());
    left.next = right.next;
    if (right.next >= 0) {
        symbols[right.next].prev = top->left;
    }
    right.piece = {};

    MaybeAddPair(symbols[top->left].prev, top->left);
    MaybeAddPair(top->left, symbols[top->left].next);
}
```

合并直接扩展左侧 `string_view`，再修改 `next`、`prev` 索引并清空右侧 Symbol。因为两个片段来自同一段连续文本，所以无需创建新的字符串。

**完整编码示例**

假设模型学到了 `lo`、`low`、`est`，以及形成 `est` 所需的中间 piece，并为最终结果分配了示例 id：

```text
lo   → 300
low  → 301
est  → 417
```

编码 `lowest` 的过程可以表示为：

```text
l | o | w | e | s | t
    ↓ l+o
lo | w | e | s | t
    ↓ lo+w
low | e | s | t
          ↓ 按模型中已有的中间合并形成 est
low | est
    ↓ 查询 id
301 | 417
```

实际结果完全由模型中的 piece 和 score 决定，示例 id 只用于说明过程。

**Byte fallback**

如果某个剩余片段不在普通词表中，Tokenizer 会将它转换成 UTF-8 字节，并逐字节输出对应的 byte piece：

```text
未知字符
  ↓ UTF-8
E7 8C AB
  ↓ byte fallback
<0xE7> | <0x8C> | <0xAB>
```

因此任意输入都可以编码，不需要把整段文本替换成 `<unk>`。

**解码**

解码执行相反过程：普通 piece 直接拼接，byte piece 还原成原始字节，`UNKNOWN` 和控制 token 不写入文本，最后再把 `▁` 还原为空格。

```text
文本
  ↓ Normalize / 必要的 PreTokenize
UTF-8 Symbol
  ↓ 按模型分数合并
pieces
  ↓ 查询词表
token ids
  ↓ 拼接与 byte 恢复
文本
```

至此，SentencePiece 的训练与推理形成闭环：Normalize 统一字符表示，PreTokenize 确定合并边界，Counter 学习词表，Tokenizer 使用同一模型完成编码和解码。

配套实现：[Ismantic/PieceTokenizer](https://github.com/Ismantic/PieceTokenizer)
