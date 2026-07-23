# 正则表达式引擎：高级篇

## 引言

本篇在[正则表达式引擎：基础篇](regex-engine-1.md)的实现上继续扩展，目标是支持 Tokenizer 预分词所需的正则语法。整体仍采用递归下降解析、AST、Thompson NFA、DFA 和匹配器组成的流水线，同时增加以下能力：

本文对应 `src/new_regex.cc`。

高级篇最关键的变化是引入**字符集合**。基础篇的一条 NFA 边只匹配一个确定的 codepoint，例如 `'a'`；字符类 `[abc]`、Unicode 属性 `\p{H}` 和补集 `[^...]` 则要求一条边匹配一组甚至大量字符。直接枚举 Unicode codepoint 不现实，因此本篇用谓词函数表示集合：

```cpp
using CharPred = std::function<bool(uint32_t)>;
```

输入字符使谓词返回 `true`，就可以沿这条 NFA 边转移。NFA 到这里已经能够直接运行，但 DFA 还需要高效地缓存确定转移。若两个 codepoint 对所有谓词都产生相同的真假结果，它们在这个正则中的行为就完全相同，可以归入同一个**等价类**。DFA 因而不必为每个 Unicode codepoint 分别保存转移，只需按等价类保存，并在实际遇到输入时按需构建状态。

整个设计因果链可以概括为：

```text
字符集合
    ↓ 用 CharPred 表示集合
谓词化的 NFA 边
    ↓ 按所有谓词的真假结果分类
字符等价类
    ↓ 按等价类缓存并按需构建转移
Lazy DFA
```

其中，`CharPred` 是字符集合的程序表示；等价类则是 DFA 的构建与缓存优化，并不是 NFA 使用谓词边的必要条件。

1. **字符类** `[abc]`、`[^abc]` — 匹配一组字符或其补集
2. **Unicode 属性** `\p{A}`、`\p{H}`、`\p{N}` — 匹配字母、汉字和数字
3. **转义序列** `\r`、`\n`、`\s` — 匹配换行和空白
4. **文本分段** `Segment` — 从左到右产生不重叠的最长匹配片段
5. **Lazy DFA** — 按需构建 DFA 状态 + 等价类优化

这些特性服务于一个明确的 Tokenizer 预分词 Pattern，而不是追求兼容通用正则语法：

```
[^\r\n\p{A}\p{H}\p{N}]?\p{A}+
|\p{H}+
|\p{N}+
| ?[^\s\p{A}\p{H}\p{N}]+[\r\n]*
|\s*[\r\n]
|\s
```

**架构概览**

整体流程和 Regex 完全一致，是经典的编译流水线：

```
正则表达式字符串
    ↓ RegexParser（递归下降）
   AST（抽象语法树）
    ↓ NFABuilder（Thompson 构造，Visitor 模式）
   NFA（非确定有限自动机）
    ↓ LazyDFA（按需子集构造 + 等价类）
   DFA（确定有限自动机）
    ↓ Match / Segment
   匹配结果
```

**与 Regex 的对应关系**

| 组件 | Regex | NewRegex | 变化 |
|------|-------|---------|------|
| AST 节点 | 8 种 | 9 种 | +CharClassAst |
| Parser | 基础语法 | +`[...]` `\p{A}` `\p{H}` `\p{N}` `\s` | 面向目标 Pattern 扩展 |
| NFA 边 | `uint32_t` 字符 | `CharPred` 谓词函数 | 核心变化 |
| DFA | 完整预构建 | Lazy 按需构建 | 新方案 |
| 匹配 | `Match`（全文） | `Match` + `Segment` | +文本分段 |

## Parser

**自定义属性**

为 Tokenizer 场景定义了三个近似属性。它们借用了 `\\p{...}` 的语法，但不是 Unicode Character Database 中标准属性的完整实现：

| 属性 | 语法 | 含义 | 包含的字符 |
|------|------|------|-----------|
| Alpha | `\p{A}` | 字母（非汉字非数字） | 拉丁、西里尔、阿拉伯、日韩假名、天城文等 |
| Han | `\p{H}` | 汉字/CJK | CJK 统一汉字及扩展区 A-H |
| Digit | `\p{N}` | 数字子集 | ASCII 0-9、全角 ０-９ |

这种手写范围适合受控的预分词场景，但会遗漏部分文字和数字，也可能覆盖 Unicode 中尚未分配的码位。需要严格的 Unicode 分类时，应使用 Unicode 数据表生成范围，或接入 ICU 等 Unicode 实现。

**谓词函数**

每个属性对应一个 C++ 函数 `bool(uint32_t)`，在运行时对 Unicode codepoint 求值：

```cpp
static bool IsAlpha(uint32_t c) { return IsWordChar(c) && !IsDigit(c) && !IsHan(c); }
static bool IsHan(uint32_t c) {
    return (c>=0x3400&&c<=0x4DBF) || (c>=0x4E00&&c<=0x9FFF) ||
           (c>=0xF900&&c<=0xFAFF) || (c>=0x20000&&c<=0x323AF);
}
```

这些函数直接作为 NFA 边的匹配条件，不需要枚举所有 Unicode 码位。

**CharPred — 谓词化的字符匹配**

这是 NewRegex 和 Regex 最核心的架构差异。

**Regex 的方案**

Regex 中 NFA 的边用 `uint32_t` 表示要匹配的字符：

```cpp
// Regex: NFA 边 = 一个具体的 Unicode codepoint
std::map<uint32_t, std::vector<NFAState*>> transitions;
```

这种方案对字面量字符很自然，但无法高效表示 `\p{H}`（覆盖大量 codepoint）或 `[^abc]`（取反集合）。

**NewRegex 的方案**

用 `std::function<bool(uint32_t)>` 作为边的匹配条件：

```cpp
using CharPred = std::function<bool(uint32_t)>;

struct Edge {
    CharPred pred;    // 匹配条件：任意 bool(uint32_t) 函数
    NFAState* to;     // 目标状态
};
```

这样任何匹配规则都可以统一表示：

```cpp
// 字面量 'a'
[](uint32_t c) { return c == 'a'; }

// Unicode 属性 \p{H}
IsHan  // 直接传函数指针

// 取反 [^abc]
[pred](uint32_t c) { return !pred(c); }

// 组合 [\r\n]
[a, b](uint32_t c) { return a(c) || b(c); }
```

**字符类的解析**

`[...]` 内部的每个元素被解析为一个 CharPred，然后用 `||` 组合：

```cpp
CharPred ParseCharClass() {
    bool negated = Match('^');
    CharPred pred = [](uint32_t) { return false; };  // 空集

    while (!AtEnd() && pattern_[pos_] != ']') {
        CharPred cp;
        if (pattern_[pos_] == '\\') {
            // 转义：\s, \p{A}, 等
            ++pos_;  // 跳过反斜杠
            auto [p, r] = ParseEscape();
            cp = std::move(p);
        } else {
            uint32_t c = NextChar();
            cp = [c](uint32_t x) { return x == c; };
        }
        // 用 || 组合到 pred 中
        pred = [a=std::move(pred), b=std::move(cp)](uint32_t x) {
            return a(x) || b(x);
        };
    }

    if (negated) pred = [p=std::move(pred)](uint32_t x) { return !p(x); };
    return pred;
}
```

以 `[^\r\n\p{A}\p{H}\p{N}]` 为例，解析过程：
1. `^` → 标记取反
2. `\r` → `pred = (c == '\r')`
3. `\n` → `pred = (c == '\r') || (c == '\n')`
4. `\p{A}` → `pred = ... || IsAlpha(c)`
5. `\p{H}` → `pred = ... || IsHan(c)`
6. `\p{N}` → `pred = ... || IsDigit(c)`
7. 取反 → `pred = !(上述)`

最终的 pred 函数含义："既不是 `\r\n`，也不是字母、汉字、数字"——即标点和符号。

**新增语法**

在 Regex 的 BNF 基础上扩展：

```
Pattern     = Sequence ('|' Sequence)*
Sequence    = Quantified*
Quantified  = Atom Quantifier?
Quantifier  = '*' | '+' | '?'
Atom        = '(' Pattern ')'          -- 分组
            | '[' '^'? ClassItem* ']'  -- 字符类        [NEW]
            | '.'                       -- 任意字符
            | '\\' Escape              -- 转义          [NEW]
            | Literal                   -- 字面量
Escape      = 'r' | 'n' | 's'
            | 'p' '{' ('A'|'H'|'N') '}' -- Unicode 属性 [NEW]
            | <any char>               -- 转义字面量
ClassItem   = Escape | Char
```

## AST

**新增节点：CharClassAst**

表示字符类 `[...]`、转义序列 `\s`、Unicode 属性 `\p{A}` 等：

```cpp
class CharClassAst : public Ast {
    CharPred pred_;     // 匹配谓词
    std::string repr_;  // 用于打印的文本表示
};
```

和 LiteralAst 的区别：
- LiteralAst 匹配一个确定的 codepoint
- CharClassAst 匹配满足谓词的任意 codepoint

在 NFA 构建时，两者生成相同结构的片段（一条边连接起始和终止状态），区别只在边的 CharPred 内容。

## NFA

**CharPred 边**

与 Regex 最大的区别。Regex 中 NFA 的 `Visit(LiteralAst*)` 创建一条 `uint32_t` 边：

```cpp
// Regex
start->InsertTransition(codepoint, end);
```

NewRegex 中统一用 CharPred：

```cpp
// NewRegex
void PushPred(CharPred pred) {
    auto *s = NewState(), *e = NewState();
    e->accept = true;
    s->edges.push_back({std::move(pred), e});
    stack_.push({s, e});
}

void Visit(const LiteralAst* n) override {
    uint32_t p = n->GetPoint();
    PushPred([p](uint32_t c) { return c == p; });
}

void Visit(const CharClassAst* n) override {
    PushPred(n->GetPred());  // 直接使用解析好的谓词
}
```

对于字符类 `[^\r\n\p{A}]`，NFA 的边就是一个组合谓词函数。NFA 结构不需要任何改动——只是边的"标签"从具体字符变成了谓词。

## DFA

这是和 Regex 在 DFA 层面的主要区别。

**问题：Unicode 状态爆炸**

Regex 的 DFA 用完整的子集构造：预先计算所有可达的 DFA 状态。当 NFA 边是具体字符时，字母表有限，没有问题。

但 NewRegex 的 NFA 边是谓词，一个 `\p{A}` 覆盖几万个 codepoint。如果为每个 codepoint 维护 DFA 转移表，空间会爆炸。

**方案：等价类 + 按需构建**

**等价类**：两个 codepoint 如果对所有 NFA 边的谓词给出相同的 true/false 结果，它们在 DFA 中的行为完全一致，归为同一个等价类。

例如，所有 ASCII 小写字母 `a-z` 都满足 `IsAlpha=true, IsDigit=false, IsHan=false, IsWhitespace=false`，它们属于同一个等价类。

等价类的本质是**自动机无法区分的一组输入字符**。从数学上看，它是一个 codepoint 集合：

```text
C = {cp | Signature(cp) = [true, false, false, false]}
```

其中 `Signature(cp)` 表示依次用 NFA 中所有 `CharPred` 判断 `cp` 得到的真假序列。这个集合由当前正则中的谓词决定，并不是 Unicode 固有的字符类别；换一个 Pattern，使用的谓词发生变化，字符的等价类也可能随之变化。

实现中不会真的构造 `std::set<uint32_t>` 来枚举集合成员。Unicode 属性和补集可能包含大量 codepoint，完整保存既浪费空间，也没有必要。代码改用共同的谓词签名隐式表示这个集合：

```text
数学上的集合：
{'a', 'b', 'c', ...}

代码中的表示：
[true, false, false, false] → class_id
```

凡是得到相同签名的 codepoint，都取得同一个 `class_id`，并在 DFA 转移表中共用同一条转移。`sig_map_` 保存“谓词签名 → 等价类编号”，而 `cp_cache_` 只缓存运行时已经遇到的“codepoint → 等价类编号”；两者都没有保存等价类的完整成员集合。

```cpp
int ClassifyCP(uint32_t cp) {
    // 1. 查缓存
    auto it = cp_cache_.find(cp);
    if (it != cp_cache_.end()) return it->second;

    // 2. 计算签名：对每个 NFA 边的谓词求值
    std::vector<bool> sig;
    for (auto* p : all_preds_) sig.push_back((*p)(cp));

    // 3. 签名相同 → 同一个等价类 ID
    auto sit = sig_map_.find(sig);
    int cls;
    if (sit != sig_map_.end()) cls = sit->second;
    else { cls = next_cls_++; sig_map_[sig] = cls; }

    cp_cache_[cp] = cls;
    return cls;
}
```

在当前测试 Pattern 和已观察字符上，通常只会产生十余个等价类，因此 DFA 转移表比较紧凑。这不是一般上界：若有 \\(P\\) 个相互独立的谓词，理论上最多可能出现 \\(2^P\\) 种真假签名。

**按需构建**：DFA 状态不预先全部构建，而是在首次遇到某个 (DFA状态, 等价类) 组合时才计算：

```cpp
int Step(int dfa_st, uint32_t cp) {
    int cls = ClassifyCP(cp);

    // 查 DFA 转移表缓存
    auto it = states_[dfa_st].trans.find(cls);
    if (it != states_[dfa_st].trans.end()) return it->second;

    // 首次遇到：执行 NFA 子集构造
    // 1. 找到当前 DFA 状态对应的 NFA 状态集
    // 2. 对集合中每个 NFA 状态的每条边，用 pred(cp) 测试
    // 3. 收集所有可达的 NFA 状态 → ε 闭包 → 新 DFA 状态
    // 4. 缓存结果
    ...
}
```

首次匹配时按需构建，后续匹配直接查表——兼顾了构建效率和匹配性能。

**与 Regex DFA 的对比**

| | Regex DFA | NewRegex Lazy DFA |
|---|---|---|
| 构建时机 | 编译时一次性全部构建 | 运行时按需构建 |
| 转移表 key | Unicode codepoint | 等价类 ID |
| 空间 | 可能很大 | 取决于实际生成的状态和谓词签名 |
| 首次匹配 | 快（已构建） | 略慢（需构建） |
| 后续匹配 | 快 | 同样快（已缓存） |

**Segment — 文本分段**

Regex 只有 `Match`（全文匹配），NewRegex 增加了 `Segment`（从左到右产生不重叠的最长匹配片段），这是 Tokenizer 的核心需求。

**算法：左到右、最长匹配**

这里明确采用的是左端起点固定后的最长匹配策略。它不等同于常见回溯引擎的“左端优先、分支从左到右优先”：将所有分支合并成普通 DFA 后，只保留 accept 标记会丢失分支优先级。若目标是逐字节复现某个现有 Regex 引擎，还需要在接受状态中记录分支优先级，并定义长度与优先级的比较规则。

```cpp
std::vector<std::string_view> Segment(std::string_view text) {
    std::vector<std::string_view> result;
    size_t pos = 0;

    while (pos < text.size()) {
        std::ptrdiff_t n = MatchAt(text.data() + pos, text.size() - pos);
        if (n > 0) {
            result.emplace_back(text.data() + pos, n);
            pos += n;      // 跳过已匹配的部分
        } else {
            // Tokenizer 不能静默丢弃未匹配输入
            size_t bytes = UTF8Len(text.data() + pos);
            result.emplace_back(text.data() + pos, bytes);
            pos += bytes;
        }
    }

    return result;
}
```

如果表达式能够匹配空串，`MatchAt()` 可能返回 0。上面的实现不输出空匹配，而是把当前 UTF-8 字符作为回退片段输出，以避免死循环和数据丢失；通用正则 API 则需要单独规定空匹配语义。

`MatchAt` 驱动 DFA 从位置 0 开始匹配，记录最后一个 accept 状态的位置（最长匹配）：

```cpp
std::ptrdiff_t MatchAt(const char* data, size_t len) {
    int cur = 0;                              // DFA 起始状态
    std::ptrdiff_t last = states_[0].accept ? 0 : -1;
    size_t pos = 0;

    while (pos < len) {
        uint32_t cp = DecodeUTF8(...);
        int next = Step(cur, cp);             // 查 DFA 转移
        if (next < 0) break;                  // 无转移，停止
        cur = next;
        pos += bytes;
        if (states_[cur].accept) last = pos;  // 记录最长匹配
    }

    return last;  // 返回最长匹配的字节数，-1 表示无匹配
}
```

**示例**

输入 `"Hello, World!"` 对本文的 Pattern 执行 Segment：

```
pos=0: MatchAt("Hello, World!") → 5 ("Hello")  → 输出 "Hello"
pos=5: MatchAt(", World!")      → 1 (",")       → 输出 ","
pos=6: MatchAt(" World!")       → 6 (" World")  → 输出 " World"
pos=12: MatchAt("!")            → 1 ("!")        → 输出 "!"
```

结果：`['Hello', ',', ' World', '!']`

**Tokenizer Pattern**

逐段拆解本实现使用的 pattern：

```
[^\r\n\p{A}\p{H}\p{N}]?\p{A}+
|\p{H}+
|\p{N}+
| ?[^\s\p{A}\p{H}\p{N}]+[\r\n]*
|\s*[\r\n]
|\s
```

**分支 1：字母 run（可带一个前缀）**

```
[^\r\n\p{A}\p{H}\p{N}]?\p{A}+
```

核心分支，处理所有字母文本。

- `[^\r\n\p{A}\p{H}\p{N}]?` — 可选的一个前缀字符。排除了换行、字母、汉字、数字，剩下空格、标点、符号。最多一个。
- `\p{A}+` — 一个或多个字母字符

三种场景：
- **无前缀**：`Hello` → `Hello`
- **空格前缀**：` World` → ` World`（空格粘到字母）
- **标点前缀**：`'t` → `'t`，`,world` → `,world`，`$hello` → `$hello`

常见英文缩写也能得到类似切分：`don't` → `don` + `'t`。不过，这只是当前最长匹配策略下的结果；它不代表简化分支在所有输入上都与带大小写和分支优先级的原规则等价。

**分支 2：汉字 run**

```
\p{H}+
```

连续汉字，无前缀。空格遇到汉字时分支 1 不匹配（`\p{A}+` 不含汉字），空格会被分支 6 独立输出。

**分支 3：数字 run**

```
\p{N}+
```

连续数字，无前缀，无长度限制。空格同样不会粘上来。

**分支 4：标点 run（可带空格前缀）**

```
 ?[^\s\p{A}\p{H}\p{N}]+[\r\n]*
```

- ` ?` — 可选的一个空格前缀（字面空格，不是 `\s`）
- `[^\s\p{A}\p{H}\p{N}]+` — 一个或多个标点/符号
- `[\r\n]*` — 可选的尾部换行

什么时候轮到分支 4？当标点后面**不是字母**时。如果后面跟字母，分支 1 匹配更长（`,world` → 分支 1 赢），分支 4 只在 `,` `---` `$`（后跟数字或结尾）这类场景触发。

**分支 5：换行**

```
\s*[\r\n]
```

换行符（可带前导空白）。

**分支 6：单个空白**

```
\s
```

匹配单个空白字符。是最低优先级的兜底。

连续空格的处理：每个多余空格独立输出为一个 token，最后一个空格被下一个字母或标点分支的前缀吸收。连续标点则由分支 4 合并，例如 `"___hello"` → `"___"`、`"hello"`。

**运行与性能**

下面给出简化 Pattern 的预期输出：

```text

'Hello, World!'         -> ['Hello', ',', ' World', '!']
"don't"                 -> ["don", "'t"]
'$100'                  -> ['$', '100']
'24h'                   -> ['24', 'h']
'hello123world'         -> ['hello', '123', 'world']
' hello'                -> [' hello']
'  hello'               -> [' ', ' hello']
'   hello'              -> [' ', ' ', ' hello']
'hello  world'          -> ['hello', ' ', ' world']
'你好，世界！'            -> ['你好', '，', '世界', '！']
' 你好世界'              -> [' ', '你好世界']
'Hello, 你好! 123abc'   -> ['Hello', ',', ' ', '你好', '!', ' ', '123', 'abc']
```

**性能分析**

**编译时复杂度**

- Parser：O(|pattern|)
- NFA 构建：O(|pattern|)
- DFA：按需构建，不在编译时消耗

**匹配时复杂度**

- 等价类查询：首次 O(|preds|)，后续 O(1)（缓存）
- DFA 转移：首次 O(|NFA states|)（子集构造），后续 O(1)（缓存）
- 对覆盖全部输入的 Tokenizer Pattern，Segment 通常按匹配长度向前推进，热缓存下接近 O(|text|)
- 对一般 Pattern，某个起点可能扫描很远后失败，再从下一个字符重试，最坏可达到 O(|text|²)
- Lazy DFA 的首次运行还要承担状态和转移的构建成本，不能与已经完整预构建的 DFA 简单视为相同常数开销
- `new_regex.cc` 的 `MatchAt` 在解码每个字符时会构造剩余文本的临时 `std::string`，因此实测结果还包含额外分配与复制成本。面向吞吐量优化时，应改为直接按指针和剩余长度解码

**空间复杂度**

- NFA：O(|pattern|) 个状态
- DFA：按需分配，最坏 O(2^|NFA|)，实际远小于此
- 等价类：最多受谓词真假签名数限制，理论上可达 O(2^|preds|)，实际通常少得多

至此，正则引擎已经能将文本切分成适合后续统计的字符串片段。这些片段还不是最终的 token ID；后续 Tokenizer 章节将继续讲解归一化、预分词、频率统计、词表训练和编码过程。
