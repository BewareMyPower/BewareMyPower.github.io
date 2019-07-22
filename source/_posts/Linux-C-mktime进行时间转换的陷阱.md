---
title: Linux C mktime进行时间转换的陷阱
date: 2019-07-22 22:10:36
tags:
  - C++
top: 1
---

## 前言

最近做的一个小程序，需要对时间戳和对应日期字符串进行相互转换，于是二话不说直接翻看*The Linux Programming Interface*(*TLPI*)查API。翻到了下面这张图：

![时间格式转换函数](时间格式转换函数.JPG)

我的时间戳是自epoch(UTC)以来的毫秒数表示，拟定转换的是年月日时分秒，外加个毫秒，思路就很简单：

1. 输入字符串：用`strptime`转换成`struct tm`类型，再用`sscanf`读取毫秒，最后用`mktime`将`tm`对象转换成`time_t`（自epoch(UTC)以来的秒数）乘以1000加上毫秒数；
2. 输入时间戳：除以1000得到秒数，模1000得到毫秒数，然后用`strftime`将秒数格式化，再用`snprintf`将毫秒数格式化和'.'一起添加到末尾。

当然，输入字符串的情况下，考虑健壮性的话需要对`tm`对象的各字段进行合法性检查，这里就不详述了。

## 奇妙的BUG

但是写完后进行测试，输入字符串，转换成时间戳，然后再转换回字符串。发现一个十分奇葩的错误，就是转换回去后比原来要少了1给小时，比如"2000-02-29 10:01:20.094"会变成"2000-02-29 09:01:20.094"，也就是说其他的功能都没错。

在此之前我已经考虑到了时区的问题，因此确认过`mktime`的输入参数是本地时区，因此`strftime`的输入参数需要用`localtime`而非`gmtime`。

为了复现这个BUG，以及描述问题的原因，可以编译运行下面这段代码（忽略了返回值检查）：

```
#include <stdio.h>
#include <time.h>

int main(int argc, char* argv[]) {
  if (argc < 2) {
    fprintf(stderr, "Usage: %s yyyy-mm-dd hh:mm:dd\n", argv[0]);
    return 1;
  }
  printf("before: %s\n", argv[1]);
  struct tm tm_;
  strptime(argv[1], "%F %T", &tm_);
  auto dump_tm = [](const struct tm* tmp, const char* msg) {
    printf("%s: %04d-%02d-%02d %02d:%02d:%02d\n", msg, tmp->tm_year + 1900,
           tmp->tm_mon + 1, tmp->tm_mday, tmp->tm_hour, tmp->tm_min,
           tmp->tm_sec);
  };
  dump_tm(&tm_, "before mktime");
  auto timestamp = mktime(&tm_);
  dump_tm(&tm_, "after mktime");
  char buf[128];
  strftime(buf, sizeof(buf), "%F %T", localtime(&timestamp));
  printf("after: %s\n", buf);
  return 0;
}
```

设置几个时间，运行结果如下：

```
# ./a.out "2001-02-28 01:00:00"
before: 2001-02-28 01:00:00
before mktime: 2001-02-28 01:00:00
after mktime: 2001-02-28 01:00:00
after: 2001-02-28 01:00:00
# ./a.out "2000-02-29 01:00:00"
before: 2000-02-29 01:00:00
before mktime: 2000-02-29 01:00:00
after mktime: 2000-02-29 00:00:00
after: 2000-02-29 00:00:00
# ./a.out "2004-02-29 01:00:00"
before: 2004-02-29 01:00:00
before mktime: 2004-02-29 01:00:00
after mktime: 2004-02-29 00:00:00
after: 2004-02-29 00:00:00
```

可以发现中间的输入结果有误，一开始怀疑是闰年的缘故，但是2004年和2000年的结果并不相同，而它们都是闰年。此外实测发现"2000-01-29 01:00:00"也出错。

## 原因及解决方法

其实问题的关键出在`struct tm`结构的`tm_idst`字段，可以发现无论结果是否转换错误，`mktime`始终把`tm_idst`重置为0，而调用之前`tm_idst`为非零值。

这个字段即DST，Daylight Saving Time。若大于0则将该时间视为夏令时，若为0则将该时间视为标准间(忽略夏令时)，若小于0则试图使用时区信息和系统数据库来确定设置。而mktime()在进行转换时会对时区进行设置，若DST未生效，则将`tm_idst`置为0，若DST生效，则会将其置为正值。

因此就是**夏令时**的问题，[struct tm中的tm_idst以及mktime](https://blog.csdn.net/duyiwuer2009/article/details/42459677)的测试中2001年以前的时间使用DST则会比其他情况晚1小时，当然，这个测试和我的略有出入，但我测试的2001年之后的确实也没出现这问题。

[mktime 夏令时](https://www.cnblogs.com/dongzhiquan/archive/2011/11/05/2237075.html)则使用了一种叫较为复杂的方法。

这个问题确实造成了不少人的困扰，最简单的方法就是在`mktime`之前将`tm_idst`设为-1，让系统为你解决这个问题。但实际上并非如此，比如[mktime 夏令时](https://www.cnblogs.com/dongzhiquan/archive/2011/11/05/2237075.html)文中就提到了：

> 俄罗斯时间2008年10月26日2:30由于夏令时的跳变会经过2次，这2次所代表的日历时间明显不同。

stackoverflow上也有讨论：[mktime-and-tm-isdst](https://stackoverflow.com/questions/8558919/mktime-and-tm-isdst)，其中Rich Jahn也提到了即使设为-1也不代表能“自动推断是否使用夏令时：

> -1 is a possible input, but I would think of it as meaning "Unknown". Don't think of it as meaning "determine automatically", because in general, mktime() can't always determine it automatically.
>
> The explicit DST status (0 or 1) should come from something external to the software, for example store it in the file or database, or prompt the user.

最好的解决方法还是在时间后面加上UTC，比如：

```
struct tm tm_;
char* p = strptime("2004-02-29 01:00:00.039 UTC", "%F %T", &tm_);
```

调用完毕后返回值`p`指向的是`".039 UTC"`，后缀`UTC`并不影响返回值，因此仍然可以对`p`进行`sscanf`或者`strtol`操作获取毫秒数。
