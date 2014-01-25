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
domains = map(lambda u: u.split('/')[0], urls) # We get all domains here

def get_domain_stat(stat, domain):
    if domain not in stat:
        stat[domain] = 0
    stat[domain] += 1
    return stat

domain_stat = reduce(get_domain_stat, domains, {}) # We get the stat of domains here
{% endhighlight %}

从上面的例子可以看到，通过`map`我们从url得到了所有的域名，
而通过`reduce`，我们得到了所有域名的统计。
而这里最主要的一点是，map是无状态的，而reduce的状态转变非常简单，
这也说明`map`和`reduce`要并行化非常简单(事实上reduce可以利用hash也做成无状态)。
我们可以根据需要，在`map`的实现里面开10个线程，或者是用分布式系统做成10个worker。
而`MapReduce`正是利用了这一点，把`map`和`reduce`做进了分布式系统。

