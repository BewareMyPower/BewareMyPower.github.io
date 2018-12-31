---
title: 'C++11中std::thread和pthread混用的坑'
date: 2018-12-31 16:30:14
tags:
  - C++
  - thread
top: 1
---
## 前言
早在之前我就因为混用`std::thread`和`pthread`出现bug时纠结过，当时还去查看过`std::thread`的实现源码，但后来还是没得出结果。现在看来主要原因是我没有找对位置，被C++又臭又长的模板元编程手法给转移了注意力。直到最近我把这个实现弄清楚后，自己造轮子时遇到了另一个bug，才回过头想起之前遇到的问题。

## 1. 在线程函数中调用`pthread_exit`和`pthread_join`
在C++11中，单靠`thread`是无法取得线程函数的返回值的，必须得借助`async`等异步设施，*Effective Modern C++*也推崇使用基于任务的并发编程而非基于线程。当然这个是后话了，
现在，我就是想直接取得线程函数的返回值，又不继续写`pthread`那套麻烦的线程函数，那么直接在`std::thread`的线程函数中调用`pthread` API？于是有了下面这段代码。
```
// test.cc
#include <pthread.h>
#include <stdio.h>
#include <thread>

int main() {
  // NOTE: make sure cast between long and void* is safe.
  std::thread t([] { pthread_exit(reinterpret_cast<void*>(1)); });
  void* ret;
  pthread_join(t.native_handle(), &ret);
  printf("thread exit with %ld\n", reinterpret_cast<long>(ret));
  return 0;
}
```
编译运行，结果如下
```
# g++ -std=c++11 test.cc -pthread && ./a.out                                                                                                              
thread exit with 1
terminate called without an active exception 
Aborted                                
```
嗯？是不是还要调用`t.join()`？于是我加上了`t.join()`。
```
thread exit with 1
terminate called after throwing an instance of 'std::system_error'
  what():  No such process
Aborted
```
这次是直接抛出异常了。好像都不行啊，那有没有解决方法呢？
在此之前，首先得明确一个问题，这里究竟该不该调用`t.join()`？

## 2. `std::terminate()`
`std::terminate()`在头文件`<exception>`中声明，函数签名如下
```
[[noreturn]] void terminate() noexcept;
```
功能是调用当前的terminate handler，可以用`set_terminate()`来指定。
```
typedef void (*terminate_handler)();
terminate_handler set_terminate(terminate_handler f) noexcept;
```
对于没有被catch的异常，会默认调用`terminate()`，而`terminate()`会默认调用`abort()`。当然这三者并非等价，因为前两者还会打印出各自的信息，比如我们给出这三种调用的输出。

```
throw std::exception();  // 1
std::terminate();        // 2
abort();                 // 3
```
分别运行这三条语句，均会使进程异常终止(比如在我的系统上`echo $?`得到结果是134)
语句1的输出是
```
terminate called after throwing an instance of 'std::exception'
  what():  std::exception
Aborted
```
语句2的输出是
```
terminate called without an active exception
Aborted
```
语句3的输出是
```
Aborted
```
从上面3种输出的区分可以看出，异常默认处理器、`std::terminate()`都会打印出自己的信息，然后再递归调用下一步。
`std::terminate()`的主要用途是在禁止使用异常的C++项目中进行调用。
再回过头看第1节的运行结果，不难得出结论，调用`t.join()`会抛出异常，而不调用则仅仅是触发`std::terminate()`，检查`t.joinable()`的返回值，是true，但是`t.join()`却抛出了异常，原因当然是`pthread_join()`改变了`std::thread`的底层线程句柄，但是并没有将`std::thread`对象内部的句柄置为初始状态，导致`std::thread`误认为该线程还没有被`join()`。

# 3. 一窥`std::thread`实现
很遗憾，`<thread.h>`头文件中没有包含一些关键的函数的实现，所以得自己去下载`libstdc++`源码，该源码是`gcc`的一部分。
```
git clone https://github.com/gcc-mirror/gcc.git
```
就我当前的版本(commit 0b47d0)，`std::thread`的实现在这两个文件中
```
libstdc++-v3/include/std/thread
libstdc++-v3/src/c++11/thread.cc
```
查看`thread.cc`中`join()`方法的实现
```
  void
  thread::join()
  {
    int __e = EINVAL;

    if (_M_id != id())
      __e = __gthread_join(_M_id._M_thread, 0);

    if (__e)
      __throw_system_error(__e);

    _M_id = id();
  }
```
以及`thread`中`joinable()`方法的实现
```
    bool
    joinable() const noexcept
    { return !(_M_id == id()); }
```
其中`_M_id`定义如下(`// ...`是我自己添加的注释，代表省略了其他代码)
```
  /// thread
  class thread
  {
  public:
    typedef __gthread_t			native_handle_type;

    ~thread()
    {
      if (joinable())
	std::terminate();
    }

    /// thread::id
    class id
    {
      native_handle_type	_M_thread;
      // ...
    }; 
    // ...
  private:
    id				_M_id;
  }; 
```
`std::thread`本身只有1个成员变量`_M_id`，类型是`std::thread::id`，而`id`内部也只有1个成员变量`_M_thread`，类型是`std::thread::native_handle_type`，而这个类型标准没有给出规定，由不同操作系统和编译器来决定，显然gcc的实现是`__gthread_t`，而且在`join()`方法中是将该类型的变量传入了`__gthread_join`中。
简化上面的代码，也就是说实际上`std::thread`的`join`是类似这样的
```
__gthread_t t;

void myjoin() {
  int e = EINVAL;
  if (t != __gthread_t())
    e = __gthread_join(t, 0);
  if (e)
    throw std::system_error(e);
  t = __gthread_t();
}
```
把这里的`__g`前缀替换成`p`的话，相当于就是调用`pthread_join()`后，重置线程句柄的值为`pthread_t`的默认值，然后用句柄是否等于`pthread_t`的默认值来判断是否以及调用了`join()`。
而对于同一个句柄，重复调用`pthread_join()`会导致返回非0的错误码`e`，并抛出`std::system_error`异常。
因此，像我在第1节中做的，手动取得`native_handle`，然后手动调用`pthread_join()`，就会导致`std::thread`内部的线程句柄没有重置，进而`joinable()`返回了不应该返回的true，进而在析构函数中调用了`join()`。注意，C++析构函数是禁止抛出异常的，在C++11中析构函数默认为`noexcept(true)`，也就是说直接调用`std::terminate()`终止程序。
这也解释了第1节中两种不同输出的原因，简单总结如下
1. 不调用`t.join()`: `t`内部的线程句柄没有重置，析构函数判断`joinable()`为true，从而调用`t.join()`，抛出异常不被捕获，直接调用`std::terminate()`终止进程；
2. 调用`t.join()`: `t`内部的线程句柄没有重置，调用`t.join()`，抛出异常，执行默认异常处理流程。

PS: 为何析构函数禁止抛出异常，以及`noexcept`的概念，分别参考*Effective C++*和*Effective Modern C++*的相关条款。
哦对了，刚才分析的前提是，`__g`前缀可以被替换成`p`，为何可以呢？原因是gcc实际上也是跨平台的，因此会定义一组别名来屏蔽系统API的差异。由于Linux是遵守POSIX接口的，所以这组定义在gcc项目的`libgcc/gthr-posix.h`中，在该头文件中可以看到下列定义
```
typedef pthread_t __gthread_t;
/* Typically, __gthrw_foo is a weak reference to symbol foo.  */
#define __gthrw(name) __gthrw2(__gthrw_ ## name,name,name)
__gthrw(pthread_join)
```
类型别名可以直接typedef，函数名称就得改变符号表了，这里我就不给出内部的`__attribute__`、`__weakref__`等具体实现手段了。

## 4. 不可移植的混合调用方法
有了上述的源码分析后，实际上在`pthread_join()`之后要做的，仅仅是将`std::thread`的内部线程句柄重置就完了。不过源码直接将其用`private`保护，所以得强行做指针转换来修改。
```
  *reinterpret_cast<std::thread::native_handle_type*>(&t) = pthread_t();
```
将这一句代码添加到`pthread_join()`之后即可。不过本身不可移植指的是这种做法是无法在不同编译器之间移植的，如果指定了编译器使用gcc版本，像这样hack下也未尝不可。

## 5. `pthread_exit`和`pthread_cancel`会抛出异常
对，你没看错，这两个C接口，会抛出异常。我之前在造线程轮子的时候，在某个调用`pthread_exit()`的包装函数声明为`noexcept`，结果被强制调用`std::terminate()`了。后来发现，原因是`pthread_exit`抛出了异常，而`noexcept`函数中抛出异常不会被捕获，只是简单调用`std::terminate()`了事。
嗯？问题来了，那这个异常又是怎么处理的呢？
异常的实现我没有仔细研究过，但大体上和`setjmp`、`longjmp`类似，让函数调用栈回绕(unwind)。只不过异常支持多层回绕，因此可以层层传播异常，把底层异常一步步抛到最上层一并处理，从而将错误处理和功能实现相分离。
C++和C虽然都是用同一套头文件，但是编译C++时链接到的动态库是libstdc++.so而非libc.so，其中用抛出`abi::__forced_unwind`异常的方式来简化这个回绕的实现，该异常的默认处理方式就是进行线程栈回绕。包含`<cxxabi.h>`头文件即可捕获该异常，当然，注意catch处理之后要重新throw抛出异常。
这个问题在stackoverflow上有所讨论，参考[why does pthread_exit throw something caught by ellipsis](https://stackoverflow.com/questions/11452546/why-does-pthread-exit-throw-something-caught-by-ellipsis) 
