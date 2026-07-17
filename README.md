# 底层实现

本仓库包含三本彼此独立的 `mdBook`：

| 目录 | 书名 | 主题 |
|---|---|---|
| `Text/` | 底层实现：文本处理 | Unicode、正则、Trie、分词、Tokenizer |
| `Zero/` | 底层实现：训练引擎 | 张量、自动微分、模型训练、GPU 与 GPT |
| `Matx/` | 底层实现：编译器 | AST、Visitor、运行时对象、容器、函数与 FFI |

## 本地构建

在仓库根目录分别运行：

```sh
mdbook build Text
mdbook build Zero
mdbook build Matx
```

构建结果依次写入 `book-text/`、`book-zero/` 和 `book-matx/`。开发时也可以运行 `mdbook serve Text`、`mdbook serve Zero` 或 `mdbook serve Matx` 预览单本书。
