---
published: true
layou: post
author: Moevis
categories: bug
---
在编译好一个 C++ 库后，我并没有把它安装到用户目录 `/usr/local/bin`，而是手动指定了一个目录：

```
make install DESTDIR=../install
```

然而这时候到目录中运行可执行文件时就出现了一个问题：

```
dyld: Library not loaded: @rpath/AAA.dylib
  Referenced from: AAA_EXE
  Reason: image not found
```

`AAA_EXE` 中用了 AAA.dylib 的动态库，但是在安装到自定义的目录后，可执行文件在用户的 `/usr/local/bin` 中没有找到 AAA.dylib。一种解决方案是我直接安装到 `/usr/local/bin`，但是我就是不想安装到系统目录而已。

所以可以这么做：

```cmake
set_target_properties(AAA_EXE PROPERTIES
        INSTALL_RPATH "@loader_path/../lib"
        )
```

这样可以指定到可执行文件的上一级目录下的 lib 中找库。

暂时解决这个问题了。
