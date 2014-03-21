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

咦？代码中明明初始化了啊，怎么会是`NULL`呢？难道我穿进来的这个`d`指针的值不对？
看来要祭出`gdb`这个杀器才行。

# gdb调试

我们`gdb`一下我们的程序，在`main`函数调用`SuperChild`那设一个断点好了。

    airekans@test-host:~/programming/test/cpp-debug-01$ gdb -q ./test_app 
    Reading symbols from ~/programming/test/cpp-debug-01/test_app...done.
    (gdb) b main.cpp:8
    Breakpoint 1 at 0x40052d: file main.cpp, line 8.
    (gdb) run
    Starting program: ~/programming/test/cpp-debug-01/test_app 
    Breakpoint 1, main () at main.cpp:8
    8	    SuperChild c1(&i);
    (gdb)

太好了，我们看看现在穿进去的这个指针的值：

    (gdb) p i
    $1 = 345
    (gdb) p &i
    $2 = (unsigned int *) 0x7fffffffdc6c

嗯，一切正常的样子。我们继续进去`SuperChild`的构造函数看看：

    (gdb) s
    SuperChild::SuperChild (this=0x7fffffffdc40, d=0x7fffffffdc6c) at Child.cpp:9
    9	: Child(0, d, 1)
    (gdb) p d
    $3 = (unsigned int *) 0x7fffffffdc6c

我打印了一下穿进来的指针`d`，值是对的。好的，那我们进去父类`Child`的构造函数继续看：

    (gdb) s
    Child::Child (this=0x7fffffffdc40, s=0, d=0x7fffffffdc6c, i=1) at Child.h:10
    10	    {
    (gdb) n
    11	        seq = s;
    (gdb) n
    12	        data = d;
    (gdb) n
    13	        i_data = i;
    (gdb) p d
    $4 = (unsigned int *) 0x7fffffffdc6c
    (gdb) p data
    $5 = (unsigned int *) 0x7fffffffdc6c
    (gdb) p this->data
    $6 = (unsigned int *) 0x7fffffffdc6c

嗯，构造函数已经把这个指针的值赋给了成员`data`。而我也确认了指针和成员的值都是对的。
嗯，看起来程序都很正常，难道刚才的crash只是个美丽的误会？哈哈哈哈，好吧，试一下继续好了：

    (gdb) c
    Continuing.
    test_app: Child.cpp:11: SuperChild::SuperChild(unsigned int*): Assertion `data != __null' failed.
     
    Program received signal SIGABRT, Aborted.
    0x00007ffff7a51425 in raise () from /lib/x86_64-linux-gnu/libc.so.6

啊？`assert`还是失败了？先看看data的值……

    #4  0x00000000004005e8 in SuperChild::SuperChild (this=0x7fffffffdc40, 
        d=0x7fffffffdc6c) at Child.cpp:11
    11	    assert(data != NULL);
    (gdb) p data
    $7 = (unsigned int *) 0x0

`data`竟然是0！！！天啊，难道我遇见了*薛定鄂的bug*？！刚才明明还是正常的啊，为什么这里就变成0了呢……
是谁改变了我的`data`啊？

要看数据怎么变化，看来这次是要用数据断点了。

## 数据断点

重启在gdb里面运行一次程序，这次我们到了`Child`的构造函数里面之后，对于`data`成员设定一个数据断点看看：

    (gdb) p &data
    $14 = (unsigned int **) 0x7fffffffdc50
    (gdb) watch *(unsigned*) 0x7fffffffdc50
    Hardware watchpoint 4: *(unsigned*) 0x7fffffffdc50

可以看到`data`成员的地址是`0x7fffffffdc50`，然后我们看看在`Child`里面设置的时候会不会停下来：

    (gdb) c
    Continuing.
    Hardware watchpoint 4: *(unsigned*) 0x7fffffffdc50
     
    Old value = 0
    New value = 4294958188
    Child::Child (this=0x7fffffffdc40, s=0, d=0x7fffffffdc6c, i=1) at Child.h:13
    13	        i_data = i;

嗯，停下来的，说明的确是改变了值，我们查看一下，确认一下：

    (gdb) p d
    $15 = (unsigned int *) 0x7fffffffdc6c
    (gdb) p data
    $16 = (unsigned int *) 0x7fffffffdc6c

好的，那从`Child`返回之后，我们再看看。是没有停下来的，说明数据没有被改变。这个时候我们来看看`data`的值：

    (gdb) n
    SuperChild::SuperChild (this=0x7fffffffdc40, d=0x7fffffffdc6c) at Child.cpp:11
    11	    assert(data != NULL);
    (gdb) p data
    $17 = (unsigned int *) 0x0

！！！T_T 是见鬼了吗，明明连数据断点都没有触发啊，但是为什么值会改变了啊……

连数据断点都不管用了，这回我只能老老实实的乖乖看汇编代码了……

## 反汇编

用gdb里面的`disassemble`可以用来查看当前栈帧的函数反汇编结果。
下面我们来看看`SuperChild`的构造函数的汇编(只看到assert调用)：

{% highlight linenos=table %}
(gdb) disassemble 
Dump of assembler code for function SuperChild::SuperChild(unsigned int*):
   0x0000000000400598 <+0>:	push   %rbp
   0x0000000000400599 <+1>:	mov    %rsp,%rbp
   0x000000000040059c <+4>:	sub    $0x10,%rsp
   0x00000000004005a0 <+8>:	mov    %rdi,-0x8(%rbp)
   0x00000000004005a4 <+12>:	mov    %rsi,-0x10(%rbp)
   0x00000000004005a8 <+16>:	mov    -0x8(%rbp),%rax
   0x00000000004005ac <+20>:	mov    -0x10(%rbp),%rdx
   0x00000000004005b0 <+24>:	mov    $0x1,%ecx
   0x00000000004005b5 <+29>:	mov    $0x0,%esi
   0x00000000004005ba <+34>:	mov    %rax,%rdi
   0x00000000004005bd <+37>:	callq  0x400552 <Child::Child(unsigned int, unsigned int*, int)>
=> 0x00000000004005c2 <+42>:	mov    -0x8(%rbp),%rax
   0x00000000004005c6 <+46>:	mov    0x8(%rax),%rax
   0x00000000004005ca <+50>:	test   %rax,%rax
   0x00000000004005cd <+53>:	jne    0x4005e8 <SuperChild::SuperChild(unsigned int*)+80>
   0x00000000004005cf <+55>:	mov    $0x400720,%ecx
   0x00000000004005d4 <+60>:	mov    $0xb,%edx
   0x00000000004005d9 <+65>:	mov    $0x400700,%esi
   0x00000000004005de <+70>:	mov    $0x40070a,%edi
   0x00000000004005e3 <+75>:	callq  0x400400 <__assert_fail@plt>
{% endhighlight %}



