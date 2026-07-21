# 抽象语法树

编译器不能直接在 Python 源码字符串上完成类型检查和代码生成。它需要先把程序转换为结构化数据。例如：

```python
def add(a: int, b: int) -> int:
    c = a + b
    return c
```

在 Matx 中，这段代码可以表示成下面的节点关系：

```text
PrimFunc "add"
├── 参数: PrimVar "a", PrimVar "b"
├── 返回类型: PrimType(int64)
└── 函数体: SeqStmt
    ├── AllocaVarStmt "c"
    │   └── PrimAdd(a, b)
    └── ReturnStmt(c)
```

树中的节点不再依赖源码的空格和换行，而是直接表达“函数”“变量”“加法”和“返回”等语义。后续阶段可以遍历、检查或改写这棵树，而不必重复解析源代码。

## AST 也是运行时对象

所有 AST Node 类最终都继承 `object_t`，因此自动获得引用计数和运行时类型索引。对外使用的引用类继承 `object_r`：

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

Node 类保存字段，引用类负责生命周期和类型化访问。`DEFINE_NODE_CLASS` 为引用类生成构造、复制、移动和 `operator->` 等重复接口。于是 AST 可以像普通 C++ 值一样传递，同时共享底层节点。

这种设计也把前几篇串联起来：AST 节点由对象体系管理，节点数组使用 `Array<T>`，构造函数通过全局函数表暴露给 Python。

## 表达式

表达式以 `BaseExprNode` 为公共基类，并进一步分为 `PrimExprNode` 和 `AstExprNode`。`PrimExprNode` 带有 `DataType`，表示能够参与运算或产生运行时值的表达式；`AstExprNode` 用于函数、运算符等更高层结构。

当前主要表达式包括：

| 类别 | 节点 | 含义 |
| --- | --- | --- |
| 字面量 | `IntImm`、`FloatImm`、`Bool`、`NullImm`、`StrImm` | 源码中的常量 |
| 名称 | `PrimVar`、`GlobalVar` | 局部变量和全局符号 |
| 运算 | `PrimAdd`、`PrimMul`、`PrimEq`、`PrimNot` 等 | 算术、比较和逻辑运算 |
| 调用 | `PrimCall` | 调用运算符或全局函数 |
| 容器 | `ListLiteral`、`DictLiteral`、`SetLiteral` | 容器字面量 |
| 容器操作 | `ContainerGetItem`、`ContainerSetItem`、`ContainerMethodCall` | 索引、赋值和方法调用 |
| 对象访问 | `ClassGetItem` | 读取对象成员 |

二元运算节点共享 `PrimBinaryOpNode<T>`：

```cpp
template <typename T>
class PrimBinaryOpNode : public PrimExprNode {
public:
    PrimExpr a;
    PrimExpr b;
};
```

`PrimAdd(a, b)` 的结果类型取自左操作数；比较节点则将结果设为布尔类型。当前构造器只记录这些类型，并不完成完整的操作数类型兼容检查。

`PrimCall` 将被调用对象和参数分开保存：

```cpp
class PrimCallNode : public PrimExprNode {
public:
    BaseExpr op;
    Array<PrimExpr> gs;
};
```

`op` 可以是表示内置操作的 `Op`，也可以是表示用户函数的 `GlobalVar`。这样不同来源的调用可以共享一种树结构。

## 语句

表达式产生值，语句描述控制和执行顺序。Matx 当前定义了变量声明、赋值、返回、表达式求值、顺序块、条件、循环和类语句：

```text
Stmt
├── ExprStmt / Evaluate
├── AllocaVarStmt / AssignStmt
├── ReturnStmt
├── SeqStmt
├── IfStmt / WhileStmt
└── ClassStmt
```

`SeqStmt` 用 `Array<Stmt>` 保存一个代码块。`IfStmt` 保存条件、真分支和假分支，`WhileStmt` 保存条件和循环体。这种嵌套关系自然形成树，而不需要额外维护控制结构之间的位置。

变量声明被单独表示为 `AllocaVarStmt`：

```cpp
class AllocaVarStmtNode : public StmtNode {
public:
    PrimVar var;
    BaseExpr init_value;
};
```

它同时记录变量的名称、类型和初始值，便于后端生成明确的 C++ 局部变量声明。

## 类型

`DataType` 描述可直接映射到底层表示的标量类型，例如不同位宽的整数、浮点数和句柄。AST 还提供一层对象化的 `Type`：

```text
Type
├── PrimType       标量运行时类型
├── ClassType      类类型
├── TypeVar        类型参数
└── GlobalTypeVar  全局类型名称
```

`PrimExpr` 直接保存紧凑的 `DataType`，函数返回类型则使用 `Type`，以便容纳类和类型变量。`GetRuntimeDataType` 可以从 `PrimType` 取回 `DataType`；其他类型目前统一退化为句柄。

## 函数与模块

`BaseFunc` 保存函数名和属性，具体有两种节点：`PrimFunc` 的参数是 `PrimVar`，适合已经明确运行时类型的函数；`AstFunc` 还能保存普通 `BaseExpr` 参数和类型参数。当前 Python 前端实际生成的是 `PrimFunc`。

一个 `PrimFunc` 包含参数 `gs`、默认参数 `fs`、函数体 `body` 和返回类型 `rt`。多个函数由 `IRModule` 组织：

```cpp
Map<GlobalVar, BaseFunc> func_;
Map<GlobalTypeVar, ClassType> class_;
```

`GlobalVar` 和 `GlobalTypeVar` 是模块内的符号，Map 中的值才是对应的函数或类定义。这样调用节点只需引用符号，不必复制整棵函数树。

AST 解决了程序的表示问题，但不同工具还需要按节点类型执行不同操作：打印器要输出文本，代码生成器要输出 C++，分析器要读取字段。下一篇将介绍 Matx 如何通过 Visitor、Printer 和 Rewriter 遍历同一棵树。
