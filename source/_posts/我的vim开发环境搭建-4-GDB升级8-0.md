---
title: '我的vim开发环境搭建(4): GDB升级8.0'
date: 2019-07-26 23:16:10
tags:
  - 搭环境
  - vim
top: 1
---
## 0. 前言
因为CentOS 6的GDB版本太老，所以在调试我自己安装的高版本GCC编译的程序时，很多时候打印不出变量来，所以需要升级GDB。

之前在CentOS 6上下载GDB源码编译总是出问题，网上也搜不到解决原因，不过最后自己折腾出来原因了。问题关键在于configure阶段，正确的宏没有检测出来，导致条件编译出错，比如：

```
#ifdef HAVE_XXX
// 某些类型的定义或相应头文件的包含...
#endif
```

如果`HAVE_XXX`宏没有检测出来的话，本应该定义的类型就没有了。这既可能造成编译阶段找不到类型定义，也可能造成链接阶段找不到函数的定义(因为某些函数的参数是typedef别名或者宏)。

明白这一点后，只需要make时根据错误在对应代码中删除相应的`#ifdef`或者`#ifndef`块就行，这里以GDB 8.0的编译为例。

## 1. 工具和说明

有时候代码出错，提示某些类型或宏定义不存在。这时可以查找这些宏是在哪个文件中：

```
find . -name "*.h" | xargs grep -n "XXX"
```

有时候不在GDB源码目录下，而是在系统头文件中，比如`B0`就是`termios.h`中的定义，此时可以利用搜索引擎搜索，并YouCompleteMe的代码跳转功能来确认，有些头文件是`sys/xxx.h`而非`xxx.h`（旧版本的系统），这些GDB源码通过条件编译判断到底是包含哪个。

此外，跳转功能还是要结合ctags，YCM好用在于它是基于语义的分析，但是得设置包含路径，默认包含路径只有当前目录，如果直接包含子目录头文件而不给出子目录名（编译时`-I`指定子目录路径即可）。ctags无脑找标签，所以有时候直接包含子目录的头文件，即使不做设置也可以跳转。ctags只需要在源码目录下`ctags -R .`生成tags文件，之后`Ctrl+]`就可以跳转了。

一般提示`xxx.c`出错，先仔细看看对应的`xxx.h`头文件，是否因为条件编译导致某些类型定义找不到。如果该头文件找不到，就可能通过上面的方法找到类型定义处，基本上一大堆宏都是在`config.h`中。

另外，错误提示的文件目录为GDB源码解压后的gdb子目录。

## 2. 出错解决

### 2.1. 编译错误处理
> In file included from inf-ptrace.c:27:0:
nat/gdb_ptrace.h:150:19: error: ‘PTRACE_TYPE_ARG1’ was not declared in this scope
          ptrace ((PTRACE_TYPE_ARG1) request, pid, addr, data)

仅给出其中一处代表性错误。进入`nat/gdb_ptrace.h`可以看到这样的语句：

```
#ifdef HAVE_PTRACE_H
# include <ptrace.h>
#elif defined(HAVE_SYS_PTRACE_H)
# include <sys/ptrace.h>
#endif
```

我的系统上是`sys/ptrace.h`，那么明显条件编译出问题了，删掉`#include <sys/ptrace.h>`的另外4行即可。

类似地，去掉`#ifndef HAVE_DECL_PTRACE`。然后包含`../config.h`头文件（因为在`gdb`目录下，而`gdb_ptrace.h`在`gdb/nat`目录下），这个头文件必须在`sys/ptrace.h`下方，因为相关地宏，因为其中定义地宏在后面地头文件中也可能用到。

修改后还有个错误
> inf-ptrace.c: In function ‘void inf_ptrace_interrupt(target_ops*, ptid_t)’:
inf-ptrace.c:304:34: error: ‘inferior_process_group’ was not declared in this scope
   kill (-inferior_process_group (), SIGINT);
   
利用ctags找到该函数定义在`inflow.c`中：

```
   93 #ifdef PROCESS_GROUP_TYPE
   94 
   95 /* Return the process group of the current inferior.  */
   96 
   97 PROCESS_GROUP_TYPE
   98 inferior_process_group (void)
   99 {
  100   return get_inflow_inferior_data (current_inferior ())->process_group;
  101 }
  102 #endif
```

嗯估计又是`PROCESS_GROUP_TYPE`找不到，该类型在`inflow.h`中定义，用条件编译宏包括起来了：

```
   25 #ifdef HAVE_TERMIOS
   26 # define PROCESS_GROUP_TYPE pid_t
   27 #elif defined (HAVE_TERMIO) || defined (HAVE_SGTTY)
   28 # define PROCESS_GROUP_TYPE int
   29 #endif 
```

原因是`HAVE_TERMIOS`没检测出来保。留第26行，其余4行删掉。可以看到上面包含了`gdb_termios.h`中（在`gdb/common`目录下），更根本的解决方法是在这个头文件中定义宏`HAVE_TERMIOS`。（也能解决后面其他和termios相关的问题）

下文就不详细给出错误信息和分析了，简单描述错误和给出解决方法。可能比较复杂，最简单的应该是在`def.h`中手动设置条件编译宏，不过反正能编译就行，我都是一个窗口make，另一个窗口根据make出错信息编辑代码。其实懂了出错原理后后自己应该也能找出更简单的解决方法。

### 2.2. 链接错误处理

> value.c:3709: undefined reference to `store_typed_floating(void*, type const*, double)'
collect2: error: ld returned 1 exit status

打开`value.c`找到该函数，ctags跳转到`doublest.c`中，发现函数第3个参数是`DOUBLEST`，也就是没找到这个宏或者类型别名的定义。继续ctags跳转到`DOUBLEST`的定义位置，果然也被条件编译宏包含起来了。从错误信息看它应该是`double`的类型别名，因此保留`typedef double DOUBLEST;`的分支。

### 2.3. 其他编译错误处理

`ser-tcp.c`：
- 去掉`HAVE_SYS_IOCTL_H`的`#ifdef`；
- 删除`typedef __socklen_t socklen_t`的代码。

`gregset.h`：
- 去掉`#HAVE_SYS_PROCFS_H`的`#ifdef`块（使得`sys/procfs.h`被包含）；
- 第一处`grepset_t`改成`elf_gregset_t`；
- 第一处`fpregset_t`改成`elf_fpregset_t`（这两个类型都定义在`sys/procfs.h`中）；

`gdb_proc_service.h`：
- 删除`lwpid_t`的定义；
- 包含`common/common-defs.h`；

`gdb/gdb_wait.h`：保留条件编译的`sys/wait.h`分支；

`nat/amd64-linux-siginfo.c`：将`common-defs.h`作为第一个头文件；

`gdb_curses.h`：保留条件编译的`curses.h`分支；

在下列C文件中包含`config.h`作为第一个头文件；
- `auto-load.c`
- `doublest.c`
- `jit.c`
- `main.c`
- `top.c`
- `utils.c`

`common/signals.c`：
- 去掉`HAVE_SIGNAL_H`的`#ifdef`块；
- 包含`sys/signal.h`；

`gdb-server/remote-utils.c`：把`USE_WIN32API`,`__QNX__`之外（其实都在它们前面定义了）的`#if`/`#endif`块全部去掉。

`gdb-server/linux-low.h`：
- 删除`Elf_32_auxv_t`和`Elf64_auxv_t`的定义；
- 删除`HAVE_LINUX_REGSETS`的`#ifdef`块（有很多处）；
- 删除`USE_THREAD_DB`的`#ifdef`块；

## 3. 安装后出错

> common/filestuff.c:401: internal-error: int gdb_pipe_cloexec(int*): pipe not available on this host

找到如下代码：
```
  385 #ifdef HAVE_PIPE2
  386   result = pipe2 (filedes, O_CLOEXEC);
  387   if (result != -1)
  388     {
  389       maybe_mark_cloexec (filedes[0]);
  390       maybe_mark_cloexec (filedes[1]);
  391     }
  392 #else
  393 #ifdef HAVE_PIPE
  394   result = pipe (filedes);
  395   if (result != -1)
  396     {
  397       mark_cloexec (filedes[0]);
  398       mark_cloexec (filedes[1]);
  399     }
  400 #else /* HAVE_PIPE */
  401   gdb_assert_not_reached (_("pipe not available on this host"));
  402 #endif /* HAVE_PIPE */
  403 #endif /* HAVE_PIPE2 */
```

还是条件编译宏的问题，触发了`gdb_assert_not_reached`，这里保留`HAVE_PIPE`分支就行了。重新编译安装就OK。

安装时会有个警告导致出错：

> gdb-8.0/missing: line 81: makeinfo: command not found

但是并不影响GDB的正常使用，也没必要特地安装`makeinfo`。至于环境变量和prefix这种东西，就不多讲了。

## 4. 启用termdebug

要求vim 8.1和GDB 7.12以上，命令`:packadd termdebug`加载termdebug插件即可。然后在vim中就可启动termdebug调试程序：`:Termdebug <your-program>`。

但是默认是横向切分窗口，很不喜欢，vim中`:help termdebug`可以查看文档，默认会打开gdb窗口和程序窗口，搜索vertical就可以找到配置，只需要在`~/.vimrc`中设置如下属性即可：

```
let g:termdebug_wide = 163 # 列宽，可调整
```

运行效果如下：

![termdebug示例](./termdebug示例.jpg)
