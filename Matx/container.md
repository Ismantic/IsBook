# Container 数据容器

## 引言

运行时不仅要传递整数和浮点数，还要表示下面这样的 Python 数据：

```python
record = {
    "name": "matx",
    "scores": [91, 87, 95],
    "tags": {"compiler", "runtime"},
}

for name, value in record.items():
    print(name, value)
```

这些容器可以嵌套不同类型的值，还需要支持索引、查找和遍历。Matx 没有直接把 `std::vector` 或 `std::unordered_map` 暴露给编译后的程序，而是在运行时对象体系之上实现了自己的容器。

从运行时看，`record` 不是一块连续的数据，而是一组通过 `McValue` 相互引用的对象：

```text
DictNode
├── "name"   → StrNode("matx")
├── "scores" → ListNode
│              └── [91, 87, 95]
└── "tags"   → SetNode
               └── {"compiler", "runtime"}
```

外层字典负责键值查找，列表和集合分别保存自己的元素；字符串、列表、集合和字典又都由运行时对象体系管理生命周期。容器需要解决的因此不只是“把元素放在一起”，还包括嵌套对象怎样持有、动态值怎样比较，以及不同容器怎样被统一遍历。

## 实现

Matx 的容器建立在运行时对象与 `McValue` 之上。类型明确的内部结构和类型动态的 Python 数据采用不同容器，但共享相同的对象生命周期与遍历基础。

**Node**

容器沿用运行时对象的两层结构：Node 类保存数据，引用类提供接口。

| 运行时类型 | Node | 引用类 | 底层存储 |
| --- | --- | --- | --- |
| 字符串 | `StrNode` | `Str` | `std::string` |
| 静态数组 | `ArrayNode` | `Array<T>` | `std::vector<object_r>` |
| 静态映射 | `MapNode` | `Map<K, V>` | `std::unordered_map<object_r, object_r>` |
| 列表 | `ListNode` | `List` | `std::vector<McValue>` |
| 字典 | `DictNode` | `Dict` | `std::unordered_map<McValue, McValue>` |
| 集合 | `SetNode` | `Set` | `std::unordered_set<McValue>` |
| 元组 | `TupleNode` | `Tuple` | 连续的 `McValue` 数组 |

例如，`ListNode` 继承 `object_t`，负责保存元素；`List` 继承 `object_r`，负责持有节点并转发操作：

```cpp
class ListNode : public object_t {
    std::vector<McValue> data_;
};

class List : public object_r {
public:
    McValue& operator[](int64_t i) const;
    void append(McValue value) const;
    size_t size() const;
};
```

这种分工让容器直接复用对象体系的引用计数和运行时类型检查，同时保持接近普通 C++ 容器的使用方式。

**Array 与 Map**

`Array<T>` 和 `Map<K, V>` 保存经过类型擦除的 `object_r`，但在接口处通过模板恢复类型。它们主要服务于 AST 等类型明确的内部结构。例如，一个函数的参数可以表示为：

```cpp
Array<PrimVar> params{a, b};
```

调用者只能向其中放入 `PrimVar`，读取元素时也直接得到 `PrimVar`。底层对象仍然通过 `object_r` 统一保存，模板接口则在 C++ 编译期保留元素约束。

`List`、`Dict`、`Set` 和 `Tuple` 则保存 `McValue`。它们允许整数、浮点数、字符串和其他运行时对象出现在同一个容器中，更接近 Python 的动态语义：

```cpp
List values{McValue(42), McValue(3.14), McValue("matx")};
values.append(true);

Dict config;
config[McValue("debug")] = McValue(true);
```

两组容器不是重复实现：前者在 C++ 编译期提供类型约束，后者在运行时保留动态类型。

```text
Array<PrimVar>：元素类型由 C++ 模板确定
List：          元素类型由每个 McValue 的标签确定
```

AST 结构优先使用前者，避免把类型错误推迟到运行时；编译后程序中的 Python 容器使用后者，保留混合存储不同值的能力。

**共享语义**

容器引用类的复制不会复制全部元素，而是共享同一个 Node：

```cpp
List a{McValue(1), McValue(2)};
List b = a;
b.append(3);
```

此时 `a` 和 `b` 都能看到第三个元素，因为二者持有同一个 `ListNode`。Node 的引用计数保证最后一个容器引用离开后才释放元素，元素中的 `McValue` 再分别处理立即数、字符串和对象的所有权。

这种浅复制符合 Python 可变对象的引用语义，也避免在函数传参时复制整个容器。需要注意的是，共享 Node 和复制 `McValue` 是两个不同层次：复制容器引用只增加 Node 的引用计数，向容器中插入元素则会按照 `McValue` 的规则复制或移动该元素。

**List 与 Tuple**

`List` 使用 `std::vector<McValue>`，支持追加、删除末尾元素和清空。索引由 `ListNode::at` 完成，并接受负数索引：`-1` 会先转换为最后一个元素的位置，再交给 `std::vector::at` 检查边界。这让生成代码可以保留 Python 常用的负索引行为，而不必在每个调用位置重复转换。

`Tuple` 只公开读取和遍历接口。`TupleNode` 在构造时复制元素，并用连续数组保存它们。接口上的不可修改性使它适合表示固定参数、字典条目等结构。List 与 Tuple 因此可以保存相同的 `McValue`，区别在于容器是否允许结构发生变化。

**Dict 与 Set**

`Dict` 和 `Set` 建立在哈希表之上，因此 `McValue` 必须提供相等比较和哈希规则。查找一个键时，运行时先根据类型和值计算哈希，再用相等比较确认是否为同一个键。字典保存键值对，集合只保存唯一的值：

```cpp
Dict env{{McValue("x"), McValue(10)}};
bool found = env.contains(McValue("x"));

Set labels{McValue("B"), McValue("I")};
labels.insert(McValue("E"));
```

`Dict` 还提供 items、keys 和 values 三类迭代视图。它们共享同一个哈希表迭代器，只改变解引用时返回键值对、键还是值，对应 Python 中的 `dict.items()`、`dict.keys()` 和 `dict.values()`。

**Iterator**

当前代码中存在两层迭代机制。第一层是容器直接提供的 `begin()` 和 `end()`。它兼容 C++ 范围循环，也是目前完整可用的遍历方式：

```cpp
for (const McValue& value : values) {
    // 使用 value
}
```

这种方式要求调用位置知道具体容器类型。函数注册或跨语言调用只拿到一个 `Any` 时，则需要不依赖 `List::iterator`、`Dict::iterator` 等具体 C++ 类型的遍历接口。第二层因此是类型擦除的运行时 `Iterator`。`IteratorNode` 定义统一协议：

```cpp
virtual bool HasNext() const = 0;
virtual McValue Next() = 0;
virtual int64_t Distance() const = 0;
```

`GenericIteratorNode` 用回调适配具体遍历过程，并把原容器保存在一个 `McValue` 中。这样即使外部只持有 `Iterator`，容器也不会在遍历结束前被释放。

`Iterator::MakeGenericIterator` 可以通过回调包装具体遍历过程，但接收 `Any` 并按运行时类型自动选择容器适配器的重载尚未接通。因此，当前完整可用的路径仍是各容器自身的 `begin()` 和 `end()`；类型擦除的 `Iterator` 只在调用方已经提供遍历回调时使用。

容器由此把上一章的对象和值真正组合起来：容器自身依靠对象体系获得身份和生命周期，内部元素通过 `McValue` 保留动态类型，迭代接口再把这些元素交给函数调用和后续程序处理。
