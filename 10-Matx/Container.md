# Runtime Container

## 概述

运行时容器系统提供了一套完整的类型安全容器类。

**关键点**
1. **Node/Container 分离**: Node 类负责数据存储，Container 类提供用户接口
2. **统一对象模型**: 所有 Node 类继承自 `object_t`，所有 Container 类继承自 `object_r`
3. **类型安全**: 模板化设计确保编译时和运行时类型安全
4. **RAII 语义**: 自动内存管理和生命周期控制
5. **STL 兼容**: 提供符合 STL 标准的迭代器和接口

**具体容器**
| 容器类型 | Node 类 | Container 类 | 底层存储 | 主要用途 |
|---------|---------|-------------|----------|----------|
| 字符串 | `StrNode` | `Str` | `std::string` | 字符串处理，COW 优化 |
| 数组 | `ArrayNode` | `Array<T>` | `std::vector<object_r>` | 动态数组，类型安全 |
| 映射 | `MapNode` | `Map<K,V>` | `std::unordered_map<object_r, object_r>` | 类型化键值映射 |
| 字典 | `DictNode` | `Dict` | `std::unordered_map<McValue, McValue>` | 键值映射 |
| 集合 | `SetNode` | `Set` | `std::unordered_set<McValue>` | 唯一元素集合 |
| 列表 | `ListNode` | `List` | `std::vector<McValue>` | 动态列表 |
| 元组 | `TupleNode` | `Tuple` | `McValue* data_` | 不可变序列 |

**实现模式**
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Node 类       │    │  Container 类   │    │   用户接口      │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│ 继承 object_t   │    │ 继承 object_r   │    │ STL 兼容接口    │
│ 数据存储        │ -> │ 智能指针包装    │ -> │ 类型安全操作    │
│ 基础操作        │    │ 高级操作        │    │ 迭代器支持      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## StrNode/Str

### StrNode

字符串数据类：

```cpp
class StrNode : public object_t {
public:
    static constexpr const int32_t INDEX = TypeIndex::RuntimeStr;
    static constexpr const std::string_view NAME = "RuntimeStr";
    DEFINE_TYPEINDEX(StrNode, object_t);

    // 构造和赋值
    StrNode() = default;
    explicit StrNode(container_type data) : data_(std::move(data)) {}
    StrNode(const StrNode& other) : data_(other.data_) {}
    StrNode& operator=(const StrNode& other);

    // 基础访问接口
    size_type size() const noexcept { return data_.size(); }
    bool empty() const noexcept { return data_.empty(); }
    const char* data() const noexcept { return data_.data(); }
    const char* c_str() const noexcept { return data_.c_str(); }

private:
    std::string data_;  // 底层字符串存储

    friend class Str;
};
```

**设计特点**:
- **最小化接口**: 只提供基础的数据访问方法
- **标准兼容**: 基于 `std::string` 实现
- **友元访问**: `Str` 类可以直接访问 `data_` 成员

### Str

字符串容器接口：

```cpp
class Str : public object_r {
public:
    using node_type = StrNode;
    using value_type = char;
    using size_type = size_t;

    // 构造函数
    Str();                                    // 空字符串
    Str(const char* str);                     // C 字符串
    Str(const std::string& str);              // std::string
    explicit Str(object_p<object_t> obj);     // 对象包装

    // 访问接口
    size_type size() const noexcept;
    bool empty() const noexcept;
    const char* data() const noexcept;
    const char* c_str() const noexcept;

    // 比较操作
    bool operator==(const Str& other) const noexcept;
    bool operator!=(const Str& other) const noexcept;

    // COW 支持 - Copy-on-Write 优化
    DEFINE_COW_METHOD(StrNode)

    // 类型转换
    operator std::string_view() const noexcept;

private:
    const StrNode* Get() const;
    StrNode* GetMutable();
};
```

**Copy-on-Write (COW) 机制**:
```cpp
// DEFINE_COW_METHOD 宏的实际展开
StrNode* CopyOnWrite() {
    if (!data_.unique()) {                                    // 如果不是独占访问
        auto n = MakeObject<StrNode>(*(operator->()));       // 创建当前对象的副本
        object_p<object_t>(std::move(n)).swap(data_);         // 替换当前引用
    }
    return static_cast<StrNode*>(data_.get());               // 返回可修改的指针
}
```

**字符串操作符重载**:
```cpp
// 字符串连接
inline Str operator+(const Str& lhs, const Str& rhs) {
    std::string result;
    result.reserve(lhs.size() + rhs.size());  // 预分配内存
    result.append(lhs.data(), lhs.size());
    result.append(rhs.data(), rhs.size());
    return Str(result);
}

// 流输出
inline std::ostream& operator<<(std::ostream& os, const Str& str) {
    return os.write(str.data(), str.size());
}

// 哈希支持 (标准库特化)
template<>
struct std::hash<mc::runtime::Str> {
    size_t operator()(const mc::runtime::Str& str) const noexcept {
        return std::hash<std::string_view>{}(std::string_view(str));
    }
};
```

## ArrayNode/Array

### ArrayNode

类型擦除数组节点：


```cpp
class ArrayNode : public object_t {
public:
    static constexpr const int32_t INDEX = TypeIndex::RuntimeArray;
    static constexpr const std::string_view NAME = "Array";
    DEFINE_TYPEINDEX(ArrayNode, object_t);

    using value_type = object_r;               // 类型擦除的对象引用
    using container_type = std::vector<value_type>;
    using const_iterator = typename container_type::const_iterator;

    // 修改操作
    void push_back(object_r value) { data_.push_back(std::move(value)); }
    void clear();

    // 访问操作
    size_t size() const { return data_.size(); }
    size_t capacity() const { return data_.capacity(); }
    const object_r& operator[](size_t i) const { return data_[i]; }

    // 迭代器
    const_iterator begin() const { return data_.begin(); }
    const_iterator end() const { return data_.end(); }

    // 内存管理
    object_r* MutableBegin() { return data_.data(); }
    static object_p<ArrayNode> Empty(int64_t size);

private:
    std::vector<object_r> data_;
};
```

**特殊的清理机制**:
```cpp
void ArrayNode::clear() {
    // 显式调用 object_r 析构函数，确保引用计数正确递减
    for (size_t i = 0; i < data_.size(); ++i) {
        data_[i].object_r::~object_r();
    }
    data_.clear();
}
```

### Array<T>

类型安全数组容器：

```cpp
template<typename T>
class Array : public object_r {
public:
    using value_type = T;
    using node_type = ArrayNode;

    // 构造函数
    Array();                                          // 空数组
    Array(std::initializer_list<T> init);             // 初始化列表
    Array(const std::vector<T>& init);                // std::vector
    explicit Array(object_p<object_t> obj);           // 对象包装

    // 元素访问
    T operator[](size_t i) const {
        return Downcast<T>(Get()->operator[](i));     // 类型安全的向下转换
    }

    // 修改操作
    void push_back(const T& value) { Get()->push_back(value); }

    // 容量信息
    size_t size() const { return Get()->size(); }

private:
    ArrayNode* Get() const;
};
```

**类型安全的赋值机制**:
```cpp
template <typename IterType>
void Array<T>::Assign(IterType first, IterType last) {
    int64_t cap = std::distance(first, last);
    ArrayNode* p = Get();
    
    // 优化: 如果当前数组独占且容量足够，直接重用
    if (p != nullptr && data_.unique() && p->capacity() >= cap) {
        p->clear();
    } else {
        // 创建新的数组节点
        data_ = ArrayNode::Empty(cap);
        p = Get();
    }
    
    // 使用 placement new 直接构造元素
    object_r* itr = p->MutableBegin();
    for (int64_t i = 0; i < cap; ++i, ++first, ++itr) {
        new (itr) object_r(*first);  // placement new 避免额外的移动
    }
}

// placement new 语法详解：
// new (address) Type(constructor_args)
//     |         |    |
//     |         |    └── 构造函数参数
//     |         └─────── 要构造的类型
//     └─────────────────── 指定的内存地址
```

**placement new 语法解析**:

```cpp
new (itr) object_r(*first);

// 等价的分步操作：
// 1. itr 是已分配内存的地址（object_r* 类型）
// 2. object_r(*first) 是构造函数调用，以 *first 为参数
// 3. 在 itr 指向的内存位置直接构造 object_r 对象

// 对比普通 new：
object_r* obj1 = new object_r(*first);        // 普通 new：分配内存 + 构造对象
new (itr) object_r(*first);                   // placement new：在指定内存构造对象
```

**为什么使用 placement new**:

1. **避免重复分配**: 内存已由 `ArrayNode::Empty(cap)` 预分配
2. **性能优化**: 避免额外的内存分配和释放开销
3. **精确控制**: 直接在 vector 的内部缓冲区构造对象

**内存布局对比**:

```cpp
// 传统方式（低效）：
std::vector<object_r> data_(cap);           // 1. 分配内存 + 默认构造 cap 个对象
for (int i = 0; i < cap; ++i) {
    data_[i] = object_r(*first++);          // 2. 赋值操作（可能触发析构+构造）
}

// placement new 方式（高效）：
std::vector<object_r> data_;
data_.resize(cap);                          // 1. 分配内存但不构造对象
object_r* itr = data_.data();
for (int i = 0; i < cap; ++i, ++first, ++itr) {
    new (itr) object_r(*first);             // 2. 直接在预分配位置构造
}
```

**类型转换链解析**:

Array 中的 `Get()` 方法包含了必要的双重类型转换：

```cpp
ArrayNode* Get() const {
    return static_cast<ArrayNode*>(const_cast<object_t*>(get()));
}

// 转换步骤分解：
// 1. get() 返回 const object_t*（因为 Get() 方法声明为 const）
// 2. const_cast<object_t*>(...) 移除 const 限定符
// 3. static_cast<ArrayNode*>(...) 进行向下类型转换
```

**为什么需要这种转换**:

1. **const 语义问题**: 
   - `Array` 的很多方法声明为 `const`（如 `operator[]`, `size()`）
   - 但这些方法仍需要访问和修改底层 `ArrayNode` 数据
   - `object_r::get()` 返回 `const object_t*` 保护基类接口
   - `const_cast` 在容器层面获得修改权限

2. **类型安全转换**:
   - 基类指针 `object_t*` 向下转换为派生类指针 `ArrayNode*`
   - 这是安全的，因为构造时已验证类型匹配
   - 如果类型不匹配，构造函数会抛出 `runtime_error`

3. **运行时类型检查**:
```cpp
explicit Array(object_p<object_t> obj) : object_r(std::move(obj)) {
    if (data_ && !data_->IsType<ArrayNode>()) {
        throw std::runtime_error("Type mismatch: expected Array");
    }
}
```

**类型安全迭代器**:
```cpp
class Array<T>::const_iterator {
public:
    using iterator_category = std::forward_iterator_tag;
    using value_type = T;
    using difference_type = std::ptrdiff_t;
    using pointer = const T*;
    using reference = const T;

    value_type operator*() const { 
        return Downcast<T>(*it_);  // 运行时类型转换
    }

    const_iterator& operator++() { ++it_; return *this; }
    bool operator==(const const_iterator& other) const { return it_ == other.it_; }

private:
    node_type::const_iterator it_;
};
```

## MapNode/Map

### MapNode

类型擦除映射数据类：

```cpp
class MapNode : public object_t {
public:
    static constexpr const int32_t INDEX = TypeIndex::RuntimeMap;
    static constexpr const std::string_view NAME = "Map";
    DEFINE_TYPEINDEX(MapNode, object_t);

    using key_type = object_r;
    using mapped_type = object_r;
    using value_type = std::pair<key_type, mapped_type>;
    using container_type = std::unordered_map<key_type, mapped_type, object_s, object_e>;
    using const_iterator = typename container_type::const_iterator;

    // 元素访问
    const mapped_type& at(const key_type& key) const;
    mapped_type& at(const key_type& key);
    mapped_type& operator[](const key_type& key) { return data_[key]; }

    // 修改操作
    void set(const key_type& key, const mapped_type& value) { data_[key] = value; }
    void set(key_type&& key, mapped_type&& value) { data_[std::move(key)] = std::move(value); }
    void erase(const key_type& key) { data_.erase(key); }

    // 查询操作
    size_t count(const key_type& key) const { return data_.count(key); }
    size_t size() const { return data_.size(); }
    const_iterator find(const key_type& key) const { return data_.find(key); }

    // 迭代器
    const_iterator begin() const { return data_.begin(); }
    const_iterator end() const { return data_.end(); }

private:
    container_type data_;
};
```

**异常安全的访问**:
```cpp
const mapped_type& MapNode::at(const key_type& key) const {
    auto it = data_.find(key);
    if (it == data_.end()) {
        throw std::out_of_range("Key not found in Map");  // 明确的异常信息
    }
    return it->second;
}
```

### Map<K,V>

类型安全映射容器：
```cpp
template<typename K, typename V>
class Map : public object_r {
public:
    using key_type = K;
    using mapped_type = V;
    using node_type = MapNode;

    // 构造函数
    Map() { data_ = MakeObject<MapNode>(); }
    Map(std::initializer_list<std::pair<K, V>> init);
    template <typename Hash = object_s, typename Equal = object_e>
    explicit Map(const std::unordered_map<K, V, Hash, Equal>& init);

    // 元素访问
    V at(const K& key) const { return Downcast<V>(Get()->at(key)); }
    V operator[](const K& key) const {
        auto* node = Get();
        auto it = node->find(key);
        if (it == node->end()) {
            return V();  // 不存在的键返回默认值
        }
        return Downcast<V>(it->second);
    }

    // 修改操作
    void set(const K& key, const V& value) { Get()->set(key, value); }
    void set(K&& key, V&& value) { Get()->set(std::move(key), std::move(value)); }
    void erase(const K& key) { Get()->erase(key); }

    // 查询操作
    size_t count(const K& key) const { return Get()->count(key); }
    size_t size() const { return Get()->size(); }
    bool empty() const { return size() == 0; }

private:
    MapNode* Get() const { return static_cast<MapNode*>(const_cast<object_t*>(get())); }
};
```


**类型安全迭代器**:
```cpp
template<typename K, typename V>
class Map<K,V>::const_iterator {
public:
    using iterator_category = std::forward_iterator_tag;
    using value_type = std::pair<K, V>;
    using difference_type = std::ptrdiff_t;
    using pointer = const value_type*;
    using reference = const value_type;

    const_iterator(node_type::const_iterator it) : it_(it) {}

    value_type operator*() const {
        return std::make_pair(
            Downcast<K>(it_->first),   // 键类型转换
            Downcast<V>(it_->second)   // 值类型转换
        );
    }

    const_iterator& operator++() { ++it_; return *this; }
    bool operator==(const const_iterator& other) const { return it_ == other.it_; }

private:
    node_type::const_iterator it_;
};
```


## DictNode/Dict

### DictNode

字典数据类：

```cpp
class DictNode : public object_t {
public:
    static constexpr const int32_t INDEX = TypeIndex::RuntimeDict;
    static constexpr const std::string_view NAME = "Dict";
    DEFINE_TYPEINDEX(DictNode, object_t);

    using container_type = std::unordered_map<McValue, McValue>;
    using iterator = typename container_type::iterator;
    using const_iterator = typename container_type::const_iterator;
    using value_type = typename container_type::value_type;
    using key_type = typename container_type::key_type;
    using mapped_type = typename container_type::mapped_type;

    // 构造函数
    DictNode() = default;
    template <typename Iterator>
    DictNode(Iterator s, Iterator e) {
        for (; s != e; ++s) {
            data_.emplace(s->first, s->second);
        }
    }

    // 元素访问
    mapped_type& operator[](const key_type& key) { return data_[key]; }
    const_iterator find(const key_type& key) const { return data_.find(key); }
    bool contains(const key_type& key) const { return find(key) != data_.end(); }

    // 容量信息
    size_t size() const { return data_.size(); }
    bool empty() const { return data_.empty(); }
    void clear() { data_.clear(); }

    // 迭代器
    iterator begin() { return data_.begin(); }
    const_iterator begin() const { return data_.begin(); }
    iterator end() { return data_.end(); }
    const_iterator end() const { return data_.end(); }

private:
    container_type data_;
};
```

### Dict


```cpp
class Dict : public object_r {
public:
    using container_type = DictNode::container_type;
    using value_type = typename container_type::value_type;
    using iterator = typename container_type::iterator;
    using const_iterator = typename container_type::const_iterator;

    // 特殊迭代器类型
    using item_iterator = item_iterator_<iterator>;
    using item_const_iterator = item_iterator_<const_iterator>;
    using key_iterator = key_iterator_<iterator>;
    using key_const_iterator = key_iterator_<const_iterator>;
    using value_iterator = value_iterator_<iterator>;
    using value_const_iterator = value_iterator_<const_iterator>;

    // 构造函数
    Dict() : object_r(MakeObject<DictNode>()) {}
    Dict(std::initializer_list<value_type> init);
    template<typename Iterator>
    Dict(Iterator s, Iterator e);

    // 元素操作
    void insert(const McValue& key, const McValue& value);
    bool contains(const McValue& key) const;
    McValue& operator[](const McValue& key);

    // 迭代器访问
    iterator begin() const { return Get()->data_.begin(); }
    iterator end() const { return Get()->data_.end(); }

private:
    DictNode* Get() const { return static_cast<DictNode*>(data_.get()); }
};
```


字典系统提供了三种专用迭代器，用于分别访问键值对、键、值：

```cpp
// 1. 项目迭代器 - 返回完整的键值对
template <class I>
struct item_iterator_ : public I {
    using value_type = typename I::value_type;
    value_type operator*() const {
        return I::operator*();  // 直接返回 std::pair<key, value>
    }
};

// 2. 键迭代器 - 只返回键
template <class I>
struct key_iterator_ : public I {
    using value_type = typename I::value_type::first_type;
    value_type operator*() const {
        return (*this)->first;  // 只返回键部分
    }
};

// 3. 值迭代器 - 只返回值
template <class I>
struct value_iterator_ : public I {
    using value_type = typename I::value_type::second_type;
    value_type operator*() const {
        return (*this)->second;  // 只返回值部分
    }
};
```

**视图包装器**:
```cpp
// 提供类似 Python dict.keys(), dict.values(), dict.items() 的功能
template <typename D>
struct DictKeys {
    using iterator = typename D::key_const_iterator;
    D data;
    
    DictKeys(const D& data) : data(data) {}
    iterator begin() const { return data.key_begin(); }
    iterator end() const { return data.key_end(); }
    int64_t size() const { return data.size(); }
};

template <typename D>
struct DictValues {
    using iterator = typename D::value_const_iterator;
    D data;
    
    DictValues(const D& data) : data(data) {}
    iterator begin() const { return data.value_begin(); }
    iterator end() const { return data.value_end(); }
    int64_t size() const { return data.size(); }
};
```

## SetNode/Set

### SetNode

集合数据类：

```cpp
class SetNode : public object_t {
public:
    static const int32_t INDEX = TypeIndex::RuntimeSet;
    static constexpr const std::string_view NAME = "Set";
    DEFINE_TYPEINDEX(SetNode, object_t);

    using value_type = McValue;
    using container_type = std::unordered_set<value_type>;
    using iterator = typename container_type::const_iterator;  // 注意：只读迭代器
    using const_iterator = iterator;

    // 构造函数
    SetNode() = default;
    SetNode(std::initializer_list<value_type> init) : data_(init) {}
    template <typename Iterator>
    SetNode(Iterator s, Iterator e) {
        for (auto it = s; it != e; ++it) {
            data_.insert(*it);
        }
    }

    // 修改操作
    bool insert(const value_type& value) { return data_.insert(value).second; }
    bool insert(value_type&& value) { return data_.insert(std::move(value)).second; }
    size_t erase(const value_type& value) { return data_.erase(value); }
    void clear() { data_.clear(); }

    // 查询操作
    bool contains(const value_type& value) const { return data_.find(value) != data_.end(); }
    size_t size() const { return data_.size(); }
    bool empty() const { return data_.empty(); }

    // 迭代器
    const_iterator begin() const { return data_.begin(); }
    const_iterator end() const { return data_.end(); }

private:
    container_type data_;
};
```

### Set

集合容器接口：

```cpp
class Set : public object_r {
public:
    using Node = SetNode;
    using value_type = Node::value_type;
    using const_iterator = Node::const_iterator;

    // 构造函数
    Set() = default;
    Set(std::initializer_list<value_type> init) { data_ = MakeObject<Node>(init); }
    template<typename Iterator>
    Set(Iterator s, Iterator e) { data_ = MakeObject<Node>(s, e); }

    // 元素操作
    bool insert(const value_type& value) const;
    template<typename T>
    bool insert(const T& value) const { return insert(value_type(value)); }
    
    bool contains(const value_type& value) const;
    template<typename T>
    bool contains(const T& value) const { return contains(value_type(value)); }

    // 集合运算
    Set set_union(const Set& other) const;      // 并集
    Set set_minus(const Set& other) const;      // 差集

    // 容量信息
    bool empty() const;
    size_t size() const;
    void clear() const;

    // 迭代器
    const_iterator begin() const { return Get()->begin(); }
    const_iterator end() const { return Get()->end(); }

private:
    Node* Get() const { return static_cast<Node*>(const_cast<object_t*>(get())); }
};
```

**集合运算实现**:
```cpp
Set Set::set_union(const Set& other) const {
    auto result = Set();
    if (empty()) {
        result = other;  // 直接复制
    } else if (!other.empty()) {
        result = *this;  // 复制当前集合
        for (const auto& item : other) {
            result.insert(item);  // 添加另一个集合的元素
        }
    }
    return result;
}

Set Set::set_minus(const Set& other) const {
    auto result = Set();
    if (!empty()) {
        for (const auto& item : *this) {
            if (!other.contains(item)) {  // 只添加不在另一个集合中的元素
                result.insert(item);
            }
        }
    }
    return result;
}
```

## ListNode/List

### ListNode

列表数据类：

```cpp
class ListNode : public object_t {
public:
    static constexpr uint32_t INDEX = TypeIndex::RuntimeList;
    static constexpr const std::string_view NAME = "List";
    DEFINE_TYPEINDEX(ListNode, object_t);

    using value_type = McValue;
    using container_type = std::vector<value_type>;
    using iterator = typename container_type::iterator;
    using const_iterator = typename container_type::const_iterator;

    // 构造函数
    ListNode() = default;
    ListNode(std::initializer_list<value_type> init) : data_(init) {}
    template <typename Iterator>
    ListNode(Iterator s, Iterator e) : data_(s, e) {}
    ListNode(size_t n, const value_type& value) : data_(n, value) {}

    // 元素访问
    value_type& operator[](int64_t i) { return data_[i]; }
    const value_type& operator[](int64_t i) const { return data_[i]; }
    value_type& at(int64_t i) {
        if (i < 0) {
            i += data_.size();  // 支持负数索引（Python 风格）
        }
        return data_.at(i);
    }

    // 修改操作
    void push_back(const value_type& value) { data_.push_back(value); }
    void push_back(value_type&& value) { data_.push_back(std::move(value)); }
    template<typename T>
    void append(T&& value) { push_back(value_type(std::forward<T>(value))); }
    void pop_back() { data_.pop_back(); }
    void clear() { data_.clear(); }

    // 容量管理
    size_t size() const { return data_.size(); }
    bool empty() const { return data_.empty(); }
    void reserve(size_t new_size) { data_.reserve(new_size); }

    // 迭代器
    iterator begin() { return data_.begin(); }
    const_iterator begin() const { return data_.begin(); }
    iterator end() { return data_.end(); }
    const_iterator end() const { return data_.end(); }

private:
    container_type data_;
};
```

### List

列表容器接口：

```cpp
class List : public object_r {
public:
    using Node = ListNode;
    using value_type = Node::value_type;
    using iterator = Node::iterator;
    using const_iterator = Node::const_iterator;

    // 构造函数
    List() = default;
    List(std::initializer_list<value_type> init) { data_ = MakeObject<Node>(init); }
    List(size_t n, const value_type& value) { data_ = MakeObject<Node>(n, value); }

    // 元素访问
    value_type& operator[](int64_t i) const { return Get()->at(i); }

    // 修改操作 (注意 const 方法)
    void push_back(const value_type& val) const;
    template<typename T>
    void append(T&& val) const;
    void pop_back() const;
    void clear() const;

    // 容量信息
    size_t size() const;
    bool empty() const;

    // 迭代器
    iterator begin() const { return Get()->begin(); }
    iterator end() const { return Get()->end(); }

private:
    Node* Get() const { return static_cast<Node*>(const_cast<object_t*>(get())); }
};
```

**Python 风格的负数索引**:
```cpp
value_type& ListNode::at(int64_t i) {
    if (i < 0) {
        i += data_.size();  // -1 表示最后一个元素，-2 表示倒数第二个
    }
    return data_.at(i);     // 使用 std::vector::at 进行边界检查
}
```

## TupleNode/Tuple

### TupleNode

不可变序列数据类：

```cpp
class TupleNode : public object_t {
public:
    static constexpr const int32_t INDEX = TypeIndex::RuntimeTuple;
    static constexpr const std::string_view NAME = "Tuple";
    DEFINE_TYPEINDEX(TupleNode, object_t);

    using value_type = McValue;
    using iterator = const value_type*;          // 只读原生指针迭代器
    using const_iterator = const value_type*;

    // 构造函数 - 从迭代器范围构造
    TupleNode(const value_type* s, const value_type* e) : size_(e - s) {
        data_ = new value_type[size_];           // 原生数组分配
        for (size_t i = 0; i < size_; ++i) {
            data_[i] = s[i];                     // 逐个复制元素
        }
    }

    // 析构函数 - 手动释放内存
    ~TupleNode() {
        delete[] data_;
    }

    // 访问接口
    size_t size() const noexcept { return size_; }
    const value_type* data() const noexcept { return data_; }
    value_type* data() noexcept { return data_; }

    // 迭代器
    const_iterator begin() const noexcept { return data_; }
    const_iterator end() const noexcept { return data_ + size_; }

private:
    size_t size_;           // 元素数量
    value_type* data_;      // 原生数组指针
};
```

### Tuple

元祖容器接口：

```cpp
class Tuple : public object_r {
public:
    using Node = TupleNode;
    using value_type = Node::value_type;
    using iterator = Node::const_iterator;       // 元组是不可变的
    using const_iterator = Node::const_iterator;

    // 构造函数
    Tuple() = default;
    Tuple(std::initializer_list<value_type> init) : Tuple(init.begin(), init.end()) {}
    template<typename Iterator>
    Tuple(Iterator s, Iterator e) {
        std::vector<value_type> t;
        std::copy(s, e, std::back_inserter(t));            // 先复制到临时 vector
        data_ = MakeObject<Node>(t.data(), t.data()+t.size());  // 再构造 Node
    }

    // 元素访问 (只读)
    const value_type& operator[](size_t i) const {
        auto node = static_cast<const Node*>(get());
        return node->data()[i];
    }

    // 容量信息
    size_t size() const noexcept {
        if (!get()) return 0;
        return static_cast<const Node*>(get())->size();
    }

    // 迭代器 (只读)
    const_iterator begin() const noexcept {
        if (!get()) return nullptr;
        return static_cast<const Node*>(get())->begin();
    }

    const_iterator end() const noexcept {
        if (!get()) return nullptr;
        return static_cast<const Node*>(get())->end();
    }

    // 比较操作
    bool operator==(const Tuple& other) const noexcept {
        if (size() != other.size()) return false;
        return std::equal(begin(), end(), other.begin());
    }
};
```

**元组的不可变性设计**:
- 元组一旦创建，内容不可修改
- 所有访问方法都是 `const` 的
- 迭代器类型是 `const_iterator`
- 没有提供修改操作的接口

## 总结

### 数据分离

所有容器都遵循相同的设计模式：

```cpp
// Node 类职责
class XxxNode : public object_t {
    // 1. 继承 object_t，参与引用计数系统
    // 2. 定义类型索引和名称常量
    // 3. 包装底层 STL 容器
    // 4. 提供基础的数据操作接口
    // 5. 管理实际的数据存储
};

// Container 类职责  
class Xxx : public object_r {
    // 1. 继承 object_r，提供智能指针语义
    // 2. 包装对应的 Node 类
    // 3. 提供类型安全的高级接口
    // 4. 实现 STL 兼容的迭代器
    // 5. 提供领域特定的操作方法
};
```

### 类型系统

所有容器都完全集成到 MC 的类型系统中：

```cpp
// 类型注册
static constexpr const int32_t INDEX = TypeIndex::RuntimeXxx;
static constexpr const std::string_view NAME = "Xxx";
DEFINE_TYPEINDEX(XxxNode, object_t);

// 类型安全转换
template<typename T>
T Downcast(const object_r& obj) {
    // 运行时类型检查 + 静态类型转换
}

// 对象构造
auto container = MakeObject<XxxNode>(constructor_args...);
```

### 内存管理

1. **引用计数**: 所有 Node 类通过 `object_t` 参与自动引用计数
2. **智能指针**: Container 类通过 `object_r` 提供 RAII 语义
3. **COW 优化**: 字符串类型支持 Copy-on-Write 优化
4. **异常安全**: 所有操作都提供强异常安全保证

## 举例

### Str

```cpp
// 创建和操作字符串
Str s1("Hello");
Str s2(" World");
Str s3 = s1 + s2;        // "Hello World"

// COW 优化演示
Str s4 = s1;             // 共享底层存储，不复制
Str s5 = s1;             // 继续共享
// s1.GetMutable();      // 这时才会触发复制
```

### Array

```cpp
// 创建类型化数组
Array<Str> arr = {"1", "2", "3", "4", "5"};
arr.push_back("6");

// 类型安全访问
Str val = arr[0];        // 自动类型转换
for (auto x : arr) {     // 类型安全迭代
    std::cout << x << " ";
}
```

### Map

```cpp
// 创建类型化映射
Map<Str, Str> map = {
    {"apple", "1"}, // 隐式转换
    {"banana", "2"},
    {"cherry", "3"}
};

// 类型安全访问
Str count = map["apple"];        // 返回 Str 类型
map.set("date", "4");              // 类型检查

// 类型安全迭代
for (const auto& [fruit, count] : map) {
    std::cout << fruit << ": " << count << std::endl;
}
```


### Dict

```cpp
// 创建字典
Dict dict = {
    {McValue("key1"), McValue(42)},
    {McValue("key2"), McValue("value")}
};

// 访问元素
dict.insert(McValue("key3"), McValue(3.14));
bool exists = dict.contains(McValue("key1"));

// 迭代字典
for (const auto& [key, value] : dict) {
    // 处理键值对
}
```

### Set

```cpp
// 创建集合
Set set1 = {McValue(1), McValue(2), McValue(3)};
Set set2 = {McValue(3), McValue(4), McValue(5)};

// 集合运算
Set union_set = set1.set_union(set2);     // {1, 2, 3, 4, 5}
Set diff_set = set1.set_minus(set2);      // {1, 2}
```

