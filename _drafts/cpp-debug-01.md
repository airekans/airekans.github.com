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

{% highlight bash linenos=table %}
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
   0x00000000004005e8 <+80>:	leaveq 
   0x00000000004005e9 <+81>:	retq   
{% endhighlight %}

从代码中我们看到第11行是`callq`，也就是调用`Child`的构造函数。
而第15行则是一个`jne`指令，后面的地址是`SuperChild`构造函数加80，我们看到是构造函数的结束的地方。
这有点奇怪，`SuperChild`的构造函数只有一个`assert`，怎么会出现经常在`if`里面才有的`jne`指令呢？

其实assert的实现正是用的一个`if`。可以在`/usr/include/assert.h`里面看到通常情况下的`assert`是一个
宏(各个版本的libc的实现可能会有稍微的差别)：

{% highlight c %}
# define assert(expr)							\
  ((expr)								        \
   ? __ASSERT_VOID_CAST (0)						\
   : __assert_fail (__STRING(expr), __FILE__, __LINE__, __ASSERT_FUNCTION))
{% endhighlight %}

这解释了上面的`jne`指令。所以jne之前的应该是载入`data`成员的值的汇编语句。
所以关键的就是下面的这几句：

{% highlight gas linenos=table %}
mov    -0x8(%rbp),%rax
mov    0x8(%rax),%rax
test   %rax,%rax
{% endhighlight %}

这几句的意思对应着下面的C++语句：

    this->data

首先我们知道，`this`指针在C++里面实际上是相当于第一个参数传递进成员函数的，就算构造函数也不例外。
而`-0x8(%rbp)`存放在什么值呢？我们看到在上面的第6行，有这么一句：

    mov    %rdi,-0x8(%rbp)

哦，原来是从`rdi`寄存器赋值过来的。而`rdi`在x64的函数调用规则里面是用来在函数调用的时候，
存放第一个整形(或者指针)参数的。(这里多说一句，由于我机器是64位的，所以汇编跟32位的会有差别)
哦？那就正好是`this`指针也！太好了，那么就是说现在rax就已经放着`this`指针了。
接下来的一句`mov 0x8(%rax),%rax`，就是说从`this`的地方位移8的地方取出值。
嗯，这正好就是`data`的偏移值。
看起来没什么问题。好吧，取值的地方没问题，那我们看看赋值到`data`的地方吧，也就是`Child`的构造函数。

### Child构造函数

我们看看Child的构造函数的汇编代码：

{% highlight gas linenos=table %}
   0x0000000000400552 <+0>:	push   %rbp
   0x0000000000400553 <+1>:	mov    %rsp,%rbp
   0x0000000000400556 <+4>:	sub    $0x20,%rsp
   0x000000000040055a <+8>:	mov    %rdi,-0x8(%rbp)
   0x000000000040055e <+12>:	mov    %esi,-0xc(%rbp)
   0x0000000000400561 <+15>:	mov    %rdx,-0x18(%rbp)
   0x0000000000400565 <+19>:	mov    %ecx,-0x10(%rbp)
   0x0000000000400568 <+22>:	mov    -0x8(%rbp),%rax
   0x000000000040056c <+26>:	mov    %rax,%rdi
   0x000000000040056f <+29>:	callq  0x400548 <Base::Base()>
   0x0000000000400574 <+34>:	mov    -0x8(%rbp),%rax
   0x0000000000400578 <+38>:	mov    -0xc(%rbp),%edx
   0x000000000040057b <+41>:	mov    %edx,0x8(%rax)
=> 0x000000000040057e <+44>:	mov    -0x8(%rbp),%rax
   0x0000000000400582 <+48>:	mov    -0x18(%rbp),%rdx
   0x0000000000400586 <+52>:	mov    %rdx,0x10(%rax)
   0x000000000040058a <+56>:	mov    -0x8(%rbp),%rax
   0x000000000040058e <+60>:	mov    -0x10(%rbp),%edx
   0x0000000000400591 <+63>:	mov    %edx,0x18(%rax)
   0x0000000000400594 <+66>:	leaveq 
   0x0000000000400595 <+67>:	retq   
{% endhighlight %}

嗯，根据上面的经验，第10行是调用父类的构造函数。
而接下来的6行，每3行对应着C++中的一个赋值语句。那么我们关注一下`data`成员的赋值：

{% highlight gas %}
mov    -0x8(%rbp),%rax
mov    -0x18(%rbp),%rdx
mov    %rdx,0x10(%rax)
{% endhighlight %}

第一行是载入`this`指针到`rax`寄存器，第二行是将`d`的值载入到`rdx`。
所以第三行的就是将`d`的值赋给`data`成员。等等，咦？为什么`data`会是`0x10(%rax)`？
上面在`SuperChild`里面，明明`data`是`0x8(%rax)`啊！怎么会相差了8的位移呢？

为什么会位移不一样呢？是父类的size不对吗？怎么能够看见父类的size呢？

# 模板静态断言

为了能够判断一个类的大小，一般来说就使用`sizeof`来看了。比如说像下面这样：

{% highlight cpp %}
std::cout << sizeof(Base) << std::endl;
{% endhighlight %}

但是这得在运行期才能看得见，而我希望在编译的时候就能够看见，有没有什么办法呢？
是有的，在C++中，能够利用一个模板技巧来达到静态断言的效果。
先来看看怎么做。我在`Child`的构造函数中利用静态断言来断言`Base`的大小为8，
因为在`Base.h`里面就是两个`unsigned`的大小嘛，而每个`unsigned`大小是4。
所以就有了下面的代码：

{% highlight cpp linenos=table %}
template<unsigned size>
class TestSize;

template<>
class TestSize<8> {};

class Child : public Base
{
public:
    Child(unsigned s, unsigned* d, int i)
    {
        TestSize<sizeof(Base)> test_size;
        // ...
    }
{% endhighlight %}

上面的代码关键的就是模版`TestSize`。他利用了模板的偏特化特性，特化了一个只对8有效的特化类。
而其他的值是没有办法产生对象的，因为其他的值并没有具体的定义。
而在`Child`的构造函数里面，我们用`sizeof(Base)`来作为模板参数，所以只有当`sizeof(Base)`为8的时候，
编译才可以通过，也这就是静态断言的一种用法。
(这种用法在["Modern C++ Design"](http://www.amazon.com/Modern-Design-Generic-Programming-Patterns/dp/0201704315)有详细的介绍)

好吧，有了上面的断言，我们来编译一下：

    g++ -g -c -o main.o main.cpp -I.
    g++ -g -c -o Child.o Child.cpp -I.
    In file included from Child.cpp:1:0:
    Child.h: In constructor ‘Child::Child(unsigned int, unsigned int*, int)’:
    Child.h:19:32: error: aggregate ‘TestSize<4u> test_size’ has incomplete type and cannot be defined
    make: *** [Child.o] Error 1

哦！真的在Child.cpp编译出现了错误，可以看见对于`Child.cpp`来说，`Base`竟然大小是4，而不是8！

为什么会这样呢？明明`main.cpp`是没问题的，为什么`Child.cpp`却有问题呢？难道他们包含的`Base`不是同一个吗？

我们仔细看看`main.cpp`的头文件包含：

    #include "App.h"
    #include "Child.h"

然后我们再来看看`Child.cpp`的头文件包含：

    #include "Child.h"
    #include "App.h"

发现对于`App.h`和`Child.h`的包含顺序是反过来的。那么这两个头文件有什么玄机呢？
我们看看`App.h`：


