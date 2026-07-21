# 容器与迭代器

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

## Node 与引用类

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

## 静态容器与动态容器

`Array<T>` 和 `Map<K, V>` 保存经过类型擦除的 `object_r`，但在接口处通过模板恢复类型。它们主要服务于 AST 等类型明确的内部结构。例如，`Array<PrimVar>` 表示一组参数，读取元素时会向下转换为 `PrimVar`。

`List`、`Dict`、`Set` 和 `Tuple` 则保存 `McValue`。它们允许整数、浮点数、字符串和其他运行时对象出现在同一个容器中，更接近 Python 的动态语义：

```cpp
List values{McValue(42), McValue(3.14), McValue("matx")};
values.append(true);

Dict config;
config[McValue("debug")] = McValue(true);
```

两组容器不是重复实现：前者在 C++ 编译期提供类型约束，后者在运行时保留动态类型。

## List 与 Tuple

`List` 使用 `std::vector<McValue>`，支持追加、删除末尾元素和清空。索引由 `ListNode::at` 完成，并接受负数索引：`-1` 会先转换为最后一个元素的位置，再交给 `std::vector::at` 检查边界。

`Tuple` 只公开读取和遍历接口。`TupleNode` 在构造时复制元素，并用连续数组保存它们。接口上的不可修改性使它适合表示固定参数、字典条目等结构。

## Dict 与 Set

`Dict` 和 `Set` 建立在哈希表之上，因此 `McValue` 必须提供相等比较和哈希规则。字典保存键值对，集合只保存唯一的值：

```cpp
Dict env{{McValue("x"), McValue(10)}};
bool found = env.contains(McValue("x"));

Set labels{McValue("B"), McValue("I")};
labels.insert(McValue("E"));
```

`Dict` 还提供 items、keys 和 values 三类迭代视图。它们共享同一个哈希表迭代器，只改变解引用时返回键值对、键还是值，对应 Python 中的 `dict.items()`、`dict.keys()` 和 `dict.values()`。

## 两层迭代机制

当前代码中存在两层迭代机制。第一层是容器直接提供的 `begin()` 和 `end()`。它兼容 C++ 范围循环，也是目前完整可用的遍历方式：

```cpp
for (const McValue& value : values) {
    // 使用 value
}
```

第二层是类型擦除的运行时 `Iterator`。`IteratorNode` 定义统一协议：

```cpp
virtual bool HasNext() const = 0;
virtual McValue Next() = 0;
virtual int64_t Distance() const = 0;
```

`GenericIteratorNode` 用回调适配具体遍历过程，并把原容器保存在一个 `McValue` 中。这样即使外部只持有 `Iterator`，容器也不会在遍历结束前被释放。

不过，`Iterator::MakeGenericIterator(const Any&)` 和 `MakeItemsIterator` 目前仍未实现按容器类型自动分派。因此，运行时已经具备统一迭代协议和通用回调适配器，但 `List`、`Dict` 等容器到该协议的入口尚未完全接通。现阶段应使用各容器自身的迭代器；后续补全适配后，解释器或生成代码便可用同一套协议处理不同容器。
