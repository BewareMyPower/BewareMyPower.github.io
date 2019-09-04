---
title: 链接选项RPATH以及在cmake和gcc中的使用
date: 2019-09-05 01:12:42
tags:
  - C++
top: 1
---
## 前言

毕业前帮师兄写框架程序时，以及最近折腾公司内部项目的编译，都遇到一些以前没有遇到的问题，这里简单地记一些。

本文要讲的也就是 `rpath` ，即 relative path 的缩写，最初遇到这个坑时是在写 `cmake` 时，直接 `make` 生成的程序能够链接到指定的动态库，但是 `make install` 之后发现就链接失效了。

## 示例项目

这里举个简单的例子来复现，目录结构为：

```bash
$ tree .
.
├── CMakeLists.txt
├── example
│   ├── CMakeLists.txt
│   └── main.c
└── src
    ├── CMakeLists.txt
    ├── foo.c
    └── foo.h
```

代码组织方式是： `src` 目录为库目录，其源码会编译成动态库 `libfoo.so`，`example` 为示例目录，其源码会包含 `src` 目录的头文件，并链接到动态库 `libfoo.so` 。具体代码也不长，依次贴出：

`./CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.5)
project(example C)
set(CMAKE_CFLAGS -g -Wall)
add_subdirectory(src)
add_subdirectory(example)
```

`./src/CMakeLists.txt`：

```cmake
add_library(foo SHARED foo.c)
install(TARGETS foo LIBRARY DESTINATION lib)
```

`./src/foo.h`：

```c
#ifndef FOO_H
#define FOO_H

void foo();

#endif  // FOO_H
```

`./src/foo.c`：

```c
#include "foo.h"
#include <stdio.h>

void foo() { printf("foo\n"); }
```

`./example/CMakeLists`：

```cmake
add_executable(main main.c)
include_directories(../src)
target_link_libraries(main foo)
install(TARGETS main RUNTIME DESTINATION bin)
```

`./example/main.c`：

```c
#include "foo.h"

int main() {
  foo();
  return 0;
}
```

## 编译、安装及测试

老方法，新建一个临时目录用来存放中间文件，以下命令在项目根目录下执行，将动态库和可执行程序安装到根目录下的 `lib` 和 `bin` 目录，然后回到根目录：

```bash
mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=..
make && make install && cd ..
```

此时目录结构为：

```bash
$ tree . -I build
.
├── bin
│   └── main
├── CMakeLists.txt
├── example
│   ├── CMakeLists.txt
│   └── main.c
├── lib
│   └── libfoo.so
└── src
    ├── CMakeLists.txt
    ├── foo.c
    └── foo.h
```

首先我们运行 `build` 目录下的可执行文件，并查看其连接的 `libfoo.so` 路径：

```bash
$ ./build/example/main 
foo
$ ldd ./build/example/main | grep foo
	libfoo.so => /home/xyz/RPATH/build/src/libfoo.so (0x00007f5d41c23000)
```

这里 `/home/xyz/RPATH` 是我的项目根目录绝对路径，可以发现 `make` 生成的可执行文件，链接的是绝对路径，并且运行也没问题。但是再看看安装后的可执行文件和链接的库：

```bash
$ ./bin/main 
./bin/main: error while loading shared libraries: libfoo.so: cannot open shared object file: No such file or directory
$ ldd ./bin/main | grep foo
	libfoo.so => not found
```

我们会发现可执行文件失去了链接，因此要运行 `main` ，必须手动将动态库添加到系统路径中：

```bash
$ LD_LIBRARY_PATH=./lib ./bin/main 
foo
```

## 问题在哪

这个问题最初出现在我帮师兄写的框架当中，当时也找到了stackoverflow上的讨论帖：[cmake: “make install” does not link against libraries in Ubuntu](https://stackoverflow.com/questions/42695839/cmake-make-install-does-not-link-against-libraries-in-ubuntu)

简单翻译下：

系统首先会在 `/etc/ld.so.conf` 文件配置的路径中寻找动态库（在我的系统上该文件记录的是 `/etc/ld.so.conf.d` 目录下的所有 `*.conf` 文件），如果找不到，则有以下4个选项：

1. 将库安装到系统默认路径比如 `/lib` 和 `/usr/lib`（但可能因为没有权限而无法实施）；
2. 编辑系统范围的搜索路径（同样可能因为没有权限而无法实施）；
3. 设置 `LD_LIBRARY_PATH`（就像我们上节末尾所做的，但它会覆盖系统路径，也就是说可能会优先选择自己的库而不是系统路径的同名库）；
4. 设置 `RPATH`，告诉可执行文件该到哪寻找它的库。

OK，现在来看问题的产生原因：`RPATH` 在 `make install` 后会被自动地清除。为什么会这样呢？因为 `cmake` 安装的可执行文件和动态库的相对路径，可能和 `make` 生成的不一样，因此无法自动记住。

## cmake的解决方法

当然，`cmake` 本身也提供了解决方法，参见：[RPATH handling](https://gitlab.kitware.com/cmake/community/wikis/doc/cmake/RPATH-handling)。

不想看官网的长篇大论的话，针对本文的示例，在项目根目录的 `CMakeLists.txt` 中添加：

```cmake
set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
```

`make install` 安装时可以看到如下提示信息：

```
-- Installing: /home/xyz/RPATH/bin/main
-- Set runtime path of "/home/xyz/RPATH/bin/main" to "/home/xyz/RPATH/lib"
```

这种方式还是设置的绝对路径，也就是 `cmake` 安装目录下的 `lib` 子目录，然后可以发现安装后的 `main` 成功链接到了 `libfoo.so`，并且改变 `main` 路径，仍然可以链接到 `libfoo.so`：

```bash
$ ldd bin/main | grep foo
	libfoo.so => /home/xyz/RPATH/lib/libfoo.so (0x00007f60818c2000)
$ mv bin/main .
$ ldd main | grep foo
	libfoo.so => /home/xyz/RPATH/lib/libfoo.so (0x00007f6ff516f000)
```

类似地，我们可以把 `CMAKE_INSTALL_RPATH` 指定为相对路径 `../lib`，但这样的话局限性比较大，也就是说必须保证动态库在 `$PWD/../lib` 下，比如按这种方式编译安装后：

```bash
$ ldd ./bin/main | grep foo
	libfoo.so => not found
$ cd bin
$ ldd ./main | grep foo
	libfoo.so => ../lib/libfoo.so (0x00007f12fe0c7000)
```

但这种方式也有个优点，也就是说哦，只要动态库在当前工作目录的相对路径 `../lib` 下，就能链接到该动态库，此时可以写一个脚本，和 `main` 放在同一目录：

```bash
#!/bin/bash
SHELL_DIR=$(cd $(dirname $0) && echo $PWD)
cd $SHELL_DIR && ./main
```

第一行是取得脚本目录的绝对路径，第2行是进入该路径，这样只要动态库在可执行文件（以及运行脚本）的相对路径 `../lib` 下，无论从哪个目录调用该脚本，都能成功使可执行文件链接到该动态库：

```bash
$ ls bin/
main  run.sh
$ ls lib/
libfoo.so
$ ./bin/run.sh 
foo
$ cd bin/ && ./run.sh && cd -
foo
/home/xyz/RPATH
$ mkdir temp
$ mv bin/ lib/ temp/
$ ./temp/bin/run.sh 
foo
```

不过既然借助了辅助脚本了，实际上在脚本里手动设置 `LD_LIBRARY_PATH` 为相对路径看起来更简单一些。

## GCC的解决方法

为什么说到这个呢？因为我最近在使用公司内部项目的时候，发现自己的测试代码一直出错，查看日志，竟然源码路径来自其他用户的个人目录。也就是说是其他用户编译了动态库，然后使用超级权限将其安装到了系统目录。

对于这种情况，可以用 `LD_LIBRARY_PATH` 覆盖，但是如果修改了目录后，每次都要重新设置 `LD_LIBRARY_PATH`，此时用 `gcc` 的链接选项就行了，还是对这个项目，手动用 `gcc` 进行编译：

```bash
$ cd src/
$ gcc -c -fPIC foo.c 
$ gcc -shared foo.o -o libfoo.so
$ cd ../example/
$ gcc main.c -I../src -L../src -lfoo -Wl,-rpath=../src
```

注意，`-Wl,-rpath` 这个选项必不可少，它指定了 `RPATH` 的相对路径，为此，我将原来的 `libfoo.so` 放在系统目录 `/lib64` 下，然后修改 `foo.c` （打印 `"new foo"` 而不是 `"foo"`）后编译成动态库放在 `src` 目录下，测试如下：

```bash
$ cd example/
$ gcc main.c -I../src -L../src -lfoo
$ ./a.out 
foo
$ gcc main.c -I../src -L../src -lfoo -Wl,-rpath=../src
$ ./a.out 
new foo
$ cd .. && mkdir temp && mv src/ example/ temp
$ tree temp/
temp/
├── example
│   ├── a.out
│   ├── CMakeLists.txt
│   └── main.c
└── src
    ├── CMakeLists.txt
    ├── foo.c
    ├── foo.h
    ├── foo.o
    └── libfoo.so
$ ./temp/example/a.out 
foo
$ cd temp/example/ && ./a.out
new foo
```

和刚才 `cmake` 设置 `RPATH` 的测试结果一样，只要当前工作目录满足和动态库所在目录的相对路径是 `RPATH`，那么运行可执行文件所链接到的动态库就是相对路径的动态库。
