# TODO

这个文档记录当前在 `Zero` 代码库里已经被最小复现测试确认的问题，暂不直接修复，等进一步人工确认后再处理。

## 1. Broadcast Backward 梯度错误

### 现象

广播场景下的二元算子反向传播结果不正确。

当前新增的最小断言测试位于：

- `apps/test.cc`
- `test_broadcast_backward_assert()`

运行后得到的失败信息：

```text
Broadcast backward assert failed: broadcast x_grad: mismatch at index 0, actual=1.000000, expected=2.000000
```

### 复现用例

测试构造：

- `x = ones([2, 3])`, `requires_grad=true`
- `y = [2, 3, 4]`, shape `[3]`, `requires_grad=true`
- `z = x * y`
- `z.backward()`

期望：

- `x.grad = [[2, 3, 4], [2, 3, 4]]`
- `y.grad = [2, 2, 2]`

实际：

- `x.grad` 不符合期望

### 优先排查位置

- `src/ops.cc`
- `OpAdd::Backward_`
- `OpSub::Backward_`
- `OpMul::Backward_`
- `OpDiv::Backward_`

重点检查广播分支下返回梯度的顺序是否和原始输入槽位一致。

## 2. Transpose 后 MatMul 数值错误

### 现象

`Transpose()` 本身只是修改 metadata，但当前 `MatMul` 对转置后的 tensor 没有给出正确数值结果。

当前新增的最小断言测试位于：

- `apps/test.cc`
- `test_transpose_matmul_assert()`

运行后得到的失败信息：

```text
Transpose matmul assert failed: transpose matmul: mismatch at index 0, actual=25.000000, expected=43.000000
```

### 复现用例

测试构造：

- `a = [[1,2,3],[4,5,6]]`, shape `[2,3]`
- `b = [[7,8],[9,10]]`, shape `[2,2]`
- `c = a.T().MatMul(b)`

期望：

```text
[[43, 48],
 [59, 66],
 [75, 84]]
```

实际：

- 第一项已经不一致，`actual=25`, `expected=43`

### 初步判断

这不代表 `Transpose()` 只改 metadata 有问题。问题更可能在于：

- 转置后的 tensor 变成了非连续布局
- 后续 kernel 仍按连续内存方式读取
- `MatMul` 没有正确消费 stride / layout 信息

### 优先排查位置

- `src/tensor.cc`
- `Tensor::Transpose`
- `src/tensor_ops_cpu.cc`
- `matmul_cpu`
- `matmul_backward_cpu`

## 3. 现有测试的覆盖方式偏“打印结果”

### 现象

当前 `apps/test.cc` 中很多测试虽然能跑通，但主要是打印前向值和梯度，没有断言 expected result。

这会导致：

- 程序运行成功
- 但数值结果错误时不容易第一时间发现

### 建议

后续补充更多最小断言测试，至少覆盖：

- broadcast forward/backward
- transpose/view 后的算子正确性
- attention 中 `View + Transpose + MatMul` 组合路径
- optimizer step 是否在 no-grad 语义下执行

## 4. 当前状态说明

## 5. 待进一步确认的问题

下面两项是代码审阅时发现的风险点，但本轮还没有像前两项那样补最小复现断言测试，因此先标记为“待确认”。

### 5.1 AdamW 参数更新可能被错误纳入计算图

### 现象

`AdamW::Step()` 里的参数更新路径看起来没有显式关闭 grad tracking。

对比：

- `Parameter::Update()` 使用的是 `(*op)({...}, true)`，也就是显式 `no_grad=true`
- `AdamW::Step()` 里构造 update 和最终 `*param = (*op_sub)(inputs)` 时，没有看到同样的 no-grad 保护

### 风险

如果这里确实在有梯度追踪的语义下执行参数更新，可能会导致：

- 参数更新步骤本身进入 autograd graph
- 多轮训练时图持续增长
- 训练语义偏离通常期望的 optimizer no-grad 行为

### 优先排查位置

- `src/optimizer.cc`
- `src/parameter.h`

### 建议的后续验证

补一个最小训练循环测试，检查：

- 连续多次 `optimizer.step()` 后 graph 是否异常增长
- 参数更新结果是否携带不应该存在的反向依赖

### 5.2 simple_test 没有真正验证梯度链路

### 现象

`apps/simple_test.cc` 中的 `loss` 是通过读取 `d.Data<float>()` 后手工求和，再重新构造一个新的 `Tensor` 得到的。

这一步会把原本的计算图断开：

- `c = a + b`
- `d = c * c`
- 然后不是直接对 `d` 做图内归约
- 而是把 `d` 的数据拷出来手工求和，再新建 `loss`

### 风险

这意味着当前 `simple_test` 里的：

```cpp
loss.Backward();
```

并不能证明：

- `a` 的梯度是正确的
- `b` 的梯度是正确的
- `a -> c -> d -> loss` 这条链路真的打通了

它更像是在验证：

- 可以创建一个新的标量 tensor
- 这个新 tensor 本身可以 `Backward()`

### 优先排查位置

- `apps/simple_test.cc`

### 建议的后续验证

补一个真正图内的标量 loss 测试，例如：

- 实现 `sum` / `mean` 类型的归约 op
- 直接对图中的结果做归约后 `Backward()`
- 检查 `a.grad` / `b.grad` 是否符合预期

### 已保留但未提交处理的改动

- `apps/test.cc` 中新增了两个最小断言测试：
  - `test_broadcast_backward_assert()`
  - `test_transpose_matmul_assert()`

这些测试目前保留，供后续继续定位问题。

### 本轮使用过的验证命令

```bash
cmake --build build --target test
./build/apps/test
```
