---
title: C标准I/O库自定义缓冲区
date: 2018-10-25 04:39:00
tags:
  - C++
top: 1
---
# 1. 标准I/O库的缓冲机制
Linux上C标准I/O库封装了底层的`write()`、`read()`等系统调用，以减少系统调用次数。
比如现在需要打印10000次"hello world"，如果直接用系统调用
```
for (int i = 0; i < 10000; i++)
    write(STDOUT_FILENO, "hello world", 11);
```
那么系统调用`write()`会执行10000次。
如果用C标准库的`printf()`或`fputs()`之类，比如
```
for (int i = 0; i < 10000; i++)
    fputs("hello world", stdout);
```
在我的系统(Ubuntu 16.04)上`write()`只会执行108次，查看`write()`调用次数可以借助`strace`工具，比如我是用
```
strace ./a.out 2>err
grep "write(" err | wc -l
```
查看的，如果想实际看看缓冲区大小，可以直接打开err文件，可以看到下列输出
```
write(1, "hello worldhello worldhello worl"..., 1024) = 1024
...(省略若干行)
write(1, "orldhello worldhello worldhello "..., 1024) = 1024
write(1, "rldhello worldhello worldhello w"..., 432) = 432
```
每次执行C标准库的打印函数时，并未立刻调用`write()`将字符串打印到屏幕上(严谨点说，`write()`也只不过是将内容递交给内核缓冲区)，而是等到填满缓冲区后再调用`write()`。
通过这样的缓冲机制，可以减少系统调用的次数，普通函数调用不涉及到用户态和内核态的切换，因此开销远低于系统调用。
缓冲区大小没有严格定义，即使在同一系统上，打印到不同文件中的缓冲区大小也不一样。比如将输出重定向到普通文件
```
# strace ./a.out >out 2>err
# grep "write(" err | wc -l
27
```
查看err文件可以看到缓冲区大小变成了4096。
再就是最后一次`write()`，可以发现即使未等到填满缓冲区，仍然打印出来了，原因是程序正常终止（调用`exit()`）时会关闭所有文件流，从而导致缓冲区被刷新，效果等价于调用`fflush()`。
比如在每次`fputs()`之后加上`fflush(stdout);`，可以发现`write()`调用执行了10000次。
或者在for循环之后，加上`write(STDOUT_FILEFNO, "exit(0)", 7)`，可以发现err文件中最后两次`write()`为
```
write(1, "exit(0)", 7)                  = 7
write(1, "rldhello worldhello worldhello w"..., 432) = 432
```
上述缓冲方式为全缓冲，即等缓冲区填满才调用底层I/O函数。而常见的标准输入流和标准输出流都是采用行缓冲，即遇到换行符才调用底层I/O函数。
比如调用`fgets()`函数时，等键盘敲入回车键时函数才会返回，如果是直接用`read()`系统调用，则是等待固定字符数量被键盘敲入才返回。
又比如调用`printf("hello\n");`时，由于末尾有换行符，因此会调用`write()`将其打印到屏幕上而不是等输出缓冲区填满。
另外，标准错误流是全缓冲的。

# 2. 使用自定义缓冲区
正是因为标准库没有严格定义缓冲区的设计，因此才催生了用户自定义缓冲区的需求。
通过`man setbuf`可以看到手册对下列函数的说明
```
void setbuf(FILE *stream, char *buf);
void setbuffer(FILE *stream, char *buf, size_t size);
void setlinebuf(FILE *stream);
int setvbuf(FILE *stream, char *buf, int mode, size_t size);
```
其中`setbuffer()`和`setlinebuf()`需要定义`_BSD_SOURCE`宏。
- `stream`: 缓冲区对应的文件流
- `buf`: 自定义缓冲区，若为`NULL`则使用系统自带缓冲区；
- `mode`: 缓冲方式，下列三个宏之一
  | 宏 | 意义 |
  | :- | :--- |
  | `\_IONBF` | 无缓冲，此时buf和size失去了意义 |
  | `\_IOLBF` | 行缓冲 |
  | `\_IOFBF` | 全缓冲 |
- `size`: 缓冲区大小至少有size字节

本质上前3个库函数都是调用`setvbuf()`函数，对应关系如下
| 原始调用 | 等价调用 |
| :------- | :------- |
| setbuf(stream, buf) | setvbuf(stream, buf, buf ? \_IOFBF : \_IONBF, BUFSIZ) |
| setbuffer(stream, buf, size) | setvbuf(stream, buf, buf ? \_IOFBF : \_IONBF, size) |
| setlinebuf(stream) | setvbuf(stream, NULL, \_IOLBF, 0) |

可见`setbuf`和`setbuffer`是用的自定义缓冲区，区别只是前者使用标准库的宏`BUFSIZ`作为缓冲区大小，后者使用`size`参数。
`setlinebuf`则仍然使用标准库的缓冲区，只不过缓冲机制改成行缓冲。

# 3. 自定义缓冲区的陷阱
一般情况下没必要自定义缓冲区，除非能证明在当前场景下你的自定义缓冲区有性能优势，以及这个性能提升能解决系统的效率瓶颈。毕竟自定义缓冲区会遇到一些问题，这也是这章要讲的。
## 3.1 缓冲区的生存周期
首先是手册上给出的示例
```
#include <stdio.h>

int
main(void)
{
    char buf[BUFSIZ];
    setbuf(stdin, buf);
    printf("Hello, world!\n");
    return 0;
}
```
在`stream`关闭之前，`buf`必须存在，否则在关闭时会出问题。在第1章也提过，进程终止时会导致缓冲区被刷新。而`main()`结束在这些额外操作之前，此时栈上的`buf`空间会被回收。
正确的方式是将缓冲区定义为全局或静态变量，或者采用`malloc()`动态申请内存作为缓冲区，并保证在`stream`被关闭之后才`free()`释放缓冲区占用内存。
PS: 虽然在我的系统上实际执行这段代码运行正常，设置在非main函数中定义局部缓冲区仍然运行正常，但这实质上是不合法的。
## 3.2 缓冲区溢出
它指定的是缓冲区的最小大小，然而即使你设定的`size`比你的`buf`数组大小要大，该函数调用也不会出错，这样就造成了缓冲区溢出的问题。
给出下列代码
```
#include <stdio.h>
#include <stdlib.h>

static char g_buf[4];
static int g_i = 0;

int main() {
    // 模拟缓冲区溢出，全缓冲模式
    setvbuf(stdout, g_buf, _IOFBF, sizeof(g_buf) + 4);
    for (; g_i < 5; g_i++)
        printf("%d\n", g_i + 10000);
    return 0;
}
```
输出结果是
```
$ ./a.out
10000
10011
```
来解释下原因，这里我设置的`size`为8，也就是说，标准库把`g_buf`开始的8个字节都当做缓冲区，由于`g_buf`本身只有4个字节，因此会把后面4个字节，也就是`g_i`所占的内存作为缓冲区。
使用gdb调试
```
Breakpoint 1, main () at stdout.c:12
12          printf("%d\n", g_i + 10000);
(gdb) p g_buf
$1 = "\000\000\000"
(gdb) x/8bx &g_buf[0]
0x60104c <g_buf>:   0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
(gdb) c
Continuing.
10000
Breakpoint 1, main () at stdout.c:12
12          printf("%d\n", g_i + 10000);
(gdb) p g_buf
$2 = "\n\000\000"
(gdb) x/8xb &g_buf[0]
0x60104c <g_buf>:   0x0a    0x00    0x00    0x00    0x01    0x00    0x00    0x00
(gdb) n
11      for (; g_i < 5; g_i++)
(gdb) x/8cb &g_buf[0]
0x60104c <g_buf>:   10 '\n' 49 '1'  48 '0'  48 '0'  48 '0'  49 '1'  10 '\n' 0 '\000'
(gdb) x/8xb &g_buf[0]
0x60104c <g_buf>:   0x0a    0x31    0x30    0x30    0x30    0x31    0x0a    0x00
(gdb) p/x g_i
$10 = 0xa3130
```
执行完第1次循环后，`g_i`的值为1，由于系统是小端的，所以低位的0x01放在低地址，也就是`g_buf[4]`的位置上。
执行完第2次循环后，缓冲区后4个字节变成了`0x30 0x31 0x0a 0x00`，因此`g_i`的值变成了0x000a3130，`g_i < 5`不再成立，跳出了循环。

# 4. 再谈全缓冲方式
本来在写博客的时候到上一节就戛然而止了，但是突然发现一个问题。
注意到3.2中第2次跳出循环时，缓冲区中的内容："\n10001\n"，第1次`write()`的只有5个字节"10000"，而缓冲区大小是8。
也就是说，全缓冲并未按照我在第1节中我说明的那样去运作，即等到缓冲区满了才刷新。
用`strace`查看`write()`的调用情况验证了我的观点
```
write(1, "10000", 5)                    = 5
write(1, "\n10011\n", 7)                = 7
```
更改代码重新运行，看看打印10000到10005是怎样？
```
for (g_i = 10000; g_i < 10005; g_i++)
    printf("%d\n", g_i);
```
```
write(1, "10000", 5)                    = 5
write(1, "\n10001\n1", 8)               = 8
write(1, "0002", 4)                     = 4
write(1, "\n10003\n1", 8)               = 8
write(1, "0004", 4)                     = 4
write(1, "\n", 1)                       = 1
```
这次可以看到填满缓冲区再打印的情况，但只是中间2次。由于`printf`是格式化输出，在把int转换成字符串时会计算长度，会不会是这个原因呢？
修改代码如下
```
printf("%s", "10001\n");
printf("%s", "10002\n");
printf("%s", "10003\n");
printf("%s", "10004\n");
printf("%s", "10005\n");
```
```
write(1, "10001", 5)                    = 5
write(1, "\n10002\n1", 8)               = 8
write(1, "0003", 4)                     = 4
write(1, "\n10004\n1", 8)               = 8
write(1, "0005", 4)                     = 4
write(1, "\n", 1)                       = 1
```
和预想的不一样，那么直接改成`fputs("10001\n", stdout)`呢？
```
write(1, "10001\n", 6)                  = 6
write(1, "10002\n10", 8)                = 8
write(1, "003\n", 4)                    = 4
write(1, "10004\n10", 8)                = 8
write(1, "005\n", 4)                    = 4
```
说明全缓冲模式下`printf("%s", s)`和`fputs(s, stdout)`并不是完全等价的。
看来只有去看源码了，查看glibc版本
```
# ldd a.out | grep libc
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f49ed348000)
# /lib/x86_64-linux-gnu/libc.so.6
GNU C Library (Ubuntu GLIBC 2.23-0ubuntu9) stable release version 2.23, by Roland McGrath et al.
```
去[glibc官网](https://www.gnu.org/software/libc/)下载2.23版本的源码。
具体地源码阅读就不在本篇讲述，没有习惯glibc的一些宏，读起来还是很吃力的
这里简单讲一点，找到`libio/vsnprintf.c`
```
int
_IO_vsnprintf (char *string, _IO_size_t maxlen, const char *format,
	       _IO_va_list args)
{
  // 该结构体包含2个字段
  // _IO_strfile f;  // 文件相关结构，暂不深入
  // /* This is used for the characters which do not fit in the buffer
  //    provided by the user.  */
  // 重点在这，定义了额外的缓冲区来保存用户缓冲区可能无法保存的字符
  // 比如snprintf(buf, 2, "%d", 10000);
  // 用户缓冲区buf光2个字节无法保存"10000"这5个字符，多余的就存在overflow_buf中
  // char overflow_buf[64];
  _IO_strnfile sf;
  int ret;
#ifdef _IO_MTSAFE_IO
  sf.f._sbf._f._lock = NULL;
#endif

  /* We need to handle the special case where MAXLEN is 0.  Use the
     overflow buffer right from the start.  */
  // 通过snprintf(NULL, 0, format, ...)取得格式化后字符串实际大小就是在
  // 这里实现的。利用了overflow缓冲区，并重置maxlen为其大小(64)。
  // 因此string参数此时没有意义，可以随便设置，并不一定需要为NULL。
  if (maxlen == 0)
    {
      string = sf.overflow_buf;
      maxlen = sizeof (sf.overflow_buf);
    }

  _IO_no_init (&sf.f._sbf._f, _IO_USER_LOCK, -1, NULL, NULL);
  _IO_JUMPS (&sf.f._sbf) = &_IO_strn_jumps;
  string[0] = '\0';
  _IO_str_init_static_internal (&sf.f, string, maxlen - 1, string);
  // TODO: _IO_vfprintf实质上是vfprintf，参考stdio-common/vfprintf.c
  // 具体实现有400多行，比较麻烦
  ret = _IO_vfprintf (&sf.f._sbf._f, format, args);
  if (sf.f._sbf._f._IO_buf_base != sf.overflow_buf)
    *sf.f._sbf._f._IO_write_ptr = '\0';
  return ret;
}
```
之后有空再详细看下怎么实现的，留下的问题：
C格式化主要解析的就是字符串和整型，`overflow_buf`只有64个字节，对于整型是足够了，
但是对于较长的字符串是如何保存的呢？
或者说并不是用于保存多出字符的，而是为了计算理论长度的临时缓冲区？
比如对如下的格式化字符串和格式化参数
```
const char* format = "%s %d";
const char* s = "hello";
int i = 100;
```
s就直接`strlen()`计算长度，i则`strtol()`到`overflow_buf`中再计算长度，最后求和？
再就是问题的关键，全缓冲模式下，假设已经判断出多出字符的数量，如何保存中断位置呢？
我也有个大概思路，如果是在字符串的位置中断，则尽可能用s填充缓冲区剩余部分，然后移动s指针。
如果是在整数中断，则`overflow_buf`记录整数转换成的字符串，然后用`"%s"`替换format的`"%d"`。
总之还没有非常明确的思路，以后有空自己写个`Buffer`类。
