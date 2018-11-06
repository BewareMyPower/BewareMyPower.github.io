---
title: gdb调试类模板成员函数
date: 2018-11-06 15:29:44
tags:
  - C++
top: 1
---
## gdb常用命令简单回顾
首先列出一些常用的gdb命令
`l`即`list`，从第1行开始查看当前文件，后面可接数字，比如`l 10`就是查看以第10行为中间行的若干行(比如第5-14行)。
`b`即`break`，后接数字n，则代表在第n行设置断点。另外后接`if <condition>`可以设条件断点。
`i`即`info`，后接`b`可以查看所有的断点信息(包括断点编号)。
`d`即`delete`，后接断点编号，可以删除特定断点。
`r`即`run`，运行程序。如果需要添加命令行参数，则后接命令行参数即可，也可以后接`< file`这种将文件file的内容重定向到标准输入。
`n`即`next`，运行1步。
`s`即`step`，进入这一行的函数中。

## step的局限性
上面几个命令算是调试的通用命令，比较值得注意的是最后一个`step`操作，因为有可能一行会调用多个函数，比如下一行代码
```
vector<string> v = {"hello", "world"};
```
在这一行设断点，运行至此处，然后输入`s`。
```
Breakpoint 1, main () at gdb.cc:8
8     vector<string> v = {"hello", "world"};
(gdb) s
std::allocator<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >::allocator (this=0x7fffffffe22f) at /usr/include/c++/5/bits/allocator.h:113
113       allocator() throw() { }
```
简单看，这一句涉及到下面几个操作：
1. 构造`std::initialize_list<string>`对象，包含2个字符串`"hello"`和`"world"`；
2. 调用`vector<string>`的构造函数，接收初始化列表；

但实际并不然，`string`对象会调用接收字符串字面值的构造函数，range for语法会调用到底层的`begin()`、`size()`、`end()`等方法，以及`distance()`函数之类，如果继续`s`下去，会看到大量无用信息。
因此需要在合适的时机用`n`跳到下一句，此时可以用`l`查看当前跳转到哪个文件了。比如我继续`step`两次跳转到了`vector`的构造函数
```
(gdb) s
std::vector<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, std::allocator<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > > >::vector (this=0x7fffffffe230, __l=..., __a=...) at /usr/include/c++/5/bits/stl_vector.h:373
373       vector(initializer_list<value_type> __l,
```
但这是盲目的，因为我不清楚要跳转几次，如果提前`next`进入下一步了，可能就把后面整个过程就跳出去了。
因此如果想调试`vector`的构造函数而不去管其他的杂七杂八的函数，需要精确跳转到特定函数中。

## gdb函数或文件设断点
gdb可以给文件设断点，比如我的程序是`Blob.cc`编译的，包含了自定义的头文件`Blob.h`。
用`b <file>:<number>`即可设断点，如下所示
```
(gdb) b Blob.h:25
Breakpoint 1 at 0x401da0: file Blob.h, line 25.
```
但问题是，怎么在gdb中查看`Blob.h`的内容以知道该在哪设断点？gdb不提供查看文件的命令，但是可以查看函数和类。
但对于类模板和函数模板而言，由于它们都是在编译时进行实例化的，所以需要提供模板参数。
且对于类成员函数而言，需要用`::`来指定对应成员函数。调试流程如下所示
```
(gdb) l BlobPtr<int>
35    void check(size_type i, const std::string& msg);
36  };
37
38  // 若试图访问一个不存在的元素，BlobPtr抛出一个异常
39  template <typename T>
40  class BlobPtr {
    41   public:
42    BlobPtr() = default;
43    BlobPtr(Blob<T>& a, size_t sz = 0) : wptr(a.data), curr(sz) {}
44
(gdb) l BlobPtr<int>::operator++
file: "Blob.h", line number: 137
file: "Blob.h", line number: 151
(gdb) l BlobPtr<int>::operator++(int)
146   check(curr, "operator-- past begin of BlobPtr");
147   return *this;
148 }
149
150 template <typename T>
151 BlobPtr<T> BlobPtr<T>::operator++(int) {
    152   auto ret = *this;
153   ++*this;
154   return ret;
155 }
```
C++支持重载，因此查看自增运算符时会提示两个地方，如果这两个地方在一起，则会一起显示。可以通过指定函数参数列表来查看特定的函数。
```
(gdb) b BlobPtr<int>::operator++
Breakpoint 1 at 0x402099: BlobPtr<int>::operator++. (2 locations)
(gdb) i b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   <MULTIPLE>
1.1                         y     0x0000000000402099 in BlobPtr<int>::operator++() at Blob.h:137
1.2                         y     0x000000000040217e in BlobPtr<int>::operator++(int) at Blob.h:151
(gdb) b BlobPtr<int>::operator++(int)
Note: breakpoint 1 also set at pc 0x40217e.
Breakpoint 2 at 0x40217e: file Blob.h, line 151.
(gdb) i b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   <MULTIPLE>
1.1                         y     0x0000000000402099 in BlobPtr<int>::operator++() at Blob.h:137
1.2                         y     0x000000000040217e in BlobPtr<int>::operator++(int) at Blob.h:151
2       breakpoint     keep y   0x000000000040217e in BlobPtr<int>::operator++(int) at Blob.h:151
```
可以在函数设断点，可以看到，断点编号是1.1和1.2，因为有2处重载，因此也可以通过明确参数列表来为特定的重载设置断点。

总结：通过`list`查看特定函数或类后，就可以针对特定函数设置断点，也可以针对该函数所在文件的某一行设置断点，这样就不用盲目地step了，而是精确地跳转到指定函数。
