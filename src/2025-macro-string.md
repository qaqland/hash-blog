# C 跨行字符串

2025-12-13

实用小技巧之跨行字符串（字面量）

```c
#define SQL(...) #__VA_ARGS__

const char *foo = SQL(
    HELLO WORLD
    \n // 111
    /* 222 */
    next  line     111?
    ""
);

// equal

const char *foo = "HELLO WORLD \n next line 111? \"\"";
```

细节拆解：

- Stringizing in C involves more than putting double-quote characters around
the fragment. 不仅仅会在字符串两端加上双引号
- The preprocessor backslash-escapes the quotes surrounding embedded string
constants, and all backslashes within string and character constants.
而且会转义字符串中适当位置的双引号
- Any sequence of whitespace in the middle of the text is converted to a single
space in the stringized result. 像 Markdown 一样压缩所有的空白字符到一个空格
- Comments are replaced by whitespace
忽略注释（因为注释会在预处理器之前被处理）

## 上下文

为了给 bushi-index 找可以学习的素材，在 GitHub 搜索 SQLite3 项目找到了这里。
对 SQL 来说（吃掉换行符）转为一行的行为，比转成多行然后前面带缩进好。不错！

## 参考链接

- <https://github.com/yonasBSD/litterbox/blob/1.9/database.h#L41>
- <https://github.com/yonasBSD/litterbox/blob/1.9/database.h#L325>
- <https://gcc.gnu.org/onlinedocs/cpp/Stringizing.html>

