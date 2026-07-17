# Unicode 与 UTF-8

这一章先不谈模型，也不谈分词，而是先回答一个更基本的问题：文本在机器里到底是什么。

这看起来像一个很“底层”的问题，但它其实会一路影响后面的所有章节。因为无论是正则、Trie、分词还是 tokenizer，本质上都建立在同一前提上：你必须先知道自己处理的对象到底是字符、码点，还是字节序列。

很多实现上的混乱，最后都能追溯到这里的几个概念没有分清：

- 以为“一个字符”总是对应一个字节
- 把字符串长度和字节长度混为一谈
- 把 Unicode 和 UTF-8 当成同一个东西
- 没有意识到“文本表示”本身就是一层协议

这一章的目标，就是先把这些概念理顺。只有把文本真正看成“带结构的字节序列”，后面的内容才会自然。

## 本章导读

这一章分成两部分：

1. 先把 Unicode、码点和 UTF-8 的关系讲清楚
2. 再通过几个小函数把 UTF-8 的编解码过程拆开来看

如果你以前已经“会用 UTF-8”，但总觉得这些概念是模糊记住的，那么这一章就是为这种情况准备的。

## 1. 基础概念

### 1.1 Unicode 是什么？

Unicode 的核心目标，是给全世界各种书写系统里的字符分配统一的编号。这个编号通常叫做码点（code point）。

可以先把它理解成一张巨大的映射表：每个字符，都对应一个确定的数值。

例如：

- **英文**：`'a'` → 97, `'b'` → 98, `'c'` → 99, ...
- **中文**：`'你'` → 20320, `'我'` → 25105, `'他'` → 20182, ...  
- **Emoji**：`'😀'` → 128512, `'🌍'` → 127757, ...

但这里有一个非常重要的区分：

- **Unicode** 解决的是“字符编号”的问题
- 它本身并不直接规定“这些编号在内存里怎样存”

也就是说，Unicode 首先是一套字符空间，而不是一种具体的字节编码格式。

目前 Unicode 的有效范围是 `0` 到 `0x10FFFF`，大约 110 多万个可用码点。实际已经登记使用的字符远没有把这个空间用满，但这个范围已经足够覆盖绝大多数现代文本场景。

### 1.2 为什么需要 UTF-8？

既然 Unicode 已经给字符分配了统一编号，下一步问题就是：这些编号怎样真正落到字节序列里？

一个最直接的想法是：每个字符都用 `uint32_t` 存，也就是统一用 4 字节表示。这样当然可以，但会很浪费。

原因至少有三点：

1. **存储效率**：英文字符非常常见，但 ASCII 范围内的字符只需要 7 位
2. **兼容性**：现实世界里早就存在大量基于 ASCII 的文本和协议
3. **传输成本**：文本越短，存储和网络传输越便宜

UTF-8 就是在这种背景下被广泛采用的。它不是新的字符集，而是一种把 Unicode 码点编码成字节序列的方法。

它最关键的特点是**变长编码**：

- ASCII 范围内的字符仍然只用 1 个字节
- 更大的码点使用 2 到 4 个字节表示

这让 UTF-8 既兼顾了英语世界的存储效率，也兼顾了对全球文字的统一支持。

### 1.3 UTF-8 编码规则

UTF-8 使用 1 到 4 个字节来表示一个 Unicode 码点。为了让解码过程不混乱，它对每个字节的高位形式做了严格约束。

#### 编码格式
```
1字节：0xxxxxxx
2字节：110xxxxx 10xxxxxx
3字节：1110xxxx 10xxxxxx 10xxxxxx
4字节：11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```

这里的关键思想是：

- 首字节负责告诉你“这是几字节编码”
- 后续字节统一以 `10` 开头，表示“我是续字节”

#### 解码规则

从 UTF-8 字节序列提取 Unicode 值时，本质上就是把所有标记位去掉，再把有效位拼起来：

```
1字节：0xxxxxxx → xxxxxxx
2字节：110xxxxx 10xxxxxx → xxxxx xxxxxx
3字节：1110xxxx 10xxxxxx 10xxxxxx → xxxx xxxxxx xxxxxx  
4字节：11110xxx 10xxxxxx 10xxxxxx 10xxxxxx → xxx xxxxxx xxxxxx xxxxxx
```

#### 容量分配
- **1字节**：7位 → 128种字符（0-127）- ASCII兼容
- **2字节**：5+6=11位 → 2048种字符（128-2047）
- **3字节**：4+6+6=16位 → 65536种字符（2048-65535）
- **4字节**：3+6+6+6=21位 → 2097152种字符（65536-1114111）

到这里可以先记住一条非常实用的结论：

> Unicode 关心“字符对应哪个码点”，UTF-8 关心“码点怎样编码成字节”。

这条区分在后面的实现里会一直用到。

### 1.4 三个最容易混淆的概念

到这里，至少还要把下面三组概念彻底分开。很多文本处理 bug，并不是算法本身出了问题，而是这里先混了。

#### 码点长度，不等于字节长度

以 `"你"` 为例：

- 从 Unicode 的角度看，它对应 1 个码点
- 从 UTF-8 的角度看，它通常占 3 个字节

所以：

- 如果你在做字符级处理，它是 1 个单位
- 如果你在做底层存储或扫描，它是 3 个单位

这也是为什么很多语言里的字符串长度 API，如果没有额外说明，往往不能直接当“字符数”理解。

#### 字符串，不等于裸字节数组

在实现层面，字符串最终确实表现为一串字节。但“字符串”不是“随便一段字节”这么简单，它还隐含着一个解释规则。

同一串字节：

- 在 UTF-8 下可能能被正确解释
- 换一种编码假设，就可能完全失真

所以在工程里，字符串处理永远有两层：

- 存储层：一串字节
- 解释层：这些字节按什么规则被还原成字符

#### ASCII 兼容，不等于 ASCII 思维仍然成立

UTF-8 的一个巨大优势是兼容 ASCII：

- ASCII 范围内的字符仍然只占 1 个字节
- 并且字节值完全不变

但这不意味着你可以继续用“一个字节就是一个字符”的思路处理所有文本。

只要文本里出现中文、日文、Emoji 或任何非 ASCII 字符，这种假设就立刻失效。

### 1.5 一个很实用的判断问题

实际写代码时，可以先问自己一个问题：

> 你现在处理的是“字节”，还是“字符”？

这个问题非常朴素，但它会直接决定后面的实现方式。

例如：

- **按字节处理**：更适合做编码验证、文件读写、底层扫描
- **按字符/码点处理**：更适合做分词、字符类别判断、显示长度统计

而很多 NLP 系统之所以复杂，就是因为它们必须在这两层之间不断切换：

- 输入通常是字节流
- 中间必须识别字符边界
- 更上层还要做 token 切分和语义建模

## 2. 编程实现

理解完概念之后，再进入实现部分。

这里的目标不是写一个“功能尽可能全”的库，而是通过几个小函数，把 UTF-8 的编解码过程真正拆开来看。这样后面无论是自己写字符串处理工具，还是阅读 tokenizer / 分词相关代码，都会更有把握。

### 2.1 工具函数：PrintBinary

在实现 UTF-8 编解码函数之前，先准备一个二进制打印工具。因为后面会频繁使用掩码和移位操作，只看十六进制有时不够直观。

```cpp
void PrintBinary(uint32_t num) {
    std::cout << "Number: " << std::dec << num << std::endl;
    std::cout << "Hex: 0x" << std::hex << std::uppercase << num << std::endl;
    std::cout << "Oct: " << std::oct << num << std::endl;
    std::cout << "Binary: " << std::bitset<8>(num) << std::endl;  // 改为8位
    std::cout << std::endl;
}
```

### 2.2 关键数值表格

先把 UTF-8 编解码里最常用的一批常量打印出来，形成一个参考表格。

```cpp
void PrintBinary(uint32_t num) {
    std::cout << "Number: " << std::dec << num << std::endl;
    std::cout << "Hex: 0x" << std::hex << std::uppercase << num << std::endl;
    std::cout << "Binary: " << std::bitset<8>(num) << std::endl;
    std::cout << std::endl;
}
```

然后打印所有在Decode/Encode函数中用到的关键数值：

```cpp
// UTF-8格式检测掩码
PrintBinary(0x80);  // 0x80: 10000000 - 续字节标识
PrintBinary(0xC0);  // 0xC0: 11000000 - 2字节首字节前缀
PrintBinary(0xE0);  // 0xE0: 11100000 - 3字节首字节前缀  
PrintBinary(0xF0);  // 0xF0: 11110000 - 4字节首字节前缀

// 格式检测掩码
PrintBinary(0xE0);  // 0xE0: 11100000 - 检测2字节首字节的掩码
PrintBinary(0xF0);  // 0xF0: 11110000 - 检测3字节首字节的掩码
PrintBinary(0xF8);  // 0xF8: 11111000 - 检测4字节首字节的掩码

// 数据提取掩码
PrintBinary(0x1F);  // 0x1F: 00011111 - 2字节首字节数据掩码 (5位)
PrintBinary(0x0F);  // 0x0F: 00001111 - 3字节首字节数据掩码 (4位)
PrintBinary(0x07);  // 0x07: 00000111 - 4字节首字节数据掩码 (3位)
PrintBinary(0x3F);  // 0x3F: 00111111 - 续字节数据掩码 (6位)

// 1字节判断
PrintBinary(0x7F);  // 0x7F: 01111111 - 1字节最大值 (127)
```

**完整输出结果：**
```
Number: 128   Hex: 0x80   Binary: 10000000
Number: 192   Hex: 0xC0   Binary: 11000000
Number: 224   Hex: 0xE0   Binary: 11100000
Number: 240   Hex: 0xF0   Binary: 11110000
Number: 224   Hex: 0xE0   Binary: 11100000
Number: 240   Hex: 0xF0   Binary: 11110000
Number: 248   Hex: 0xF8   Binary: 11111000
Number: 31    Hex: 0x1F   Binary: 00011111
Number: 15    Hex: 0x0F   Binary: 00001111
Number: 7     Hex: 0x07   Binary: 00000111
Number: 63    Hex: 0x3F   Binary: 00111111
Number: 127   Hex: 0x7F   Binary: 01111111
```

这个表格涵盖了后续所有函数实现中会反复出现的常量。把它们先看明白，后面读位运算时就不会总处在“知道代码在判断，但不知道判断的究竟是什么”的状态。

### 2.3 函数1：`IsTrailByte`，判断续字节

先实现一个小函数，用来判断某个字节是不是 UTF-8 的续字节。根据前面的规则，只要一个字节以 `10` 开头，它就是续字节。

```cpp
bool IsTrailByte(uint8_t x) {
    return (x & 0xC0) == 0x80;
}
```

**实现原理**
- `0xC0`（11000000）作为掩码，提取字节的最高2位
- `0x80`（10000000）是续字节的标准模式
- 通过位与操作`&`提取高2位，然后与`0x80`比较

这个函数能准确识别所有续字节，也就是 `0x80` 到 `0xBF` 这 64 种情况。

### 2.4 函数2：`DecodeOneUTF8`，解码单个字符

接下来实现解码单个字符的核心函数。它从字符串起始位置读取一个 UTF-8 字符，并返回两样东西：

- 这个字符对应的 Unicode 码点
- 这次一共消耗了多少个字节

之所以这两个返回值缺一不可，是因为 UTF-8 是变长编码。你不只要知道“解出来是什么”，还要知道“下一次该从哪里继续读”。

```cpp
uint32_t DecodeOneUTF8(const std::string& str, size_t* bytes) {
    if (str.empty()) {
        *bytes = 0;
        return 0;
    }
    
    const uint8_t* data = reinterpret_cast<const uint8_t*>(str.data());
    const size_t size = str.size();
    
    // 1字节：0xxxxxxx
    if (data[0] < 0x80) {  // 0x80: 10000000
        *bytes = 1;
        return data[0];
    }
    
    // 2字节：110xxxxx 10xxxxxx
    else if ((data[0] & 0xE0) == 0xC0 && size >= 2 &&  // 0xE0: 11100000, 0xC0: 11000000
             IsTrailByte(data[1])) {
        const uint32_t codepoint = ((data[0] & 0x1F) << 6) |  // 0x1F: 00011111
                                   (data[1] & 0x3F);          // 0x3F: 00111111
        *bytes = 2;
        return codepoint;
    }
    
    // 3字节：1110xxxx 10xxxxxx 10xxxxxx
    else if ((data[0] & 0xF0) == 0xE0 && size >= 3 &&  // 0xF0: 11110000, 0xE0: 11100000
             IsTrailByte(data[1]) && IsTrailByte(data[2])) {
        const uint32_t codepoint = ((data[0] & 0x0F) << 12) |  // 0x0F: 00001111
                                   ((data[1] & 0x3F) << 6) |   // 0x3F: 00111111
                                   (data[2] & 0x3F);           // 0x3F: 00111111
        *bytes = 3;
        return codepoint;
    }
    
    // 4字节：11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
    else if ((data[0] & 0xF8) == 0xF0 && size >= 4 &&  // 0xF8: 11111000, 0xF0: 11110000
             IsTrailByte(data[1]) && IsTrailByte(data[2]) && IsTrailByte(data[3])) {
        const uint32_t codepoint = ((data[0] & 0x07) << 18) |  // 0x07: 00000111
                                   ((data[1] & 0x3F) << 12) |  // 0x3F: 00111111
                                   ((data[2] & 0x3F) << 6) |   // 0x3F: 00111111
                                   (data[3] & 0x3F);           // 0x3F: 00111111
        *bytes = 4;
        return codepoint;
    }
    
    // 异常情况
    *bytes = 0;
    return 0;
}
```

**实现原理**

以2字节情况为例说明关键技术：

1. **格式验证**：`(data[0] & 0xE0) == 0xC0` 检查首字节前3位是否为110  // 0xE0: 11100000, 0xC0: 11000000
2. **续字节验证**：使用`IsTrailByte`确保后续字节格式正确
3. **数据提取**：
   - `data[0] & 0x1F`：提取首字节的低5位数据（掩码00011111）  // 0x1F: 00011111
   - `data[1] & 0x3F`：提取续字节的低6位数据（掩码00111111）   // 0x3F: 00111111
4. **位组合**：`(首字节数据 << 6) | 续字节数据` 将5位和6位数据拼接

3字节和4字节的处理逻辑类似，只是掩码和移位数不同。

### 2.5 函数3：`DecodeUTF8`，解码整个字符串

有了解码单个字符的函数，整个字符串的解码就只是一个循环过程：每次解出一个码点，按实际消耗的字节数把位置向前推进。

```cpp
std::vector<uint32_t> DecodeUTF8(const std::string& str) {
    std::vector<uint32_t> codepoints;
    
    size_t pos = 0;
    while (pos < str.size()) {
        size_t bytes_consumed;
        
        std::string substr = str.substr(pos);
        uint32_t codepoint = DecodeOneUTF8(substr, &bytes_consumed);
        
        if (bytes_consumed == 0) {
            break;  // 遇到无法解码的序列，停止处理
        }
        
        codepoints.push_back(codepoint);
        pos += bytes_consumed;
    }
    
    return codepoints;
}
```

这个函数通过循环调用 `DecodeOneUTF8`，逐个字符解码整个字符串，最终返回 Unicode 码点数组。

### 2.6 函数4：`EncodeOneUTF8`，编码单个字符

编码函数基本上是解码过程的镜像：把 Unicode 码点重新拆成 1 到 4 个 UTF-8 字节。

这里最值得注意的一点是：实现通常不是“顺着把字节写出去”，而是先不断提取低位数据，再从输出缓冲区的后面往前填。这正好贴合 UTF-8 每个续字节承载 6 位数据的结构。

```cpp
size_t EncodeOneUTF8(uint32_t c, char* output) {
    if (c <= 0x7F) {  // 0x7F: 01111111
        // 1字节：0xxxxxxx
        *output = static_cast<char>(c);
        return 1;
    }
    if (c <= 0x7FF) {  // 0x7FF: 011111111111
        // 2字节：110xxxxx 10xxxxxx
        output[1] = 0x80 | (c & 0x3F);         // 0x80: 10000000, 0x3F: 00111111 - 低6位 + 续字节前缀
        c >>= 6;
        output[0] = 0xC0 | c;                  // 0xC0: 11000000 - 高5位 + 首字节前缀
        return 2;
    }
    if (c <= 0xFFFF) {  // 0xFFFF: 1111111111111111
        // 3字节：1110xxxx 10xxxxxx 10xxxxxx
        output[2] = 0x80 | (c & 0x3F);         // 0x80: 10000000, 0x3F: 00111111 - 最低6位
        c >>= 6;
        output[1] = 0x80 | (c & 0x3F);         // 0x80: 10000000, 0x3F: 00111111 - 中间6位
        c >>= 6;
        output[0] = 0xE0 | c;                  // 0xE0: 11100000 - 最高4位 + 首字节前缀
        return 3;
    }
    // 4字节：11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
    output[3] = 0x80 | (c & 0x3F);             // 0x80: 10000000, 0x3F: 00111111 - 最低6位
    c >>= 6;
    output[2] = 0x80 | (c & 0x3F);             // 0x80: 10000000, 0x3F: 00111111 - 次低6位
    c >>= 6;
    output[1] = 0x80 | (c & 0x3F);             // 0x80: 10000000, 0x3F: 00111111 - 次高6位
    c >>= 6;
    output[0] = 0xF0 | c;                      // 0xF0: 11110000 - 最高3位 + 首字节前缀
    return 4;
}
```

**实现原理**

编码采用"从低位到高位，倒序构建"的策略：

1. **数据分解**：用`c & 0x3F`提取低6位，然后右移6位处理下一组  // 0x3F: 00111111
2. **格式添加**：用`0x80 | 数据`给续字节添加10前缀  // 0x80: 10000000
3. **倒序填充**：从最后一个字节开始向前填充，自然处理变长编码

### 2.7 函数5：`EncodeUTF8`，编码整个码点数组

最后，把整组码点重新编码成 UTF-8 字符串。到这里，码点数组和 UTF-8 字节串之间的往返路径就完整闭合了。

```cpp
std::string EncodeUTF8(const std::vector<uint32_t>& codepoints) {
    std::string result;
    
    for (uint32_t cp : codepoints) {
        char buffer[4];  // UTF-8最多需要4个字节
        size_t bytes = EncodeOneUTF8(cp, buffer);
        
        result.append(buffer, bytes);
    }
    
    return result;
}
```

这个函数遍历码点数组，逐个编码后拼接成完整的UTF-8字符串。

## 3. 小结

通过这几个函数，我们完成了 Unicode 码点与 UTF-8 字节序列之间的往返转换：

- **PrintBinary**：辅助理解二进制数值
- **IsTrailByte**：识别UTF-8续字节
- **DecodeOneUTF8 + DecodeUTF8**：UTF-8 → Unicode解码
- **EncodeOneUTF8 + EncodeUTF8**：Unicode → UTF-8编码

更重要的是，我们在这一章里建立了几个后面会一直反复出现的基本区分：

- 字符不是字节
- 字符串长度不等于字节长度
- Unicode 不是 UTF-8
- UTF-8 的本质是“带标记位的变长编码”

这几条看上去朴素，但只要其中任何一条没有真正吃透，后面的文本处理实现就很容易在边界条件上出错。

下一章会进入正则表达式。到那时，文本不再只是“如何表示”的问题，而开始变成“如何描述和匹配模式”的问题。
