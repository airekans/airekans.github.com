---
layout: post
title: "MapReduce简介"
description: "讲述了MapReduce的原理，用Python来讲述了一个统计url的例子。最后讲了MapReduce面临的挑战和一些解决方法。"
category: cloud-computing
tags: [distributed system, mapreduce]
---
{% include JB/setup %}

# 什么是MapReduce？

自从Google公开了他的`MapReduce`框架之后，`MapReduce`这个单词就一直频繁的出现。
但是到底什么是`MapReduce`呢？

`MapReduce`严格来说是一种编程的范式，这种范式是从函数式编程里面的`map`和`reduce`函数演化来的。
而不同语言和不同公司都有对于`MapReduce`都有的不同实现，
比如[Google的MapReduce](http://research.google.com/archive/mapreduce.html)、
[Apache的Hadoop](http://hadoop.apache.org/)。
所以从这种角度来说，`MapReduce`也是一种框架。

## 一个简单例子

先让我们来看看`MapReduce`是怎么用的。假设有10亿个url，而我们想统计出总共有多少个域名，
每个域名出现了多少次。下面我用Python的`map`和`reduce`写下计算的流程。
为了简单起见，我们建设url都不以`http://`开头，并且都是`weibo.com/airekans`这种格式。

{% highlight python linenos=table %}
urls = [url1, url2, ... ]
# We get all domains here
domains = map(lambda u: u.split('/')[0], urls)

def get_domain_stat(stat, domain):
    if domain not in stat:
        stat[domain] = 0
    stat[domain] += 1
    return stat

# We get the stat of domains here
domain_stat = reduce(get_domain_stat, domains, {})
{% endhighlight %}

从上面的例子可以看到，通过`map`我们从url得到了所有的域名，
而通过`reduce`，我们得到了所有域名的统计。
而这里最主要的一点是，map是无状态的，而reduce的状态转变非常简单，
这也说明`map`和`reduce`要并行化非常简单(事实上reduce可以利用hash也做成无状态)。
我们可以根据需要，在`map`的实现里面开10个线程，或者是用分布式系统做成10个worker。
而`MapReduce`正是利用了这一点，把`map`和`reduce`做进了分布式系统。

## 利用MapReduce重写

`MapReduce`实际上就是定义了两个接口：`Map`和`Reduce`。用户只需要提供Map函数用以转化输入得到中间结果，
和`Reduce`函数用从中间结果转化到结果。而当用户指定了输入之后，就可以很简单的通过参数指定`Map`和`Reduce`
的并行数量，而`MapReduce`则帮你搞定了分布式任务调度分发和提供高可靠性。

这里我用假想的一个Python `MapReduce`框架来说明一下如果写`Map`和`Reduce`(说不定之后我会真的写一个，这里先挖个坑)。
假设我们的输入的10亿个url都保存在`urls.txt`文件，而每一行包含一个url。下面是定义的`MyMap`和`MyReduce`函数。

{% highlight python linenos=table %}
def MyMap(input, output):
    domain = input.Value().split('/')
    output.OutputWithKey(domain, '')
    
def MyReduce(input, output):
    domain_stat = 0
    domain = input.Key()
    for v in input.Value():
        domain_stat += 1
    output.Output('%s %d' % (domain, domain_stat))
{% endhighlight %}

从上面可以看到，函数的输入都用`input`表示，输出都用`output`来表示。
其中`MyMap`里的`input.Value()`获取输入文件中的一行，`output.OutputWithKey`是以
第一个参数为key，第二个参数为value的输出。
而`MyReduce`的`input`是对应的，而输出则是用`output.Output`直接输出一行。

有了上面的代码，我们就可以用下面的命令启动这个`MapReduce`程序，
其中指定了`Map`的数量为100和`Reduce`的量为50。

    $ mapreduce --input=/path/to/urls.txt --mapper=MyMap --reducer=MyReduce
        --mapper-num=100 --reducer-num=50 --output=/path/to/output.txt

# MapReduce需要解决什么问题？

看了上面的例子，也许有人会问，这么简单的事情，貌似并不需要用`MapReduce`？
其实如果尝试过处理大树据量，比如上G甚至上T的数据的时候，
这个时候单机的处理速度就会非常慢，甚至是以天为单位的。
但是如果利用`MapReduce`进行并行化，则整个处理数度就会降低非常多，
降低到小时级甚至是分钟级别的。

所以`MapReduce`主要是用来进行一些大树据量的处理，而且处理过程能够用`MapReduce`范式
进行较为简单的描述的过程，比如说搜索中的网页索引处理、或者是一些存储数据的统计等。

既然`MapReduce`为我们提供了一个这么易用的分布式框架，那么它自身又面临一些什么样的挑战呢？
简单来说有下面几种问题(在Google的`MapReduce`论文里面也有描述，这里只是在我自己的理解上再阐述一遍)：

1. 整体架构：如何分布式的处理`Map`和`Reduce`？如何分发任务？对于这个问题，常见的实现是利用经典的
    一主多从结构，也就是一个Master负责任务的调度和分发，还有一些状态的维护也放在Master上。
    这样设计的优点是状态的维护很简单，一个Master的状态可以省去多主的一些状态不一致。
2. 数据如何流动：从最简单的模型来看，应该是数据先从本地到`Mapper`，然后再到`Reducer`。
    中间的数据是如何流动比较有效呢？还是说有更有效的方式？比如用NFS，或者是类似的方案，
    比如说Google的GFS或者是Hadoop的HDFS？
3. 高可靠性：可靠性是每一个分布式系统都需要考虑的问题，其中在这种主从结构的系统里面，
    可靠性就包括两方面：Master的可靠性和Slave的可靠性。在Google的`MapReduce`实现中，
    Master可靠性是考集群管理系统的自动拉起及Checkpoint机制来实现的。
    而Slave可靠性是也主要是靠checkpoint来做的，Master会检查Slave的健康情况，
    调整任务的调度。而Google的`MapReduce`对于不同级别的Master/Slave失败都定义了对应的处理措施。
4. 任务调度：既然`MapReduce`的Slave是要进行`Map`和`Reduce`的操作，而这些任务都是由Master分发的，
    那么Master如何调度任务则又是一个很重要的问题。在任务调度中，最重要的几个点包括负载均衡、
    输入局部性(locality)和Slave失败的处理。其中locality是最重要的一点，locality说的是，
    在分派任务时，着重考虑下发的任务的输入是否和任务本身处在同一台机器上。因为如果是一台机器，
    则任务的处理速度相比不同机器的环境，延时要低很多。这个概念和我们写本地程度是一样的。
5. Straggler处理：在任务处理中，在最后的阶段，往往有几个任务，在slave上面跑，但是耗时却很长，
    从而延长了整个`MapReduce`的执行时长。在Google的paper中，称这几个任务为straggler。
    而对于这种任务的处理，可以通过下发straggler到多个worker中进行执行，先执行完的则标识整个
    `MapReduce`执行完。这是因为straggler执行慢，往往是因为执行任务的slave，网络、磁盘、内存等
    出了问题。通过这种多slave执行，可以避免这个问题。
6. Value的排序：注意到在上面的url例子里面，顺序对于我们来说貌似不是太重要，但是如果我们
    就是想做一个分发的排序呢？貌似用`MapReduce`的模型解决不了啊？在这里，Google的论文中
    则给出了答案，就是默认对同一个`Reducer`的输入进行排序。这样当我们相对结果进行某种排序的时候，
    会方便非常多。而在Hadoop中，这个排序的过程叫做_Shuffle_。Shuffle意味着从不同的`Mapper`
    拿到对应`Reducer`的结果，同时进行排序的过程。在后续的文章中，我会对Shuffle的过程进行讲述。
7. 坏记录的处理：在代码没有写好的情况下，在`Mapper`或`Reducer`遇到特定的输入时会crash。
    但是因为这些记录而导致整个`MapReduce`没有办法跑下去通常是不合理，也就是说忽略这些坏记录
    是一种更好的做法。
8. 状态的实时监控：因为`MapReduce`执行时间通常是数十分钟或者是几小时，这个时候如果能够通过
    某些接口查询整个`MapReduce`的状态是非常方便的。通常提供一个Http服务器或者类似的Web API
    提供给用户查询，就可以达到目的了。

上面的几个问题，都是作为一个高可靠的`MapReduce`系统需要面临和解决的。在开源的Hadoop里面，
我们能够看到对应的解决方案。而在大数据处理方面，除了`MapReduce`解决的计算问题之外，
还有数据如何存储的问题，这也就是Google剩下的两大法宝`GFS`和`Bigtable`所要解决的问题。
除了这些之外，整个集群如何管理，机器资源如何分配也是需要解决的，这方面Google有`Borg`(未开源)，
Hadoop里面有`Yarn`，而Twitter也有`Mesos`。在后面我还会这几块进行一些深入的讲解。

最后，在这里用Google的Paper里面给出的`MapReduce`架构图让大家了解一下整个`MapReduce`的宏观结构。
(图片本身引用CSDN)。

![MapReduce Architecture](http://img.my.csdn.net/uploads/201204/26/1335443612_8438.jpg)


