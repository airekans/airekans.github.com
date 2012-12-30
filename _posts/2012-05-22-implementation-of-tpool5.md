---
layout: post
title: "线程池库Tpool实现笔记(5)"
description: ""
category: multi-threaded
tags: [tpool, async]
---
{% include JB/setup %}

在实现Tpool的过程中，除了主要的几个类——线程池、任务队列、任务、工作者线程之外，还需要一些辅助的工具类，主要有下面几个。

# Mutex

mutex(互斥锁)是用来实现多线程同步的主要机制之一。而Linux里面的C接口用起来多少有点不方便(对于C++程序员)，因为C++里面一般都会用[RAII][11]来实现资源的自动管理，否则管理成本会比较高。

在C++等支持RAII机制的语言里面，一般是写好获取和释放资源的函数，然后程序自动在某个上下文就帮你释放资源了。目前大多数的多线程库都是利用了RAII的技术来对C接口做一层封装，比如说boost::thread和wx。

在Tpool中也是类似，Mutex的定义如下：

{% highlight cpp linenos %}
class Mutex : private boost::noncopyable {
    friend class MutexLocker;
    friend class MutexWaitLocker;

public:
    Mutex();
    ~Mutex();

private:
    // These two functions can only called by MutexLocker
    void Lock();
    void Unlock();
	
    pthread_mutex_t m_mutex;
};{% endhighlight %}

注意到我将Lock和Unlock函数都设置为private，不对外部暴露，因为我觉得如果接口以暴露，程序员总有一种冲动去使用它，所以我在设计这个库的时候就是秉着尽量不让用户干坏事的原则来设计的。但是变成private之后有一个问题就是和他紧密相关的类也访问不了这些函数了，暂时我的解决方法是用friend来处理这个问题。

然后用的时候通过一个Locker来自动的把Mutex加锁和解锁：

{% highlight cpp linenos %}
MutexLocker::MutexLocker(Mutex& m)
  : m_mutex(m)
{
  m_mutex.Lock();
}

MutexLocker::~MutexLocker()
{
  m_mutex.Unlock();
}{% endhighlight %}

# ConditionVariable

条件变量是实现同步的重要手段。比如在实现任务队列的时候，假设当前的消费者线程想空的队列取任务的话，其中一种实现就是让线程block在那，然后等队列非空的事后再唤醒线程。

上面的等待，一个经典的实现就是通过条件变量来实现。在pthread里面有C的条件变量接口，而我在Tpool里面对其进行了简单的封装。

由于条件变量是与一个互斥锁联系起来的，所以我实现上要求在构造条件变量的时候就要传入一个Mutex。定义如下：

{% highlight cpp linenos %}
class ConditionVariable : private boost::noncopyable {
    friend class ConditionWaitLocker;
    friend class ConditionNotifyLocker;
    friend class ConditionNotifyAllLocker;

public:
    explicit ConditionVariable(Mutex& m);
    ~ConditionVariable();

private:
    void Notify();
    void NotifyAll();
    void Wait();
    void Lock();
    void Unlock();

    Mutex& m_mutex;
    pthread_cond_t m_cond;
};{% endhighlight %}

有定义可以看出，用户必须保证在`ConditionVariable`的生命周期内，Mutex必须一直有效(也就是Mutex的生命周期必须>=ConditionVariable的生命周期)。  
其中最重要的函数就是`Notify`和`NotifyAll`，分别是对`pthread_cond_signal`和`pthread_cond_broadcast`的简单封装。

而对于等待和唤醒，在UNPv2[1]里面有介绍过有几个经典的模式。而我在这里将这几个模式通过类的形式实现，从而减少用户出错的可能。

对于等待，是通过`ConditionWaitLocker`来实现的，用法如下：

{% highlight cpp linenos %}
{
  sync::ConditionNotifyLocker l(cond, NotifyFunc());
  WAIT_CONDITION = false; // 设置条件为true
}{% endhighlight %}

而唤醒是通过`ConditionNotifyLocker`和`ConditionNotifyAllLocker`来使用，用法如下：

{% highlight cpp linenos %}
{
  sync::ConditionNotifyLocker l(condition, NotifyFunc());
  WAIT_CONDITION = false; // 设置条件为true
}{% endhighlight %}

# References

1.  《Unix Network Programming Vol.2》


[11]: http://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization "Resource Acquisition Is Initialization"
