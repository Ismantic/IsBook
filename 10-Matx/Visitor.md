# Visitor

## 概述

访问者模式是编译器框架的核心机制，提供了一套完善的AST遍历和操作基础设施。它实现了类型安全的多态分发机制，支持对表达式、语句、类型等不同AST节点的统一访问和处理。

**关键点**
1. **类型安全分发**: 基于运行时类型系统的高性能函数分发机制
2. **多态访问**: 支持表达式、语句、类型等不同AST节点类型的统一处理
3. **模板化设计**: 高度参数化的访问者模板，支持任意返回类型和参数
4. **可扩展性**: 易于添加新的AST节点类型和访问模式
5. **实用工具**: 内置属性访问、代码打印、代码重写等常用功能


**架构**:
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   NodeVisitor   │    │ PrimExprVisitor │    │   StmtVisitor   │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│ 基础分发机制    │ -> │ 表达式访问      │    │ 语句访问        │
│ 类型索引映射    │    │ 虚函数重写      │    │ 虚函数重写      │
│ 函数指针表      │    │ 递归遍历        │    │ 递归遍历        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  │
                ┌─────────────────────────────────┐
                │          应用层                 │
                ├─────────────────────────────────┤
                │ AstPrinter (代码打印)           │
                │ Rewriter (代码重写)             │
                │ AttrVisitor (属性访问)          │
                └─────────────────────────────────┘
```

## NodeVisitor

`NodeVisitor` 是整个访问者系统的核心，实现了基于运行时类型索引的高效函数分发：

`NodeVisitor` 是整个访问者系统的核心，实现了基于运行时类型索引的高效函数分发：

```cpp
template <typename R, typename... Args>
class NodeVisitor<R(const object_r& n, Args...)> {
private:
    using FnPointer = R (*)(const object_r& n, Args...);
    std::vector<FnPointer> func_;

public:
    using result_type = R;

    // 检查节点是否有对应的处理函数
    bool InDispatch(const object_r& n) const {
        uint32_t t = n->Index();
        return t <= func_.size() && func_[t] != nullptr;
    }

    // 注册特定类型的处理函数
    template<typename Node>
    NodeVisitor& SetDispatch(FnPointer fn) {
        uint32_t t = Node::RuntimeTypeIndex();
        if (func_.size() <= t) {
            func_.resize(t+1, nullptr);
        }
        if (func_[t] != nullptr) {
            std::cerr << "Dispatch for " << Node::NAME << " error" << "\n";
            return *this;
        }
        func_[t] = fn;
        return *this;
    }

    // 执行分发
    R operator()(const object_r& n, Args... args) const {
        if (!InDispatch(n)) {
            std::cerr << "NodeVisitor " << n->Name() << "\n";
            return R();
        }
        return (*func_[n->Index()])(n, std::forward<Args>(args)...);
    }
};
```

**设计特点**:
- **O(1) 分发**: 通过类型索引直接查表，避免了虚函数的间接调用开销
- **类型安全**: 编译时确保函数签名的一致性
- **可扩展**: 支持动态注册新的类型处理函数
- **完美转发**: 支持任意参数类型和数量

**分发机制详解**:
```cpp
// 1. 获取节点的运行时类型索引
uint32_t type_index = n->Index();

// 2. 直接查表获取对应的处理函数
FnPointer handler = func_[type_index];

// 3. 调用处理函数，转发所有参数
return handler(n, std::forward<Args>(args)...);
```

## PrimExprVisitor

TODO：对特化的说明

专门用于处理原始表达式（PrimExpr）的访问者模板：

```cpp
template <typename R, typename... Args>
class PrimExprVisitor<R(const PrimExpr& n, Args...)> {
private:
    using Self = PrimExprVisitor<R(const PrimExpr& n, Args...)>;
    using Fn = NodeVisitor<R(const object_r& n, Self*, Args...)>;

public:
    virtual ~PrimExprVisitor() = default;

    R operator()(const PrimExpr& n, Args... args) {
        return VisitExpr(n, std::forward<Args>(args)...);
    }

    virtual R VisitExpr(const PrimExpr& n, Args... args) {
        static Fn vt = InitVTable();
        return vt(n, this, std::forward<Args>(args)...);
    }

    // 默认处理函数
    virtual R VisitExprDefault_(const object_t* op, Args... args) {
        std::cerr << "Do not have a default case for " << op->Name() << "\n";
        return R();
    }

    // 各种表达式节点的虚函数处理接口
    virtual R VisitExpr_(const IntImmNode* op, Args... args) { 
        return VisitExprDefault_(op, std::forward<Args>(args)...); 
    }
    virtual R VisitExpr_(const PrimVarNode* op, Args... args) {
        return VisitExprDefault_(op, std::forward<Args>(args)...);
    }
    virtual R VisitExpr_(const PrimAddNode* op, Args... args) {
        return VisitExprDefault_(op, std::forward<Args>(args)...);
    }
    virtual R VisitExpr_(const PrimMulNode* op, Args... args) {
        return VisitExprDefault_(op, std::forward<Args>(args)...);
    }
    virtual R VisitExpr_(const PrimCallNode* op, Args... args) {
        return VisitExprDefault_(op, std::forward<Args>(args)...);
    }

private:
    static Fn InitVTable() {
        Fn vt;
        vt.template SetDispatch<IntImmNode>(
            [](const object_r& n, Self* self, Args... args) {
                return self->VisitExpr_(static_cast<const IntImmNode*>(n.get()), 
                                      std::forward<Args>(args)...);
            });
        vt.template SetDispatch<PrimVarNode>(
            [](const object_r& n, Self* self, Args... args) {
                return self->VisitExpr_(static_cast<const PrimVarNode*>(n.get()),
                                      std::forward<Args>(args)...);
            });
        vt.template SetDispatch<PrimAddNode>(
            [](const object_r& n, Self* self, Args... args) {
                return self->VisitExpr_(static_cast<const PrimAddNode*>(n.get()),
                                      std::forward<Args>(args)...);
            });
        vt.template SetDispatch<PrimMulNode>(
            [](const object_r& n, Self* self, Args... args) {
                return self->VisitExpr_(static_cast<const PrimMulNode*>(n.get()),
                                      std::forward<Args>(args)...);
            });
        vt.template SetDispatch<PrimCallNode>(
            [](const object_r& n, Self* self, Args... args) {
                return self->VisitExpr_(static_cast<const PrimCallNode*>(n.get()),
                                      std::forward<Args>(args)...);
            });
        return vt;
    }
};
```

**设计特点**:
- **静态分发表**: `InitVTable()` 在首次调用时初始化，之后重复使用
- **虚函数重写**: 子类可以重写特定节点类型的处理逻辑
- **类型安全转换**: 自动进行从基类到派生类的类型转换
- **CRTP模式**: 通过Self指针实现递归调用

## StmtVisitor

专门用于处理语句（Stmt）的访问者模板，设计模式与PrimExprVisitor类似：

```cpp
template <typename R, typename... Args>
class StmtVisitor<R(const Stmt& n, Args...)> {
private:
    using Self = StmtVisitor<R(const Stmt& n, Args...)>;
    using Fn = NodeVisitor<R(const object_r& n, Self*, Args...)>;

public:
    virtual ~StmtVisitor() = default;

    R operator()(const Stmt& n, Args... args) {
        return VisitStmt(n, std::forward<Args>(args)...);
    }

    virtual R VisitStmt(const Stmt& n, Args... args) {
        static Fn vt = InitVTable();
        return vt(n, this, std::forward<Args>(args)...);
    }

    virtual R VisitStmtDefault_(const object_t* op, Args... args) {
        std::cerr << "Do not have a default case for " << op->Name() << "\n";
        return R();
    }

    // 各种语句节点的处理接口
    virtual R VisitStmt_(const AllocaVarStmtNode* op, Args... args) {
        return VisitStmtDefault_(op, std::forward<Args>(args)...);
    }
    virtual R VisitStmt_(const AssignStmtNode* op, Args... args) {
        return VisitStmtDefault_(op, std::forward<Args>(args)...);
    }
    virtual R VisitStmt_(const ReturnStmtNode* op, Args... args) {
        return VisitStmtDefault_(op, std::forward<Args>(args)...);
    }
    virtual R VisitStmt_(const SeqStmtNode* op, Args... args) {
        return VisitStmtDefault_(op, std::forward<Args>(args)...);
    }
    virtual R VisitStmt_(const EvaluateNode* op, Args... args) {
        return VisitStmtDefault_(op, std::forward<Args>(args)...);
    }

private:
    static Fn InitVTable() {
        Fn vt;
        vt.template SetDispatch<AllocaVarStmtNode>(
            [](const object_r& n, Self* self, Args... args) {
                return self->VisitStmt_(static_cast<const AllocaVarStmtNode*>(n.get()),
                                      std::forward<Args>(args)...);
            });
        vt.template SetDispatch<AssignStmtNode>(
            [](const object_r& n, Self* self, Args... args) {
                return self->VisitStmt_(static_cast<const AssignStmtNode*>(n.get()),
                                      std::forward<Args>(args)...);
            });
        vt.template SetDispatch<ReturnStmtNode>(
            [](const object_r& n, Self* self, Args... args) {
                return self->VisitStmt_(static_cast<const ReturnStmtNode*>(n.get()),
                                      std::forward<Args>(args)...);
            });
        vt.template SetDispatch<SeqStmtNode>(
            [](const object_r& n, Self* self, Args... args) {
                return self->VisitStmt_(static_cast<const SeqStmtNode*>(n.get()),
                                      std::forward<Args>(args)...);
            });
        vt.template SetDispatch<EvaluateNode>(
            [](const object_r& n, Self* self, Args... args) {
                return self->VisitStmt_(static_cast<const EvaluateNode*>(n.get()),
                                      std::forward<Args>(args)...);
            });
        return vt;
    }
};
```

## TypeVisitor

用于处理类型系统节点的访问者：

```cpp
template <typename R, typename... Args>
class TypeVisitor<R(const Type& n, Args...)> {
private:
    using Self = TypeVisitor<R(const Type& n, Args...)>;
    using Fn = NodeVisitor<R(const object_r& n, Self*, Args...)>;

public:
    using result_type = R;
    virtual ~TypeVisitor() {}

    R operator()(const Type& n, Args... args) {
        return VisitType(n, std::forward<Args>(args)...);
    }

    virtual R VisitType(const Type& n, Args... args) {
        static Fn vt = InitTable();
        return vt(n, this, std::forward<Args>(args)...);
    }

    virtual R VisitTypeDefault_(const object_t* op, Args...) {
        std::cerr << "Do not have a default case for " << op->Name() << "\n";
        return R();
    }

    virtual R VisitType_(const PrimTypeNode* op, Args... args) {
        return VisitTypeDefault_(op, std::forward<Args>(args)...);
    }

private:
    static Fn InitTable() {
        Fn vt;
        vt.template SetDispatch<PrimTypeNode>(
            [](const object_r& n, Self* self, Args... args) {     
                return self->VisitType_(static_cast<const PrimTypeNode*>(n.get()), 
                                       std::forward<Args>(args)...);
        });
        return vt;
    }
};
```

## AttrVisitor

提供统一的节点属性访问接口：

```cpp
class AttrVisitor {
public:
    virtual ~AttrVisitor() = default;
    virtual void Visit(std::string_view n, int64_t* v) = 0;
    virtual void Visit(std::string_view n, double* v) = 0;
    virtual void Visit(std::string_view n, std::string* v) = 0;
    virtual void Visit(std::string_view n, Object* v) = 0;
    virtual void Visit(std::string_view n, DataType* v) = 0;
};
```

**设计特点**:
- **类型重载**: 针对不同属性类型提供专门的访问方法
- **名称参数**: 通过属性名称进行访问
- **指针参数**: 使用指针参数支持读写操作

## NodeAttrNameCollector

用于收集节点的所有属性名称：

```cpp
class NodeAttrNameCollector : public AttrVisitor {
public:
    std::vector<std::string> names;

    void Visit(std::string_view n, int64_t*) override {
        names.push_back(std::string(n));
    }

    void Visit(std::string_view n, double*) override {
        names.push_back(std::string(n));
    }

    void Visit(std::string_view n, std::string*) override {
        names.push_back(std::string(n));
    }

    void Visit(std::string_view n, Object*) override {
        names.push_back(std::string(n));
    }

    void Visit(std::string_view n, DataType* v) override {
        names.push_back(std::string(n));
    }
};
```

**使用示例**:
```cpp
McValue NodeGetAttrNames(object_t* o) {
    if (!o) return McValue();

    NodeAttrNameCollector c;
    o->VisitAttrs(&c);  // 调用节点的VisitAttrs方法

    Tuple t = Tuple(c.names.begin(), c.names.end());
    return t;
}
```

## NodeAttrGetter

用于获取指定名称的属性值：

```cpp
class NodeAttrGetter : public AttrVisitor {
public:
    NodeAttrGetter(std::string_view n) : n_(n) {}

    McValue GetValue() const { return v_; }
    bool IsNone() const { return v_.T() != TypeIndex::Null; }

    void Visit(std::string_view n, int64_t* v) override {
        if (n == n_) v_ = McValue(*v);
    }

    void Visit(std::string_view n, double* v) override {
        if (n == n_) v_ = McValue(*v);
    }

    void Visit(std::string_view n, std::string* v) override {
        if (n == n_) v_ = McValue(*v);
    }

    void Visit(std::string_view n, Object* v) override {
        if (n == n_) v_ = McValue(*v);
    }

    void Visit(std::string_view n, DataType* v) override {
        if (n == n_) v_ = McValue(*v);
    }

private:
    std::string_view n_;
    McValue v_;
};
```

**使用示例**:
```cpp
McValue NodeGetAttr(object_t* o, const std::string& n) {
    if (!o) return McValue();

    NodeAttrGetter g(n);
    o->VisitAttrs(&g);
    return g.GetValue();
}
```

## Printer

### Doc

`Doc` 提供了结构化的文档构建功能，支持缩进、换行等格式化操作：

```cpp
class Doc {
public:
    Doc() {}

    Doc& operator<<(const Doc& r);
    Doc& operator<<(const std::string& r);
    Doc& operator<<(const DocAtom& r);

    template <typename T, typename = typename std::enable_if<!std::is_class<T>::value>::type>
    Doc& operator<<(const T& value) {
        std::ostringstream os;
        os << value;
        return *this << os.str();
    }  

    std::string str();

    // 静态构造函数
    static Doc Text(std::string value);
    static Doc RawText(std::string value);
    static Doc NewLine(int indent = 0);
    static Doc Indent(int indent, Doc doc);
    static Doc Brace(std::string open, const Doc& body, 
                     std::string close, int indent = 2);
    static Doc Concat(const std::vector<Doc>& vec, 
                      const Doc& sep = Text(", "));

private:
    std::vector<DocAtom> stream_;
};
```

**核心原子类型**:

```cpp
// 文本节点
class DocTextNode : public DocAtomNode {
public:
    static constexpr const std::string_view NAME = "printer.DocText";
    DEFINE_TYPEINDEX(DocTextNode, DocAtomNode);

    std::string str;
    explicit DocTextNode(std::string str) : str(str) {}
};

// 换行节点
class DocLineNode : public DocAtomNode {
public:
    static constexpr const std::string_view NAME = "printer.DocLineNode";
    DEFINE_TYPEINDEX(DocLineNode, DocAtomNode);

    int indent;
    explicit DocLineNode(int indent) : indent(indent) {}
};
```

**文档生成示例**:
```cpp
// 基础用法
Doc doc;
doc << "function add(";
doc << "a: int, b: int";
doc << ") -> int ";
doc << Doc::Brace("{", 
    Doc::NewLine() << "return a + b;",
    "}");

// 结果：
// function add(a: int, b: int) -> int {
//   return a + b;
// }
```

### AstPrinter
AstPrinter继承了多个访问者，实现对不同AST节点的递归遍历和打印。让我们通过具体例子来理解其遍历机制。

```cpp
class AstPrinter : public PrimExprVisitor<Doc(const PrimExpr&)>,
                   public StmtVisitor<Doc(const Stmt&)>,
                   public TypeVisitor<Doc(const Type&)> {
public:
    explicit AstPrinter() = default;

    Doc Print(const object_r& node);
    Doc PrintFunction(const PrimFunc& fn);

private:
    // 核心分发函数
    Doc PrintType(const Type& t);
    Doc PrintDataType(DataType datatype);
    Doc GetName(const std::string& p);

    // 表达式节点访问者实现
    Doc VisitExpr_(const IntImmNode* op) override;
    Doc VisitExpr_(const PrimVarNode* op) override;
    Doc VisitExpr_(const PrimAddNode* op) override;
    Doc VisitExpr_(const PrimMulNode* op) override;
    Doc VisitExpr_(const PrimCallNode* op) override;

    // 语句节点访问者实现
    Doc VisitStmt_(const AllocaVarStmtNode* op) override;
    Doc VisitStmt_(const AssignStmtNode* op) override;
    Doc VisitStmt_(const ReturnStmtNode* op) override;
    Doc VisitStmt_(const SeqStmtNode* op) override;
    Doc VisitStmt_(const EvaluateNode* op) override;

    // 类型节点访问者实现
    Doc VisitType_(const PrimTypeNode* node) override;

private:
    std::unordered_map<Expr, Doc, object_s, object_e> var_memo_;
    std::unordered_map<Type, Doc, object_s, object_e> t_memo_;
    std::unordered_map<std::string, int> names_;
};
```

AstPrinter的核心是`Print`方法，它根据节点类型分发到不同的访问者：

```cpp
Doc AstPrinter::Print(const object_r& node) {
    // 1. 语句节点：分发到StmtVisitor
    if (node.As<StmtNode>()) {
        return VisitStmt(Downcast<Stmt>(node));
    } 
    // 2. 表达式节点：分发到PrimExprVisitor
    if (node.As<PrimExprNode>()) {
        return VisitExpr(Downcast<PrimExpr>(node));
    }
    // 3. 函数节点：特殊处理
    if (node.As<PrimFuncNode>()) {
        return PrintFunc(Downcast<PrimFunc>(node));
    }
    // 4. 类型节点：分发到TypeVisitor
    if (node.As<TypeNode>()) {
        return PrintType(Downcast<Type>(node));
    }
    // 5. 默认情况：打印类型信息和地址
    std::stringstream ss;
    ss << node->Name() << "@" << node.get();
    return Doc::Text(ss.str());
}
```

#### 表达式遍历

让我们通过一个具体例子来看AstPrinter如何遍历复杂的AST结构：

**输入AST**：
```cpp
// 创建表达式：(x + (y * 42)) + func_call(z)
auto x = PrimVar("x", DataType::Int(32));
auto y = PrimVar("y", DataType::Int(32)); 
auto z = PrimVar("z", DataType::Int(32));
auto const42 = IntImm(DataType::Int(32), 42);

auto mul_expr = PrimMul(y, const42);                    // y * 42
auto add_expr1 = PrimAdd(x, mul_expr);                  // x + (y * 42)
auto call_expr = PrimCall(DataType::Int(32), 
                         GlobalVar("func_call"), {z});   // func_call(z)
auto final_expr = PrimAdd(add_expr1, call_expr);        // (x + (y * 42)) + func_call(z)

AstPrinter printer;
Doc result = printer.Print(final_expr);
```

**详细遍历过程**：

```cpp
// 第1步：Print(final_expr) -> 识别为PrimAddNode
Doc AstPrinter::Print(final_expr) {
    // final_expr是PrimAdd类型，分发到表达式访问者
    return VisitExpr(Downcast<PrimExpr>(final_expr));
}

// 第2步：VisitExpr调用静态分发表，路由到VisitExpr_(PrimAddNode*)
Doc VisitExpr_(const PrimAddNode* op) override {
    Doc doc;
    doc << "(";
    doc << Print(op->a);  // 递归调用：打印左子表达式 (x + (y * 42))
    doc << " + ";
    doc << Print(op->b);  // 递归调用：打印右子表达式 func_call(z)
    doc << ")";
    return doc;
}
```

**左子表达式遍历** `(x + (y * 42))`:
```cpp
// 第3步：Print(op->a) -> 又是一个PrimAddNode
Doc VisitExpr_(const PrimAddNode* op) override {  // 第二层递归
    Doc doc;
    doc << "(";
    doc << Print(op->a);  // 递归：打印 x
    doc << " + ";
    doc << Print(op->b);  // 递归：打印 (y * 42)
    doc << ")";
    return doc;
}

// 第4步：Print(x) -> PrimVarNode
Doc VisitExpr_(const PrimVarNode* op) override {
    Doc doc;
    doc << op->var_name;  // 输出 "x"
    return doc;
}

// 第5步：Print(y * 42) -> PrimMulNode  
Doc VisitExpr_(const PrimMulNode* op) override {
    Doc doc;
    doc << "(";
    doc << Print(op->a);  // 递归：打印 y
    doc << " * ";
    doc << Print(op->b);  // 递归：打印 42
    doc << ")";
    return doc;
}

// 第6步：Print(y) -> PrimVarNode
Doc VisitExpr_(const PrimVarNode* op) override {
    return Doc::Text("y");  // 输出 "y"
}

// 第7步：Print(42) -> IntImmNode
Doc VisitExpr_(const IntImmNode* op) override {
    return PrintConstScalar(op->datatype, op->value);  // 输出 "42"
}
```

**右子表达式遍历** `func_call(z)`:
```cpp
// 第8步：Print(func_call(z)) -> PrimCallNode
Doc VisitExpr_(const PrimCallNode* op) override {
    Doc doc;
    auto* p = op->op.As<OpNode>();
    if (p) {
        doc << "@" << Doc::Text(p->op_name) << "(";
    } else {
        // op->op是GlobalVar类型
        doc << Print(op->op) << "(";  // 递归打印函数名
    }

    for (size_t i = 0; i < op->gs.size(); ++i) {
        if (i > 0) doc << ", ";
        doc << Print(op->gs[i]);  // 递归打印参数 z
    }
    doc << ")";
    return doc;
}

// 第9步：Print(GlobalVar("func_call"))
// GlobalVar不是PrimExpr，回到Print的默认分支
Doc AstPrinter::Print(GlobalVar("func_call")) {
    // 输出 "GlobalVar@<address>" 或者特殊处理输出函数名
    return Doc::Text("func_call");
}

// 第10步：Print(z) -> PrimVarNode
Doc VisitExpr_(const PrimVarNode* op) override {
    return Doc::Text("z");  // 输出 "z"
}
```

**最终输出**：
```
((x + (y * 42)) + func_call(z))
```

#### 语句遍历

让我们看一个更复杂的语句遍历：

**输入AST**：
```cpp
// 创建函数体：
// {
//   int sum = x + y;
//   int result = sum * 2;
//   return result;
// }
auto x = PrimVar("x", DataType::Int(32));
auto y = PrimVar("y", DataType::Int(32));

auto seq_stmt = SeqStmt({
    AllocaVarStmt("sum", DataType::Int(32), PrimAdd(x, y)),
    AllocaVarStmt("result", DataType::Int(32), 
                  PrimMul(PrimVar("sum", DataType::Int(32)), 
                         IntImm(DataType::Int(32), 2))),
    ReturnStmt(PrimVar("result", DataType::Int(32)))
});

AstPrinter printer;
Doc result = printer.Print(seq_stmt);
```

**详细遍历过程**：

```cpp
// 第1步：Print(seq_stmt) -> 识别为SeqStmtNode
Doc AstPrinter::Print(seq_stmt) {
    return VisitStmt(Downcast<Stmt>(seq_stmt));
}

// 第2步：VisitStmt路由到VisitStmt_(SeqStmtNode*)
Doc VisitStmt_(const SeqStmtNode* op) override {
    Doc doc;
    for (const Stmt& stmt : op->s) {  // 遍历语句列表
        doc << Print(stmt) << Doc::NewLine();  // 递归打印每个语句
    }
    return doc;
}
```

**第一个语句** `AllocaVarStmt("sum", ...)`:
```cpp
// 第3步：Print(AllocaVarStmt) -> AllocaVarStmtNode
Doc VisitStmt_(const AllocaVarStmtNode* op) override {
    Doc doc;
    doc << "Alloca " << Print(op->var);          // 打印变量声明
    doc << " = " << Print(op->init_value);       // 打印初始化表达式
    return doc;
}

// 第4步：Print(op->var) -> PrimVar("sum")
Doc VisitExpr_(const PrimVarNode* op) override {
    return Doc::Text("sum");
}

// 第5步：Print(op->init_value) -> PrimAdd(x, y)
Doc VisitExpr_(const PrimAddNode* op) override {
    Doc doc;
    doc << "(";
    doc << Print(op->a);  // "x"
    doc << " + ";
    doc << Print(op->b);  // "y"  
    doc << ")";
    return doc;  // 结果："(x + y)"
}
```

**第二个语句** `AllocaVarStmt("result", ...)`:
```cpp
// 类似过程，最终输出："Alloca result = (sum * 2)"
```

**第三个语句** `ReturnStmt(...)`:
```cpp
// 第6步：Print(ReturnStmt) -> ReturnStmtNode
Doc VisitStmt_(const ReturnStmtNode* op) override {
    Doc doc;
    doc << "Return " << Print(op->value);  // 递归打印返回值
    return doc;
}

// 第7步：Print(PrimVar("result"))
// 结果："Return result"
```

**最终输出**：
```
Alloca sum = (x + y)
Alloca result = (sum * 2)
Return result
```

#### 递归说明

1. **分发入口**：`Print(node)` 根据节点类型分发到不同的访问者
2. **访问者路由**：每个访问者通过静态分发表路由到具体的`VisitXxx_`方法
3. **递归调用**：在`VisitXxx_`方法中通过`Print(child_node)`递归处理子节点
4. **结果组合**：将子节点的打印结果组合成当前节点的完整输出

**关键特点**：
- **深度优先遍历**：先处理子节点，再组合父节点
- **类型安全**：每种节点类型都有专门的处理方法
- **可扩展**：新增节点类型只需添加对应的`VisitXxx_`方法
- **文档化输出**：使用Doc系统支持格式化和缩进

## TextPrinter

提供模块级别的打印功能：

```cpp
class TextPrinter {
public:
    explicit TextPrinter() : ast_printer_() {}

    AstPrinter ast_printer_;

    Doc Print(const Object& n) {
        Doc doc;
        if (n->IsType<IRModuleNode>()) {
            doc << PrintModule(Downcast<IRModule>(n));
        } else {
            doc << ast_printer_.Print(n);
        }
        return doc;
    }

    Doc PrintModule(const IRModule& m);
};
```

## Rewriter

Rewriter提供将AST转换为目标代码的基础框架。与AstPrinter不同，Rewriter直接输出到字符流，并支持SSA（Static Single Assignment）形式的代码生成。让我们通过具体例子来理解其遍历和重写机制。


```cpp
class Rewriter : public PrimExprVisitor<void(const PrimExpr&, std::ostream&)>,
                 public StmtVisitor<void(const Stmt&, std::ostream&)>,
                 public TypeVisitor<void(const Type&, std::ostream&)> {
public:
    virtual ~Rewriter() = default;

    void Init(bool ssa) {
        print_ssa_ = ssa;  // 是否启用SSA形式
    }

    void ResetState() {
        name_dict_.clear();
        ssa_dict_.clear();
        var_dict_.clear();
        m_vec_.clear();
        t_ = 0;
    }

    virtual std::string Done() {
        return stream_.str();
    }

    virtual void InsertFunction(const PrimFunc& func);

protected:
    // 核心遍历接口
    void PrintExpr(const PrimExpr& expr, std::ostream& os);
    void PrintStmt(const Stmt& stmt, std::ostream& os);
    
    // 变量管理
    std::string AllocVarID(const PrimVarNode* v);
    std::string GetVarID(const PrimVarNode* v);
    
    // SSA支持
    std::string SSAGetID(std::string src, Type t, std::ostream& os);
    
    // 作用域管理
    int BeginScope();
    void EndScope(int s);
    void PrintIndent(std::ostream& os);

    // 访问者实现
    void VisitExpr_(const IntImmNode* op, std::ostream& os) override;
    void VisitExpr_(const PrimVarNode* op, std::ostream& os) override;
    void VisitExpr_(const PrimAddNode* op, std::ostream& os) override;
    void VisitExpr_(const PrimMulNode* op, std::ostream& os) override;
    
    void VisitStmt_(const AllocaVarStmtNode* op, std::ostream& os) override;
    void VisitStmt_(const AssignStmtNode* op, std::ostream& os) override;
    void VisitStmt_(const ReturnStmtNode* op, std::ostream& os) override;
    void VisitStmt_(const SeqStmtNode* op, std::ostream& os) override;

private:
    // 状态管理
    std::unordered_map<const BaseExprNode*, std::string> var_dict_;
    std::unordered_map<std::string, int> name_dict_;
    std::unordered_map<std::string, SSAEntry> ssa_dict_;
    std::vector<bool> m_vec_;
    int t_{0};
    bool print_ssa_{false};

protected:
    std::ostringstream stream_;
};
```

### InsertFunction

让我们看一个完整的函数遍历过程：

**输入AST**：
```cpp
// 创建函数：int add_complex(int x, int y) { return (x + y) * 2; }
Array<PrimVar> params = {
    PrimVar("x", DataType::Int(32)),
    PrimVar("y", DataType::Int(32))
};

auto body = ReturnStmt(
    PrimMul(
        PrimAdd(PrimVar("x", DataType::Int(32)), 
                PrimVar("y", DataType::Int(32))),
        IntImm(DataType::Int(32), 2)
    )
);

auto func = PrimFunc(params, {}, body, PrimType(DataType::Int(32)));

Rewriter rewriter;
rewriter.Init(false);  // 非SSA模式
rewriter.InsertFunction(func);
std::string result = rewriter.Done();
```


**详细遍历过程**：

```cpp
void Rewriter::InsertFunction(const PrimFunc& func) {
    ResetState();  // 清理状态

    // 第1步：生成函数签名
    PrintType(func->rt, stream_);              // 输出："int32_t"
    stream_ << " " << GetFuncName(func) << "("; // 输出："test_func("
    
    // 第2步：处理参数列表
    for (size_t i = 0; i < func->gs.size(); ++i) {
        if (i > 0) stream_ << ", ";
        const auto& param = func->gs[i];
        
        // 为参数分配变量ID
        AllocVarID(param.operator->());  // 注册 "x" -> "x", "y" -> "y"
        
        PrintDataType(param->datatype, stream_);  // 输出："int32_t"
        stream_ << " " << GetVarID(param.operator->());  // 输出：" x"
    }
    stream_ << ") {\n";  // 输出：") {\n"
    
    // 当前输出："int32_t test_func(int32_t x, int32_t y) {\n"

    // 第3步：处理函数体
    int scope = BeginScope();  // 进入新作用域，缩进层次+1
    PrintStmt(func->body, stream_);  // 递归处理函数体
    EndScope(scope);  // 退出作用域，缩进层次-1
    
    stream_ << "}\n\n";  // 输出："}\n\n"
}
```

### PrintStmt

**语句遍历的入口**：
```cpp
void Rewriter::PrintStmt(const Stmt& stmt, std::ostream& os) {
    VisitStmt(stmt, os);  // 分发到对应的VisitStmt_方法
}

// 对于我们的ReturnStmt，会路由到：
void VisitStmt_(const ReturnStmtNode* op, std::ostream& os) override {
    PrintIndent(os);  // 输出缩进："  "
    os << "return ";   // 输出："return "
    if (!op->value.none()) {
        PrintExpr(Downcast<PrimExpr>(op->value), os);  // 递归处理返回值表达式
    }
    os << ";\n";  // 输出：";\n"
}
```

### PrintExpr

**表达式遍历的关键机制**：
```cpp
void Rewriter::PrintExpr(const PrimExpr& expr, std::ostream& os) {
    if (print_ssa_) {
        // SSA模式：先生成到临时流，再处理SSA变量
        std::ostringstream temp;
        VisitExpr(expr, temp);
        std::string src = temp.str();
        std::string ssa_var = SSAGetID(src, PrimType(expr->datatype), os);
        os << ssa_var;
    } else {
        // 直接模式：直接输出到目标流
        VisitExpr(expr, os);
    }
}
```

**具体表达式遍历过程**：

对于返回值表达式 `(x + y) * 2`：

```cpp
// 第1步：PrintExpr(PrimMul(...)) -> VisitExpr_(PrimMulNode*)
void VisitExpr_(const PrimMulNode* op, std::ostream& os) override {
    os << "(";
    PrintExpr(op->a, os);  // 递归处理：(x + y)
    os << " * ";
    PrintExpr(op->b, os);  // 递归处理：2
    os << ")";
}

// 第2步：PrintExpr(PrimAdd(...)) -> VisitExpr_(PrimAddNode*)
void VisitExpr_(const PrimAddNode* op, std::ostream& os) override {
    os << "(";
    PrintExpr(op->a, os);  // 递归处理：x
    os << " + ";
    PrintExpr(op->b, os);  // 递归处理：y
    os << ")";
}

// 第3步：PrintExpr(PrimVar("x")) -> VisitExpr_(PrimVarNode*)
void VisitExpr_(const PrimVarNode* op, std::ostream& os) override {
    os << GetVarID(op);  // 输出已注册的变量ID："x"
}

// 第4步：PrintExpr(PrimVar("y")) -> VisitExpr_(PrimVarNode*)
void VisitExpr_(const PrimVarNode* op, std::ostream& os) override {
    os << GetVarID(op);  // 输出："y"
}

// 第5步：PrintExpr(IntImm(2)) -> VisitExpr_(IntImmNode*)
void VisitExpr_(const IntImmNode* op, std::ostream& os) override {
    os << op->value;  // 输出："2"
}
```

**递归展开的输出过程**：
```cpp
// VisitExpr_(PrimMulNode*):
os << "(";            // 输出："("
// PrintExpr(op->a) 递归展开：
    os << "(";        // 输出："("
    os << "x";        // 输出："x"
    os << " + ";      // 输出：" + "
    os << "y";        // 输出："y"
    os << ")";        // 输出：")"
// 回到 VisitExpr_(PrimMulNode*):
os << " * ";          // 输出：" * "
os << "2";            // 输出："2"
os << ")";            // 输出：")"

// 最终表达式输出："((x + y) * 2)"
```

**最终生成的完整代码**：
```c
int32_t test_func(int32_t x, int32_t y) {
  return ((x + y) * 2);
}
```

### SSA

现在让我们看启用SSA的情况：

```cpp
Rewriter rewriter;
rewriter.Init(true);  // 启用SSA模式
rewriter.InsertFunction(func);
```

**SSA遍历机制**：

SSA模式的关键在于`SSAGetID`方法，它为复杂表达式生成临时变量：

```cpp
std::string Rewriter::SSAGetID(std::string src, Type t, std::ostream& os) {
    // 1. 如果表达式已经是一个简单名称，直接返回
    if (name_dict_.count(src)) {
        return src;
    }

    // 2. 检查是否已经为此表达式生成过SSA变量
    auto it = ssa_dict_.find(src);
    if (it != ssa_dict_.end() && m_vec_.at(it->second.s_)) {
        return it->second.v_;  // 返回已存在的SSA变量
    }

    // 3. 生成新的SSA变量
    SSAEntry e;
    e.v_ = GetUniqueName("_ssa_");  // 生成唯一名称：_ssa_0, _ssa_1, ...
    e.s_ = m_vec_.size() - 1;       // 记录作用域
    ssa_dict_[src] = e;

    // 4. 输出SSA赋值语句
    PrintIndent(os);
    PrintSSA(e.v_, src, t, os);
    return e.v_;
}

void PrintSSA(const std::string& target, const std::string& src, Type t, std::ostream& os) {
    PrintType(t, os);
    os << " " << target << " = " << src << ";\n";
}
```

**SSA遍历的具体过程**：

对于相同的表达式 `(x + y) * 2`，SSA模式下的遍历：

```cpp
// 第1步：ProcessExpr(PrimMul(...))
void PrintExpr(const PrimExpr& expr, std::ostream& os) {
    std::ostringstream temp;
    VisitExpr(expr, temp);  // 先生成到临时流
    std::string src = temp.str();  // src = "((x + y) * 2)"
    std::string ssa_var = SSAGetID(src, PrimType(expr->datatype), os);
    os << ssa_var;  // 输出SSA变量名
}

// 第2步：VisitExpr生成临时代码
// 这个过程与非SSA模式相同，但输出到临时流temp
// temp.str() = "((x + y) * 2)"

// 第3步：SSAGetID处理复杂表达式
std::string SSAGetID("((x + y) * 2)", PrimType(DataType::Int(32)), os) {
    // 生成新的SSA变量 "_ssa_0"
    
    PrintIndent(os);  // 输出："  "
    PrintSSA("_ssa_0", "((x + y) * 2)", PrimType(DataType::Int(32)), os);
    // 输出："int32_t _ssa_0 = ((x + y) * 2);\n"
    
    return "_ssa_0";
}

// 第4步：使用SSA变量
// 在return语句中输出："_ssa_0"而不是完整表达式
```

**SSA形式的最终输出**：
```c
int32_t test_func(int32_t x, int32_t y) {
  int32_t _ssa_0 = ((x + y) * 2);
  return _ssa_0;
}
```

让我们看一个更复杂的例子：

**输入AST**：
```cpp
// 函数体：
// {
//   int a = x + y;
//   int b = a * 2;
//   return b + (x * y);
// }
auto seq_stmt = SeqStmt({
    AllocaVarStmt("a", DataType::Int(32), PrimAdd(x, y)),
    AllocaVarStmt("b", DataType::Int(32), 
                  PrimMul(PrimVar("a", DataType::Int(32)), IntImm(DataType::Int(32), 2))),
    ReturnStmt(PrimAdd(PrimVar("b", DataType::Int(32)), 
                      PrimMul(x, y)))
});
```

**SSA遍历过程**：

```cpp
// 第1步：VisitStmt_(SeqStmtNode*)
for (const Stmt& stmt : op->s) {
    PrintStmt(stmt, os);  // 逐个处理语句
}

// 第2步：处理第一个AllocaVarStmt
void VisitStmt_(const AllocaVarStmtNode* op, std::ostream& os) override {
    auto var = op->var.As<PrimVarNode>();
    PrintIndent(os);
    std::string var_id = AllocVarID(var);  // 分配变量ID："a"
    PrintType(PrimType(var->datatype), os);  // 输出："int32_t"
    os << " " << var_id;  // 输出：" a"

    if (!op->init_value.none()) {
        os << " = ";
        if (print_ssa_) {
            // SSA模式：init_value直接输出，不生成SSA变量
            std::ostringstream temp;
            VisitExpr(Downcast<PrimExpr>(op->init_value), temp);
            os << temp.str();  // 输出："(x + y)"
        } else {
            PrintExpr(Downcast<PrimExpr>(op->init_value), os);
        }
    }
    os << ";\n";
}
// 输出："  int32_t a = (x + y);\n"

// 第3步：处理第二个AllocaVarStmt
// 类似过程，输出："  int32_t b = (a * 2);\n"

// 第4步：处理ReturnStmt
void VisitStmt_(const ReturnStmtNode* op, std::ostream& os) override {
    PrintIndent(os);
    os << "return ";
    PrintExpr(Downcast<PrimExpr>(op->value), os);  // 这里会触发SSA
    os << ";\n";
}

// 第5步：PrintExpr处理 PrimAdd(b, PrimMul(x, y))
// temp.str() = "(b + (x * y))"
// SSAGetID生成："int32_t _ssa_0 = (b + (x * y));\n"
// 然后输出："_ssa_0"
```

**最终SSA形式的输出**：
```c
int32_t test_func(int32_t x, int32_t y) {
  int32_t a = (x + y);
  int32_t b = (a * 2);
  int32_t _ssa_0 = (b + (x * y));
  return _ssa_0;
}
```

### 作用域

Rewriter还有重要的作用域和变量管理机制：

```cpp
// 作用域管理
int BeginScope() {
    int s = m_vec_.size();
    m_vec_.push_back(true);  // 添加新作用域标记
    t_ += 1;                 // 增加缩进层次
    return s;
}

void EndScope(int s) {
    m_vec_[s] = false;  // 标记作用域失效
    t_ -= 1;            // 减少缩进层次
}

// 变量ID管理：避免重名冲突
std::string GetUniqueName(std::string p) {
    // 处理特殊字符
    for (size_t i = 0; i < p.size(); ++i) {
        if (p[i] == '.') {
            p[i] = '_';
        }
    }

    auto it = name_dict_.find(p);
    if (it != name_dict_.end()) {
        int counter = it->second + 1;
        it->second = counter;
        return p + "_" + std::to_string(counter);  // 生成：var_1, var_2, ...
    }

    name_dict_[p] = 0;
    return p;  // 第一次使用，直接返回原名
}
```

### 关键点

1. **流式输出**：直接输出到字符流，避免中间数据结构
2. **SSA支持**：能够生成静态单赋值形式的代码
3. **变量管理**：自动处理变量命名冲突和作用域
4. **缩进管理**：自动处理代码缩进和格式化
5. **递归遍历**：通过访问者模式递归处理AST结构

**与AstPrinter的区别**：
- **输出方式**：Rewriter直接输出字符，AstPrinter构建Doc结构
- **SSA支持**：Rewriter支持SSA变换，AstPrinter不支持
- **用途**：Rewriter用于代码生成，AstPrinter用于调试和显示

## SourceRewriter

继承自Rewriter，专门用于生成可编译的C代码：

```cpp
class SourceRewriter : public Rewriter {
public:
    SourceRewriter() = default;

    void Init(bool ssa);
    void InsertFunction(const PrimFunc& fn);
    std::string Done();

private:
    void EmitHeaders();
    void EmitModuleContext();
    void BeginAnonymousNamespace();
    void EndAnonymousNamespace();
    void DefineCAPIFunction(const PrimFunc& fn);
    void EmitModuleRegistry();

    struct TypeInfo {
        const char* index_name;
        const char* type_name;
    };

    TypeInfo GetTypeInfo(DataType dt) {
        if (dt.IsInt()) {
            return {"TypeIndex::Int", "int"};
        } else if (dt.IsFloat()) {
            return {"TypeIndex::Float", "float"};
        } else if (dt.IsVoid()) {
            return {"TypeIndex::Void", "void"};
        }
        return {"TypeIndex::Object", "object"};
    }

    void EmitArgumentTypeCheck(const PrimFunc& f, int scope);
    void EmitModuleInitFunction();

private:
    std::vector<std::string> func_names_;
};
```

**代码生成流程**:

1. **头文件包含**:
```cpp
void SourceRewriter::EmitHeaders() {
    const char* headers[] = {
        "<stdint.h>", "<string.h>", "<stdexcept>", "<stdio.h>",
        "\"runtime_value.h\"", "\"parameters.h\"", 
        "\"registry.h\"", "\"c_api.h\""
    };
    
    for (const auto& header : headers) {
        stream_ << "#include " << header << "\n";
    }
    stream_ << "\nusing namespace mc::runtime;\n\n";
}
```

2. **模块上下文**:
```cpp
void SourceRewriter::EmitModuleContext() {
    stream_ << "const char* __mc_module_version = \"1.0.0\";\n\n";
    stream_ << "void* __mc_module_ctx = nullptr;\n\n";
    stream_ << "static thread_local char error_buffer[1024];\n\n";
}
```

3. **C API包装函数**:
```cpp
void SourceRewriter::DefineCAPIFunction(const PrimFunc& f) {
    std::string func_name = GetFuncName(f);
    
    // C API 包装函数声明
    stream_ << "int " << func_name << "__c_api"
            << "(Value* args, int num_args, Value* ret_val, void* resource_handle) {\n";
    
    auto scope = BeginScope();

    // 生成参数检查
    EmitArgumentTypeCheck(f, scope);

    // 调用原始函数
    stream_ << "    auto result = " << func_name << "(";
    for (size_t i = 0; i < f->gs.size(); ++i) {
        if (i > 0) stream_ << ", ";
        auto dt = f->gs[i]->datatype;
        stream_ << "args[" << i << "].u.";
        if (dt.IsInt()) {
            stream_ << "v_int";
        } else if (dt.IsFloat()) {
            stream_ << "v_float";
        } else if (dt.IsBool()) {
            stream_ << "v_bool";
        }
    }
    stream_ << ");\n\n";
    
    // 设置返回值
    auto ret_type = GetTypeInfo(f->rt.As<PrimTypeNode>()->datatype);
    stream_ << "    ret_val->t = " << ret_type.index_name << ";\n";
    if (!f->rt.As<PrimTypeNode>()->datatype.IsVoid()) {
        stream_ << "    ret_val->u.v_" << ret_type.type_name << " = result;\n";
    }
    stream_ << "    return 0;\n";

    EndScope(scope);
    stream_ << "}\n\n";
}
```

4. **模块注册**:
```cpp
void SourceRewriter::EmitModuleRegistry() {
    stream_ << "extern \"C\" {\n\n";

    // 函数数组
    stream_ << "BackendFunc __mc_func_array__[] = {\n";
    for (const auto& name : func_names_) {
        stream_ << "    (BackendFunc)" << name << "__c_api,\n";
    }
    stream_ << "};\n\n";

    // 注册表
    stream_ << "FuncRegistry __mc_func_registry__ = {\n";
    std::string func_names_str = "\\1" + func_names_[0];
    stream_ << "    \"" << func_names_str << "\",\n";
    stream_ << "    __mc_func_array__,\n";
    stream_ << "};\n\n";

    stream_ << "} // extern \"C\"\n";
}
```

## 举例

### Printer

**打印函数**:
```cpp
// 创建函数
Array<PrimVar> params = {
    PrimVar("a", DataType::Int(32)),
    PrimVar("b", DataType::Int(32))
};

auto body = SeqStmt({
    AllocaVarStmt("sum", DataType::Int(32), 
                  PrimAdd(PrimVar("a", DataType::Int(32)), 
                         PrimVar("b", DataType::Int(32)))),
    ReturnStmt(PrimVar("sum", DataType::Int(32)))
});

auto func = PrimFunc(params, {}, body, PrimType(DataType::Int(32)));

// 打印函数
AstPrinter printer;
Doc doc = printer.PrintFunction(func);
std::cout << doc.str() << std::endl;

// 输出：
// fn(a: int32, b: int32) -> int32 {
//   Alloca sum = (a + b)
//   Return sum
// }
```

### Rewriter

**生成C代码**:
```cpp
// 使用SourceRewriter生成C代码
auto func = PrimFunc(params, {}, body, PrimType(DataType::Int(32)));

SourceRewriter rewriter;
rewriter.Init(true);  // 启用SSA形式
rewriter.InsertFunction(func);
std::string c_code = rewriter.Done();

std::cout << c_code << std::endl;
```

**生成的C代码**:
```c
#include <stdint.h>
#include <string.h>
#include <stdexcept>
#include <stdio.h>
#include "runtime_value.h"
#include "parameters.h"
#include "registry.h"
#include "c_api.h"

using namespace mc::runtime;

const char* __mc_module_version = "1.0.0";
void* __mc_module_ctx = nullptr;
static thread_local char error_buffer[1024];

namespace {

int32_t test_func(int32_t a, int32_t b) {
  int32_t _ssa_0 = (a + b);
  int32_t sum = _ssa_0;
  return sum;
}

int test_func__c_api(Value* args, int num_args, Value* ret_val, void* resource_handle) {
    if (num_args != 2) {
        snprintf(error_buffer, sizeof(error_buffer),
                "test_func() takes 2 positional arguments but %d were given",
                num_args);
        SetError(error_buffer);
        return -1;
    }

    if (args[0].t != TypeIndex::Int) {
        snprintf(error_buffer, sizeof(error_buffer),
                "test_func argument 0 type mismatch, expect 'int' type");
        SetError(error_buffer);
        return -1;
    }
    if (args[1].t != TypeIndex::Int) {
        snprintf(error_buffer, sizeof(error_buffer),
                "test_func argument 1 type mismatch, expect 'int' type");
        SetError(error_buffer);
        return -1;
    }

    auto result = test_func(args[0].u.v_int, args[1].u.v_int);

    ret_val->t = TypeIndex::Int;
    ret_val->u.v_int = result;
    return 0;
}

} // namespace

extern "C" {

BackendFunc __mc_func_array__[] = {
    (BackendFunc)test_func__c_api,
};

FuncRegistry __mc_func_registry__ = {
    "\\1test_func",
    __mc_func_array__,
};

const char* __mc_closures_names__ = "0\\000";

__attribute__((constructor))
void init_module() {
    printf("Module loaded, registry at: %p\\n", &__mc_func_registry__);
}

__attribute__((destructor))
void cleanup_module() {
    if (__mc_module_ctx) {
        __mc_module_ctx = nullptr;
    }
}

} // extern "C"
```

### NodeAttr

**获取节点属性**:
```cpp
// 创建一个变量节点
auto var = PrimVar("test_var", DataType::Int(32));

// 获取所有属性名称
McValue attr_names = NodeGetAttrNames(var.get());
Tuple names_tuple = attr_names.As<Tuple>();

std::cout << "Attributes: ";
for (size_t i = 0; i < names_tuple.size(); ++i) {
    auto name = names_tuple[i].As<std::string>();
    std::cout << name << " ";
}
std::cout << std::endl;

// 获取特定属性值
McValue var_name = NodeGetAttr(var.get(), "var_name");
std::string name_str = var_name.As<std::string>();
std::cout << "Variable name: " << name_str << std::endl;

McValue datatype = NodeGetAttr(var.get(), "datatype");
DataType dt = datatype.As<DataType>();
std::cout << "Data type: " << DtToStr(dt.data_) << std::endl;
```
