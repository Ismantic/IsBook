# Function 函数调用

## 引言

编译器中的许多能力最终都表现为函数调用：Python 前端需要构造 AST 节点，编译流程需要调用代码生成入口，动态模块也需要向外提供已经编译好的函数。然而，这些函数原本具有不同的 C++ 签名：

```cpp
Str make_str(std::string value);
PrimExpr make_add(PrimExpr a, PrimExpr b);
McValue node_get_attr(Parameters args);
```

如果调用者必须在编译期知道每个函数的参数和返回类型，Python 前端就只能为每个 C++ 接口编写一套绑定，动态加载的模块也无法通过统一方式暴露能力。Matx 需要在保留 C++ 强类型接口的同时，为运行时建立一个与具体签名无关的调用入口。

这个入口由函数注册表提供。函数先被转换成统一的 `Function`，再以字符串名称保存；调用者只需准备一组运行时参数，就能查找并执行目标函数：

```text
函数名称 + 运行时参数
          ↓
     FunctionRegistry
          ↓
  参数转换 → C++ 函数 → 返回值包装
          ↓
        McValue
```

这样，调用者与函数实现不必直接依赖对方。Python 前端、编译器组件和动态模块只需共同遵守名称与运行时值的约定。

## 实现

Matx 将所有注册函数统一为一种签名：

```cpp
using Function = std::function<McValue(Parameters)>;
```

`Parameters` 表示输入参数，`McValue` 表示返回值。无论原函数接收整数、字符串还是 AST 对象，进入注册表以后都表现为 `Parameters → McValue`。

**Parameters**

一次函数调用可能包含不同数量、不同类型的参数。`Parameters` 不复制这些值，只保存参数数组的首地址和长度：

```cpp
class Parameters {
    Any* item_;
    size_t size_;
};
```

其中每个元素都由 `Any` 表示，因此同一组参数可以同时包含整数、字符串和对象引用。`Parameters` 只是调用期间的临时视图，不拥有底层数组；调用者必须保证参数在函数返回前有效。这种设计避免了跨边界调用时不必要的复制，也允许函数直接遍历数量不固定的参数。

**函数适配**

开发者仍然可以使用清晰的强类型接口注册函数：

```cpp
REGISTER_GLOBAL("runtime.Str")
    .SetBody([](std::string value) {
        return Str(value);
    });
```

`FunctionWrapper` 负责将这个 Lambda 适配成统一的 `Function`。它首先通过 `FunctionTraits` 取得参数数量和类型，再使用 `std::index_sequence` 展开参数下标。上面的函数在运行时相当于执行：

```cpp
return McValue(
    func(ConvertArg<std::string>(params[0]))
);
```

`ConvertArg<T>` 位于动态值与强类型 C++ 之间。它按照目标类型检查并恢复参数：

- 整数和浮点数从 `Any` 中读取数值；
- `std::string` 从运行时字符串恢复；
- `DataType` 读取编译器中的数据类型；
- `object_r` 的派生类型先检查对象，再恢复相应引用；
- `McValue` 保留原始动态值。

如果参数不足或类型不匹配，转换过程会抛出异常，错误不会继续进入实际函数。目前包装器只要求实参数量不少于形参数量，多出的参数不会被拒绝。

**两种注册方式**

`REGISTER_GLOBAL` 用于普通函数或 Lambda：

```cpp
REGISTER_GLOBAL("ast.PrimAdd")
    .SetBody([](PrimExpr a, PrimExpr b) {
        return PrimAdd(a, b);
    });
```

它保留强类型函数的写法，并自动完成参数转换。对于本身已经接受动态参数的函数，可以直接使用 `REGISTER_FUNCTION`：

```cpp
static McValue NewTuple(Parameters args) {
    // 遍历全部参数并构造 Tuple
}

REGISTER_FUNCTION("runtime.Tuple", NewTuple);
```

后者不再经过强类型参数适配，适合可变参数函数以及需要自行解释参数的底层入口。两种方式最终都会得到相同的 `Function`。

**函数注册表**

`FunctionRegistry` 是进程内的全局注册表，使用 `std::unordered_map` 保存名称与函数，并用互斥锁保护注册、查找、删除和枚举操作。重复注册同名函数会直接报错，避免后加载的模块悄悄覆盖已有实现。

函数名称通常带有命名空间：

```text
runtime.Str
runtime.Tuple
ast.PrimAdd
rewriter.BuildFunction
```

名称不仅用于分类，也构成不同组件之间的接口约定。运行时构造函数以 `runtime` 开头，AST 构造函数以 `ast` 开头，代码生成入口则位于 `rewriter` 下。C API 的 `GetGlobal` 同样从这张表中取得函数，因此 Python 前端不需要直接链接每个具体的 C++ 实现。

**静态注册**

注册宏会定义一个静态变量。程序或动态库被加载时，这个变量先于正常调用完成初始化，将名称和函数写入注册表：

```cpp
#define REGISTER_GLOBAL(Name)                   \
    static auto& unique_name =                  \
        FunctionRegistry::Register(Name)
```

因此，新增一个 AST 节点构造函数时，不需要再修改集中式的初始化列表。实现文件只要包含一条注册语句，链接进程序以后，相应能力就能通过名称被发现。这种分散声明、集中查找的方式降低了模块之间的依赖，但也要求包含注册代码的目标文件确实被链接或加载，否则对应名称不会出现在注册表中。

注册表中保存的不是函数地址本身，而是 `std::function`。它既能容纳普通函数，也能保存带捕获状态的 Lambda。代价是调用时多了一层间接跳转和动态参数转换；对编译器控制流程和跨语言调用而言，这部分开销通常小于它带来的接口统一。

**C API 边界**

Python 不能直接操作 `std::function`、`McValue` 或 C++ 异常，因此 Matx 又在注册表外提供了一层稳定的 C API。`GetGlobal` 根据名称找到 `Function`，复制一份并以不透明的 `FunctionHandle` 返回。Python 只保存这个句柄，不需要了解它在 C++ 中的实际类型；句柄不再使用时由 `FuncFree` 释放。

真正调用时，`FuncCall_PYTHON_C_API` 接收一组 C 结构体 `Value`：

```text
Python 对象
    ↓  参数打包
Value[]
    ↓  构造只读视图
McView[] → Parameters
    ↓
Function
    ↓  返回值转换
Value
    ↓
Python 对象
```

C API 先为每个 `Value` 建立 `McView`，再用它们构造 `Parameters`。这种视图不会取得输入值的所有权，调用结束后即可丢弃。函数返回的 `McValue` 则会被转换回 `Value`；对象的引用计数和字符串缓冲区由相应的 C API 释放函数继续管理。

C++ 中的类型错误也不能越过 C ABI 直接传播。C API 会捕获运行时异常，把错误文本保存在线程局部区域，并用非零状态码通知 Python。Python 扩展再读取错误信息并抛出 Python 异常。这样，参数转换失败仍能保留清晰的错误边界，而不会让 C++ 异常穿过不兼容的调用栈。

## 示例

以构造加法表达式为例，Python Frontend 的 `add()` 实际查找 `ast._OpAdd`。它在 C++ 中接收两个 `PrimExpr`：

```cpp
REGISTER_GLOBAL("ast._OpAdd")
    .SetBody([](PrimExpr a, PrimExpr b) {
        return PrimAdd(a, b);
    });
```

这里保留了宏展开后的核心结构；源码通过 `REGISTER_MAKE_BINARY_OP` 完成同样的注册。Python 前端调用这个名字时，参数已经被转换成运行时对象。C API 将它们放入 `Parameters`，随后从注册表取得函数：

```cpp
Function* fn = FunctionRegistry::Get("ast._OpAdd");
McValue result = (*fn)(params);
```

在 Python 一侧，这个函数并不会以手写绑定的形式出现。前端通过 `GetGlobal` 取得一个通用的可调用对象，调用时再把两个表达式对象交给相同的参数打包逻辑。包装器依次将两个 `Any` 恢复为 `PrimExpr`，调用 Lambda 构造 `PrimAddNode`，再把结果包装成 `McValue` 返回。整个过程可以表示为：

```text
Python: add(lhs, rhs)
              ↓
       GetGlobal("ast._OpAdd")
              ↓
       Parameters(lhs, rhs)
              ↓
 ConvertArg<PrimExpr> × 2
              ↓
       C++: PrimAdd(a, b)
              ↓
      McValue(PrimAddNode)
```

这个例子中存在两次类型转换。第一次发生在 Python 与 C API 之间，将 Python 表达式对象转换为带类型索引的 `Value`；第二次发生在 `FunctionWrapper` 中，将动态的 `Any` 恢复为注册函数声明的 `PrimExpr`。返回路径按相反顺序执行。Runtime 负责保存值的类型与生命周期，Function 只负责按照目标签名检查和传递它们。

注册表也让调用方向得以反转。Python 前端不必由 C++ 主动调用，而是在加载扩展后自行查找 `ast._OpAdd`、`runtime.Tuple` 或 `rewriter.BuildFunctions`。后续增加新的构造函数时，只要遵循相同的命名和参数约定，Python 侧就能继续复用同一个调用器。

同一条调用链既适用于整数和字符串，也适用于容器与 AST 对象。函数注册表解决了“如何找到并调用一种能力”，而参数转换将 Runtime 的动态值重新带回 C++ 的强类型世界。下一篇将继续讨论这些函数最常构造和处理的对象：抽象语法树。
