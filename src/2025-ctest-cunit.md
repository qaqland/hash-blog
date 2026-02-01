# CUnit Cheet Sheet

用 C 写代码是一件简单又困难的事，尤其是从 0 开始搓，补测试可以减少心理上的恐惧。

## 框架选择

C 语言生态很权威，只需要去 Linux 发行版的源中找到一个能顺利安装的库就行。
本文随机选到了 CUnit，需要注意两点：

1. GPL-2.0-or-later 协议
2. 10 年未更新了

## 添加依赖

如果是 Alpine Linux，可以看到系统包管理安装后已经有了 `.pc` 文件

```bash
$ pkgconf --cflags --libs cunit
 -lcunit
```

此时只需要在 `meson.build` 中增加一行即可

```patch
--- a/meson.build
+++ b/meson.build
@@ -12,4 +12,5 @@ project(
 dependencies = [
   dependency('sqlite3'),
   dependency('libgit2'),
+  dependency('cunit'),
 ]
```

然后新开一个 target 补上一条 `test(target)`。

## 测试层级

与一般的测试框架一样，具有层级结构

```
Test Registry
├── Suite '1'
│   ├── Test '1-1'
│   ├── ...
│   └── Test '1-M'
├── ...
└── Suite 'N'
    ├── Test 'N-1'
    ├── ...
    └── Test 'N-M'
```

因此最基础的测试可以这样写：

- 初始化**测试表** `CU_initialize_registry(void)`
- 新建**测试套件** `CU_add_suite()`
- 添加**测试用例** `CU_add_test()`
- 终端运行测试 `CU_console_run_tests(void)`
- 清理**测试表**等 `CU_cleanup_registry(void)`

以上所有内容发生在一个单文件内，多文件部分暂无测试，等待后续补充。

## 生命周期

对于**测试表**，需要在一开始初始化并在最后清理（见上），这是全局作用域范围。

对于**测试套件**，可以在新建时传入 NULL 或两个函数指针管理生命周期，
分别在**测试套件**运行的前后各调用一次。

```c
CU_pSuite CU_add_suite(const char* strName,     // UNIQUE
                       CU_InitializeFunc pInit, // int (*)(void *)
                       CU_CleanupFunc pClean);  // int (*)(void *)
```


## 错误处理

CUnit 的函数基本不需要错误处理，内存不足几乎不可能（只在测试框架中）发生。

> 内存不足返回的 NULL 触发 SIGSEV 是有预期而且可以接受的，程序应当崩溃。
> 例如 GLib 有说：
>
> unless otherwise specified, any allocation failure will result in the
> application being terminated.
>
> [GLib – 2.0: Memory Allocation](https://docs.gtk.org/glib/memory.html)


## 辅助函数

自己写的**测试用例**需要使用库提供的辅助函数，然后添加到**测试套件**。

```c
#define CU_ADD_TEST(suite, test) (CU_add_test(suite, #test, (CU_TestFunc)test))

CU_pTest CU_add_test(CU_pSuite pSuite,
                     const char* strName,       // UNIQUE
                     CU_TestFunc pTestFunc);    // void (*)(void)
```

辅助函数以 `CU_` 为前缀，和常见的 `assert.h` 差不多，只不过更丰富一点

```
CU_{ASSERT,TEST}
CU_ASSERT_{TRUE,FALSE}
CU_ASSERT_{,NOT_}EQUAL
CU_ASSERT_PTR_{,NOT_}EQUAL
CU_ASSERT_PTR_{,NOT_}NULL
CU_ASSERT_STRING_{,NOT_}EQUAL
CU_ASSERT_NSTRING_{,NOT_}EQUAL
CU_ASSERT_DOUBLE_{,NOT_}EQUAL
CU_{PASS,FAIL}
```

最后一行的 PASS 和 FAIL 小 helper 不做实际判断，仅输出 message 供用户观察。
大部分辅助函数还有带 `_FATAL` 后缀的版本，失败会立即崩溃退出，不推荐使用，
猜测可以用来刻意 coredump 然后 GDB 介入。

## 测单文件

写 C 怎么可能不写宏呢？为了避免拆多文件，把原本的 `main` 入口修饰一下：

```c
#ifdef RUN_TEST
#define test_main main
#else
#define true_main main
#endif
```

## 参考阅读

- <https://cunit.sourceforge.net/>
- <https://cunit.sourceforge.net/doc/index.html>
- <https://stackoverflow.com/questions/7940279/should-we-check-if-memory-allocations-fail>

