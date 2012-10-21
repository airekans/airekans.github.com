---
layout: post
title: "Pimpl Idiom in C++"
description: ""
category: 
tags: [C++, design pattern]
---
{% include JB/setup %}

# Introduction
----

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

# Pimpl Idiom
----

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

其实要fix上面的编译错误，你只需要加上A的destructor的声明，然后在cpp文件里面实现一个
空的destructor就可以了。

但是这个是为什么呢？

## auto_ptr的模板特化

其实上面问题的原因，是跟模板特化的这个C++变态特性有关的。

我们先来看一下auto_ptr的简化定义：

{% highlight cpp linenos %}
template <typename T>
class auto_ptr
{
public:
    auto_ptr()
	: m_ptr(NULL)
	{}
    
    auto_ptr(T* p)
	: m_ptr(p)
	{}
    
    ~auto_ptr()
	{
	    if (m_ptr)
	    {
		delete m_ptr;
	    }
	}

private:
    T* m_ptr;
};{% endhighlight %}

我们看到auto\_ptr在他的构造函数里面自动的delete了他的m\_ptr，这个就是比较经典的
利用RAII实现的智能指针了。

然后还要知道，auto_ptr是一个模板类，而模板类的一个特点是，
**当他的成员函数只有在被调用的时候才会真正的做函数特化**。

也就是说，如果有下面的这样一个模板类：

{% highlight cpp linenos %}
template <typename T>
class TemplateClass
{
public:
    void Foo()
	{
	    int a = 1;
	    return;
	}

    void Bar()
	{
	    this->m_ptr = "syntax correct, but semantic incorrect.";
	}
};

int main(int argc, char *argv[])
{
    TemplateClass<int> a;
    a.Foo();
    return 0;
}{% endhighlight %}

上面的代码，是可以通过编译并且正确运行的。可以看到Foo这个函数是正确的，而Bar函数虽然
语法上是正确的，但是他的语义是错的。但是由于我们只调用了Foo，没有调用Bar，
所以只有Foo被真正的特化并且做了完全的编译，而Bar只是做了语法上的检查，
并没有做语义的检查。所以上面的代码在C++里面是100%的正确的。

所以auto\_ptr里面的成员函数，包括构造和析构函数，都是在被调用的时候才进行真正的特化。

## Default Destructor

还记得在学C++的刚开始的时候书上这么说过，不定义构造函数或者析构函数，
那么编译器会帮我们造一个默认的。而这个默认的构造或者析构函数只会做成员变量还有父类的
默认初始化或者析构，其他什么都不会做。

那么我们看回利用了Pimpl的A的定义。在这个定义里面，由于我没有写析构函数的声明，
所以编译器自动帮我定义了一个。而A里面有一个auto\_ptr成员变量，所以在这个默认的
析构函数里面会析构这个成员变量。所谓的析构，其实就是调用析构函数而已。
所以，在这个默认的析构函数里面，调用了auto\_ptr的析构函数，这个时候，
auto\_ptr的析构函数就被编译器特化了。

而在auto\_ptr的析构函数里面，delete了模板参数的指针类型的成员变量。
而在A这个例子里面，模板参数就是Pimpl。而在特化的这一瞬间，Pimpl是被声明了，
但是还没有被定义。

