---
title: Python lambda 实现回调函数
date: 2022-04-08 23:14:09
tags:
  - Python
  - C++
top: 1
---

[toc]

## 背景

本文偏实用性，从一个熟悉 C++ 但是对 Python 不太熟悉的用户角度，讲讲我的历程。不讨论语言上的细节。最近遇到一个 Python 中传递回调函数的场景，由于 Python 写得不多，导致 Python 的行为并不像我预期的那样。首先以一个错误示例开始：

```python
l = []
for i in range(3):
    def inner_f():
        print(i)
    l.append(inner_f)
for f in l:
    f()
```

看起来这里的 `i` 捕获了循环变量 `i`，将闭包先存入列表 `l` 中然后延迟调用。然而输出结果违反了直觉：

```
2
2
2
```

## 按引用捕获，而非按值捕获

我的第一反应是这里的 `i` 并不是按值捕获，而是按引用捕获。我修改代码如下：

```python
l = []
for i in range(3):
    s = "prefix-" + str(i)
    def inner_f():
        print(s)
    l.append(inner_f)
for f in l:
    f()
```

打印结果却是：

```
prefix-2
prefix-2
prefix-2
```

这就令人非常迷惑了，`s` 应该每次循环都创建的局部变量，难道 `s` 的创建也推迟到函数运行了？这里打印了 `i` 和 `s` 的地址：

```python
l = []
for i in range(3):
    s = "prefix-" + str(i)
    print('i: {} ({}), s: {} ({})'.format(i, id(i), s, id(s)))
    def inner_f():
        print('i: {} ({}), s: {} ({})'.format(i, id(i), s, id(s)))
    l.append(inner_f)
for f in l:
    f()
```

输出：

```
i: 0 (4536600848), s: prefix-0 (4539234672)
i: 1 (4536600880), s: prefix-1 (4539234800)
i: 2 (4536600912), s: prefix-2 (4539234736)
i: 2 (4536600912), s: prefix-2 (4539234736)
i: 2 (4536600912), s: prefix-2 (4539234736)
i: 2 (4536600912), s: prefix-2 (4539234736)
```

看起来确实……所有循环里的 `i` 和 `s` 都是复用了同一个变量。

## Python 局部变量生命周期

其实到这里，我突然想起最开始入门 Python 的时候，那本小册子上特地讲了，Python 与很多主流语言处理局部变量的不同之处。

比如：

```python
for i in range(3):
    s = "prefix-" + str(i)
print("{}, {}".format(i, s))
```

循环变量 `i` 和在循环体内定义的变量 `s`，在脱离循环后，仍然可以访问。可见 Python 在循环中使用的所有局部变量。Python 的局部变量不会随着循环脱离作用域，而是会随着函数退出才脱离作用域。用 C++ 作为比方，可能上述代码被解释成类似下面这样的 C++ 代码：

```c++
int i;
std::string s;
for (i = 0; i < 3; i++) {
  s = "prefix-" + std::to_string(i);
}
std::cout << i << ", " << s << std::endl;
```

## 闭包按值捕获

这个时候也许需要借助类似[偏函数](https://en.wikipedia.org/wiki/Partial_function)的概念。

```python
from functools import partial

l = []
for i in range(3):
    s = "prefix-" + str(i)
    def inner_f(i, s):
        print('i: {} ({}), s: {} ({})'.format(i, id(i), s, id(s)))
    # 使用 partial 将二元谓词 inner_f 的两个参数绑定为 i 和 s 的值
    l.append(partial(inner_f, i, s))

for f in l:
    f()
```

输出：

```
i: 0 (4373719312), s: prefix-0 (4376353072)
i: 1 (4373719344), s: prefix-1 (4376353200)
i: 2 (4373719376), s: prefix-2 (4376353264)
```

## 使用 lambda 进行简化

其实 Python 也提供了方便的 lambda 表达式，而不用使用 `functools` 下面的工具。

```python
l = []
for i in range(3):
    s = "prefix-" + str(i)
    l.append(
        lambda i=i, s=s: print("i: {} ({}), s: {} ({})".format(i, id(i), s, id(s)))
    )
for f in l:
    f()
```

这里 `i=i` 和 `s=s` 看起来比较诡异，实际上是进行初始化，用外部变量（实参）来初始化 lambda 的内部变量（形参）。对于不熟悉这种语法的人而言，上面代码改成这样可能更好理解（当然，也更冗长了，实际上就用上面这种代码更好）：

```python
l = []
for i in range(3):
    s = "prefix-" + str(i)
    l.append(
        lambda inner_i=i, inner_s=s: print(
            "i: {} ({}), s: {} ({})".format(inner_i, id(inner_i), inner_s, id(inner_s))
        )
    )
for f in l:
    f()
```

总之，输出也是符合期望的：

```
i: 0 (4427942160), s: prefix-0 (4430575856)
i: 1 (4427942192), s: prefix-1 (4430576048)
i: 2 (4427942224), s: prefix-2 (4430576112)
```

## 总结

总之磕磕绊绊，发现 Python 在处理稍微复杂一点点的逻辑时，并不是像初入门那么简单，易如 Python 也有所谓的坑。本文很多细节都没深究，只是以一个 C++er 的角度踩踩坑。

## 附录：C++ 风格的等价代码

```c++
#include <functional>
#include <iostream>
#include <string>
#include <vector>
using namespace std;

int main(int argc, char* argv[]) {
  vector<function<void()>> l;
  for (int i = 0; i < 3; i++) {
    std::string s{"prefix-" + std::to_string(i)};
    l.emplace_back([ i, s{std::move(s)} ] {
      std::cout << "i: " << i << ", s: " << s << std::endl;
    });
  }
  for (auto&& f : l) {
    f();
  }
  return 0;
}
```

上述 lambda 表达式中：

- `i`：直接按值来捕获循环变量 `i`，发生一次 `int` 的拷贝。
- `s`：对于 `std::string` 类型，使用移动构造来避免拷贝的开销。当然，在这个例子里没必要，因为对于短字符串的移动构造实际上还是发生了拷贝，一般都是 SSO（短字符串优化）实现。

这里用到了 C++14 的特性，也就是支持 lambda 表达式的捕获列表初始化（`s{std::move(s)}`），因此编译器至少需要支持 C++14 然后加上 `-std=c++14` 这种选项来编译。
