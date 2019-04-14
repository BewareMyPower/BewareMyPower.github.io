---
title: make_shared调用私有构造函数的解决方法
date: 2019-04-14 01:08:07
tags:
  - C++
top: 1
---

## 1. `enable_from_shared`
在*Effective Modern C++*的条款19中提到了`shared_ptr`的问题，对于某个共享所有权的类(记为`Widget`)的示例，可能会将引用对象分享给其他类，比如观测者模式中，将当前对象添加到观测列表中，给出一个简单的示例:
```
vector<shared_ptr<Widget>> observed_widgets;

void Widget::addToObserver() {
  observed_widgets.emplace_back(this);
}
```
`Widget`类的成员函数`addToObserver`将自身指针添加到全局的`Widget`列表`observed_widgets`中。
上述代码的问题在于，`this`的类型是裸指针`Widget*`，在添加进`vector`中时，会用裸指针构造`shared_ptr`对象，创建一个新的引用计数。如果调用`addToObserver`的对象本身也是`shared_ptr`，会导致对同一对象有2个引用计数，因此该对象会被析构2次，产生未定义的行为。
解决方式是继承`enable_shared_from_this<Widget>`类(C++的CRTP模式)，像这样:
```
struct Widget;  // 前置声明
vector<shared_ptr<Widget>> observed_widgets;

class Widget : public enable_shared_from_this<Widget> {
 public:
  void addToObserver() {
    observed_widgets.emplace_back(shared_from_this());
  }
};
```
`Widget`类从基类继承了`shared_from_this`方法，该方法会查询当前对象是否已经被`shared_ptr`引用了，如果有，则返回该`shared_ptr`对象而非裸指针`this`。
需要注意的是如果没有，会出现未定义行为，比如在我的系统上是抛出`std::back_weak_ptr`异常。因此良好的设计需要把`Widget`的构造函数私有化，并使用工厂方法来创建`shared_ptr<Widget>`对象，如下所示:
```
class Widget : public enable_shared_from_this<Widget> {
 public:
  // C++14语法，自动推导返回类型，参考Effective Modern C++条款2和3
  static auto create() { return shared_ptr<Widget>(new Widget); }
 private:
  Widget() = default;
};
```

## 2. `make_shared`代替`new`与私有构造函数的冲突
*Effective Modern C++*的条款21阐述了`make_shared`的优点，这里不详述，简单总结就是:
1. 更高的异常安全级别，防止构造`shared_ptr`之前就调用`new`，抛出异常;
2. 仅分配1次内存来保存引用对象和控制块(两者内存分布是连续的);
缺点则是无法指定自定义析构器，再就是大括号初始化的歧义，但在这两种情况之外，`make_shared`存在尺寸和速度上的优势，原则上是能使用`make_shared`则使用之。
*Effective Modern C++*给出了例外场景:
1. 自定义内存管理的类: 需要传递自定义析构器，`make_shared`无法使用;
2. 内存紧张的系统或非常大的对象: 由于内存碎片的问题，分配1次内存比分配2次内存更容易失败。

OK，回到这里，修改工厂方法如下所示(这里和后文均忽略基类和其他成员):
```
class Widget {
public:
  static auto create() { return make_shared<Widget>(); }
private:
  Widget() = default;
}
```
问题来了，编译失败，错误提示如下:
```
error: ‘constexpr Widget::Widget()’ is private
   Widget() = default;
   ^
/usr/include/c++/5/ext/new_allocator.h:120:4: error: within this context
  { ::new((void *)__p) _Up(std::forward<_Args>(__args)...); }
```
原因是`make_shared`函数模板并非`Widget`类的友元函数，其访问了私有构造函数。而静态成员函数可以访问类的私有成员(比如这里的私有构造函数)，因此可以在`create`内部调用`new`(两步：分配内存、调用构造函数)。
不知道是不是C++在制定`make_shared`的标准时疏忽的一点，但是在保持可移植性的情况下，最简单的方法就是用`new`替代`make_shared`，而且仔细来看，`make_shared`的性能优势可能并没那么重要，至于异常安全，大多数时候程序处理`new`抛出的异常就是任其终止。
说句题外话，这也是C++er经常背负的心智负担，由于C++自身的一些缺陷，导致coder纠结要不要以一种看起来丑陋的方式来换取可能并没什么用的"优化"。
但既然本篇讨论这个问题了，那就给出一些解决方式。

## 3. 解决方式
stackoverflow上有相关讨论，本文对其进行一些整合，给出2种典型的方法。
参考链接:
https://stackoverflow.com/questions/8147027/how-do-i-call-stdmake-shared-on-a-class-with-only-protected-or-private-const
### 3.1 使用公有构造的派生类
```
class Widget {
 public:
  template <typename... Args>
  static auto create(Args&&... args) {
    struct EnableMakeShared : public Widget {
      EnableMakeShared(Args&&... args) : Widget(std::forward<Args>(args)...) {}
    };
    return static_pointer_cast<Widget>(
        make_shared<EnableMakeShared>(std::forward<Args>(args)...));
  }

 private:
  Widget() = default;
  // other constructors...
};
```
该方法通过原封不动地继承`Widget`类，并利用可变参数模板覆盖`Widget`类的构造函数。然后通过派生类的`shared_ptr`向上转型为基类的`shared_ptr`。
这个方法的巧妙之处在于局部类的使用，如果是在类的外部定义其派生类，则必须将基类构造函数权限改成`protected`才能访问。
局部类在*C++ Primer*第5版19.7提及过，局部类只能访问外层作用域定义的类型名、静态变量以及枚举成员，而不能访问普通局部变量。但是类成员函数的局部类对类的私有成员的访问权限在书中并未提及我，网上搜到的一些资料也没找到，只有查看接近1400多页的C++标准文档，在**9.8 Local class declaration**中找到了如下定义:
> The local class is in the scope of the enclosing scope, and has the same access to names outside the function as does the enclosing function.

局部类和所在的函数对函数外的名称具有相同的访问权限，因此类成员函数的局部类可以访问该类的所有成员，包括私有构造函数。
最后一个问题是使用`Widget`的派生类构造派生类，是否需要将`Widget`的析构函数声明为虚函数？这里是不需要的，因为派生类并未额外申请任何资源，因此不执行派生类的析构函数也是没有问题的。

## 3.2 使用PassKey手法(idiom)
```
class Widget {
 private:
  struct PassKey {
    explicit PassKey() {}
  };

 public:
  template <typename... Args>
  explicit Widget(PassKey, Args&&... args)
      : Widget(std::forward<Args>(args)...) {}

  template <typename... Args>
  static auto create(Args&&... args) {
    return make_shared<Widget>(PassKey{}, std::forward<Args>(args)...);
  }

 private:
  Widget() = default;
  // other constructors...
};
```
PassKey手法参考https://arne-mertz.de/2016/10/passkey-idiom/
其原理是通过给公有构造函数增加一个无意义的嵌套类(在这里即`PassKey`类)，将其定义在类的私有域，在构造函数中使用嵌套类作为第1个参数，这样就无法从类的外部使用嵌套类的名字，因此无法调用这些构造函数。而类的静态成员函数可以访问嵌套类的名字，因此可以调用构造函数。
但这里有个细节，`PassKey`类的构造函数是`explicit`的，否则便可以在类外部传入`{}`作为构造函数的第1个参数，隐式转换成`PassKey`类。
另外还有个细节，`PassKey`类的构造函数不可以写成`explicit PassKey() = default`，否则在间接调用带实际参数的私有构造函数时会出错。比如，假设`Widget`类有如下私有构造函数：
```
Widget() {}
Widget(const char*) {}
```
那么有
```
Widget w1;  // 错误! Widget()是私有的!
Widget w2({}, "hello");  // 通过编译
```
但如果把`PassKey`类的构造函数像之前那样显式定义，而不是用`=default`使用编译器自动合成的构造函数，上述代码中`w2`的构造就无法通过编译。
至于原因暂时还没弄懂。

## 4. 总结
前文提到的2种方法都是用了一些比较冷门的语法，但是用起来还是比较麻烦且不直观。优雅的解决方案可能需要等待新的C++标准，比如[Andrew Schepler的C++标准提案](https://groups.google.com/a/isocpp.org/forum/#!topic/std-proposals/YXyGFUq7Wa4)。
虽然等到新标准估计要到猴年马月了，总之C++11的一些新东西虽然好用，但是也留了不少坑，这点还是挺让人不愉快的，研究这些东西其实实质上也没还什么意义，自娱自乐罢了。
