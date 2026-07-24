# Runtime 运行时

## 引言

编译器不只需要把加法、循环等语法翻译成 C++，还要决定程序中的值怎样存在。生成的代码必须知道一个值是整数、字符串还是容器，明确它占用多少内存、由谁持有，以及通过函数参数和返回值传递时怎样保留类型信息。否则，即使一条表达式能够被翻译成合法的 C++，程序也无法正确保存和交换其中的数据。

这些问题在 Python 中通常不会直接暴露。变量可以随时引用不同类型的值，容器可以混合保存整数和对象，字符串的内存也由运行时自动管理。先看一段普通的 Python 代码：

```python
count = 42
name = "matx"
items = [count, name]
result = items[1]
```

`count` 是整数，可以直接映射到 `int64_t`；`name` 是需要管理内存的字符串；`items` 又在同一个容器中保存了两种不同类型。取出 `items[1]` 时，底层代码还必须知道这个值是字符串，才能正确复制、返回和释放它。

如果只使用 C++ 原生类型，整数、字符串和容器需要完全不同的函数签名与存储方式。它们也很难通过同一套 C API 在 Python 和生成的动态库之间传递。Matx 的运行时就是这段 Python 代码与底层 C++ 表示之间的桥梁。

Matx 因此需要回答三个问题：

1. 不同类型的值怎样通过同一个函数接口传递？
2. 字符串、容器等动态对象由谁管理生命周期？
3. 只拿到一个通用值时，怎样判断它的实际类型？

在运行时中，整数和浮点数可以直接保存在一个带类型标签的联合体中：

```cpp
McValue number(42);
McValue ratio(0.5);
```

字符串、容器和 AST 节点则需要额外的内存或对象结构。运行时在堆上创建这些对象，管理它们的生命周期，再把对象指针放入通用值：

```cpp
Str text("matx");
McValue value(text);
```

为什么不只使用一套结构？一种做法是让所有值都继承同一个对象基类，包括整数和浮点数。这样类型关系很统一，但每个数字都要在堆上分配对象，并参与引用计数。前面的 `count = 42` 原本只需要复制八字节整数，现在却会变成一次对象创建和一次间接访问。大量算术运算会为这种统一付出不必要的成本。

另一种做法是把所有内容都直接放进 `Value`。固定大小的联合体可以保存整数、浮点数和指针，却无法直接容纳长度不定的字符串、列表或 AST 节点。如果让 `Value` 分别管理每一种复杂类型，它就必须知道所有对象的内存布局、继承关系和释放方式，最终会变成一个不断扩张的巨大分支。

Matx 因此采用两套相互配合的结构：

- **值体系**直接保存整数、浮点数等轻量值，并提供固定的 C ABI 布局。
- **对象体系**管理字符串、容器、AST 和动态模块等堆对象，负责引用计数、继承与运行时类型。
- 当复杂对象需要通过统一接口传递时，值体系只保存它的指针和类型索引。

这样，`count` 可以直接存在 `Value` 的联合体中，`items` 则由对象体系管理，`McValue` 仍能使用相同接口表示两者：

```text
对象体系
object_t ── object_p<T> ── object_r
   │          │               │
对象基类   管理引用计数      类型擦除引用
   │
   └──────────────┐
                  ▼
值体系       Value ── Any ── McView / McValue
             固定布局  类型接口   借用 / 拥有
```

`object_r` 适合在 C++ 内部以统一方式引用不同对象；`McValue` 则可以同时保存立即数和对象指针，适合函数参数、返回值与 FFI。对象体系负责身份、继承和生命周期，值体系负责统一传递。本章将分别解释它们，并说明两者如何在 `McValue` 中汇合。

## 实现

运行时需要分别实现对象管理、类型识别和值传递，再通过 `McValue` 将它们组合成统一接口。下面先从具有身份和生命周期的对象开始。

**Object**

`object_t` 是运行时对象的共同基类。AST 节点、字符串、容器和动态模块最终都继承自它：

```cpp
class object_t {
public:
    virtual ~object_t();
    virtual int32_t Index() const;
    virtual std::string Name() const;

    void IncCounter() noexcept { ++count_; }
    void DecCounter() noexcept {
        if (--count_ == 0) delete this;
    }

protected:
    int32_t t_{0};
    std::atomic<int32_t> count_{0};
};
```

对象内部保存类型索引和引用计数。Matx 没有直接用 `std::shared_ptr`，而是通过 `object_p<T>` 管理侵入式引用计数。复制 `object_p<T>` 时计数加一，析构或重新赋值时计数减一；移动则直接转移指针。

```cpp
auto node = MakeObject<StrNode>(std::string("matx"));
object_p<StrNode> copy = node;
```

`MakeObject<T>` 负责创建对象并写入运行时类型索引。对象的引用计数存放在对象本身，因此同一个对象无论经过何种包装，都共享同一份所有权状态。

`object_r` 在 `object_p<object_t>` 外再提供一层类型擦除。上层代码可以统一保存 `Object`，需要具体类型时再通过 `As<T>()` 检查：

```cpp
class object_r {
public:
    template<typename T>
    const T* As() const noexcept {
        if (data_ && data_->IsType<T>()) {
            return static_cast<const T*>(data_.get());
        }
        return nullptr;
    }
};
```

具体运行时类型通常采用 Node 与引用类分离的形式。例如 `StrNode` 保存 `std::string`，`Str` 则继承 `object_r`，向用户提供构造、访问和运算接口。这种结构也为写时复制提供了统一入口。

**TypeContext**

C++ 的 RTTI 只能处理 C++ 继承关系，不能直接作为稳定的 FFI 类型编号。Matx 为每个运行时对象分配一个整数类型索引，并记录它的父类型：

```cpp
class StrNode : public object_t {
public:
    static constexpr int32_t INDEX = TypeIndex::RuntimeStr;
    static constexpr std::string_view NAME = "RuntimeStr";
    DEFINE_TYPEINDEX(StrNode, object_t);
};
```

`TypeContext` 保存类型名称、索引和父索引。`IsFrom(child, parent)` 沿父链向上查找，因此 `IsType<T>()` 不只能够判断精确类型，也能判断继承关系。内置类型使用固定索引，扩展类型可以从 `TypeIndex::Dynamic` 开始动态分配。

```text
Object
├── RuntimeStr
├── RuntimeList
├── RuntimeDict
├── RuntimeSet
├── Module
└── 动态注册类型
```

类型名称与索引的对应关系同时被 C API、函数注册和 Python 对象包装使用。它不仅服务于 C++ 内部的类型转换，也是不同语言之间识别对象的共同协议。

**Value**

对象适合字符串和容器等需要生命周期的数据，但整数和浮点数没有必要单独分配对象。Matx 使用 C 兼容的 `Value` 统一保存立即数、字符串、指针和对象地址：

```cpp
union Union {
    int64_t v_int;
    double  v_float;
    char*   v_str;
    void*   v_pointer;
    Dt      v_datatype;
};

struct Value {
    Union u{};
    int32_t p{0};
    int32_t t{0};
};
```

`t` 是类型标签，决定联合体中哪个成员有效；`p` 保存字符串长度等附加信息。这个结构没有构造函数、析构函数或模板成员，可以直接出现在 C API 的函数签名中。

基础类型使用负数标签，对象类型使用非负索引：

```text
Null / Int / Float / Str / Pointer / DataType < 0
Object 及其派生类型                         >= 0
```

这种划分让 `value.t >= TypeIndex::Object` 成为对象判断，同时避免给整数、浮点数和裸指针增加引用计数。

**Any 与 McValue**

裸 `Value` 适合 ABI，却不适合直接在 C++ 中使用。`Any` 为它增加类型查询与转换接口：

```cpp
Any value = McValue(42);

if (value.Is<int64_t>()) {
    int64_t number = value.As<int64_t>();
}
```

`Any` 本身不负责资源释放。在它之上，Matx 区分两种使用方式：

- `McView` 只观察已有值，不增加对象引用计数，也不释放字符串或对象。
- `McValue` 拥有值，负责复制、移动和销毁资源。

这一区分对函数调用很重要。输入参数只在调用期间有效，可以使用一组 `McView` 或 `Any`；返回值需要离开当前栈帧继续存在，因此必须由 `McValue` 持有。

`McValue` 的复制行为取决于类型：整数和浮点数直接复制，C 字符串分配新缓冲区，运行时对象增加引用计数。移动操作则转移底层 `Value`，并将来源设为 `Null`。

```cpp
McValue a("matx");
McValue b = a;             // 复制字符串

McValue x(Str("runtime"));
McValue y = x;             // 共享对象并增加引用计数
```

析构时，`McValue::Clean()` 根据类型标签执行对应操作：字符串使用 `delete[]`，对象调用 `DecCounter()`，立即数不需要清理。所有资源规则都集中在这一处，容器、函数和 FFI 不需要重复判断所有权。

**DataType**

`DataType` 和 `TypeIndex` 都描述类型，但用途不同。

`TypeIndex` 回答“这个运行时值是什么对象”，例如 `RuntimeList` 或 `Module`；`DataType` 描述编译器中的标量数据格式：

```cpp
struct Dt {
    uint8_t c_;   // int、uint、float 或 handle
    uint8_t b_;   // 位宽
    uint16_t a_;  // lanes
};
```

例如 `DataType::Int(32)` 表示 32 位整数，`DataType::Bool()` 表示单 lane 的 1 位无符号值，`DataType::Handle()` 表示运行时对象或指针。`Dt` 只有四字节，可以作为 `Value` 的一个联合体成员跨越 C API。

两套类型信息分别服务于不同阶段：AST 和代码生成使用 `DataType` 推导表达式类型，运行时和 FFI 使用 `TypeIndex` 判断实际对象。不要把“编译期数据类型”和“运行时对象类型”混为一套系统，是 Matx 能同时处理静态代码与动态值的基础。

**运行时边界**

对象系统解决身份、继承和生命周期，值系统解决统一传递与跨语言布局。二者在 `McValue` 处汇合：立即数直接存入联合体，对象则以指针和类型索引进入同一接口。

容器正是这套机制的直接使用者。`List` 和 `Dict` 自身是引用计数对象，内部元素则由 `McValue` 保存。下一篇将沿着这个边界说明不同容器如何共享一套值表示，并提供各自的遍历接口。
