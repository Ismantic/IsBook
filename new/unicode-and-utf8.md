# 底层实现：Unicode 与 UTF-8

## Unicode

Unicode指的就是把全世界各种文字的"字"都编码成一个个具体的数字，比如：

- **英文**：`'a'` → 97, `'b'` → 98, `'c'` → 99, ...
- **中文**：`'你'` → 20320, `'我'` → 25105, `'他'` → 20182, ...  
- **Emoji**：`'😀'` → 128512, `'🌍'` → 127757, ...

全部的数字范围现在是2³²，也就是用`uint32_t`来表示，实际上是没有用满的。目前登记在用的字符超过了15万个，但理论上Unicode的有效范围是0到0x10FFFF（约110万个码点）。

虽然有了Unicode，但如果把所有文字都用`uint32_t`（4字节）存储会比较低效：

1. **存储效率问题**：英文字符在互联网上出现频率很高，但只需要7位就能表示，用4字节存储浪费了75%的空间
2. **向后兼容**：需要兼容Unicode之前的ASCII标准（用1字节表达拉丁文字符）
3. **网络传输**：更少的字节意味着更快的传输速度

因此设计了UTF-8（8-bit Unicode Transformation Format），它是一种**变长编码**：出现频率高的字符用更少的字节表示，最多能节省75%的存储空间。

## UTF-8

UTF-8使用1到4个字节的变长方式来表达Unicode字符。为了解码时不发生混乱，制定了严格的格式约束，编码格式：

```
1字节：0xxxxxxx
2字节：110xxxxx 10xxxxxx
3字节：1110xxxx 10xxxxxx 10xxxxxx
4字节：11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```

解码规则，从UTF-8字节序列提取Unicode值时，只需要取出所有的x位并拼接：

```
1字节：0xxxxxxx → xxxxxxx
2字节：110xxxxx 10xxxxxx → xxxxx xxxxxx
3字节：1110xxxx 10xxxxxx 10xxxxxx → xxxx xxxxxx xxxxxx  
4字节：11110xxx 10xxxxxx 10xxxxxx 10xxxxxx → xxx xxxxxx xxxxxx xxxxxx
```
容量分配：
- **1字节**：7位 → 128种字符（0-127）- ASCII兼容
- **2字节**：5+6=11位 → 2048种字符（128-2047）
- **3字节**：4+6+6=16位 → 65536种字符（2048-65535）
- **4字节**：3+6+6+6=21位 → 2097152种字符（65536-1114111）

## C++ 实现

**函数1**：IsTrailByte - 判断续字节


首先实现一个函数来判断一个字节是不是UTF-8的续字节。从上面的编码规则可以看到，当字符长度>1字节时，后续的字节都以10开头。

```cpp
bool IsTrailByte(uint8_t x) {
    return (x & 0xC0) == 0x80;
}
```

**实现原理**
- `0xC0`（11000000）作为掩码，提取字节的最高2位
- `0x80`（10000000）是续字节的标准模式
- 通过位与操作`&`提取高2位，然后与`0x80`比较

这个函数能准确识别所有64种续字节（0x80-0xBF）。  // 0x80: 10000000, 0xBF: 10111111

**函数2**：DecodeOneUTF8 - 解码单个字符

接下来实现UTF-8解码的核心函数，它从字符串开头解码一个UTF-8字符：

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


**函数3**：DecodeUTF8 - 解码整个字符串

基于`DecodeOneUTF8`，我们可以实现解码整个UTF-8字符串的函数：

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

这个函数通过循环调用`DecodeOneUTF8`，逐个字符解码整个字符串，最终返回Unicode码点数组。


**函数4**：EncodeOneUTF8 - 编码单个字符

编码函数基本上是解码的镜像操作，将Unicode码点转换为UTF-8字节序列：

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

**函数5**：EncodeUTF8 - 编码整个码点数组

最后实现将Unicode码点数组编码为UTF-8字符串的函数：

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


