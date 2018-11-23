---
title: '实现thread库细节：使用std::decay保存函数指针'
date: 2018-11-21 16:45:21
tags:
  - C++
top: 1
---
## 0. 前言
最近想包装`pthread`，像C++11线程库一样，毕竟标准库对于很多东西都没有包装，比如线程属性的设置。虽然可以用`thread::native_handle_type()`取得底层的`pthread_t`句柄，当然，本身就没想去麻烦地跨平台。
主要目的还是出于学习，以及写个顺手的线程库。
参考了知乎上的一篇回答：[C++11是如何封装thread库的](https://www.zhihu.com/question/30553807)
花了不少精力终于理解了整套流程，期间通过《C++ Primer》第16章复习了完美转发、可变模板参数(解包、包扩展)的知识，通过qicosmos前辈的博客[泛化之美--C++11可变模板参数的妙用](https://www.cnblogs.com/qicosmos/p/4325949.html)补充了一些知识。
在做了一些简单的测试后，便开始建立一个仓库实现我的想法。
但是自己实现过程中也遇到一些细节问题，毕竟回答也不可能太详细，比如这篇提到的`decay`。

## 1. `decay`类型退化
C++ Primer中提到，在模板类型推导中，一般的类型转换是禁止的，否则无法准确推断是哪种类型。但是两种类型转换是允许的：
1. 将非`const`的引用或指针传递给`const`的引用或指针形参，比如
```
template <typename T> void f(const T&);  // 函数模板声明

int i = 0;
int& ri = i;
f(ri);  // ri被转换成const int&类型，因此T被推断为int
```
2. 若形参不是引用类型，数组实参会被转换成指针形参，函数实参会被转换成函数指针形参，比如
```
template <typename T> void f(T param);

void func();
f(func);  // T被推断为void(*)()

int a[10];
f(a);     // T被推断为int*
```

`decay`完成的功能即将数组和函数转换成退化后的类型，对于其他类型，则是移除引用以及`const`和`volatile`描述符，分别对应上述的2和1。
注意，在模板类型推断中，实参的引用类型都会被忽略，就像上述1中的`const`被忽略一样。
比如传入`int&`到形参`T t`中，推断方式是：先忽略`&`，然后匹配`int`和`T`，因此T永远不会被推断为引用类型，除非形参是右值引用`T&& t`，根据引用折叠规则(`&& &&`折叠为`&&`，`&& &`、`& &&`、`& &`被折叠为`&`)，才有可能推断`T`为引用，这也是C++11实现完美转发的基础。
从类型`T`到退化类型`typename std::decay<T>::type`的映射示例如下
```
int () => int (*)()
int (&)() => int (*)()
int [10] => int*
int (&) [10] => int*
int const => int
int const& => int
int const* => int const*
int* const => int*
```

## 2. 使用可变模板参数构造线程函数的简单原理
C++11线程的构造函数是
```
template <typename F, typename... Args>
explicit thread(F&&, Args&&...);
```
而底层线程接口是C风格的，一般需要将参数类型和`void*`进行转换，比如对于`pthread`线程函数，一般像这样使用
```
void* threadFunc(void* arg) {
  auto input = static_cast<input_type*>(arg);
  // do sth... and generate an exit_code
  return reinterpret_cast<void*>(exit_code);
}
```
当然，返回值也可以是动态分配的对象的指针，但这就需要显式`pthread_join`，处理完返回类型后将该指针指向的资源回收。
实现C++11风格的线程函数的思路是：用`std::tuple<Args...>`(记为`TupleType`)将整个打包，然后动态申请一个`TupleType*`指针传入C风格线程函数。
由于线程函数是编译时期就决定的，无法像lambda表达式一样捕获外部参数，所以需要用虚函数实现：
C风格线程函数中将`void*`转换成`Base*`，而继承自`Base`类的`Derived`则实现可变模板参数的构造函数，将函数和输入参数打包，然后override基类的虚函数`void invoke()`，调用打包的函数。
```
struct BaseData {
  virtual ~BaseData();
  virtual void invoke();
};

template <typename F, typename... Args>
struct DerivedData;  // 省略具体实现，内部包含一个tuple打包数据，实现了invoke函数调用函数

void threadFunc(void* args) {
  auto p = static_cast<BaseData*>(args);
  p->invoke();
}

// 线程函数的实现
template <typename F, typename... Args>
void createThread(F&& f, Args&&... args) {
  auto data = new DerivedData(std::forward<F>(f), std::forward<Args>(args)...);
  pthread_t tid;
  int error = pthread_create(&tid, nullptr, threadFunc, static_cast<void*>(data));
  // handle error
}
```
其中有一些细节，比如如何在编译期确定`get<i>()`的`i`的范围，需要一点模板元的技巧，在C++17中`index_sequence`和`invoke`均已实现，C++11中也可以照着libstdc++源码自行实现。
另外为了线程安全，可以使用`std::unique_ptr`代替`new`，构造对象时使用`unique_ptr`，然后调用`release()`方法传递内部指针，再在线程函数中构造`unique_ptr`。
不过本文重点说的就是怎么打包。

## 3. 打包退化类型的数据
看似打包数据应该使用完美转发，保留参数的`cv`修饰符以及引用类型，其实并不然，这点在本文链接的知乎回答中没有提及，毕竟是个细节，但是贴出的代码中都会发现，使用了`decay`。
### 3.1 `result_of`的使用
对于底层C风格接口，返回的`void*`并不一定是实际你想返回的数据，因为线程函数往往只是完成一个任务，对主线程报告一个状态码。
但有时候也需要取得线程的计算结果，比如对一堆数据，分块并行求和，需要取得每个线程计算的结果，然后主线程将其相加。
C++11线程设施中，`thread`本身无法取得线程函数的返回值，需要结合`future`来完成，当然，在底层需要知道函数的返回类型，这里可以用`<type_traits>`中的`result_of`来实现。
```
using F = int& (*)(int, int);  // 函数指针
using R = std::result_of(F(int, int))::type;  // int&
```
但是`result_of`有个陷阱，比如把上述代码在实际中可能是这个样子
```
int& f(int x, int y) {
  // ...
}

using F = decltype(f);
using R = std::result_of<F(int, int)>::type;
```
然后编译就会出错
```
error:  type name’ declared as function returning a function
```
原因是这样的，`F`是`int& ()(int, int)`，结合起来`F(int, int)`就是`int& ((int, int))(int, int)`，该类型实际上被看做是：
- 输入参数为2个`int`；
- 返回类型为`int& ()(int, int)`。

因此会提示你返回一个函数，而函数本身是无法作为返回值的。但是如果`F`是函数指针`int (*)(int, int)`，那么结合起来`F(int, int)`就是`int (*(int, int))(int, int)`，该类型实际上是：
- 输入参数为2个`int`；
- 返回类型为`int& (*)(int, int)`。

这一段非常绕，归根到底还是C++为了兼容C语言函数指针一层套一层的语法。总之，由于返回值只能是函数的指针或引用，而不能是函数，才会出现这种情况。
因此如果要将函数`f`和参数`args...`打包，如果完美转发`f`，那么可能保存的类型是函数而非函数指针、引用，导致调用`result_of`时编译出错。

## 3.2 能够被退化的类型本身无法进行拷贝
其实使用`decay`来打包的根本原因并不是`result_of`，毕竟，完全可以在`invoke()`方法的实现中再去把函数类型转换成函数指针。
初学C语言时，无论是书籍还是老师都会讲到，数组不能拷贝
```
int a[2], b[2];
a = b;  // 编译错误
```
正因为如此，传入形参时才将其退化成指向数组首地址的指针。而C语言中不存在这么复杂的类型推导以及引用类型，根本无需区分函数和函数指针，所以没有强调函数退化成函数指针这个例子。但是，函数和数组一样，都是无法拷贝的。
而引用类型，看似可以拷贝
```
int i, j;
int &ri = i, &rj = j;
ri = rj;  // 编译通过
```
实际拷贝的并不是引用本身，而是引用的对象之间进行拷贝。
那么，考虑数组进行打包，`tuple`的模板参数是无法设为不可拷贝的类型的，因此只能存为数组指针。
但是问题在于到底存放退化后的`int*`还是指向数组的指针`int (*)[N]`呢？参考如下代码
```
  int a[3] = {0, 1, 2};
  std::tuple<int*> t1(a);
  cout << get<0>(t1)[2] << endl;  // 2
  std::tuple<int(*)[3]> t2(&a);
  cout << get<0>(t2)[2] << endl;  // 某个地址，比如0x7ffc462e1a98
```
只有退化类型才能表达出和原来传入实参一样的结果，就像完美转发一样。而另一种所谓的类型退化，即`int(*)[N]`，实际上已经改变了实参的意义。
引用类型`int(&)[N]`能够和实参一样，但是问题是引用本身无法拷贝。
至于丢失的信息，比如数组丢失的长度信息，在函数调用中如果不是传入数组引用，本身就会丢失。既然参数转发前后都丢失了同样的信息，某种程度上也可以称为 *完美转发* 了。

## 3.3 打包数据的实现
类型退化可以看做是直接匹配，是很自然而然的操作，因此仅需把`tuple`模板参数声明为退化类型即可，传参时还是完美转发，实现如下
```
template <typename T>
using decay_t = typename std::decay<T>::type;

template <typename F, typename... Args>
struct DataTuple {
  std::tuple<decay_t<F>, decay_t<Args>...> t_;

  DataTuple(F&& f, Args&&... args)
      : t_(std::forward<F>(f), std::forward<Args>(args)...) {}
};

template <typename F, typename... Args>
DataTuple<F, Args...>* makeDataTuple(F&& f, Args&&... args) {
  return new DataTuple<F, Args...>(std::forward<F>(f),
                                   std::forward<Args>(args)...);
}
```
到这里，其实还有个关键问题，那就是，既然引用类型会丢失，那么传入引用类型的话，不就无法转发了吗？C++11线程库也考虑到了这点，因此使用了`ref`来对引用类型进行包装，这里给出一个机遇我们的`DataTuple`的示例
```
void f(int& x) { x++; }

int main() {
  int i = 0;
  int& ri = i;
  auto p = makeDataTuple(f, std::ref(ri));
  get<0>(p->t_)(get<1>(p->t_));
  cout << i << endl;  // 1
  // delete p and exit.
}
```
输出结果和注释一样，是1，说明引用成功传递了，其实内部实现是一个`reference_wrapper<T>`，其内部保存一个`T*`，从而使得`ref(ri)`具有值语义。
