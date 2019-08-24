---
title: C++11利用逗号表达式实现constexpr函数参数检查
date: 2019-08-25 02:12:49
tags:
  - C++
top: 1
---
最近在抽空造个C++11环境下的通用轮子，以便每次写一些基础设施时都要重写一遍，比如错误处理库，每次重写时可能都不一样。然后发现C++17的`string_view`挺好用，而且不做成类模板处理模板参数的各个traits的话，实现起来还比较简单。

对着[cppreference](https://zh.cppreference.com/w/cpp/string/basic_string_view)实现来了一通后，发现有不少函数都不是`constexpr`返回值。

然后在一个个地加上`constexpr`修饰符时，出现了问题：

> error: body of constexpr function ‘constexpr const char& cpputil::cxx17::string_view::at(size_t) const’ not a return-statement

对应的代码为：

``` 
  constexpr const char& at(size_t pos) const {
    if (pos >= size())
      throwOutOfRangeError("at", pos);  // 自定义的方法，抛出和方法名以及pos相关的out_of_range异常
    return (*this)[pos];
  }
```

产生上述错误的原因是C++11中`constexpr`函数只能包含一个执行语句，`constexpr`修饰的函数意味着如果输入参数为编译期常量的话，返回值也是编译期常量，而C++11对编译期计算的支持还不够成熟，比如不能自动把运行期的`if`、`for`语句给转换成编译期计算，因此只允许一条语句，至于编译期的`if`、`for`就得用户自己写。

此时可以参考**Effective Modern C++**上的示例，用三目运算符来代替`if-else`，用递归来代替`for`。但是这里比较麻烦的是，抛异常的语句是没有返回值的，比如，我们不能写出这样的代码：

```
constexpr const char& at(size_t pos) const {
  return (pos < size()) ? (*this)[pos] : throwOutOfRangeError("at", pos);  // 错误代码！
}
```

此时，我想到了一个利用逗号运算符的技巧，也算是一种充分利用C++庞大语法的丑陋方法：

```
  constexpr const char& at(size_t pos) const {
    return (pos < size()) ? (*this)[pos]
                          : (throwOutOfRange("at", pos), (*this)[0]);
  }
```

嗯，这里用`(throwOutOfRange("at", pos), (*this)[0])`这个逗号表达式，这是一个很少有人会注意到的C++语法，比如`int i = (1, 2, 3)`，`i`的值也就是逗号表达式`(1, 2, 3)`中的最后一个数，这里前面的1和2都是不会产生任何副作用(side effect)的字面值，所以和`int i = 3`没区别。

然而逗号表达式前面的部分若是会产生副作用的表达式，则会依次执行，比如下面这段代码：

```
#include <iostream>
using namespace std;

void f() { cout << "f()" << endl; }
void g() { cout << "g()" << endl; }

int main() {
  int i = (f(), g(), 3);
  cout << i << endl;
  return 0;
}
```

运行结果为：

```
f()
g()
3
```

所以回到之前的代码，为了返回类型一致，我在逗号表达式最后边随便写了个`const char&`类型的返回值`(*this)[0]`，然后逗号表达式左边调用函数抛出异常，也就执行不到返回部分了。

嗯，之所以想到逗号表达式这东西，主要就是用Eigen库时被它逗号初始化矩阵的方式给惊了，可以参考[Eigen comma initializer](https://eigen.tuxfamily.org/dox/group__TutorialAdvancedInitialization.html#title0)以及[如何实现像Eigen一样的逗号初始化](https://stackoverflow.com/questions/29523119/how-could-comma-separated-initialization-such-as-in-eigen-be-possibly-implemente)。

不过这东西毕竟还是冷门语法，能不用就不用，不仅影响阅读，而且写复杂了还会出现运算符优先级的问题。

而C++14的`constexpr`函数就支持多行语句，就没这么多P事了，毕竟是C++11的小补丁。(说实话我已经对C++每个版本都要挖坑让后一个版本填坑心累了)
