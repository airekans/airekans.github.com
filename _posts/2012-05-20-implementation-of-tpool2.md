---
layout: post
title: "线程池库Tpool实现笔记(2)"
description: ""
category: multi-threaded
tags: [tpool, async]
---
{% include JB/setup %}

在上一节中，介绍了线程池的主要概念和[Tpool][12]的主要对外接口。并且在之前也讲过实现线程封装的一些方案。在Tpool里面，我选择的是类似于boost::thread的实现方式，也就是线程类是通过接受一个functor来指定线程的执行方式的。

在这一节里，我会讲述线程池里面最重要的数据结构——任务队列在Tpool中的实现。

# 什么是任务队列(TaskQueue)？

所谓的任务队列，就是线程池用来存放用户发送过来的任务的一个数据结构，这些任务会在之后以某种顺序被工作者线程取出并执行。

在Tpool中，定义了一个抽象的`TaskQueueBase`接口，定义如下：

{% highlight cpp linenos %}
namespace tpool {
  class TaskQueueBase {
  public:
	typedef boost::shared_ptr Ptr;

	virtual ~TaskQueueBase() {}

	virtual void Push(TaskBase::Ptr task) = ;
	virtual TaskBase::Ptr Pop() = ;
	virtual size_t Size() const = ;
  };
}{% endhighlight %}

所有实现的任务队列都必须遵守这个接口。其中`Push`是往这个队列中加入任务，`Pop`则是从队列中取出任务。

实现这个接口的队列都会以某种方式存取任务。一般任务队列都会实现为FIFO式的队列。在Tpool中有一个默认的实现`LinearTaskQueue`，就是一个无界的FIFO队列。当然也可以实现一个具有任务优先级概念的任务队列，这个队列里的任务都具有优先级，而在Pop任务的时候总是获取优先级最高的任务。

# 实现LinearTaskQueue

`LinearTaskQueue`的声明如下：

{% highlight cpp linenos %}
namespace tpool {
  class LinearTaskQueue : public TaskQueueBase {
  public:
	virtual void Push(TaskBase::Ptr task);
	virtual TaskBase::Ptr Pop();
	virtual size_t Size() const;

  private:
	typedef std::queue TaskQueueImpl;
	TaskQueueImpl m_tasks;
	mutable sync::MutexConditionVariable m_mutexCond;
  };
}{% endhighlight %}

我用了`std::queue`来作为这个队列的内部实现，其中的`Push`和`Pop`操作怎么保证同步就是最为重要的地方。

因为在线程池里，很有可能同时有多个线程在同时向任务队列取任务，所以怎么保证取任务的正确性是很重要的。还有可能是在线程池往队列添加任务的同时工作者线程也在从队列取任务，这时候确保`Push`和`Pop`的同步也是非常重要的。

在队列同步的实现上，有以下几种实现：

1.  Single Lock: 用一把互斥锁锁住Push和Pop来保证操作的同步。
2.  Double Lock: 用两把锁分别锁住Push和Pop，使得读和写之间不存在互斥，从而提高了效率。
3.  Non-blocking: 完全不用锁的实现，目前Java的concurrent包里面就有一个NonBlockingQueue，使用的就是这种实现。

从效率上来说，1到3的实现是递增的，但是实现的难度也是递增的。在`LinearTaskQueue`里面用的是最简单的Single Lock实现。

首先可以看到我在`LinearTaskQueue`里面声明了一个[MutexConditionVariable][16]，这是一个绑定了Mutex的一个条件变量。如果不用条件变量而只用Mutex的话，需要在Pop的时候用Mutex来进行状态的轮询，因为如果Pop的时候队列为空，需要等待队列变为非空，这是非常没有效率的一种实现。而是用条件变量的话，可以避免使用轮询，而在队列为空的时候让线程等待并阻塞住，这样就可以提高效率。

下面是Push的实现：

{% highlight cpp linenos %}
void LinearTaskQueue::Push(TaskBase::Ptr task)
{
  ConditionNotifyAllLocker l(m_mutexCond,
				 bind(&TaskQueueImpl::empty, &m_tasks));
  m_tasks.push(task);
}{% endhighlight %}

上面的意思是先将`m_mutexCond`锁上，并且当队列为空的时候通知其他等待的线程。然后往队列里面添加任务。这种加锁 → 通知 → 设置状态的方式是一种典型的模式，在UNPv1\[1\]里面有详细的说明。

而`Pop`的实现如下：

{% highlight cpp linenos %}
TaskBase::Ptr LinearTaskQueue::Pop()
{
  // wait until task queue is not empty
  ConditionWaitLocker l(m_mutexCond,
			bind(&TaskQueueImpl::empty, &m_tasks));

  TaskBase::Ptr task = m_tasks.front();
  m_tasks.pop();
  return task;
}{% endhighlight %}

同样的，`Pop`里面也进行了下面几步：

1.  加锁，并且判断队列是否为空，如果为空，阻塞住。
2.  从队列里面取出任务，然后返回。

注意到在`Push`里面用的是`NotifyAll`而不是`Notify`，也就是在放入队列的时候，会通知所有的等待线程，而不是通知一个。有人就会问，通知所有的不会有性能问题么？用`Notify`不是也可以么？

对于第一个问题，暂时来说由于用的是一种指定执行顺序的唤醒模式，也就是：

1.  A线程加锁，唤醒等待线程B。
2.  B执行，在唤醒之后尝试加锁，但是由于锁被A获取，所以再次阻塞。
3.  A继续执行，设置状态为真，解锁。
4.  B被唤醒，加锁，然后执行接下来的操作，解锁。

所以执行顺序肯定是A → B，所以在A唤醒B这个过程中如果使用的是`NotifyAll`的话，会有多个线程同时尝试加锁，但是都会阻塞住，这个过程比较短，所以不会造成太大的性能开销。况且实现上我还是只有在队列为空的情况下才会去唤醒等待线程。

而对于第二个问题，答案是不能简单的用`Notify`来替换`NotifyAll`。想象一下下面这样的执行场景：

1.  队列为空，此时有两个线程在等待。
2.  此时另一个线程执行Push，这个过程唤醒了一个线程。
3.  假设这个被唤醒的线程还没有来得及被调度，这时另一个线程又调用了一次`Push`，注意，这个时候并不会执行`Notify`，因为我的`Notify`条件是当队列为空才会执行，而这个时候队列不为空。
4.  在这个情况下，本来应该是两个等待的线程都被唤醒，但是实际上只有一个线程被唤醒，而另一个线程则一直等在那里，没有人去唤醒他。

解决方法也很简单，就是把`Notify`的条件改成每次`Push`的时候都会`Notify`一次，不过这样的开销和用`NotifyAll`到底哪个大还需要看操作系统怎么实现了。

当然有一种最好的实现是是用类似于读写锁。把调用Pop的线程当做读者，而把调用`Push`的线程当做写者，并且把当前等待的读者数量记录下，而在这个数不为零的时候去`Notify`。

至此，一个基本的`TaskQueue`已经实现完毕了。

# References

1.  《Unix Network Programming, vol.1》


[12]: https://github.com/airekans/Tpool
[16]: https://github.com/airekans/Tpool/blob/master/include/ConditionVariable.h "ConditionVariable的实现"


