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

    #include "Base.h"
    #include "Child.h"

嗯，在头文件的部分，是先include的`Base.h`，然后include的`Child.h`。那Child.h呢？

    #include "Base1.h"
    
咦？`Child.h`竟然include了一个`Base1.h`？这是啥？跟`Base.h`有什么关系？
我们来看看`Base.h`和`Base1.h`的diff：

{% highlight diff %}
--- Base.h	2014-03-17 23:19:05.980027339 +0800
+++ Base1.h	2014-03-17 23:19:05.980027339 +0800
@@ -1,11 +1,10 @@
 #ifndef _BASE_H_
 #define _BASE_H_
 
 class Base
 {
 protected:
     unsigned u_data;
-    unsigned m_data;
 };
 
 #endif
{% endhighlight %}

好嘛，`Base1.h`除了少了一个`m_data`之外，竟然其他都全部一样的！！
这回真相大白了。

# Bug分析

为什么上面的`Base1.h`会导致bug呢？首先`Base1.h`和`Base.h`基本一样，连[include guard](http://en.wikipedia.org/wiki/Include_guard)都一样。
然后我们看出问题的`Child.cpp`的include链是什么样子的(顺序按从左到右)：

    Child.cpp
       |     \
    Child.h  App.h
       |      |   \
    Base1.h Base.h Child.h

由于`Base1.h`和`Base.h`的include guard是一样的，所以由于`Base1.h`在`Base.h`之前include，
所以只会include `Base1.h`，而`Base.h`的内容会直接被忽略。
所以对于`Child.cpp`来说，整个`SuperChild`的继承体系是这样的(我把成员写在类名的后面)：

    SuperChild []
        |
      Child [seq, data, i_data]
        |
      Base (in Base1.h) [u_data]

好吧，那对于`main.cpp`来说呢？include链就会是下面的样子：

      main.cpp
       |     \
     App.h    Child.h
       |   \        \
    Base.h Child.h  Base1.h

这里因为`Base.h`在`Base1.h`的前面，所以`Base1.h`就直接被忽略了。
所以从`main.cpp`的角度来看，`SuperChild`的继承体系就是这个样子：

    SuperChild []
        |
      Child [seq, data, i_data]
        |
      Base (in Base.h) [u_data, m_data]

看见了吧？从两个不同的编译单元来看，这个`SuperChild`的大小根本就是不一样的！
最直接的原因就是`Base`被定义在两个不同的头文件，而且大小也不一样，
导致了在`Child`的构造函数中，看见的`data`成员的偏移值和在`SuperChild`中看见的是不一样的。
这就导致了我们说的这个bug。

首先从`main.cpp`看见的`Child`类，我们看看他的内存布局：

    起始地址    成员    类型   
        0    | u_data | unsigned
        4    | m_data | unsigned
        8    |  seq   | unsigned
        16   | data   | unsigned*
        24   | i_data | int

而对于`Child.cpp`看见的`Child`类，内存布局是下面这样的：

    起始地址    成员    类型   
        0    | u_data | unsigned
        4    |  seq   | unsigned
        8    | data   | unsigned*
        16   | i_data | int

所以在`Child`中的构造函数中赋值给`data`是会赋值到16这个偏移中的，
而在`SuperChild`中取`data`，是会取到偏移8的，也就是对应到原来的`seq`的值。
而我们在`SuperChild`给传的`seq`的初始值正正就是0。
这终于解释了为什么在`SuperChild`的`data`的值总是0了。

这真是非常愚蠢的bug啊……但是一般愚蠢的bug，都需要极其变态的debug过程才能找的出来……

## 如何fix这个bug

其实修复这个bug非常简单……只需要把`Child.h`中的`include "Base1.h"`改成`include "Base.h"`，
也就是无论如何都用`Base.h`就好了。

当然，这个bug的最主要的原因就是出现了`Base1.h`，这个很有可能是一个源文件的两个不同版本。
这里就需要在用SCM的时候，更新的时候就直接修改源代码，不要copy一份，这样非常容易出问题……

同时，注意到google的c++编程规范中，给出了一个[关于头文件的规定](http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml#Names_and_Order_of_Includes)，
其中有一个*隐含单没有明说*的规定，就是包含头文件的时候尽可能的不要用相对路径，
而是直接从项目根目录一直写下来，比如`common/base.h`这样写，而不是`base.h`并加上`-I`的编译选项。
这是在工程实践中非常重要的点，因为不用`-I`而是写全路径，可以加快编译速度，
并且还可以最大程度的避免我遇到的这个bug。
为什么这么说呢？因为如果出问题而相同的两个文件同名但在不同的目录下的话，
如果在编译的时候利用`-I`选项是很可能在两个不同的编译单元包含了不同的头文件的。
而如果用全路径的话，是完全不会出现这个bug。可以看到google的很多开源项目都是遵循这个规定的，
比如protobuf。

# 一个插曲

不知道大家上面有没有注意到一个细节，就是下面的内存布局中，`data`的起始地址：

    起始地址    成员    类型   
        0    | u_data | unsigned
        4    | m_data | unsigned
        8    |  seq   | unsigned
        16   | data   | unsigned*
        24   | i_data | int

为什么命名`seq`是一个`unsigned`类型，也就是大小是4的成员，但是`data`的对象的地址却比`seq`多了8。
多了的4个byte去了哪里呢？其实对于有经验的人来说，很快就会猜到，这应该是padding造成的。
但是padding不是一般是以4为单位的吗？这里刚好是4啊，不需要padding啊。

要注意到padding的单位是和CPU的word长度一致的，在32位系统上面，word的大小是4，所以padding也是4。
但是在64位系统上，word的大小是8，这表示什么呢？这表示一个指针的大小是8 byte，
并且他的地址必须是8的倍数。所以在上面的例子中，就产生了一个padding。
具体的内存如下图所示：

    0        4        8      12         16    24
    | u_data | m_data | seq   | padding | data | i_data

至此，终于完成的修复并分析完了这个bug。


