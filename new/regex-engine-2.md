# 正则表达式引擎：高级篇

## 引言

本篇在[正则表达式引擎：基础篇](regex-engine-1.md)的实现上继续扩展，目标是支持 Tokenizer 预分词所需的正则语法。整体仍采用递归下降解析、AST、Thompson NFA、DFA 和匹配器组成的流水线，同时增加以下能力：

1. **字符类** `[abc]`、`[^abc]`、`[a-z]` — 匹配一组字符
2. **Unicode 属性** `\p{A}`、`\p{H}`、`\p{N}`、`\p{L}` — 按 Unicode 类别匹配
3. **转义序列** `\r`、`\n`、`\t`、`\s`、`\S`、`\d` — 匹配特殊字符类
4. **重复量词** `{m,n}`、`{m,}`、`{m}` — 精确控制重复次数
5. **非捕获分组** `(?:...)` — 分组但不捕获
6. **量词后缀识别** `*+`、`++`、`?+`、`{m,n}+` — Parser 可以识别，但当前 DFA 不实现完整的占有语义
7. **全文搜索** `FindAll` — 找出所有不重叠的最长匹配
8. **Lazy DFA** — 按需构建 DFA 状态 + 等价类优化

这些特性可以描述 GPT Tokenizer 风格的预分词规则。不同模型使用的 Pattern 并不完全相同，本文只讨论下面这组规则及其简化版本。

**参考的 GPT Tokenizer Pattern**

```
'(?i:[sdmt]|ll|ve|re)|[^\r\n\p{L}\p{N}]?+\p{L}+|\p{N}{1,3}| ?[^\s\p{L}\p{N}]++[\r\n]*|\s*[\r\n]|\s+(?!\S)|\s+
```

**本实现的 Pattern**

```
[^\r\n\p{A}\p{H}\p{N}]?\p{A}+
|\p{H}+
|\p{N}+
| ?[^\s\p{A}\p{H}\p{N}]+[\r\n]*
|\s*[\r\n]
|\s
```

与 GPT-4 的区别：
- **无缩写分支**：GPT-4 有 `'(?i:[sdmt]|ll|ve|re)` 专门匹配英文缩写（`'t`、`'ll` 等），本实现省略——字母分支的标点前缀规则自然产生相同结果（`'` 作为前缀粘到 `t`）
- **汉字独立分组**：`\p{H}+` 单独匹配，空格不会粘到汉字前面（GPT-4 把汉字归入 `\p{L}`，空格会粘连）
- **数字不限长度**：`\p{N}+` 而非 `\p{N}{1,3}`，且空格不会粘到数字前面
- **单个空白**：`\s` 而非 `\s+(?!\S)|\s+`。每个多余空格独立输出，最后一个被下一个字母或标点分支的前缀吸收。这会改变送入 BPE 的初始片段及频率统计，是本实现主动选择的不同语义

## 架构概览

整体流程和 Regex 完全一致，是经典的编译流水线：

```
正则表达式字符串
    ↓ RegexParser（递归下降）
   AST（抽象语法树）
    ↓ NFABuilder（Thompson 构造，Visitor 模式）
   NFA（非确定有限自动机）
    ↓ LazyDFA（按需子集构造 + 等价类）
   DFA（确定有限自动机）
    ↓ Match / FindAll
   匹配结果
```

**与 Regex 的对应关系**

| 组件 | Regex | NewRegex | 变化 |
|------|-------|---------|------|
| AST 节点 | 8 种 | 10 种 | +CharClassAst, RepeatAst |
| AST 基类 | `Ast` | `Ast` + `Clone()` | 为 {m,n} 复制子树 |
| Parser | 基础语法 | +`[...]` `\p{}` `\s` `{m,n}` `(?:)` | 大幅扩展 |
| NFA 边 | `uint32_t` 字符 | `CharPred` 谓词函数 | 核心变化 |
| DFA | 完整预构建 | Lazy 按需构建 | 新方案 |
| 匹配 | `Match`（全文） | `Match` + `FindAll` | +全文搜索 |

## Unicode 属性

**自定义属性**

为 Tokenizer 场景定义了四个近似属性。它们借用了 `\\p{...}` 的语法，但不是 Unicode Character Database 中标准属性的完整实现：

| 属性 | 语法 | 含义 | 包含的字符 |
|------|------|------|-----------|
| Alpha | `\p{A}` | 字母（非汉字非数字） | 拉丁、西里尔、阿拉伯、日韩假名、天城文等 |
| Han | `\p{H}` | 汉字/CJK | CJK 统一汉字及扩展区 A-H |
| Digit | `\p{N}` | 数字子集 | ASCII 0-9、全角 ０-９ |
| Letter | `\p{L}` | 所有字母 | `\p{A}` ∪ `\p{H}` |

取反用 `\P{}`：
- `\P{A}` = 非字母字符
- `\P{N}` = 非数字字符

这种手写范围适合受控的预分词场景，但会遗漏部分文字和数字，也可能覆盖 Unicode 中尚未分配的码位。需要严格兼容 `\\p{L}`、`\\p{N}` 时，应使用 Unicode 数据表生成范围，或接入 ICU 等 Unicode 实现。

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

## CharPred — 谓词化的字符匹配

这是 NewRegex 和 Regex 最核心的架构差异。

**Regex 的方案**

Regex 中 NFA 的边用 `uint32_t` 表示要匹配的字符：

```cpp
// Regex: NFA 边 = 一个具体的 Unicode codepoint
std::map<uint32_t, std::vector<NFAState*>> transitions;
```

这种方案对字面量字符很自然，但无法高效表示 `\p{L}`（覆盖几万个 codepoint）或 `[^abc]`（取反集合）。

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

// 字符范围 [a-z]
[](uint32_t c) { return c >= 'a' && c <= 'z'; }

// Unicode 属性 \p{H}
IsHan  // 直接传函数指针

// 取反 [^abc]
[pred](uint32_t c) { return !pred(c); }

// 组合 [a-z\p{H}]
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
            auto [p, r] = ParseEscape();
            cp = std::move(p);
        } else {
            uint32_t c = NextChar();
            if (/* 下一个是 '-' */) {
                // 范围：a-z
                uint32_t c2 = NextChar();
                cp = [c, c2](uint32_t x) { return x >= c && x <= c2; };
            } else {
                // 单字符
                cp = [c](uint32_t x) { return x == c; };
            }
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

## AST 扩展

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

**新增节点：RepeatAst**

表示 `{m,n}` 量词：

```cpp
class RepeatAst : public Ast {
    std::unique_ptr<Ast> element_;
    int min_, max_;  // max_ == -1 表示无上限 {m,}
};
```

- `{3}` → min=3, max=3
- `{2,4}` → min=2, max=4
- `{2,}` → min=2, max=-1

**Clone 机制**

如果在 AST 层把 `{m,n}` 改写为多个串联节点，就需要通过 `Clone()` 深拷贝子树：

```cpp
class Ast {
    virtual std::unique_ptr<Ast> Clone() const = 0;
};
```

每个子类实现深拷贝。例如 SequenceAst：

```cpp
std::unique_ptr<Ast> Clone() const override {
    auto n = std::make_unique<SequenceAst>();
    for (auto& e : elements_) n->Add(e->Clone());
    return n;
}
```

不过，后文的 NFA Builder 采用另一种方式：对同一个子节点多次调用 `Accept()`，每次创建新的 NFA 状态。采用这种构建方式时，RepeatAst 本身并不依赖 Clone。两种方案任选其一即可，不应同时把 Clone 描述为必要条件。

## Parser 扩展

**新增语法**

在 Regex 的 BNF 基础上扩展：

```
Pattern     = Sequence ('|' Sequence)*
Sequence    = Quantified*
Quantified  = Atom Quantifier?
Quantifier  = '*' '+'? | '+' '+'? | '?' '+'? | '{' Num (',' Num?)? '}' '+'?
Atom        = '(' '?:'? Pattern ')'    -- 分组（可选非捕获）
            | '[' '^'? ClassItem* ']'  -- 字符类        [NEW]
            | '.'                       -- 任意字符
            | '\\' Escape              -- 转义          [NEW]
            | Literal                   -- 字面量
Escape      = 'r' | 'n' | 't' | 's' | 'S' | 'd'
            | 'p' '{' Name '}'         -- Unicode 属性  [NEW]
            | 'P' '{' Name '}'         -- Unicode 属性取反
            | <any char>               -- 转义字面量
ClassItem   = Escape | Char '-' Char | Char
```

**占有量词**

占有量词在消费输入后不允许把字符退还给后续表达式，因此会影响匹配语言。例如 `a*a` 可以匹配 `aaa`，而 `a*+a` 在回溯语义下不能匹配它。DFA 本身不回溯，并不意味着可以把占有量词直接当作普通量词。

当前 Parser 只会读取并丢弃 `+` 后缀：

```cpp
if (Match('*')) { Match('+'); return std::make_unique<StarAst>(...); }
if (Match('+')) { Match('+'); return std::make_unique<PlusAst>(...); }
```

所以这只是语法兼容，不是完整实现。本文后续使用的简化 Pattern 已经移除了占有量词；如果要兼容原始 Pattern，必须在自动机中编码原子边界，或者在进入编译流程前证明这些量词在具体上下文中不会改变结果。

## NFA 构建

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

**RepeatAst 的 NFA 构建**

`{m,n}` 在 NFA 中展开为 m 个必须的拷贝 + (n-m) 个可选的拷贝：

```
a{2,4}  =  a · a · a? · a?
           ↑必须↑  ↑可选↑

a{3,}   =  a · a · a · a*
           ↑必须↑    ↑star↑
```

实现中通过 Visitor 递归调用子节点的 `Accept` 来生成每个拷贝的 NFA 片段，然后串联：

```cpp
void Visit(const RepeatAst* n) override {
    // lo 个必须拷贝
    for (int i = 0; i < lo; i++) {
        n->GetElement()->Accept(this);   // 生成一个 NFA 片段
        append(stack_.top()); stack_.pop();
    }

    if (hi == -1) {
        // {lo,} → 再加一个 Star
        n->GetElement()->Accept(this);
        // ... Star 构造 ...
    } else {
        // {lo,hi} → 再加 (hi-lo) 个 Optional
        for (int i = lo; i < hi; i++) {
            n->GetElement()->Accept(this);
            // ... Optional 构造 ...
        }
    }
}
```

注意：每次调用 `Accept` 都会生成独立的 NFA 片段。由于 NFA 片段的节点是动态分配的（`NewState()`），不需要像 AST 那样手动 Clone。

## Lazy DFA + 等价类

这是和 Regex 在 DFA 层面的主要区别。

**问题：Unicode 状态爆炸**

Regex 的 DFA 用完整的子集构造：预先计算所有可达的 DFA 状态。当 NFA 边是具体字符时，字母表有限，没有问题。

但 NewRegex 的 NFA 边是谓词，一个 `\p{A}` 覆盖几万个 codepoint。如果为每个 codepoint 维护 DFA 转移表，空间会爆炸。

**方案：等价类 + 按需构建**

**等价类**：两个 codepoint 如果对所有 NFA 边的谓词给出相同的 true/false 结果，它们在 DFA 中的行为完全一致，归为同一个等价类。

例如，所有 ASCII 小写字母 `a-z` 都满足 `IsAlpha=true, IsDigit=false, IsHan=false, IsWhitespace=false`，它们属于同一个等价类。

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

## FindAll — 全文搜索

Regex 只有 `Match`（全文匹配），NewRegex 增加了 `FindAll`（找出所有不重叠的最长匹配），这是 Tokenizer 的核心需求。

**算法：左到右、最长匹配**

这里明确采用的是左端起点固定后的最长匹配策略。它不等同于常见回溯引擎的“左端优先、分支从左到右优先”：将所有分支合并成普通 DFA 后，只保留 accept 标记会丢失分支优先级。若目标是逐字节复现某个现有 Regex 引擎，还需要在接受状态中记录分支优先级，并定义长度与优先级的比较规则。

```cpp
std::vector<std::string_view> FindAll(std::string_view text) {
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

输入 `"Hello, World!"` 对本文的简化 Pattern 执行 FindAll：

```
pos=0: MatchAt("Hello, World!") → 5 ("Hello")  → 输出 "Hello"
pos=5: MatchAt(", World!")      → 1 (",")       → 输出 ","
pos=6: MatchAt(" World!")       → 6 (" World")  → 输出 " World"
pos=12: MatchAt("!")            → 1 ("!")        → 输出 "!"
```

结果：`['Hello', ',', ' World', '!']`

## Pattern 解析

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

## 运行示例

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

## 性能分析

**编译时复杂度**

- Parser：O(|pattern|)
- NFA 构建：O(|pattern|)，{m,n} 展开为 O(n) 个片段
- DFA：按需构建，不在编译时消耗

**匹配时复杂度**

- 等价类查询：首次 O(|preds|)，后续 O(1)（缓存）
- DFA 转移：首次 O(|NFA states|)（子集构造），后续 O(1)（缓存）
- 对覆盖全部输入的 Tokenizer Pattern，FindAll 通常按匹配长度向前推进，热缓存下接近 O(|text|)
- 对一般 Pattern，某个起点可能扫描很远后失败，再从下一个字符重试，最坏可达到 O(|text|²)
- Lazy DFA 的首次运行还要承担状态和转移的构建成本，不能与已经完整预构建的 DFA 简单视为相同常数开销

**空间复杂度**

- NFA：O(|pattern|) 个状态
- DFA：按需分配，最坏 O(2^|NFA|)，实际远小于此
- 等价类：最多受谓词真假签名数限制，理论上可达 O(2^|preds|)，实际通常少得多
