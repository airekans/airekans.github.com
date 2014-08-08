---
layout: post
title: "C++中的Empty Base Optimization"
description: "本文介绍C++中的Empty Base Optimization，并用两个例子介绍他的使用方法。"
category: cpp
tags: [cpp]
---
{% include JB/setup %}

# 什么是Empty Base Optimization？

说到C++中的Empty Base Optimization(简称ebo)可能大家还是比较陌生，但是C++中每天都在用的`std::string`中就用到了ebo。

那么到底什么是ebo呢？
其实ebo就是当一个类的对象理想内存占用可以为0的时候，把这个类的对象作为另一个类的成员时，把其内存占用变为0的一种优化方法。
说起来可能有点绕，还是用一个例子来说明一下吧，看下面的代码：

{% highlight cpp linenos=table %}
#include <iostream>
using namespace std;

class Base
{};

int main()
{
    cout << "sizeof(Base) " << sizeof(Base) << endl;

    Base obj1;
    Base obj2;

    cout << "addr obj1 " << (void*) &obj1 << endl;
    cout << "addr obj2 " << (void*) &obj2 << endl;

    return 0;
}{% endhighlight %}

大家能猜到上面的代码的输出吗？`sizeof(Base)`会是0吗？`obj1`的地址会和`obj2`的一样吗？

自己编译上面的代码，运行一下，会得到类似下面的输出(第2、3行会略有不同)：

    sizeof(Base) 1
    addr obj1 0xbfdc9033
    addr obj2 0xbfdc9032

看见了吧？就算`Base`不包含任何的成员，编译器也会让`Base`占1 byte。
这是因为如果一个类的内存占用为0，那么连续的分配对象有可能会有同一个内存地址，这个是不合理的。
所以编译器为了避免这种情况，让空的类也会占有1 byte的大小。

那么如果我要用`Base`作为另一个类的成员变量呢，比如下面这样：

{% highlight cpp %}
class TestCls
{
    Base m_obj;
    int m_num;
};

int main()
{
    cout << "sizeof(TestCls) " << sizeof(TestCls) << endl;
    
    return 0;
}{% endhighlight %}

知道上面的输出会是多少吗？5？
在32位的机器上面是8，因为编译器为了存取的方便，会在`m_obj`的后面产生3 byte的padding，以和机器字对齐。
总之答案不会是4。

但是在内存非常紧张的情况下，还真的会想要让`TestCls`的size是4。有办法吗？
这里就可以用到今天介绍的`ebo`了，看下面的代码：

{% highlight cpp %}
class TestCls : public Base
{
    int m_num;
};

int main()
{
    cout << "sizeof(TestCls) " << sizeof(TestCls) << endl;
    return 0;
}{% endhighlight %}

这次能猜到输出是多少吗？没错，就是我们想要的4！
当我们把空的类作为基类的时候，编译器就会把这个基类的size去掉，做了优化，
从而使得整个对象占有真正需要的size。

那么如果这个子类除了基类之外，没有别的成员呢？如下面：

{% highlight cpp %}
class TestCls : public Base
{};

int main()
{
    cout << "sizeof(TestCls) " << sizeof(TestCls) << endl;
    return 0;
}{% endhighlight %}

上面的代码输出仍然是1，因为如果这个类本身除了空基类之外没别的成员，
说明这个类本身也是一个空类，所以最开始说的情况就适用于这里。
编译器就给空类给了1的size。

上面说的就是Empty Base Optimization了。那么现实中哪里使用到了这个技巧呢？
除了最开始提到的`std::string`之外，Google的[cpp-btree](https://code.google.com/p/cpp-btree/)也用到了这个技巧。
下面我们来看看这两个现实中的例子。

# STL中的string

C++每天都用的string中就用到了ebo。我们来看看string是如何定义成员的(省略函数定义，以下代码源自gcc 4.1.2 c++)：

{% highlight cpp %}
template<typename _CharT, typename _Traits, typename _Alloc>
class basic_string
{
public:
    mutable _Alloc_hider      _M_dataplus;
};{% endhighlight %}

注意`string`实际上是模板类`basic_string`的一个特化类。而`basic_string`只包含了一个成员`_M_dataplus`，
其类型为`_Alloc_hider`。

我们来看看`_Alloc_hider`是怎么定义：

{% highlight cpp %}
template<typename _CharT, typename _Traits, typename _Alloc>
class basic_string
{
private:
    struct _Alloc_hider : _Alloc // Use ebo
    {
        _CharT* _M_p; // The actual data.
    };
};{% endhighlight %}

`_Alloc_hider`继承于模板参数类`_Alloc`(并且还是私有继承)，还有一个自己的成员`_M_p`。
`_M_p`是用来存放实际数据的，而`_Alloc`呢？熟悉STL的人可能还记得STL里面有一个allocator。
这个allocator一般的实现都是没有任何的数据成员，只有static函数的。
所以这个类是一个空类。
默认的string就是将这个allocator当作模板参数传递到`_Alloc`。
所以`_Alloc`大多数情况下都是空类，而string经常会在程序中用到，
还很经常会大量的使用，比如在容器中，这个时候就需要考虑内存占用了。
所以在这里就是用了ebo的优化。

可能会有人会问，`string`里面实际上只有`char*`，但是不是说`string`还记录了size，
还用到了*copy on write*技术的吗？那怎么只有一个`char*`呢？
这个和`string`的实现中的内存布局相关，我会专门写一篇文章解析一下，先在这里挖个坑 :)

# cpp-btree中的ebo

[cpp-btree](https://code.google.com/p/cpp-btree/)是Google出的一个基于B树的模板容器类库。如果有不熟悉B树的童鞋，可以移步[这里](https://www.cs.usfca.edu/~galles/visualization/BTree.html)
看一看这个数据结构的动画演示。

B树是一种平衡树结构，一般常用于数据库的磁盘文件数据结构(不过一般会用其变体B+树)。而cpp-btree则是全内存的，和`std::map`类似的一种容器实现，其对于大量元素(>100w)的存取效率要高于`std::map`的红黑树实现，并且还节省内存。

关于cpp-btree的广告就卖到这里，我们看看他哪里使用了ebo。
在cpp-btree里面提供了`btree_set`和`btree_map`两个容器类，
而他们的公共实现都在`btree`这个类里面。
`btree`这个类实现了主要的B树的功能，而其成员定义如下：

{% highlight cpp %}
template <typename Params>
class btree : public Params::key_compare {
private:
  typedef typename Params::allocator_type allocator_type;
  typedef typename allocator_type::template rebind<char>::other
    internal_allocator_type;

  template <typename Base, typename Data>
  struct empty_base_handle : public Base {
    empty_base_handle(const Base &b, const Data &d)
        : Base(b),
          data(d) {
    }
    Data data;
  };

  empty_base_handle<internal_allocator_type, node_type*> root_;
};{% endhighlight %}

可以看见`btree`这个类里面只包含了`root_`这一个成员，其类型为`empty_base_handle`。
`empty_base_handle`是一个继承于Base的类，在这里，
`Base`特化成`internal_allocator_type`。
从名字可以看出`internal_allocator_type`是一个allocator，
而在默认的`btree_map`实现中，这个allocator就是`std::allocator`。
所以一般情况下，`Base`也是一个空类。

这里`btree`也利用了ebo节省了内存占用。

# 一个例外

在编译器判断是否做ebo的时候，有这么一个例外，就是虽然继承于一个空类，
但是子类的第一个非static成员的类型也是这个空类或者是这个类的一个子类。
在这种情况下，编译器是不会做ebo的。

有点绕，我们看看下面的代码就明白了：

{% highlight cpp %}
#include <iostream>
using namespace std;

class Base
{};

class TestCls : public Base
{
public:
    Base m_obj; // <<<<
    int m_num;
};

int main()
{
    cout << "sizeof(Base) " << sizeof(Base) << endl;
    cout << "sizeof(TestCls) " << sizeof(TestCls) << endl;

    TestCls obj;

    cout << "addr obj " << (void*) &obj << endl;
    cout << "addr obj.m_obj " << (void*) &(obj.m_obj) << endl;
    cout << "addr obj.m_num " << (void*) &(obj.m_num) << endl;

    return 0;
}{% endhighlight %}

运行一下上面的代码，你会看到，`TestCls`的size是8，并且`obj`的地址和`obj.m_obj`的地址并不一样。
这说明了ebo并没有进行。
