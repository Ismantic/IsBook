# Matx 编译流程

## 引言

Python 适合快速编写和组合程序，但当一段逻辑需要进入 C++ 运行时、生成可独立加载的动态库，或者与其他本地模块共同执行时，仅保留 Python 函数对象还不够。编译器需要取得函数源码，理解其中的参数、类型和表达式，再为它生成具有明确调用边界的本地实现。

可以从一个最小的带类型函数开始。它只有一次加法，却已经包含参数声明、类型标注、返回语句和函数调用这些基本要素，足以展示一条完整的编译链路：

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

## 原理

编译器的本质不是替换关键字，而是在两种程序表示之间保持语义。Python 中的加法、变量、循环和容器都有自己的执行规则；生成 C++ 时，可以改变它们的表示方式，却不能改变参数的求值顺序、变量指向的值以及函数返回的结果。

Matx 是一个提前编译（Ahead-of-Time，AOT）的源码到源码编译器。它不直接生成机器指令，而是先把 Python 子集转换成 C++，再借助现有 C++ 编译器生成动态库：

```text
Python 源码
    ↓ 解析与语义检查
Matx AST
    ↓ 代码生成
C++ 源码
    ↓ g++
本地动态库
```

选择 C++ 作为目标语言，可以复用成熟编译器完成优化、机器码生成和平台适配。但 C++ 只能解决静态代码的编译，无法自动提供 Python 的动态值、容器、对象生命周期和跨语言调用。最终程序实际上由两部分共同组成：

```text
                         编译期

 Python 函数
      │
      ▼
 Python Frontend ──────────────┐
      │                        │ 构造 C++ AST 对象
      ▼                        ▼
   Matx AST ◄──────── Function Registry
      │
      ▼
 Visitor / Rewriter
      │
      ▼
  C++ 源码 ────────► C++ Compiler ────────► 动态库
      │                                      │
      │ 调用                                 │ 加载与查找
      ▼                                      ▼
 ┌──────────────── Matx Runtime ─────────────────┐
 │ Object / Value / Container / Function / FFI  │
 └──────────────────────┬────────────────────────┘
                        │
                        ▼
                   Python 调用结果

                         运行期
```

图的上半部分是编译器：Frontend 理解 Python，AST 保存程序语义，Rewriter 生成 C++。下半部分是运行时：它既让 Python Frontend 能够创建 C++ AST 对象，也为生成代码提供动态值、容器和函数接口，最后再通过 FFI 把动态库接回 Python。

简单的整数加法可以直接生成 C++ 运算符；Python 列表、字典和动态对象则必须调用 Matx Runtime。代码生成器需要为每种语义选择正确的落点：能够静态表达的部分交给 C++，需要动态行为的部分保留为 Runtime 操作。

**编译期与运行期**

Matx 的流程还可以分成两个时间阶段。编译期读取 Python 源码、构造 AST、确定类型并输出 C++；运行期加载生成的动态库，接收 Python 参数并执行本地函数：

```text
编译期
Python Function → Python AST → Matx AST → C++ → .so

运行期
Python Value → FFI → .so 中的函数 → FFI → Python Value
```

Runtime 同时服务于两个阶段。编译期，Python 前端通过 Runtime 创建并持有 C++ AST 对象；运行期，生成代码通过同一套值、容器和函数协议处理参数与结果。这种统一让编译器自身和编译产物不必各自实现一套对象模型。

**两条主线**

本书首先讨论如何把 Python 编译成 C++。这条主线关心源码解析、AST 转换、类型处理、C++ 生成、本地编译和动态加载。它回答的是：一个 Python 函数如何变成真正执行的本地代码，并继续保留可从 Python 调用的接口？

但这不只是语法翻译。为了传递 Python 的动态值，Matx 需要实现值表示；为了持有字符串、容器和 AST，Matx 需要实现对象与生命周期；为了支持列表和字典，Matx 还需要定义容器语义。函数、作用域、类型标签和跨语言对象同样不能由 C++ 编译器自动提供。

因此，第二条主线是亲手实现 Python 的关键机制：

```text
Python 的值       → Value / McValue
Python 的对象     → object_t / object_r
Python 的容器     → List / Dict / Set / Tuple
Python 的函数     → PrimFunc / Function
Python 的名称     → ScopeContext / PrimVar
Python 的执行边界 → C API / 动态模块
```

这里的“实现 Python”不是复刻 CPython，也不意味着兼容完整的 Python 语言。Matx 选择一个受限子集，重新实现支撑它所必需的语言结构和运行时，再用 C++ 代码生成代替字节码虚拟机完成执行。两条主线最终汇合为同一个问题：要把 Python 编译成 C++，究竟需要自己实现多少 Python？

## 实现

下面沿着 `add` 的编译过程，观察输入程序如何穿过上述结构。这里的重点不是再次罗列文件，而是确定每个模块位于哪一层，以及前一层的输出如何成为后一层的输入。

**Python Frontend**

`simple_compile` 接收的是一个已经存在的 Python 函数对象。函数能够被 CPython 调用，但编译器真正需要的是定义它的源码。Frontend 先使用 `inspect.getsource` 取得文本，再通过 `ast.parse` 得到 Python AST：

```text
Python Function
      ↓ inspect.getsource
Python Source
      ↓ ast.parse
Python AST
```

Python AST 解决了语法识别，却仍然包含完整 Python 的节点和动态语义。`SimpleParser` 从中选择 Matx 支持的子集，解析类型标注和名称作用域，并把较复杂的语法降低为更基础的结构。例如第一次赋值变成变量声明，`for range` 变成初始化、条件和 `WhileStmt`。Frontend 决定“哪些 Python 程序可以进入编译器”，也负责让进入后端的程序具有明确含义。

这里的 Python Frontend 是一个编译器层次，而不是名为 `frontend` 的包。当前代码主要对应 `compiler.py`、`parser.py` 和 `scope_context.py`：它们分别启动编译、转换 AST，并管理作用域与符号。

**AST**

Python AST 仍然属于 Python 自身。Frontend 会继续把它转换成 Matx 的抽象语法树：

```text
PrimFunc "add"
├── 参数: PrimVar "a", PrimVar "b"
├── 返回类型: PrimType(int64)
└── ReturnStmt
    └── PrimAdd(a, b)
```

从这一刻开始，函数不再是一段文本，而是一组可以检查、遍历和转换的 C++ 对象。

AST 是编译器各阶段之间的协议。Frontend 不需要知道每个节点最终输出成什么 C++，Rewriter 也不需要重新理解 Python 的缩进和语法；两者只需对 `PrimFunc`、`PrimAdd` 和 `ReturnStmt` 等节点的含义达成一致。这种中间表示把“理解源语言”和“生成目标语言”分开。

**Runtime**

AST 中的 `PrimFunc`、`PrimVar` 和 `PrimAdd` 都有不同的字段与类型，但 Python 前端需要通过统一接口持有它们。节点还会被多个父节点共享，不能依靠裸指针随意管理生命周期。

因此 Matx 首先需要一套对象体系：Node 类保存实际数据，引用类提供类型化接口，侵入式引用计数决定何时释放对象。AST、函数和模块都建立在这套机制上。

但不是所有数据都适合变成堆对象。调用 `native_add(10, 20)` 时，两个整数更适合直接保存在固定布局的值中。Matx 因此还有一套值体系，用类型标签和联合体统一传递整数、浮点数、字符串指针和对象指针。

对象体系解决复杂数据的身份与生命周期，值体系解决不同数据的统一传递。两者在运行时接口处相互配合。

**Container**

一个函数不只有单个节点。它有参数列表、默认参数和多条语句，模块还要保存多个函数。以 `add` 为例，两个 `PrimVar` 被放入 `Array<PrimVar>`，函数体中的语句被放入 `SeqStmt`。

Matx 同时需要表示 Python 的动态容器，因此实现了两组容器：`Array<T>`、`Map<K, V>` 为 C++ 内部结构保留类型约束；`List`、`Dict`、`Set` 和 `Tuple` 则用 `McValue` 保存运行时动态值。迭代器让打印器和代码生成器能够遍历这些内容。

**Function**

Python 前端中的 `PrimVar(...)` 并不是纯 Python 数据类。它最终需要调用 C++ 构造函数，但 Python 不知道这些函数的链接符号和具体签名。

Matx 将构造函数注册为带名字的全局函数：

```text
ast.PrimVar
ast.PrimAdd
ast.ReturnStmt
ast.PrimFunc
```

函数包装器把不同的 C++ 签名统一成 `McValue(Parameters)`。Python 只需按名称取得函数，传入运行时值，就能获得对应的 AST 对象。函数注册表由此成为 Python 前端与 C++ 编译器核心之间的入口。

这也解释了 Runtime 为什么出现在编译期：`SimpleParser` 虽然由 Python 编写，真正构造的却是 C++ 中的 `PrimVarNode`、`PrimAddNode` 和 `PrimFuncNode`。Function 与 FFI 把 Python 的解析逻辑连接到 C++ 的程序表示。

**Visitor**

得到 `PrimFunc` 后，不同阶段会对它执行不同操作。Printer 把节点转换成便于观察的文本，Rewriter 则生成可以交给 C++ 编译器的源码。

它们都通过 Visitor 遍历 AST。Visitor 根据节点的运行时类型索引，将 `PrimAddNode` 分派给加法处理函数，将 `ReturnStmtNode` 分派给返回语句处理函数。节点只负责保存程序结构，具体操作则放在独立的访问者中。

对 `add` 函数，Rewriter 最终会生成类似下面的主体：

```cpp
int64_t add(int64_t a, int64_t b) {
    return (a + b);
}
```

实际输出还包括参数检查、返回值转换和模块注册信息。`simple_compile` 将源码写入 `.cpp`，再调用 `g++` 生成 `add.so`。

这里完成了从源语言到目标语言的语义映射：

```text
PrimFunc       → C++ 函数定义
PrimVar        → 带类型的局部变量或参数
PrimAdd        → C++ 加法表达式
ReturnStmt     → C++ return 语句
List / Dict    → Matx Runtime 容器操作
```

目标代码不是 Python 源码的逐行转写，而是 AST 节点在 C++ 与 Runtime 中的对应实现。

**生成结果**

实际使用当前 Matx 编译这个函数：

```python
def add(a: int, b: int) -> int:
    return a + b
```

生成的 C++ 文件包含下面的函数主体：

```cpp
int64_t add(int64_t a, int64_t b) {
  return (a + b);
}
```

但这个函数还不能由 Python 直接调用。生成文件中还有一个 C 包装函数。省略重复的错误文本后，其关键代码为：

```cpp
int add__c_api(Value* args, int num_args,
               Value* ret_val, void* resource_handle) {
    // 检查 num_args == 2，且两个参数均为 Int

    auto result = add(args[0].u.v_int,
                      args[1].u.v_int);

    ret_val->t = TypeIndex::Int;
    ret_val->u.v_int = result;
    return 0;
}
```

真正的 `add` 只负责计算，`add__c_api` 负责检查并拆出参数，再设置返回值及其类型。生成文件最后把包装函数写入函数数组，并用名称 `"add"` 建立注册信息：

```cpp
BackendFunc __mc_func_array__[] = {
    (BackendFunc)add__c_api,
};

FuncRegistry __mc_func_registry__ = {
    "\1add\000",
    __mc_func_array__,
};
```

因此，“把 Python 编译成 C++”实际包含两类输出：

```text
Python 函数语义 ──→ add          执行真正的计算
Python 调用边界 ──→ add__c_api   转换参数与返回值
模块发现协议   ──→ registry     让加载器找到函数
```

这三部分一起进入 `add.so`。缺少第一部分就没有本地实现，缺少后两部分则虽然生成了机器代码，Python 却无法发现和调用它。

**FFI**

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

从全书结构来看，Runtime、Container 和 Function 建立编译器与生成程序共用的运行时基础；AST 与 Visitor 负责程序的内部表示和目标代码生成；Python Frontend 提供源语言入口；FFI 则把生成的本地模块重新接回 Python。后面的每一篇都会展开其中一层，但它们共同服务于同一条主线：

```text
理解 Python 程序
    ↓
用 Matx AST 保存其语义
    ↓
用 C++ 与 Runtime 实现这些语义
    ↓
把本地结果重新交给 Python
```
