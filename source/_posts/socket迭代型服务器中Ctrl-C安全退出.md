---
title: socket迭代型服务器中Ctrl+C安全退出
date: 2019-01-27 20:31:24
tags:
  - C++
  - 网络编程
top: 1
---
## 前言
最近抽空在重新造网络库轮子，之前的[socket_util](https://github.com/BewareMyPower/socket_util)太过简陋，就是把socket地址、socket API、epoll和`sendall`、`recvn`之类的函数给封装下，发现自己的基础知识还是欠缺，于是边重做轮子边补充知识。
于是在做最基本的迭代型服务器时遇到了问题，就是`Ctrl+C`退出循环失败，这里记录下解决方案。

## 1. 信号中断后自动重启的系统调用
最初的解决思路如下
```
::signal(SIGINT, [](int) {});
while (true) {
  // ...
  int ret = some_system_call();  // 代替某些自动重启的系统调用
  if (ret == -1) {
    if (errno == EINTR)
      break;
    // else ...
  }
}
```
思路就是对于某些被信号中断后会自动重启的系统调用，检查返回值，为-1代表返回错误，然后检查`errno`，为`EINTR`就相当于被信号中断，此时跳出循环。
Linux上某些系统调用会导致阻塞，比如某些慢速I/O操作(`read`、`write`、`recv`、`send`)，当然也包括socket的`accept`这种可以触发`epoll`监听事件的(在我看来算广义的I/O操作)。如果对某个信号使用`sigaction`设置过信号处理器，并且时`sa_flags`字段包含`SA_RESTART`，那么这些系统调用在被该信号中断时会自动重启，避免了对`errno`的检查(像下面这样的代码)
```
int ret;
while ((ret = some_system_call()) == -1 && errno == EINTR) continue;
```
其中，`sigaction`使用示例如下
```
struct sigaction act;
act.sa_handler = [](int) {};  // 信号处理器
act.sa_flags = SA_RESTART;    // 设置重启标志
sigemptyset(&act.sa_mask);

sigaction(SIGINT, &act, nullptr);  // 使信号处理器生效
```
当然，这种自动重启的功能仅对特定的系统调用有效，可以参考*The Linux Programming Interface*的21.5节，个人简单总结如下：
- socket的阻塞操作(`accept`、`connect`、`recv`、`send`等)；
- 进程间IPC的阻塞操作(管道、FIFO、POSIX消息队列、信号量、文件锁)；
- 线程同步的阻塞操作(条件变量的等待)
- 终端的阻塞操作(比如向`STDOUT_FILENO`写或从`STDIN_FILENO`读)；
- `wait`系列，等待子进程终止；
- 可能会阻塞的`open`；
- `futex`(Linux独有)；

而像`epoll`这种I/O多路复用的，即使指定`SA_RESTART`也不会自动重启。其实从设计的出发点看很好理解。

## 2. 细节上的修改
最初的解决方案在我实际应用中遇到了一些问题，比如这个循环是封装在我的库函数中的，而我需要传递一个回调函数给这个库函数来处理表示连接的socket用于读写。
这样问题来了，我在回调函数中如果是`while`循环来`recv`，那么在回调函数中是无法直接跳出所在函数的循环的，比如我的函数可能是这样
```
void runIterativeServer(std::function<void(int)> callback, ...) {
  // 1. 初始化socket相关操作...
  // 2. 为SIGINT设置信号处理器
  while (true) {
     // 3. accept得到connfd，处理errno为EINTR的情况
     callback(connfd);  // 4. 执行回调函数
     // 5. 关闭connfd
  }
  // 6. 清理操作
}
```
解决方法很简单，处理`EINTR`时`continue`即可，用信号处理器来退出`while`循环
```
static volatile bool run = false;
::signal(SIGINT, [](int) { run = false; });
run = true;
while (run) {
  // ...
  int ret = some_system_call();  // 代替某些自动重启的系统调用
  if (ret == -1) {
    if (errno == EINTR)
      break;
    // else ...
  }
}
```
注意细节，`static`和`volatile`。
使用`static`是因为信号处理器只能接受`void (*)(int)`这样的函数指针，不带捕获的`lambda`表达式可以与之兼容，但是带捕获的`lambda`表达式不行。本质上就是信号处理器只能处理全局变量。静态变量和全局变量本质是一样的，只是方便命名信息隔离，所以这里用函数内的静态变量。
使用`volatile`则是因为，开优化选项时编译器可能做某些优化来把`run`放到寄存器中读取而不是直接从内存读取，这样就会导致信号处理器的修改并不会被看到，从而`while`循环仍然判定为`true`，之前写过相关博客[C/C++的volatile关键字应用示例](https://www.jianshu.com/p/0388f6f21fe0)。

这样一来在回调函数中就可以写出这样的代码了
```
char buf[1024];
while (true) {
  ssize_t num_recv = recv(connfd, buf, sizeof(buf), MSG_NOSIGNAL);
  if (num_recv == -1 && errno == EINTR) break;
  // ...
}
```
到这里其实还有个很关键的问题，也是很容易被忽视的。其实无论是这个版本还是之前的版本，都是无效的，因为系统调用还是会自动重启，语句根本执行不到检查`errno`那一步。
原因是`Linux`的`signal`函数，默认是调用`sigaction`，并且将`sa_flags`设置为`SA_RESTART`，从而支持某些系统调用自动重启。在其他系统上，`signal`也可能具有不同的语义，因此必须手动调用`sigaction`来代替`signal`。
```
struct sigaction act;
act.sa_flags = 0;    // 禁止系统调用自动重启
// ...
```
参考*Unix环境高级编程*的10.14节。
最后，可以用`sigaction`取得之前的信号处理器，然后在函数结束时恢复`SIGINT`的默认处置。
```
struct sigaction act, oldact;
// TODO: 设置act的字段
sigaction(SIGINT, &act, &oldact);
// ...
sigaction(SIGINT, &oldact, nullptr);
```
