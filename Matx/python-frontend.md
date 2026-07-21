# Python 前端与代码生成

前面几篇从 C++ 内部解释了运行时和 AST。这些结构真正组合起来的入口是 `simple_compile`：

```python
def sum_to(n: int) -> int:
    total = 0
    for i in range(n):
        total = total + i
    return total

simple_compile(sum_to, "sum_to.so")
```

它并不是调用 CPython 执行函数，而是读取函数源码，将支持的 Python 子集转换成 Matx AST，再生成并编译 C++ 动态库。

## 编译流水线

整个过程可以概括为：

```text
Python 函数对象
    ↓ inspect.getsource
Python 源码
    ↓ ast.parse
CPython AST
    ↓ SimpleParser
Matx PrimFunc
    ↓ rewriter.BuildFunctions
C++ 源码
    ↓ g++ -shared -fPIC -O2
动态库 .so
```

这里存在两棵不同的 AST。`ast.parse` 产生的是 CPython 提供的语法树，它忠实描述 Python 语法；`SimpleParser` 产生的是 Matx AST，它只保留后端需要的节点，并为变量和表达式附加运行时类型。

## 获取源码

`simple_compile` 接收一个 Python 函数对象，通过 `inspect.getsource` 取得定义，再用 `textwrap.dedent` 去掉嵌套环境带来的公共缩进：

```python
source = textwrap.dedent(inspect.getsource(target))
tree = ast.parse(source)
```

这种方式简单，但也意味着目标必须有可读取的源码。交互式环境中临时创建的函数、动态生成的函数或某些装饰器包装后的对象，可能无法被 `inspect` 正确还原。

编译器还会扫描目标函数中的直接构造调用。如果调用名对应全局命名空间中的 Python 类，它会一并取得类源码，与函数源码合并后重新解析。当前类支持主要用于前端分析和方法展开，并不是完整的 Python 对象模型。

## 从 Python AST 到 Matx AST

`SimpleParser` 继承 `ast.NodeVisitor`，但覆盖了默认行为：遇到没有显式支持的语法节点时直接报错，而不是悄悄遍历其子节点。这保证编译器不会忽略自己无法表达的语义。

以赋值为例：

```python
total = total + i
```

前端先把右侧变成 `PrimAdd`，再查询当前作用域。如果 `total` 尚未定义，就生成 `AllocaVarStmt`；如果已经定义，则生成 `AssignStmt`。因此 Python 中同一种赋值语法，在 Matx AST 中会区分变量声明和后续赋值。

函数参数和返回值必须带有受支持的类型标注。当前映射为：

| Python 标注 | Matx 类型 |
| --- | --- |
| `int` | `int64` |
| `float` | `float64` |
| `bool` | `bool` |
| `str`、`handle`、`None` | `handle` |

`handle` 表示由运行时管理的动态值或对象。局部变量的类型由初始化表达式推断，后续赋值会检查是否兼容；目标类型为 `handle` 时可以接收不同的运行时对象。

## 作用域与符号

`ScopeContext` 为每个代码块维护符号表和类型表。查找变量时从最内层作用域向外进行：

```text
当前 if/while 作用域
        ↓ 未找到
外层函数作用域
```

Python 的名称在前端阶段被解析为同一个 `PrimVar` 对象。代码生成器随后用 Node 地址识别变量，并为它分配唯一的 C++ 名称。这比在后端再次使用字符串查找更可靠，也能避免不同作用域中的同名变量发生冲突。

## 语法降低

Python 语法与当前 C++ AST 并不总是一一对应，前端需要将复杂结构降低为已有节点。

`for i in range(...)` 是最直观的例子。Matx 没有独立的 `ForStmt`，因此前端只支持 `range` 循环，并把它转换为初始化、条件判断、`WhileStmt` 和步长更新。`range` 的参数先保存到临时变量，避免带副作用的表达式被重复求值。

布尔表达式、链式比较、条件表达式以及包含前置语句的循环条件也会被拆成临时变量、`IfStmt`、`WhileStmt` 和基础逻辑节点。这里的“降低”不是格式变化，而是用更小的节点集合保持原语义和求值顺序。

容器则有直接对应关系：

```text
[a, b]       → ListLiteral
{a, b}       → SetLiteral
{k: v}       → DictLiteral
obj[index]   → ContainerGetItem
obj.append(x) → ContainerMethodCall
```

## 当前支持边界

前端目前覆盖函数、赋值、返回、`if`、`while`、`for range`、基础算术与比较、布尔运算、容器字面量、索引和部分方法调用。它有意只实现一个 Python 子集，例如：

- `for` 只支持遍历 `range`，目标必须是单个名称；
- 不支持 `for-else`、`while-else` 和切片；
- 一次赋值只支持一个目标；
- 普通函数调用尚未作为通用语法处理，属性调用主要面向容器和受限的类方法；
- 不支持的类型标注和 AST 节点会直接抛出异常。

这些限制应被看作当前编译器的语言边界，而不是运行时自动提供的 Python 兼容能力。

## 生成并编译 C++

解析完成后，`SimpleParser` 保存入口函数及解析过程中产生的其他函数。`simple_compile` 将它们放入 `Array<PrimFunc>`，再通过函数注册表取得：

```python
to_source = matx_script_api.GetGlobal(
    "rewriter.BuildFunctions", True
)
cpp_code = to_source(Array(functions), Str("fn")).data
```

生成的源码被写到与目标动态库同名的 `.cpp` 文件，随后执行类似命令：

```bash
g++ -shared -fPIC -O2 sum_to.cpp \
  -I/path/to/Matx/src \
  -L/path/to/Matx/build -lcase \
  -o sum_to.so
```

当前 `simple_compile` 直接调用系统 `g++`，并依赖配置中的源码目录和构建目录。编译成功只说明动态库已经产生；要从 Python 中查找并调用其中的函数，还需要模块加载器和跨语言调用协议，这正是下一篇的主题。
