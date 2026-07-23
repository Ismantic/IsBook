# Unicode 与 UTF-8

## Unicode

Unicode 为世界各地文字、符号和控制字符分配统一的码点，例如：

- **英文**：`'a'` → 97, `'b'` → 98, `'c'` → 99, ...
- **中文**：`'你'` → 20320, `'我'` → 25105, `'他'` → 20182, ...  
- **Emoji**：`'😀'` → 128512, `'🌍'` → 127757, ...

Unicode 码点通常用 `uint32_t` 保存，但有效范围只有 `U+0000` 到 `U+10FFFF`，其中代理项区间 `U+D800` 到 `U+DFFF` 也不能表示 Unicode 标量值。因此，32 位整数只是方便的内存表示，并不意味着全部数值都合法。

虽然有了 Unicode，但如果把所有文字都用 `uint32_t`（4 字节）存储会比较低效：

1. **存储效率问题**：ASCII 字符只需要 7 位，用 4 字节存储会浪费大量空间
2. **向后兼容**：需要兼容 Unicode 之前的 ASCII 标准
3. **网络传输**：更少的字节意味着更快的传输速度

UTF-8（8-bit Unicode Transformation Format）使用 1 到 4 个字节编码一个 Unicode 标量值。ASCII 码点仍只占一个字节，并且编码结果与原有 ASCII 字节完全一致；更大的码点使用更多字节。

## UTF-8

UTF-8 使用 1 到 4 个字节表示一个 Unicode 标量值。首字节决定序列长度，后续字节统一以 `10` 开头：

```
1字节：0xxxxxxx
2字节：110xxxxx 10xxxxxx
3字节：1110xxxx 10xxxxxx 10xxxxxx
4字节：11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```

解码时只需从 UTF-8 字节序列中取出所有 `x` 位，并按顺序拼接为 Unicode 码点：

```
1字节：0xxxxxxx → xxxxxxx
2字节：110xxxxx 10xxxxxx → xxxxx xxxxxx
3字节：1110xxxx 10xxxxxx 10xxxxxx → xxxx xxxxxx xxxxxx  
4字节：11110xxx 10xxxxxx 10xxxxxx 10xxxxxx → xxx xxxxxx xxxxxx xxxxxx
```
对应的合法码点范围为：

- **1 字节**：`U+0000`～`U+007F`
- **2 字节**：`U+0080`～`U+07FF`
- **3 字节**：`U+0800`～`U+FFFF`，排除代理项
- **4 字节**：`U+10000`～`U+10FFFF`

## C++ 实现

**函数 1：IsTrailByte**


续字节的最高两位必须是 `10`：

```cpp
bool IsTrailByte(uint8_t x) {
    return (x & 0xC0) == 0x80;
}
```

**实现原理**
- `0xC0`（11000000）作为掩码，提取字节的最高 2 位
- `0x80`（10000000）是续字节的标准模式
- 通过位与操作 `&` 提取最高 2 位，然后与 `0x80` 比较

它可以识别 `0x80`～`0xBF` 范围内的 64 个续字节。

**函数 2：DecodeOneUTF8**

解码不仅要检查字节前缀，还要拒绝过长编码、代理项和超过 `U+10FFFF` 的结果。非法序列返回替换字符 `U+FFFD`，并消费一个字节，使调用方可以继续处理后续输入：

```cpp
uint32_t DecodeOneUTF8(const char* begin, const char* end, size_t* bytes) {
    constexpr uint32_t kError = 0xFFFD;
    const size_t len = end - begin;
    if (len == 0) {
        *bytes = 0;
        return kError;
    }

    const auto* data = reinterpret_cast<const uint8_t*>(begin);
    uint32_t cp = 0;
    size_t width = 0;
    uint32_t minimum = 0;

    if (data[0] < 0x80) {
        *bytes = 1;
        return data[0];
    } else if ((data[0] & 0xE0) == 0xC0) {
        width = 2; minimum = 0x80; cp = data[0] & 0x1F;
    } else if ((data[0] & 0xF0) == 0xE0) {
        width = 3; minimum = 0x800; cp = data[0] & 0x0F;
    } else if ((data[0] & 0xF8) == 0xF0) {
        width = 4; minimum = 0x10000; cp = data[0] & 0x07;
    }

    if (width != 0 && len >= width) {
        for (size_t i = 1; i < width; ++i) {
            if (!IsTrailByte(data[i])) {
                width = 0;
                break;
            }
            cp = (cp << 6) | (data[i] & 0x3F);
        }
        const bool surrogate = cp >= 0xD800 && cp <= 0xDFFF;
        if (width != 0 && cp >= minimum && cp <= 0x10FFFF && !surrogate) {
            *bytes = width;
            return cp;
        }
    }

    *bytes = 1;
    return kError;
}
```

`minimum` 是当前字节数可以表示的最小码点，用来排除本可用更短序列表示的过长编码。循环则逐个验证续字节，并通过左移 6 位还原码点。

**函数 3：DecodeUTF8**

基于 `DecodeOneUTF8`，可以实现整个 UTF-8 字符串的解码：

```cpp
std::vector<uint32_t> DecodeUTF8(const std::string& str) {
    std::vector<uint32_t> codepoints;
    
    size_t pos = 0;
    while (pos < str.size()) {
        size_t bytes_consumed;
        
        uint32_t codepoint = DecodeOneUTF8(
            str.data() + pos, str.data() + str.size(), &bytes_consumed);
        
        codepoints.push_back(codepoint);
        pos += bytes_consumed;
    }
    
    return codepoints;
}
```

这个函数循环调用 `DecodeOneUTF8`，最终返回 Unicode 码点数组。非法输入会以替换字符保留在结果中，而不会截断整个字符串。


**函数 4：EncodeOneUTF8**

编码是解码的镜像操作，将 Unicode 码点转换为 UTF-8 字节序列：

```cpp
size_t EncodeOneUTF8(uint32_t c, char* output) {
    constexpr uint32_t kError = 0xFFFD;
    if (c > 0x10FFFF || (c >= 0xD800 && c <= 0xDFFF)) {
        c = kError;
    }

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

编码采用“从低位到高位，倒序构建”的策略：

1. **数据分解**：用 `c & 0x3F` 提取低 6 位，然后右移 6 位处理下一组
2. **格式添加**：用 `0x80 | 数据` 给续字节添加 `10` 前缀
3. **倒序填充**：从最后一个字节开始向前填充，自然处理变长编码

**函数 5：EncodeUTF8**

最后将 Unicode 码点数组编码为 UTF-8 字符串：

```cpp
std::string EncodeUTF8(const std::vector<uint32_t>& codepoints) {
    std::string result;
    
    for (uint32_t cp : codepoints) {
        char buffer[4];  // UTF-8 最多需要 4 个字节
        size_t bytes = EncodeOneUTF8(cp, buffer);
        
        result.append(buffer, bytes);
    }
    
    return result;
}
```

这个函数遍历码点数组，逐个编码后拼接成完整的 UTF-8 字符串。

完成编码与解码之后，后续算法就可以在码点层面处理字符，而把字节边界留在输入输出层。下一篇将以此为基础，实现能够正确处理 UTF-8 文本的正则表达式引擎。

配套实现：[Ismantic/Ustr](https://github.com/Ismantic/Ustr)
