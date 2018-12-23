---
title: lambda捕获this指针
date: 2018-12-23 23:37:12
tags:
  - C++
top: 1
---
最近看某qt项目，看到`connect`使用了lambda表达式，比如
```
void MainWindow::initOutput() {
    // ...
    connect(ui->tableViewOutput, &QWidget::customContextMenuRequested, this, [=](const QPoint &pos) {
        m_outputContextMenu.exec(ui->tableViewOutput->mapToGlobal(pos));
    });
    // ...
}
```
命名规范是`m_`开头的是成员变量，可以发现类成员函数中用lambda表达式中直接使用了成员变量。
于是我突然有了疑问:
1. 成员变量不是局部变量，那是怎么捕获的呢？
2. 如果含有不可拷贝的成员变量，那么`=`岂不是失效了？

为了解决这些疑问，一步步用代码验证，首先测试不可拷贝的成员变量
```
#include <iostream>
#include <memory>
using namespace std;

struct Foo {
  std::unique_ptr<int> p;

  void f() {
    auto f = [=] { cout << p.get() << endl; };
    f();
  }
};

int main() {
  Foo foo;
  foo.f();
}
```
上述代码运行正确，打印`0`。而`unique_ptr`是典型的不可拷贝的类，用`=`却捕获成功了。
怀着疑问，我尝试了下局部变量，把类的定义改成下面这样
```
struct Foo {
  void f() {
    std::unique_ptr<int> p;
    auto f = [=] { cout << p.get() << endl; };
    f();
  }
};
```
编译出错(嗯，其实在编译前，我的vim插件`ale`已经提示了错误)
```
use of deleted function 'std::unique_ptr<_Tp, _Dp>::unique_ptr(const std::unique_ptr<_Tp, _Dp>&) [with _Tp = int; _Dp = std::default_delete<int>]
```
那么，也就是说，类成员函数中的lambda表达式并不是像捕获局部变量一样"捕获"类成员变量，而是通过某些其他途径得以访问类成员变量。
参阅*Effective Modern C++*后面的部分(嗯，我从前往后抽空看的，目前还没看完)，恍然大悟。
在**条款31：避免默认捕获模式**中，书上举出了一个类似例子，并给出了说明
> 捕获只能针对于在创建lambda式的作用域内可见的非静态局部变量(包括形参)。

也就是说，在函数体外的类成员变量(这里的`p`)是无法被捕获的，那么为何可以直接访问`p`呢？
其实是捕获了`this`指针。
> 每一个非静态成员函数都持有一个`this`指针，然后每当提及该类的成员变量时都会用到这个指针。

那么这里也解释通了，其实`[=]`捕获了`this`指针，然后编译器看到`p.get()`时，把它翻译成了`this->p.get()`。把代码中的`[=]`改成`[this]`，仍然成功编译。
也就是说，如果`this`指向的对象已经被析构，捕获`this`指针的lambda式就可能出现问题，比如下面代码
```
#include <functional>
#include <iostream>
#include <memory>
using namespace std;

struct Foo {
  std::unique_ptr<int> p;
  std::function<void()> f() {
    p.reset(new int(1));
    return [=] { cout << *p << endl; };
  }
};

int main() {
  auto foo = new Foo();
  auto f = foo->f();
  delete foo;
  f();
}
```
运行结果为0而非1，而且这里输出0是未定义行为，因为访问的实际上是被回收的空间，只是因为编译器的`delete`并没有对回收的空间做额外的操作，所以`p`指向的仍然是原来那块，只不过那块已经被`unique_ptr`的析构函数自动清除了，只不过将清除的部分全部置为0而已。

由于`[=]`很容易让人忽略掉`this`也被捕获了，所以很容易让人忽视这个问题，所以不如`[this]`直观。
