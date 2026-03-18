# Function

## 概述

函数注册系统是本项目的核心基础设施，它通过一套精巧的C++模板元编程技术，实现了**任意签名的C++函数都能通过统一接口调用**的能力。这个系统解决了编译器框架中的一个根本问题：怎么能让Python前端能够统一调用具有不同签名的C++函数。

**关键点**
1. **类型擦除**: 将不同签名的函数统一为 `std::function<McValue(Parameters)>`
2. **编译时参数展开**: 使用 `std::index_sequence` 实现零开销的参数解包
3. **运行时类型转换**: 使用 `constexpr if` 实现类型安全的参数转换
4. **全局函数管理**: 线程安全的单例模式管理所有注册函数
5. **宏化注册**: 简化函数注册流程，支持自动类型推导

**示意图**：
```
┌─────────────────────────────────────────────────────────────┐
│           不同签名的 C++ 函数                               │
├─────────────────────────────────────────────────────────────┤
│ int add(int, int)                                          │
│ string concat(string, string)                              │
│ PrimExpr make_var(string, DataType)                       │
│ Array<Stmt> parse_block(Array<AstExpr>)                   │
└─────────────────┬───────────────────────────────────────────┘
                  │
         ┌────────▼────────┐
         │   类型擦除层    │
         │ FunctionWrapper │
         └────────┬────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│                统一函数接口                                 │
│         std::function<McValue(Parameters)>                │
└─────────────────────────────────────────────────────────────┘
```

### 数据结构

### Function

整个系统的基础是统一的函数类型：

```cpp
using Function = std::function<McValue(Parameters)>;
```

这个类型定义看似简单，但它是整个类型擦除系统的核心：
- **输入**: `Parameters` - 统一的参数容器，持有 `Any` 类型的参数数组
- **输出**: `McValue` - 统一的返回值类型，支持所有MC类型
- **调用语义**: 所有函数都遵循相同的调用约定

### FunctionRegistry

全局函数注册表的核心实现：

```cpp
class FunctionRegistry {
public:
    struct FunctionData {
        Function func_;              // 包装后的统一函数
        std::string_view name_;      // 函数名称（不拥有字符串）
        std::string doc_;           // 文档字符串（可选）

        template <typename F>
        FunctionData& SetBody(F&& func);  // 设置函数体
        
        FunctionData& set(Function func) {
            func_ = std::move(func);
            return *this;
        }
    };

    // 静态方法 - 单例模式
    static FunctionData& Register(std::string_view name);
    template<typename F>
    static FunctionData& SetFunction(std::string_view name, F&& func);
    static Function* Get(std::string_view name);
    static bool Remove(std::string_view name);
    static std::vector<std::string_view> ListNames();

private:
    static FunctionRegistry& Instance();
    
    std::mutex mutex_;                                           // 线程安全保护
    std::unordered_map<std::string_view, FunctionData> functions_; // 函数存储
};
```

**设计特点**:
- **单例模式**: 全局唯一的函数注册表
- **线程安全**: 使用互斥锁保护并发访问
- **懒初始化**: 函数在首次注册时才创建
- **名称视图**: 使用 `string_view` 避免不必要的字符串拷贝



## 类型擦除

### FunctionWrapper

`FunctionWrapper` 是整个系统最精巧的部分，它使用C++模板元编程实现类型擦除：

```cpp
class FunctionWrapper {
public:
    using FuncType = std::function<McValue(Parameters)>;
    
    // 为普通函数指针创建包装器
    template<typename R, typename... Args>
    static FuncType Create(R (*func)(Args...)) {
        return [func](Parameters ps) -> McValue {
            if (ps.size() < sizeof...(Args)) {
                throw std::runtime_error("Not enough parameters");
            }
            return CallImpl(func, ps, std::make_index_sequence<sizeof...(Args)>{});
        };
    }

    // 为函数对象（lambda、std::function等）创建包装器
    template<typename F, typename = std::enable_if_t<std::is_class_v<std::remove_reference_t<F>>>>
    static FuncType Create(F&& f) {
        return [f = std::forward<F>(f)](Parameters ps) -> McValue {
            return CallLambda(f, ps);
        };
    }

private:
    // 核心：编译时参数展开
    template<typename R, typename... Args, size_t... Is>
    static McValue CallImpl(R (*func)(Args...), 
                          const Parameters& ps, 
                          std::index_sequence<Is...>) {
        return McValue(func(ConvertArg<Args>(ps[Is])...));
    }
    
    // 处理函数对象的调用
    template<typename F>
    static McValue CallLambda(F& f, const Parameters& ps);
    
    // 类型转换核心
    template<typename T>
    static auto ConvertArg(const Any& value);
};
```

### 模板元编程

#### 1. `std::index_sequence` 的魔法

`std::index_sequence` 是C++14引入的编译时整数序列：

```cpp
// 编译时生成：std::index_sequence<0, 1, 2>
std::make_index_sequence<3>{}

// 在模板参数包展开中使用
template<size_t... Is>
static McValue CallImpl(R (*func)(Args...), 
                      const Parameters& ps, 
                      std::index_sequence<Is...>) {
    // Is... 在编译时展开为 0, 1, 2, ...
    return McValue(func(ConvertArg<Args>(ps[Is])...));
    //                  ↑
    //                  这里展开为：
    //                  ConvertArg<Arg0>(ps[0]),
    //                  ConvertArg<Arg1>(ps[1]),
    //                  ConvertArg<Arg2>(ps[2]), ...
}
```

**展开过程示例**：

```cpp
// 原始函数：PrimExpr MakeAdd(PrimExpr lhs, PrimExpr rhs)
// sizeof...(Args) = 2
// std::make_index_sequence<2>{} 生成 std::index_sequence<0, 1>{}

// 模板实例化后的实际代码：
static McValue CallImpl(PrimExpr (*func)(PrimExpr, PrimExpr), 
                      const Parameters& ps, 
                      std::index_sequence<0, 1>) {
    return McValue(func(ConvertArg<PrimExpr>(ps[0]),
                       ConvertArg<PrimExpr>(ps[1])));
}
```

#### 2. SFINAE 和 `std::enable_if_t`

```cpp
template<typename F, typename = std::enable_if_t<std::is_class_v<std::remove_reference_t<F>>>>
static FuncType Create(F&& f) {
    // 只有当 F 是类类型时，这个重载才参与重载决议
}
```

**SFINAE (Substitution Failure Is Not An Error) 工作原理**：

1. **模板替换**: 编译器尝试用实际类型替换模板参数
2. **失败忽略**: 如果替换失败，该模板重载被忽略（而不是编译错误）
3. **重载选择**: 编译器从剩余的有效重载中选择最佳匹配

**类型检查链**：
```cpp
F = lambda表达式
    ↓
std::remove_reference_t<F> = 移除引用，得到lambda类型
    ↓
std::is_class_v<...> = true（lambda是类类型）
    ↓
std::enable_if_t<true> = void（启用这个重载）

F = 函数指针
    ↓
std::remove_reference_t<F> = 函数指针类型
    ↓
std::is_class_v<...> = false（函数指针不是类类型）
    ↓
std::enable_if_t<false> = 替换失败，重载被排除
```

### 函数特征

系统使用 `FunctionTraits` 来提取函数类型信息：

```cpp
// 基础模板
template<typename F>
struct FunctionTraits;

// 特化：普通函数
template<typename R, typename... Args>
struct FunctionTraits<R(Args...)> {
    static constexpr size_t arity = sizeof...(Args);     // 参数个数
    using result_type = R;                               // 返回类型
    
    template<size_t I>
    struct arg {
        using type = typename std::tuple_element<I, std::tuple<Args...>>::type;
    };
};

// 特化：成员函数指针（const）
template<typename C, typename R, typename... Args>
struct FunctionTraits<R(C::*)(Args...) const> {
    static constexpr size_t arity = sizeof...(Args);
    using result_type = R;
    
    template<size_t I>
    struct arg {
        using type = typename std::remove_cv_t<
            typename std::remove_reference_t<
                typename std::tuple_element<I, std::tuple<Args...>>::type
            >
        >;
    };
};

// 通用适配器：lambda表达式
template<typename F>
struct FunctionTraits : public FunctionTraits<decltype(&F::operator())> {};
```

**使用示例**：

```cpp
// 对于 lambda: [](int x, string y) -> bool { ... }
using traits = FunctionTraits<decltype(lambda)>;

static_assert(traits::arity == 2);                    // 参数个数
static_assert(std::is_same_v<traits::result_type, bool>); // 返回类型
static_assert(std::is_same_v<traits::arg<0>::type, int>); // 第0个参数类型
static_assert(std::is_same_v<traits::arg<1>::type, string>); // 第1个参数类型
```

### Lambda调用

```cpp
template<typename F>
static McValue CallLambda(F& f, const Parameters& ps) {
    using traits = FunctionTraits<F>;
    if (ps.size() < traits::arity) {
        throw std::runtime_error("Not enough parameters");
    }
    return CallLambdaImpl(f, ps, std::make_index_sequence<traits::arity>{});
}

template<typename F, size_t... Is>
static McValue CallLambdaImpl(F& f, const Parameters& ps, std::index_sequence<Is...>) {
    using traits = FunctionTraits<F>;
    return f(ConvertArg<typename traits::template arg<Is>::type>(ps[Is])...);
}
```

**关键点解析**：
1. **`typename traits::template arg<Is>::type`**: 复杂的依赖名称查找
   - `traits::arg<Is>` 是依赖模板，需要 `template` 关键字
   - `::type` 是依赖类型，需要 `typename` 关键字
2. **编译时展开**: `Is...` 展开为参数索引序列
3. **类型安全**: 每个参数都经过精确的类型转换

## 类型转换

`ConvertArg` 是连接类型擦除世界和强类型世界的桥梁：

```cpp
template<typename T>
static auto ConvertArg(const Any& value) {
    using clean_type = std::remove_cv_t<std::remove_reference_t<T>>;
    
    if constexpr (std::is_same_v<clean_type, McValue>) {
        return McValue(value);
    }
    else if constexpr (std::is_same_v<clean_type, Object>) {
        if (!value.IsObject()) {
            throw std::runtime_error("Expected object type");
        }
        return AsObject(value);
    } 
    else if constexpr (std::is_base_of_v<object_r, clean_type>) {
        if (!value.IsObject()) {
            throw std::runtime_error("Expected object type");
        }
        return value.As<clean_type>();
    } 
    else if constexpr (std::is_integral_v<clean_type>) {
        if (!value.IsInt()) {
            throw std::runtime_error("Expected integer type");
        }
        return static_cast<clean_type>(value.As<int64_t>());
    }
    else if constexpr (std::is_floating_point_v<clean_type>) {
        if (!value.IsFloat() && !value.IsInt()) {
            throw std::runtime_error("Expected floating point type");
        }
        return static_cast<clean_type>(value.As<double>());
    }
    else if constexpr (std::is_same_v<clean_type, std::string>) {
        if (!value.IsStr()) {
            throw std::runtime_error("Expected string type");
        }
        return std::string(value.As<const char*>());
    }
    else if constexpr (std::is_same_v<clean_type, DataType>) {
        if (!value.IsDataType()) {
            throw std::runtime_error("Expected DataType");
        }
        return value.As<DataType>();
    }
    else {
        throw std::runtime_error("Unsupported parameter type");
    }
}
```


C++17 的 `constexpr if` 提供了编译时条件分支：

```cpp
// 传统 C++14 方式（复杂的SFINAE）
template<typename T>
typename std::enable_if_t<std::is_integral_v<T>, T>
ConvertArg_Old(const Any& value) { /* 整数转换 */ }

template<typename T>  
typename std::enable_if_t<std::is_floating_point_v<T>, T>
ConvertArg_Old(const Any& value) { /* 浮点转换 */ }

// C++17 方式（清晰的条件分支）
template<typename T>
auto ConvertArg(const Any& value) {
    if constexpr (std::is_integral_v<T>) {
        // 整数转换 - 只有T是整数时才编译这个分支
    } else if constexpr (std::is_floating_point_v<T>) {
        // 浮点转换 - 只有T是浮点时才编译这个分支
    }
}
```

**`constexpr if` 的工作原理**：
1. **编译时求值**: 条件在编译时计算
2. **分支剪枝**: 未选中的分支不会被编译
3. **类型安全**: 每个分支可以使用不同的类型操作
4. **性能优化**: 运行时没有条件判断开销


```cpp
using clean_type = std::remove_cv_t<std::remove_reference_t<T>>;
```

这个类型清理链确保转换的鲁棒性：

```cpp
// 原始类型：const PrimExpr&
//     ↓
// std::remove_reference_t<const PrimExpr&> = const PrimExpr
//     ↓  
// std::remove_cv_t<const PrimExpr> = PrimExpr
//     ↓
// clean_type = PrimExpr
```

**为什么需要类型清理？**
- 函数参数可能有各种修饰符：`const T&`, `T&&`, `volatile T` 等
- 转换逻辑需要基于纯净的类型进行判断
- 确保模板特化能够正确匹配

## 注册机制

### 函数注册


FunctionRegistry提供了多种注册方法：

```cpp
// 方法1：分步注册
static FunctionData& Register(std::string_view name) {
    auto& instance = Instance();
    std::lock_guard<std::mutex> lock(instance.mutex_);

    auto it = instance.functions_.find(name);
    if (it != instance.functions_.end()) {
        throw std::runtime_error(
                std::string("Function already registered: ")+std::string(name));
        return it->second;
    }

    return instance.functions_.emplace(
                name, FunctionData{nullptr, name}).first->second;
}

// 方法2：一步注册
template<typename F>
static FunctionData& SetFunction(std::string_view name, F&& func) {
    auto& data = Register(name);
    data.func_ = FunctionWrapper::Create(std::forward<F>(func));
    return data;
}
```

### 链式调用

```cpp
template <typename F>
FunctionData& FunctionData::SetBody(F&& func) {
    return set(FunctionWrapper::Create(std::forward<F>(func)));
}
```

**链式调用示例**：
```cpp
auto& data = FunctionRegistry::Register("ast._OpAdd")
    .SetBody(MakeAdd)                    // 设置函数体
    .set_doc("Create addition expression"); // 设置文档（假设有这个方法）
```

### 注册宏

为了简化注册过程，系统提供了便利宏：

```cpp
#define STR_CONCAT_IMPL(x, y) x##y
#define STR_CONCAT(x, y) STR_CONCAT_IMPL(x, y)

#define REGISTER_GLOBAL(Name) \
    static auto& STR_CONCAT(__reg_global_, __COUNTER__) = \
        mc::runtime::FunctionRegistry::Register(Name)

#define REGISTER_FUNCTION(Name, Func) \
    static auto& STR_CONCAT(__reg_global_, __COUNTER__) = \
        mc::runtime::FunctionRegistry::SetFunction(Name, Func)
```

**宏展开示例**：

```cpp
// 原始代码
REGISTER_FUNCTION("ast._OpAdd", MakeAdd);

// 展开后（假设 __COUNTER__ = 123）
static auto& __reg_global_123 = 
    mc::runtime::FunctionRegistry::SetFunction("ast._OpAdd", MakeAdd);
```

**宏设计的精妙之处**：
1. **静态初始化**: 使用静态变量确保在main函数前完成注册
2. **唯一命名**: `__COUNTER__` 宏确保变量名不冲突
3. **自动推导**: 模板自动推导函数类型，无需手动指定
4. **RAII**: 变量的构造就是注册过程

### 全过程

让我们跟踪一个完整的函数从注册到调用的全过程：

```cpp
// 1. 定义C++函数
PrimExpr MakeAdd(PrimExpr lhs, PrimExpr rhs) {
    auto node = MakeObject<PrimAddNode>();
    node->a = std::move(lhs);
    node->b = std::move(rhs);
    node->datatype = lhs->datatype;  // 继承左操作数类型
    return PrimExpr(node);
}

// 2. 注册函数（程序启动时自动执行）
REGISTER_FUNCTION("ast._OpAdd", MakeAdd);

// 宏展开为：
static auto& __reg_global_42 = 
    FunctionRegistry::SetFunction("ast._OpAdd", MakeAdd);

// 3. SetFunction的执行流程
template<typename F>
static FunctionData& SetFunction(std::string_view name, F&& func) {
    // 3.1 注册函数名
    auto& data = Register("ast._OpAdd");
    
    // 3.2 创建包装器
    data.func_ = FunctionWrapper::Create(MakeAdd);
    
    // 3.3 包装器创建过程
    return [MakeAdd](Parameters ps) -> McValue {
        if (ps.size() < 2) {
            throw std::runtime_error("Not enough parameters");
        }
        return CallImpl(MakeAdd, ps, std::index_sequence<0, 1>{});
    };
    
    return data;
}

// 4. Python调用（通过FFI系统）
// Python: result = op_add_(expr1, expr2)

// 5. C API层接收调用
int FuncCall_PYTHON_C_API(FunctionHandle func, Value* vs, int n, Value* r) {
    // 5.1 转换参数
    std::vector<McView> gs;
    for (int i = 0; i < n; ++i) {
        gs.push_back(McView(vs[i]));
    }
    
    // 5.2 调用统一接口
    auto* function = static_cast<const Function*>(func);
    McValue rv = (*function)(Parameters(gs.data(), gs.size()));
    
    // 实际上调用的是包装后的lambda：
    // [MakeAdd](Parameters ps) -> McValue { ... }
}

// 6. 包装器内部执行
McValue lambda_body(Parameters ps) {
    // 6.1 编译时参数展开
    return CallImpl(MakeAdd, ps, std::index_sequence<0, 1>{});
    
    // 6.2 CallImpl展开为：
    return McValue(MakeAdd(ConvertArg<PrimExpr>(ps[0]),
                          ConvertArg<PrimExpr>(ps[1])));
    
    // 6.3 ConvertArg执行类型转换
    PrimExpr arg0 = ps[0].As<PrimExpr>();  // 运行时类型检查和转换
    PrimExpr arg1 = ps[1].As<PrimExpr>();
    
    // 6.4 调用原始C++函数
    return McValue(MakeAdd(arg0, arg1));
}
```

## 举例

### 基础函数


```cpp
// 1. 定义各种签名的函数
int simple_add(int a, int b) {
    return a + b;
}

std::string string_concat(const std::string& a, const std::string& b) {
    return a + b;
}

PrimExpr make_variable(const std::string& name, DataType dtype) {
    return PrimVar(name, dtype);
}

// 2. 注册函数
REGISTER_FUNCTION("math.add", simple_add);
REGISTER_FUNCTION("string.concat", string_concat);
REGISTER_FUNCTION("ast.make_var", make_variable);

// 3. Python端调用 (FFI调用系统)
// add_func = mc_api.GetGlobal("math.add", True)
// result = add_func(3, 4)  # 返回 7
//
// concat_func = mc_api.GetGlobal("string.concat", True)
// text = concat_func("hello", " world")  # 返回 "hello world"
//
// var_func = mc_api.GetGlobal("ast.make_var", True)
// var = var_func("x", DataType.Int(32))  # 返回 PrimVar 对象
```

### Lambda表达式

```cpp
// 注册Lambda表达式
REGISTER_FUNCTION("math.multiply", [](int a, int b) -> int {
    return a * b;
});

// 注册带捕获的Lambda
int global_offset = 100;
REGISTER_FUNCTION("math.add_offset", [global_offset](int value) -> int {
    return value + global_offset;
});

// 注册std::function
std::function<bool(int)> is_even = [](int n) { return n % 2 == 0; };
REGISTER_FUNCTION("math.is_even", is_even);
```

### Ast类型

```cpp
// 处理容器类型
Array<PrimExpr> make_expr_array(const std::vector<std::string>& names) {
    Array<PrimExpr> result;
    for (const auto& name : names) {
        result.push_back(PrimVar(name, DataType::Int(32)));
    }
    return result;
}
REGISTER_FUNCTION("ast.make_expr_array", make_expr_array);

// 处理可选参数（通过重载）
PrimExpr make_const_int(int64_t value) {
    return IntImm(DataType::Int(64), value);
}

PrimExpr make_const_int_with_type(int64_t value, DataType dtype) {
    return IntImm(dtype, value);
}

REGISTER_FUNCTION("ast.make_const", make_const_int);
REGISTER_FUNCTION("ast.make_const_typed", make_const_int_with_type);
```

TODO：REGISTER_GLOBAL


### 函数查找

```cpp
// 运行时查找和调用函数
void call_function_by_name(const std::string& func_name, 
                          const std::vector<McValue>& args) {
    // 1. 查找函数
    auto* func = FunctionRegistry::Get(func_name);
    if (!func) {
        throw std::runtime_error("Function not found: " + func_name);
    }
    
    // 2. 准备参数
    std::vector<McView> views;
    for (const auto& arg : args) {
        views.push_back(McView(arg));
    }
    Parameters params(views.data(), views.size());
    
    // 3. 调用函数
    McValue result = (*func)(params);
    
    std::cout << "Result: " << result.As<std::string>() << std::endl;
}

// 使用示例
call_function_by_name("string.concat", {
    McValue("Hello"),
    McValue(" World")
});
```