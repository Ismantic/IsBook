# Runtime Object 

## 概述

运行时对象系统提供了一个基于引用计数的对象基础设施，具有运行时类型检查和多态性。它由三个主要组件和关键宏定义组成：

**核心组建：**
- `object_t`：运行时对象的基类
- `object_p<T>`：用于自动内存管理的智能指针
- `object_r`：具有运行时类型检查的类型擦除对象引用

**关键宏定义：**
- `DEFINE_TYPEINDEX`：定义类型索引和运行时类型系统
- `REGISTER_TYPEINDEX`：注册类型索引
- `DEFINE_COW_METHOD`：定义写时复制方法
- `MakeObject`：对象创建函数

## 核心组件

### object_t

`object_t`是系统中所有运行时对象的基类，提供引用计数和类型系统支持。


```cpp
class object_t {
public:
    virtual ~object_t();
    virtual int32_t Index() const { return t_; }
    virtual std::string Name() const;
    virtual void VisitAttrs(AttrVisitor* visitor) {}
    
    // 引用计数
    void IncCounter() noexcept { ++count_; }
    void DecCounter() noexcept { if (--count_ == 0) delete this; }
    int32_t UseCount() const noexcept { return count_; }
    
    // 类型检查
    bool IsFrom(int32_t parent_type) const;
    template<typename T>
    bool IsType() const noexcept { return IsFrom(T::RuntimeTypeIndex()); }
    
    // 类型系统常量
    static constexpr const int32_t INDEX = TypeIndex::Object;
    static constexpr const std::string_view NAME = "Object";
    static int32_t _GetOrAllocRuntimeTypeIndex() { return TypeIndex::Object; }
    static int32_t RuntimeTypeIndex() { return TypeIndex::Object; }

protected:
    int32_t t_{0};                    // 类型索引
    std::atomic<int32_t> count_{0};   // 引用计数
};
```

**特性说明：**
- **原子引用计数**: 线程安全的自动内存管理
- **运行时类型系统**: 每个对象都有类型索引 `t_`
- **类型层次检查**: `IsFrom()` 检查继承关系
- **访问者模式**: 支持 `VisitAttrs()` 进行对象遍历

### object_p<T>

`object_p<T>` 提供自动引用计数和类型安全访问的智能指针模板。

```cpp
template<typename T>
class object_p {
public:
    object_p() noexcept = default;
    explicit object_p(T* p) noexcept : data_{p} {
        if (data_) data_->IncCounter();
    }
    ~object_p() {
        if (data_) data_->DecCounter();
    }
    
    // 拷贝构造（含多态支持）
    object_p(const object_p& other) noexcept;
    template<typename U>
    object_p(const object_p<U>& other) noexcept;  // 派生类到基类
    
    // 赋值操作
    object_p& operator=(const object_p& other) noexcept;
    object_p& operator=(object_p&& other) noexcept;
    
    // 访问操作符
    T* get() const noexcept { return data_; }
    T* operator->() const noexcept { return data_; }
    T& operator*() const noexcept { return *data_; }
    
    // 状态管理
    void reset() noexcept;
    bool unique() const noexcept { return data_ && data_->UseCount() == 1; }
    explicit operator bool() const noexcept { return data_ != nullptr; }
    
    // 比较操作
    bool operator==(const object_p<T>& other) const noexcept;
    bool operator!=(std::nullptr_t) const noexcept;

private:
    T* data_{nullptr};
};
```

**特性说明：**
- **RAII**: 构造时增加引用计数，析构时减少引用计数
- **多态支持**: 支持从派生类到基类的隐式转换
- **移动语义**: 高效的所有权转移，避免不必要的引用计数操作
- **空指针安全**: 所有操作都进行空指针检查

### object_r

`object_r` 可以持有任何 `object_t` 派生对象并提供运行时类型检查的类型擦除包装器。

```cpp
class object_r {
public:
    object_r() noexcept = default;
    explicit object_r(object_p<object_t> p) noexcept : data_(std::move(p)) {}
    
    // 访问
    const object_t* get() const noexcept { return data_.get(); }
    object_t* get_mutable() const noexcept { return data_.get(); }
    const object_t* operator->() const noexcept { return get(); }
    
    // 运行时类型转换
    template<typename T>
    const T* As() const noexcept {
        if (data_ && data_->IsType<T>()) {
            return static_cast<const T*>(data_.get());
        }
        return nullptr;
    }
    
    // 状态检查
    bool none() const noexcept { return data_ == nullptr; }
    
    // 比较操作
    bool operator==(const object_r& other) const noexcept;
    bool operator!=(const object_r& other) const noexcept;

protected:
    object_p<object_t> data_;
};
```

**特性说明：**
- **类型擦除**: 能够统一存储不同类型的对象，保留运行时类型信息
- **安全类型转换**: `As<T>()` 先检查类型再转换，失败返回 `nullptr`
- **常量正确性**: 分离只读和可变访问
- **引用语义**: 内部使用 `object_p<object_t>` 管理生命周期

## 关键宏

### DEFINE_TYPEINDEX

用来定义类型的运行时类型索引。

```cpp
#define DEFINE_TYPEINDEX(TypeName, ParentType)  \
  static int32_t RuntimeTypeIndex() {                                     \
    if (TypeName::INDEX != ::mc::runtime::TypeIndex::Dynamic) {          \
      _GetOrAllocRuntimeTypeIndex();                                      \
      return TypeName::INDEX;                                             \
    }                                                                     \
    return _GetOrAllocRuntimeTypeIndex();                                 \
  }                                                                       \
  static int32_t _GetOrAllocRuntimeTypeIndex() {                         \
    static int32_t t = object_t::GetOrAllocRuntimeTypeIndex(             \
        TypeName::NAME, TypeName::INDEX, ParentType::_GetOrAllocRuntimeTypeIndex()); \
    return t;                                                             \
  }
```

**实现说明：**
1. **静态类型**: 如果 `INDEX != Dynamic`，使用预定义的静态类型索引
2. **动态类型**: 如果 `INDEX == Dynamic`，动态分配类型索引
3. **类型注册**: 向全局类型注册表注册类型信息和继承关系
4. **延迟初始化**: 使用 `static` 变量确保类型索引只分配一次

### REGISTER_TYPEINDEX

用来程序初运行时注册类型索引。

```cpp
#define REGISTER_TYPEINDEX(TypeName)  \
  STR_CONCAT(REG_VAR, __COUNTER__) =  \
    TypeName::_GetOrAllocRuntimeTypeIndex()
```

**展开后的实际代码：**

```cpp
// 假设在某个源文件中调用 REGISTER_TYPEINDEX(MyObject)
// 宏会展开为：
static UNUSED uint32_t __Make_Object_T0 = MyObject::_GetOrAllocRuntimeTypeIndex();
```

**详细说明：**

1. **宏定义分析**：
   - `STR_CONCAT(REG_VAR, __COUNTER__)`: 生成唯一的静态变量名
   - `REG_VAR`: 定义为 `static UNUSED uint32_t __Make_Object_T`
   - `__COUNTER__`: 编译器内置宏，每次使用时递增（0, 1, 2, ...）

2. **静态变量创建**：
   ```cpp
   // runtime_port.h:9
   #define REG_VAR static UNUSED uint32_t __Make_Object_T
   
   // STR_CONCAT 宏展开过程
   #define STR_CONCAT_IMPL(x, y) x##y
   #define STR_CONCAT(x, y) STR_CONCAT_IMPL(x, y)
   
   // 最终结果：创建唯一命名的静态变量
   static UNUSED uint32_t __Make_Object_T0 = TypeName::_GetOrAllocRuntimeTypeIndex();
   static UNUSED uint32_t __Make_Object_T1 = TypeName::_GetOrAllocRuntimeTypeIndex();
   // ...
   ```

3. **静态初始化时机**：
   - 程序启动时，静态变量都会被初始化
   - 触发 `TypeName::_GetOrAllocRuntimeTypeIndex()` 的调用
   - 类型索引在 `main()` 函数执行前就已经注册完成

4. **UNUSED 属性**：
   - 防止编译器因为变量未使用而产生警告
   - 变量的作用仅仅是触发初始化，不需要实际使用

**使用场景：**

- **提前注册**: 避免首次使用时的延迟初始化开销
- **确定性**: 保证类型在任何代码使用前都已注册
- **线程安全**: 静态初始化在单线程环境中完成
- **调试友好**: 类型在程序启动时就已经在全局类型表中

**实际使用示例：**

```cpp
// 在某个 .cpp 文件中
class MyCustomType : public object_t {
public:
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    static constexpr const std::string_view NAME = "MyCustomType";
    DEFINE_TYPEINDEX(MyCustomType, object_t);
};

// 注册类型索引
REGISTER_TYPEINDEX(MyCustomType);  // 展开为: static UNUSED uint32_t __Make_Object_T0 = MyCustomType::_GetOrAllocRuntimeTypeIndex();

// 现在即使在 main() 函数中也可以立即使用类型检查
int main() {
    // 类型已经注册，可以直接使用
    int32_t type_id = GetIndex("MyCustomType");  // 不会抛出异常
    return 0;
}
```

**与延迟初始化的对比：**

```cpp
// 不使用 REGISTER_TYPEINDEX 的情况
auto obj1 = MakeObject<MyType>();  // 第一次使用时才注册类型，可能有延迟

// 使用 REGISTER_TYPEINDEX 的情况  
REGISTER_TYPEINDEX(MyType);        // 程序启动时就注册
auto obj2 = MakeObject<MyType>();  // 直接使用，无延迟
```

**注意事项：**

- 通常在类型定义的源文件中使用，而不是头文件
- 每个类型只需要注册一次
- 适用于需要确定性初始化顺序的场景

### DEFINE_COW_METHOD

类定义写时复制（Copy-on-Write）方法。

```cpp
#define DEFINE_COW_METHOD(TypeName)                                 \
  TypeName* CopyOnWrite() {                                         \
    if (!data_.unique()) {                                          \
      auto n = MakeObject<TypeName>(*(operator->()));               \
      object_p<object_t>(std::move(n)).swap(data_);                 \
    }                                                               \
    return static_cast<TypeName*>(data_.get());                    \
  }
```

**实现说明：**
1. **唯一性检查**: `!data_.unique()` 检查当前对象是否为唯一引用
2. **对象复制**: `MakeObject<TypeName>(*(operator->()))` 创建当前对象的副本
3. **类型转换与交换**: `object_p<object_t>(std::move(n)).swap(data_)` 
4. **返回指针**: 返回可修改的对象指针

**执行流程**：
```cpp
// 执行前：data_ -> 旧对象（引用计数 > 1）
auto n = MakeObject<TypeName>(...);          // n -> 新对象（引用计数 = 1）
object_p<object_t> temp(std::move(n));       // temp -> 新对象，n 为空
temp.swap(data_);                            // 交换指针
// 执行后：data_ -> 新对象（引用计数 = 1）
//        temp -> 旧对象，temp 析构时释放旧对象（引用计数-1）
```

**使用示例：**

```cpp
class MyObject : public object_t {
public:
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    static constexpr const std::string_view NAME = "MyObject";
    DEFINE_TYPEINDEX(MyObject, object_t);
    DEFINE_COW_METHOD(MyObject);  // 生成 CopyOnWrite() 方法
    
    MyObject(int value) : value_(value) {}
    
    void SetValue(int value) {
        CopyOnWrite()->value_ = value;  // 只在需要时复制
    }
    
    int GetValue() const { return value_; }
    
private:
    int value_;
};

// 使用示例
auto obj1 = MakeObject<MyObject>(42);
object_r ref1 = object_r(obj1);
object_r ref2 = ref1;  // 共享同一个对象

// 修改时才触发复制
if (auto* my_obj = ref1.As<MyObject>()) {
    my_obj->SetValue(100);  // 触发 CopyOnWrite，创建独立副本
}
```

这种 COW 机制在需要频繁传递但很少修改的对象中非常有用，可以显著减少不必要的内存复制。

### MakeObject

对象创建的标准方法，确保正确设置类似索引。

```cpp
template <typename T, typename... Gs>
object_p<T> MakeObject(Gs&&... gs) {
    static_assert(std::is_base_of<object_t, T>::value, 
                  "MakeObject can only be used to create object_t");
    T* p = new T(std::forward<Gs>(gs)...);
    p->t_ = T::RuntimeTypeIndex();  // 设置类型索引
    return object_p<T>(p);
}
```

**实现说明：**
- **类型安全**: 编译时检查类型必须继承自 `object_t`
- **完美转发**: 支持任意构造函数参数
- **索引设置**: 自动设置对象的运行时类型索引

## 类型系统

### TypeIndex

预订了一组类似索引：

```cpp
struct TypeIndex {
  enum : int32_t {
    Null = -1,
    Int = -2,
    Float = -3,
    Str = -4,
    Func = -5,
    DataType = -6,
    Object = 0,           // object_t 的基础类型
    Module = 1,
    RuntimeStr = 3,
    RuntimeArray = 5,
    RuntimeMap = 6,
    RuntimeSet = 7,
    RuntimeIterator = 8,
    RuntimeTuple = 9,
    Dynamic = 256         // 动态类型起始索引
  };
};
```

**类型索引分类：**
- **负数索引**: 基础类型（Null、Int、Float等）
- **0-255**: 预定义的运行时类型
- **256+**: 动态分配的类型索引

### TypeContext

TypeContext 是整个对象系统的核心，负责管理全部类型的注册、索引分配和集成关系。

#### 数据结构

```cpp
struct TypeData {
    int32_t t;           // 类型索引（自己的索引）
    int32_t p;           // 父类型索引
    std::string name;    // 类型名称
};

class TypeContext {
private:
    std::mutex mutex_;                                    // 保护并发访问
    std::atomic<uint32_t> counter_{TypeIndex::Dynamic};  // 动态类型计数器（从256开始）
    std::vector<TypeData> vec_;                          // 类型信息数组，索引即类型ID
    std::unordered_map<std::string_view, int32_t> names_; // 类型名称到索引的快速查找
    
public:
    static TypeContext* Global();  // 单例模式
};
```

#### 初始化

```cpp
TypeContext() {
    vec_.resize(TypeIndex::Dynamic, TypeData());  // 预分配256个位置
    vec_[0].name = "Object";                       // 索引0是基础 object_t 类型
    // 索引 1-255 保留给预定义类型（Module、RuntimeStr等）
    // 索引 256+ 用于动态分配
}
```

#### 索引分配

这是系统的核心函数，负责类型注册和索引分配：

```cpp
int32_t GetOrAllocRuntimeTypeIndex(const std::string_view& name, 
                                  int32_t static_index, 
                                  int32_t parent_index) {
    std::lock_guard<std::mutex> lock(mutex_);
    
    // 1. 检查是否已注册
    auto it = names_.find(name);
    if (it != names_.end()) {
        return it->second;  // 返回已存在的索引
    }
    
    // 2. 验证父类型有效性
    assert(parent_index < vec_.size());
    TypeData& parent = vec_[parent_index];
    assert(parent.t == parent_index);
    
    int32_t new_index;
    
    // 3. 分配索引
    if (static_index != TypeIndex::Dynamic) {
        // 静态类型：使用预定义索引
        std::cout << "TypeIndex[" << static_index << "]: static: " << name 
                  << ", parent " << vec_[parent_index].name << std::endl;
        new_index = static_index;
        assert(static_index < vec_.size());
    } else {
        // 动态类型：分配新索引
        std::cout << "TypeIndex[" << counter_ << "]: dynamic: " << name 
                  << ", parent " << vec_[parent_index].name << std::endl;
        new_index = counter_++;  // 原子递增
        
        // 扩展数组容量
        if (vec_.size() <= counter_) {
            vec_.resize(counter_, TypeData());
        }
    }
    
    // 4. 记录类型信息
    vec_[new_index].t = new_index;      // 自己的索引
    vec_[new_index].p = parent_index;   // 父类索引
    vec_[new_index].name = name;        // 类型名称
    
    // 5. 建立名称映射
    names_[name] = new_index;
    
    return new_index;
}
```

```cpp
// object_t 对 TypeContext 的调用
int32_t object_t::GetOrAllocRuntimeTypeIndex(const std::string_view& n, int32_t t, int32_t p) {
  return TypeContext::Global()->GetOrAllocRuntimeTypeIndex(n, t, p);
}
```

再回顾下 `DEFINE_TYPEINDEX` 和 `REGISTER_TYPEINDEX` , 前者给当前 TypeName 定义了 RuntimeTypeIndex 和 _GetOrAllocRuntimeTypeIndex 方法， 后者是实际运行了 _GetOrAllocRuntimeTypeIndex（其又会调用TypeContext::GetOrAllocRuntimeTypeIndex），给 TypeContext 中的 vec_ 和 names_ 添加了新的类型信息，而 MakeObject 调用的时候又会给当前 instance 设置 t_ （RuntimeTypeIndex -> _GetOrAllocRuntimeTypeIndex）。

### 类型转换

- `object_t::IsFrom(int32_t p)`: 检查对象是否继承自父类型
- `object_t::IsType<T>()`: 基于模板的类型检查
- `object_r::As<T>()`: 安全的运行时向下转换

```cpp
template <typename T>
bool IsType() const noexcept {
return IsFrom(T::RuntimeTypeIndex());
}
```

```cpp
// object_r::As<T>
template<typename T>
const T* As() const noexcept {
if (data_ && data_->IsType<T>()) {
    return static_cast<const T*>(data_.get());
}
return nullptr;
}
```

以及还有三个专门的转换函数，用来在不同类型和表示形式之间进行转换。

#### Downcast

类型擦除到具体类型的转换

```cpp
using Object = object_r;
template <typename R>
inline R Downcast(const Object& from) {
    return R(object_p<object_t>(const_cast<object_t*>(from.get())));
}
```

**功能**：将类型擦除的 `Object`（即 `object_r`）转换为具体的类型化引用。


**实现分析**：
```cpp
// 分解步骤：
const Object& from;                           // 输入：object_r 类型
from.get();                                   // 获取 const object_t* 指针
const_cast<object_t*>(from.get());           // 去除 const 限定符
object_p<object_t>(...);                     // 创建 object_p<object_t>
R(...);                                       // 构造目标类型 R
```

**使用场景**：
```cpp
// 示例：从 Object 转换为具体类型
Object obj_ref = object_r(MakeObject<NewStrNode>("hello"));

// 转换为具体的类型化引用
MyStringRef typed_ref = Downcast<NewStr>(obj_ref);

// 或者转换为智能指针
object_p<MyString> typed_ptr = Downcast<object_p<NewStrNode>>(obj_ref);
```

#### RTcast

运行时类型转换

```cpp
template <typename R, typename T>
inline R RTcast(const T* p) {
    return R(object_p<object_t>(
               const_cast<object_t*>(
                static_cast<const object_t*>(p))));
}
```

**功能**：将任何继承自 `object_t` 的指针转换为目标类型(`Object`)。

**实现分析**：
```cpp
// 分解步骤：
const T* p;                                   // 输入：任何继承自 object_t 的指针
static_cast<const object_t*>(p);            // 向上转换为 object_t*
const_cast<object_t*>(...);                 // 去除 const 限定符
object_p<object_t>(...);                    // 创建基类智能指针
R(...);                                       // 构造目标类型 R
```

**使用场景**：
```cpp
// 示例：从具体类型指针转换为其他形式
NewStr* str_ptr = new NewStr("hello");

// 转换为 Object 引用
Object obj_ref = RTcast<Object>(str_ptr);

// 转换为其他类型的智能指针
object_p<object_t> base_ptr = RTcast<object_p<object_t>>(str_ptr);

// 注意：需要手动管理原始指针的生命周期
```

#### NTcast

```cpp
template <typename BaseType, typename ObjType>
inline object_p<BaseType> NTcast(ObjType* ptr) noexcept {
  static_assert(std::is_base_of<BaseType, ObjType>::value,
                "Can only cast to the ref of same container type");
  return object_p<BaseType>(static_cast<object_t*>(ptr));
}
```

**功能**：在编译时验证的类型安全转换，用于兼容类型间的转换。

**实现分析**：
```cpp
// 编译时检查
static_assert(std::is_base_of<BaseType, ObjType>::value, ...);
// 确保 ObjType 继承自 BaseType

// 运行时转换
static_cast<object_t*>(ptr);                 // 转换为 object_t*
object_p<BaseType>(...);                     // 创建目标类型智能指针
```

**使用场景**：
```cpp
// 示例：派生类指针转换为基类智能指针
class Container : public object_t { ... };
class Array : public Container { ... };

Array* arr_ptr = new Array();

// 编译时安全的向上转换
object_p<Container> container_ptr = NTcast<Container>(arr_ptr);
object_p<object_t> base_ptr = NTcast<object_t>(arr_ptr);

// 编译错误的示例：
// object_p<Array> invalid = NTcast<Array>(container_ptr);  // 编译错误！
```

#### 转换示例

```cpp
// 创建不同类型的对象
auto str_obj = MakeObject<MyString>("hello");
auto arr_obj = MakeObject<MyArray>();

// 1. 使用 RTcast 转换为 Object
Object str_ref = RTcast<Object>(str_obj.get());
Object arr_ref = RTcast<Object>(arr_obj.get());

// 2. 存储在容器中（类型擦除）
std::vector<Object> objects = { str_ref, arr_ref };

// 3. 使用 Downcast 还原具体类型
for (const auto& obj : objects) {
    // 需要运行时类型检查
    if (obj.get()->IsType<MyString>()) {
        auto str_ref = Downcast<object_p<MyString>>(obj);
        std::cout << "Found string: " << str_ref->GetValue() << std::endl;
    }
    else if (obj.get()->IsType<MyArray>()) {
        auto arr_ref = Downcast<object_p<MyArray>>(obj);
        std::cout << "Found array: size = " << arr_ref->Size() << std::endl;
    }
}

// 4. 使用 NTcast 进行安全转换
class Container : public object_t { ... };
class SpecialContainer : public Container { ... };

auto special = MakeObject<SpecialContainer>();
object_p<Container> base_container = NTcast<Container>(special.get()); // 安全
object_p<object_t> base_object = NTcast<object_t>(special.get());     // 安全
```

TODO：这些Cast也许到后面再讲才合适。

### 类型示例

```cpp
// 定义新的对象类型
class MyObject : public object_t {
public:
    // 必须定义的类型常量
    static constexpr const int32_t INDEX = TypeIndex::Dynamic;
    static constexpr const std::string_view NAME = "MyObject";
    
    // 必须使用宏定义类型索引
    DEFINE_TYPEINDEX(MyObject, object_t);
    
    // 构造函数
    MyObject(int value) : value_(value) {}
    
    // 具体方法
    int GetValue() const { return value_; }
    
private:
    int value_;
};

REGISTER_TYPEINDEX(MyObject);  

// 使用示例
int main() {
    // 1. 创建对象
    auto obj = MakeObject<MyObject>(42);
    
    // 2. 类型化访问
    std::cout << obj->GetValue() << std::endl;
    
    // 3. 类型擦除存储
    object_r ref = object_r(obj);
    
    // 4. 运行时类型检查
    if (auto* my_obj = ref.As<MyObject>()) {
        std::cout << "Type match: " << my_obj->GetValue() << std::endl;
    }
    
    // 5. 类型层次检查
    if (ref.get()->IsType<MyObject>()) {
        std::cout << "Is MyObject type" << std::endl;
    }
    
    if (ref.get()->IsFrom(TypeIndex::Object)) {
        std::cout << "Inherits from Object" << std::endl;
    }
    
    return 0;
}
```

总结：这里介绍了一个**运行时**类型系统的实现。