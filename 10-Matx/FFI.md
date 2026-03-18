# FFI

## 概述

FFI 实现了 C++ 运行时系统与 Python 前端之间的交互操作。整个 FFI 的设计哲学是：函数接口统一、类型安全转化、跨语言对象管理。


MC 实现了两种 FFI 方式：

```
┌─────────────────────────────────────────────────────────────┐
│                    Python Layer (ffi.py)               │
│          对象注册装饰器、AST编译器、Python包装类              │ 
└─────────────┬──────────────────────┬────────────────────────┘
              │                      │
        CTypes直接调用          Python C Extension
              │                      │
┌─────────────▼──────────────┐  ┌───▼────────────────────────┐
│     Direct C API Access    │  │  Python Extension          │
│      (CTypes -> C API)     │  │     (case_ext.cc)          │
│  GetGlobal, GetError,      │  │  PyObjectBase、类型转换、   │
│  ListGlobalNames等简单调用  │  │  回调机制、对象管理         │
└─────────────┬──────────────┘  └───┬────────────────────────┘
              │                      │
              └──────────┬───────────┘
                         │
┌─────────────────────────▼───────────────────────────────────┐
│                    C API Layer (c_api.h/cc)                │
│  Value结构体、函数调用接口、错误处理、对象生命周期管理          │
└─────────────────────────┬───────────────────────────────────┘
                          │ Function Registry
┌─────────────────────────▼───────────────────────────────────┐
│                C++ Runtime Core                             │
│  FunctionRegistry、类型擦除、参数转换、模块管理                  │
└─────────────────────────────────────────────────────────────┘
```

**两种FFI方式的分工：**

1. **CTypes方式**：用于简单的C函数调用
   - 直接加载libcase.so动态库
   - 调用基础的C API函数：GetGlobal、GetError、ListGlobalNames等
   - 适合简单的数据交换和函数查找

2. **Python C Extension方式**：用于复杂的对象管理
   - 通过case_ext.cc提供的Python扩展模块
   - 实现复杂的类型转换、对象回调、生命周期管理
   - 支持Python对象与C++对象的无缝集成

## C_API

由于Python的CTypes和C扩展只能调用C风格的函数，而MC的核心是C++，因而需要一个C API层来适配。

### Value

```cpp
union Union {
    int64_t v_int;      // 整数
    double  v_float;    // 浮点数
    char*   v_str;      // 字符串指针
    void*   v_pointer;  // 通用指针
    Dt      v_datatype; // 数据类型枚举
};

struct Value {
    Union u{};         // 联合体存储实际值
    int32_t p{0};      // 辅助信息（如字符串长度）
    int32_t t{0};      // 类型索引（TypeIndex枚举）
};
```

Value结构体是跨语言传递数据的"通用货币"：
- **类型安全**：通过类型索引`t`确保运行时类型安全
- **高效传输**：联合体设计避免了不必要的内存开销
- **扩展性强**：新类型只需要在联合体中增加字段

### FuncCall

这是整个FFI系统的关键函数(会被case_ext调用)，它将C风格的函数调用转换为C++风格：

```cpp
int FuncCall_PYTHON_C_API(FunctionHandle func, Value* vs, int n, Value* r) {
    API_BEGIN();
    
    // 1. C Value数组 -> C++ McView数组
    std::vector<McView> gs;
    gs.reserve(n);
    for (int i = 0; i < n; ++i) {
        gs.push_back(McView(vs[i]));  // 这里发生C->C++转换
    }
    
    // 2. 调用Function.md中注册的统一函数接口
    auto* function = static_cast<const Function*>(func);
    McValue rv = (*function)(Parameters(gs.data(), gs.size()));
    
    // 3. C++ McValue -> C Value
    if (rv.T() == TypeIndex::DataType) {
        // 特殊处理：DataType需要转换为字符串
        std::string s = DtToStr(rv.As<Dt>());
        StrAsValue(s, r);
    } else {
        rv.AsValue(r);  // 这里发生C++->C转换
    }
    
    API_END();
}
```

这个函数是整个调用链的中枢：
- **输入**：C风格的函数句柄和参数数组
- **处理**：转换为C++对象并调用registry中的统一函数
- **输出**：将C++结果转换回C风格

### 错误处理

```cpp
template <typename T>
class LocalStore {
public:
    static T* Get() {
        static thread_local T inst;  // 每个线程独立的实例
        return &inst;
    }
};

struct RuntimeEntry {
    std::string error;
};

// 设置错误信息（C++异常被捕获后调用）
void SetError(const char* s) {
    LocalStore<RuntimeEntry>::Get()->error = s;
}

// 获取错误信息（Python通过ctypes调用）
const char* GetError() {
    return LocalStore<RuntimeEntry>::Get()->error.c_str();
}
```

错误处理的设计考虑：
- **线程安全**：每个线程有独立的错误存储
- **异常隔离**：C++异常不会穿越C边界
- **信息保留**：错误信息被转换为字符串保留

### API宏

```cpp
#define API_BEGIN() try {

#define API_END() \
    } \
    catch (std::runtime_error& _except_) { \
        return HandleError(_except_); \
    } \
    return 0;

int HandleError(const std::runtime_error& e) {
    SetError(NormalizeError(e.what()).c_str());
    return -1;  // C API标准：0成功，非0失败
}
```

这套宏确保了：
- **异常不泄露**：所有C++异常都被捕获
- **错误码统一**：成功返回0，失败返回-1
- **错误信息保留**：异常信息被转换为线程局部的错误字符串

### 基础函数

C API层提供了一系列基础函数供Python调用：

```cpp
// 函数查找
int GetGlobal(const char* name, FunctionHandle* handle) {
    API_BEGIN();
    
    auto* func = FunctionRegistry::Get(name);
    if (!func) {
        throw std::runtime_error(std::string("Function not found: ") + name);
    }
    
    *handle = reinterpret_cast<FunctionHandle>(func);
    API_END();
}

// 函数列表
int ListGlobalNames(uint32_t* size, const char*** names) {
    API_BEGIN();
    
    auto name_list = FunctionRegistry::ListNames();
    static std::vector<const char*> name_ptrs;
    static std::vector<std::string> name_storage;
    
    name_storage.clear();
    name_ptrs.clear();
    name_storage.reserve(name_list.size());
    name_ptrs.reserve(name_list.size());
    
    for (const auto& name : name_list) {
        name_storage.push_back(std::string(name));
        name_ptrs.push_back(name_storage.back().c_str());
    }
    
    *size = name_ptrs.size();
    *names = name_ptrs.data();
    
    API_END();
}

// 类型索引查找
int GetIndex(const char* name, uint32_t* index) {
    API_BEGIN();
    
    auto* ctx = TypeContext::Global();
    auto it = ctx->name_to_index_.find(name);
    if (it == ctx->name_to_index_.end()) {
        throw std::runtime_error(std::string("Type not found: ") + name);
    }
    
    *index = it->second;
    API_END();
}
```

### 封装使用

C API函数能通过ctypes模块封装使用，注意设置restype与argtypes。

```python

def c_str(str):
    return ctypes.c_char_p(str.encode('utf-8'))

lib_path = os.path.join(os.path.dirname(__file__), "build/libcase.so")
_LIB = ctypes.CDLL(lib_path, ctypes.RTLD_GLOBAL)
_LIB.GetError.restype = ctypes.c_char_p  

py_str = lambda x : x.decode('utf-8')

def list_global_func_names():
    plist = ctypes.POINTER(ctypes.c_char_p)()
    size = ctypes.c_uint()

    _LIB.ListGlobalNames(ctypes.byref(size),
                                    ctypes.byref(plist))
    fnames = []
    for i in range(size.value):
        fnames.append(py_str(plist[i]))
    return fnames

for n in list_global_func_names():
    print (n)

# 定义函数句柄类型
FunctionHandle = ctypes.c_void_p

# 设置函数参数和返回类型
_LIB.GetGlobal.argtypes = [ctypes.c_char_p, ctypes.POINTER(FunctionHandle)]
_LIB.GetGlobal.restype = ctypes.c_int

def get_global(name: str) -> Optional[FunctionHandle]:
    """获取全局函数
    
    Args:
        name: 函数名
        
    Returns:
        函数句柄，如果函数不存在返回 None
    """
    fn = FunctionHandle()
    ret = _LIB.GetGlobal(
        name.encode('utf-8'),  # 转换为 bytes
        ctypes.byref(fn)       # 传递指针
    )
    
    if ret != 0:
        raise RuntimeError(f"Failed to get function: {name}")
        
    if fn.value is None:
        return None
        
    return fn
```

## C_EXT

接下来对下面这段代码，做一个彻底的分析：

```python
pv = PrimVar("x", "int64")  # 创建一个原始变量
result = op_add(pv, PrimExpr(42))  # 进行加法运算
```

1. `PrimVar("x", "int64")` 如何调用 C++ 的构造函数？
2. `op_add` 函数返回的 C++ 对象如何变成 Python 对象？
3. Python 如何知道要创建什么类型的对象？

以上问题要说清楚，关键点是 C_EXT 的回调机制。

> **回调机制就是要在 C++ 代码里调用 Python 代码**

**为什么需要回调机制？**
1. **C++ 不知道如何创建 Python 对象**：C++ 只能管理 C++ 对象，不知道如何创建复杂的 Python 对象
2. **Python 端有创建策略**：Python 端知道要创建什么类型的对象，以及如何创建
3. **解耦设计**：让 C++ 端专注于数据管理，让 Python 端专注于对象创建策略

**回调机制的工作流程**：
```
C++ 函数返回数据 → C++ 发现需要创建 Python 对象 → 调用 Python 注册的创建器 → Python 创建对象 → 返回给 C++
```

### 模块基础

通过 Python C Extention 机制创建的 Python 模块，本项目中是`case_ext.so`，其使用方式是：

**Python 端的使用**
```python
mc_api = _load_case_ext("case_ext.so")
```

- `mc_api` 就是从 `case_ext.so` 动态库加载的 Python 模块
- `case_ext.so` 是由 `case_ext.cc` 编译生成的 Python C Extension 模块
- `mc_api` 这个模块向 Python 暴露了 C++ 定义的类型和函数

#### 初始化

**C++ 端的模块定义**
```c
static struct PyModuleDef CaseExtModule = {
    PyModuleDef_HEAD_INIT,
    "case_ext",              /* 模块名 */
    "Case Extension Module", /* 模块文档 */
    -1,                      /* 模块状态大小 */
    CaseExtMethods           /* 模块方法表 */
};
```

**模块方法表（case_ext.cc:890-899）：**
```c
static PyMethodDef CaseExtMethods[] = {
    {"GetGlobal", get_global_func, METH_VARARGS, "Get global function by name"},
    {"RegisterObject", register_object, METH_VARARGS, "Register object creator"},
    {"RegisterObjectCallback", register_object_callback, METH_VARARGS, "Register object callback"},
    {"SetClassObject", set_class_object, METH_VARARGS, "Set default class object creator"},
    {"RegisterInputCallback", register_input_instance_callback, METH_VARARGS, "Register input callback"},
    {NULL, NULL, 0, NULL}  // Sentinel
};
```

#### 对象类

两个核心对象类型：

**PackedFuncBase**：函数包装器
```c
typedef struct PyPackedFuncBase {
    PyObject_HEAD;
    FunctionHandle handle;    // 指向 C++ 函数的句柄
} PyPackedFuncBase;
```

**ObjectBase**：对象包装器
```c
typedef struct PyObjectBase {
    PyObject_HEAD;
    ObjectHandle handle;      // 指向 C++ 对象的句柄
    int32_t type_code;       // 运行时类型索引
} PyObjectBase;
```

### 对象创建

1. **PackedFuncBase 创建**：处理函数对象的创建
2. **ObjectBase 创建**：处理普通对象的创建

#### Function 路线

调用流程：

```
Python: mc_api.GetGlobal("ast.PrimVar")
    ↓
C Extension: GetGlobalFunc()
    ↓
C API: GetGlobal(name, &handle)
    ↓
C Extension: ValueSwitchToPackedFunc()
    ↓
C Extension: 直接创建 PackedFuncBase 对象
    ↓
Python: 返回可调用的函数对象
```

**第一步：GetGlobalFunc - 入口函数**

**用户调用**：
```python
prim_var_ = mc_api.GetGlobal("ast.PrimVar")
```

**C++ 实现**：
```c
static PyObject* GetGlobalFunc(PyObject* self, PyObject* args) {
    const char* name = NULL;
    PyObject* allow_missing = NULL;

    // 1. 解析 Python 参数
    if (!PyArg_ParseTuple(args, "sO", &name, &allow_missing)) {
        return NULL;
    }

    // 2. 调用 C API 获取函数句柄
    FunctionHandle handle;
    if (GetGlobal(name, &handle)) {
        PyErr_SetString(PyExc_RuntimeError, "failed to call GetGlobal");
        return NULL;
    }

    if (handle) {
        // 3. 构造 Value 结构
        Value value;
        value.t = mc::runtime::TypeIndex::Func;
        value.u.v_pointer = handle;
        value.p = 0;
        
        // 4. 调用 ValueSwitchToPackedFunc 创建 Python 对象
        return ValueSwitchToPackedFunc(&value);
    }

    Py_RETURN_NONE;
}
```

**第二步：ValueSwitchToPackedFunc - 创建函数对象**

**作用**：将 C++ 函数句柄转换为 Python 可调用对象

```c
static PyObject* ValueSwitchToPackedFunc(Value* value) {
    // 1. 直接创建 PackedFuncBase 对象
    PyObject* obj = PyPackedFuncBase_New(&PyType_PackedFuncBase, NULL, NULL);
    if (obj == NULL) {
        return NULL;
    }
    
    // 2. 设置函数句柄
    PyPackedFuncBase* func = (PyPackedFuncBase*)obj;
    func->handle = value->u.v_pointer;
    
    // 3. 返回 Python 对象
    return obj;
}
```

**第三步：PackedFuncBase 对象的使用**

**创建过程**：
```c
// PyPackedFuncBase_new() 创建对象
static PyObject* PyPackedFuncBase_New(PyTypeObject* type, PyObject* args, PyObject* kwargs) {
    PyPackedFuncBase* self = (PyPackedFuncBase*)type->tp_alloc(type, 0);
    if (self != NULL) {
        self->handle = NULL;  // 初始化为 NULL
    }
    return (PyObject*)self;
}
```

**调用过程**：
```c
// PyPackedFuncBase_call() 使对象可调用
static PyObject* PyPackedFuncBase_Call(PyObject* self, PyObject* args, PyObject* kwargs) {
    PyPackedFuncBase* func = (PyPackedFuncBase*)self;
    
    // 1. 转换 Python 参数为 C++ Value 数组
    Py_ssize_t size = PyTuple_GET_SIZE(args);
    Value* values = new Value[size];
    for (Py_ssize_t i = 0; i < size; ++i) {
        PyObject* item = PyTuple_GET_ITEM(args, i);
        if (0 != PyObjectToValue(item, &values[i])) {
            goto FREE_ARGS;
        }
    }
    
    // 2. 调用 C++ 函数
    Value ret_val;
    if (0 != FuncCall_PYTHON_C_API(func->handle, values, size, &ret_val)) {
        PyErr_SetString(PyExc_TypeError, GetError());
        goto FREE_ARGS;
    }
    
    // 3. 转换返回值为 Python 对象
    PyObject* result = ValueToPyObject(&ret_val);
    
FREE_ARGS:
    delete[] values;
    return result;
}
```

**举例**
```python
# 1. 获取函数对象
prim_var_ = mc_api.GetGlobal("ast.PrimVar", True)

# 2. 调用函数对象
result = prim_var_("x", "int64")  # 调用 PyPackedFuncBase_Call
```

ValueToPyObject 涉及到怎么在 C++ 端调用 Python 端函数，生成 Python 对象，这就是 Object 路线要说明的创建过程。

#### Object 路线

调用流程：

```
C++ 函数返回对象
    ↓
C Extension: ValueToPyObject()
    ↓
C Extension: ValueSwitchToObject()
    ↓
C Extension: 查找 RETURN_SWITCH (通过RegisterObjectCreator设置)
    ↓
C Extension: 调用 Python 创建器
    ↓
C Extension: 设置对象属性
    ↓
C Extension: 调用 OBJECT_CALLBACK_TABLE (通过RegisterObjectCallback设置)
    ↓
Python: 返回完整对象
```


**第一步：ValueToPyObject - 类型分发**

**作用**：根据 Value 的类型决定如何创建 Python 对象

**实现（case_ext.cc:428-490）**：
```c
static PyObject* ValueToPyObject(Value* value) {
    switch (value->t) {
        case mc::runtime::TypeIndex::Null:
            Py_RETURN_NONE;
        case mc::runtime::TypeIndex::Int:
            return PyLong_FromLongLong(value->u.v_int);
        case mc::runtime::TypeIndex::Float:
            return PyFloat_FromDouble(value->u.v_float);
        case mc::runtime::TypeIndex::Str:
            return PyUnicode_FromStringAndSize(value->u.v_str, value->p);
        case mc::runtime::TypeIndex::Object:
            // 基础对象类型，直接创建 ObjectBase
            PyObject* obj = PyObjectBase_new(&PyType_ObjectBase, NULL, NULL);
            PyObjectBase* base = (PyObjectBase*)obj;
            base->handle = value->u.v_pointer;
            base->type_code = value->t;
            return obj;
        default:
            // 扩展对象类型，需要通过回调机制创建
            if (value->t < 0) {
                PyErr_SetString(PyExc_TypeError, "Unknown value type");
                return NULL;
            }
            return ValueSwitchToObject(value);  // 【关键】进入回调机制
    }
}
```

**第二步：ValueSwithToObject - 回调机制**

**作用**：通过回调机制创建特定类型的 Python 对象

**实现（case_ext.cc:377-426）**：
```c
static PyObject* ValueSwitchToObject(Value* value) {
    // 1. 根据类型索引查找专门的创建器
    PyObject* index = PyLong_FromLongLong(value->t);
    PyObject* creator = PyDict_GetItem(RETURN_SWITCH, index);
    Py_DECREF(index);

    if (!creator) {
        // 2. 如果没有找到，使用默认创建器
        if (DEFAULT_CLASS_OBJECT) {
            creator = DEFAULT_CLASS_OBJECT;
        } else {
            PyErr_SetString(PyExc_TypeError, "type_code is not registered");
            return NULL;
        }
    }

    // 3. 【关键】C++ 调用 Python 创建器
    PyObject* func_args = PyTuple_Pack(0);
    PyObject* result = PyObject_Call(creator, func_args, NULL);
    Py_DECREF(func_args);
    
    // 4. 设置对象句柄和类型代码
    PyObjectBase* super = (PyObjectBase*)result;
    super->handle = value->u.v_pointer;
    super->type_code = value->t;

    // 5. 【关键】调用后处理回调
    for (int i = 0; i < OBJECT_CALLBACK_CUR_IDX; ++i) {
        if (OBJECT_CALLBACK_TABLE[i].index == value->t) {
            PyObject* callback_args = PyTuple_Pack(1, result);
            PyObject* ret = PyObject_Call(OBJECT_CALLBACK_TABLE[i].callback, callback_args, NULL);
            Py_DECREF(callback_args);
            Py_DECREF(ret);
            break;
        }
    }
    
    return result;
}
```

**第三步：RegisterObjectCreator - 注册类型特定创建器**

**作用**：让 C++ 代码能够调用 Python 代码来创建特定类型的对象

**Python 端使用**：
```python
def _register_object(index, cls, callback):
    def _creator():
        obj = cls.__new__(cls)  # 创建未初始化的对象
        return obj

    mc_api.RegisterObjectCreator(index, _creator)
    if callback is not None:
        mc_api.RegisterObjectCallback(index, callback)

def register_object(object_name, callback=None):
    def register(cls):
        tidx = ctypes.c_uint()
        _LIB.GetIndex(c_str(object_name), ctypes.byref(tidx))
        _register_object(tidx, cls, callback)
        return cls
    return register

# 使用装饰器注册
@register_object("PrimVar")
class PrimVar(PrimExprWithOp):
    def __init__(self, name, datatype):
        self.__init_constructor__(prim_var_, name, datatype)
```


**C++ 端实现**：
```c
static PyObject* RegisterObjectCreator(PyObject* self, PyObject* args) {
    long long index = 0;
    PyObject* creator;

    if (!PyArg_ParseTuple(args, "LO", &index, &creator)) {
        return NULL;
    }

    // 将创建器存储到 RETURN_SWITCH 字典中
    if (RETURN_SWITCH == NULL) {
        RETURN_SWITCH = PyDict_New();
    }
    
    PyObject* index_obj = PyLong_FromLongLong(index);
    PyDict_SetItem(RETURN_SWITCH, index_obj, creator);
    Py_DECREF(index_obj);
    
    Py_RETURN_NONE;
}
```

对 Callback：

**Python 端使用**：
```python
def prim_var_callback(obj):
    print(f"PrimVar object created: {obj}")
```

**C++ 端实现**：
```c
static PyObject* RegisterObjectCallback(PyObject* self, PyObject* args) {
    long long index = 0;
    PyObject* callback;

    if (!PyArg_ParseTuple(args, "LO", &index, &callback)) {
        return NULL;
    }
    
    // 存储到 OBJECT_CALLBACK_TABLE 数组中
    OBJECT_CALLBACK_TABLE[OBJECT_CALLBACK_CUR_IDX].index = index;
    OBJECT_CALLBACK_TABLE[OBJECT_CALLBACK_CUR_IDX].callback = callback;
    ++OBJECT_CALLBACK_CUR_IDX;

    Py_RETURN_NONE;
}
```

**第四步：init_constructor - Python 调用 C++ 构造函数**

**作用**：让 Python 对象能够调用 C++ 构造函数进行初始化

**使用方式**：
```python
class PrimVar(PrimExprWithOp):
    def __init__(self, name, datatype):
        # 调用 C++ 构造函数
        self.__init__constructor__(prim_var_, name, datatype)
```

**C++ 实现**：
```c
static PyObject* PyObjectBase_init_constructor(PyObject* self, PyObject* args) {
    PyObjectBase* super = (PyObjectBase*)self;
    
    // 1. 获取构造函数句柄
    PyObject* item_0 = PyTuple_GET_ITEM(args, 0);
    void* func_addr = ((PyPackedFuncBase*)(item_0))->handle;
    
    // 2. 转换构造函数参数
    Py_ssize_t size = PyTuple_GET_SIZE(args);
    Value* item_buffer = new Value[size];
    for (Py_ssize_t i = 1; i < size; ++i) {
        PyObject* item = PyTuple_GET_ITEM(args, i);
        PyObjectToValue(item, item_buffer + i - 1);
    }
    
    // 3. 调用 C++ 构造函数
    Value ret_val;
    FuncCall_PYTHON_C_API(func_addr, item_buffer, size - 1, &ret_val);
    
    // 4. 绑定 C++ 对象到 Python 对象
    super->handle = ret_val.u.v_pointer;
    super->type_code = ret_val.t;
    
    delete[] item_buffer;
    Py_RETURN_NONE;
}
```

#### 举例

**场景设置**

```python
# 1. 获取函数（Function 线路）
prim_var_ = mc_api.GetGlobal("ast.PrimVar", True)  # 构造函数
op_add_ = mc_api.GetGlobal("ast._OpAdd", True)     # 运算函数

# 2. 注册对象类型（Object 线路准备）
@register_object("PrimVar")
class PrimVar(PrimExprWithOp):
    def __init__(self, name, datatype):
        self.__init__constructor__(prim_var_, name, datatype)

@register_object("PrimExpr")  # 加法结果是 PrimExpr 类型
class PrimExpr(Expr):
    pass
```

**调用流程**

```python
# 3. 创建对象（使用构造函数，不走 ValueToPyObject）
pv1 = PrimVar("x", "int64")
pv2 = PrimVar("y", "int64")

# 4. 调用运算函数（这里会走 Object 线路的 ValueToPyObject）
result = op_add_(pv1, pv2)  # C++ 函数返回对象，触发 Object 线路
```


**第1步 - 获取函数（Function 线路）**：
```
mc_api.GetGlobalFunc("ast._OpAdd", True)
↓
GetGlobalFunc() → GetGlobal() → ValueSwitchToPackedFunc()
↓
返回 PackedFuncBase 对象（op_add_）
```

**第2步 - 创建对象（构造函数调用，不走回调）**：
```
PrimVar("x", "int64")
↓
__init__constructor__() → FuncCall_PYTHON_C_API()
↓
C++ 构造函数执行 → 返回对象句柄 → 直接赋值给 Python 对象
```

**第3步 - 函数调用（触发 Object 线路）**：
```
op_add_(pv1, pv2)
↓
PyPackedFuncBase_Call() → 转换参数 → FuncCall_PYTHON_C_API()
↓
C++ 执行加法运算，返回新的 OpAdd 对象
↓
【关键】ValueToPyObject() → ValueSwitchToObject()
↓
查找 RETURN_SWITCH["OpAdd"] → 调用 Python 创建器
↓
创建 OpAdd Python 对象 → 设置 handle 和 type_code
↓
返回完整的 Python 对象
```

#### 总结

Ast类型系统需要在Python端，通过Python的继承体系也构建一份，而C++端是无法知道Python的对象创建器的，因而要在Python端得到C++端生成的Ast类型的对象，就需要通过设置回调函数的方式把对象创建器传给C++，且通过类型Index实现一一对应。除了对象创建之外，还需要对成员赋值，因而还要引入init_constructor函数，通过在Python段调用Function对象，把Python参数传入，实现完整的初始化。

### 输入转换

除了对象创建，还提供输入参数转换的回调机制，让Python对象能够在传递给C++函数时进行特殊处理。

**问题**：当 Python 对象需要作为参数传递给 C++ 函数时，如何进行特殊的转换？

**解决方案**：通过 `RegisterInputCallback` 注册转换函数

**使用示例**：
```python
def _conv(mod: Module):
    handle = mod.handle
    return mc_api.MakeAny(1, 0, handle, 0)

mc_api.RegisterInputCallback(Module, _conv)
```

**转换过程**：
1. Python 调用 C++ 函数时，参数通过 `PyObjectToValue` 转换
2. `PyObjectToValue` 检查是否有注册的转换器
3. 如果有，调用转换器进行特殊处理
4. 否则，使用默认转换逻辑

**C++ 实现**：
```c
static int PyObjectToValue(PyObject* arg_0, Value* value) {
    // 1. 检查是否有注册的转换器
    for (int i = 0; i < INPUT_INSTANCE_CALLBACK_CUR; ++i) {
        if (PyObject_IsInstance(arg_0, INPUT_INSTANCE_CALLBACK[i][0])) {
            // 2. 调用转换器
            PyObject* args = PyTuple_Pack(1, arg_0);
            PyObject* result = PyObject_Call(INPUT_INSTANCE_CALLBACK[i][1], args, NULL);
            Py_DECREF(args);
            
            // 3. 转换结果
            if (result) {
                PyAny* any = (PyAny*)result;
                *value = any->value;
                Py_DECREF(result);
                return 0;
            }
            return -1;
        }
    }
    
    // ... 其他类型转换
}
```

## LibLoader
接下来要解决的问题是：该怎么动态加载库，且让其中的函数能够通过FFI调用。

**概览**

```
┌─────────────────────────────────────────────────────────────────┐
│                        Python 用户层                             │
│  test_func = module_loader_("./test_func.so")                   │
│  result = test_func.get_function("test_func")(3, 4)             │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                    Python FFI 层                               │
│  Module 对象包装、RegisterObject、RegisterInputCallback      │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                    C++ 抽象层                                   │
│  Module、ModuleNode、Library 抽象接口                          │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                 动态库实现层                                     │
│  LibraryModuleNode、DefaultLibray、函数注册表解析               │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                   系统调用层                                     │
│  dlopen、dlsym、函数符号解析                                   │
└─────────────────────────────────────────────────────────────────┘
```
### 动态库结构

```cpp
#include "runtime_value.h"
#include "parameters.h"
#include "registry.h"
#include "c_api.h"

using namespace mc::runtime;

// 1. 模块上下文指针（必须）
extern "C" void* __mc_module_ctx = nullptr;

namespace {
    // 2. 业务逻辑函数（C++ 实现）
    int32_t test_func(int32_t a, int32_t b) {
        int32_t c = (a + b);
        int32_t ssa = (c * b);
        c = ssa;
        return c;
    }

    // 3. C API 包装函数（必须）
    int test_func__c_api(Value* args, int num_args, Value* ret_val, void* resource_handle) {
        // 参数验证
        if (num_args != 2) {
            char error_msg[100];
            snprintf(error_msg, sizeof(error_msg), 
                    "test_func() takes 2 positional arguments but %d were given", 
                    num_args);
            SetError(error_msg);
            return -1;
        }

        // 类型检查
        if (args[0].t != TypeIndex::Int || args[1].t != TypeIndex::Int) {
            SetError("test_func argument type mismatch, expect 'int' type");
            return -1;
        }

        // 调用业务逻辑
        auto result = test_func(args[0].u.v_int, args[1].u.v_int);
        
        // 设置返回值
        ret_val->t = TypeIndex::Int;
        ret_val->u.v_int = result;
        return 0;
    }
}

extern "C" {
    // 4. 函数指针数组（必须）
    BackendFunc __mc_func_array__[] = {
        (BackendFunc)test_func__c_api,
    };

    // 5. 函数注册表（必须）
    FuncRegistry __mc_func_registry__ = {
        "\1test_func",        // 编码：1个函数，名称"test_func"
        __mc_func_array__,    // 函数指针数组
    };

    // 6. 闭包函数名称（可选）
    const char* __mc_closures_names__ = "0\000";  // 没有闭包函数

    // 7. 模块初始化函数（可选）
    __attribute__((constructor))
    void init_module() {
        printf("Module loaded, registry at: %p\n", &__mc_func_registry__);
    }
}
```

**C API 包装函数模式**：
- **输入**：`Value* args` 数组，跨语言的通用数据格式
- **职责**：参数验证 → 类型转换 → 业务逻辑 → 结果转换
- **输出**：返回码（0成功，-1失败）+ 结果填充到 `ret_val`

**函数注册表编码**：
- **格式**：`[函数数量:1字节][函数名1\0][函数名2\0]...`
- **示例**：`"\1test_func"` = 1个函数，名称为"test_func"

**编译命令**：
```bash
g++ -shared -fPIC test_func.cc -I./src -lcase -o test_func.so
```

### 动态库接口

**Library**：提供平台无关的动态库操作接口

```cpp
class Library : public object_t {
public:
    virtual ~Library() {}
    virtual void* GetSymbol(std::string_view name) = 0;  // 符号查找
};
```

**具体实现**：
```cpp
class DefaultLibray final : public Library {
private:
    void* pointer_{nullptr};

    void Load(const std::string& name) {
        pointer_ = dlopen(name.c_str(), RTLD_NOW | RTLD_LOCAL);
        if (!pointer_) {
            printf("Failed to load library: %s\n", dlerror());
        }
    }

    void* GetSymbol_(const char* name) {
        return dlsym(pointer_, name);
    }

public:
    void Init(const std::string& name) { Load(name); }
    void* GetSymbol(std::string_view name) final {
        return GetSymbol_(name.data());
    }
};
```

**Module**：三层设计 ModuleNode/LibraryModuleNode/Module
- `ModuleNode`：抽象基类，定义模块功能接口
- `LibraryModuleNode`：具体实现，管理动态库函数
- `Module`：用户接口，提供简洁的操作方法

```cpp
class ModuleNode : public object_t {
public:
    virtual Function GetFunction(const std::string_view& name,
                                 const object_p<object_t>& ss) = 0;
};

class Module : public object_r {
public:
    explicit Module(object_p<object_t> n) : object_r(std::move(n)) {}
    
    Function GetFunction(const std::string_view& name,
                         const object_p<object_t>& ss) {
        return operator->()->GetFunction(name, ss);
    }
};
```


### 函数注册表

动态库包含特殊的符号来描述其函数接口：

```cpp
namespace Symbol {
    constexpr const char* ModuleCtx = "__mc_module_ctx";          // 模块上下文
    constexpr const char* FuncRegistry = "__mc_func_registry__";  // 函数注册表
    constexpr const char* ClosuresNames = "__mc_closures_names__"; // 闭包函数名
}
```

**函数注册表格式**：(c_api.h中定义)
```cpp
typedef struct {
    const char* names;     // 编码的函数名称列表
    BackendFunc* funcs;    // 函数指针数组
} FuncRegistry;
```

**函数名称编码解析**

**编码格式**：
```
[函数数量:1字节][函数名1\0][函数名2\0]...[函数名N\0]
```

**解析实现**：
```cpp
static std::vector<std::string_view> ReadFuncRegistryNames(const char* names) {
    std::vector<std::string_view> result;
    if (!names) return result;
    
    // 1. 读取函数数量
    uint8_t num_funcs = static_cast<uint8_t>(names[0]);
    
    // 2. 逐个解析以 null 结尾的字符串
    const char* p = names + 1;
    while (*p && result.size() < num_funcs) {
        size_t len = strlen(p);
        result.push_back(std::string_view(p, len));
        p += len + 1;  // 跳到下一个字符串
    }
    
    return result;
}
```

**构造时预加载**： 把动态库中的函数表__mc_func_registry__转换到func_table_。

```cpp
class LibraryModuleNode final : public ModuleNode {
    object_p<Library> p_;
    std::unordered_map<std::string_view, BackendFunc> func_table_;
    std::unordered_set<std::string_view> closure_names_;

public:
    explicit LibraryModuleNode(object_p<Library> p) : p_(std::move(p)) {
        LoadFunctions();  // 构造时立即加载所有函数
    }

    void LoadFunctions() {
        // 1. 获取函数注册表
        auto* func_reg = reinterpret_cast<FuncRegistry*>(
            p_->GetSymbol("__mc_func_registry__"));
        
        if (!func_reg) {
            throw std::runtime_error("Missing function registry");
        }

        // 2. 解析函数名称
        auto func_names = ReadFuncRegistryNames(func_reg->names);

        // 3. 建立函数名到函数指针的映射
        for (size_t i = 0; i < func_names.size(); ++i) {
            func_table_.emplace(func_names[i], func_reg->funcs[i]);
        }

        // 4. 加载闭包函数名称
        auto* closure_names = reinterpret_cast<const char**>(
            p_->GetSymbol("__mc_closures_names__"));
        
        if (closure_names) {
            auto names = ReadFuncRegistryNames(*closure_names);
            closure_names_.insert(names.begin(), names.end());
        }
    }
};
```

**BackendFunc 到 Function 的转换**：
```cpp
inline Function WrapFunction(BackendFunc func,
                           const object_p<object_t>& sptr_to_self,
                           bool capture_resource = false) {
    return Function([func, sptr_to_self, capture_resource](Parameters args) -> McValue {
        // 转换参数
        std::vector<Value> c_args;
        c_args.reserve(args.size());
        for (int i = 0; i < args.size(); ++i) {
            c_args.push_back(args[i].value());
        }

        // 调用C函数
        Value ret_val;
        void* resource = capture_resource ? sptr_to_self.get() : nullptr;
        if (int ret = (*func)(c_args.data(), c_args.size(), &ret_val, resource); ret != 0) {
            throw std::runtime_error(GetError());
        }

        return McValue(McView(&ret_val));
    });
}
```

**函数查找**：
```cpp
Function GetFunction(const std::string_view& name,
                     const object_p<object_t>& sptr_to_self) override {
    // 1. 首先查找预加载的函数表（O(1)）
    auto it = func_table_.find(name);
    if (it != func_table_.end()) {
        bool is_closure = closure_names_.find(name) != closure_names_.end();
        return WrapFunction(it->second, sptr_to_self, is_closure);
    }
    
    // 2. 动态查找（fallback）
    auto faddr = reinterpret_cast<BackendFunc>(
        p_->GetSymbol(std::string(name).c_str()));
    if (!faddr) {
        return Function();  // 未找到
    }
    
    bool is_closure = closure_names_.find(name) != closure_names_.end();
    return WrapFunction(faddr, sptr_to_self, is_closure);
}
```

### 调用流程

**用户代码**：
```python
# 1. 获取模块加载器
module_loader_ = mc_api.GetGlobal("runtime.ModuleLoader")

# 2. 加载动态库 (ValueToPyObject)
test_module = module_loader_("./test_func.so")

# 3. 获取函数
test_func = test_module.get_function("test_func")

# 4. 调用函数
result = test_func(3, 4)
```

**ModuleLoader 函数实现**：
```cpp
McValue ModuleLoader(Parameters gs) {
    // 1. 创建动态库加载器
    auto n = MakeObject<DefaultLibray>();
    auto name = gs[0].As<const char*>();
    
    // 2. 初始化动态库（调用 dlopen）
    n->Init(name);
    
    // 3. 创建模块节点
    auto m = CreateModuleFromLibrary(n);
    
    return m;  // 返回 Module 对象
}
REGISTER_FUNCTION("runtime.ModuleLoader", ModuleLoader);
```

**CreateModuleFromLibrary 实现**：
```cpp
Module CreateModuleFromLibrary(object_p<Library> p) {
    // 1. 创建库模块节点
    auto n = MakeObject<LibraryModuleNode>(p);
    Module root = Module(n);
    
    // 2. 关键：设置模块上下文
    if (auto* ctx_addr = reinterpret_cast<void**>(
          p->GetSymbol(Symbol::ModuleCtx))) {
        *ctx_addr = root.operator->();  // 让模块能找到自己
    }
    
    return root;
}
```

函数调用的完整链路：分为两个阶段，函数获取和函数调用

**阶段1：函数获取流程**

```
1. Python: test_func = test_module.get_function("test_func")
   ↓
2. ffi.py: Module.get_function() 调用 ctypes
   ↓
3. ctypes: _LIB.GetBackendFunction(ModuleHandle, "test_func", 0, &ret_handle)
   ↓
4. c_api.cc: GetBackendFunction() 类型转换桥梁
   ↓
5. module.cc: LibraryModuleNode::GetFunction() 查找预加载函数
   ↓
6. 函数包装: WrapFunction(test_func__c_api, sptr_to_self, false)
   ↓
7. 返回: Function对象 → new Function(pn) → FunctionHandle
   ↓
8. Python: mc.PackedFuncBase(ret_handle.value)
```

**阶段2：函数调用流程**
```
1. Python: test_func(3, 4)
   ↓
2. case_ext.cc: PyPackedFuncBase_Call() 参数转换
   ↓
3. c_api.cc: FuncCall_PYTHON_C_API() 统一调用接口
   ↓
4. runtime_module.cc: WrapFunction包装的lambda
   ↓
5. test_func.so: test_func__c_api() 动态库函数
   ↓
6. 参数验证 → 类型检查 → 业务逻辑 → 结果设置
   ↓
7. 逆向传播: Value → McValue → Python int
   ↓
8. Python: 接收结果28
```

### FFI 集成

C++端GetBackendFunction，取Module内的函数：

```cpp
int GetBackendFunction(ModuleHandle m, 
                       const char* func_name, 
                       int use_imports,
                       ModuleHandle* out) {
    API_BEGIN();
    
    // 1. 类型转换：ModuleHandle → ModuleNode*
    auto me = static_cast<ModuleNode*>(static_cast<object_t*>(m));
    
    // 2. 调用模块的GetFunction方法
    auto pn = me->GetFunction(func_name, use_imports != 0);
    
    // 3. 创建新的Function对象并返回
    if (pn != nullptr) {
        *out = new Function(pn);  // 在堆上分配
    } else {
        *out = nullptr;
    }
    
    API_END();
}

Python端Module实现：

```python
class Module(object):
    def __init__(self, handle):
        self.handle = handle

    def get_function(self, name):
        ret_handle = PackedFuncHandle()
        status = _LIB.GetBackendFunction(
            ModuleHandle(self.handle),
            c_str(name),
            ctypes.c_int(0),
            ctypes.byref(ret_handle)
        )
        return mc_api.PackedFuncBase(ret_handle.value)

# FFI集成
def _conv(mod: Module):
    handle = mod.handle
    return matx_script_api.make_any(1, 0, handle, 0)

mc_api.RegisterObjectCreator(1, Module)
mc_api.RegisterInputCallback(Module, _conv)
```

交互图：

```
┌─────────────────────────────────────────────────────────────────┐
│                          Python 层                             │
│                                                                 │
│  test_module.get_function("test_func")                         │
│              ↓                                                 │
│  Module.get_function() [new_ffi.py]                           │
│              ↓                                                 │
│  _LIB.GetBackendFunction() [ctypes调用]                       │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                       C API 层                                 │
│                                                                 │
│  GetBackendFunction() [c_api.cc]                               │
│    ├─ ModuleHandle → ModuleNode* 类型转换                      │
│    ├─ me->GetFunction(func_name, use_imports)                  │
│    └─ new Function(pn) → FunctionHandle                       │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                   C++ 运行时层                                 │
│                                                                 │
│  LibraryModuleNode::GetFunction() [runtime_module.cc]         │
│    ├─ func_table_.find(name) 查找预加载函数                   │
│    ├─ WrapFunction(backend_func, sptr_to_self, is_closure)     │
│    └─ 返回 Function 对象                                       │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                      动态库层                                   │
│                                                                 │
│  test_func.so                                                  │
│    ├─ __mc_func_registry__ 函数注册表                         │
│    ├─ test_func__c_api BackendFunc 实现                       │
│    ├─ __mc_module_ctx 模块上下文                              │
│    └─ test_func(int32_t, int32_t) 业务逻辑                    │
└─────────────────────────────────────────────────────────────────┘
```

## Compiler

Python 端的编译流程主要是把Python代表通过 Visitor 模式变成 AST,再把 AST 传给 Rewritier,就生成 C++ 代码了，把 C++ 代码编译成动态库，再把动态库加载调用，就完成了全部流程。

**编译流程**：
```
┌─────────────────────────────────────────────────────────────────┐
│                     Python 源代码                               │
│  def test_func(a: int, b: int) -> int:                           │
│      c = a + b                                                  │
│      return c                                                   │
└─────────────────────────┬───────────────────────────────────────┘
                          │ inspect.getsource()
┌─────────────────────────▼───────────────────────────────────────┐
│                    Python AST                                  │
│  Module(body=[FunctionDef(name='test_func', args=...)])         │
└─────────────────────────┬───────────────────────────────────────┘
                          │ SimpleParser.visit()
┌─────────────────────────▼───────────────────────────────────────┐
│                   MC IR (PrimFunc)                             │
│  PrimFunc(params=[PrimVar('a'), PrimVar('b')],                 │
│           body=SeqStmt([AllocaVarStmt(), ReturnStmt()]))       │
└─────────────────────────┬───────────────────────────────────────┘
                          │ BuildFunction()
┌─────────────────────────▼───────────────────────────────────────┐
│                     C++ 源代码                                 │
│  int32_t test_func(int32_t a, int32_t b) {                      │
│      int32_t c = (a + b);                                      │
│      return c;                                                 │
│  }                                                             │
└─────────────────────────────────────────────────────────────────┘
```

### AST


MC 的 AST 类型系统建立在 Python 的类继承体系之上，通过 `@register_object` 装饰器与 C++ 运行时进行绑定：

```python
class Node(Object):
    """AST 节点基类"""
    def astext(self):
        fn = matx_script_api.GetGlobal("ast.AsText", True)
        text = fn(self)
        return text.decode('utf-8') if isinstance(text, bytes) else text

class Type(Node):
    """类型节点基类"""
    pass

class Expr(Node):
    """表达式节点基类"""
    pass

class Stmt(Node):
    """语句节点基类"""
    pass
```

具体类型定义：

**原始类型**：
```python
prim_type_ = mc_api.GetGlobal("ast.PrimType")
@register_object("PrimType")
class PrimType(Type):
    def __init__(self, datatype):
        self.__init__constructor__(prim_type_, datatype)

# 使用示例
int_type = PrimType("int64")
float_type = PrimType("float32")
```

**表达式类型**：
```python
prim_expr_ = mc_api.GetGlobal("ast.PrimExpr")
@register_object("PrimExpr")
class PrimExpr(BaseExpr):
    def __init__(self, v):
        self.__init__constructor__(prim_expr_, v)

prim_var_ = mc_api.GetGlobal("ast.PrimVar")
@register_object("PrimVar")
class PrimVar(PrimExprWithOp):
    def __init__(self, name, datatype):
        self.__init__constructor__(prim_var_, name, datatype)

# 使用示例
var_x = PrimVar("x", "int64")
const_42 = PrimExpr(42)
```

**语句类型**：
```python
allocavar_stmt_ = mc_api.GetGlobal("ast.AllocaVarStmt")
@register_object("AllocaVarStmt")
class AllocaVarStmt(Stmt):
    def __init__(self, name_hint, type_annotation, init_value=None):
        self.__init__constructor__(allocavar_stmt_, 
                                   name_hint, 
                                   type_annotation, 
                                   init_value)

assign_stmt_ = mc_api.GetGlobal("ast.AssignStmt")
@register_object("AssignStmt")
class AssignStmt(Stmt):
    def __init__(self, lhs, rhs):
        self.__init__constructor__(assign_stmt_, lhs, rhs)

return_stmt_ = mc_api.GetGlobal("ast.ReturnStmt")
@register_object("ReturnStmt")
class ReturnStmt(Stmt):
    def __init__(self, value):
        self.__init__constructor__(return_stmt_, value)
```

**函数类型**：
```python
prim_func_ = mc_api.GetGlobal("ast.PrimFunc")
func_cp_ = mc.GetGlobal("ast.FuncCopy")
with_attr_ = mc.GetGlobal("ast.FuncWithAttr")
@register_object("PrimFunc")
class PrimFunc(BaseFunc):
    def __init__(self, gs, fs, body, rt=None):
        self.__init__constructor__(prim_func_, gs, fs, body, rt)
    
    def with_attr(self, name, value=None):
        """为函数添加属性"""
        res = func_cp_(self)
        return with_attr_(res, Str(name), Str(value))
```

### Parse

`SimpleParser` 是一个基于 Python `ast.NodeVisitor` 的递归下降解析器，将 Python AST 转换为 MC IR：

```python
import ast
import inspect
from scope_context import ScopeContext

class SimpleParser(ast.NodeVisitor):
    _op_maker = {
        ast.Add: lambda lhs, rhs: add(lhs, rhs)
    }
    
    _ty_maker = {
        'int': lambda: PrimType("int64")
    }
    
    def __init__(self):
        self.context = None
        self.functions = {}
```


解析器使用 `ScopeContext` 管理作用域和符号表：

```python
def init_function_parsing_env(self):
    """初始化函数解析环境"""
    self.context = ScopeContext()
    
def visit_FunctionDef(self, node: ast.FunctionDef):
    """函数定义解析"""
    argtypes = []
    argnames = []
    
    # 初始化解析环境
    self.init_function_parsing_env()
    self.context.new_scope(nodes=node.body)
    
    # 处理函数参数
    for arg in node.args.args:
        var_type = self._ty_maker[arg.annotation.id]()
        argtypes.append(var_type)
        argnames.append(arg.arg)
        arg_var = PrimVar(arg.arg, var_type)
        self.context.update_symbol(arg.arg, arg_var)
        self.context.func_params.append(arg_var)
    
    # 处理返回类型
    ret_type = self._ty_maker[node.returns.id]()
    self.context.func_ret_type = ret_type
    
    # 构造函数 IR
    func = PrimFunc(
        Array(self.context.func_params),
        Array([]),
        self.parse_body(),
        ret_type
    )
    
    # 添加函数属性
    func = func.with_attr("GlobalSymbol", node.name)
    self.functions[node.name] = func
    
    self.context.pop_scope()
    return func
```


**变量赋值**：
```python
def visit_Assign(self, node: ast.Assign):
    """赋值语句解析"""
    if len(node.targets) != 1:
        raise RuntimeError('Only one-valued assignment is supported')
    
    lhs_node = node.targets[0]
    rhs_node = node.value
    rhs_value = self.visit(rhs_node)
    lhs_name = lhs_node.id
    
    return self.lookup_or_alloca(lhs_name, rhs_value)

def lookup_or_alloca(self, name_hint, init_value):
    """查找或分配变量"""
    inf_ty = init_value.datatype
    symbol = self.context.lookup_symbol(name_hint)
    
    if symbol is None:
        # 首次使用，创建分配语句
        alloca_stmt = AllocaVarStmt(name_hint, inf_ty, init_value)
        self.context.update_symbol(name_hint, alloca_stmt.var)
        return alloca_stmt
    else:
        # 重新赋值，创建赋值语句
        return AssignStmt(symbol, init_value)
```

**二元运算**：
```python
def visit_BinOp(self, node):
    """二元运算解析"""
    lhs = self.visit(node.left)
    rhs = self.visit(node.right)
    op = self._op_maker[type(node.op)]
    return op(lhs, rhs)
```

**返回语句**：
```python
def visit_Return(self, node):
    """返回语句解析"""
    rt_expr = self.visit(node.value)
    return ReturnStmt(rt_expr)
```


`ScopeContext` 提供了完整的作用域管理机制：

```python
class ScopeContext:
    def __init__(self):
        self.node_stack = []      # AST 节点栈
        self.symbols = []         # 符号表栈
        self.func_params = []     # 函数参数列表
        self.func_ret_type = None # 函数返回类型
    
    def new_scope(self, nodes=None):
        """创建新作用域"""
        if nodes is None:
            nodes = []
        self.node_stack.append(list(reversed(nodes)))
        self.symbols.append(dict())
    
    def update_symbol(self, name, symbol):
        """更新符号表"""
        self.symbols[-1][name] = symbol
    
    def lookup_symbol(self, name):
        """查找符号"""
        for symbols in reversed(self.symbols):
            if name in symbols:
                return symbols[name]
        return None
```

### Compile

`SimpleCompile` 是编译的主要入口点，完成从 Python 函数到 C++ 代码的转换：

```python
def SimpleCompile(target):
    """编译 Python 函数到 C++ 代码"""
    func_name = target.__name__
    
    # 1. 获取 Python 源代码
    source_code = inspect.getsource(target)
    
    # 2. 解析为 Python AST
    ast_tree = ast.parse(source_code)
    
    # 3. 转换为 MC IR
    parser = SimpleParser()
    func_ir = parser.visit(ast_tree)
    
    print(f"Generated IR: {func_ir}")
    print(f"IR type: {type(func_ir)}")
    
    # 4. 生成 C++ 代码
    to_source = mc.GetGlobal("rewriter.BuildFunction")
    code = to_source(func_ir, Str("fn"))
    
    print("Generated C++ code:")
    print(code.data)
```

### 全流程

```python
# 1. 定义要编译的函数
def test_func(a: int, b: int) -> int:
    c = a + b
    return c

# 2. 编译函数
SimpleCompile(test_add, "test_func.so")

# 3. 加载编译后的模块
module_loader_ = mc.GetGlobal("runtime.ModuleLoader")
test_module = module_loader_("./test_func.so")

# 4. 获取函数并调用
test_func = test_module.get_function("test_func")
result = test_func(3, 4)
print(f"Result: {result}")
```

至此，这个Python -> C++的编译器全部讲述完毕，采用的是自底向上的方法，Object->Container->Ast->Visitor->Value->Function->FFI，全部七个部分。


