# 从 Python 函数到动态库

假设我们希望把下面的 Python 函数编译成本地代码：

```python
def add(a: int, b: int) -> int:
    return a + b
```

对 Python 来说，它只是一个可以直接调用的函数；对编译器来说，它却要经过源码解析、程序表示、代码生成、本地编译和动态加载，最后才能以接近原函数的方式重新回到 Python：

```python
simple_compile(add, "add.so")

module = module_loader("./add.so")
native_add = module.get_function("add")
result = native_add(10, 20)
```

Matx 实现的正是这条链路。本书不会试图讲完一个工业编译器，而是通过这个规模较小的项目，观察一门动态语言如何穿过 C++ 运行时，最终变成可以加载和调用的本地程序。

## 先把程序变成数据

编译器不能直接操作函数对象。`simple_compile` 先通过 `inspect.getsource` 取得源码，再交给 Python 的 `ast.parse`：

```text
def add(a, b):
    return a + b
```

Python AST 仍然属于 Python 自身。Matx 的前端会继续把它转换成自己的抽象语法树：

```text
PrimFunc "add"
├── 参数: PrimVar "a", PrimVar "b"
├── 返回类型: PrimType(int64)
└── ReturnStmt
    └── PrimAdd(a, b)
```

从这一刻开始，函数不再是一段文本，而是一组可以检查、遍历和转换的 C++ 对象。

## 为什么先讲运行时

AST 中的 `PrimFunc`、`PrimVar` 和 `PrimAdd` 都有不同的字段与类型，但 Python 前端需要通过统一接口持有它们。节点还会被多个父节点共享，不能依靠裸指针随意管理生命周期。

因此 Matx 首先需要一套对象体系：Node 类保存实际数据，引用类提供类型化接口，侵入式引用计数决定何时释放对象。AST、函数和模块都建立在这套机制上。

但不是所有数据都适合变成堆对象。调用 `native_add(10, 20)` 时，两个整数更适合直接保存在固定布局的值中。Matx 因此还有一套值体系，用类型标签和联合体统一传递整数、浮点数、字符串指针和对象指针。

对象体系解决复杂数据的身份与生命周期，值体系解决不同数据的统一传递。两者在运行时接口处相互配合。

## 容器把节点组织成树

一个函数不只有单个节点。它有参数列表、默认参数和多条语句，模块还要保存多个函数。以 `add` 为例，两个 `PrimVar` 被放入 `Array<PrimVar>`，函数体中的语句被放入 `SeqStmt`。

Matx 同时需要表示 Python 的动态容器，因此实现了两组容器：`Array<T>`、`Map<K, V>` 为 C++ 内部结构保留类型约束；`List`、`Dict`、`Set` 和 `Tuple` 则用 `McValue` 保存运行时动态值。迭代器让打印器和代码生成器能够遍历这些内容。

## Python 如何创建 C++ AST

Python 前端中的 `PrimVar(...)` 并不是纯 Python 数据类。它最终需要调用 C++ 构造函数，但 Python 不知道这些函数的链接符号和具体签名。

Matx 将构造函数注册为带名字的全局函数：

```text
ast.PrimVar
ast.PrimAdd
ast.ReturnStmt
ast.PrimFunc
```

函数包装器把不同的 C++ 签名统一成 `McValue(Parameters)`。Python 只需按名称取得函数，传入运行时值，就能获得对应的 AST 对象。函数注册表由此成为 Python 前端与 C++ 编译器核心之间的入口。

## 同一棵树可以有不同用途

得到 `PrimFunc` 后，不同阶段会对它执行不同操作。Printer 把节点转换成便于观察的文本，Rewriter 则生成可以交给 C++ 编译器的源码。

它们都通过 Visitor 遍历 AST。Visitor 根据节点的运行时类型索引，将 `PrimAddNode` 分派给加法处理函数，将 `ReturnStmtNode` 分派给返回语句处理函数。节点只负责保存程序结构，具体操作则放在独立的访问者中。

对 `add` 函数，Rewriter 最终会生成类似下面的主体：

```cpp
int64_t add(int64_t a, int64_t b) {
    return (a + b);
}
```

实际输出还包括参数检查、返回值转换和模块注册信息。`simple_compile` 将源码写入 `.cpp`，再调用 `g++` 生成 `add.so`。

## 动态库如何回到 Python

生成的动态库不能直接向 Python 暴露 C++ 对象。它导出一组遵循 C ABI 的函数指针，以及函数名和函数指针之间的对应关系：

```text
add.so
├── add                  生成的 C++ 函数
├── add__c_api           C 调用包装
├── __mc_func_array__    函数指针数组
└── __mc_func_registry__ 函数注册信息
```

`module_loader` 使用 `dlopen` 打开动态库，再通过 `dlsym` 读取注册信息。`module.get_function("add")` 找到对应函数指针，并将它重新包装成 Matx 的统一 `Function`。

当 Python 调用 `native_add(10, 20)` 时，FFI 把 Python 参数转换成 `Value[]`，生成模块读取两个整数并执行 C++ 函数，返回值再沿相反方向变成 Python 整数：

```text
Python int
   ↓
Value[]
   ↓
C API 包装函数
   ↓
生成的 C++ 函数
   ↓
Value
   ↓
Python int
```

至此，一个 Python 函数完成了从源码到本地执行，再回到 Python 调用界面的闭环。

## 本书的阅读顺序

后面的七篇文章会沿着这条链路逐层展开：

1. **运行时**：对象和值为什么要采用两套相互配合的结构。
2. **容器与迭代器**：AST 和动态数据如何被组织和遍历。
3. **函数注册与类型擦除**：不同 C++ 函数如何获得统一调用接口。
4. **抽象语法树**：表达式、语句、函数和模块如何表示程序。
5. **Visitor、Printer 与 Rewriter**：同一棵 AST 如何被分派、观察并转换成 C++。
6. **Python 前端与代码生成**：Python 子集如何降低成 Matx AST 并完成本地编译。
7. **FFI 与动态模块**：生成的 `.so` 如何被发现、加载和跨语言调用。

Matx 是一个实验性的编译器与运行时项目，不以实现完整 Python 为目标。本书关注的是当前已经连通的普通函数编译主线；尚不完整的解释器、类、闭包和模块导入能力只会在相关位置说明边界，不把设计目标当成现有实现。
