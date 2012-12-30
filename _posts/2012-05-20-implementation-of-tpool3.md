---
layout: post
title: "线程池库Tpool实现笔记(3)"
description: ""
category: multi-threaded
tags: [tpool, async]
---
{% include JB/setup %}

上一节我们已经实现了一个基本的任务队列了。而在这一节我会讲述工作者线程的实现。

Tpool的工作者线程使用了类似`boost::thread`实现的线程实现。

工作者线程应该实现下面几个功能点：

1.  工作者线程在执行的时候不断地从任务队列获取任务，一旦获取了任何，则执行它。当一个任务执行完之后，继续获取任务。
2.  支持工作者线程的生命周期管理，也就是可以让用户开始、结束工作者线程。

假设我们有下面这样一个[基本的线程][12]定义：

{% highlight cpp linenos %}
class Thread : private boost::noncopyable {
public:
	template
	explicit Thread(const Func& f);

	~Thread();

private:
	template
	static void* ThreadFunction(void* arg);

	pthread_t m_threadId;
	bool m_isStart;
};{% endhighlight %}

其中最重要的是构造函数是接受一个`Functor`，而这个`Functor`就是这个线程要执行的函数。而线程的析构函数里面则会去join这个线程，也就是这个线程默认是Joinable的。

有了上面的线程定义，很容易就会想到在创建这个Thread的时候将一个不断循环的从任务队列里面取任务的functor传递进去。

首先看一下`WorkerThread`的声明：

{% highlight cpp linenos %}
class WorkerThread {
private:
	enum State {
	  INIT,
	  RUNNING,
	  FINISHED,
	};

public:
	typedef boost::shared_ptr Ptr;

	WorkerThread(TaskQueueBase::Ptr taskQueue);
	template 
	WorkerThread(TaskQueueBase::Ptr taskQueue, FinishAction action);
	~WorkerThread();

	void Cancel();
	void CancelAsync();
	void CancelNow();
};{% endhighlight %}

目前的`WorkerThread`是设计成在构造函数里面就启动一个新的线程，而不是通过一个`Start`函数。而`Cancel`函数和其他几个变体都是为了完成线程的生命周期管理的。

而`WorkerThread`的定义如下：

{% highlight cpp linenos %}
template 
WorkerThread::WorkerThread(TaskQueueBase::Ptr taskQueue,
				 FinishAction action)
{
    using boost::bind;

    m_taskQueue = taskQueue;

    // ensure that the thread is created successfully.
    while (true)
    {
        try
        {
            // check for the creation exception
            m_thread.reset(new Thread(bind(&WorkerThread::
                                           ThreadFunction,
    									   this, action)));
            break;
        }
        catch (const std::exception& e)
        {
            ProcessError(e);
        }    
    }
}{% endhighlight %}

其中那个While循环是因为Thread在创建失败的时候会抛出异常，而我需要确保当`WorkerThread`的构造函数执行完的时候，线程已经被构造好。

而其中的`ThreadFunction`就是线程函数，定义如下：

{% highlight cpp linenos %}
template 
void WorkerThread::ThreadFunction(FinishAction action)
{
  WorkFunction();
  action(); // WorkerThread finished.
  NotifyFinished();
}{% endhighlight %}

这个线程是首先执行`WorkFunction`，然后执行一个用户传递进来的functor，这个functor是用户希望在线程结束之后能够执行的某个行为。最后再通知一下等待`WorkerThread`结束的线程。

最重要的就是`WorkFunction`。定义如下：

{% highlight cpp linenos %}
void WorkerThread::WorkFunction()
{
    SetState(RUNNING);
    while (true)
    {
		try
		{
			// 1. check cancel request
			CheckCancellation();

			// 2. fetch task from task queue
			GetTaskFromTaskQueue();

			// 2.5. check cancel request again
			CheckCancellation();

			// 3. perform the task
			if (m_runningTask)
			{
				if (dynamic_cast(m_runningTask.get()) != NULL)
				{
					break; // stop the worker thread.
				}
				else
				{
					m_runningTask->Run();
				}
			}
			// 4. perform any post-task action
		}
		catch (const WorkerThreadExitException&)
		{
			// stop the worker thread.
			break;
		}
		catch (...) // caught other exception
		{
			// continue
		}
    }
}{% endhighlight %}

可以看到这里`WorkFunction`就是用了一个While循环来不断的从任务队列里面取任务，然后执行。同时会判断拿出来的任务是不是类型为`EndTask`的任务，如果是，就意味着用户要求结束工作者线程，函数可以结束执行了。

`WorkFunction`的基本思想还是比较简单的。但是除了任务的执行之外，还需要支持生命周期管理。假设一下当执行任务到一半，用户想要中止工作者线程的执行，这个时候如何去停止就是一个很重要的考虑了。如果是通过向任务队列里面添加`EndTask`这种缓和的方式，如果在多个工作者线程共享一个任务队列的时候，很难确保工作者线程可以马上中止，因为也许队列中会有其他的任务排在`EndTask`前面。

为了让用户(主要是线程池)能对工作者线程有更加细粒度的生命周期控制，我将中止的类型做了以下几种区分：

1.  线程池中止：整个线程池中止执行，此时线程池不再接受新的任务请求，同时往任务队列添加`EndTask`，使得工作者线程可以在执行完其他任务之后结束执行。这种中止方式是最缓和，也是最保险的。
2.  工作者线程非紧急中止：这种方式要求工作者不再取新的任务，并且在执行完当前正在执行的任务之后就结束执行。
3.  工作者线程紧急中止：类似于前一种方式，但是会尝试直接中止当前正在执行的任务并中止线程。

除了第一种方式之外，其他两种中止都比较复杂。

初看起来，也许会觉得直接用`pthread_cancel`就可以实现类似的功能了。但是不要忘记，`pthread_cancel`是非常危险的一种线程取消机制，无论是async模式还是defered模式的，稍微不小心就会导致死锁的出现。

为了避免这种糟糕的实现，必须在线程之上自己实现一种线程取消的机制，使得线程可以安全的退出。

从线程函数的角度看，因为工作者线程主要是一个While循环执行任务的模式，就可以采用一种查询flag然后退出循环的方式来实现退出机制。这里主要有两个问题：

1.  什么时候查询flag。
2.  怎么退出循环。

# 什么时候查询flag？

回去看到`WorkFunction`的实现，可以看到我在获取新的任务之前和获取了任务之后都查询了一次flag，如果flag被设置了，那么就退出。为什么要在这两个时候呢？

首先任务的运行途中，任务的退出是归任务自己管的，这个在Task的实现里面会有，而工作者线程不负责。而在其余的时候就应该尽可能的去检查flag，从而提高响应性。

第一个check会在工作者线程第一次进入循环或者是执行完任务的时候进行判断，而第二个check是在工作者线程获取完任务的时候，因为线程有可能在获取任务的时候阻塞住，所以这个时候检查也是必须的。

# 怎么退出循环？

一般的程序语言里面，可以有以下几种退出控制的方式(不管程序现在嵌套了多少个程序栈)：

1.  函数的返回值表示某种退出状态，然后在没有函数调用的地方都去check一下返回值，根据具体的值去返回或者是继续执行。这种方式对于程序员来说非常的繁琐。
2.  goto语句，当发生了某种情况之后，就用goto语句跳转到处理错误的逻辑那里，而不管现在是在哪个地方。
3.  C++里面的异常：通过异常，不论程序运行了多少个嵌套的函数，都可以在抛出异常之后，跳转到对应的异常处理代码段。当然实际上异常的实现可能也就是某种程度上的goto。

实现上的难度来说，在C++里面用异常来实现退出机制是最方便的一种方式，当然C++的异常机制有很多defects，但是只要小范围里面小心的运用，还是可以放心的用的。

所以我在`CheckCancellation`里面抛出一个异常，然后在While循环里面去catch这个异常就可以达到退出的目的了。

至此，就已经实现好了一个简单可用的工作者线程了。


[12]: https://github.com/airekans/Tpool/blob/master/include/Thread.h "Tpool::Thread"
