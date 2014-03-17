---
layout: post
title: "一次调试C++程序的艰苦历程"
description: "一天遇见一个极其诡异的bug，从而开始了调试C++程序的艰苦历程。期间曾经翻遍了汇编，甚至用了模板调试，才最终定位bug。"
category: cpp
tags: [cpp, debug]
---
{% include JB/setup %}

# 项目背景

某天在用C++做一个feature的时候，发现一个对象的成员变量无论如何都写不对，而用gdb调试之，竟然发现print出来值又是对的……
为了最简化这个bug的背景，我在github上直接创建了一个[简化的repo](https://github.com/airekans/cpp-debug-01)，大家可以看看。

简单来说，这个项目的结构大致如下：

    cpp-debug-01/
    ├── App.h
    ├── Base1.h
    ├── Base.h
    ├── Child.cpp
    ├── Child.h
    ├── main.cpp
    └── Makefile

编译完之后，运行编译出来的`./test_app`就会输出下面的crash信息：

    test_app: Child.cpp:11: SuperChild::SuperChild(unsigned int*): Assertion `data != __null' failed.
    Aborted (core dumped)

OK，既然crash了，那我们来看看出问题的代码(Child.cpp)：

{% highlight cpp linenos=table %}
#include "Child.h"
#include "App.h"
#include <cstdio>

#include <cassert>


SuperChild::SuperChild(unsigned* d)
: Child(0, d, 1)
{
    assert(data != NULL);
}
{% endhighlight %}

`data`是`SuperChild`的父类`Child`的一个`protected`成员，所以说明`data`没有初始化？

{% highlight cpp linenos=table %}
// Child构造函数定义
Child(unsigned s, unsigned* d, int i)
{
    seq = s;
    data = d; // <<<<
    i_data = i;
}
{% endhighlight %}

咦？
