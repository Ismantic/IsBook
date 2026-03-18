# AST

## 概述

AST 提供了一个完整的抽象语法树基础设施，用来表示和操作程序的语法结构。当前由三个核心模块组成：表达式（Expression）、语句（Statement）和函数（Function），构建了一个统一的中间表示（IR）框架。

**关键点**
1. **分层设计**: 表达式 → 语句 → 函数 → 模块的层次化组织
2. **类型安全**: 基于对象系统的强类型 AST 节点
3. **统一接口**: 所有 AST 节点都继承自 `object_t`，参与统一的类型和内存管理
4. **访问者模式**: 支持 `VisitAttrs` 进行 AST 遍历和分析
5. **可扩展性**: 清晰的继承层次，便于添加新的 AST 节点类型**关键点**

**AST 层次结构**：
```
object_t
├── ExprNode (基础表达式)
│   ├── PrimExprNode (原始表达式)
│   │   ├── PrimVarNode (变量)
│   │   ├── IntImmNode (整数字面量)
│   │   ├── PrimAddNode/PrimMulNode (二元运算)
│   │   ├── PrimNotNode (一元运算)
│   │   └── PrimCallNode (函数调用)
│   ├── AstExprNode (AST 表达式)
│   │   └── OpNode (运算符)
│   └── GlobalVarNode (全局变量)
├── StmtNode (语句)
│   ├── ExprStmtNode (表达式语句)
│   ├── AllocaVarStmtNode (变量声明)
│   ├── AssignStmtNode (赋值语句)
│   ├── ReturnStmtNode (返回语句)
│   ├── EvaluateNode (求值语句)
│   └── SeqStmtNode (顺序语句)
├── TypeNode (类型系统)
│   ├── PrimTypeNode (原始类型)
│   ├── ClassTypeNode (类类型)
│   ├── TypeVarNode (类型变量)
│   └── GlobalTypeVarNode (全局类型变量)
├── FuncNode (函数)
│   ├── PrimFuncNode (原始函数)
│   └── AstFuncNode (AST 函数)
└── IRModuleNode (模块)
```

## Node

**DEFINE_NODE_CLASS 宏**

所有 AST 容器类都使用统一的宏来定义标准接口：

```cpp
#define DEFINE_NODE_CLASS(TypeName, ParentType, NodeType) \
  TypeName() noexcept = default;    \
  explicit TypeName(object_p<object_t> n) noexcept : ParentType(std::move(n)) {} \
  TypeName(const TypeName& other) noexcept = default; \
  TypeName(TypeName&& other) noexcept = default; \
  TypeName& operator=(const TypeName& other) noexcept = default;    \
  TypeName& operator=(TypeName&& other) noexcept = default; \
  const NodeType* operator->() const noexcept {  \
    return static_cast<const NodeType*>(get()); \
  } \
  NodeType* operator->() noexcept { \
    return static_cast<NodeType*>(get_mutable()); \
  } \
  using ContainerType = NodeType;
```

**设计特点**:
- **统一接口**: 所有 AST 容器类都有相同的构造、赋值和访问接口
- **类型安全**: 通过模板参数指定具体的节点类型
- **智能指针语义**: 基于 `object_p` 的自动内存管理
- **const 正确性**: 分离只读和可变访问

## Expression

### ExprNode/Expr


所有表达式的基类：

```cpp
class ExprNode : public object_t {
public:
    static constexpr const std::string_view NAME = "BaseExpr";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(ExprNode, object_t);
};

class Expr : public object_r {
public:
    DEFINE_NODE_CLASS(Expr, object_r, ExprNode);
};
```

**设计特点**:
- **类型擦除**: `Expr` 可以持有任何具体的表达式类型
- **动态类型**: 使用 `TypeIndex::Dynamic` 进行动态类型分配
- **多态基础**: 为所有表达式提供统一的接口

### PrimExprNode/PrimExpr

原始表达式，包含数据类型信息：

```cpp
class PrimExprNode : public ExprNode {
public:
    static constexpr const std::string_view NAME = "PrimExpr";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(PrimExprNode, ExprNode);

    DataType datatype;  // 表达式的数据类型
};

class PrimExpr : public Expr {
public:
    DEFINE_NODE_CLASS(PrimExpr, Expr, PrimExprNode);
};
```

### PrimVarNode/PrimVar

原始变量表达式：

```cpp
class PrimVarNode : public PrimExprNode {
public:
    static constexpr const std::string_view NAME = "PrimVar";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(PrimVarNode, PrimExprNode);

    std::string var_name;  // 变量名

    void VisitAttrs(AttrVisitor* v) override;
};

class PrimVar : public PrimExpr {
public:
    DEFINE_NODE_CLASS(PrimVar, PrimExpr, PrimVarNode);

    explicit PrimVar(std::string name, DataType t);
    explicit PrimVar(std::string name, Type t);
};
```

**访问者模式实现**:
```cpp
void PrimVarNode::VisitAttrs(AttrVisitor* v) {
    v->Visit("var_name", &var_name);
    v->Visit("datatype", &datatype);
}
```

**构造函数实现**:
```cpp
PrimVar::PrimVar(std::string name, DataType datatype) {
    auto node = MakeObject<PrimVarNode>();
    node->var_name = std::move(name);
    node->datatype = std::move(datatype);
    data_ = std::move(node);
}

PrimVar::PrimVar(std::string name, Type t) {
    auto node = MakeObject<PrimVarNode>();
    node->var_name = std::move(name);
    node->datatype = GetRuntimeDataType(t);  // 类型转换
    data_ = std::move(node);
}
```

### GlobalVarNode/GlobalVar

全局变量表达式：

```cpp
class GlobalVarNode : public ExprNode {
public:
    static constexpr const std::string_view NAME = "GlobalVar";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(GlobalVarNode, ExprNode);

    std::string var_name;  // 全局变量名
};

class GlobalVar : public Expr {
public:
    DEFINE_NODE_CLASS(GlobalVar, Expr, GlobalVarNode);

    explicit GlobalVar(std::string name);

    std::string var_name() const {
        return static_cast<const GlobalVarNode*>(get())->var_name;
    }
};
```

### IntImmNode/IntImm

整数字面量：

```cpp
class IntImmNode : public PrimExprNode {
public:
    static constexpr const std::string_view NAME = "IntImm";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(IntImmNode, PrimExprNode);
    
    int64_t value;  // 整数值
};

class IntImm : public PrimExpr {
public:
    DEFINE_NODE_CLASS(IntImm, PrimExpr, IntImmNode);

    explicit IntImm(DataType datatype, int64_t value);
};
```

### Bool


布尔字面量（基于 IntImm 实现）：

```cpp
class Bool : public PrimExpr {
public:
    DEFINE_NODE_CLASS(Bool, PrimExpr, IntImmNode);  // 复用 IntImmNode

    explicit Bool(bool value);

    operator bool() const {
        if (const auto* node = dynamic_cast<const IntImmNode*>(get())) {
            return node->value != 0;
        }
        return false;
    }
};
```

**实现细节**:
```cpp
Bool::Bool(bool value) {
    auto node = MakeObject<IntImmNode>();
    node->value = value ? 1 : 0;
    node->datatype = DataType::Bool();
    data_ = std::move(node);
}
```

### Binary运算

**运算模板**:

```cpp
template <typename T>
class PrimBinaryOpNode : public PrimExprNode {
public:
  static constexpr const int32_t INDEX = TypeIndex::Dynamic;
  DEFINE_TYPEINDEX(T, PrimExprNode);

  PrimExpr a;  // 左操作数
  PrimExpr b;  // 右操作数

  void VisitAttrs(AttrVisitor* v) override;
};

template <typename T>
void PrimBinaryOpNode<T>::VisitAttrs(AttrVisitor* v) {
    v->Visit("datatype", &(this->datatype));
    v->Visit("a", &a);
    v->Visit("b", &b);
}
```

**加法运算**:
```cpp
class PrimAddNode : public PrimBinaryOpNode<PrimAddNode> {
public:
    static constexpr const std::string_view NAME = "PrimAdd";
};

class PrimAdd : public PrimExpr {
public:
    DEFINE_NODE_CLASS(PrimAdd, PrimExpr, PrimAddNode);

    explicit PrimAdd(PrimExpr a, PrimExpr b);
};
```

**乘法运算**:
```cpp
class PrimMulNode : public PrimBinaryOpNode<PrimMulNode> {
public:
    static constexpr const std::string_view NAME = "PrimMul";
};

class PrimMul : public PrimExpr {
public:
    DEFINE_NODE_CLASS(PrimMul, PrimExpr, PrimMulNode);

    explicit PrimMul(PrimExpr a, PrimExpr b);
};
```

**实现示例**:
```cpp
PrimAdd::PrimAdd(PrimExpr a, PrimExpr b) {
    auto node = MakeObject<PrimAddNode>();
    node->a = std::move(a);
    node->b = std::move(b);
    node->datatype = node->a->datatype;  // 继承操作数类型
    data_ = std::move(node);
}
```

### Unary运算

**逻辑非运算**:
```cpp
class PrimNotNode : public PrimExprNode {
public:
    static constexpr const std::string_view NAME = "PrimNot";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(PrimNotNode, PrimExprNode);

    PrimExpr a;  // 操作数
};

class PrimNot : public PrimExpr {
public:
    DEFINE_NODE_CLASS(PrimNot, PrimExpr, PrimNotNode);

    explicit PrimNot(PrimExpr a);
};
```

**实现细节**:
```cpp
PrimNot::PrimNot(PrimExpr a) {
    auto node = MakeObject<PrimNotNode>();
    node->datatype = DataType::Bool(a->datatype.a());  // 结果为布尔类型
    node->a = std::move(a);
    data_ = std::move(node);
}
```

### 函数调用

```cpp
class PrimCallNode : public PrimExprNode {
public:
    static constexpr const std::string_view NAME = "PrimCall";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(PrimCallNode, PrimExprNode);

    Expr op;              // 被调用的函数
    Array<PrimExpr> gs;       // 参数列表
};

class PrimCall : public PrimExpr {
public:
    DEFINE_NODE_CLASS(PrimCall, PrimExpr, PrimCallNode);

    explicit PrimCall(DataType datatype, Expr op, Array<PrimExpr> gs);
};
```

**实现细节**:
```cpp
PrimCall::PrimCall(DataType datatype, Expr op, Array<PrimExpr> gs) {
    auto node = MakeObject<PrimCallNode>();
    node->datatype = std::move(datatype);
    node->op = std::move(op);
    node->gs = std::move(gs);
    data_ = std::move(node);
}
```

注意前面实现的Array容器派上用场了。

## Ops

实现了一个高级的运算符抽象系统，将具体的运算操作抽象为可重用的运算符对象。这个系统在原始表达式的基础上提供了更灵活和可扩展的运算符管理。

### OpNode/Op

运算符节点，表示一个抽象的运算操作：

```cpp
class OpNode : public AstExprNode {
public:
    static constexpr const std::string_view NAME = "Op";
    DEFINE_TYPEINDEX(OpNode, AstExprNode);

    std::string op_name;  // 运算符名称
};

class Op : public AstExpr {
public:
    DEFINE_NODE_CLASS(Op, AstExpr, OpNode);

    // 获取指定名称的运算符
    static const Op& Get(const std::string& name);
};
```

**设计特点**:
- **继承自 AstExprNode**: 运算符本身也是一种表达式
- **名称标识**: 通过字符串名称唯一标识运算符
- **单例管理**: 相同名称的运算符在全局范围内是唯一的


### OpRegEntry 

运算符注册系统的核心组件，管理运算符的元信息：

```cpp
class OpRegEntry {
public:
    static OpRegEntry& RegisterOrGet(const std::string& name);

    const Op& op() const {
        return op_;
    }

    // 设置运算符的参数个数
    OpRegEntry& set_num(int32_t n) {
        num_gs = n;
        return *this;
    }

private:
    std::string name;   // 运算符名称
    Op op_;             // 运算符对象
    int32_t num_gs{0};  // 参数个数

    friend class OpRegistry;
};
```

**功能特点**:
- **元信息管理**: 存储运算符的参数个数等元信息
- **流式接口**: `set_num()` 返回自身引用，支持链式调用
- **懒加载**: 通过 `RegisterOrGet` 实现按需创建


### OpRegistry

管理所有运算符的全局注册表：

```cpp
class OpRegistry {
public:
    static OpRegistry* Global() {
        static OpRegistry instance;
        return &instance;
    }

    OpRegEntry& RegisterOrGet(const std::string& name) {
        std::lock_guard<std::mutex> lock(mutex_);
        auto it = registry_.find(name);
        if (it != registry_.end()) {
            return it->second;  // 返回已存在的条目
        }
        
        // 创建新的运算符条目
        OpRegEntry entry;
        entry.name = name;
        auto node = MakeObject<OpNode>();
        node->op_name = name;
        entry.op_ = Op(std::move(node));
        
        registry_[name] = entry;
        return registry_[name];
    }

private:
    OpRegistry() = default;
    std::mutex mutex_;                                      // 线程安全保护
    std::unordered_map<std::string, OpRegEntry> registry_; // 运算符注册表
};
```

**设计特点**:
- **单例模式**: 全局唯一的注册表实例
- **线程安全**: 使用互斥锁保护并发访问
- **按需创建**: 运算符在首次访问时才创建
- **持久化存储**: 已注册的运算符永久保存在注册表中


### 运算符定义宏

系统提供了一系列宏来简化运算符的定义和注册：

#### DEFINE_OP 宏

```cpp
#define DEFINE_OP(OpName) \
  const Op& OpName() { \
    static const Op& op = Op::Get("ast."#OpName); \
    return op; \
  } \
  auto _unused_##OpName = REGISTER_OP("ast." #OpName)
```

**宏展开示例**:
```cpp
// DEFINE_OP(add) 展开为：
const Op& add() { 
    static const Op& op = Op::Get("ast.add"); 
    return op; 
} 
auto _unused_add = REGISTER_OP("ast.add")
```

**功能说明**:
- **访问器函数**: 创建返回运算符的函数
- **静态初始化**: 使用静态变量确保运算符只创建一次
- **自动注册**: 通过静态变量的初始化触发运算符注册

#### REGISTER_MAKE_BINARY_OP 宏

```cpp
#define REGISTER_MAKE_BINARY_OP(Node, Func)                          \
  REGISTER_GLOBAL("ast." #Node).SetBody([](PrimExpr a, PrimExpr b) { \
    return (Func(a, b));                                       \
  })
```

**功能**: 为二元运算符注册全局构造函数

#### REGISTER_MAKE_UNARY_OP 宏

```cpp
#define REGISTER_MAKE_UNARY_OP(Node, Func)               \
  REGISTER_GLOBAL("ast." #Node).SetBody([](PrimExpr a) { \
    return (Func(a));                                    \
  })
```

**功能**: 为一元运算符注册全局构造函数


### 内置运算符

```cpp
PrimExpr op_if_then_else(PrimExpr c, PrimExpr t, PrimExpr f) {
    // 类型检查：条件必须是布尔类型
    if (c->datatype != DataType::Bool()) {
        throw std::runtime_error("if_then_else only accepts boolean condition");
    }

    // 编译时优化：如果条件是常量，直接返回对应分支
    if (c.get()->template IsType<IntImmNode>()) {
        auto* imm = static_cast<const IntImmNode*>(c.get());
        return imm->value != 0 ? t : f;
    }

    // 运行时条件：创建函数调用表达式
    return PrimCall(t->datatype, 
                    if_then_else(),    // 运算符作为被调用函数
                    {c, t, f});        // 参数列表
}

DEFINE_OP(if_then_else).set_num(3);
```

op_if_then_else会生成一个PrimCall，而PrimCall中的参数if_then_else是通过DEFINE_OP定义的。

**实现特点**:
- **类型安全**: 编译时检查条件表达式的类型
- **常量折叠**: 对常量条件进行编译时优化
- **函数调用**: 运行时条件转换为函数调用表达式
- **类型推导**: 结果类型从 then 分支推导


```cpp
// Op 对象的核心价值：为 PrimCall 提供统一的操作符表示
// 无论是内置运算符还是用户函数，都可以用相同的方式调用

// 内置运算符
PrimCall(result_type, if_then_else(), {cond, then_expr, else_expr});

// 用户函数
PrimCall(result_type, GlobalVar("user_func"), {arg1, arg2});

// 复杂表达式
PrimCall(result_type, some_complex_expr, {args...});
```

```cpp
// 添加新的运算符只需要：
// 1. 实现运算符函数
PrimExpr op_custom(PrimExpr a, PrimExpr b) {
    // 自定义逻辑
    return PrimCall(a->datatype, custom(), {a, b});
}

// 2. 定义运算符
DEFINE_OP(custom).set_num(2);
```

Op 对象的真正价值在于提供统一的函数调用表示：

```cpp
// 内置运算符调用
auto builtin_call = PrimCall(result_type, if_then_else(), {cond, then_expr, else_expr});

// 用户函数调用  
auto user_call = PrimCall(result_type, GlobalVar("user_func"), {arg1, arg2});

// 都使用相同的 PrimCall 结构，op 字段可以是：
// - Op 对象（内置运算符）
// - GlobalVar（用户函数）
// - 其他 BaseExpr（更复杂的表达式）
```

## Statement

### StmtNode/Stmt

所有语句的基类：

```cpp
class StmtNode : public object_t {
public:
    static constexpr const std::string_view NAME = "Stmt";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(StmtNode, object_t);
};

class Stmt : public object_r {
public:
    DEFINE_NODE_CLASS(Stmt, object_r, StmtNode);
};
```

### ExprStmtNode/ExprStmt

表达式语句（将表达式作为语句执行）：

```cpp
class ExprStmtNode : public StmtNode {
public:
    static constexpr const std::string_view NAME = "ExprStmt";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(ExprStmtNode, StmtNode);

    Expr expr;  // 要执行的表达式
};

class ExprStmt : public Stmt {
public:
    DEFINE_NODE_CLASS(ExprStmt, Stmt, ExprStmtNode);

    explicit ExprStmt(Expr expr);
};
```

实现细节：
```cpp
ExprStmt::ExprStmt(Expr expr) {
    auto node = MakeObject<ExprStmtNode>();
    node->expr = std::move(expr);
    data_ = std::move(node);
}

REGISTER_TYPEINDEX(ExprStmtNode);
```

### AllocaVarStmtNode/AllocaVarStmt

变量声明语句：

```cpp
class AllocaVarStmtNode : public StmtNode {
public:
    static constexpr const std::string_view NAME = "AllocaVarStmt";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(AllocaVarStmtNode, StmtNode);

    PrimVar var;              // 声明的变量
    Expr init_value;      // 初始化表达式

    void VisitAttrs(AttrVisitor* v) override;
};

class AllocaVarStmt : public Stmt {
public:
    DEFINE_NODE_CLASS(AllocaVarStmt, Stmt, AllocaVarStmtNode);

    explicit AllocaVarStmt(std::string name, DataType datatype, 
                           Expr init_value = Expr());
};
```

**访问者模式实现**:
```cpp
void AllocaVarStmtNode::VisitAttrs(AttrVisitor* v) {
    v->Visit("var", &var);
    v->Visit("init_value", &init_value);
}
```

**实现细节**:
```cpp
AllocaVarStmt::AllocaVarStmt(std::string name, DataType datatype, Expr init_value) {
    auto node = MakeObject<AllocaVarStmtNode>();
    node->var = std::move(PrimVar(name, datatype));
    node->init_value = std::move(init_value);
    data_ = std::move(node);
}
```

### AssignStmtNode/AssignStmt

赋值语句：

```cpp
class AssignStmtNode : public StmtNode {
public:
    static constexpr const std::string_view NAME = "AssignStmt";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(AssignStmtNode, StmtNode);

    Expr u;  // 左值（被赋值的变量）
    Expr v;  // 右值（赋值表达式）
};

class AssignStmt : public Stmt {
public:
    DEFINE_NODE_CLASS(AssignStmt, Stmt, AssignStmtNode);

    explicit AssignStmt(Expr u, Expr v);
};
```

### ReturnStmtNode/ReturnStmt

返回语句：

```cpp
class ReturnStmtNode : public StmtNode {
public:
    static constexpr const std::string_view NAME = "ReturnStmt";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(ReturnStmtNode, StmtNode);

    Expr value;  // 返回值表达式
};

class ReturnStmt : public Stmt {
public:
    DEFINE_NODE_CLASS(ReturnStmt, Stmt, ReturnStmtNode);

    explicit ReturnStmt(Expr value);
};
```

### EvaluateNode/Evaluate

求值语句（计算表达式但不使用结果）：

```cpp
class EvaluateNode : public StmtNode {
public:
    static constexpr const std::string_view NAME = "Evaluate";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(EvaluateNode, StmtNode);

    PrimExpr value;  // 要求值的表达式
};

class Evaluate : public Stmt {
public:
    DEFINE_NODE_CLASS(Evaluate, Stmt, EvaluateNode);

    explicit Evaluate(PrimExpr value);
};
```

### SeqStmtNode/SeqStmt

顺序语句（语句序列）：

```cpp
class SeqStmtNode : public StmtNode {
public:
    static constexpr const std::string_view NAME = "SeqStmt";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(SeqStmtNode, StmtNode);

    Array<Stmt> s;  // 语句列表
};

class SeqStmt : public Stmt {
public:
    DEFINE_NODE_CLASS(SeqStmt, Stmt, SeqStmtNode);

    explicit SeqStmt(Array<Stmt> s);
};
```

## Type

### TypeNode/Type

类型系统的基类：

```cpp
class TypeNode : public object_t {
public:
    static constexpr const std::string_view NAME = "Type";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(TypeNode, object_t);
};

class Type : public object_r {
public:
    DEFINE_NODE_CLASS(Type, object_r, TypeNode);
};
```

### PrimTypeNode/PrimType

原始类型（基于 DataType）：

```cpp
class PrimTypeNode : public TypeNode {
public:
    static constexpr const std::string_view NAME = "PrimType";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(PrimTypeNode, TypeNode);

    DataType datatype;  // 底层数据类型
};

class PrimType : public Type {
public:
    DEFINE_NODE_CLASS(PrimType, Type, PrimTypeNode);

    explicit PrimType(DataType datatype);
};
```

### TypeVarNode/TypeVar

类型变量（用于泛型）：

```cpp
class TypeVarNode : public TypeNode {
public:
    static constexpr const std::string_view NAME = "TypeVar";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(TypeVarNode, TypeNode);

    std::string name;  // 类型变量名
};

class TypeVar : public Type {
public:
    DEFINE_NODE_CLASS(TypeVar, Type, TypeVarNode);

    explicit TypeVar(std::string name);
};
```

### GlobalTypeVarNode/GlobalTypeVar

全局类型变量：

```cpp
class GlobalTypeVarNode : public TypeNode {
public:
    static constexpr const std::string_view NAME = "GlobalTypeVar";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(GlobalTypeVarNode, TypeNode);

    std::string var_name;  // 全局类型变量名
};

class GlobalTypeVar : public Type {
public:
    DEFINE_NODE_CLASS(GlobalTypeVar, Type, GlobalTypeVarNode);

    explicit GlobalTypeVar(std::string name);
};
```

### 类型转换函数

```cpp
// 从 Type 获取 DataType
DataType GetRuntimeDataType(const Type& t);

// 检查是否为运行时数据类型
bool IsRuntimeDataType(const Type& t);
```

**实现细节**:
```cpp
DataType GetRuntimeDataType(const Type& t) {
    if (auto* n = t.As<PrimTypeNode>()) {
        return n->datatype;
    }
    return DataType::Handle();  // 非原始类型返回句柄类型
}

bool IsRuntimeDataType(const Type& t) {
    if (auto* n = t.As<PrimTypeNode>()) {
        return true;
    } else {
        return false;
    }
}
```

## Function

### DictAttrsNode/DictAttrs

函数属性字典：

```cpp
class DictAttrsNode : public object_t {
public:
    static constexpr const std::string_view NAME = "DictAttrs";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(DictAttrsNode, object_t);

    Map<Str, Object> dict;  // 属性映射

    DictAttrsNode() : dict(Map<Str,Object>{}) {}
};

class DictAttrs : public object_r {
public:
    DEFINE_NODE_CLASS(DictAttrs, object_r, DictAttrsNode);
    DEFINE_COW_METHOD(DictAttrsNode);  // 写时复制支持

    explicit DictAttrs(Map<Str, Object> dict);
};
```

注意 Str 和 Map 用上了。

### FuncNode/Func

所有函数的基类：

```cpp
class FuncNode : public AstExprNode {
public:
    static constexpr const std::string_view NAME = "BaseFunc";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(BaseFuncNode, AstExprNode);

    DictAttrs attrs;  // 函数属性

    // 获取函数属性
    template <typename T>
    T GetAttr(const Str& attr_key,
              const T& default_value = T()) const {
        static_assert(std::is_base_of<object_r, T>::value,
                     "Can only call GetAttr with object_r types.");
        
        if (attrs.get() == nullptr) {
            return default_value;
        }
        
        auto it = attrs->dict.find(attr_key);
        if (it != attrs->dict.end()) {
            auto pair = *it;  
            if (auto* value = Downcast<T>(pair.second)) {
                return *value;
            }   
        } 
        
        return default_value;
    }
};

class Func : public AstExpr {
public:
    DEFINE_NODE_CLASS(Func, AstExpr, FuncNode);
};
```

### WithAttr

```cpp
// 为函数添加属性
template <typename TFunc,
          typename = typename std::enable_if<std::is_base_of<BaseFunc, TFunc>::value>::type>
inline TFunc WithAttr(TFunc func, Str attr_key, object_r attr_value) {
    using TNode = typename TFunc::ContainerType;
    
    TNode* node = func.CopyOnWrite();  // 写时复制
    
    if (node->attrs.none()) {
        Map<Str, Object> dict;
        dict.set(attr_key, attr_value);
        node->attrs = DictAttrs(std::move(dict));
    } else {
        node->attrs->dict.set(attr_key, attr_value);
    }

    return func;
}
```

### PrimFuncNode/PrimFunc

原始函数（可直接执行的函数）：

```cpp
class PrimFuncNode : public FuncNode {
public:
    static constexpr const std::string_view NAME = "PrimFunc";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(PrimFuncNode, FuncNode);

    Array<PrimVar> gs;     // 参数列表
    Array<PrimExpr> fs;    // 默认参数值
    Stmt body;             // 函数体
    Type rt;               // 返回类型
};

class PrimFunc : public Func {
public:
    DEFINE_NODE_CLASS(PrimFunc, Func, PrimFuncNode);
    DEFINE_COW_METHOD(PrimFuncNode);  // 写时复制支持

    explicit PrimFunc(Array<PrimVar> gs,
             Array<PrimExpr> fs,
             Stmt body,
             Type rt);
};
```

### AstFuncNode/AstFunc

AST 函数（支持模板参数的函数）：

```cpp
class AstFuncNode : public FuncNode {
public:
    static constexpr const std::string_view NAME = "AstFunc";
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    DEFINE_TYPEINDEX(AstFuncNode, FuncNode);

    Array<Expr> gs;    // 参数列表
    Array<Expr> fs;    // 默认参数值
    Stmt body;             // 函数体
    Type rt;               // 返回类型
    Array<TypeVar> ts;     // 模板参数列表（类似 C++ 的模板参数）
};

class AstFunc : public Func {
public:
    DEFINE_NODE_CLASS(AstFunc, Func, AstFuncNode);
    DEFINE_COW_METHOD(AstFuncNode);  // 写时复制支持

    explicit AstFunc(Array<Expr> gs, Array<Expr> fs,
                      Stmt body, Type rt, Array<TypeVar> ts);
};
```

## Module

### IRModuleNode/IRModule

中间表示模块（程序的顶层容器）：

```cpp
class IRModuleNode : public object_t {
public:
    static constexpr const std::string_view NAME = "IRModule";
    static constexpr const int32_t INDEX = TypeIndex::Module;
    DEFINE_TYPEINDEX(IRModuleNode, object_t);

    Map<GlobalVar, Func> func_;        // 函数映射
    Map<GlobalTypeVar, ClassType> class_;  // 类型映射

    void VisitAttrs(AttrVisitor* v) override;

    // 添加函数
    void Insert(const GlobalVar& var, const Func& func) {
        func_[var] = func;
    }

private:
    std::unordered_map<std::string, GlobalVar> vars_;           // 函数名映射
    std::unordered_map<std::string, GlobalTypeVar> class_vars_; // 类型名映射
    friend class IRModule;
};

class IRModule : public object_r {
public:
    DEFINE_COW_METHOD(IRModuleNode);

    explicit IRModule(object_p<object_t> n) noexcept 
        : object_r(std::move(n)) {} 
    
    IRModuleNode* operator->() const noexcept {
        auto* p = get_mutable();
        return static_cast<IRModuleNode*>(p);
    }

    explicit IRModule(Map<GlobalVar,Func> funcions,
                      Map<GlobalTypeVar, ClassType> ts = {});

    IRModule() : IRModule(Map<GlobalVar,Func>({})) {}

    // 从表达式创建模块
    static IRModule From(const AstExpr& e,
                         const Map<GlobalVar, Func>& funcs = {},
                         const Map<GlobalTypeVar, ClassType>& types = {});

    using ContainerType = IRModuleNode;
};
```

**模块构造实现**:
```cpp
IRModule::IRModule(Map<GlobalVar, Func> functions,
                   Map<GlobalTypeVar, ClassType> ts) {
    auto node = MakeObject<IRModuleNode>();
    node->func_ = std::move(functions);
    node->class_ = std::move(ts);
    node->vars_ = {};
    node->class_vars_ = {};

    // 建立名称到变量的映射
    for (const auto& p : node->func_) {
        node->vars_.emplace(p.first->var_name, p.first);
    }

    for (const auto& p : node->class_) {
        node->class_vars_.emplace(p.first->var_name, p.first);
    }

    data_ = std::move(node);
}
```

**从表达式创建模块**:
```cpp
IRModule IRModule::From(const AstExpr& expr,
                        const Map<GlobalVar, Func>& funcs, 
                        const Map<GlobalTypeVar, ClassType>& clss) {
    auto m = IRModule(funcs, clss);
    Func func;
    std::string name = "main";

    // 检查表达式是否为函数
    if (auto* func_node = expr.As<FuncNode>()) {
        func = RTcast<Func>(func_node);
    }

    auto main = GlobalVar(name);
    m->Insert(main, func);

    return m;
}
```

## 举例

### 表达式

```cpp
// 创建变量
auto x = PrimVar("x", DataType::Int(32));
auto y = PrimVar("y", DataType::Int(32));

// 创建字面量
auto val1 = IntImm(DataType::Int(32), 42);
auto val2 = IntImm(DataType::Int(32), 10);

// 方式1：直接使用 AST 节点
auto add_expr = PrimAdd(x, val1);      // x + 42
auto mul_expr = PrimMul(y, val2);      // y * 10
auto complex_expr = PrimAdd(add_expr, mul_expr);  // (x + 42) + (y * 10)

// 方式2：使用运算符函数
auto add_expr2 = op_add(x, val1);      // x + 42
auto mul_expr2 = op_mul(y, val2);      // y * 10
auto complex_expr2 = op_add(add_expr2, mul_expr2);

// 方式3：使用运算符对象
auto add_op = add();
auto mul_op = mul();
auto call_add = PrimCall(DataType::Int(32), add_op, {x, val1});
auto call_mul = PrimCall(DataType::Int(32), mul_op, {y, val2});

// 条件表达式
auto condition = Bool(true);
auto conditional = op_if_then_else(condition, x, y);

// 创建函数调用
Array<PrimExpr> args = {x, y};
auto call_expr = PrimCall(DataType::Int(32), GlobalVar("my_func"), args);
```

### 语句

```cpp
// 变量声明
auto decl_x = AllocaVarStmt("x", DataType::Int(32), IntImm(DataType::Int(32), 0));

// 赋值语句
auto assign = AssignStmt(x, add_expr);

// 返回语句
auto ret = ReturnStmt(complex_expr);

// 顺序语句
Array<Stmt> stmts = {decl_x, assign, ret};
auto seq = SeqStmt(stmts);
```

### 函数

```cpp
// 函数参数
Array<PrimVar> params = {
    PrimVar("a", DataType::Int(32)),
    PrimVar("b", DataType::Int(32))
};

// 默认参数值
Array<PrimExpr> defaults = {
    IntImm(DataType::Int(32), 0)  // 第二个参数的默认值
};

// 函数体
auto body = SeqStmt({
    ExprStmt(PrimAdd(PrimVar("a", DataType::Int(32)), 
                     PrimVar("b", DataType::Int(32)))),
    ReturnStmt(PrimVar("a", DataType::Int(32)))
});

// 创建函数
auto func = PrimFunc(params, defaults, body, PrimType(DataType::Int(32)));

// 添加属性
func = WithAttr(func, Str("inline"), Str("true"));
```

### 模块

```cpp
// 创建全局函数
Map<GlobalVar, Func> functions;
functions.set(GlobalVar("add"), func);
functions.set(GlobalVar("multiply"), another_func);

// 创建类型定义
Map<GlobalTypeVar, ClassType> types;
types.set(GlobalTypeVar("MyClass"), ClassType());

// 创建模块
auto module = IRModule(functions, types);

// 或从主函数创建
auto main_module = IRModule::From(func);
```







