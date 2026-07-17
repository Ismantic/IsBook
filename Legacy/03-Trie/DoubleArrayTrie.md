# DoubleArray

## 1. Introduction

DoubleArrayTrie 是一种高校的字典树 (Trie) 压缩表示方法，通过巧妙的数组布局和状态转移机制，实现了空间高效且访问快速的字符串检索数据结构。本文通过一个C++实现，深入阐述其核心原理和实现细节。

### 1.1 问题

以存储单词 {"he", "she", "his"} 为例，传统指针字典树结构如下：

```
    root(0)
    /     \
   h(1)    s(7)
  / | \    |
 e(2) i(3) h(8)
 |   |     |
$(4) s(5)  e(9)
     |     |
    $(6)  $(10)

节点编号及其含义：
0:root  1:h  2:he  3:hi  4:"he"  5:his  6:"his"  7:s  8:sh  9:she  10:"she"
```

**传统实现的问题**：
- 节点需要存储多个子节点指针，内存开销大
- 内存分散，缓存不友好
- 指针访问增加间接寻址开销

### 1.2 思路

双数组的本质是把树"平铺"到一维数组中：
- **state**：存储节点的"基地址"
- **character**：存储字符信息
- **up**：验证父子关系的正确性

数组表示：
```cpp
struct Node {
    int state;
    int up;
    char character;
};
std::vector<Node> array;
```

状态转移：
```cpp
// -> ct
state = array[pos].state
new_pos = pos ^ state ^ ct
// <- ct
array[new_pos].up = pos
array[new_pos].character = ct
```

由状态转移关系，pos 和 new_pos 的两个节点就能联系起来，就可以设计算法把整个树平铺到数组上。

### 1.3 XOR

状态转移用到了 XOR 操作，以下是详细介绍。

**性质1：交换律**

```
a ^ b = b ^ a
```

**性质2：结合律**

```
(a ^ b) ^ c = a ^ (b ^ c)
```

**性质3：自反性**

```
a ^ a = 0
a ^ 0 = a
```

**性质4：可逆性**

```
如果 c = a ^ b，那么 a = c ^ b，b = c ^ a
```

### 1.4 举例
接下来把通过把前面的例子 {he, his, she} 用数组方式存储做详细阐述。首先再说明下状态转移公式：

```cpp
uint32_t next_pos = pos ^ state ^ ct;
```

**举例说明**：
假设当前位置 `pos = 5`，状态值 `state = array_[pos].state = 12`，字符 `ct = 'a'(97)`：

```
next_pos = 5 ^ 12 ^ 97
         = 5 ^ 12 ^ 97
         = 104
```

**验证机制**：
```cpp
// 检查目标位置的字符是否匹配
if (array_[next_pos].character != ct) return false;

// 检查父子关系是否正确  
if (array_[next_pos].up != pos) return false;
```

其次，需要注意的是state的取值，要设计一个GetFreeIndex函数，其能返回一个能满足存储当前节点的孩子节点的位置，得到该位置之后，假设当前位置为pos,返回的为index,那么 state = pos ^ index。


**完整示例：** 对前面的 {he, his, she} 的例子，建好的数组如下表显示（eow表示是否词结束）：

| pos | character | up  | state | eow   | 含义        | 说明                    |
|-----|-----------|-----|-------|-------|-------------|-----------------------|
| 0   | root      | -   | 5     | false | 根状态      | 分配index=5给子节点{h,s} |
| 109 | h         | 0   | 103   | false | 'h'状态     | 分配index=10给子节点{e,i} |
| 118 | s         | 0   | 98    | false | 's'状态     | 分配index=20给子节点{h} |
| 111 | e         | 109 | 200   | true  | 'he'状态    | 叶子节点，state指向值节点200 |
| 99  | i         | 109 | 125   | false | 'hi'状态    | 分配index=30给子节点{s} |
| 124 | h         | 118 | 84    | false | 'sh'状态    | 分配index=40给子节点{e} |
| 93  | s         | 99  | 300   | true  | 'his'状态   | 叶子节点，state指向值节点300 |
| 77  | e         | 124 | 400   | true  | 'she'状态   | 叶子节点，state指向值节点400 |
| 200 | \0        | 0   | 0     | true  | "he"值节点  | 存储值0               |
| 300 | \0        | 0   | 1     | true  | "his"值节点 | 存储值1               |
| 400 | \0        | 0   | 2     | true  | "she"值节点 | 存储值2               |


**详细构建过程**

1. 根节点分配 (pos=0)
```
子节点: {h(104), s(115)}
GetFreeIndex找到: index = 5
子节点位置计算:
  h位置: 5 ^ 104 = 109
  s位置: 5 ^ 115 = 118
根节点state: 0 ^ 5 = 5
```

2. h节点分配 (pos=109)
```
子节点: {e(101), i(105)}
GetFreeIndex找到: index = 10
子节点位置计算:
  e位置: 10 ^ 101 = 111
  i位置: 10 ^ 105 = 99
h节点state: 109 ^ 10 = 103
```

3. s节点分配 (pos=118)
```
子节点: {h(104)}
GetFreeIndex找到: index = 20
子节点位置计算:
  h位置: 20 ^ 104 = 124
s节点state: 118 ^ 20 = 98
```

4. hi节点分配 (pos=99)
```
子节点: {s(115)}
GetFreeIndex找到: index = 30
子节点位置计算:
  s位置: 30 ^ 115 = 93
hi节点state: 99 ^ 30 = 125
```

5. sh节点分配 (pos=124)
```
子节点: {e(101)}
GetFreeIndex找到: index = 40
子节点位置计算:
  e位置: 40 ^ 101 = 77
sh节点state: 124 ^ 40 = 84
```

注意，以上构建的过程中子节点位置的计算，用的是 new_pos = index ^ ct，而前面说的状态转移过程，new_pos = pos ^ state ^ ct，其实这两者是等价的，state = pos ^ index，因而 pos ^ state ^ ct = pos ^ pos ^ index ^ ct = 0 ^ index ^ ct = index ^ ct。这也是巧妙利用 XOR 性质的地方。

其实到这里，已经基本上能把DoubleArray创建的方法说的差不多了，就是不断的找到GetFreeIndex,分配给子节点，再对子节点迭代下去，直到把Trie树遍历完。

## 2. Implementation

深入代码细节之前，先整体了解下DoubleArrayTrie的核心实现思路：

**设计要点**
1. 通过XOR运算实现状态转移（pos^state^ct）
2. 通过up字段验证父子关系
3. 叶子节点复用state字段存储值(value)

**构建过程**
```
传统Trie → 递归平铺 → 双数组结构
```

1. 初始化根节点（pos=0）
2. 对每个节点的子节点集合：
   - 调用GetFreeIndex寻找合适基址
   - 计算所有子节点位置（index ^ char）
   - 设置父子互指关系（up字段）
3. 递归处理每个子节点

### 2.1 数据结构

Node 是数组中的基本单元，注意state和value用了union，这是因为叶子节点用不上state,这样就可以把空间给value用了。还有就是eow，对一个字符串"abc"，其实eow会设置两次，一是会把字符串延长成"abc\0"，二是会对c和\0都设置eow，叶子节点的时候value是有值的，此时会满足character == 0 和 eow == true.

```cpp
struct Node {
    uint8_t character;  // 当前节点代表的字符
    bool eow;          // End of Word 标记
    uint32_t up;       // 父节点位置
    union { 
        uint32_t state;    // 状态值（用于中间节点）
        int32_t value;     // 存储值（用于叶子节点）
    };
    
    bool Empty() const {
        return character == 0 && !eow && up == 0 && state == 0;
    }
    
    bool Value() const {
        return character == 0 && eow;
    }
};
```

以及，构建过程中用到的数据结构：
```cpp
struct TrieNode {
    std::map<uint8_t, std::unique_ptr<TrieNode>> down_nodes;
    int32_t value;
    bool eow;
};

std::unique_ptr<TrieNode> root_;
std::vector<Node> units_;
std::vector<bool> uses_;

uint32_t prev_pos_ = 0;
```

### 2.2 构建过程

#### 阶段一：构建传统字典树
```cpp
void TrieInsert(const std::string& str, int32_t value) {
    TrieNode* current = root_.get();
    for (char c : str) {
        uint8_t t = static_cast<uint8_t>(c);
        if (current->down_nodes.find(t) == current->down_nodes.end()) {
            current->down_nodes[t] = std::make_unique<TrieNode>();
        }
        current = current->down_nodes[t].get();
    }
    current->eow = true;
    current->value = value;
}
```

#### 阶段二：转换为双数组

核心的状态分配算法：

```cpp
uint32_t SetupDownNodes(const std::vector<uint8_t>& es, uint32_t pos, TrieNode* node) {
    uint32_t index = GetFreeIndex(es);
    units_[pos].state = pos ^ index;  // 关键：利用 XOR 存储状态
    
    for (size_t i = 0; i < es.size(); ++i) {
        uint8_t e = es[i];
        uint32_t p = index ^ e;  // 利用 XOR 计算子节点位置
        
        uses_[p] = true;
        
        if (e == '\0') {
            units_[p].value = node->value;
            units_[p].eow = true;
            units_[pos].eow = true;
        }
        
        units_[p].character = e;
        units_[p].up = pos;
    }
    
    return index;
}
```

#### 阶段三：递归转换

```cpp
void NodeConvert(TrieNode* node, uint32_t pos) {
    std::vector<uint8_t> es;
    std::vector<TrieNode*> down_nodes;
    
    // 收集所有子节点
    for (const auto& n : node->down_nodes) {
        es.push_back(n.first);
        down_nodes.push_back(n.second.get());
    }
    
    // 如果是单词结尾，添加值节点
    if (node->eow) {
        es.push_back('\0');
        down_nodes.push_back(nullptr);
    }
    
    if (!es.empty()) {
        uint32_t index = SetupDownNodes(es, pos, node);
        
        // 递归处理子节点
        for (size_t i = 0; i < es.size(); ++i) {
            uint8_t e = es[i];
            if (e != '\0') {
                size_t down_pos = index ^ e;
                NodeConvert(down_nodes[i], down_pos);
            }
        }
    }
}
```

注意，对 '\0' 的处理，递归处理子节点的时候不会对这个节点做处理，因为其不存在子节点，而 SetupDownNodes 的时候会对 '\0' 创建 value 节点。

#### 具体示例


以 {"cat", "car"} 为例，展示完整构建过程：

**步骤1：构建传统树**
```
root
 |
 c
 |
 a
/ \
t   r
$   $
```

**步骤2：转换根节点**
```
es = {'c'}
尝试 index = 1:
  'c'(99) 的位置 = 1 ^ 99 = 98
  检查 uses_[98] = false ✓

root: state = 0 ^ 1 = 1
'c' 节点放在位置 98
```

**步骤3：转换 'c' 节点**
```
es = {'a'}  
尝试 index = 2:
  'a'(97) 的位置 = 2 ^ 97 = 99
  检查 uses_[99] = false ✓

'c': state = 98 ^ 2 = 100
'a' 节点放在位置 99
```

**步骤4：转换 'a' 节点**
```
es = {'t', 'r'}
尝试 index = 10:
  't'(116) 的位置 = 10 ^ 116 = 126
  'r'(114) 的位置 = 10 ^ 114 = 124
  检查 uses_[126] = false, uses_[124] = false ✓

'a': state = 99 ^ 10 = 105
't' 节点放在位置 126，'r' 节点放在位置 124
```

### 2.3 精准匹配

```cpp
Piece GetPiece(const std::string& str) const {
    if (Empty()) return Piece();

    uint32_t pos = 0;
    const char* s = str.c_str();
    size_t n = str.size();

    for (size_t i = 0; i < n; ++i) {
        Node unit = array_[pos];
        uint32_t state = unit.state;
        uint8_t ct = static_cast<uint8_t>(s[i]);

        // 核心：XOR 状态转移
        uint32_t next_pos = pos ^ state ^ ct;

        if (next_pos >= size_) return Piece();

        pos = next_pos;
        unit = array_[pos];

        if (unit.character != ct) {
            return Piece();
        }
    }

    // 验证是否为完整单词
    Node unit = array_[pos];
    if (!unit.eow) return Piece();

    // 获取存储的值
    uint32_t value_pos = pos ^ unit.state;
    if (value_pos >= size_) return Piece();

    Node value_unit = array_[value_pos];
    if (!value_unit.Value()) 
        return Piece();
    
    return Piece(value_unit.value, n, str);
}
```

**查找 "cat" 的完整过程**：

```
初始: pos = 0

第1步 ('c'):
  unit = array_[0], state = 1
  next_pos = 0 ^ 1 ^ 99 = 98
  检查 array_[98].character == 'c' ✓
  pos = 98

第2步 ('a'):  
  unit = array_[98], state = 100
  next_pos = 98 ^ 100 ^ 97 = 99
  检查 array_[99].character == 'a' ✓
  pos = 99

第3步 ('t'):
  unit = array_[99], state = 105  
  next_pos = 99 ^ 105 ^ 116 = 126
  检查 array_[126].character == 't' ✓
  pos = 126

验证单词结尾:
  array_[126].eow == true ✓
  value_pos = 126 ^ array_[126].state
  返回 array_[value_pos].value
```

### 2.4 前缀匹配

这个函数找出输入字符串中所有作为字典中单词前缀的部分：

```cpp
std::vector<Piece> GetUpPieces(const std::string& str) const {
    std::vector<Piece> rs;
    if (Empty()) return rs;

    uint32_t pos = 0;
    const char* s = str.c_str();
    size_t n = str.size();

    for (size_t i = 0; i < n; ++i) {
        Node unit = array_[pos];
        uint32_t state = unit.state;
        uint8_t ct = static_cast<uint8_t>(s[i]);

        uint32_t next_pos = pos ^ state ^ ct;
        if (next_pos >= size_) break;

        pos = next_pos;
        Node next_unit = array_[pos];

        if (next_unit.character != ct) break;

        // 检查当前位置是否为单词结尾
        if (next_unit.eow) {
            size_t value_pos = pos ^ next_unit.state;
            if (value_pos < size_ && array_[value_pos].Value()) {
                Node value_node = array_[value_pos];
                rs.emplace_back(value_node.value, i+1, str.substr(i+1));
            }
        }
    }

    return rs;
}
```

**示例**：字典包含 {"car", "card", "care"}，输入 "card"

```
输入: "card"

i=0, 字符='c': 
  找到 'c' 节点，不是单词结尾，继续

i=1, 字符='a': 
  找到 'a' 节点，不是单词结尾，继续

i=2, 字符='r': 
  找到 'r' 节点，是单词结尾 ✓
  添加 Piece(value=0, num=3, str="d") 到结果 ("car" 匹配)

i=3, 字符='d': 
  找到 'd' 节点，是单词结尾 ✓  
  添加 Piece(value=1, num=4, str="") 到结果 ("card" 匹配)

返回: [Piece("car", 3, "d"), Piece("card", 4, "")]
```

### 2.5 后缀匹配


这是最复杂的查找模式，找出所有以给定字符串为前缀的单词：

```cpp
std::vector<Piece> GetDownPieces(const std::string& str) const {
    std::vector<Piece> rs;
    if (Empty()) return rs;

    // 先定位到前缀对应的节点
    uint32_t pos = 0;
    for (size_t i = 0; i < str.size(); ++i) {
        Node unit = array_[pos];
        pos ^= unit.state ^ static_cast<uint8_t>(str[i]);

        if (pos >= size_) return rs;

        unit = array_[pos];
        if (unit.character != static_cast<uint8_t>(str[i]))
            return rs;
    }

    // 从这个位置开始深度优先搜索
    std::string cw = str;
    CollectDownPieces(pos, cw, rs);
    return rs;
}
```

**深度优先搜索实现**：

```cpp
void CollectDownPieces(uint32_t pos, std::string& cw, 
                       std::vector<Piece>& rs) const {
    if (pos >= size_) return;

    Node unit = array_[pos];

    // 如果当前位置是单词结尾，添加到结果
    if (unit.eow) {
        uint32_t value_pos = pos ^ unit.state;
        if (value_pos < size_ && array_[value_pos].Value()) {
            Node value_node = array_[value_pos];
            rs.emplace_back(value_node.value, cw.size(), cw);
        }
    }

    // 遍历所有可能的子节点（利用 XOR 的遍历特性）
    uint32_t state = unit.state;
    for (int ct = 0; ct <= 255; ++ct) {
        uint32_t new_pos = pos ^ state ^ ct;

        if (new_pos >= size_) continue;
        if (new_pos == pos) continue;

        Node new_unit = array_[new_pos];

        // 验证是否为有效子节点
        if (new_unit.character == ct && new_unit.character != '\0'
            && new_unit.up == pos) {
            cw.push_back(static_cast<char>(ct));
            CollectDownPieces(new_pos, cw, rs);
            cw.pop_back();
        }
    }
}
```

**示例**：输入前缀 "ca"，字典包含 {"cat", "car", "card", "can"}

```
1. 定位到 "ca" 对应的节点位置 pos=99

2. 从这个位置开始 DFS：
   遍历字符 0-255:
   
   ct=116('t'): 
     new_pos = 99 ^ state ^ 116 = 126
     验证 array_[126].character == 't' ✓
     验证 array_[126].up == 99 ✓
     递归搜索，发现 "cat" 是单词结尾
     
   ct=114('r'):
     new_pos = 99 ^ state ^ 114 = 124  
     验证 array_[124].character == 'r' ✓
     验证 array_[124].up == 99 ✓
     递归搜索，发现 "car" 是单词结尾
     继续从 124 搜索，发现 "card"
     
   ct=110('n'):
     类似地发现 "can"

返回: [Piece("cat"), Piece("car"), Piece("card"), Piece("can")]
```

#### 2.6 Index 

`GetFreeIndex` 是 DoubleArray 实现中最核心的算法，它负责为每个节点的子节点集合分配一个合适的基址（index），确保所有子节点位置不冲突。这个算法的效率直接决定了 DoubleArray 的空间利用率和构建速度。

**基本要求**

给定一个字符集合 `{c1, c2, ..., cn}`，需要找到一个 `index`，使得：
- 所有位置 `index ^ c1, index ^ c2, ..., index ^ cn` 都未被占用
- 这些位置互不相同
- index 尽可能小，以提高空间局部性

**数学表达**

```cpp
对于字符集合 es = {e1, e2, ..., en}
找到最小的 index，满足：
∀i,j ∈ [1,n], i≠j => (index ^ ei) ≠ (index ^ ej)
且 ∀i ∈ [1,n] => uses[index ^ ei] == false
```

**具体实现**

涉及到的变量：
```cpp
std::vector<bool> uses_;
uint32_t prev_pos_;
```

```cpp
uint32_t GetFreeIndex(const std::vector<uint8_t>& es) {
    if (es.empty()) return 0;
    
    // 局部性优化：从上次分配位置附近开始搜索
    uint32_t start_index = (prev_pos_ > 256) ? prev_pos_ - 256 : 1;
    
    for (uint32_t i = start_index; ; ++i) {
        bool valid = true;
        
        // 检查所有子节点位置是否可用
        for (uint8_t e : es) {
            size_t p = i ^ e;
            EnsureSize(p + 1);
            
            if (uses_[p]) {
                valid = false;
                break;
            }
        }
        
        if (valid) {
            prev_pos_ = i;  // 记录这次分配位置，用于下次优化
            return i;
        }
    }
}
```

该函数本质上就是个线性探测法，由上次分配的位置附近继续向前探测，虽然实现的比较简单，但鉴于1）XOR较为分散的性质；2）一次性会把全组子节点都传过来；实际上的性能也是能接受的，更复杂的算法也存在，能把构建的过程更快一些。以及每一次Index分配，子节点占用的空间其实不会超过Index+256，也不会小于Index-256。


**输入**：`es = {'a'(97), 'e'(101), 'i'(105), 'o'(111), 'u'(117)}`

```cpp
尝试 index=10:
  'a' 位置: 10 ^ 97 = 107
  'e' 位置: 10 ^ 101 = 111  
  'i' 位置: 10 ^ 105 = 99
  'o' 位置: 10 ^ 111 = 101
  'u' 位置: 10 ^ 117 = 127
  
检查所有位置 {107, 111, 99, 101, 127}:
  - 互不相同 ✓
  - 都未被占用 ✓
  → 返回 index=10
```
