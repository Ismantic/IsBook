# 底层实现

本仓库包含三本彼此独立的 `mdBook`：

| 目录 | 书名 | 主题 |
|---|---|---|
| `Text/` | 底层实现：文本处理 | Unicode、正则、Trie、分词、Tokenizer 与语言模型文本处理 |
| `Matx/` | 底层实现：编译器 | AST、Visitor、运行时对象、容器、函数与 FFI |
| `Zero/` | 底层实现：训练引擎 | 张量、自动微分、模型训练、GPU 与 GPT |

旧版章节与站点结构保存在 `Legacy/`，仅供迁移和内容核对。

## 本地构建

在仓库根目录分别运行：

```sh
mdbook build Text
mdbook build Matx
mdbook build Zero
```

构建结果依次写入 `book-text/`、`book-matx/` 和 `book-zero/`。开发时也可以运行 `mdbook serve Text`、`mdbook serve Matx` 或 `mdbook serve Zero` 预览单本书。
