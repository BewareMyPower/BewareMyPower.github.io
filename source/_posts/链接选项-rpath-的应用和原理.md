---
title: 链接选项 rpath 的应用和原理
date: 2020-07-14 18:28:05
tags:
  - C++
  - 链接
top: 1
---

## 前言

在测试和部署 C++ 动态库时，经常遇到的问题就是程序链接到了系统路径下的动态库，有时候 `make` 编译时链接到本地路径的动态库，但实际 `make install` 时则会丢失这个依赖。本文将要介绍的就是一种通用解决方法，使用 `RPATH` 来绑定链接路径。

## 简单动态库编译和使用示例

给出以下示例库：

```c++
// foo.h
#pragma once

#ifdef __cplusplus
extern "C" {
#endif

void foo();

#ifdef __cplusplus
}
#endif
```

```c++
// foo.cc
#include "foo.h"
#include <stdio.h>

void foo() { printf("foo\n"); }
```

生成动态库 libfoo.so：

```bash
g++ -o libfoo.so -fPIC -shared foo.cc
```

然后给出调用代码：

```c++
// main.cc
#include "foo.h"

int main(int argc, char* argv[]) {
  foo();
  return 0;
}
```

编译时指定链接当前目录：

```bash
$ gcc main.c -L . -lfoo
$ ./a.out 
foo
$ ldd a.out | grep libfoo
	libfoo.so (0x00007fc2010f7000)
```

至此，一切正常。

## 依赖动态库的动态库

实际编写程序时，往往会依赖一些第三方库来避免重复造轮子。比如，这里我们要写一个库依赖于 libfoo.so。

目录层次：

```
.
├── include
│   └── bar.h
├── src
│   └── bar.cc
└── thirdparty
    ├── include
    │   └── foo.h
    └── lib
        └── libfoo.so
```

然后编译 libbar.so：

```bash
g++ -o libbar.so -fPIC -shared src/bar.cc -I include/ -I thirdparty/include/ -L thirdparty/lib/ -lfoo
```

问题来了，编译出的 libbar.so 找不到 libfoo.so 的依赖：

```bash
$ ldd libbar.so | grep foo
	libfoo.so => not found
```

当然，这样的话，你编译依赖 libbar.so 的程序时会直接失败，从而提醒你去寻找依赖的 libfoo.so：

> /usr/bin/ld: warning: libfoo.so, needed by ./libbar.so, not found (try using -rpath or -rpath-link)
> ./libbar.so: undefined reference to `foo'

注意这里我们第一次见到 *rpath* 这个概念。

但是问题更大的是，假如 libfoo.so 是一个旧版的库，而有个其他用户完全无视影响，直接将 libfoo.so 安装到了系统目录，比如 `/usr/lib64` 下面。这样，你的程序依赖的 libbar.so 将会找到系统目录下旧的 libfoo.so，而不是你自己维护的新版。如果新版 libfoo.so 的 ABI 发生了改变而 API 不变，比如这里 C 库变成了 C++ 库：

```C++
// foo.h
#pragma once

void foo(int i = 0);

```

```c++
// foo.cc
#include "foo.h"
#include <stdio.h>

void foo(int i) { printf("foo: %d\n", i); }

```

API 兼容指的是，调用 `foo()` 仍然合法，但是由于 C++ 的 name mangling，带有默认参数的 `foo` 对应的符号发生了变化，因此 foo 可能还会出现这样的错误（main.cc 仅仅是调用 `bar()` 函数，这里就不贴代码了）：

```c++
$ g++ main.cc -L . -lbar
./libbar.so: undefined reference to `foo(int)'
collect2: error: ld returned 1 exit status

```

查看 libbar.so 的依赖就什么都明白了：

```
$ ldd libbar.so | grep libfoo
	libfoo.so => /usr/lib64/libfoo.so (0x00007efd6daf1000)

```

原因是 `foo` 的函数签名变成了 `void foo(int)`，而链接到的动态库却是全局的 libfoo.so。简单的解决方式是，将本地库的路径加入 `LD_LIBRARY_PATH` 中：

```bash
$ export LD_LIBRARY_PATH=$PWD/thirdparty/lib:$LD_LIBRARY_PATH
$ g++ main.cc -L . -lbar
$ ./a.out 
bar
foo: 0

```

用 `ldd` 也能查看 libbar.so 依赖的 libfoo.so 不再是全局的，而是本地的。但问题是，如果发布单独的 libbar.so 给用户，而用户又因为某些原因无法升级全局的 libfoo.so，那么每次用户都要手动设置 `LD_LIBRARY_PATH`。

此时，另一种解决方法刚好能避免这个问题，也就是使用 rpath。

## rpath 的使用

rpath 即 runtime path，运行时路径。既可以指定相对路径也可以指定绝对路径。

编译方式：

```bash
$ g++ -o libbar.so -fPIC -shared src/bar.cc \
   -I include/ -I thirdparty/include/ \
   -L thirdparty/lib/ -lfoo -Wl,-rpath=thirdparty/lib/
$ ldd libbar.so | grep foo
	libfoo.so => thirdparty/lib/libfoo.so (0x00007f8319965000)

```

注意最后的 `-Wr,-rpath` 指定的是动态库的路径。看似和 `-L` 重复，实际不然。`-L` 指定的是编译时链接的 libfoo.so 路径，而 `-Wl,-rpath` 指定的是（libbar.so）运行时链接的 libfoo.so 路径。这里指定的是相对路径。

因此如果我们安装 libbar.so 到全局又不影响全局的 libfoo.so，比如安装到 `/usr/lib64` 下面：

```bash
$ sudo cp libbar.so /usr/lib64

```

我们继续编译 libbar.so 的使用程序：

```bash
$ g++ main.cc -lbar
$ ldd a.out | grep foo
	libfoo.so => thirdparty/lib/libfoo.so (0x00007f5d3d79f000)
$ ldd a.out | grep bar
	libbar.so => /usr/lib64/libbar.so (0x00007fed07e78000)
$ ls /usr/lib64/libfoo.so 
/usr/lib64/libfoo.so

```

这样想发布依赖高版本 libfoo.so 的 libbar.so 时，用户只需要在**编译和运行**时，相对路径 `thirdparty/lib` 下面有高版本 libfoo.so 就行了，无需覆盖全局路径下的低版本 libfoo.so。

注意如果换个路径运行 a.out，由于 rpath 指定的是相对路径，此时会找不到 libfoo.so。

所以 rpath 指定绝对路径的做法也是比较常见的，比如编译 libbar.so 时将 libfoo.so 置于不会冲突的系统目录：

```bash
$ sudo mkdir -p /usr/lib64/foo-1.1
$ sudo cp thirdparty/lib/libfoo.so /usr/lib64/foo-1.1
$ g++ -o libbar.so -fPIC -shared src/bar.cc \
   -I include/ -I thirdparty/include/ \
   -L /usr/lib64/foo-1.1 -lfoo -Wl,-rpath=/usr/lib64/foo-1.1
 $ ldd libbar.so | grep foo
	libfoo.so => /usr/lib64/foo-1.1/libfoo.so (0x00007f83df270000)

```

那么用户部署时，只需要将 libfoo.so 放置在 `/usr/lib64/foo-1.1` 下面就行，这里的 1.1 用于标识版本号。由于该目录并不会被自动连接，从而防止了其他程序自动链接到这个版本的 libfoo.so。

## 回到现代，使用 CMake

但凡稍有规模的程序，直接使用 GCC 编译来构建项目是难以维护的。即使有了 Makefile，管理和维护起来还是相对麻烦。C++ 缺乏类似 Maven 那样的构建系统，但退而求其次，CMake 已经成为了事实上的 C++ 构建通用解决方案。（虽然早期流行的 Autotools 仍然有一定市场）

以一个极简的 CMakeLists.txt 为例，将 rpath 指定为相对路径

```cmake
cmake_minimum_required(VERSION 2.8.12)
project(Bar CXX)

# 设置 find_path/find_library 查找的根目录，默认会从 include 以及 lib 子目录查找
set(CMAKE_PREFIX_PATH "${PROJECT_SOURCE_DIR}/thirdparty")
find_path(FOO_INCLUDE_DIR NAMES foo.h)
find_library(FOO_LIB NAMES libfoo.so)

add_library(bar SHARED src/bar.cc)
include_directories(./include ${FOO_INCLUDE_DIR})
target_link_libraries(bar ${FOO_LIB})

# 设置 rpath，这里是绝对路径
set(CMAKE_INSTALL_RPATH "${PROJECT_SOURCE_DIR}/thirdparty/lib")

# 安装到 lib 子目录，该相对路径是相对 CMAKE_INSTALL_PREFIX 而言的
install(TARGETS bar LIBRARY DESTINATION lib)

```

PS：说是回到现代，我这个 CMake 还是老式的风格，现代 CMake 又是另一个话题了，不熟悉 CMake 的话，可以直接从现代 CMake 学起。

当前目录层次：

```
.
├── CMakeLists.txt
├── include
│   └── bar.h
├── main.cc
├── src
│   └── bar.cc
└── thirdparty
    ├── include
    │   └── foo.h
    └── lib
        └── libfoo.so

```

使用 CMake 构建项目：

```bash
$ mkdir _builds && $ cd _builds/
$ cmake .. -DCMAKE_INSTALL_PREFIX=$PWD/..
$ make
$ make install

```

之后目录层次（忽略中间目录 `_builds`）：

```
.
├── CMakeLists.txt
├── include
│   └── bar.h
├── lib
│   └── libbar.so
├── main.cc
├── src
│   └── bar.cc
└── thirdparty
    ├── include
    │   └── foo.h
    └── lib
        └── libfoo.so

```

类似地，为了部署的话，可以将 libfoo.so 部署到系统目录 `/usr/lib64` 的子目录。

也可以修改成相对路径：

```cmake
set(CMAKE_SHARED_LINKER_FLAGS ${CMAKE_SHARED_LINKER_FLAGS} -Wl,-rpath,'$ORIGIN/thirdparty')

```

其中 `$ORIGIN` 会被替换成动态库所处的绝对路径，也就是说只要 libfoo.so 处于 libbar.so 同级目录 thirdparty 下面，如下图所示：

```
.
├── libbar.so
└── thirdparty
    └── libfoo.so

```

之后 libbar.so 就会链接到 thirdparty/libfoo.so，并且都是绝对路径。

## Linux 动态库查找路径

最后一节，以理论来结尾。前文侧重实践，有了实践作为基础，回过头来看原理就更有体会了。

一个典型的 C/C++ 程序的构建流程是：预处理，汇编，编译，链接。而执行链接的程序其实是 `ld`，通常编译器比如 GCC 都会自动调用 `ld` 去进行链接，用户不必关注其中的细节。而 `ld` 查找动态库的顺序是：

1. rpath 指定的目录；
2. 环境变量 `LD_LIBRARY_PATH` 指定的目录；
3. runpath 指定的目录；
4. `/etc/ld.so.cache ` 缓存文件，通常包含 `/etc/ld.so.conf` 文件编译出的二进制俩别哦（比如 CentOS 上，该文件会使用 include 从而使用 `ld.so.conf.d` 目录下面所有的 `*.conf` 文件，这些都会缓存在 ld.so.cache 中）
5. 系统默认路径，比如 `/lib`，`/usr/lib`。

在编译时若使用 `-z nodefaultlib` 选项编译，则会跳过 4 和 5。至于 runpath，和 rpath 类似，都是二进制（ELF）文件的动态 section 属性（分别为 `DT_RUNPATH` 和 `DT_RPATH`），唯一区别就是是否优先于 `LD_LIBRARY_PATH` 来查找。这里就不详述了。

## 总结

至此，读者对如何编译/部署动态库，以及动态库之间的依赖关系应该有了一定的认识。

相比而言，静态链接，静态链接部署简单，像 Golang 这种语言直接全部静态链接，受到了不少用户的青睐，而且占用体积大在现代已经几乎不再是需要特别考虑的问题。

但动态库有动态库的好处，比如在大型项目有多个组件依赖时，如果全部静态链接，则每次修改依赖的模块，都要将主模块重新编译一遍，对于 C++ 这种编译速度可能会非常耗时的语言是灾难性的。

另外，提供插件式接口给解释型语言（比如 Python 和 PHP）来调用时，动态库是必须的，解释器可以动态加载动态库。如果使用静态链接，恐怕没人愿意每换一个插件就要将解释器重新编译一遍。