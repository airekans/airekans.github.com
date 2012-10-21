---
layout: post
title: "Pimpl Idiom in C++"
description: ""
category: 
tags: [C++, design pattern]
---
{% include JB/setup %}

在C++里面, 经常出现的情况就是头文件里面的类定义太庞大了，而这个类的成员变量涉及了很多
其他文件里面的类，从而导致了其他引用这个类的文件也依赖于这些成员变量的定义。
在这种情况下，就出现了在C++里面特有的一个idiom，叫做Pimpl idiom。

考虑一下下面的情况，假设有一个类A，它包含了成员变量b和c，类型分别为B和C，而如果D类
要使用A类的话，那也变相依赖了B和C。如下：

{% highlight cpp linenos %}
#include "B.h"
#include "C.h"

class A
{
private:
    B b;
    C c;
};{% endhighlight %}

这个时候如果D要使用A类的话，那么D就要像下面那样去写：

{% highlight cpp linenos %}
#include "A.h"

class D
{
private:
    A a;
};{% endhighlight %}

虽然形式上是只需要include A.h，但是在链接程序的时候，却需要把B和C的模块也一并链接进去。

初步的解决方案可以是把A里面的b和c变成指针类型，然后利用指针声明的时候类型可以是不完全类型，
从而在A.h里面不用include B.h和C.h。当然，这也只是解决的部分的问题。
如果A里面需要用到十几个成员变量的话，这个时候头文件的size就会变得很大，这也是一个问题。
而且有些时候，变成指针类型也不一定是可行的。这个时候，一个简单的想法就是把所有私有的
成员变量的声明都放到cpp文件里面去，这样使用A的类就可以完全不用知道A类的成员变量了。

而Pimpl idiom就是这样的解决方案。所谓的Pimpl idiom，就是声明一个类中类，
然后再声明一个成员变量，类型是这个类中类的指针。用上面的例子来说明一下会清楚一下，
代码如下：

{% highlight cpp linenos %}
class A
{
private:
    struct Pimpl;
    Pimpl* m_pimpl;
};{% endhighlight %}

有了上面的定义，那么D类就可以完全不用知道A类的细节，而且链接的时候也可以完全不用管B和C了。
然后在A.cpp里面，我们就像下面这样去定义就好了：

{% highlight cpp linenos %}
struct A::Pimpl
{
    B b;
	C c;
};

A::A()
: m_pimpl(new Pimpl)
{
    m_impl->b; // 使用b
}{% endhighlight %}

而现在我们STL有auto\_ptr，boost有shared\_ptr，再要自己来管理内存好像
就有写多次一举了。所以在Herb Sutter的[Using auto_ptr Effectively](http://www.gotw.ca/publications/using_auto_ptr_effectively.htm)里面，
也提到了用auto\_ptr来进行“经典”的Pimpl的编写。

也就是如下面这样：

{% highlight cpp linenos %}
#include <memory>

class A
{
public:
    A();

private:
    struct Pimpl;
    std::auto_ptr<Pimpl> m_pimpl;
};{% endhighlight %}

可以当你写了上面的代码之后，编译，Bang! 编译器给你报一个错，说是Pimpl是incomplete
type。这下你就蒙了吧？！

其实要fix上面的编译错误，你只需要加上A的

