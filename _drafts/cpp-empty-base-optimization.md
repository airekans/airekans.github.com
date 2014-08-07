# C++中的Empty Base Optimization

说到Empty Base Optimization(简称ebo)可能大家还是比较陌生，但是C++中每天都在用的`std::string`中就用到了ebo。

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
