# Regex

## Introduction

正则表达式（Regex） —— 它是一种**声明式的语言定义方法**，通过简洁的语法描述复杂的字符串集合。

- **语言** = 字符串的集合
- **语法** = 生成规则（正则表达式就是一种语法）
- **识别** = 判断给定字符串是否属于这个语言

举例：regex = `5?3*`
其对应的集合 `{ε, 5, 3, 53, 33, 533, 333, 5333, ...}`。（其中 ε 表示空串）

更深层次，触及到了计算理论的核心：
- 正则语言：正则表达式定义的语言类别，是Chomsky层次结构中最简单的一类
- 有限状态机：每个正则表达式都等价于一个有限状态自动机
- 语言的运算：连接/并集/闭包等操作对应正则表达式的语法结构。

正则表达式的局限性，比如做不到括号匹配。

## Matcher
正则表达式的实现分两部分:
1. **正则表达式的语法建模**
2. **对给定字符串的匹配判别**

当对匹配效率要求不高以及语法比较简单的时候，可以先从**匹配**这个层面入手，快速实现一个正则引擎。
以下给出一个示例，来自开源项目(`Wapiti`)，实现思路很简单：用三个函数分别处理不同层面的匹配逻辑
- **字符层面**的匹配，对特殊字符的处理（比如匹配数字字符）
- **模式层面**的匹配，对量词(*?)字符的处理
- **逐字搜索**的匹配，处理锚点语法（^$）并在字符串中定位匹配位置


### 字符层面匹配
```cpp
bool MatchCharacter(const std::string& pattern, size_t pos, char c) {
    if (c == '\0') return false;
    if (pattern[pos] == '.') return true; 
    if (pattern[pos] == '\\' && pos + 1 < pattern.length()) {
        switch (pattern[pos + 1]) {
            case 'a': return std::isalpha(c);
            case 'd': return std::isdigit(c);
            case 'l': return std::islower(c);
            case 'p': return std::ispunct(c);
            case 's': return std::isspace(c);
            case 'u': return std::isupper(c);
            case 'w': return std::isalnum(c);
            case 'A': return !std::isalpha(c);
            case 'D': return !std::isdigit(c);
            case 'L': return !std::islower(c);
            case 'P': return !std::ispunct(c);
            case 'S': return !std::isspace(c);
            case 'U': return !std::isupper(c);
            case 'W': return !std::isalnum(c);
            default: return pattern[pos + 1] == c;
        }
    }
    return pattern[pos] == c;
}
```
该函数判断给定字符 `c` 是否与 `pattern[pos]` 匹配：
- **`.`** 是通配符，匹配任意字符，总是返回 `true`
- **`\`** 是转移字符，其后的 `pattern[pos+1]` 代表一类字符：
  - 小写字母（如`\d` 代表数字）表示匹配该字符
  - 大小字母（如`\D` 代表非数字）表示匹配该类字符的**取反**

### 模式层面匹配
```cpp
bool MatchPattern(const std::string& re, const std::string& str, uint32_t& n) {
    if (re.empty()) return true;
    if (re[0] == '$' && re.length() == 1) return str.empty();

    size_t cn = (re[0] == '\\') ? 2 : 1;
    std::string next = re.substr(cn);

    if (!next.empty() && next[0] == '*') {
        next = next.substr(1);
        size_t pos = 0;
        do {
            uint32_t save = n;
            if (MatchPattern(next, str.substr(pos), n)) return true;
            n = save + 1;
            pos++;
        } while (pos <= str.length() && MatchCharacter(re, 0, str[pos-1]));
        return false;
    }

    if (!next.empty() && next[0] == '?') {
        next = next.substr(1);
        if (!str.empty() && MatchCharacter(re, 0, str[0])) {
            ++n;
            if (MatchPattern(next, str.substr(1), n)) return true;
            --n;
        }
        return MatchPattern(next, str, n);
    }

    ++n;
    return !str.empty() && MatchCharacter(re, 0, str[0]) &&
           MatchPattern(next, str.substr(1), n);
}
```
该函数实现了三种核心的模式层面匹配：

#### 1. 量词 `*` （零次或多次匹配）
`*` 量词的实现是最复杂的，采用**贪婪匹配**策略（贪婪说的是匹配next）：

```cpp
if (!next.empty() && next[0] == '*') {
    next = next.substr(1);        // 跳过 '*'，获取后续模式
    size_t pos = 0;               // 当前尝试匹配的位置
    do {
        uint32_t save = n;        // 保存当前匹配计数
        // 尝试在当前位置匹配剩余模式
        if (MatchPattern(next, str.substr(pos), n)) return true;
        n = save + 1;             // 恢复计数并增加
        pos++;                    // 尝试下一个位置
    } while (pos <= str.length() && MatchCharacter(re, 0, str[pos-1]));
    // 只要当前匹配*仍然成立，就不会跳出循环
    return false;
}
```

**执行逻辑**：
1. **从零开始尝试**：先尝试匹配 0 次（pos=0），直接匹配后续模式
2. **逐步增加匹配**：如果失败，则尝试匹配 1 次、2 次...直到不能匹配为止
3. **贪婪策略**：每次都先尝试匹配后续模式，而不是贪婪地消耗字符
4. **回溯机制**：如果某个匹配长度失败，则消耗一个字符继续尝试

**示例**：模式 `a*b` 匹配字符串 `"aaab"`
- 尝试 0 个 `a`：匹配 `"aaab"` 的 `b` → 失败
- 尝试 1 个 `a`：匹配 `"aab"` 的 `b` → 失败  
- 尝试 2 个 `a`：匹配 `"ab"` 的 `b` → 失败
- 尝试 3 个 `a`：匹配 `"b"` 的 `b` → 成功！

这个实现主要是避免过度匹配导致的失败
经典问题：如果 * 贪婪地先消耗所有匹配的字符，可能会导致后续模式无法匹配
示例：模式 ".*b" 匹配字符串 "aabb""
- 贪婪实现：.* 先匹配整个"aabb""，然后b无字符可匹配，导致整体匹配失败
- 当前实现：.* 从0开始尝试，逐步增加，直到找到合适的分割点


#### 2. 量词 `?`（零次或一次匹配）

`?` 量词相对简单，但也需要处理两种情况：

```cpp
if (!next.empty() && next[0] == '?') {
    next = next.substr(1);        // 跳过 '?'，获取后续模式
    // 情况1：尝试匹配一次
    if (!str.empty() && MatchCharacter(re, 0, str[0])) {
        ++n;                      // 增加匹配计数
        if (MatchPattern(next, str.substr(1), n)) return true;
        --n;                      // 匹配失败，恢复计数
    }
    // 情况2：匹配零次（跳过当前字符）
    return MatchPattern(next, str, n);
}
```

**执行逻辑**：
1. **优先匹配一次**：如果当前字符能匹配，先尝试消耗一个字符
2. **递归验证**：检查剩余模式是否能匹配剩余字符串
3. **回退到零次**：如果匹配一次失败，则尝试匹配零次（不消耗字符）
4. **贪婪特性**：优先选择匹配一次而不是零次

**示例**：模式 `a?b` 匹配字符串 `"ab"`
- 尝试匹配 1 个 `a`：消耗 `"a"`，剩余 `"b"` 匹配模式 `b` → 成功！

**示例**：模式 `a?b` 匹配字符串 `"b"`  
- 尝试匹配 1 个 `a`：当前字符是 `b`，不匹配 `a`
- 尝试匹配 0 个 `a`：直接用 `"b"` 匹配模式 `b` → 成功！


#### 3. 顺序匹配（精确一次匹配）

当模式中没有量词时，执行顺序匹配：

```cpp
++n;                              // 增加匹配计数
return !str.empty() &&           // 确保字符串非空
       MatchCharacter(re, 0, str[0]) &&  // 当前字符必须匹配
       MatchPattern(next, str.substr(1), n);  // 递归匹配剩余部分
```

**执行逻辑**：
1. **字符串检查**：首先确保目标字符串不为空
2. **字符匹配**：使用 `MatchCharacter` 验证当前字符是否匹配模式
3. **递归处理**：消耗一个字符，继续匹配剩余的模式和字符串
4. **计数更新**：成功匹配时增加匹配字符计数

**示例**：模式 `abc` 匹配字符串 `"abc"`
- 匹配 `a`：`str[0]='a'` 匹配 `re[0]='a'` → 成功
- 递归匹配 `bc` vs `"bc"`：
  - 匹配 `b`：`str[0]='b'` 匹配 `re[0]='b'` → 成功  
  - 递归匹配 `c` vs `"c"`：
    - 匹配 `c`：`str[0]='c'` 匹配 `re[0]='c'` → 成功
    - 递归匹配 `` vs `""`：空模式匹配空串 → 成功

**失败情况**：模式 `abc` 匹配字符串 `"axc"`
- 匹配 `a`：`str[0]='a'` 匹配 `re[0]='a'` → 成功
- 递归匹配 `bc` vs `"xc"`：
  - 匹配 `b`：`str[0]='x'` 不匹配 `re[0]='b'` → 失败

### 逐字搜索匹配
```cpp
int32_t MatchRegex(const std::string re, const std::string& str, uint32_t& n) {
    if (re[0] == '^') {
        n = 0;
        if (MatchPattern(re.substr(1), str, n)) return 0;
        return -1;
    }    

    for (size_t pos = 0; pos <= str.length(); ++pos) {
        n = 0;
        if (MatchPattern(re, str.substr(pos), n)) return pos;
    }
    return -1;
}
```

该函数是整个正则引擎的入口，负责处理锚点和搜索逻辑：

#### 锚点处理

**开头锚点 '^'**
```cpp
if (re[0] == '^') {
    n = 0;
    if (MatchPattern(re.substr(1), str, n)) return 0;
    return -1;
}
```

- 如果模式以 `^` 开头，则只在字符串开始位置尝试匹配
- 去掉 `^` 后调用 `MatchPattern` 匹配剩余模式
- 成功返回位置 0，失败返回 -1- 如果模式以 '^' 开头，则只在字符串开始位置尝试匹配
- 去掉 '^' 后调用 'MatchPattern

**结尾锚点 `$`**：
在 `MatchPattern` 中处理：
```cpp
if (re[0] == '$' && re.length() == 1) return str.empty();
```
- 只有当 `$` 是整个模式时才作为结尾锚点
- 要求剩余字符串为空才匹配成功

#### 逐字搜索

```cpp
for (size_t pos = 0; pos <= str.length(); ++pos) {
    n = 0;
    if (MatchPattern(re, str.substr(pos), n)) return pos;
}
return -1;
```

**执行逻辑**：
1. **遍历所有位置**：从字符串的每个位置开始尝试匹配
2. **重置计数器**：每次尝试前将匹配计数 `n` 重置为 0
3. **子串匹配**：对从当前位置开始的子串进行模式匹配
4. **返回首次匹配位置**：找到匹配则立即返回位置，否则继续搜索
5. **全部失败**：所有位置都匹配失败则返回 -1

**示例**：模式 `abc` 在字符串 `"xyzabc"` 中搜索
- pos=0：`MatchPattern("abc", "xyzabc")` → 失败
- pos=1：`MatchPattern("abc", "yzabc")` → 失败
- pos=2：`MatchPattern("abc", "zabc")` → 失败
- pos=3：`MatchPattern("abc", "abc")` → 成功！返回 3

### 性能分析

Wapiti项目中用这个Regex实现来做特征抽取，场景比较固定，性能要求不高，
不过要是实现一个通用的正则引起，这个方案就不行了。除去代码中涉及到递归函数，
更关键的问题是对量词的处理上。

当前方案的时间复杂度为 **O(n^m)**，其中 m 为量词个数。

#### 经典案例分析

考虑模式 `a*a*b` 匹配字符串 `"aaaaac"`（最后是 c 不是 b，必然失败）：

```
字符串: a a a a a c
模式:   a * a * b
```

对应的关键代码：
```cpp
do {
    uint32_t save = n;
    if (MatchPattern(next, str.substr(pos), n)) return true;  // 分支
    n = save + 1;
    pos++;
} while (pos <= str.length() && MatchCharacter(re, 0, str[pos-1]));
```

#### 数学分析

对于 n 个 `a` 字符，两个 `a*` 的分配方案数：
- 第一个 `a*` 匹配 i 个，第二个匹配 (n-i) 个
- i 可以从 0 到 n，共 (n+1) 种组合

实际上，回溯会尝试**所有可能的组合**：

**n=4 时的尝试次数**：
- 第一个 `a*` 匹配 0 个：第二个 `a*` 尝试 0,1,2,3,4 → 5次
- 第一个 `a*` 匹配 1 个：第二个 `a*` 尝试 0,1,2,3   → 4次  
- 第一个 `a*` 匹配 2 个：第二个 `a*` 尝试 0,1,2     → 3次
- 第一个 `a*` 匹配 3 个：第二个 `a*` 尝试 0,1       → 2次
- 第一个 `a*` 匹配 4 个：第二个 `a*` 尝试 0         → 1次

**总计**：5+4+3+2+1 = 15 = O(n²)

#### 一般化公式

对于 m 个量词和 n 个字符的情况，复杂度为：
- **2 个量词**：O(n²)
- **3 个量词**：O(n³)
- **m 个量词**：O(n^m)

## Compiler

前面实现的匹配方法虽然简单直观，却存在指数级时间复杂度的问题。高性能的正则表达式引擎，就是前面说的语法建模部分，需要用更系统化的编译器方法：
1. **词法分析**：把正则表达式字符串分解成Token序列
2. **语法分析**：用递归下降解析器构建抽象语法树(AST)
3. **语义分析**：通过访问者模式(Visitor)处理AST
4. **代码转换**：把AST转换为非确定有限自动机(NFA)
5. **性能优化**：NFA再转换到DFA以提高匹配效率

这套下来除了能把正则表达式的匹配问题转换为有限自动机的状态转换问题，实现线性时间复杂度的字符串匹配，还能让扩展正则表达式的功能也更容易些，这会是一种教科书级别的实现方案。

### BNF 语法

**BNF** (Backus-Naur Form) 是一种用于描述上下文无关文法的标准表示法，其使用以下符号：

- `::=`  表示"定义为"或"产生"
- `|`  表示"或者"，用于分隔不同的选择
- `<>` 包围非终结符
- 不在 `<>` 中的符合是终结符（具体的字符或Token）

#### 正则表达式的BNF定义

以下实现的正则表达式语法支持以下操作：

```BNF
<Pattern>    ::= <Sequence> ('|' <Sequence>)*
<Sequence>   ::= <Element>*
<Element>    ::= <Atom> <Quantifier>?
<Atom>       ::= <Literal> | '.' | '(' <Pattern> ')'
<Quantifier> ::= '*' | '+' | '?'
<Literal>    ::= UTF-8字符（除了特殊字符）
```

#### 语法详解

##### Pattern层（模式层）
```BNF
<Pattern> ::= <Sequence> ('|' <Sequence>)*
```

- **含义**：模式是由`|`分隔的一个或多个序列的选择
- **示例**：
  - `a|b` → 匹配字符`a`或`b`
  - `hello|world|regex` → 匹配三个单词中的任意一个

##### Sequence层（序列层）
```BNF
<Sequence> ::= <Element>*
```

- **含义**：序列是零个或多个元素的连接
- **示例**：
  - `abc` → 三个字符的序列
  - `a*b+` → 量词修饰的原子序列
  - 空序列 → 匹配空字符串

##### Element层（元素层）
```BNF
<Element> ::= <Atom> <Quantifier>?
```

- **含义**：元素是一个原子可选地跟随一个量词
- **示例**：
  - `a` → 原子，无量词
  - `a*` → 原子后跟星号量词
  - `(ab)+` → 组合原子后跟加号量词


##### Atom层（原子层）
```BNF
<Atom> ::= <Literal> | '.' | '(' <Pattern> ')'
```

- **含义**：原子是最基本的匹配单元
- **示例**：
  - `a` → 字面量字符
  - `.` → 通配符，匹配任意字符
  - `(a|b)` → 括号分组，包含子模式


##### Quantifier层（量词层）
```BNF
<Quantifier> ::= '*' | '+' | '?'
```

- **含义**：量词修饰原子的重复次数
- **说明**：
  - `*` → 零次或多次
  - `+` → 一次或多次  
  - `?` → 零次或一次

#### 语法特性

##### 优先级和结合性

该语法具有以下优先级（从高到低）：

1. **原子**：字面量、点、括号表达式
2. **量词**：`*`、`+`、`?`（后缀，右结合）
3. **连接**：序列中的元素连接（左结合）
4. **选择**：`|` 操作符（左结合）

**示例分析**：
- `ab*` → `a(b*)` 而不是 `(ab)*`
- `a|bc` → `a|(bc)` 而不是 `(a|b)c`
- `a|b|c` → `((a|b)|c)` 左结合

#### 递归结构

语法中存在递归定义：
- `<Pattern>` 在 `<Atom>` 的括号表达式中出现
- 这使得可以构建任意深度的嵌套表达式

**示例**：`((a|b)*c)+` 的解析树：

```
<Pattern>
└── <Sequence>
    └── <Element>
        ├── <Atom>: '(' <Pattern> ')'
        │   └── <Pattern>
        │       └── <Sequence>
        │           ├── <Element>
        │           │   ├── <Atom>: '(' <Pattern> ')'
        │           │   │   └── <Pattern>: a|b
        │           │   └── <Quantifier>: '*'
        │           └── <Element>
        │               └── <Atom>: 'c'
        └── <Quantifier>: '+'
```

### 递归下降解析器

**递归下降解析**是一种自顶向下的语法分析技术，其核心思想是：

1. **每个非终结符对应一个解析函数**
2. **函数调用模拟语法规则的推导**
3. **递归处理体现语法的递归结构**
4. **回溯处理选择和可选元素**

#### 实现详解

`RegexParser`类包含以下核心组件：

```Cpp
class RegexParser {
private:
    std::string pattern;    // 待解析的正则表达式
    size_t pos = 0;        // 当前解析位置
    int cnt = 0;           // 调试用的缩进计数

public:
    explicit RegexParser(std::string p) : pattern(std::move(p)) {}
    std::unique_ptr<Ast> Parse();  // 主解析入口

private:
    // 对应BNF中的每个非终结符
    std::unique_ptr<Ast> ParsePattern();
    std::unique_ptr<Ast> ParseSequence();
    std::unique_ptr<Ast> ParseElement();
    std::unique_ptr<Ast> ParseAtom();
};
```

##### ParsePattern函数 - 处理选择操作

```Cpp
std::unique_ptr<Ast> ParsePattern() {
    PrintEnter("ParsePattern", "<sequence> ('|' <sequence>)*");

    if (pos >= pattern.length()) {
        PrintExit("ParsePattern", "Empty");
        return std::make_unique<EmptyAst>();
    }

    auto n = std::make_unique<AlternativeAst>();
    n->InsertBranch(ParseSequence());  // 解析第一个序列

    while (pos < pattern.length() && pattern[pos] == '|') {
        std::cout << std::string(cnt*2, ' ') << "Got '|', Parse next branch\n";
        ++pos;  // 消耗'|'字符
        n->InsertBranch(ParseSequence());  // 解析下一个序列
    }

    PrintExit("ParsePattern", "AlternativeNode");
    return n;
}
```

**关键设计点**：

1. **空模式处理**：如果到达字符串末尾，返回`EmptyAst`
2. **至少一个分支**：总是解析一个序列，确保Alternative节点至少有一个分支
3. **循环处理多个分支**：使用while循环处理所有`|`分隔的序列
4. **位置管理**：每次遇到`|`时递增`pos`以消耗该字符

**解析示例** - `a|b|c`：

```
初始: pos=0, pattern="a|b|c"
1. ParseSequence() → 解析"a", pos=1
2. 发现pattern[1]='|' → 消耗'|', pos=2
3. ParseSequence() → 解析"b", pos=3  
4. 发现pattern[3]='|' → 消耗'|', pos=4
5. ParseSequence() → 解析"c", pos=5
6. pos=5 >= length=5 → 结束
```


##### ParseSequence函数 - 处理连接操作

```Cpp
std::unique_ptr<Ast> ParseSequence() {
    PrintEnter("ParseSequence", "<element>*");

    auto s = std::make_unique<SequenceAst>();

    while (pos < pattern.length() && 
           pattern[pos] != '|' && 
           pattern[pos] != ')') {
        std::cout << std::string(cnt*2, ' ') << "Parse next element\n";
        s->InsertElement(ParseElement());
    }

    PrintExit("ParseSequence", "SequenceNode");
    return s;
}
```

**终止条件分析**：

1. **到达字符串末尾**：`pos >= pattern.length()`
2. **遇到选择分隔符**：`pattern[pos] == '|'`
3. **遇到分组结束**：`pattern[pos] == ')'`

这些条件确保了序列解析在适当的边界停止，不会越界处理属于上层语法结构的字符。

**解析示例** - `"abc"`：

```
初始: pos=0, pattern="abc"
1. ParseElement() → 解析'a', pos=1
2. ParseElement() → 解析'b', pos=2
3. ParseElement() → 解析'c', pos=3
4. pos=3 >= length=3 → 结束
结果: SequenceAst包含三个LiteralAst节点
```

##### ParseElement函数 - 处理量词

```Cpp
std::unique_ptr<Ast> ParseElement() {
    PrintEnter("ParseElement", "<atom> <quantifier>?");

    if (pos >= pattern.length()) {
        PrintExit("ParseElement", "Empty");
        return std::make_unique<EmptyAst>();
    }

    auto atom = ParseAtom();  // 先解析原子

    if (pos < pattern.length()) {
        char quantifier = pattern[pos];
        switch (quantifier) {
            case '*':
                std::cout << std::string(cnt * 2, ' ') << "Got Quantifier '*'\n";
                ++pos;
                PrintExit("ParseElement", "StarNode");
                return std::make_unique<StarAst>(std::move(atom));
            case '+':
                std::cout << std::string(cnt * 2, ' ') << "Got Quantifier '+'\n";
                ++pos;
                PrintExit("ParseElement", "PlusNode");
                return std::make_unique<PlusAst>(std::move(atom));
            case '?':
                std::cout << std::string(cnt * 2, ' ') << "Got Quantifier '?'\n";
                ++pos;
                PrintExit("ParseElement", "OptionalNode");
                return std::make_unique<OptionalAst>(std::move(atom));
        }
    }

    PrintExit("ParseElement", "AtomNode");
    return atom;
}
```

**处理流程**：

1. **原子优先**：总是先解析原子部分
2. **量词检测**：检查原子后是否有量词字符
3. **包装创建**：根据量词类型创建相应的包装节点
4. **位置递进**：消耗量词字符并更新位置

**解析示例** - `"a*"`：

```
初始: pos=0, pattern="a*"
1. ParseAtom() → 解析'a', pos=1, 返回LiteralAst('a')
2. pattern[1]='*' → 检测到星号量词
3. 创建StarAst节点，包装LiteralAst('a')
4. pos递增到2
结果: StarAst(LiteralAst('a'))
```

##### ParseAtom函数 - 处理基本单元

```Cpp
std::unique_ptr<Ast> ParseAtom() {
    PrintEnter("ParseAtom", "<literal> | '.' | '(' <pattern> ')'");

    if (pos >= pattern.length()) {
        throw std::runtime_error("ParseAtom: Unexpected EOF");
    }

    char c = pattern[pos];
    std::unique_ptr<Ast> element;

    if (c == '(') {
        // 处理分组 '(' <pattern> ')'
        std::cout << std::string(cnt*2, ' ') << "Got '(', Parse down pattern\n";
        ++pos;  // 消耗'('
        element = ParsePattern();  // 递归解析子模式

        if (pos >= pattern.length() || pattern[pos] != ')') {
            throw std::runtime_error("ParseAtom: No matching ')'");
        }
        std::cout << std::string(cnt*2, ' ') << "Got ')', Parse down pattern done\n";
        ++pos;  // 消耗')'
        PrintExit("ParseAtom", "() Expression");
        return element;
        
    } else if (c == '.') {
        // 处理通配符
        std::cout << std::string(cnt*2, ' ') << "Got '.'\n";
        element = std::make_unique<DotAst>();
        ++pos;
        PrintExit("ParseAtom", "DotNode");
        return element;
        
    } else if (c != '|' && c != ')' && c != '*' && c != '+' && c != '?') {
        // 处理UTF-8字面量
        size_t bytes;
        uint32_t codepoint = DecodeUTF8At(pattern, pos, &bytes);

        if (bytes > 0) {
            std::string char_str = pattern.substr(pos, bytes);
            std::cout << std::string(cnt*2, ' ') << "Got UTF-8 '" << char_str 
                      << "' (U+" << std::hex << codepoint << std::dec << ")\n";
            element = std::make_unique<LiteralAst>(codepoint);
            pos += bytes;  // UTF-8字符可能占多个字节
            PrintExit("ParseAtom", "LiteralNode");
            return element;
        } else {
            throw std::runtime_error("ParseAtom: Invalid UTF-8 at " + std::to_string(pos));
        }
    } else {
        throw std::runtime_error("ParseAtom: Unexpected '" + std::string(1, c) + "' at pos " + std::to_string(pos));
    }
}
```

**关键特性**：

1. **分组处理**：括号表达式通过递归调用`ParsePattern`处理
2. **UTF-8支持**：使用专门的UTF-8解码函数处理多字节字符
3. **错误检测**：对不匹配的括号和无效字符进行错误处理
4. **特殊字符过滤**：确保特殊语法字符不被当作字面量处理

#### 完整示例

通过一个完整的例子来观察解析过程：


**输入**：`"(a|b)*c"`

```
ParsePattern() - pos=0
├── ParseSequence() - pos=0
    ├── ParseElement() - pos=0
    │   ├── ParseAtom() - pos=0
    │   │   ├── 发现'(' - pos=1
    │   │   └── ParsePattern() - pos=1 (递归)
    │   │       ├── ParseSequence() - pos=1
    │   │       │   └── ParseElement() - pos=1
    │   │       │       └── ParseAtom() - pos=1
    │   │       │           └── 解析'a' - pos=2
    │   │       ├── 发现'|' - pos=3
    │   │       └── ParseSequence() - pos=3
    │   │           └── ParseElement() - pos=3
    │   │               └── ParseAtom() - pos=3
    │   │                   └── 解析'b' - pos=4
    │   ├── 发现')' - pos=5
    │   └── 发现'*' - pos=6 (创建StarAst)
    └── ParseElement() - pos=6
        └── ParseAtom() - pos=6
            └── 解析'c' - pos=7
```

**最终AST结构**：
```
SequenceAst
├── StarAst
│   └── AlternativeAst
│       ├── LiteralAst('a')
│       └── LiteralAst('b')
└── LiteralAst('c')
```

### 抽象语法树

**抽象语法树(Abstract Syntax Tree, AST)**是源代码语法结构的树状表示，它具有以下特点：

1. **抽象化**：去除了具体语法中的冗余信息（如括号，分隔符）
2. **结构和**：保留了语义相关的层次结构关系
3. **类型化**：每个节点都有明确的类型和语义
4. **可遍历**：支持各种遍历和转化操作

#### AST节点类型

正则表达式AST包含以下节点类型：

```Cpp
enum class AstType {
    Empty,        // 空表达式 ε
    Literal,      // 字面量字符
    Dot,          // 通配符 .
    Sequence,     // 序列连接
    Alternative,  // 选择 |
    Star,         // 零或多次 *
    Plus,         // 一或多次 +
    Optional      // 零或一次 ?
};
```

#### 基础类

```Cpp
class Ast {
public:
    virtual ~Ast() = default;
    virtual AstType GetType() const = 0;
    virtual void Accept(AstVisitor* visitor) const = 0;  // 访问者模式接口
};
```

#### 叶子节点类

**空节点** - 表示空字符串：
```Cpp
class EmptyAst : public Ast {
public:
    AstType GetType() const override { return AstType::Empty; }
    void Accept(AstVisitor* v) const override { v->Visit(this); }
};
```

**字面量节点** - 表示具体字符：
```Cpp
class LiteralAst : public Ast {
private:
    uint32_t point;  // Unicode码点

public:
    explicit LiteralAst(uint32_t p) : point(p) {}
    
    AstType GetType() const override { return AstType::Literal; }
    uint32_t GetPoint() const { return point; }
    void Accept(AstVisitor* v) const override { v->Visit(this); }
};
```

**通配符节点** - 表示点操作符：
```Cpp
class DotAst : public Ast {
public:
    AstType GetType() const override { return AstType::Dot; }
    void Accept(AstVisitor* v) const override { v->Visit(this); }
};
```

#### 复合节点类

**序列节点** - 表示元素连接：
```Cpp
class SequenceAst : public Ast {
private:
    std::vector<std::unique_ptr<Ast>> elements;

public:
    void InsertElement(std::unique_ptr<Ast> e) {
        elements.push_back(std::move(e));
    }

    const std::vector<std::unique_ptr<Ast>>& GetElements() const {
        return elements;
    }

    AstType GetType() const override { return AstType::Sequence; }
    void Accept(AstVisitor* v) const override { v->Visit(this); }
};
```

**选择节点** - 表示或操作：
```Cpp
class AlternativeAst : public Ast {
private:
    std::vector<std::unique_ptr<Ast>> branches;  // 注意：原代码中变量名有拼写错误

public:
    void InsertBranch(std::unique_ptr<Ast> branch) {
        branches.push_back(std::move(branch));
    }

    const std::vector<std::unique_ptr<Ast>>& GetBranches() const {
        return branches;
    }

    AstType GetType() const override { return AstType::Alternative; }
    void Accept(AstVisitor* v) const override { v->Visit(this); }
};
```

**量词节点** - 表示重复操作：
```Cpp
// 星号量词 (0或多次)
class StarAst : public Ast {
private:
    std::unique_ptr<Ast> element;

public:
    explicit StarAst(std::unique_ptr<Ast> e) : element(std::move(e)) {}
    
    const Ast* GetElement() const { return element.get(); }
    AstType GetType() const override { return AstType::Star; }
    void Accept(AstVisitor* v) const override { v->Visit(this); }
};

// 加号量词 (1或多次) 和 问号量词 (0或1次) 结构类似...
```

### 访问者模式

**访问者模式(Visitor Pattern)**是一种行为设计模式，它允许在不修改现有类结构的情况下，对类层次结构添加新的操作。

#### 核心组件

**1. 访问者接口**：
```Cpp
class AstVisitor {
public:
    virtual ~AstVisitor() = default;
    virtual void Visit(const EmptyAst* node) = 0;
    virtual void Visit(const LiteralAst* node) = 0;
    virtual void Visit(const DotAst* node) = 0;
    virtual void Visit(const SequenceAst* node) = 0;
    virtual void Visit(const AlternativeAst* node) = 0;
    virtual void Visit(const StarAst* node) = 0;
    virtual void Visit(const PlusAst* node) = 0;
    virtual void Visit(const OptionalAst* node) = 0;
};
```

**2. 可访问接口**：
```Cpp
// 在每个AST节点中
virtual void Accept(AstVisitor* visitor) const = 0;

// 具体实现（双分派机制）
void Accept(AstVisitor* v) const override { 
    v->Visit(this);  // this的类型决定了调用哪个Visit重载
}
```

**访问者模式的好处**

1. **开闭原则**：可以添加新操作而不修改AST节点类
2. **单一职责**：每个访问者专注于一种操作
3. **类型安全**：编译时确定调用哪个Visit方法
4. **集中化**：相关操作集中在一个访问者类中

#### Printer

通过`RegexPrinter`类来深入理解访问者模式的应用：

```Cpp
class RegexPrinter : public AstVisitor {
private:
    int cnt = 0;  // 缩进计数器

    void PrintIndent() {
        for (int i = 0; i < cnt; ++i) {
            std::cout << " ";
        }
    }

public:
    // 访问叶子节点
    void Visit(const EmptyAst* node) override {
        PrintIndent();
        std::cout << "Empty\n";
    }

    void Visit(const LiteralAst* node) override {
        PrintIndent();
        uint32_t p = node->GetPoint();
        std::string c = EncodeUTF8(p);  // 将Unicode码点转换为UTF-8字符串
        std::cout << "Literal('" << c << "', U+" 
                  << std::hex << p << std::dec << ")\n";
    }

    void Visit(const DotAst* node) override {
        PrintIndent();
        std::cout << "Dot(.)\n";
    }

    // 访问复合节点 - 递归处理子节点
    void Visit(const SequenceAst* node) override {
        PrintIndent();
        std::cout << "Sequence\n";
        cnt++;  // 增加缩进
        for (const auto& e : node->GetElements()) {
            e->Accept(this);  // 递归访问每个子元素
        }
        cnt--;  // 恢复缩进
    }

    void Visit(const AlternativeAst* node) override {
        PrintIndent();
        std::cout << "Alternative\n";
        cnt++;
        for (const auto& branch : node->GetBranches()) {
            branch->Accept(this);  // 递归访问每个分支
        }
        cnt--;
    }

    // 访问量词节点
    void Visit(const StarAst* node) override {
        PrintIndent();
        std::cout << "Star\n";
        cnt++;
        node->GetElement()->Accept(this);  // 递归访问被量词修饰的元素
        cnt--;
    }
    
    // PlusAst和OptionalAst的实现类似...
};
```

##### 示例

对于正则表达式`"(a|b)*c"`，打印器会产生以下输出：

```
Sequence
 Star
  Alternative
   Literal('a', U+61)
   Literal('b', U+62)
 Literal('c', U+63)
```

这个树状结构清晰地展示了：
1. 整体是一个序列（连接）
2. 第一个元素是星号量词
3. 星号修饰的是一个选择（a或b）
4. 第二个元素是字面量字符c

#### 执行流程

跟踪`"a*"`的打印过程：

```Cpp
// 1. 从根节点开始
StarAst* root = ...;  // 解析得到的AST根节点
RegexPrinter printer;
root->Accept(&printer);

// 2. StarAst的Accept方法被调用
void StarAst::Accept(AstVisitor* v) const {
    v->Visit(this);  // this是StarAst*类型
}

// 3. 由于this是StarAst*，调用Visit(const StarAst*)重载
void RegexPrinter::Visit(const StarAst* node) {
    PrintIndent();           // 打印缩进
    std::cout << "Star\n";   // 打印节点类型
    cnt++;                   // 增加缩进层级
    
    // 4. 递归访问子节点
    node->GetElement()->Accept(this);  // 访问LiteralAst('a')
    
    cnt--;                   // 恢复缩进层级
}

// 5. LiteralAst的Accept方法被调用
void LiteralAst::Accept(AstVisitor* v) const {
    v->Visit(this);  // this是LiteralAst*类型
}

// 6. 调用Visit(const LiteralAst*)重载
void RegexPrinter::Visit(const LiteralAst* node) {
    PrintIndent();
    uint32_t p = node->GetPoint();  // 获取Unicode码点 0x61
    std::string c = EncodeUTF8(p);  // 转换为字符串 "a"
    std::cout << "Literal('" << c << "', U+" 
              << std::hex << p << std::dec << ")\n";
}
```

**最终输出**：
```
Star
 Literal('a', U+61)
```

#### 其他应用

访问者模式为AST提供了强大的扩展性，可以轻松实现各种操作：

##### AST验证器
```Cpp
class AstValidator : public AstVisitor {
private:
    bool valid = true;
    std::string error_msg;

public:
    void Visit(const SequenceAst* node) override {
        if (node->GetElements().empty()) {
            // 空序列可以转换为EmptyAst
            return;
        }
        for (const auto& element : node->GetElements()) {
            element->Accept(this);
            if (!valid) break;
        }
    }

    void Visit(const AlternativeAst* node) override {
        if (node->GetBranches().size() < 2) {
            valid = false;
            error_msg = "Alternative must have at least 2 branches";
            return;
        }
        for (const auto& branch : node->GetBranches()) {
            branch->Accept(this);
            if (!valid) break;
        }
    }

    bool IsValid() const { return valid; }
    const std::string& GetError() const { return error_msg; }
};
```

##### AST大小计算器
```Cpp
class AstSizeCalculator : public AstVisitor {
private:
    size_t total_nodes = 0;

public:
    void Visit(const EmptyAst* node) override { total_nodes++; }
    void Visit(const LiteralAst* node) override { total_nodes++; }
    void Visit(const DotAst* node) override { total_nodes++; }
    
    void Visit(const SequenceAst* node) override {
        total_nodes++;
        for (const auto& element : node->GetElements()) {
            element->Accept(this);
        }
    }
    
    void Visit(const StarAst* node) override {
        total_nodes++;
        node->GetElement()->Accept(this);
    }
    
    size_t GetTotalNodes() const { return total_nodes; }
};
```

### NFA

**非确定有限自动机**(Nondeterministic Finite Automation, NFA)是一种理论计算模型，具有以下特征：

1. **状态集合**：有限个状态的集合
2. **输入字母表**：可接受的输入符号集合
3. **转换函数**：从状态和输入符号到状态集合的映射（非确定性）
4. **初始状态**：自动机的起始状态
5. **接受状态集合**：表示匹配成功的状态集合

**NFA**的"非确定性"体现在：
- **ε转换**：不消耗输入字符的状态转换
- **多重转换**：从一个状态在同一输入上可以转换到多个状态
- **并行执行**：可以同时处于多个状态

下面通过代码详解。

#### 状态设计

NFA状态类包含以下组件：

```Cpp
class NFAState {
public:
    int i;                                              // 状态编号
    bool end = false;                                   // 是否为接受状态
    std::map<uint32_t, std::vector<NFAState*>> transitions;  // 字符转换
    std::vector<NFAState*> e_transitions;              // ε转换
    
    static constexpr uint32_t DOT_CHAR = 0xFFFFFFFF;   // 通配符的特殊标记
    static int next;                                    // 全局状态计数器

    NFAState() : i(next++) {}

    void InsertTransition(uint32_t p, NFAState* t) {
        transitions[p].push_back(t);
    }

    void InsertEpsilonTransition(NFAState* t) {
        e_transitions.push_back(t);
    }

    void InsertDotTransition(NFAState* t) {
        transitions[DOT_CHAR].push_back(t);
    }
};
```

##### 设计要点

1. **状态编号**：每个状态有唯一的编号，便于调试和可视化
2. **接受标记**：`end`字段标识接受状态
3. **字符转换表**：`map<uint32_t, vector<NFAState*>>`支持一对多的转换
4. **ε转换列表**：专门处理不消耗字符的转换
5. **通配符处理**：使用特殊值`0xFFFFFFFF`表示通配符转换

##### 片段结构

```Cpp
struct NFA {
    NFAState* start;  // 起始状态
    NFAState* end;    // 结束状态

    NFA(NFAState* s, NFAState* e) : start(s), end(e) {}
};
```

每个NFA片段都有明确的入口和出口，这种设计便于组合和连接。

#### Thompson构造法

Thompson构造法将正则表达式AST转换为NFA。这种方法为每种正则表达式操作定义了标准的NFA模板。

##### 基本构造模板

**1. 空表达式 (ε)**

```
状态图:
[S] --ε--> [E]

代码实现:
void Visit(const EmptyAst* node) override {
    auto start = NewState();
    auto end = NewState();
    end->end = true;
    start->InsertEpsilonTransition(end);
    stack.push(NFA(start, end));
}
```

**2. 字面量字符 (a)**

```
状态图:
[S] --a--> [E]

代码实现:
void Visit(const LiteralAst* node) override {
    uint32_t p = node->GetPoint();
    auto start = NewState();
    auto end = NewState();
    end->end = true;
    start->InsertTransition(p, end);
    stack.push(NFA(start, end));
}
```

**3. 通配符 (.)**

```
状态图:
[S] --.--> [E]

代码实现:
void Visit(const DotAst* node) override {
    auto start = NewState();
    auto end = NewState();
    end->end = true;
    start->InsertDotTransition(end);
    stack.push(NFA(start, end));
}
```

##### 复合构造模板

**4. 序列连接 (AB)**

对于序列`A B C`，构造过程：

```
A: [S1] --> [E1]
B: [S2] --> [E2]  
C: [S3] --> [E3]

连接后:
[S1] --> [E1] --ε--> [S2] --> [E2] --ε--> [S3] --> [E3]
```

```Cpp
void Visit(const SequenceAst* node) override {
    const auto& elements = node->GetElements();
    if (elements.empty()) {
        Visit(static_cast<const EmptyAst*>(nullptr));  // 创建空NFA
        return;
    }

    elements[0]->Accept(this);  // 构造第一个元素的NFA

    for (size_t i = 1; i < elements.size(); ++i) {
        auto left = stack.top(); stack.pop();    // 左侧NFA
        elements[i]->Accept(this);               // 构造右侧NFA
        auto right = stack.top(); stack.pop();  // 右侧NFA

        // 连接：左端点 --ε--> 右起点
        left.end->end = false;                   // 左端点不再是接受状态
        left.end->InsertEpsilonTransition(right.start);
        
        stack.push(NFA(left.start, right.end)); // 新NFA：左起点到右端点
    }
}
```

**5. 选择操作 (A|B)**

对于选择`A | B`，构造模板：

```
原始:
A: [S1] --> [E1]
B: [S2] --> [E2]

构造后:
        --ε--> [S1] --> [E1] --ε--
       /                           \
[新S]                               --> [新E]
       \                           /
        --ε--> [S2] --> [E2] --ε--
```

```Cpp
void Visit(const AlternativeAst* node) override {
    const auto& branches = node->GetBranches();
    if (branches.empty()) {
        Visit(static_cast<const EmptyAst*>(nullptr));
        return;
    }

    branches[0]->Accept(this);  // 构造第一个分支

    for (size_t i = 1; i < branches.size(); ++i) {
        auto left = stack.top(); stack.pop();    // 左分支NFA
        branches[i]->Accept(this);               // 构造右分支NFA
        auto right = stack.top(); stack.pop();  // 右分支NFA
        
        auto start = NewState();  // 新的起始状态
        auto end = NewState();    // 新的结束状态
        end->end = true;
        
        // 起始状态分别连接两个分支的起点
        start->InsertEpsilonTransition(left.start);
        start->InsertEpsilonTransition(right.start);
        
        // 两个分支的终点都连接到新的结束状态
        left.end->end = false;
        right.end->end = false;
        left.end->InsertEpsilonTransition(end);
        right.end->InsertEpsilonTransition(end);
        
        stack.push(NFA(start, end));
    }
}
```

**6. 星号量词 (A*)**

对于`A*`，构造模板：

```
       --ε--> [S1] --> [E1] --ε--
      /          ^           \   |
[新S]            |            v  |
      \          ε            [新E]
       \         |           /
        -------ε----------->
```

```Cpp
void Visit(const StarAst* node) override {
    node->GetElement()->Accept(this);  // 构造内部表达式A的NFA
    auto inner = stack.top(); stack.pop();
    
    auto start = NewState();
    auto end = NewState();
    end->end = true;
    
    // 可以直接跳过A（匹配0次）
    start->InsertEpsilonTransition(inner.start);
    start->InsertEpsilonTransition(end);
    
    // A的结束可以回到A的开始（匹配多次）或者结束
    inner.end->end = false;
    inner.end->InsertEpsilonTransition(inner.start);  // 循环
    inner.end->InsertEpsilonTransition(end);          // 结束
    
    stack.push(NFA(start, end));
}
```

**7. 加号量词 (A+)**

对于`A+`，构造模板（与A*类似，但必须至少匹配一次）：

```
[新S] --ε--> [S1] --> [E1] --ε--> [新E]
                ^           |
                |           |
                ----ε-------
```

```Cpp
void Visit(const PlusAst* node) override {
    node->GetElement()->Accept(this);
    auto inner = stack.top(); stack.pop();
    
    auto start = NewState();
    auto end = NewState();
    end->end = true;
    
    // 必须至少匹配一次A
    start->InsertEpsilonTransition(inner.start);
    
    // A的结束可以回到A的开始（匹配多次）或者结束
    inner.end->end = false;
    inner.end->InsertEpsilonTransition(inner.start);  // 循环
    inner.end->InsertEpsilonTransition(end);          // 结束
    
    stack.push(NFA(start, end));
}
```

**8. 问号量词 (A?)**

对于`A?`，构造模板：

```
       --ε--> [S1] --> [E1] --ε--
      /                        \
[新S]                          --> [新E]
      \                        /
       --------ε -------->----
```

```Cpp
void Visit(const OptionalAst* node) override {
    node->GetElement()->Accept(this);
    auto inner = stack.top(); stack.pop();
    
    auto start = NewState();
    auto end = NewState();
    end->end = true;
    
    // 可以匹配A或者跳过A
    start->InsertEpsilonTransition(inner.start);  // 匹配A
    start->InsertEpsilonTransition(end);          // 跳过A
    
    // A匹配完成后到达结束状态
    inner.end->end = false;
    inner.end->InsertEpsilonTransition(end);
    
    stack.push(NFA(start, end));
}
```

#### NFA构建示例


通过构建`"a*b"`的NFA来演示完整过程：

##### 步骤1：解析AST
```
SequenceAst
├── StarAst
│   └── LiteralAst('a')
└── LiteralAst('b')
```

##### 步骤2：构建子NFA

**构建LiteralAst('a')**：
```
状态0 --'a'--> 状态1(接受)
```

**构建StarAst**：
```
状态2 --ε--> 状态0 --'a'--> 状态1 --ε--> 状态3(接受)
  |                            |
  |                            v
   ----ε----> 状态3 <----ε----
```

**构建LiteralAst('b')**：
```
状态4 --'b'--> 状态5(接受)
```

##### 步骤3：连接序列

将`a*`和`b`连接：
```
状态2 --ε--> 状态0 --'a'--> 状态1 --ε--> 状态3 --ε--> 状态4 --'b'--> 状态5(接受)
  |                            |
  |                            v
   ----ε----> 状态3 <----ε----
```

简化后的最终NFA：
```
状态2 --ε--> 状态0 --'a'--> 状态1 --ε--> 状态4 --'b'--> 状态5(接受)
  |                            |           ^
  |                            v           |
   ----ε----> 状态4 <----ε----            |
              |                            |
               --------'b'---------------->状态5
```


#### PrintNFA

该实现包含了详细的NFA结构打印功能：

```cpp
void PrintNFA(const NFA& n) {
    std::cout << "\n=== NFA Struct ===\n";
    std::set<NFAState*> visited;
    std::queue<NFAState*> queue;

    queue.push(n.start);
    visited.insert(n.start);

    while (!queue.empty()) {
        auto state = queue.front();
        queue.pop();

        std::cout << "State " << state->i;
        if (state == n.start) std::cout << " (START)";
        if (state->end) std::cout << " (END)";
        std::cout << ":\n";

        // 打印字符转换
        for (const auto& [codepoint, targets] : state->transitions) {
            for (auto target : targets) {
                if (codepoint == NFAState::DOT_CHAR) {
                    std::cout << "  --[.]-->";
                } else if (codepoint <= 127 && codepoint >= 32) {
                    std::cout << "  --'" << static_cast<char>(codepoint) << "'-->";
                } else {
                    std::cout << "  --U+" << std::hex << codepoint << std::dec << "-->";
                }
                std::cout << " State " << target->i << "\n";

                if (visited.find(target) == visited.end()) {
                    visited.insert(target);
                    queue.push(target);
                }
            }
        }

        // 打印ε转换
        for (auto target : state->e_transitions) {
            std::cout << "  --ε--> State " << target->i << "\n";
            if (visited.find(target) == visited.end()) {
                visited.insert(target);
                queue.push(target);
            }
        }
    }
}
```

对于正则表达式`"a*"`，输出：
```
=== NFA Struct ===
State 0 (START):
  --ε--> State 1
  --ε--> State 3

State 1:
  --'a'--> State 2

State 2:
  --ε--> State 1
  --ε--> State 3

State 3 (END):
```

### DFA

**确定有限自动机**(Deterministic Finite Automation, DFA) 是NFA的确定性版本，具有以下特征：

1. **确定性转换**：每个状态在给定输入下最多只能转换到一个状态
2. **无ε转换**：所有转换都必须消耗输入字符
3. **高效匹配**：匹配时间复杂度为O(n)，其中n是输入字符串长度
4. **空间换时间**：可能产生指数级的状态数量

**DFA vs NFA对比**

| 特性 | NFA | DFA |
|------|-----|-----|
| 状态转换 | 非确定性（一对多） | 确定性（一对一） |
| ε转换 | 支持 | 不支持 |
| 并行状态 | 可以同时处于多个状态 | 任意时刻只在一个状态 |
| 构造复杂度 | 简单，状态数较少 | 复杂，状态数可能指数增长 |
| 匹配效率 | O(nm)，需要跟踪多个状态 | O(n)，直接状态转换 |

#### 状态设计

```Cpp
class DFAState {
public:
    int i;                                    // 状态编号
    bool end = false;                         // 是否为接受状态
    std::map<uint32_t, DFAState*> transitions; // 确定性转换表（一对一）

    static int next;

    DFAState() : i(next++) {}
};
```

注意DFA状态与NFA状态的区别：
- **转换表类型**：`map<uint32_t, DFAState*>` vs `map<uint32_t, vector<NFAState*>>`
- **无ε转换**：DFA不需要ε转换列表
- **确定性**：每个输入字符最多对应一个目标状态

#### 子集构造法

子集构造法将NFA转换为等价的DFA。基本思想是：DFA的每个状态对应NFA状态的一个子集。

##### 核心算法组件

**1. ε闭包计算(Epsilon Closure)**

ε闭包是指从给定状态集合出发，通过任意数量的ε转换能够到达的所有状态集合。

```Cpp
void EpsilonClosure(std::set<NFAState*>& states) {
    std::stack<NFAState*> stack;

    // 将所有当前状态压入栈
    for (auto state : states) {
        stack.push(state);
    }

    while (!stack.empty()) {
        auto current = stack.top();
        stack.pop();

        // 处理当前状态的所有ε转换
        for (auto target : current->e_transitions) {
            if (states.find(target) == states.end()) {
                states.insert(target);  // 添加新发现的状态
                stack.push(target);     // 继续探索新状态的ε转换
            }
        }
    }
}
```

**算法示例**：
```
输入状态集合: {状态0}
状态0的ε转换: [状态1, 状态3]

执行过程:
1. 初始: states = {0}, stack = [0]
2. 处理状态0: 发现状态1和3，states = {0,1,3}, stack = [1,3]
3. 处理状态1: 无ε转换，stack = [3]
4. 处理状态3: 无ε转换，stack = []
5. 结果: states = {0,1,3}
```

**2. Move操作**

Move操作计算状态集合在给定输入字符下能够转换到的所有状态。

```Cpp
std::set<NFAState*> Move(const std::set<NFAState*>& states, uint32_t input) {
    std::set<NFAState*> result;

    for (auto state : states) {
        // 处理精确字符匹配
        auto it = state->transitions.find(input);
        if (it != state->transitions.end()) {
            for (auto target : it->second) {
                result.insert(target);
            }
        }

        // 处理通配符匹配（除了换行符）
        if (input != 10 && input != 13) {  // 不是\n或\r
            auto dot_it = state->transitions.find(NFAState::DOT_CHAR);
            if (dot_it != state->transitions.end()) {
                for (auto target : dot_it->second) {
                    result.insert(target);
                }
            }
        }
    }

    return result;
}
```

**3. 接受状态检测**

```Cpp
bool ContainsEndState(const std::set<NFAState*>& states) {
    for (auto state : states) {
        if (state->end) return true;
    }
    return false;
}
```



##### 主构造算法

```Cpp
DFAState* Build(const NFA& nfa) {
    std::map<std::set<NFAState*>, DFAState*> state_map;  // NFA状态集合到DFA状态的映射
    std::queue<std::set<NFAState*>> queue;               // 待处理的状态集合队列

    // 1. 创建初始DFA状态
    std::set<NFAState*> start_set = {nfa.start};
    EpsilonClosure(start_set);  // 计算初始状态的ε闭包

    auto start_dfa = NewState();
    if (ContainsEndState(start_set)) {
        start_dfa->end = true;  // 如果ε闭包包含接受状态，则DFA起始状态也是接受状态
    }

    state_map[start_set] = start_dfa;
    queue.push(start_set);

    // 2. 处理队列中的每个状态集合
    while (!queue.empty()) {
        auto current_set = queue.front();
        queue.pop();

        auto current_dfa = state_map[current_set];

        // 3. 收集所有可能的输入字符
        std::set<uint32_t> alphabet;
        for (auto state : current_set) {
            for (const auto& [input, targets] : state->transitions) {
                alphabet.insert(input);
            }
        }

        // 4. 为每个输入字符构造转换
        for (uint32_t input : alphabet) {
            auto next_set = Move(current_set, input);  // 计算转换后的状态集合
            if (next_set.empty()) continue;

            EpsilonClosure(next_set);  // 计算ε闭包

            DFAState* next_dfa;
            if (state_map.find(next_set) == state_map.end()) {
                // 发现新的状态集合，创建对应的DFA状态
                next_dfa = NewState();
                if (ContainsEndState(next_set)) {
                    next_dfa->end = true;
                }
                state_map[next_set] = next_dfa;
                queue.push(next_set);  // 添加到队列中继续处理
            } else {
                // 状态集合已存在，直接获取对应的DFA状态
                next_dfa = state_map[next_set];
            }

            current_dfa->transitions[input] = next_dfa;  // 添加转换
        }
    }

    return start_dfa;
}
```

#### 构建示例


通过`"a*"`的例子来演示DFA构建过程：

##### 输入NFA

```
State 0 (START):
  --ε--> State 1
  --ε--> State 3

State 1:
  --'a'--> State 2

State 2:
  --ε--> State 1
  --ε--> State 3

State 3 (END):
```

##### 构建步骤

**步骤1：初始状态集合**
```
NFA状态集合: {0}
ε闭包: {0, 1, 3}  # 状态0可以通过ε转换到达状态1和3
DFA状态: State 0 (接受状态，因为包含NFA状态3)
```

**步骤2：处理输入'a'**
```
当前集合: {0, 1, 3}
输入'a'的Move结果: {2}  # 只有状态1在输入'a'时转换到状态2
ε闭包({2}): {1, 2, 3}   # 状态2可以通过ε转换到状态1和3
创建DFA状态: State 1 (接受状态，因为包含NFA状态3)
添加转换: State 0 --'a'--> State 1
```

**步骤3：处理新状态集合{1, 2, 3}**
```
当前集合: {1, 2, 3}
输入'a'的Move结果: {2}  # 状态1在输入'a'时转换到状态2
ε闭包({2}): {1, 2, 3}   # 与已存在的状态集合相同
添加转换: State 1 --'a'--> State 1 (自循环)
```

##### 最终DFA

```
=== DFA Struct ===
State 0 (START, END):
  --'a'--> State 1

State 1 (END):
  --'a'--> State 1
```

这个DFA完美地表示了`a*`的语义：
- **State 0**：初始状态，也是接受状态（可以匹配空字符串）
- **State 1**：匹配了至少一个'a'后的状态，也是接受状态
- **自循环**：State 1的自循环表示可以匹配任意多个'a'

#### 示例：`(a|b)*c`

考虑一个更复杂的例子：

##### NFA结构（简化表示）

```
Start --ε--> Choice --ε--> APath --'a'--> AEnd --ε--> Loop --ε--> AfterStar --'c'--> End
             |                                        ^     |
             |                                        |      v
              --ε--> BPath --'b'--> BEnd --ε----------      CPath --'c'--> End
                                                      |
                                                       --ε--> End
```

##### 构建过程

**初始状态**：
```
NFA集合: {Start}
ε闭包: {Start, Choice, APath, BPath, AfterStar, CPath}
DFA State 0: 可以直接匹配'c'或者开始匹配'a'/'b'
```

**转换分析**：
- **输入'a'**：到达能继续匹配'a'/'b'或匹配'c'的状态
- **输入'b'**：类似于'a'的处理
- **输入'c'**：到达接受状态

**最终DFA**（简化）：
```
State 0 (START):
  --'a'--> State 1
  --'b'--> State 1  
  --'c'--> State 2

State 1:
  --'a'--> State 1
  --'b'--> State 1
  --'c'--> State 2

State 2 (END):
```

这个DFA清晰地表达了`(a|b)*c`的语义：匹配任意数量的'a'或'b'，最后以'c'结束。

#### 匹配算法


DFA的匹配算法非常简单高效：

```Cpp
bool Match(DFAState* start, const std::string& text) {
    auto current = start;
    size_t pos = 0;

    while (pos < text.size()) {
        size_t bytes;
        uint32_t codepoint = DecodeUTF8At(text, pos, &bytes);

        if (bytes == 0) {
            return false;  // 无效的UTF-8编码
        }

        // 查找当前状态在该输入下的转换
        auto it = current->transitions.find(codepoint);
        if (it == current->transitions.end()) {
            return false;  // 没有对应的转换，匹配失败
        }

        current = it->second;  // 转换到下一个状态
        pos += bytes;          // 移动到下一个字符
    }

    return current->end;  // 检查最终状态是否为接受状态
}
```

**算法特点**：
1. **线性时间复杂度**：O(n)，其中n是输入字符串长度
2. **确定性执行**：每次只需要查找一个转换
3. **UTF-8支持**：正确处理多字节字符
4. **简单直观**：状态转换逻辑清晰

### 引擎实现

#### Regex类

现在把全部组件整合到一个`Regex`类中：

```Cpp
class Regex {
private:
    std::unique_ptr<Ast> ast;        // 抽象语法树
    DFAState* dfa = nullptr;         // 编译后的DFA
    DFABuilder builder;              // DFA构建器

public:
    Regex(const std::string& pattern) {
        std::cout << "\n=== Compile Regex Pattern: \""
                  << pattern << "\" ===\n";
        
        // 1. 语法分析：构建AST
        RegexParser parser(pattern);
        ast = parser.Parse();
        std::cout << "\n✓ Recursive Descent Parse Done\n";

        // 2. AST可视化
        std::cout << "\n=== AST Structure ===\n";
        RegexPrinter printer;
        ast->Accept(&printer);

        // 3. NFA构建
        std::cout << "\n=== AST -> NFA Conversion ===\n";
        NFABuilder nfa_builder;
        ast->Accept(&nfa_builder);
        auto nfa = nfa_builder.GetNFA();
        std::cout << "✓ AST -> NFA Conversion Done\n";
        nfa_builder.PrintNFA(nfa);

        // 4. DFA构建
        dfa = builder.Build(nfa);
        builder.PrintDFA(dfa);
    }

    bool Match(const std::string& text) {
        return builder.Match(dfa, text);
    }
};
```

#### Pipeline

该正则表达式引擎使用了标准的编译器流水线：


```
正则表达式字符串
        ↓
    [词法分析] (隐含在解析器中)
        ↓
   [语法分析] (递归下降解析器)
        ↓
   抽象语法树 (AST)
        ↓
   [语义分析] (访问者模式遍历)
        ↓
      [代码生成] (Thompson构造法)
        ↓
    非确定有限自动机 (NFA)
        ↓
     [优化] (子集构造法)
        ↓
    确定有限自动机 (DFA)
        ↓
    [执行] (状态机匹配)
        ↓
      匹配结果
```

#### 测试函数

```Cpp
void test_regex(const std::string& pattern, const std::vector<std::string>& cases) {
    try {
        Regex regex(pattern);

        std::cout << "\n=== Test Results ===\n";
        for (const auto& text : cases) {
            bool result = regex.Match(text);
            std::cout << "\"" << text << "\" -> " 
                      << (result ? "Match" : "Not Match") << "\n";
        }

        std::cout << "\n" << std::string(60, '=') << "\n";

    } catch (const std::exception& e) {
        std::cout << "Error: " << e.what() << "\n\n";
    }
}
```

#### 运行示例

##### 基础字符

```Cpp
test_regex("a", {"a", "b", ""});
```

输出：
```
=== Compile Regex Pattern: "a" ===
=== AST Structure ===
Literal('a', U+61)

=== AST -> NFA Conversion ===
✓ AST -> NFA Conversion Done
=== NFA Struct ===
State 0 (START):
  --'a'--> State 1
State 1 (END):

=== DFA Struct ===
State 0 (START):
  --'a'--> State 1
State 1 (END):

=== Test Results ===
"a" -> Match
"b" -> Not Match
"" -> Not Match
```

##### Unicode字符

```Cpp
test_regex("你好", {"你好", "你", "好", "再见"});
```

输出：
```
=== AST Structure ===
Sequence
 Literal('你', U+4f60)
 Literal('好', U+597d)

=== Test Results ===
"你好" -> Match
"你" -> Not Match
"好" -> Not Match
"再见" -> Not Match
```

##### 复杂模式

```Cpp
test_regex("(你|好)*", {"", "你", "好", "你好", "好你", "你你好好"});
```

输出：
```
=== AST Structure ===
Star
 Alternative
  Literal('你', U+4f60)
  Literal('好', U+597d)

=== Test Results ===
"" -> Match
"你" -> Match
"好" -> Match
"你好" -> Match
"好你" -> Match
"你你好好" -> Match
```

#### 性能分析

##### 编译时复杂度

1. **解析**：O(m)，其中m是模式长度
2. **NFA构建**：O(m)，每个AST节点处理一次
3. **DFA构建**：O(2^n)，最坏情况下指数级状态数

##### 匹配时复杂度

1. **NFA匹配**：O(mn)，需要跟踪多个状态
2. **DFA匹配**：O(n)，确定性状态转换

##### 空间复杂度

1. **AST**：O(m)，与模式长度成正比
2. **NFA**：O(m)，Thompson构造法保证线性状态数
3. **DFA**：O(2^m)，最坏情况下指数级

### 扩展示例

通过锚点支持来说明该怎么继续增加引擎的能力，添加行首(^)和行尾($)锚点：

```Cpp
class AnchorAst : public Ast {
public:
    enum Type { START, END };
    
private:
    Type type;
    
public:
    explicit AnchorAst(Type t) : type(t) {}
    Type GetAnchorType() const { return type; }
};
```