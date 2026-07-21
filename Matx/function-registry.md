# 函数注册与类型擦除

运行时已经能用 `McValue` 表示不同类型的值，但函数仍然有各自的 C++ 签名：

```cpp
int add(int a, int b);
Str concat(Str a, Str b);
PrimExpr make_add(PrimExpr a, PrimExpr b);
```

如果调用者必须在编译期知道每个函数的准确类型，Python 前端和动态模块就无法通过同一个入口调用它们。Matx 因此将函数统一为一种签名，并用名字建立全局索引：

```cpp
using Function = std::function<McValue(Parameters)>;
```

`Parameters` 是一段不拥有元素的参数视图，`McValue` 是拥有返回值。无论原函数接收整数还是 AST 对象，进入注册表以后都表现为 `Parameters → McValue`。

## 一次完整调用

下面的函数仍然保留清晰的强类型接口：

```cpp
REGISTER_GLOBAL("math.add")
    .SetBody([](int a, int b) {
        return a + b;
    });
```

程序启动时，注册宏创建一个静态变量。它先把名字 `math.add` 放入 `FunctionRegistry`，再由 `FunctionWrapper` 将 Lambda 包装成统一的 `Function`。调用过程可以概括为：

```text
查找 "math.add"
        ↓
Function(Parameters)
        ↓
取出参数 0 和 1，并转换为 int
        ↓
调用原始 Lambda
        ↓
把结果包装成 McValue
```

调用端不再关心原函数的 C++ 类型：

```cpp
Any args[]{McValue(10), McValue(20)};
Parameters params(args, 2);

if (Function* fn = FunctionRegistry::Get("math.add")) {
    McValue result = (*fn)(params);
}
```

这正是类型擦除的作用：隐藏具体函数签名，同时在包装器内部保留必要的类型转换。

## Parameters

`Parameters` 只保存首地址和元素数量：

```cpp
class Parameters {
    Any* item_;
    size_t size_;
};
```

它不会复制参数，也不负责释放参数，因此适合用作一次函数调用期间的临时视图。调用者必须保证底层 `Any` 数组在调用结束前有效。

这种设计也允许注册直接处理动态参数列表的函数。例如 `runtime.Tuple` 不要求固定参数数量，而是遍历全部 `Parameters`，再构造一个 `Tuple`。

## 函数特征与参数展开

对于强类型 Lambda，`FunctionTraits` 从 `operator()` 中提取返回类型、参数数量和每个参数的类型。随后，`std::index_sequence` 在编译期生成参数下标：

```cpp
return f(ConvertArg<Arg0>(ps[0]),
         ConvertArg<Arg1>(ps[1]));
```

源码并没有手写 `Arg0` 和 `Arg1`，而是通过参数包展开生成。这样同一个包装器可以处理不同参数数量的函数。

`ConvertArg<T>` 是强类型世界与运行时值之间的边界。它先移除 `const` 和引用修饰，再根据目标类型执行转换：

- 整数和浮点数从 `Any` 中读取数值；
- `std::string` 从运行时字符串转换；
- `DataType` 读取编译器数据类型；
- `object_r` 的派生类通过运行时类型索引检查并恢复；
- `McValue` 保留原始的动态值。

类型不匹配或参数不足会抛出异常。当前包装器只检查参数是否少于声明数量，多余参数不会被拒绝。

## 两种注册方式

Matx 提供的两个宏用途不同：

```cpp
REGISTER_GLOBAL("runtime.Str")
    .SetBody([](std::string value) {
        return Str(value);
    });
```

`REGISTER_GLOBAL(...).SetBody(...)` 接收普通强类型函数或 Lambda，并通过 `FunctionWrapper` 完成类型擦除。AST 构造函数大多使用这种方式。

```cpp
static McValue NewTuple(Parameters args) {
    // 直接处理动态参数
}

REGISTER_FUNCTION("runtime.Tuple", NewTuple);
```

`REGISTER_FUNCTION` 则用于已经符合 `McValue(Parameters)` 统一签名的函数，不再执行强类型参数适配。两者最终都把 `Function` 存入同一个注册表。

## 全局注册表

`FunctionRegistry` 是进程内的单例，内部使用 `std::unordered_map` 保存名称和函数。它提供注册、查找、删除和枚举名称等操作，并用互斥锁保护每次表操作。重复注册同名函数会直接报错，可以避免不同模块悄悄覆盖已有实现。

函数名采用带命名空间的字符串，例如：

```text
runtime.Str
runtime.Tuple
ast.PrimAdd
rewriter.BuildFunction
```

这种命名方式把运行时构造、AST 构造和代码生成入口放在同一套发现机制中。C API 的 `GetGlobal` 也通过这张表取得函数副本，因此 Python 前端不需要链接到具体 C++ 函数，只需知道约定的注册名称。

函数注册解决的是“如何找到并调用能力”。下一篇将进入这些函数最常创建和处理的数据结构：抽象语法树。
