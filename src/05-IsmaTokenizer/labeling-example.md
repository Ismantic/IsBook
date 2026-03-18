# BytePieceCounter标注法完整示例

让我们用简化的文本"南京"来演示完整的标注法过程。

## 示例设置

```
文本："南京"
UTF-8字节：[E5, 8D, 97, E4, BA, AC] (6个字节)
max_piece_count_ = 3 (状态0,1,2)
```

## 第一步：UTF-8边界预处理

```cpp
utf8_position = [0, 1, 2, 0, 1, 2]

解释：
位置0,1,2：属于字符"南"，分别是第1,2,3字节 (utf8_position = 0,1,2)
位置3,4,5：属于字符"京"，分别是第1,2,3字节 (utf8_position = 0,1,2)
```

## 第二步：状态转移矩阵

```cpp
T矩阵 (3×3)：
     → 0  1  2
从 0   -∞ 0  -∞
   1   0  -∞ 0
   2   0  -∞ 0

转移规则：
- 任何状态都可以转移到状态0 (开始新token)
- 状态i只能转移到状态i+1 (token继续增长)
- 最高状态2可以自环 (支持长token)
```

## 第三步：节点评分填充（获取转移分数）

### 核心概念

nodes[i][j]表示**在位置i处于状态j时，消费当前字符/字节的得分**。这个得分来自于N-gram统计，表示从某个起始位置到当前位置i的子串的概率。

```cpp
nodes[i][j] = 当前位置i，状态j对应的N-gram概率
其中：状态j表示当前token长度为j+1
     对应的N-gram为：text.substr(i-j, j+1)
```

### 状态与N-gram的对应关系

```
状态j=0: 当前token长度=1，对应1-gram，使用N_[1]统计
状态j=1: 当前token长度=2，对应2-gram，使用N_[2]统计  
状态j=2: 当前token长度=3，对应3-gram，使用N_[3]统计
...

例如：在位置2，状态2
→ 当前token长度=3
→ 对应的N-gram = text.substr(2-2, 2+1) = text.substr(0, 3)
→ 查找N_[3]中对应子串的概率
```

### 转移得分的计算

动态规划的状态转移得分由三部分组成：
```cpp
total_score = nodes[i-1][prev_j] + T_[prev_j][curr_j] + nodes[i][curr_j]
            = 前一位置的累积得分 + 状态转移代价 + 当前N-gram得分
```

其中：
- `nodes[i-1][prev_j]`：到达前一位置的最优累积得分
- `T_[prev_j][curr_j]`：从状态prev_j转移到状态curr_j的代价（通常为0）
- `nodes[i][curr_j]`：当前位置消费字符的N-gram得分

### 假设N-gram统计结果

```cpp
N_[1]: {"E5": -1.0, "8D": -2.0, "97": -2.0, "E4": -1.0, "BA": -2.0, "AC": -2.0}
N_[2]: {"E58D": -0.5, "8D97": -1.0, "97E4": -1.5, "E4BA": -0.5, "BAAC": -1.0}
N_[3]: {"E58D97": -0.2, "97E4BA": -1.2, "E4BAAC": -0.2}
```

### 填充过程详解

#### 位置0 (utf8_position[0]=0)：
```
j=0: 约束检查 0≥0 ✓ 
     → 当前token长度=1，N-gram="E5"
     → 查找N_[1]["E5"] = -1.0
     → nodes[0][0] = -1.0
     
j=1,2: i-j<0，越界跳过
```

#### 位置1 (utf8_position[1]=1)：
```
j=0: 约束检查 0≥1 ✗ → continue 
     → nodes[1][0] = -INF (UTF-8约束：不能从字符中间开始新token)
     
j=1: 约束检查 1≥1 ✓
     → 当前token长度=2，N-gram=text.substr(1-1, 1+1)="E58D"
     → 查找N_[2]["E58D"] = -0.5
     → nodes[1][1] = -0.5
     
j=2: i-j<0，越界跳过
```

#### 位置2 (utf8_position[2]=2)：
```
j=0: 约束检查 0≥2 ✗ → continue → nodes[2][0] = -INF
j=1: 约束检查 1≥2 ✗ → continue → nodes[2][1] = -INF
j=2: 约束检查 2≥2 ✓
     → 当前token长度=3，N-gram=text.substr(2-2, 2+1)="E58D97"
     → 查找N_[3]["E58D97"] = -0.2
     → nodes[2][2] = -0.2
```

#### 位置3 (utf8_position[3]=0)：
```
j=0: 约束检查 0≥0 ✓
     → 当前token长度=1，N-gram="E4"
     → 查找N_[1]["E4"] = -1.0
     → nodes[3][0] = -1.0

j=1: 约束检查 1≥0 ✓
     → 当前token长度=2，N-gram=text.substr(3-1, 1+1)="97E4"
     → 查找N_[2]["97E4"] = -1.5
     → nodes[3][1] = -1.5

j=2: 约束检查 2≥0 ✓
     → 当前token长度=3，N-gram=text.substr(3-2, 2+1)="8D97E4"
     → N-gram边界检查：ngram_start=3-2=1，utf8_position[1]=1≠0 ✗
     → continue → nodes[3][2] = -INF
     (理由：N-gram "8D97E4"从UTF-8字符中间开始，违反语义完整性)
```

#### 位置4 (utf8_position[4]=1)：
```
j=0: 约束检查 0≥1 ✗ → continue → nodes[4][0] = -INF
j=1: 约束检查 1≥1 ✓
     → 当前token长度=2，N-gram="E4BA"
     → 查找N_[2]["E4BA"] = -0.5
     → nodes[4][1] = -0.5
j=2: 约束检查 2≥1 ✓
     → 当前token长度=3，N-gram="97E4BA"
     → N-gram边界检查：ngram_start=2，utf8_position[2]=2≠0 ✗
     → continue → nodes[4][2] = -INF
```

#### 位置5 (utf8_position[5]=2)：
```
j=0: 约束检查 0≥2 ✗ → continue → nodes[5][0] = -INF
j=1: 约束检查 1≥2 ✗ → continue → nodes[5][1] = -INF
j=2: 约束检查 2≥2 ✓
     → 当前token长度=3，N-gram="E4BAAC"
     → 查找N_[3]["E4BAAC"] = -0.2
     → nodes[5][2] = -0.2
```

### 最终nodes矩阵（转移得分表）

```
      位置: 0    1     2     3     4     5
    状态0: -1.0 -INF  -INF  -1.0  -INF  -INF
    状态1:      -0.5  -INF  -1.5  -0.5  -INF
    状态2:            -0.2        -INF  -0.2

解释：
- nodes[0][0]=-1.0：位置0状态0，消费1-gram "E5"的得分
- nodes[1][1]=-0.5：位置1状态1，消费2-gram "E58D"的得分
- nodes[2][2]=-0.2：位置2状态2，消费3-gram "E58D97"的得分
- -INF：表示由于UTF-8约束而被禁止的状态
```

### 关键洞察

1. **nodes矩阵存储的是局部转移得分**：每个有效的nodes[i][j]都对应一个特定长度的N-gram及其概率

2. **UTF-8约束过滤不合理状态**：通过设置-INF确保算法不会考虑破坏字符完整性的路径

3. **为动态规划提供基础**：这些转移得分将在后续的DP过程中用于计算最优路径

4. **语义完整性保证**：N-gram边界检查确保统计的子串不会跨越UTF-8字符边界

## 第四步：动态规划状态转移

### 初始化
```cpp
routes矩阵初始化为0
从位置1开始向前转移
```

### 位置1→位置2：
```
curr_j=0: 约束检查 0≥utf8_position[2]=2 ✗ → continue
curr_j=1: 约束检查 1≥utf8_position[2]=2 ✗ → continue  
curr_j=2: 约束检查 2≥utf8_position[2]=2 ✓
  prev_j=0: 约束检查 0≥utf8_position[1]=1 ✗ → continue
  prev_j=1: 约束检查 1≥utf8_position[1]=1 ✓
    转移检查：T[1][2]=0 ✓
    score = nodes[1][1] + T[1][2] + nodes[2][2] = -0.5 + 0 + (-0.2) = -0.7
    routes[1][2] = 1，更新 nodes[2][2] = -0.7
  prev_j=2: 越界跳过
```

### 位置2→位置3：
```
curr_j=0: 约束检查 0≥utf8_position[3]=0 ✓
  prev_j=0: 约束检查 0≥utf8_position[2]=2 ✗ → continue
  prev_j=1: 约束检查 1≥utf8_position[2]=2 ✗ → continue
  prev_j=2: 约束检查 2≥utf8_position[2]=2 ✓
    转移检查：T[2][0]=0 ✓
    score = nodes[2][2] + T[2][0] + nodes[3][0] = -0.7 + 0 + (-1.0) = -1.7
    routes[2][0] = 2，更新 nodes[3][0] = -1.7

curr_j=1: 约束检查 1≥utf8_position[3]=0 ✓
  prev_j=2: 约束检查 2≥utf8_position[2]=2 ✓
    转移检查：T[2][1]=-INF ✗ → continue (状态2不能直接转移到状态1)

curr_j=2: 约束检查 2≥utf8_position[3]=0 ✓
  N-gram边界检查：ngram_start=3-2=1，utf8_position[1]=1≠0 ✗ → continue
```

### 位置3→位置4：
```
curr_j=0: 约束检查 0≥utf8_position[4]=1 ✗ → continue
curr_j=1: 约束检查 1≥utf8_position[4]=1 ✓
  prev_j=0: 约束检查 0≥utf8_position[3]=0 ✓
    转移检查：T[0][1]=0 ✓
    score = nodes[3][0] + T[0][1] + nodes[4][1] = -1.7 + 0 + (-0.5) = -2.2
    routes[3][1] = 0，更新 nodes[4][1] = -2.2
  prev_j=1: 约束检查 1≥utf8_position[3]=0 ✓
    转移检查：T[1][1]=-INF ✗ → continue

curr_j=2: N-gram边界检查失败 → continue
```

### 位置4→位置5：
```
curr_j=0: 约束检查 0≥utf8_position[5]=2 ✗ → continue
curr_j=1: 约束检查 1≥utf8_position[5]=2 ✗ → continue
curr_j=2: 约束检查 2≥utf8_position[5]=2 ✓
  prev_j=0: 约束检查 0≥utf8_position[4]=1 ✗ → continue
  prev_j=1: 约束检查 1≥utf8_position[4]=1 ✓
    转移检查：T[1][2]=0 ✓
    score = nodes[4][1] + T[1][2] + nodes[5][2] = -2.2 + 0 + (-0.2) = -2.4
    routes[4][2] = 1，更新 nodes[5][2] = -2.4
  prev_j=2: 约束检查失败 → continue
```

**最终状态：**
```
      位置: 0    1     2     3     4     5
    状态0: -1.0 -INF  -INF  -1.7  -INF  -INF
    状态1:      -0.5  -INF  -INF  -2.2  -INF
    状态2:            -0.7        -INF  -2.4

routes矩阵：
      位置: 1  2  3  4
    状态0:    0  2  0
    状态1:    0  0  0  
    状态2:    0  1  0  1
```

## 第五步：回溯最优路径

### 寻找最佳终止状态
```cpp
位置5的有效状态检查：
- 状态0: 0≥utf8_position[5]=2 ✗ 无效
- 状态1: 1≥utf8_position[5]=2 ✗ 无效  
- 状态2: 2≥utf8_position[5]=2 ✓ 有效，得分-2.4

最佳终止状态：状态2，得分-2.4
```

### 回溯过程
```cpp
curr_pos=5, curr_state=2
opt_route[5] = 2
curr_state = routes[4][2] = 1, curr_pos = 4

curr_pos=4, curr_state=1  
opt_route[4] = 1
curr_state = routes[3][1] = 0, curr_pos = 3

curr_pos=3, curr_state=0
opt_route[3] = 0
curr_state = routes[2][0] = 2, curr_pos = 2

curr_pos=2, curr_state=2
opt_route[2] = 2
curr_state = routes[1][2] = 1, curr_pos = 1

curr_pos=1, curr_state=1
opt_route[1] = 1
curr_pos = 0 → 结束

最终路径：opt_route = [?, 1, 2, 0, 1, 2]
```

### 路径解读
```
位置: 0  1  2  3  4  5
状态: ?  1  2  0  1  2
含义: ?  ┗━━━┛  ┗━━━┛
      ?  token1   token2

token1: 从位置0开始，在位置3结束(状态0表示token结束)
token2: 从位置3开始，在位置6结束
```

## 第六步：提取tokens

### 切分点识别
```cpp
split_points = [0]  // 起始点

for i in [1,2,3,4,5]:
  if opt_route[i] == 0 && utf8_position[i] == 0:
    split_points.add(i)

检查过程：
i=1: opt_route[1]=1 ≠ 0 ✗
i=2: opt_route[2]=2 ≠ 0 ✗
i=3: opt_route[3]=0 ✓ && utf8_position[3]=0 ✓ → 添加切分点3
i=4: opt_route[4]=1 ≠ 0 ✗
i=5: opt_route[5]=2 ≠ 0 ✗

split_points.add(6)  // 结束点

最终切分点：split_points = [0, 3, 6]
```

### 提取最终tokens
```cpp
token1 = text.substr(0, 3-0) = text.substr(0, 3) = "南" (字节E58D97)
token2 = text.substr(3, 6-3) = text.substr(3, 3) = "京" (字节E4BAAC)

最终分词结果：["南", "京"]
总得分：-2.4
```

## 算法总结

这个完整示例展示了BytePieceCounter标注法的核心机制：

1. **UTF-8约束保证字符完整性**：通过utf8_position数组确保token不会从字符中间开始
2. **状态空间动态规划**：在复杂的约束条件下寻找最优分词路径
3. **N-gram边界检查**：确保统计信息的语义连贯性
4. **全局最优解**：通过动态规划保证找到概率最大的分词方案

最终结果["南", "京"]完美体现了算法在字节级统计基础上实现字符级分词的能力。