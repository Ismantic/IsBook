# FFI 与动态模块

完成代码生成和动态库编译以后，还剩最后一个问题：Python 如何调用 `.so` 中的函数？一次完整使用如下：

```python
simple_compile(sum_to, "sum_to.so")

module = module_loader("./sum_to.so")
sum_to = module.get_function("sum_to")
result = sum_to(10)
```

这几行代码跨越了两个边界：Python 先通过 FFI 调用 Matx 核心库；核心库再加载生成的动态模块，并把其中的 C 函数包装成统一的 `Function`。

## 为什么以 C 接口为边界

Python 无法直接理解 `object_r`、`McValue` 或 `std::function`。即使两个动态库都由 C++ 编写，直接共享模板类型还会受到编译器、标准库和名称改编的影响。

Matx 因此在边界上只使用固定布局的数据和 C ABI：

```cpp
struct Value {
    Union u;
    int32_t p;
    int32_t t;
};

using BackendFunc = int (*)(
    Value* args,
    int num_args,
    Value* result,
    void* resource
);
```

`Value` 携带数据、辅助信息和类型标签，`BackendFunc` 约定参数数组、返回值地址、资源句柄和错误码。接口没有暴露 C++ 类，因此 Python 扩展、核心运行时和生成模块可以遵循同一套调用协议。

## Python FFI

Python 端同时使用两种连接方式。`ctypes` 加载 `libcase.so`，声明 `GetGlobal`、`GetBackendFunction` 和 `FuncCall_PYTHON_C_API` 等基础 C 函数的参数布局。`case_ext.so` 则是 Python C Extension，负责更复杂的可调用对象、参数转换和对象生命周期。

扩展模块提供两个关键 Python 类型：

- `PackedFuncBase` 持有 `FunctionHandle`，并通过 `tp_call` 表现得像普通 Python 函数；
- `ObjectBase` 持有 `ObjectHandle` 和类型索引，是 `PrimVar`、`PrimFunc`、`Array` 等包装类的基类。

例如 Python 构造 `PrimVar` 时，实际先查找注册函数 `ast.PrimVar`，再调用：

```python
self.__init_handle_by_constructor__(
    prim_var_, name, datatype
)
```

扩展模块把 Python 参数转换成 `Value[]`，调用 `FuncCall_PYTHON_C_API`。C API 再将每个 `Value` 视为 `McView`，组成 `Parameters`，最终调用注册表中的 `Function`：

```text
Python 参数
   ↓ case_ext
Value[]
   ↓ FuncCall_PYTHON_C_API
Parameters / McView
   ↓ Function
McValue 返回值
   ↓
Python 对象
```

## 返回哪一种 Python 对象

整数、浮点数、布尔值和字符串可以直接转换成对应的 Python 值。对象返回值只包含 C++ 指针和运行时类型索引，扩展模块还需要知道应该创建哪个 Python 包装类。

`@register_object("PrimVar")` 首先通过 `GetIndex` 查询 `PrimVarNode` 的类型索引，再把索引与 Python 的 `PrimVar` 类关联。收到对象返回值时，扩展模块按索引调用已注册的创建器，创建 `PrimVar` 实例并填入句柄。

这让 C++ 负责对象的真实数据和类型，Python 负责用户可见的类和方法。新增一种 AST 对象时，两端需要使用同一个类型名称完成注册。

## 跨语言生命周期

Python 包装对象并不拥有另一份 AST 数据，它只持有 C++ 对象指针。参数转换需要暂时保留对象时，扩展模块调用 `ObjectRetain` 增加侵入式引用计数；`ObjectBase` 被回收时调用 `ObjectFree` 减少计数。计数归零后，C++ Node 对象才会析构。

函数句柄采用不同的管理方式。`GetGlobal` 返回一个新复制的 `std::function`，`PackedFuncBase` 结束生命周期时通过 `FuncFree` 删除该副本。这样 Python 侧的函数对象不直接依赖注册表内部元素的地址。

## 错误不能穿过 C ABI

C++ 异常不能直接越过 C 函数边界。C API 使用统一约定：成功返回 `0`，失败返回 `-1`，错误文本保存在当前线程的存储中：

```cpp
try {
    // 调用 C++ 运行时
} catch (std::runtime_error& error) {
    SetError(error.what());
    return -1;
}
```

Python 扩展检查错误码，再读取 `GetError()` 并转换为 Python 异常。线程局部存储避免多个线程相互覆盖错误文本。当前通用宏只捕获 `std::runtime_error`，其他异常类型仍不属于这套转换协议。

## 动态模块协议

生成的 `.so` 不依赖 C++ 符号名来发现函数，而是导出约定的 C 符号：

```text
__mc_func_array__       BackendFunc 函数指针数组
__mc_func_registry__    函数名称与数组的对应关系
__mc_closures_names__   需要资源句柄的函数名称
__mc_module_ctx         模块上下文
```

`SourceRewriter` 在生成 C++ 时创建这些结构。每个普通函数还会生成一个 `name__c_api` 包装器，用来检查参数数量和基础类型、调用真正的 C++ 函数，再把结果写入 `Value`。

## 加载与查找

`runtime.ModuleLoader` 创建 `DefaultLibray`，使用 `dlopen(..., RTLD_NOW | RTLD_LOCAL)` 打开文件。随后 `LibraryModuleNode` 通过 `dlsym` 找到 `__mc_func_registry__`，读取函数名及其对应的 `BackendFunc`，建立模块自己的查找表。

Python 的 `Module.get_function(name)` 调用 C API `GetBackendFunction`。找到函数后，`WrapFunction` 将 `BackendFunc` 重新包装成核心运行时使用的：

```cpp
std::function<McValue(Parameters)>
```

包装函数负责完成反方向转换：将 `Parameters` 变成 `Value[]`，调用模块函数，再把返回的 `Value` 变成 `McValue`。它还捕获 `Module` 的对象引用，使函数仍被持有时动态库不会提前 `dlclose`。

因此，前面的调用最终形成一条闭环：

```text
Python PackedFunc
  → 核心 Function
  → 模块 BackendFunc
  → 生成的 C++ 函数
  → Value 返回值
  → Python 值
```

## 当前模块边界

模块系统已经完成普通全局函数的生成、加载、查找和调用。代码中还保留了类方法分组、闭包资源句柄和模块导入等扩展方向，但完整度不同：类生成仍是受限实现，生成器当前不产生闭包，`use_imports` 的递归模块查找也尚未实现。

这条边界很重要：Matx 并不是把生成的 C++ 直接焊死在 Python 扩展中，而是让所有模块遵循同一个小型 ABI。前端、运行时和生成代码由此可以分别演进，这也完成了本书从值表示到动态执行的整条主线。
