# Visitor、Printer 与 Rewriter

AST 把程序变成了一组节点，但节点本身不会打印、分析或生成代码。以表达式 `a + b` 为例，同一个 `PrimAddNode` 可能被用于：

- 打印成便于调试的 `(a + b)`；
- 生成可以编译的 C++ 表达式；
- 收集其中引用的变量；
- 在优化阶段替换某个子表达式。

如果把这些操作都写进 Node 类，新增一种操作就要修改所有节点。Matx 采用 Visitor，把“数据结构”和“对数据执行的操作”分开。

## 从类型索引到处理函数

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

## 递归遍历并非自动发生

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

没有注册处理函数的节点会进入默认分支。当前实现打印错误信息并返回默认值，因此增加 AST 节点时，还要同步检查相关 Visitor 是否需要支持它。

## AttrVisitor

另一类遍历针对节点的字段，而不是节点类型。Node 类通过 `VisitAttrs` 暴露具名属性：

```cpp
void PrimVarNode::VisitAttrs(AttrVisitor* visitor) {
    visitor->Visit("var_name", &var_name);
    visitor->Visit("datatype", &datatype);
}
```

`NodeAttrNameCollector` 忽略字段内容，只收集名称；`NodeAttrGetter` 则根据名称读取值。它们通过全局函数 `runtime.NodeGetAttrNames` 和 `runtime.NodeGetAttr` 暴露给前端，使 Python 可以检查 C++ AST 对象，而不必为每个字段编写一套 C API。

目前只有部分节点实现了 `VisitAttrs`，所以它更像一套按需开放的反射接口，并不是对全部 AST 字段的完整描述。

## Printer：生成可读表示

`AstPrinter` 同时继承表达式、语句和类型 Visitor。它把节点转换成 `Doc`，再由 `Doc::str()` 生成字符串。

`Doc` 不只保存普通文本，还保存换行和缩进等结构化原子。这样打印函数和代码块时，可以先组合文档，再统一处理排版：

```cpp
Doc doc;
doc << "return " << Print(value) << ";";
```

Printer 的主要用途是调试和观察 AST。注册函数 `ast.AsText` 让 Python 前端也能取得这种文本表示。它并不承担最终编译输出，而且当前对某些调用形式的处理仍有限，例如 `PrimCall` 的打印路径主要面向 `Op`。

## Rewriter：生成 C++

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

## SourceRewriter：补齐模块边界

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

类生成目前比普通函数生成更受限制：成员依赖 `ClassMembers` 属性，方法依赖 `MethodName` 和 `MethodType`，缺少属性时只进行有限推断或输出占位实现。因此当前最完整的主线仍是普通 `PrimFunc` 的代码生成。

至此，AST 已经可以从结构化表示变成 C++ 源码。下一篇将回到流程起点，介绍 Python 前端如何读取 Python AST、创建 Matx 节点并启动代码生成。
