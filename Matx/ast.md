# AST 抽象语法树

## 引言

Python 源码适合人阅读，却不适合编译器直接处理。空格和换行承担语法作用，同一种运算也可能写在不同上下文中；如果类型检查、代码改写和 C++ 生成都反复分析源码字符串，各阶段不仅容易重复工作，也很难共享已经得到的信息。

编译器因此先把程序转换成抽象语法树（Abstract Syntax Tree，AST）。例如：

```python
def add(a: int, b: int) -> int:
    c = a + b
    return c
```

进入 Matx 后，函数本身可以表示为下面的节点关系：

```text
PrimFunc "add"
├── 参数: PrimVar "a", PrimVar "b"
├── 返回类型: PrimType(int64)
└── 函数体: SeqStmt
    ├── AllocaVarStmt
    │   ├── 变量: PrimVar "c"
    │   └── 初始值: PrimAdd(a, b)
    └── ReturnStmt(c)
```

这棵树省略了缩进、括号等表面形式，只保留函数、变量、运算和返回等语义结构。一个节点也不再只是语法名称：`PrimVar` 保存变量类型，`PrimAdd` 保存操作数和结果类型，`PrimFunc` 保存参数、函数体与返回类型。后续阶段只需处理这些结构化对象。

Matx 的 Python 前端会先读取 Python AST，再构造自己的 AST。前者描述完整的 Python 语法，后者只保留编译器支持的子集，并补充 C++ 生成需要的类型和运行时信息。因此，Matx AST 既是源程序的内部表示，也是 Python 与 C++ 之间的语义边界。

## 实现

Matx 将 AST 分成表达式、语句、类型、函数和模块几层。它们不是相互独立的容器，而是逐层组合：

```text
IRModule
  └── BaseFunc
        └── Stmt
              └── BaseExpr

Type ──描述变量、表达式和函数返回值
```

表达式组成计算，语句安排执行顺序，函数把参数和语句组织成可调用单元，模块再为多个函数和类建立全局名称空间。

**Node 与引用**

所有 AST Node 最终都继承 `object_t`，自动获得引用计数和运行时类型索引。具体数据保存在 Node 中，对外传递的则是继承 `object_r` 的引用类：

```cpp
class PrimVarNode : public PrimExprNode {
public:
    std::string var_name;
    // datatype 继承自 PrimExprNode
};

class PrimVar : public PrimExpr {
public:
    DEFINE_NODE_CLASS(PrimVar, PrimExpr, PrimVarNode);
};
```

`PrimVarNode` 保存名称和类型，`PrimVar` 管理节点的引用并提供类型化访问。`DEFINE_NODE_CLASS` 生成构造、复制、移动和 `operator->` 等公共代码。复制一个 `PrimVar` 时通常只增加底层节点的引用计数，不会复制整棵子树。

这种表示适合 AST：同一个变量节点可以同时出现在初始化语句、加法表达式和返回语句中。各处共享的是同一个符号对象，而不是三个只有名称相同的副本。节点数组由 `Array<T>` 保存，函数属性由 `Map` 保存，构造入口则通过 Function 注册表暴露给 Python，前面介绍的 Runtime 结构都在这里汇合。

**表达式**

表达式以 `BaseExprNode` 为公共基类，并分成两条分支：

```text
BaseExpr
├── PrimExpr    带有 DataType，可产生运行时值
└── AstExpr     函数等更高层的编译结构
```

常见的 `PrimExpr` 节点包括：

| 类别 | 节点 | 含义 |
| --- | --- | --- |
| 字面量 | `IntImm`、`FloatImm`、`Bool`、`NullImm`、`StrImm` | 常量 |
| 名称 | `PrimVar`、`GlobalVar` | 局部变量和全局符号 |
| 运算 | `PrimAdd`、`PrimMul`、`PrimEq`、`PrimNot` | 算术、比较和逻辑运算 |
| 调用 | `PrimCall` | 调用内置操作或全局函数 |
| 容器 | `ListLiteral`、`DictLiteral`、`SetLiteral` | 容器字面量 |
| 访问 | `ClassGetItem`、`ContainerGetItem` | 成员与索引读取 |
| 修改 | `ContainerSetItem`、`ContainerMethodCall` | 容器写入与方法调用 |

二元运算共享相同的节点布局：

```cpp
template <typename T>
class PrimBinaryOpNode : public PrimExprNode {
public:
    PrimExpr a;
    PrimExpr b;
};
```

`PrimAdd(a, b)` 保存左右操作数，并以左操作数的 `DataType` 作为结果类型；比较节点则产生布尔类型。类型直接附着在表达式上，代码生成器不必在输出每个运算时重新查找变量声明。当前构造器主要记录类型，完整的操作数兼容性仍由前端解析和检查过程保证。

函数调用使用稍有不同的结构：

```cpp
class PrimCallNode : public PrimExprNode {
public:
    BaseExpr op;
    Array<PrimExpr> gs;
};
```

`op` 表示被调用对象，`gs` 保存实参。被调用对象可以是内置 `Op`，也可以是用户函数对应的 `GlobalVar`。这样，`a + b`、内置函数和普通函数调用虽然来源不同，最终都能通过明确的表达式节点交给后端。

**语句**

表达式描述“计算什么”，语句描述“何时计算以及结果放在哪里”。Matx 支持的语句形成另一棵继承树：

```text
Stmt
├── ExprStmt / Evaluate
├── AllocaVarStmt / AssignStmt
├── ReturnStmt
├── SeqStmt
├── IfStmt / WhileStmt
└── ClassStmt
```

`SeqStmt` 使用 `Array<Stmt>` 表示一个顺序代码块。`IfStmt` 保存条件、真分支和假分支，`WhileStmt` 保存条件与循环体。代码块嵌套在控制语句中，控制语句又可以作为外层代码块的一个元素，执行结构自然由树的层次表达。

变量的首次定义和后续赋值被区分为两种节点：

```cpp
class AllocaVarStmtNode : public StmtNode {
public:
    PrimVar var;
    BaseExpr init_value;
};

class AssignStmtNode : public StmtNode {
public:
    BaseExpr u;
    BaseExpr v;
};
```

`AllocaVarStmt` 同时建立变量和初始值，后端可以据此生成带类型的 C++ 局部变量声明；`AssignStmt` 只改变已经存在的目标。这个区别在 Python 源码中并不总是显式存在，却是生成静态 C++ 代码时必须补充的信息。

**类型**

Matx 使用两种相互配合的类型表示。`DataType` 是紧凑的值类型，记录标量类别、位宽和通道数，适合直接附着在大量表达式上。`Type` 是运行时对象，可以继续派生：

```text
Type
├── PrimType          标量类型
├── ClassType         类类型
├── TypeVar           类型参数
└── GlobalTypeVar     全局类型名称
```

局部变量和普通运算主要使用 `DataType`，函数返回值则使用 `Type`，从而为类和类型参数保留空间。`GetRuntimeDataType` 可以从 `PrimType` 取出底层 `DataType`；不能直接映射为标量的对象类型则以句柄形式进入运行时。

这两层表示分别服务于高频的底层计算和可扩展的语言类型。如果所有地方都使用对象化的 `Type`，简单整数运算也需要访问运行时对象；如果只使用 `DataType`，又无法表示类和类型变量。

**函数与模块**

函数节点继承 `BaseFunc`。`PrimFunc` 的参数是已经确定类型的 `PrimVar`，适合进入当前代码生成流程；`AstFunc` 还能保存普通 `BaseExpr` 参数和类型参数，用于表达更高层的函数结构。当前 Parser 生成的是 `PrimFunc`：

```cpp
class PrimFuncNode : public BaseFuncNode {
public:
    Array<PrimVar> gs;   // 参数
    Array<PrimExpr> fs;  // 默认参数
    Stmt body;
    Type rt;             // 返回类型
};
```

AST 还定义了 `IRModule`，可以用来组织多个函数和类：

```cpp
Map<GlobalVar, BaseFunc> func_;
Map<GlobalTypeVar, ClassType> class_;
```

`GlobalVar` 是函数在模块中的符号，`BaseFunc` 才是函数定义。调用表达式只引用 `GlobalVar`，不用把被调用函数的整棵树嵌入当前函数。模块结构因此可以同时保存定义和定义之间的引用关系。

不过，当前 Python `simple_compile` 走的是一条更直接的路径：`SimpleParser` 将函数保存在自身的字典和顺序表中，编译器取出全部 `PrimFunc`，组成 `Array` 后传给 `rewriter.BuildFunctions`。`IRModule` 已经存在于 C++ AST 中，但尚未成为这条 Python 编译路径的必经容器。区分这两点很重要：前者说明 AST 的组织能力，后者才是当前代码实际执行的流程。

## 示例

再次观察开头的 `add`。解析函数签名时，前端先为参数建立带类型的变量：

```text
a → PrimVar("a", int64)
b → PrimVar("b", int64)
```

解析 `c = a + b` 时，符号表让名称 `a` 和 `b` 指向已有的 `PrimVar`。加法生成 `PrimAdd(a, b)`，赋值左侧第一次出现，因此整条语句被表示成：

```text
AllocaVarStmt
├── var: PrimVar("c", int64)
└── init_value: PrimAdd
    ├── a: PrimVar("a", int64)
    └── b: PrimVar("b", int64)
```

前端随后把 `c` 加入符号表。解析 `return c` 时，得到的是引用同一变量的 `ReturnStmt`。两条语句进入 `SeqStmt`，再与参数和返回类型共同构成 `PrimFunc`。`SimpleParser` 以函数名记录这个结果，`simple_compile` 最后将收集到的 `PrimFunc` 交给代码生成器。

```text
源码名称
   ↓ 符号表
PrimVar / GlobalVar
   ↓ 组合表达式
PrimAdd
   ↓ 组织执行顺序
AllocaVarStmt + ReturnStmt
   ↓
PrimFunc
   ↓
Array<PrimFunc>
   ↓
rewriter.BuildFunctions
```

到了代码生成阶段，后端看到的已经不是 Python 文本，而是一组类型明确、层次稳定的对象。它可以把 `AllocaVarStmt` 输出为局部变量声明，把 `PrimAdd` 输出为加法表达式，再把 `ReturnStmt` 输出为 `return`。AST 解决了程序如何表示的问题；下一篇将继续讨论 Printer、Visitor 和 Rewriter 如何遍历并转换同一棵树。
