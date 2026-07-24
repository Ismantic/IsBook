# Visitor 遍历与重写

## 引言

AST 把程序变成了一组节点，但节点本身不会打印、分析或生成代码。以表达式 `a + b` 为例，同一个 `PrimAddNode` 可能被用于：

- 打印成便于调试的 `(a + b)`；
- 生成可以编译的 C++ 表达式；
- 收集其中引用的变量；
- 在优化阶段替换某个子表达式。

如果把这些操作都写进 Node 类，新增一种操作就要修改所有节点。Matx 采用 Visitor，把“数据结构”和“对数据执行的操作”分开。

这里存在两个不同的问题。第一个是：当手中只有 `PrimExpr` 或 `Stmt` 这样的基类引用时，怎样找到具体节点对应的处理函数？第二个是：找到当前节点以后，怎样继续处理它的子节点？Visitor 负责类型分派，具体的 Printer 或 Rewriter 则决定递归顺序和处理结果。

```text
基类引用
   ↓ Visitor 按类型分派
具体 Node
   ↓ 操作决定是否递归
子表达式与子语句
   ↓
文本、分析结果或 C++ 源码
```

这种分离让 AST 的节点定义保持稳定。增加一种新的处理任务时，可以实现新的 Visitor，而不必把代码生成、调试打印和分析逻辑同时塞进每个 Node。

## 实现

**类型分派**

`NodeVisitor` 是最底层的分派表。它以节点的运行时类型索引为下标，保存对应的函数指针：

```text
PrimAddNode::Index() ──→ VisitExpr_(PrimAddNode*)
PrimVarNode::Index() ──→ VisitExpr_(PrimVarNode*)
ReturnStmtNode::Index() ──→ VisitStmt_(ReturnStmtNode*)
```

访问一个 `object_r` 时，`NodeVisitor` 读取 `Index()` 并直接查表。调用者只需要持有基类引用，分派结果仍然是具体的 Node 类。

在此之上，Matx 按节点家族提供三种访问者：

```cpp
PrimExprVisitor<R(const PrimExpr&, Args...)>
StmtVisitor<R(const Stmt&, Args...)>
TypeVisitor<R(const Type&, Args...)>
```

模板参数决定返回值和附加参数。例如 Printer 返回 `Doc`，Rewriter 不返回值，而是额外接收一个输出流。各访问者只需要覆盖自己关心的 `VisitExpr_`、`VisitStmt_` 或 `VisitType_`。

这套机制结合了两层分派：类型索引表先找到节点对应的入口，虚函数再调用当前 Visitor 子类的实现。因此 AST 节点不需要为 Printer、Rewriter 等每种用途分别增加虚函数。

**递归遍历**

Visitor 只负责“当前节点该交给谁”，不会自动访问子节点。递归逻辑由具体操作决定。打印加法表达式时，需要显式访问左右操作数：

```cpp
Doc VisitExpr_(const PrimAddNode* op) override {
    Doc doc;
    doc << "(" << Print(op->a)
        << " + " << Print(op->b) << ")";
    return doc;
}
```

这看似多写了一些代码，却允许不同任务选择不同遍历策略。例如打印器访问两个分支，常量分析可以在获得确定结果后停止，变量收集器则可以忽略类型字段。

没有注册处理函数的节点会进入默认分支。因此，增加一种 AST 节点时，需要同时为实际使用它的 Printer、Rewriter 或分析器补充处理函数。节点类型决定“它是什么”，各个 Visitor 决定“在当前任务中怎样处理它”。

**AttrVisitor**

另一类遍历针对节点的字段，而不是节点类型。Node 类通过 `VisitAttrs` 暴露具名属性：

```cpp
void PrimVarNode::VisitAttrs(AttrVisitor* visitor) {
    visitor->Visit("var_name", &var_name);
    visitor->Visit("datatype", &datatype);
}
```

`NodeAttrNameCollector` 忽略字段内容，只收集名称；`NodeAttrGetter` 则根据名称读取值。它们通过全局函数 `runtime.NodeGetAttrNames` 和 `runtime.NodeGetAttr` 暴露给前端，使 Python 可以检查 C++ AST 对象，而不必为每个字段编写一套 C API。

节点通过实现 `VisitAttrs` 明确选择要暴露的字段，因此它是一套按需开放的轻量反射接口。`NodeVisitor` 根据节点类型选择行为，`AttrVisitor` 则根据字段名称读取内容，两者解决的问题不同。

**Printer**

`AstPrinter` 同时继承表达式、语句和类型 Visitor。它把节点转换成 `Doc`，再由 `Doc::str()` 生成字符串。

`Doc` 不只保存普通文本，还保存换行和缩进等结构化原子。这样打印函数和代码块时，可以先组合文档，再统一处理排版：

```cpp
Doc doc;
doc << "return " << Print(value) << ";";
```

直接向字符串追加内容很难统一处理嵌套代码块：子节点需要知道当前缩进，父节点又需要决定换行位置。`Doc` 将“要输出什么”和“怎样排版”分开，复合节点可以先组合子文档，最后再统一生成文本。

Printer 的主要用途是观察 AST。注册函数 `ast.AsText` 让 Python 前端也能取得这种表示。它输出接近源码的可读形式，帮助确认前端生成了哪些节点，但不承担最终模块的编译输出。

**Rewriter**

当前项目中的 `Rewriter` 名字容易让人误以为它会返回一棵修改后的 AST。实际上，它遍历 AST 并将等价的 C++ 写入 `std::ostream`，职责更接近代码生成器。

例如，变量声明节点：

```text
AllocaVarStmt(c, int64, PrimAdd(a, b))
```

会被输出为类似下面的代码：

```cpp
int64_t c = (a + b);
```

Rewriter 还维护生成过程所需的状态：

- `var_dict_` 将 `PrimVarNode*` 映射为唯一的 C++ 变量名；
- 作用域栈和缩进计数控制代码块；
- `GetTypeInfo` 将 `DataType` 映射到 C++ 类型及运行时标签；
- 容器节点被转换为 `List`、`Dict`、`Set` 和 `McValue` 操作。

容器方法还需要语义映射。例如 Python 的 `list.append` 保持为 `append`，`set.add` 生成 `insert`，`set.discard` 生成 `erase`。这说明代码生成不是简单拼接源码，而是在两种语言的运行时接口之间做翻译。

Rewriter 还必须区分表达式和语句的输出环境。表达式写入当前输出流，不主动结束一行；语句负责缩进、分号和换行；`IfStmt`、`WhileStmt` 与函数节点则建立新的代码块。AST 中的嵌套关系由此重新变成 C++ 的括号和执行顺序。

**SourceRewriter**

基础 `Rewriter` 输出函数或类的 C++ 定义，`SourceRewriter` 在其外部补充可独立编译的模块结构：

```text
C++ 头文件与运行时上下文
        ↓
函数和类定义
        ↓
C API 参数检查与包装函数
        ↓
函数名表、函数指针表和模块初始化入口
```

`rewriter.BuildFunction`、`BuildFunctions` 和 `BuildClass` 是注册给前端的入口。生成函数时，C API 包装层检查参数数量和基础类型，再把 `Value` 转成 C++ 参数；返回结果则写回 `Value`。动态模块因此不需要暴露 C++ 名字改编后的符号，只需导出约定的 C 数据结构。

对于类，SourceRewriter 还会读取 `ClassMembers`、`MethodName` 和 `MethodType` 等属性，把成员定义与方法组织到同一个 C++ 类和模块接口中。函数 AST 描述函数内部的计算，而这些属性补充类和模块级别的生成信息。

## 示例

以前一篇的语句 `c = a + b` 为例，AST 中已经包含：

```text
AllocaVarStmt
├── var: PrimVar("c", int64)
└── init_value: PrimAdd(a, b)
```

Rewriter 从语句节点开始。`StmtVisitor` 根据类型索引把它分派到 `VisitStmt_(AllocaVarStmtNode*)`。该处理函数先根据 `DataType` 输出 `int64_t`，再为变量 `c` 分配当前函数内唯一的 C++ 名称，随后递归输出初始值。

初始值是一个 `PrimExpr`，因此进入 `PrimExprVisitor`：

```text
PrimAdd
├── PrimVar("a")
└── PrimVar("b")
```

`PrimAddNode` 的处理函数先写入左括号，再分别访问左右操作数。两个 `PrimVarNode` 通过 `var_dict_` 找到参数在 C++ 中的名称。递归返回后补上运算符和右括号，最终得到：

```cpp
int64_t c = (a + b);
```

完整的分派过程是：

```text
Stmt
 ↓ StmtVisitor
AllocaVarStmtNode
 ├── 输出类型和变量名
 └── PrimExpr
      ↓ PrimExprVisitor
    PrimAddNode
      ├── PrimVarNode → a
      └── PrimVarNode → b
```

Printer 遍历同一组节点时不会生成类型声明，而是得到便于阅读的表达式；其他分析器也可以复用分派结构，选择收集变量或检查节点。Visitor 让一棵 AST 支持多种解释方式，SourceRewriter 则进一步把函数输出包装成可编译、可加载的 C++ 模块。下一篇将回到流程起点，介绍 Python 前端如何读取 Python AST、创建 Matx 节点并启动代码生成。
