---
layout: post
title: "线程池库Tpool实现笔记(4)"
description: ""
category: multi-threaded
tags: [tpool, async]
---
{% include JB/setup %}

上一节已经实现好了工作者线程，而这一节就会实现用户最为关心的任务。

线程池封装了线程的实现细节，只对用户暴露了添加任务和控制生命周期的接口。所以对于用户来说，只需要把想要完成的事情封装在一个任务里面然后交给线程池就可以了，剩下的事情就交给线程池来处理。

所以任务只需要定义某种接口，然后让用户自己定义所需的任务类型就可以了。需要注意的是任务只定义了接口，而没有实现具体的线程安全性，也就是如果在多个线程池执行的任务里面使用了共享的资源的话，需要任务自己去保证线程安全。

除此之外，任务还需要定义一些基本的生命周期管理方法，使得当任务执行时间过长的情况下可以中止任务的执行。之前的`WorkerThread`就已经提到了任务的中止。

Tpool中的任务定义如下：

{% highlight cpp linenos=table %}
class TaskBase {
public:
	enum State {
	  INIT,
	  RUNNING,
	  FINISHED,
	  CANCELLED,
	};

	TaskBase();
	~TaskBase() {}

	void Run();
	void Cancel();
	void CancelAsync();

	State GetState() const;

protected:
	void CheckCancellation() const;

private:
	virtual void DoRun() = 0;
};{% endhighlight %}

其中定义的`Run`函数是用户最为关系的调用接口，用户必须重写`DoRun`函数，然后把他加到线程池里面就可以让任务正常的运行了。

比如说可以创建一个这样的任务：

{% highlight cpp linenos=table %}
struct FakeTask : public TaskBase {
	virtual void DoRun()
	{
	  sleep(2);
	}
};{% endhighlight %}

然后用下面的语句把Task加到线程池里面：

{% highlight cpp linenos=table %}
LFixedThreadPool threadPool;
threadPool.AddTask(TaskBase::Ptr(new FakeTask));{% endhighlight %}

然后任务就会被执行。

有了这个基本的接口之后，我们还需要考虑一下怎么取消任务。比如说有一个任务是向某个URL取大量数据，如果当时的网络环境不好，则这个任务可能会执行很长的时间，如果这个时候任务队列里面有大量这样的任务，则工作者线程会被这些任务阻塞住，从而影响线程池的效率。要防止这种情况发生，可以使用一种类似于工作者线程那样的方式来实现取消机制。

工作者线程是通过查询一个flag的状态来判断退出与否的。任务也是使用了这种方式。通过查询一个退出标志位，任务判断是否该取消，如果取消，则抛出退出异常。但是在任务里面，我们没有办法预先写好在什么时候进行判断，所以这个判断的时机就交给用户在实现任务的时候来决定。而任务只是提供了一个函数来check这个事情。这个函数在Tpool里面叫做`CheckCancellation`，定义如下：

{% highlight cpp linenos=table %}
void TaskBase::CheckCancellation() const
{
  if (m_isRequestCancel)
	{
	  throw TaskCancelException("cancel task");
	}
}{% endhighlight %}

而用户在实现任务的时候就需要保证隔一段时间就去check一下，比如：

{% highlight cpp linenos=table %}
struct FakeTask : public TaskBase {
	virtual void DoRun()
	{
	  for (int i = ; i < 1000; %2B%2Bi)
	  {
		CheckCancellation();
		sleep(2);  // 模拟一个耗时操作
	  }
	}
};{% endhighlight %}

而在`Run`函数里面，不是简单的去调用`DoRun`，而是先检查`CheckCancellation`一下，这样就可以在任务没有跑的情况下也能取消的效果。如下：

{% highlight cpp linenos=table %}
void TaskBase::Run()
{
  try
	{
	  CheckCancellation(); // check before running the task.

	  SetState(RUNNING);
	  DoRun();
	  SetState(FINISHED);
	}
  catch (const TaskCancelException&)
	{
	  SetState(CANCELLED);
	}

  // wake up the waiting thread it is cancelling this task.
  ConditionNotifyLocker(m_cancelCondition,
			boost::bind(&TaskBase::IsStopState, this));
}{% endhighlight %}

而其余的`Cancel`函数只需要去设置一下取消flag就可以了。

至此，一个线程池所需要的主要元素都已经基本实现完毕了。接下来的几节我会讲述实现这个库用到的一些工具类实现和一些测试多线程程序的经验。
