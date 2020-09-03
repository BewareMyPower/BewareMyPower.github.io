---
title: Linux C++ 内存分配 SIGKILL 问题
date: 2020-09-03 16:43:26
tags:
  - C++
top: 1
---
## 背景

在代码里看到发现某对象有个 buffer 每次会分配较大内存（定义一个大的 `vector`），担心如果对象太多会导致 `vector` 内存分配失败，于是自己写了个简单测试看能不能捕获 `bad_alloc`。

```c++
#include <limits.h>
#include <iostream>
#include <vector>
using namespace std;

constexpr int size = 1024 * 1024;  // 1 MiB

int main() {
  int i = 0;
  try {
    for (; i < INT_MAX - 1; i++) {
      new vector<int>(size);
    }
    cout << "Success" << endl;
  } catch (const std::bad_alloc& e) {
    cout << "Failed at " << i << "th allocate" << endl;
  }
  return 0;
}
```

比较遗憾的是，运行直接显示 Killed 崩溃，在 Linux 上也就是收到了 `SIGKILL` 信号。这让我想起来当初压测某 C++ 客户端时，对于 500 个分区的 topic，在限制内存为 1 GB 的容器中运行 C++ 客户端，很多都会直接 Killed，日志里也没异常信息。

## 内存分配

C 标准库的内存分配是使用 `malloc`（以及 `calloc`/`realloc`，这三者类似，这里不讲区别），它的使用很简单，分配失败就返回 `NULL`，见 man page：

```
       The malloc() function allocates size bytes and returns a pointer to
       the allocated memory.  The memory is not initialized.  If size is 0,
       then malloc() returns either NULL, or a unique pointer value that can
       later be successfully passed to free().
```

C++ 的 `new` 则是分为两步，首先是调用 `operator new` 分配内存，然后就地构造（placement new），对于基本类型就是内存拷贝，对于类而言调用构造函数。

一般而言分配内存是前一步，大概是直接包的 `malloc`，相比而言，C++ 可以重写 `operator new` 来执行自定义内存分配策略，虽然 C 替代 `malloc` 也可以，但不一定通用。C++ `new` 失败不会返回空指针，而是会抛出 `std::bad_alloc` 异常。

STL 内存分配则又是一回事，为了能让用户能更精确掌控内存分配（而不是受到 `new` 的制约），它提供了一个 `std::allocator<T>` 类模板进行默认的内存分配，而用户可以自行实现接口作为自己的内存分配策略。当然，是不是包的 `operator new` 我也没去细看。

怀疑是 STL allocator 的问题，于是这里用 `new` 来替代 STL allocator 看看，由于 `vector` 一般实现是 3 个指针，所以这里也模拟了实现：

```c++
#include <limits.h>
#include <iostream>
using namespace std;

constexpr int size = 1024 * 1024;  // 1 MiB

struct VectorInt {
  int* start;
  int* finish;
  int* end;
};

int main() {
  int i = 0;
  try {
    for (; ; i++) {
      auto p = new VectorInt;
      p->start = new int[size];
    }
    cout << "Success" << endl;
  } catch (const std::bad_alloc& e) {
    cout << "Failed at " << i << "th allocate" << endl;
  }
  return 0;
}
```

结果还是收到 `SIGKILL` 了。把 `new` 改成 `malloc` 并检查返回值是否为空，结果也是一样。

## malloc 真的会返回 NULL 吗？

虽然答案显然是会，毕竟如果不会，那肯定不会是我第一个发现。所以我简单试了下：

```c++
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char* argv[]) {
  for (int i = 0; i < 1024 * 1024 * 1024; i++) {
    int* p = (int*)malloc(sizeof(int));
    if (!p) {
      printf("[%d] malloc failed: %s\n", i, strerror(errno));
      return 1;
    }
  }
  return 0;
}
```

还是收到 `SIGKILL` 了。这里我想起了之前看《STL 源码剖析》时，SGI STL 内存分配器对于小内存分配会进行优化，我想 `malloc` 应该也会，可能是我单次分配的内存（`sizeof(int)`，4 字节）太小了，于是我开始尝试增加单次分配的字节数来观察现象。

- 4 B：Killed

- 4 KiB：Killed

- 4 MiB：Killed

- 20 MiB：Killed

- 80 MiB：Killed

- 1 GiB：没有被 Killed，运行多次的结果如下：

  ```
  [131070] malloc failed: Cannot allocate memory
  [131071] malloc failed: Cannot allocate memory
  [131070] malloc failed: Cannot allocate memory
  [131071] malloc failed: Cannot allocate memory
  [131071] malloc failed: Cannot allocate memory
  ```

用 `strace` 重新运行查看系统调用：

```c
brk(0x27a60149a000)                     = 0x27a5c149a000
mmap(NULL, 1073876992, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = -1 ENOMEM (Cannot allocate memory)
mmap(NULL, 134217728, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_NORESERVE, -1, 0) = 0x3fe58dc4e000
munmap(0x3fe58dc4e000, 37429248)        = 0
munmap(0x3fe594000000, 29679616)        = 0
mprotect(0x3fe590000000, 135168, PROT_READ|PROT_WRITE) = 0
mmap(NULL, 1073745920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = -1 ENOMEM (Cannot allocate memory)
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 3), ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fe5a6a28000
write(1, "[106135] malloc failed: Cannot a"..., 47) = 47
```

在此之前是大量的 `brk`，也就是说每次运行 `malloc` 其实都是调用 `brk`。而导致 `malloc` 出错的是 `mmap`，它设置了错误码 `ENOMEM`，也就是 `malloc` 的 man page 里提到的错误码，表示内存不足。

将单词内存分配大小改回 128 MiB 试试（512 MiB 仍然能正常终止），最后几次系统调用：

```c
brk(0x181922339000)                     = 0x181922339000
brk(0x18192a339000 <ptrace(SYSCALL):No such process>
+++ killed by SIGKILL +++
```

发现并没有调用 `mmap`，而是持续调用 `brk`，直到收到 `SIGKILL`。

## brk 和 mmap 系统调用

从前一节可知，这里出问题的并不是 STL allocator，而是 `malloc` 本身。而之前一直在调用 `brk`。还是参考 man page，这里首先得弄清楚 program break 的概念，首先看下进程虚拟地址空间分布（最简单的模型）：

```
high address +--------------------+
             |                    |
             |                    |
             +--------------------+
             |       stack        |
             +---------+----------+
             |         |          |
             |         |          |
             |         v          |
             |                    |
             |                    |
             |                    |
             |         ^          |
             |         |          |
             |         |          |
             +---------+----------+
             |        heap        |
             +--------------------+
             | uninitialized data |
             |       (bss)        |
             +--------------------+
             |  initialized data  |
             +--------------------+
             |       text         |
 low address +--------------------+
```

program break 就是 bss 段上面的第一个地址。`brk` 的函数签名：

```c
#include <unistd.h>

int brk(void *addr);
void *sbrk(intptr_t increment);
```

`brk` 的作用是改变 program break 的位置为 `addr`。增加 `addr` 相当于申请空间。至于 `sbrk` 则是增加而非设置。

这里以一个最简单的内存申请为例：

```c++
  int* p = (int*)malloc(1024 * 1024 * 10);
  free(p);
```

对应系统调用：

```c
brk(0)                                  = 0x2112000
brk(0x2144000)                          = 0x2144000
mmap(NULL, 10489856, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fb6338d2000
```

第一次调用是 `brk(0)`，返回的不是 0 而是 0x2112000，也就是最初状态下 bss 段上面的第一个地址。后面移动到了新位置，增加了 program break，0x2144000 - 0x2112000 = 0x0032000，也就是 204800 字节，即 197 KiB，显然是不足 10 MiB。

然而接下来的 `mmap` 第二个参数，10489856 刚好是 10 MiB。`brk` 只是调整 program break，这只是虚拟地址空间的一个标记，如果要使用这块内存，需要用 `mmap` 进行映射。

```c++
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
int munmap(void *addr, size_t length);
```

- `addr`：为 `NULL` 则由内核选择首地址（满足页对齐），即使不为 `NULL`，也只是内核根据这个值作为提示信息来选择。
- `length`：映射的字节数，实际上会被提升至分页大小（`sysconf(_SC_PAGE_SIZE)`）的整数倍。
- `prot`：保护等级，为 `PROT_NONE`（无法访问）或者 `PROT_READ`（可读）/`PROT_WRITE`（可写）/`PROT_EXEC`（可执行）的组合。如果对这部分内存映射违反了保护等级，会产生 `SIGSEGV` 信号，也就是常见的段错误（segmentation fault）。
- `flags`：`MAP_PRIVATE`（私有映射）或者 `MAP_SHARED`（共享映射），前者是对其他进程不可见，后者是可见的，也就是共享内存。

最终内存会映射到从文件描述符 `fd` 对应的文件的偏移量 `offset` 处开始，注意 `offset` 也必须是分页大小的倍数。如果 `fd` 为 -1，则不映射到文件。

回顾之前的调用：

```c
mmap(NULL, 10489856, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fb6338d2000
```

也就是将物理内存的 10 MiB 给映射到堆的虚拟内存空间。

至于 `munmap` 即删除映射，`addr` 为对应 `mmap` 的返回值。

## malloc 小内存为何不返回 NULL？

刚才了解了 `brk` 和 `mmap` 结合前文内容，我们可知只有 `mmap` 进行内存映射失败（比如物理内存不够）`malloc` 才会返回 `NULL` 并将 `errno` 置为 `ENOMEM`。但是对于频繁申请小内存的情况，则是无限调用 `brk` 导致内存崩溃，并没有出现 `mmap`。回到第一节的代码（无限创建 1 MiB 的 `vector`），用 `strace` 观察的结果是无限调用 `mmap` 导致内存崩溃：

```c
mmap(NULL, 4198400, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f3fdead1000
/* ... */
mmap(NULL, 4198400, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f3e9ddcf000
+++ killed by SIGKILL +++
```

可见每次 `mmap` 长度上限也就 4198400（差不多 4 MiB）。

而假如 `vector` 大小改成 256 MiB，那么对应 `strace` 信息为：

```c
brk(0)                                  = 0xcbb000
brk(0xced000)                           = 0xced000
mmap(NULL, 1073745920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fdda8485000
mmap(NULL, 1073745920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fdd68484000
mmap(NULL, 1073745920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fdd28483000
mmap(NULL, 1073745920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fdce8482000
mmap(NULL, 1073745920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fdca8481000
mmap(NULL, 1073745920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = -1 ENOMEM (Cannot allocate memory)
brk(0x40ced000)                         = 0xced000
mmap(NULL, 1073876992, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = -1 ENOMEM (Cannot allocate memory)
mmap(NULL, 134217728, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_NORESERVE, -1, 0) = 0x7fdca0481000
munmap(0x7fdca0481000, 62386176)        = 0
munmap(0x7fdca8000000, 4722688)         = 0
mprotect(0x7fdca4000000, 135168, PROT_READ|PROT_WRITE) = 0
mmap(NULL, 1073745920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = -1 ENOMEM (Cannot allocate memory)
```

每次 `mmap` 长度是 1073745920，差不多 1 GiB。这么看，其实根本原因在于 `malloc` 实现（调用 `brk` 和 `mmap`），因为 `malloc` 允许较为高效的小内存分配（比如它不会因为 N 次 `int` 的分配就调用 N 次 `brk`）。

其实在 `malloc` 的 man page 上面就可以看到原因，以下内容摘自 NOTE 部分。

> 默认情况下，Linux 遵循乐观的内存分配策略，也就是 `malloc` 返回非 `NULL` 时，无法保证内存确实可用，如果系统 OOM 了，进程会被 OOM killer 干掉，也就是 OOM killer 发送 `SIGKILL` 信号。
>
> 通常 `malloc()` 从堆上分配内存，并使用 `sbrk()` 调整堆的大小。当分配的内存块字节数大于 `MMAP_THRESHOLD` 时，glibc 的 `malloc()` 实现使用 `mmap()` 进行私有匿名映射来分配内存。MMAP_THRESHOLD 默认 128 kB，但是可以通过 `mallopt()` 进行调整。

这也说明了，为啥分配大容量 `vector` 时是一直调用 `mmap`，而分配 `int` 时则是一直调用 `brk`。

`mmap` 的大小也有区别，每次映射 4198400 字节时，最后引向的是 `SIGKILL`，而每次映射 1073745920 字节时，最后得到的是错误码 `ENOMEM`。从上述文档可知，前者应该是触发 OOM killer 了，也就是映射了较小的内存，但不一定保证可用就返回了。也就是所谓的 **乐观内存分配策略** 导致的。

在 [sigkill while allocating memory in c](https://stackoverflow.com/questions/11779042/sigkill-while-allocating-memory-in-c) 这个讨论帖有讲，简单说就是：

- `new` 或 `malloc` 可能从内核拿到了一个不合法的地址，即使没有足够的内存，因为：
  1. 内核直到第一次访问时才分配地址
  2. 如果所有的 overcommited 内存都被使用，操作系统只能杀死其中的一个进程（也就是 OOM killer）。

OOM killer 可参考： https://linux-mm.org/OOM_Killer

简单说就是牺牲一个或多个进程，以便在其他进程故障时为系统释放进程。

那 overcommited 内存呢？它是一个内存分配策略，其值记录在 ` /proc/sys/vm/overcommit_memory` 文件。所谓 overcommit，就是过度提交，其实也就是刚才说的，内核直到第一次访问时才分配地址，否则即使请求的内存超过限制，只要不去访问，内核仍然当作可用。

[how to check if malloc overcommits memory](https://stackoverflow.com/questions/39721013/how-to-check-if-malloc-overcommits-memory) 给出了几种解决方法，方法 2 我是开启了 `--oom-kill-disable` 选项启动 docker，但还是一样。可能是哪里操作不对。方法 3（在 `malloc` 后进行写入）可能会花很长时间来分配较大内存，但是仍然可能被 kill。

## 总结

Linux 上，在某些情况下 `malloc` 即使返回非 `NULL` 值，由于内核的 overcommit 内存分配策略，也有可能导致这片内存不是可用的，这种内存泄漏累积的结果就是导致进程被 OOM killer 干掉（发送 `SIGKILL` 信号，无法被捕获）。

单次 `malloc` 申请较大的内存可以规避这点，因为即使有 overcommit 策略，也不可能超过可用太多。

总之，如果是在限制内存（比如 docker）的环境下运行 C++ 程序，内存分配失败不一定会以 `std::bad_alloc` 结束，此时会拿不到异常栈信息。虽然用 `dmesg` 能够找到 OOM 记录，但对调试的帮助仍然有限。这种时候，根本性的解决方案还是自定义内存分配器。