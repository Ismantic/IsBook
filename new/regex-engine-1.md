# 正则表达式引擎

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

正则表达式的局限性，比如做不到括号匹配。通俗点说，只能向前走，不能回头看，不记过往。

接下来介绍正则表达式引擎的实现，用Matcher当个引子，重点是Compiler。

## Matcher
当对匹配效率要求不高以及语法比较简单的时候，可以先从**匹配**这个层面入手，快速实现一个正则引擎。
实现机制是双指针对比str与re，要么两个指针同时前进，要么只前进其中一个，只有当两个指针同时到达末尾，说明匹配成功。

以下给出一个示例，来自开源项目(`Wapiti`)，实现思路很简单：用三个函数分别处理不同层面的匹配逻辑
- **字符层面**的匹配，对特殊字符的处理（比如匹配数字字符）
- **模式层面**的匹配，对量词(*?)字符的处理
- **双指针移动**，处理锚点语法（^$）且在字符串中定位匹配位置


**字符层面匹配**

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



**模式层面匹配**

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

MatchPattern 的作用是：从 str 的当前位置开始，判断整个模式 re 能否匹配，并通过 n 记录匹配长度。

该函数实现了三种核心的模式层面匹配：

**1. 量词 `*` （零次或多次匹配）**
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

**2. 量词 `?`（零次或一次匹配）**

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


**3. 顺序匹配（精确一次匹配）**

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


**双指针移动**

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

该函数是整个正则引擎的入口，负责处理锚点和搜索逻辑，若str不匹配re返回-1，否则返回匹配开始的位置：

**锚点处理**

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

**逐字搜索**

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

**性能分析**

Wapiti项目中用这个Regex实现来做特征抽取，场景比较固定，性能要求不高，
不过要是实现一个通用的正则引起，这个方案就不行了。除去代码中涉及到递归函数，
更关键的问题是对量词的处理上。

当前方案的时间复杂度为 **O(n^m)**，其中 m 为量词个数。

**经典案例分析**

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

**数学分析**

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

**一般化公式**

对于 m 个量词和 n 个字符的情况，复杂度为：
- **2 个量词**：O(n²)
- **3 个量词**：O(n³)
- **m 个量词**：O(n^m)
