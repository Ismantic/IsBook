# BPE (Byte Pair Encoding) 分词算法详解

## 1. BPE基础原理

BPE (Byte Pair Encoding) 是一种数据压缩算法，最初用于文本压缩，后来被NLP领域广泛采用作为子词(Subword)分词方法。它的核心思想是**通过迭代地合并最频繁出现的字符对来构建词表**。

### 1.1 基本思路

1. 初始化词表：确定基础字符集合
2. 迭代地找出最频繁的相邻词对
3. 把词对合并成新的符号（新词）
4. 重复此过程直到达到预订的词汇表大小

### 1.2 具体示例

#### 步骤1：初始化词汇表
```
原始文本：["hello", "world", "hell", "low"]
初始词汇表：{h, e, l, o, w, r, d}
初始表示：["h e l l o", "w o r l d", "h e l l", "l o w"]
```

#### 步骤2：统计相邻字符对频率
```
字符对频率统计：
(h,e): 2次  (出现在"hello"和"hell"中)
(e,l): 2次  (出现在"hello"和"hell"中)
(l,l): 2次  (出现在"hello"和"hell"中)
(l,o): 3次  (出现在"hello", "hello", "low"中)
(w,o): 1次
(o,r): 1次
(r,l): 1次
(l,d): 1次
(o,w): 1次
```

#### 步骤3：合并最频繁的词对

最频繁的是`(l,o)`出现3次，将其合并为新符号`lo`：

```
更新后词汇表：{h, e, l, o, w, r, d, lo}
更新后表示：["h e l lo", "w o r l d", "h e l l", "lo w"]
```

#### 步骤4：重新统计并继续合并

```
新的字符对频率：
(h,e): 2次
(e,l): 2次  
(l,lo): 1次
(w,o): 1次
(o,r): 1次
(r,l): 1次
(l,d): 1次
(lo,w): 1次
```

继续选择最频繁的`(h,e)`合并为`he`：

```
词汇表：{h, e, l, o, w, r, d, lo, he}
表示：["he l lo", "w o r l d", "he l l", "lo w"]
```

#### 步骤5：重复直到达到目标词汇表大小

继续这个过程，可能得到：
```
最终词汇表：{h, e, l, o, w, r, d, lo, he, hel, hell, ...}
```

## 2. BPE Python实现及性能问题

### 2.1 朴素Python实现

```python
from collections import Counter

def BPE(texts, num_merges):
    # 将每个字符串转换为字符序列
    corpus = [' '.join(list(text)) for text in texts]
    
    for i in range(num_merges):
        # 统计所有相邻字符对的频率
        pairs = Counter()
        for text in corpus:
            tokens = text.split()
            for j in range(len(tokens)-1):
                pairs[(tokens[j], tokens[j+1])] += 1

        if not pairs: 
            break
            
        # 选择最频繁的字符对
        best_pair = pairs.most_common(1)[0][0]
        new_token = best_pair[0] + best_pair[1]

        # 在所有文本中替换该字符对
        corpus = [text.replace(
            f"{best_pair[0]} {best_pair[1]}", new_token)
                  for text in corpus]
        
        print(f"Iter {i+1}: Merge {best_pair} -> {new_token}")
        print(f"{corpus}\n")

    return corpus

# 测试
texts = ["hello", "world", "hell", "low"]
result = BPE(texts, 3)
```

**运行结果**：
```
Iter 1: Merge ('h', 'e') -> he
['he l l o', 'w o r l d', 'he l l', 'l o w']

Iter 2: Merge ('he', 'l') -> hel
['hel l o', 'w o r l d', 'hel l', 'l o w']

Iter 3: Merge ('hel', 'l') -> hell
['hell o', 'w o r l d', 'hell', 'l o w']
```

### 2.2 性能瓶颈分析

通过分析上述BPE实现，可以清楚地看到性能问题：

#### 主要瓶颈

1. **重复统计**：每次合并后都要重新扫描整个语料统计词对频率
2. **字符串操作**：大量的字符串查找、替换操作效率低下
3. **内存开销**：频繁的字符串创建和销毁

#### 时间复杂度分析

朴素实现的时间复杂度为**O(N×M×K)**，其中：
- N：文本总长度
- M：合并次数
- K：平均每次合并需要更新的位置数

**具体分析**：
- 每轮需要扫描整个文本统计频率：O(N)
- 需要进行M轮合并：×M
- 每次合并平均影响K个位置：×K

当处理大规模语料时（如GB级别文本），这种O(N×M×K)的复杂度变得不可接受。

#### 关键问题

```python
# 每次都要重新统计 - 大量重复计算
for text in corpus:
    tokens = text.split()
    for j in range(len(tokens)-1):
        pairs[(tokens[j], tokens[j+1])] += 1

# 字符串替换 - 效率低下
corpus = [text.replace(f"{best_pair[0]} {best_pair[1]}", new_token)
          for text in corpus]
```

**优化目标**：我们需要一种方法能够：
1. **增量更新**：只更新受影响的频率统计，而不是重新计算全部
2. **快速定位**：迅速找到所有需要合并的位置
3. **高效合并**：避免字符串操作，直接在数据结构上操作

## 3. 高效BPE实现

为了解决朴素实现的性能问题，我们设计了两个专门的数据结构：

- **Multiset**：解决频率统计问题（延迟更新 + 堆维护）
- **IndexedList**：解决位置定位问题（相邻对索引 + 链表操作）

通过这两个数据结构的配合，我们可以将BPE的时间复杂度优化到接近**O(N)**。

### 3.1 Multiset数据结构

#### 设计动机

传统方法每次合并后都要重新统计所有频率，但实际上：
- 大部分字符对的频率没有变化
- 只有涉及被合并字符对的相邻位置需要更新
- 可以通过增量更新避免重复计算

想象一下BPE训练时的常见场景：
```cpp
// 合并 (a,b) -> c 时，需要做大量统计更新
stats.Remove({prev, a});   // 移除旧的相邻对
stats.Remove({a, b});      // 移除被合并的对  
stats.Remove({b, next});   // 移除旧的相邻对
stats.Insert({prev, c});   // 添加新的相邻对
stats.Insert({c, next});   // 添加新的相邻对
```

如果每个操作都立即执行堆调整，性能会很糟糕。

#### 核心设计思想：延迟更新 + 堆维护

```cpp
template<typename T>
class Multiset {
private:
    std::vector<Node*> vec_;                    // 最大堆数组
    std::unordered_map<T, Node*> map_;          // 元素到节点的映射
    std::unordered_map<T, int> to_insert_;     // 待插入的元素
    std::unordered_map<T, int> to_remove_;     // 待删除的元素
};
```

**关键设计理念**：
- **延迟更新**：批量操作先缓存在`to_insert_`和`to_remove_`中
- **按需提交**：只在查询时才真正执行堆操作
- **双重索引**：数组实现堆，哈希表实现快速查找

#### Node节点设计

```cpp
class Node {
public:
    int count;    // 元素计数
    T value;      // 元素值
    int pos;      // 在堆数组中的位置

    bool operator<(const Node& o) const {
        return this->GetCount() < o.GetCount();  // 最大堆比较
    }
};
```

**设计要点**：
- `pos`字段：支持O(1)定位节点在堆中的位置
- 比较操作：实现最大堆语义
- 计数缓存：避免重复计算

#### 延迟更新机制

```cpp
void Insert(const T& item, int count = 1) {
    to_insert_[item] += count;  // 仅仅记录，不立即执行
}

void Remove(const T& item, int count = 1) {
    to_remove_[item] += count;  // 仅仅记录，不立即执行
}
```

**优势分析**：
- **批量优化**：多次操作同一元素时合并计算
- **性能提升**：避免频繁的堆调整操作
- **事务性**：所有操作在`_Commit()`时一次性生效

#### 核心机制：_Commit()函数详解

`_Commit()`函数是Multiset的核心，它负责将所有缓存的操作一次性提交到堆中：

```cpp
void _Commit() {
    // 先处理所有插入操作
    for (const auto& pair : to_insert_) {
        _Insert(pair.first, pair.second);
    }
    
    // 再处理所有删除操作
    for (const auto& pair : to_remove_) {
        _Remove(pair.first, pair.second);
    }
    
    // 清空缓存
    to_insert_.clear();
    to_remove_.clear();
}
```

**为什么先插入后删除？**

这个顺序很重要！考虑以下场景：
```cpp
multiset.Insert("apple", 5);
multiset.Remove("apple", 3);
```

如果先删除后插入，可能会出现删除不存在元素的情况。先插入确保元素存在后再删除。

**_Commit()的触发时机**：

```cpp
T Top() {
    _Commit();  // 查询前先提交
    return vec_.empty() ? T() : vec_[0]->value;
}

int GetCount(const T& item) {
    _Commit();  // 查询前先提交
    auto it = map_.find(item);
    return it == map_.end() ? 0 : it->second->count;
}
```

所有查询操作都会自动触发`_Commit()`，确保数据的一致性。

**完整工作示例**：

```cpp
Multiset<std::pair<int,int>> stats;

// 阶段1：只记录，不执行
stats.Insert({1,2}, 3);
stats.Insert({3,4}, 2);  
stats.Remove({1,2}, 1);
stats.Insert({1,2}, 2);

// 此时内部状态：
// vec_: []  (堆还是空的)
// to_insert_: {{1,2}: 5, {3,4}: 2}  (3+2=5)
// to_remove_: {{1,2}: 1}

// 阶段2：触发_Commit()
auto top = stats.Top();

// _Commit()执行过程：
// 1. _Insert({1,2}, 5) -> 堆中添加节点{1,2}:5
// 2. _Insert({3,4}, 2) -> 堆中添加节点{3,4}:2  
// 3. _Remove({1,2}, 1) -> {1,2}计数变为4
// 4. 清空缓存

// 最终堆状态：
// vec_[0]: {1,2}:4  (最大值)
// vec_[1]: {3,4}:2
```

#### 堆维护算法详解

**向上调整（_ItemIncrease）**：

当元素计数增加时，需要向上调整以维持最大堆性质：

```cpp
void _ItemIncrease(int pos) {
    Node* node = vec_[pos];
    
    // 向上冒泡直到满足最大堆性质
    while (pos > 0) {
        int uppos = (pos - 1) >> 1;  // 父节点位置
        Node* up = vec_[uppos];
        
        if (*up < *node) {           // 父节点更小，违反最大堆性质
            vec_[pos] = up;          // 父节点下移
            up->pos = pos;           // 更新父节点的位置信息
            pos = uppos;             // 当前节点上移
        } else {
            break;  // 堆性质已满足
        }
    }
    
    // 将节点安放到最终位置
    vec_[pos] = node;
    node->pos = pos;
}
```

**向下调整（_ItemDecrease）**：

当元素计数减少时，需要向下调整：

```cpp
void _ItemDecrease(int pos) {
    int endpos = vec_.size();
    Node* node = vec_[pos];
    int downpos = 2 * pos + 1;  // 左子节点位置
    
    // 向下筛选
    while (downpos < endpos) {
        int rightpos = downpos + 1;
        
        // 选择计数较大的子节点
        if (rightpos < endpos && !(*vec_[rightpos] < *vec_[downpos])) {
            downpos = rightpos;
        }
        
        Node* downnode = vec_[downpos];
        if (*node < *downnode) {     // 子节点更大，需要交换
            vec_[pos] = downnode;
            downnode->pos = pos;
            pos = downpos;
            downpos = 2 * pos + 1;
        } else {
            break;  // 堆性质已满足
        }
    }
    
    vec_[pos] = node;
    node->pos = pos;
}
```


#### 高级功能：TopK查询

除了基本的频率统计，Multiset还支持高效的TopK查询，这在某些BPE变体中很有用：

```cpp
std::vector<std::pair<T, int>> TopK(int k) {
    _Commit();  // 确保数据是最新的
    
    if (vec_.empty()) return {};
    
    // 使用最小堆来维护TopK候选
    using HeapItem = std::tuple<int, T, int, int>;  // -count, value, count, pos
    std::priority_queue<HeapItem, std::vector<HeapItem>, std::greater<HeapItem>> heap;
    
    // 从根节点开始
    heap.push(std::make_tuple(-vec_[0]->GetCount(), 
                              vec_[0]->value, 
                              vec_[0]->count, 
                              vec_[0]->pos));
    
    std::vector<std::pair<T, int>> result;
    
    for (int i = 0; i < k && !heap.empty(); ++i) {
        auto [_key, val, count, pos] = heap.top();
        heap.pop();
        result.emplace_back(val, count);
        
        // 将子节点加入堆中进行下一轮比较
        for (int child_pos : {pos * 2 + 1, pos * 2 + 2}) {
            if (child_pos < vec_.size()) {
                heap.push(std::make_tuple(
                    -vec_[child_pos]->GetCount(),
                     vec_[child_pos]->value,
                     vec_[child_pos]->count,
                     child_pos
                ));
            }
        }
    }
    return result;
}
```

**为什么不能直接取堆数组的前K个元素？**

这是一个常见的误解。让我们用具体例子说明：

```
最大堆结构：
           100
          /    \
        80      90
       /  \    /  \
     75   70  85   60

数组存储：[100, 80, 90, 75, 70, 85, 60]
如果直接取前3个：[100, 80, 90]
但实际Top3应该是：[100, 90, 85]

问题：第3大的元素85在数组的第6位！
```

**TopK算法的核心思想**：

1. **懒惰遍历**：不遍历整个堆，只访问可能包含TopK的节点
2. **辅助最小堆**：维护候选节点，确保按大小顺序输出
3. **逐层扩展**：每取出一个最大值，就将其子节点加入候选集

**算法步骤详解**：

```
初始状态：candidates = [100]

第1轮：
- 取出100（最大值）
- 加入100的子节点：candidates = [80, 90]
- 结果：[100]

第2轮：  
- 取出90（当前最大值）
- 加入90的子节点：candidates = [80, 85, 60]
- 结果：[100, 90]

第3轮：
- 取出85（当前最大值）  
- 85是叶子节点，无子节点
- 结果：[100, 90, 85]
```

这种方法的时间复杂度是O(K log K)，远优于排序整个堆的O(N log N)。
```

**关键点**：每次交换都要更新节点的`pos`字段，这样才能保持位置信息的准确性。

#### 智能缓存合并示例

```cpp
// 这些操作会被智能合并：
Insert("apple");
Insert("apple"); 
Remove("apple");
Insert("apple");

// 最终效果等同于：
Insert("apple", 2);  // 3次插入 - 1次删除 = 净增加2
```

#### 延迟批处理的性能优势

```
普通方案：
Insert("a") -> 立即堆操作 O(log n)
Insert("a") -> 立即堆操作 O(log n)  
Insert("a") -> 立即堆操作 O(log n)
总计：3 * O(log n)

Multiset方案：
Insert("a") -> 只记录 O(1)
Insert("a") -> 只记录 O(1) 
Insert("a") -> 只记录 O(1)
Top() -> 一次性处理：Insert("a", 3) -> O(log n)
总计：O(log n)
```

#### 完整实现

```cpp
template<typename T>
class Multiset {
public:
    Multiset() = default;
    ~Multiset() {
        for (auto node : vec_) {
            delete node;
        }
    }

    void Insert(const T& item, int count = 1) {
        to_insert_[item] += count;
    }

    void Remove(const T& item, int count = 1) {
        to_remove_[item] += count;
    }

    int GetCount(const T& item) {
        _Commit();
        auto it = map_.find(item);
        return it == map_.end() ? 0 : it->second->count;
    }

    T Top() {
        _Commit();
        return vec_.empty() ? T() : vec_[0]->value;
    }

    explicit operator bool() const {
        const_cast<Multiset*>(this)->_Commit();
        return !vec_.empty();
    }

private:
    class Node {
    public:
        int count;
        T value;
        int pos;

        Node(int count, const T& value, int pos)
            : count(count), value(value), pos(pos) {}
        
        int GetCount() const { return count; }

        bool operator<(const Node& o) const {
            return this->GetCount() < o.GetCount();
        }
    };

    std::vector<Node*> vec_;
    std::unordered_map<T, Node*> map_; 
    std::unordered_map<T, int> to_insert_;
    std::unordered_map<T, int> to_remove_;

    void _Insert(const T& item, int count = 1) {
        auto it = map_.find(item);
        if (it == map_.end()) {
            Node* node = new Node(0, item, vec_.size());
            map_[item] = node;
            vec_.push_back(node);
            it = map_.find(item);
        }
        it->second->count += count;
        _ItemIncrease(it->second->pos);
    }
    
    void _Remove(const T& item, int count = 1) {
        auto it = map_.find(item);
        if (it != map_.end()) {
            it->second->count -= count;
            _ItemDecrease(it->second->pos);
        }
    }
    
    void _Commit() {
        for (const auto& pair : to_insert_) {
            _Insert(pair.first, pair.second);
        }
        for (const auto& pair : to_remove_) {
            _Remove(pair.first, pair.second);
        }
        to_insert_.clear();
        to_remove_.clear();
    }
    
    void _ItemIncrease(int pos) {
        Node* node = vec_[pos];
        while (pos > 0) {
            int uppos = (pos - 1) >> 1;
            Node* up = vec_[uppos];
            if (*up < *node) {
                vec_[pos] = up;
                up->pos = pos;
                pos = uppos;
                continue;
            }
            break;
        }
        vec_[pos] = node;
        node->pos = pos;
    }
    
    void _ItemDecrease(int pos) {
        int endpos = vec_.size();
        Node* node = vec_[pos]; 
        int downpos = 2*pos + 1;
        while (downpos < endpos) {
            int rightpos = downpos + 1;
            if (rightpos < endpos && !(*vec_[rightpos] < *vec_[downpos])) {
                downpos = rightpos;
            }
            Node* downnode = vec_[downpos];
            if (*node < *downnode) {
                vec_[pos] = downnode;
                downnode->pos = pos;
                pos = downpos;
                downpos = 2*pos + 1;
            } else {
                break;
            }
        }
        vec_[pos] = node;
        node->pos = pos;
    }
};
```

Multiset本质上是一个支持重复元素的优先队列，通过延迟更新机制大大提升了BPE训练中的频率统计效率。

### 3.2 IndexedList数据结构

#### 设计动机

BPE合并操作需要：
1. **快速定位**：找到所有文本中(a,b)出现的位置
2. **高效合并**：将相邻的a和b合并成新的token c  
3. **动态更新**：合并后需要更新相邻对的统计信息
4. **批量操作**：一次合并可能影响成千上万个位置

传统方法需要扫描整个文本查找位置，而IndexedList通过维护"相邻对→位置"的索引实现O(1)定位。

#### 核心思想：相邻对索引

```cpp
template<typename T>
class IndexedList {
    Node* start_;                    // 链表头
    std::unordered_map<
        std::pair<T, T>, 
        std::vector<Node*>,
        PairHash
    > index_;                        // 相邻对 -> 节点位置的映射
};
```

**关键洞察**：维护一个从"相邻对"到"该对在链表中所有出现位置"的索引！

#### 节点设计

```cpp
class Node {
public:
    T value;        // token值
    Node* prev;     // 前驱节点
    Node* next;     // 后继节点
    
    void Delete() {
        if (prev) prev->next = next;
        if (next) next->prev = prev;
        next = prev = nullptr;
    }
};
```

#### 索引设计

```cpp
std::unordered_map<std::pair<T, T>, std::vector<Node*>, PairHash> index_;

// 例如：index_[{2, 3}] = [node_ptr1, node_ptr2, node_ptr3, ...]
// 表示相邻对(2,3)在链表中出现的所有位置
```

**注意**：索引指向的是相邻对中**第一个**token所在的节点！

#### 完整示例：构建过程

假设我们要表示文本"hello"，对应token序列：[8, 5, 12, 12, 15]

```cpp
// 1. 构建链表
IndexedList<int> list({8, 5, 12, 12, 15});

// 2. 链表结构
start_ -> [8] <-> [5] <-> [12] <-> [12] <-> [15]
          n1     n2      n3       n4      n5

// 3. 构建相邻对索引
index_[{8, 5}]   = [n1]     // (8,5)在n1位置出现
index_[{5, 12}]  = [n2]     // (5,12)在n2位置出现  
index_[{12, 12}] = [n3]     // (12,12)在n3位置出现
index_[{12, 15}] = [n4]     // (12,15)在n4位置出现
```

#### 索引构建过程

```cpp
template<typename Iterator>
IndexedList(Iterator begin, Iterator end) {
    if (begin == end) {
        start_ = nullptr;
        return;
    }

    // 创建第一个节点
    auto it = begin;
    T a = *it;
    start_ = new Node(a, nullptr, nullptr);
    Node* prev_node = start_;

    // 创建后续节点并建立索引
    ++it;
    while (it != end) {
        T b = *it;
        Node* node = new Node(b, prev_node, nullptr);
        prev_node->next = node;
        
        // 关键：为相邻对(a,b)建立索引
        InsertToIndex(std::make_pair(a, b), prev_node);
        
        a = b;                    // 滚动窗口
        prev_node = node;
        ++it;
    }
}
```

#### 快速查询操作

```cpp
// 快速找到所有(12,12)出现的位置
auto& nodes = list.GetIndex({12, 12});
// 返回：[n3]，表示在n3位置有(12,12)对
```

#### 核心机制：RemoveIndex()函数详解

`RemoveIndex()`函数负责从索引中移除指定节点涉及的所有相邻对，这是合并操作的关键步骤：

```cpp
void RemoveIndex(Node* node) {
    // 1. 移除涉及当前节点作为后一个元素的相邻对
    if (node->prev) {
        auto pair = std::make_pair(node->prev->value, node->value);
        auto it = index_.find(pair);
        if (it != index_.end()) {
            auto& nodes = it->second;
            // 从向量中移除对应的节点指针
            nodes.erase(
                std::remove(nodes.begin(), nodes.end(), node->prev),
                nodes.end()
            );
            if (nodes.empty()) {
                index_.erase(it);  // 如果没有更多出现，删除整个条目
            }
        }
    }
    
    // 2. 移除涉及当前节点作为前一个元素的相邻对
    if (node->next) {
        auto pair = std::make_pair(node->value, node->next->value);
        auto it = index_.find(pair);
        if (it != index_.end()) {
            auto& nodes = it->second;
            nodes.erase(
                std::remove(nodes.begin(), nodes.end(), node),
                nodes.end()
            );
            if (nodes.empty()) {
                index_.erase(it);
            }
        }
    }
}
```

**关键理解**：每个节点参与两个相邻对
- 作为**后一个元素**：`(prev->value, node->value)`，索引指向`prev`节点
- 作为**前一个元素**：`(node->value, next->value)`，索引指向`node`节点

**详细示例**：

```
链表状态：
... [10] <-> [12] <-> [15] <-> [8] ...
     n1      n2       n3      n4

索引状态：
index_[{10,12}] = [n1]  // n2作为后元素，索引指向n1
index_[{12,15}] = [n2]  // n3作为后元素，索引指向n2  
index_[{15,8}]  = [n3]  // n4作为后元素，索引指向n3

执行 RemoveIndex(n2)：
1. 移除 index_[{10,12}] 中的 n1 指针
2. 移除 index_[{12,15}] 中的 n2 指针

执行 RemoveIndex(n3)：  
1. 移除 index_[{12,15}] 中的 n2 指针 (可能已移除)
2. 移除 index_[{15,8}] 中的 n3 指针
```

#### 核心机制：UpdateIndex()函数详解

`UpdateIndex()`函数在节点合并后重新建立索引，确保新的相邻对能被正确索引：

```cpp
void UpdateIndex(Node* node) {
    // 1. 为左侧相邻对建立新索引
    if (node->prev) {
        InsertToIndex(
            std::make_pair(node->prev->value, node->value), 
            node->prev  // 注意：索引指向相邻对的第一个节点
        );
    }
    
    // 2. 为右侧相邻对建立新索引
    if (node->next) {
        InsertToIndex(
            std::make_pair(node->value, node->next->value), 
            node        // 注意：索引指向相邻对的第一个节点
        );
    }
}
```

**索引指向规则**：对于相邻对`(a,b)`，索引始终指向值为`a`的节点（第一个节点）。

**完整合并示例**：

```
初始状态：
[5] <-> [12] <-> [12] <-> [15]
 n1      n2       n3      n4

索引：
index_[{5,12}]  = [n1]  // 指向第一个12的前一个节点
index_[{12,12}] = [n2]  // 指向第一个12
index_[{12,15}] = [n3]  // 指向第二个12

合并(12,12) -> 24的过程：

步骤1：RemoveIndex(n2)
- 移除 index_[{5,12}] 中的 n1
- 移除 index_[{12,12}] 中的 n2

步骤2：RemoveIndex(n3)  
- 移除 index_[{12,12}] 中的 n2 (已移除)
- 移除 index_[{12,15}] 中的 n3

步骤3：物理合并
n2->value = 24;  // 第一个12变成24
删除 n3;         // 删除第二个12

步骤4：UpdateIndex(n2)
- 添加 index_[{5,24}] = [n1]   // 新的左侧相邻对
- 添加 index_[{24,15}] = [n2]  // 新的右侧相邻对

最终状态：
[5] <-> [24] <-> [15]
 n1      n2      n4

索引：
index_[{5,24}]  = [n1]
index_[{24,15}] = [n2]
```

#### 索引维护的核心原则

1. **一致性**：索引必须与链表状态保持同步
2. **完整性**：每个相邻对都要有对应的索引条目
3. **准确性**：索引指向的节点必须是相邻对的第一个节点
4. **及时性**：链表变化后立即更新索引

这种精确的索引维护机制使得IndexedList能够在O(1)时间内定位任意相邻对的所有出现位置，这是整个BPE优化的关键所在。

#### 完整实现

```cpp
template<typename T>
class IndexedList {
public:
    class Node {
    public:
        T value;
        Node* prev;
        Node* next;

        Node(const T& value, Node* prev, Node* next) 
            : value(value), prev(prev), next(next) {}
        
        void Delete() {
            if (prev) prev->next = next;
            if (next) next->prev = prev;
            next = prev = nullptr;
        }
    };

    struct PairHash {
        template <class T1, class T2>
        std::size_t operator() (const std::pair<T1,T2>& p) const {
            auto s1 = std::hash<T1>{}(p.first);
            auto s2 = std::hash<T2>{}(p.second);
            return s1 ^ s2;
        }
    };

private:
    Node* start_;
    std::unordered_map<std::pair<T, T>, std::vector<Node*>, PairHash> index_;

public:
    template<typename Iterator>
    IndexedList(Iterator begin, Iterator end) {
        if (begin == end) {
            start_ = nullptr;
            return;
        }

        auto it = begin;
        T a = *it;
        start_ = new Node(a, nullptr, nullptr);
        Node* prev_node = start_;

        ++it;
        while (it != end) {
            T b = *it;
            Node* node = new Node(b, prev_node, nullptr);
            prev_node->next = node;
            InsertToIndex(std::make_pair(a, b), prev_node);
            a = b;
            prev_node = node;
            ++it;
        }
    }

    std::vector<Node*>& GetIndex(const std::pair<T,T>& pair) {
        return index_[pair];
    }

    void InsertToIndex(const std::pair<T, T>& pair, Node* node) {
        index_[pair].push_back(node);
    }

    void RemoveIndex(Node* node) {
        if (node->prev) {
            auto pair = std::make_pair(node->prev->value, node->value);
            auto it = index_.find(pair);
            if (it != index_.end()) {
                auto& nodes = it->second;
                nodes.erase(
                    std::remove(nodes.begin(), nodes.end(), node->prev),
                    nodes.end()
                );
                if (nodes.empty()) {
                    index_.erase(it);
                }
            }
        }
        
        if (node->next) {
            auto pair = std::make_pair(node->value, node->next->value);
            auto it = index_.find(pair);
            if (it != index_.end()) {
                auto& nodes = it->second;
                nodes.erase(
                    std::remove(nodes.begin(), nodes.end(), node),
                    nodes.end()
                );
                if (nodes.empty()) {
                    index_.erase(it);
                }
            }
        }
    }

    void UpdateIndex(Node* node) {
        if (node->prev) {
            InsertToIndex(std::make_pair(node->prev->value, node->value), node->prev);
        }
        if (node->next) {
            InsertToIndex(std::make_pair(node->value, node->next->value), node);
        }
    }

    // 析构函数、拷贝构造等...
};
```

IndexedList本质上是一个带索引的双向链表，专门为高效支持BPE的合并操作而设计。

### 3.3 完整BPE实现

有了Multiset和IndexedList这两个数据结构，我们可以实现高效的BPE训练算法。

#### InitPairStats函数：初始化统计

```cpp
static Multiset<std::pair<int,int>> InitPairsStats(
        const std::vector<std::string>& texts) {
    Multiset<std::pair<int,int>> stats;
    std::vector<int> bytes;
    
    for (const auto& text : texts) {
        bytes.clear();
        // 将字符转换为int（因为整个词表使用int）
        for (uint8_t c : text) {
            bytes.push_back(static_cast<int>(c));
        }
        
        // 统计相邻字符对
        for (size_t i = 0; i < bytes.size()-1; i++) {
            stats.Insert({bytes[i], bytes[i+1]});
        }
    }
    return stats;
}
```

**作用**：
- 将文本转换为int序列（统一词表表示）
- 使用Multiset统计所有相邻token对的频率
- 返回初始的频率统计结果

#### BuildIndexedList函数：构建索引链表

```cpp
static IndexedList<int> BuildIndexedList(const std::string& text) {
    std::vector<int> bytes;
    for (unsigned char c : text) {
        bytes.push_back(static_cast<int>(c));
    }
    return IndexedList<int>(bytes.begin(), bytes.end());
}
```

**作用**：
- 将单个文本转换为int序列
- 构建IndexedList，建立相邻对到位置的索引
- 为每个文本建立独立的索引链表

#### BPE训练主循环

```cpp
// 核心训练过程
auto stats = InitPairsStats(texts);
std::vector<IndexedList<int>> indexed_lists;
indexed_lists.reserve(texts.size());

// 为每个文本构建IndexedList
for (const auto& text : texts) {
    indexed_lists.push_back(BuildIndexedList(text));
}

int num_merges = vocab_size_ - meta_pieces_.size();
int cnt = 0;

// 初始化基础词汇表（256个字节）
for (int i = 0; i < 256; i++) {
    std::string t(1, i);
    vocab_[i] = t;
    cnt += 1;
}

// 主训练循环
while (cnt < num_merges && stats) {
    // 1. 找到最频繁的token对
    auto top = stats.Top();
    int n = stats.GetCount(top);
    
    // 2. 创建新的token
    int new_id = vocab_.size();
    merge_tree_.emplace_back(top, new_id);
    vocab_[new_id] = vocab_[top.first] + vocab_[top.second];
    
    // 3. 在所有文本中执行合并
    for (auto& list : indexed_lists) {
        Merge(top, new_id, list, &stats);
    }
    
    cnt += 1;
}
```

**关键步骤**：
1. **频率查询**：通过`stats.Top()`获取最频繁的token对
2. **词汇扩展**：创建新token并更新词汇表
3. **批量合并**：在所有文本的IndexedList中执行合并
4. **统计更新**：Merge函数自动维护频率统计

#### Merge函数详解

Merge函数是整个BPE实现的核心，它巧妙地结合了IndexedList和Multiset的优势。

```cpp
static void Merge(const std::pair<int, int>& pair, int new_id,
                 IndexedList<int>& indexed_list,
                 Multiset<std::pair<int, int>>* stats = nullptr) {

    // 第1步：通过索引快速找到所有匹配的位置
    auto& nodes = indexed_list.GetIndex(pair);
    
    // 第2步：遍历所有出现位置，逐个合并
    for (auto* node : nodes) {
        // 第3步：验证节点有效性
        if (node->value != pair.first ||
            node->next == nullptr ||
            node->next->value != pair.second) {
            continue;  // 跳过无效或已处理的节点
        }
        
        // 第4步：更新索引（移除旧的相邻对）
        indexed_list.RemoveIndex(node);
        indexed_list.RemoveIndex(node->next);

        // 第5步：更新统计信息（如果提供了stats）
        if (stats != nullptr) {
            // 减少当前被合并对的计数
            stats->Remove(pair);
            
            // 更新右侧相邻对的统计
            if (node->next->next != nullptr) {
                stats->Remove({node->next->value, node->next->next->value});
                stats->Insert({new_id, node->next->next->value});
            }
            
            // 更新左侧相邻对的统计
            if (node->prev != nullptr) {
                stats->Remove({node->prev->value, pair.first});
                stats->Insert({node->prev->value, new_id});
            }
        }
        
        // 第6步：执行物理合并
        auto* remove = node->next;
        node->next->Delete();  // 从链表中删除节点
        delete remove;         // 释放内存
        node->value = new_id;  // 更新当前节点的值
        
        // 第7步：重建索引（添加新的相邻对）
        indexed_list.UpdateIndex(node);
    }
}
```

#### Merge函数详细步骤解析

##### 步骤1：快速定位

```cpp
auto& nodes = indexed_list.GetIndex(pair);
```

**关键理解**：这是整个函数效率的核心！

通过IndexedList的索引机制，我们可以在O(1)时间内获取所有指定token对的位置，而不需要扫描整个文本。

```cpp
// 传统方法：O(N)扫描
for (int i = 0; i < text.size()-1; i++) {
    if (text[i] == pair.first && text[i+1] == pair.second) {
        // 找到一个位置
    }
}

// IndexedList方法：O(1)直接获取
auto& nodes = indexed_list.GetIndex({12, 15});
// 返回：[node_ptr1, node_ptr2, ...]，指向所有(12,15)对的第一个token位置
```

##### 步骤2-3：遍历与验证

```cpp
for (auto* node : nodes) {
    if (node->value != pair.first ||
        node->next == nullptr ||
        node->next->value != pair.second) {
        continue;
    }
    // ...
}
```

**为什么需要验证？**

1. **并发处理**：如果之前的合并已经改变了这个位置
2. **边界情况**：节点可能已被删除或修改
3. **数据一致性**：确保索引和实际链表状态一致

##### 步骤4：索引维护

```cpp
indexed_list.RemoveIndex(node);
indexed_list.RemoveIndex(node->next);
```

**作用**：从索引中移除即将被合并的节点涉及的所有相邻对。

**详细过程**：
```
合并前的链表：
... [10] <-> [12] <-> [15] <-> [8] ...
     prev    node    next     next2

需要移除的索引条目：
- index_[{10, 12}] 中的 prev指针
- index_[{12, 15}] 中的 node指针  
- index_[{15, 8}] 中的 next指针
```

##### 步骤5：统计更新

这是最复杂的部分，需要更新4种统计：

**5.1 减少被合并对的计数**
```cpp
stats->Remove(pair);  // (12,15) 的计数 -1
```

**5.2 更新右侧相邻对**
```cpp
if (node->next->next != nullptr) {
    stats->Remove({node->next->value, node->next->next->value});  // 移除 (15,8)
    stats->Insert({new_id, node->next->next->value});             // 添加 (42,8)
}
```

**5.3 更新左侧相邻对**
```cpp
if (node->prev != nullptr) {
    stats->Remove({node->prev->value, pair.first});  // 移除 (10,12)
    stats->Insert({node->prev->value, new_id});      // 添加 (10,42)
}
```

**统计更新的完整图示**：

```
合并前：
... [10] <-> [12] <-> [15] <-> [8] ...
相邻对：  (10,12)  (12,15)  (15,8)

合并 (12,15) -> 42：
... [10] <-> [42] <-> [8] ...
相邻对：  (10,42)      (42,8)

统计变化：
- (12,15): count -= 1  ← 被合并的对
- (10,12): count -= 1  ← 左侧旧对
- (15,8):  count -= 1  ← 右侧旧对
- (10,42): count += 1  ← 左侧新对
- (42,8):  count += 1  ← 右侧新对
```

##### 步骤6：物理合并

```cpp
auto* remove = node->next;
node->next->Delete();  // 断开链表连接
delete remove;         // 释放内存
node->value = new_id;  // 更新token值
```

**Delete()函数的作用**：
```cpp
void Delete() {
    if (prev) prev->next = next;  // 前驱指向后继
    if (next) next->prev = prev;  // 后继指向前驱
    next = prev = nullptr;        // 清空自己的指针
}
```

**合并过程图示**：
```
步骤1 - 合并前：
... [10] <-> [12] <-> [15] <-> [8] ...
     prev    node    remove   next2

步骤2 - 执行Delete()：
remove->Delete() 使得:
... [10] <-> [12]     [15] <-> [8] ...
     prev    node              next2
              ↓
            连接断开

步骤3 - 更新node：
node->value = 42;
... [10] <-> [42] <-> [8] ...
     prev    node     next2
```

##### 步骤7：重建索引

```cpp
indexed_list.UpdateIndex(node);
```

**作用**：为合并后的新节点重新建立相邻对索引。

**UpdateIndex()的工作**：
```cpp
void UpdateIndex(Node* node) {
    // 重建左侧相邻对索引
    if (node->prev) {
        InsertToIndex({node->prev->value, node->value}, node->prev);
    }
    
    // 重建右侧相邻对索引
    if (node->next) {
        InsertToIndex({node->value, node->next->value}, node);
    }
}
```

#### 完整Merge执行示例

##### 初始状态
```
文本："hello" -> tokens: [8, 5, 12, 12, 15]

链表：
[8] <-> [5] <-> [12] <-> [12] <-> [15]
 n1     n2      n3       n4      n5

索引：
index_[{5, 12}]  = [n2]
index_[{12, 12}] = [n3] 
index_[{12, 15}] = [n4]

统计：
(5,12): 1次, (12,12): 1次, (12,15): 1次
```

##### 执行：Merge({12, 12}, 24, list, &stats)

**第1步：定位**
```cpp
auto& nodes = indexed_list.GetIndex({12, 12});  // 返回 [n3]
```

**第2-3步：验证**
```cpp
node = n3;  // node->value = 12 ✓
node->next = n4;  // node->next->value = 12 ✓
// 验证通过
```

**第4步：移除索引**
```cpp
RemoveIndex(n3):  移除 index_[{5,12}] 中的 n2
                  移除 index_[{12,12}] 中的 n3
RemoveIndex(n4):  移除 index_[{12,12}] 中的 n3 (已移除)
                  移除 index_[{12,15}] 中的 n4
```

**第5步：更新统计**
```cpp
stats->Remove({12, 12});        // (12,12): 1 -> 0
stats->Remove({12, 15});        // (12,15): 1 -> 0  
stats->Remove({5, 12});         // (5,12): 1 -> 0
stats->Insert({24, 15});        // (24,15): 0 -> 1
stats->Insert({5, 24});         // (5,24): 0 -> 1
```

**第6步：物理合并**
```cpp
// 合并前：[5] <-> [12] <-> [12] <-> [15]
//               n2      n3       n4      n5

n4->Delete();    // 删除第二个12
delete n4;       // 释放内存
n3->value = 24;  // 第一个12变成24

// 合并后：[5] <-> [24] <-> [15]
//               n2      n3      n5
```

**第7步：重建索引**
```cpp
UpdateIndex(n3):
  InsertToIndex({5, 24}, n2);   // 添加 index_[{5,24}] = [n2]
  InsertToIndex({24, 15}, n3);  // 添加 index_[{24,15}] = [n3]
```

##### 最终状态
```
链表：
[8] <-> [5] <-> [24] <-> [15]
 n1     n2      n3       n5

索引：
index_[{8, 5}]   = [n1]
index_[{5, 24}]  = [n2]
index_[{24, 15}] = [n3]

统计：
(8,5): 1次, (5,24): 1次, (24,15): 1次
```

**总结**：增量更新的本质
IndexedList:解决“在哪里"的问题
```
问题：找到(a,b)在文本中的所有位置
传统：for循环遍历 - O(n)
方案：维护 (a,b) → [位置1, 位置2, ...] 的映射 - O(1)
```
Multiset：解决"多频繁"的问题
```
问题：快速知道哪个相邻对最频繁
传统：每次重新统计所有对 - O(n)
方案：维护动态的频率排序，增量更新 - O(log k)
```
关键洞察：合并(a,b)→c 只影响3种相邻对的统计：

(a,b) 本身：次数-1
(x,a) 模式：变成 (x,c)
(b,y) 模式：变成 (c,y)

假设我们要合并相邻对 (e,l) → el：
```
影响前：...t h e l l o w...
相邻对：(t,h) (h,e) (e,l) (l,l) (l,o) (o,w)

影响后：...t h el l o w...  
相邻对：(t,h) (h,el) (el,l) (l,o) (o,w)

统计变化：
- (e,l): -1  ← 被合并的对
- (h,e): -1  ← 左邻旧对
- (l,l): -1  ← 右邻旧对
+ (h,el): +1  ← 左邻新对
+ (el,l): +1  ← 右邻新对
```
正确理解BPE合并
BPE每次合并：

输入：两个相邻的token A B
输出：一个新token AB
影响：只有这两个token左右紧邻的相邻对会改变

这就是为什么是"增量"更新 - 每次只影响很小的局部区域！

TODO: PieceTokenizer的整体介绍
以及，还是要引用一下 Multiset/IndexedList的原文