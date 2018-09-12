---
published: false
layout: post
author: Moevis
categories: cheatsheet
---
今天看了：https://cliutils.gitlab.io/modern-cmake/ ，大概看了一大半，把知识点总结了一下，随时更新：

## 入门介绍

### 反模式

- 不要用全局函数： `link_directories`, `include_libraries` 等类似的方法
- 不要加非必须的 PUBLIC 依赖，比如 `-Wall`，换成 PRIVATE。
- 不要用 GLOB 来收集文件：这样你的其他工具无法在你添加了文件后主动运行 CMake，值得一提的是 `CMake 3.12` 引入了 `CONFIGURE_DEPENDS` 标记来让这个过程更方便。
- 不要直接链接到目录，而是链接到 target
- 在链接时不跳过任何 PUBLIC/PRIVATE

### 正确原则

- CMake 就是代码
- 导出接口
- 编写  Config.make
- 添加别名： add_subdirectory 和 find_package 应该提供同样的 target 和 namespace
- 把公共函数都整理出来，并给出靓号文档
- 用小写的函数名，大写是用于变量
- 使用 cmake_policy 添加版本号

## 基本介绍

### 变量

本地变量用以下语法进行定义：

```cmake
set(MY_VARIABLE "value")
set(MY_LIST "one" "two")
set(MY_LIST "one;two")
```

本地变量在出了作用域后就不再存在。如果你希望在命令行给定一些参数，你应该用下面的语法：

```cmake
set(MY_CACHE_VARIABLE "VALUE" CACHE STRING "Description")
```

CMake 本身有一些预定义的变量比如：`CMAKE_BUILD_TYPE`。

上面的语法不会替换掉一个已经存在的变量，这样你就可以在命令行中传入相关值而不用担心被改写。如果你希望你的变量可以全局使用，可以使用下面的语句：

```cmake
mark_as_advanced(MY_CACHE_VARIABLE)
```

如果你想知道项目中有哪些需要设置的变量，可以使用 `cmake -L` 来获取。

如果你想声明 BOOL 变量，可以使用下面的语法：

```cmake
 option(MY_OPTION "This is settable from the command line" OFF)
```

### 环境变量

设置环境变量和读取环境变量：

```cmake
set(ENV{variable_name} value)
$ENV{variable_name}
```

### 缓存文件

CMake 的缓存文件是以纯文本的方式储存在 CMakeCache.txt 中的。这样你在重新跑 CMake 的时候就不必重新输入各种参数了。

### 属性

CMake 储存信息的另一种方式是属性（Property），它可以附加在其他的对象上，比如目录 directory 或者目标 target 。

语法如下：

```cmake
set_property(TARGET TargetName
             PROPERTY CXX_STANDARD 11)

set_target_properties(TargetName PROPERTIES
                      CXX_STANDARD 11)
```

前一种更常用一点，这样一次性会把 targets/ files/ tests 全部都设置完成。第二种用于设置单独的一个 target 的属性。同样在设置属性之外，你可以用下面的方式来获取他们的值：

```cmake
get_property(ResultVariable TARGET TargetName PROPERTY CXX_STANDARD)
```

### 控制流

CMake 的 if 语句如下：

```cmake
if(variable)
    # `ON`, `YES`, `TRUE`, `Y` 或者非零数
else()
    # `0`, `OFF`, `NO`, `FALSE`, `N`, `IGNORE`, `NOTFOUND`, `""` 或者以 `-NOTFOUND` 结尾
endif()
```

### 生成表达式

大多数 CMake 命令在 configure 阶段运行，包含上面的 if 语句。但是如果你希望在 build 的时候执行一些逻辑语句，你可以用生成表达式（Generator-expressions）。最简单的生成表达式是信息表达式，用 `$<KEYWORD>` 形式给出。另一种形式是：`$<KEYWORD:value>`，其中 `KEYWORD` 是控制流关键字，它值将对应为 0 或者 1。

比如：你希望只在 DEBUG 模式加入一个编译选项，你可以这么做：

```cmake
target_compile_options(MyTarget PRIVATE "$<$<CONFIG:Debug>:--my-flag>")
```

你可以在很多的包中看到类似下面的 CMake 代码：

```cmake
 target_include_directories(
     MyTarget
     PUBLIC
     $<BUILD_INTERFACE:"${CMAKE_CURRENT_SOURCE_DIR}/include">
     $<INSTALL_INTERFACE:include>
     )
```

### 宏和函数

你可以很轻松地定义你自己的宏和函数，他们两个区别并不大，仅仅在于函数有作用域而宏没有。如果你希望你在函数中设置的一些变量能给外部使用，你可以添加 `PARENT_SCOPE` 标记。

```cmake
function(SIMPLE REQUIRED_ARG)
    message(STATUS "Simple arguments: ${REQUIRED_ARG}, followed by ${ARGV}")
    set(${REQUIRED_ARG} "From SIMPLE" PARENT_SCOPE)
endfunction()

simple(This)
message("Output: ${This}")
```

### 与代码交互

你可以在你的代码里访问到 CMake 中设置的一些参数，方法是用 `configure_file` 函数。它的原理是将一个 `.in` 结尾的文件里标记的内容替换到代码文件中。这个方法也被广泛使用：

`Version.h.in`

```c++
#pragma once

#define MY_VERSION_MAJOR @PROJECT_VERSION_MAJOR@
#define MY_VERSION_MINOR @PROJECT_VERSION_MINOR@
#define MY_VERSION_PATCH @PROJECT_VERSION_PATCH@
#define MY_VERSION_TWEAK @PROJECT_VERSION_TWEAK@
#define MY_VERSION "@PROJECT_VERSION@"
```

`CMake inlines`

```cmake
configure_file (
    "${PROJECT_SOURCE_DIR}/include/My/Version.h.in"
    "${PROJECT_BINARY_DIR}/include/My/Version.h"
)
```

### 项目结构

这个部分并不是唯一正确的架构，仅作为参考：

```
- project
  - .gitignore
  - README.md
  - LICENCE.md
  - CMakeLists.txt
  - cmake
    - FindSomeLib.cmake
  - include
    - project
      - lib.hpp
  - src
    - CMakeLists.txt
    - lib.cpp
  - apps
    - CMakeLists.txt
    - app.cpp
  - tests
    - testlib.cpp
  - docs
    - Doxyfile.in
  - extern
    - googletest
  - scripts
    - helper.py
```

你可能需要一个 cmake 目录存放一些辅助 cmake 文件，像 `Find<library>.cmake` 这样子的，当然你也要通过下面这句命令让 CMake 知道到哪里去找你自定义的 cmake 文件。

```cmake
set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})
```

### 运行外界命令

```cmake
find_package(Git QUIET)

if(GIT_FOUND AND EXISTS "${PROJECT_SOURCE_DIR}/.git")
    execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init --recursive
                    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                    RESULT_VARIABLE GIT_SUBMOD_RESULT)
    if(NOT GIT_SUBMOD_RESULT EQUAL "0")
        message(FATAL_ERROR "git submodule update --init failed with ${GIT_SUBMOD_RESULT}, please checkout submodules")
    endif()
endif()
```

## 附加特性

### 平台无关代码

相当于给代码加上`-fPIC` 标志。实际上多数时候你并不需要这么写，CMake 会为 `SHARED` 或 `MODULE` 库文件带上这个标志，如果你想手动强调，可以这样：

```cmake
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
```

或者全局使用：

```cmake
set_target_properties(lib1 PROPERTIES POSITION_INDEPENDENT_CODE ON)
```

### 默认构建配置

你可以用下面一小段代码用来指定默认编译的模式：

```cmake
set(default_build_type "Release")
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
  set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE
      STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()
```

### 收集 configure 过程中的信息

在 cmake 开头加入下面命令：

```cmake
include(FeatureSummary)
```

接着任何使用 find_package 的库，相关信息都可以被收集到，当然你也可以自己添加一些信息比如这样：

```cmake
add_feature_info(WITH_OPENMP OpenMP_CXX_FOUND "OpenMP (Thread safe FCNs only)")
```

最后你可以打印这些信息或者输出到文件：

```cmake
if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
    feature_summary(WHAT ENABLED_FEATURES DISABLED_FEATURES PACKAGES_FOUND)
    feature_summary(FILENAME ${CMAKE_CURRENT_BINARY_DIR}/features.log WHAT ALL)
endif()
```

### clang 工具

你可以用以下工具让你的代码更好看：

- `<LANG>_CLANG_TIDY`: CMake 3.6+
- `<LANG>_CPPCHECK`
- `<LANG>_CPPLINT`
- `<LANG>_INCLUDE_WHAT_YOU_USE`

#### Clang tidy

Clang tidy 要求 CMake 版本高于 3.6：

```cmake
if(CMAKE_VERSION VERSION_GREATER 3.6)
    # Add clang-tidy if available
    option(CLANG_TIDY_FIX "Perform fixes for Clang-Tidy" OFF)
    find_program(
        CLANG_TIDY_EXE
        NAMES "clang-tidy"
        DOC "Path to clang-tidy executable"
    )

    if(CLANG_TIDY_EXE)
        if(CLANG_TIDY_FIX)
            set(CMAKE_CXX_CLANG_TIDY "${CLANG_TIDY_EXE}" "-fix")
        else()
            set(CMAKE_CXX_CLANG_TIDY "${CLANG_TIDY_EXE}")
        endif()
    endif()
endif()
```

参数 `-fix` 是可选的，若声明了这个参数，那么你的代码将被 clang tidy 自动修正，否则只是提醒你要修改哪些地方。如果你用了版本管理，这样的自动修改会比较安全，你可以随时追踪到代码被修改的地方。不过要注意的是，一定不要让你的 makefile/ninja 同时被运行，不然 clang tidy 可能不会正常工作。

如果你希望明确 clang tidy 只在你负责的代码中运行，你可以设置一个属性（我通常用`DO_CLANG_TIDY`），而不是用 `CMAKE_CXX_CLANG_TIDY` 变量来声明。

### Include what you use

 这里是个简单示例：

```
docker run --rm -it tuxity/include-what-you-use:clang_4.0
$ build # cmake .. -DCMAKE_CXX_INCLUDE_WHAT_YOU_USE=include-what-you-use
$ build # make 2> iwyu.out
$ build # fix_includes.py < iwyu.out
```
