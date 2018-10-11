---
published: true
layout: post
author: Moevis
categories: cheatsheet
---
用 C++ 来拼接一个字符串非常糟心，虽然说你可以用 stringstream 来做到，但是无数的 `<<` 就很麻烦，而 sprintf 也是一样，你需要对参数类型怎么表示很熟悉，但是我用得少，就记不住。

在看 C++ Weekly 时，看到有介绍 {fmt} ，于是试了一下，感觉很不错。

## 安装

首先是下载源码，可以先 clone 到你的项目中去，https://github.com/fmtlib/fmt ，我放到的是项目的 dependencies 目录

然后在 CMake 中加上这两句：

```
add_subdirectory(dependencies/fmt EXCLUDE_FROM_ALL)
target_link_libraries(fmt_demo fmt-header-only)
```

其中 `EXCLUDE_FROM_ALL` 是我第一次知道有这种写法，表示将这个项目移除出 make 列表。接着是链接 fmt-header-only 这个库，其实也可以用 fmt 这个，从名字上就能区别它们的不同了。

## 使用

首先是最简单用法：

```c++
fmt::print("Hello, {}!", "world");
```

这个语法类似 python，很方便，将参数依次填入第一个字符串模板的 `{}` 标记中，你也可以通过下标来制定填入顺序：

```c++
fmt::print("Hello, {0}, {0}", "world");
```

也可以用作一个方便的字符串 format 函数：

```c++
fmt::memory_buffer buf;
fmt::format_to(buf, "{}", 42);
fmt::format_to(buf, "{:x}", 42); // 十六进制表示
std::cout << buf.data();
```

或者是给输出加入特定颜色：

```c++
fmt::print(fmt::color::red, "hello {}\n", "world");
fmt::print(fmt::rgb(10, 50, 63), "hello {}\n", "world");
```

日期格式化：
```c++
std::time_t t = std::time(nullptr);
fmt::print("The date is {:%Y-%m-%d}.\n", *std::localtime(&t));
```

```
The date is 2018-10-09.
```

最后是 fmt 提供方便的 error 输出：
```c++
fmt::memory_buffer error_buff;
fmt::format_error_code(error_buff, 42, "test");
std::cout << error_buff.data() << std::endl;
```

```
test: error 42
```
