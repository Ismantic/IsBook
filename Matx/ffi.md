# FFI 动态模块

## 引言

完成代码生成和动态库编译以后，Python 函数已经变成动态库中的机器代码，但整个编译流程还没有结束。Python 只知道磁盘上多了一个 `.so` 文件，并不知道其中包含哪些函数，也不知道参数应当怎样转换、返回值应该包装成哪种 Python 对象。对于字符串、容器和 AST 对象，还需要继续处理跨语言的内存与生命周期。

因此，动态库必须提供一套稳定的调用协议：运行时先加载文件并发现其中导出的函数，再把这些函数包装成 Python 可以持有和调用的对象；每次调用时，参数从 Python 值转换成 C 接口能够识别的数据，执行结果再沿相反方向返回。一次完整使用如下：

```python
simple_compile(sum_to, "sum_to.so")

module = module_loader("./sum_to.so")
sum_to = module.get_function("sum_to")
result = sum_to(10)
```

这几行代码跨越了两个边界：Python 先通过 FFI 调用 Matx 核心库；核心库再加载生成的动态模块，并把其中的 C 函数包装成统一的 `Function`。

动态库不能只包含一段编译后的机器代码。调用者还需要知道其中有哪些函数、每个函数在哪里、参数应怎样排列、返回值属于什么类型，以及函数对象仍被使用时动态库能否卸载。Matx 为这些问题定义了一套小型模块协议：

```text
Python
   ↓ Python C Extension
Matx 核心运行时
   ↓ C ABI
生成的动态模块
   ↓
C++ 函数
```

Python 与核心运行时之间使用句柄和 `Value`，核心运行时与动态模块之间使用 `BackendFunc`。中间的 Runtime 将两侧都转换成前面介绍的 `Function`、`Parameters` 和 `McValue`，从而让上层调用者不必关心函数来自主程序还是新加载的 `.so`。

## 实现

**C ABI**

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

这里的 `Value` 与 Runtime 中的动态值具有相同目标，但使用固定的 C 布局。整数和浮点数直接保存在联合体中，字符串、对象与函数通过指针传递，`t` 记录类型索引，`p` 保存字符串长度等辅助信息。边界两侧都根据类型标签解释联合体，避免在 ABI 中出现 C++ 模板。

**Python FFI**

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

`ctypes` 适合加载库和声明少量稳定接口，Python C Extension 则适合频繁的参数转换和自定义对象。两者并不是两套独立的 FFI：它们最终调用同一组 C API，并共享 `FunctionHandle`、`ObjectHandle` 和 `Value` 协议。

**返回值**

整数、浮点数、布尔值和字符串可以直接转换成对应的 Python 值。对象返回值只包含 C++ 指针和运行时类型索引，扩展模块还需要知道应该创建哪个 Python 包装类。

`@register_object("PrimVar")` 首先通过 `GetIndex` 查询 `PrimVarNode` 的类型索引，再把索引与 Python 的 `PrimVar` 类关联。收到对象返回值时，扩展模块按索引调用已注册的创建器，创建 `PrimVar` 实例并填入句柄。

这让 C++ 负责对象的真实数据和类型，Python 负责用户可见的类和方法。新增一种 AST 对象时，两端需要使用同一个类型名称完成注册。

函数也是一种运行时返回类型。收到函数句柄后，扩展模块会创建 `PackedFuncBase`；之后对这个 Python 对象使用括号调用，就会再次进入通用的 `FuncCall_PYTHON_C_API`。因此，全局函数和模块函数在 Python 中具有相同的调用外观。

**生命周期**

Python 包装对象并不拥有另一份 AST 数据，它只持有 C++ 对象指针。参数转换需要暂时保留对象时，扩展模块调用 `ObjectRetain` 增加侵入式引用计数；`ObjectBase` 被回收时调用 `ObjectFree` 减少计数。计数归零后，C++ Node 对象才会析构。

函数句柄采用不同的管理方式。`GetGlobal` 返回一个新复制的 `std::function`，`PackedFuncBase` 结束生命周期时通过 `FuncFree` 删除该副本。这样 Python 侧的函数对象不直接依赖注册表内部元素的地址。

模块函数还会持有模块自身的引用。即使 Python 变量 `module` 已经离开作用域，只要从中取得的函数仍然存在，承载机器代码的动态库就不会被提前 `dlclose`。否则函数句柄仍在，函数地址却已经失效，下一次调用就会跳转到被卸载的内存。

**错误处理**

C++ 异常不能直接越过 C 函数边界。C API 使用统一约定：成功返回 `0`，失败返回 `-1`，错误文本保存在当前线程的存储中：

```cpp
try {
    // 调用 C++ 运行时
} catch (std::runtime_error& error) {
    SetError(error.what());
    return -1;
}
```

Python 扩展检查错误码，再读取 `GetError()` 并转换为 Python 异常。线程局部存储避免多个线程相互覆盖错误文本。调用者只面对 Python 异常，C++ 侧则仍然可以使用熟悉的异常方式报告参数、类型和模块加载错误。

**模块协议**

生成的 `.so` 不依赖 C++ 符号名来发现函数，而是导出约定的 C 符号：

```text
__mc_func_array__       BackendFunc 函数指针数组
__mc_func_registry__    函数名称与数组的对应关系
__mc_closures_names__   需要资源句柄的函数名称
__mc_module_ctx         模块上下文
```

`SourceRewriter` 在生成 C++ 时创建这些结构。每个普通函数还会生成一个 `name__c_api` 包装器，用来检查参数数量和基础类型、调用真正的 C++ 函数，再把结果写入 `Value`。

名称表和函数指针表按相同顺序排列。模块加载器读取第 `i` 个名称时，就能把它与第 `i` 个 `BackendFunc` 建立对应关系。这样只需要导出少量固定符号，新增用户函数不会改变加载器本身。

**加载与查找**

`runtime.ModuleLoader` 创建底层 `Library` 对象，使用 `dlopen(..., RTLD_NOW | RTLD_LOCAL)` 打开文件。随后 `LibraryModuleNode` 通过 `dlsym` 找到 `__mc_func_registry__`，读取函数名及其对应的 `BackendFunc`，建立模块自己的查找表。

Python 的 `Module.get_function(name)` 调用 C API `GetBackendFunction`。找到函数后，`WrapFunction` 将 `BackendFunc` 重新包装成核心运行时使用的：

```cpp
std::function<McValue(Parameters)>
```

包装函数负责完成反方向转换：将 `Parameters` 变成 `Value[]`，调用模块函数，再把返回的 `Value` 变成 `McValue`。它还捕获 `Module` 的对象引用，使函数仍被持有时动态库不会提前 `dlclose`。

## 示例

现在回到 `sum_to(10)`：

```python
module = module_loader("./sum_to.so")
sum_to = module.get_function("sum_to")
result = sum_to(10)
```

`module_loader` 本身是全局注册函数。它接收动态库路径，调用 `dlopen`，读取模块导出的函数注册表，并返回一个 `Module` 对象。Python 包装类持有这个对象的句柄。

`module.get_function("sum_to")` 随后调用 `GetBackendFunction`。`LibraryModuleNode` 在自己的名称表中找到 `sum_to__c_api` 对应的 `BackendFunc`，再通过 `WrapFunction` 得到核心运行时使用的 `Function`。Python 最终拿到的是 `PackedFuncBase`，而不是原始函数地址。

执行 `sum_to(10)` 时，整数 `10` 被放入一个 `Value`。调用依次经过两层统一包装：

```text
Python: sum_to(10)
    ↓ PackedFuncBase
Value[0] = Int(10)
    ↓ FuncCall_PYTHON_C_API
核心 Function(Parameters)
    ↓ WrapFunction
模块 BackendFunc
    ↓ sum_to__c_api
C++: sum_to(int64_t n)
    ↓
Value = Int(result)
    ↓
Python int
```

生成的 C 包装函数检查参数数量和整数类型，调用真正的 C++ `sum_to`，再把结果写入返回 `Value`。返回路径经过 `BackendFunc`、核心 `Function` 和 Python 扩展，最终得到普通的 Python `int`。

这条链路看起来较长，却把每一层的职责限制得很清楚：Python 负责用户接口，核心 Runtime 负责动态值和统一函数，模块协议负责发现能力，生成代码负责实际计算。Matx 没有把每个生成函数直接写死在 Python 扩展中，而是让动态模块遵循同一个小型 ABI。至此，从 Python 源码、Matx AST、C++ 生成到动态加载和调用的完整编译流程形成闭环。
