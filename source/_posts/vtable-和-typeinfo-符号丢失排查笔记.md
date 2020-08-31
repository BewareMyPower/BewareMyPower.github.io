---
title: vtable 和 typeinfo 符号丢失排查笔记
date: 2020-08-31 19:56:06
tags:
  - C++
top: 1
---

[toc]

## 前言

简单说原来的结构是有一个类 `Foo`，并且重载了流运算符 `operator<<`，在另一个类 `Caller` 中直接 `<<` 打印出 `Foo` 对象。

现在由于功能扩充，`Foo` 可能有多种变体，因此需要进行以下重构：

```
FooBase
  -> Foo
  -> AnotherFoo
  -> ...
```

然后 `Caller` 内部存放的对象从 `std::unique_ptr<Foo>` 改成了 `std::unique_ptr<FooBase>`。

## 基本知识回顾：流运算符重载

对于类 `Foo`，流运算符重载是一个全局函数（注意，不是类成员函数）：

```c++
std::ostream& operator<<(std::ostream& os, const Foo& foo) {
    // os << foo 的内部字段
    return os;
}
```

为了能直接访问 `Foo` 的内部字段，一般会将该重载函数声明为类 `Foo` 的友元函数：

```c++
class Foo {
    friend std::ostream& operator<<(std::ostream&, const Foo&);
    // ...
};
```

这里要避免一个误区，那就是将 `operator<<` 作为类成员函数的话，比如：

```c++
class Foo {
 public:
    ostream& operator<<(std::ostream& os) const {
        // os << 内部字段
        return os;
    }
};
```

对应的调用是这样：

```c++
Foo foo;
foo.operator<<(std::cout);  // 完整调用方式
foo << std::cout;           // 简略版调用方式
```

而 `std::cout << os` 则是由全局函数来重载的，也就是是标准库的 `std::ostream` 类的 `operator<<` 方法调用的是 `std::ostream& operator<<(std::ostream&, const T&)` 函数来实现对任意类型 `T` 的对象进行输出。

## 继承体系的解决方式以及 vtable 信息缺失问题

对于继承体系，我们想要的其实是下面这样：

```c++
struct Base { /* ... */ };
struct Derived { /* ... */ };

auto base = new Derived;
std::cout << *base << std::endl;  // 调用 Derived 相关的 operator<<，且只暴露 Base 接口
```

可以用间接的方式来实现：

```c++
class Base {
    friend std::ostream& operator<<(std::ostream& os, const Base& base) {
        return base.print(os);
    }
 public:
    virtual std::ostream& print(std::ostream& os) const {
        // os << 内部字段
        return os;
    }
};
```

派生类只要重载 `print` 方法即可。

然而我这么干了，编译 OK，但是链接时出错：

> undefined reference to `vtable for 【基类名】
>
> undefined reference to `typeinfo for 【基类名】

## 问题排查

一开始只有这个信息，所以不好排除，我对比了半天（`override` 和 `virtual` 关键字数量对比），基类的虚函数我在派生类全都实现了啊。

----

PS：用 `override` 关键字可以很大程度避免重载函数名写错的情况，比如：

```c++
struct Base {
    virtual void doSomething() {}
};
struct Derived : Base {
    void doSOmeting() override { /* ... */ }
};
```

此时不小心写错了虚函数名字，用 `override` 关键字就直接能在编译时提示错误：

> error: ‘void Derived::doSOmeting()’ marked ‘override’, but does not override

如果不加 `override` 关键字，编译就不会出错，但是调用的是基类 `Base` 的 `doSomething()` 方法，派生类并没有重写该方法。导致这种低级错误得等到运行期去排查。

----

回到问题，这里我就有点束手无策了，总感觉自己有些基本知识弄错了（实际上并没有）。首先想了下是不是我虚析构函数的问题（因为除了虚析构函数外其他虚函数都是纯虚函数）：

```c++
class FooBase {
 public:
    virtual ~FooBase() {}
};
```

然后改成了头文件声明，源文件定义：

```c++
// foo_base.h
class FooBase {
 public:
    virtual ~FooBase();
};
```

```c++
// foo_base.cc
FooBase::~FooBase() {}
```

一个有趣的现象，虽然还是报错，但是报错信息变了，有了更具体的信息：

> undefined reference to `FooBase::print(std::ostream&) const'

到这里我才回过头来审视 `print` 方法。不过为了验证观点，首先把这个虚函数给删掉，链接成功，证明了这个观点。再回过头来看，我并没有将其实现为纯虚函数，而是：

```c++
virtual void print(std::ostream&) const;
```

只是声明，因此在基类的编译单元缺少其实现信息。看起来很简单的错误，但是在压力和恐慌之下，人的眼睛是不可相信的，至少我的肉眼看到的，这个分号前面就有个 `= 0`。

## 追根溯源

能定位到这个错误，有一定程度上是因为我恰好把虚函数放到源文件中，有了新的报错信息。那么区别在哪呢？这里复现一下。

### 复现

给出以下源文件：

```c++
// base.h
#pragma once

#include <iostream>

struct Base {
  virtual ~Base() {}

  virtual std::ostream& print(std::ostream& os) const;

  friend std::ostream& operator<<(std::ostream& os, const Base& base) {
    return base.print(os);
  }
};
```

```c++
// derived.h
#pragma once

#include "base.h"

struct Derived : Base {
  std::ostream& print(std::ostream& os) const override;
};
```

```c++
// derived.cc
#include "derived.h"

std::ostream& Derived::print(std::ostream& os) const {
  return (os << "Derived");
}
```

编译成动态库 `libbase.so`：

```bash
$ g++ -o libbase.so derived.cc -std=c++11 -fPIC -shared
```

因为是动态库，开启了 `-fPIC` 选项，即  **p**osition-**i**ndependent **c**ode，位置无关的代码，也就是函数（符号）的实现暂时可以不定位到具体的位置，而是在和其他编译单元链接时再定位。

这里给出调用代码：

```c++
// main.cc
#include <memory>
#include "derived.h"

int main(int argc, char* argv[]) {
  std::unique_ptr<Base> base(new Derived);
  std::cout << *base << std::endl;
  return 0;
}
```

编译：

```bash
$ g++ main.cc -std=c++11 -L. -lfoo
/tmp/cc39RCrN.o: In function `Base::Base()':
main.cc:(.text._ZN4BaseC2Ev[_ZN4BaseC5Ev]+0x9): undefined reference to `vtable for Base'
/tmp/cc39RCrN.o: In function `Derived::Derived()':
main.cc:(.text._ZN7DerivedC2Ev[_ZN7DerivedC5Ev]+0x19): undefined reference to `vtable for Derived'
collect2: error: ld returned 1 exit status
```

### 查看符号表

使用 `nm` 查看符号表：

```bash
$ nm libbase.so | egrep "(Base|Derived)"
0000000000000cda W _ZN4BaseD0Ev
0000000000000ca4 W _ZN4BaseD1Ev
0000000000000ca4 W _ZN4BaseD2Ev
0000000000000d42 W _ZN7DerivedD0Ev
0000000000000d00 W _ZN7DerivedD1Ev
0000000000000d00 W _ZN7DerivedD2Ev
0000000000000c20 T _ZNK7Derived5printERSo
                 U _ZTI4Base
0000000000201058 V _ZTI7Derived
0000000000000dc8 V _ZTS7Derived
                 U _ZTV4Base
0000000000201030 V _ZTV7Derived
```

由于 [name mangling](https://en.wikipedia.org/wiki/Name_mangling)，上面的比较难辨认。注意第二列的符号，从 [`nm` 帮助手册](https://linux.die.net/man/1/nm) 可知：

- `W`：没有标记成**弱对象**的弱（**W**eak）符号，当弱符号链接到普通符号时不会报错，当弱符号被链接且该符号未定义时，该符号的值用一种系统特定的方式决定，不会报错。某些系统上，大写的 `W` 代表默认值被指定。
- `T`：符号在文本（代码）段。
- `U`：符号未定义（**U**ndefined ）。
- `V`：**弱对象**，其余说明同 `W`。

PS：即使使用 `delete` 禁止拷贝构造函数和拷贝赋值运算符，符号表仍然不变。

如果将 `Base::print` 改成纯虚函数呢？

```c++
  virtual std::ostream& print(std::ostream& os) const = 0;
```

符号表变成了：

```
0000000000000dda W _ZN4BaseD0Ev
0000000000000da4 W _ZN4BaseD1Ev
0000000000000da4 W _ZN4BaseD2Ev
0000000000000e42 W _ZN7DerivedD0Ev
0000000000000e00 W _ZN7DerivedD1Ev
0000000000000e00 W _ZN7DerivedD2Ev
0000000000000d20 T _ZNK7Derived5printERSo
00000000002010b8 V _ZTI4Base
00000000002010a0 V _ZTI7Derived
0000000000000ed1 V _ZTS4Base
0000000000000ec8 V _ZTS7Derived
0000000000201078 V _ZTV4Base
0000000000201050 V _ZTV7Derived
```

这里列出区别：

- `_ZTI4Base` 和 `_ZTV4Base` 从 `U`（未定义）变成了 `V`，也就是弱对象。
- 多了个弱对象 `_ZTS4Base`。

然后将析构函数单独抽离到 `base.cc` 来实现，重新编译动态库：

```bash
$ g++ -o libbase.so base.cc derived.cc -std=c++11 -fPIC -shared
```

符号表变成了：

```
0000000000000d66 T _ZN4BaseD0Ev
0000000000000d30 T _ZN4BaseD1Ev
0000000000000d30 T _ZN4BaseD2Ev
0000000000000eb2 W _ZN7DerivedD0Ev
0000000000000e70 W _ZN7DerivedD1Ev
0000000000000e70 W _ZN7DerivedD2Ev
0000000000000dec T _ZNK7Derived5printERSo
0000000000201138 V _ZTI4Base
0000000000201170 V _ZTI7Derived
0000000000000f29 V _ZTS4Base
0000000000000f38 V _ZTS7Derived
0000000000201110 V _ZTV4Base
0000000000201148 V _ZTV7Derived
```

主要区别：

- 三个 `_ZN4BaseD<i>Ev`（i 是0，1，2）从 `W`（弱符号）变成了 `T`（文本段）。

而重新将 `print` 改成未定义的函数后，符号表变成了：

```
0000000000000d96 T _ZN4BaseD0Ev
0000000000000d60 T _ZN4BaseD1Ev
0000000000000d60 T _ZN4BaseD2Ev
0000000000000ee2 W _ZN7DerivedD0Ev
0000000000000ea0 W _ZN7DerivedD1Ev
0000000000000ea0 W _ZN7DerivedD2Ev
                 U _ZNK4Base5printERSo
0000000000000e1c T _ZNK7Derived5printERSo
0000000000201168 V _ZTI4Base
00000000002011a0 V _ZTI7Derived
0000000000000f59 V _ZTS4Base
0000000000000f68 V _ZTS7Derived
0000000000201140 V _ZTV4Base
0000000000201178 V _ZTV7Derived
```

最大的区别，这里的 `U` 是 `_ZNK4Base5printERSo`，很显然，就是基类 `Base` 的 `print` 方法。虽然对链接的知识已经忘了不少（得去补课了），但回顾这 4 张符号表，还是可以大致看出为啥析构函数单独分离出去后信息发生变化。

最开始析构函数实现写在 .h 文件里时，未定义的符号有两个（一个是 `typeinfo` 一个是 `vtable`）：

- `_ZTI4Base`
- `_ZTV4Base`

而析构函数独立出去后，未定义的符号：

- `_ZNK4Base5printERSo`：`Base` 类的 `print` 函数。

至少现在我们知道了为啥会有提示错误区别，但是编译器为啥这么干，还是不清楚。只能说，从经验的角度

### 优化一下？

所以说明明可以写在源文件里，为何要写在头文件中？写在源文件里至少还能帮助调试。当然这是出自 C++er 的直觉：要是内联了呢？

于是回到最初的模式（`Base` 虚析构函数在头文件实现，`print` 不实现），开 `-O2` 编译，符号表：

```
0000000000000b30 W _ZN7DerivedD0Ev
0000000000000b20 W _ZN7DerivedD1Ev
0000000000000b20 W _ZN7DerivedD2Ev
0000000000000b00 T _ZNK7Derived5printERSo
                 U _ZTI4Base
0000000000200cb0 V _ZTI7Derived
0000000000000bc0 V _ZTS7Derived
0000000000200cc8 V _ZTV7Derived
```

相比默认的（`-O0` 编译）：

```
0000000000000cda W _ZN4BaseD0Ev
0000000000000ca4 W _ZN4BaseD1Ev
0000000000000ca4 W _ZN4BaseD2Ev
0000000000000d42 W _ZN7DerivedD0Ev
0000000000000d00 W _ZN7DerivedD1Ev
0000000000000d00 W _ZN7DerivedD2Ev
0000000000000c20 T _ZNK7Derived5printERSo
                 U _ZTI4Base
0000000000201058 V _ZTI7Derived
0000000000000dc8 V _ZTS7Derived
                 U _ZTV4Base
0000000000201030 V _ZTV7Derived
```

首先前三个 `Base` 的符号（构造函数）被直接内联了。`_ZTV4Base` 也没了（虚表？），编译 `main.cc` 报错信息也少了：

> $ g++ main.cc -std=c++11 -L. -lbase -O2
> ./libbase.so: undefined reference to `typeinfo for Base'
> collect2: error: ld returned 1 exit status

也就是说 `_ZTV4Base` 实际上就是 `vtable`（`V` 代表 **v**table），而另一个保留的 `_ZTI4Base` 则是类型信息（`I` 代表 type**i**nfo）。

看来我作为 C++er 的直觉还是对的，内联了导致虚析构函数就是个普通函数一样。但如果把虚析构函数给独立出去，那么开不开 `-O2` 优化，结果都一样。

### name mangling 还原

其实 `nm` 已经提供了还原功能了，加上 `-C` 选项即可（这是 `-O2` 优化+析构函数在头文件里+`print` 函数未定义）：

```bash
$ nm -C libbase.so | egrep "(Base|Derived)"
0000000000000b30 W Derived::~Derived()
0000000000000b20 W Derived::~Derived()
0000000000000b20 W Derived::~Derived()
0000000000000b00 T Derived::print(std::ostream&) const
                 U typeinfo for Base
0000000000200cb0 V typeinfo for Derived
0000000000000bc0 V typeinfo name for Derived
0000000000200cc8 V vtable for Derived
```

PS：嗯，前面的内容就当踩坑了……懒得改……

此外，也可以看到析构函数放在源文件里时符号表多了：

```
0000000000000d70 T Base::~Base()
0000000000000d60 T Base::~Base()
0000000000000d60 T Base::~Base()
```

对应的符号是以 `D` 结尾的，即 `D0`/`D1`/`D2`。至于为啥有三个析构函数我也不知道……对比了下普通类，只有两个析构函数（`D1`/`D2`）。

### 为何析构函数的符号有无会导致结果变化

从上述分析可知，析构函数如果放在头文件里，无论是否内联优化，最终符号表里都只是 vtable 缺失。我大致猜测是，仅有一个虚函数的符号没有定义。

于是修改 `Base` 类的实现，加一个有实现的虚函数 `f`：

```c++
// base.h
#pragma once

#include <iostream>

struct Base {
  Base() = default;
  Base(const Base&) = delete;
  Base& operator=(const Base&) = delete;
  virtual ~Base() {}

  virtual void f() const;

  virtual std::ostream& print(std::ostream& os) const;

  friend std::ostream& operator<<(std::ostream& os, const Base& base) {
    return base.print(os);
  }
};
```

```c++
// base.cc
#include "base.h"

void Base::f() const {}
```

结果：

```bash
$ g++ -o libbase.so base.cc derived.cc -std=c++11 -fPIC -shared -O2
$ nm -C libbase.so | egrep "(Base|Derived)"
0000000000000d70 W Base::~Base()
0000000000000d60 W Base::~Base()
0000000000000d60 W Base::~Base()
0000000000000de0 W Derived::~Derived()
0000000000000dd0 W Derived::~Derived()
0000000000000dd0 W Derived::~Derived()
0000000000000d50 T Base::f() const
                 U Base::print(std::ostream&) const
0000000000000db0 T Derived::print(std::ostream&) const
0000000000201038 V typeinfo for Base
0000000000201078 V typeinfo for Derived
0000000000000e68 V typeinfo name for Base
0000000000000e78 V typeinfo name for Derived
0000000000201048 V vtable for Base
0000000000201090 V vtable for Derived
```

可见，这里 `U` 不再是符号表，而是虚函数本身。

## 总结

C++ 的 unsolved symbol 问题其实挺常见的，即使是踩过 N 次坑的我也容易因为一点失误而犯错。本文主要讲述了通过 `nm` 排查问题的方式，其中如果只有 `vtable`/`typeinfo` 缺失这种难以排查的信息，可以尝试加一个带实现的虚函数（比如前文的 `f`），再来排查符号表。