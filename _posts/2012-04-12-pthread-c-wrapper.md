---
layout: post
title: "由pthread C++ wrapper引发的血案"
description: ""
category: cpp
tags: [cpp, async]
---
{% include JB/setup %}

最近用C++实现pthread线程池的时候, 研究了一下C++里面实现线程的方式。主要是由下面两种：

1.  一个`Thread`基类，用户的线程类通过继承这个`Thread`基类并重写父类中特定方法来实现线程执行函数。
1.  一个`Thread`类，定义了一个 `Run()`函数，函数的参数是一个`Functor`，当线程执行的时候，就会执行这个`Functor`。

方案一大概是下面的感觉：

{% highlight cpp linenos %}
class Thread {
  static void* ThreadFunc(void* arg)
  {
     Thread* t = static_cast<Thread*>(arg);
     t->Entry();
     return NULL;
  }
 
public:
  Thread() {}
  ~Thread()
  {
    pthread_join(m_id, NULL);
  }
 
 
  void Run()
  {
    pthread_create(&m_id, NULL, ThreadFunc, this);
  }
private:
  virtual void Entry() = 0;
 
  pthread_t m_id;
};{% endhighlight %}

注意到，我设计上是希望这个线程类是joinable的，而且在析构函数里面自动的join。这样用户在用这个线程类的时候就比较方便，不用担心线程的结束。

对于方案二，代码大概就是下面这样：

{% highlight cpp linenos %}
class Thread {
  template<typename Func>
  static void* ThreadFunc(void* arg)
  {
    auto_ptr<Func> f(static_cast<Func*>(arg));
    (*f)(); // call f
    return NULL;
  }
public:
  template<typename Func>
  void Run(Func f)
  {
    auto_ptr<Func> func(new Func(f));
    pthread_create(m_id, NULL, ThreadFunc<Func>, func.get());
  }
};{% endhighlight %}

从一个用户的角度，我觉得通过继承一个类然后override他的一个虚方法来编写线程函数会直观一些。比如说像下面这样写一个线程类来输出”hello, world”：

{% highlight cpp linenos %}
class HelloWorldThread : public Thread {
public:
  void Entry()
  {
    cout << "hello, world" << endl;
  }
};{% endhighlight %}

方案一的实现相对起来就很直观，而如果用方案二的话，就需要另外写一个`Functor`，对于没有Lambda的C++来说，it’s painful……

这样在用这个类的时候我就可以简单的写下面的代码：

{% highlight cpp linenos %}
{
  HelloWorldThread t; // 线程开始执行
} // 线程退出{% endhighlight %}

注意，我期望在block退出的时候，这个线程自动的结束。

哦活活~~理想很丰满，现实很骨感！！方案一中的这种实现是有bug的。
如果你写一个单元测试，比如说像下面这样：

{% highlight cpp linenos %}
{
  for (int i = 0; i < 10; ++i)
    HelloWorldThread t;
}{% endhighlight %}

你会发现，在跑这个程序的大多数情况下，程序跑着跑着就crash了，Linux底下给你一个”pure virtual method called”的错误……
OMG，怎么回事？

这里就需要注意到，方案一中的实现，默认是joinable的线程。而我们在`Thread`类中的析构函数里面去`pthread_join`这个线程，从而保证这个线程在出作用域的时候会结束。
而既然”pure virtual method called”，那出问题的地方肯定是`t->Entry();`这一行咯。只有这一行call了虚函数嘛。
但是我们明明在子类中override了`Entry`函数啊！况且我调`Entry()`的时候的确是通过`HelloWorldThread`去调的啊！！

请仔细想想，调`Entry()`的时候可不一定是`HelloWorldThread`啊。
`static void* ThreadFunc(void* arg)`这个函数是在另一个线程里面执行的。而`pthread_join`这个函数是在`Thread`类的析构函数里面call，所以析构函数和`ThreadFunc`是不在同一个线程的。
我们案件重播一下，当我们启动线程之后，假设这个线程没有跑，这个时候我们来到了右大括号。此时`HelloWorldThread`的析构函数调用，为空，OK，这个时候继续调用父类的析构函数，这个时候就join，然后等待线程结束。注意到，在父类的析构函数里面，这个类就已经不再是`HelloWorldThread`了，他已经是`Thread`了。而`Thread`的`Entry`函数是纯虚的，如果线程现在开始运行的话，那么就会调用`Thread`的`Entry`函数（因为这个时候的对象是`Thread`类），Bang!! 悲剧总是这么发生的……
所以说，方案一中的实现是有问题的，至少用户不能利用RAII来进行线程的自动回收。所以基于这种实现的线程类，都必须由用户手动的去Join/Wait一下，否则就crash了。至少在目前我看过的实现中，wx的就是这么实现的，而它要求用户在joinable状态里面去主动的Wait一下线程。我觉得这样的实现不太clean，因为一旦你要求用户手动的做一些事情，就容易出现bug。而C++中的重要特性RAII就等于废了，所以我觉得方案二的实现较为好，虽然使用上有点不太习惯，不过习惯嘛，可以慢慢改。:)

# References

1. [http://stackoverflow.com/questions/3160403/pure-virtual-method-called-when-implementing-a-boostthread-wrapper-interface](http://stackoverflow.com/questions/3160403/pure-virtual-method-called-when-implementing-a-boostthread-wrapper-interface)
