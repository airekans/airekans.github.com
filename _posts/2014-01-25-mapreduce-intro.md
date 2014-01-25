---
layout: post
title: "MapReduce简介"
description: ""
category: cloud-computing
tags: [distributed system, mapreduce]
---
{% include JB/setup %}

# 什么是MapReduce？

自从Google公开了他的`MapReduce`框架之后，`MapReduce`这个单词就一直频繁的出现。
但是到底什么是`MapReduce`呢？

`MapReduce`严格来说是一种编程的范式，这种范式是从函数式编程里面的`map`和`reduce`函数演化来的。
而不同语言和不同公司都有对于`MapReduce`都有的不同实现，比如Google的`MapReduce`、Apache的`Hadoop`。
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
