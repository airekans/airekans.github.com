# liburcu，一个用户态的RCU实现

----

在上一篇RCU的介绍里面，我们基本了解了RCU是如何实现Reader无锁的。
而由于RCU最开始是从Linux kernel里面实现的，kernel里面的实现非常依赖于整个内核的运行机制（比如Scheduler，软中断等），所以要把它port出来在用户态使用的话，难度并不小。
所幸目前已经有个开源的Userspace RCU实现——[liburcu][1]，不单只实现了RCU算法，而且有几种实现方案，从侵入式的到非侵入式的。而且这个库已经在比较多的项目中用到，比如比较出名的[LTTng][2]。

liburcu提供了以下几种RCU实现：

1. rcu-qsbr：性能最好的RCU实现，可以做到reader 0 zerohead，但是需要改动代码，侵入式。
2. rcu-signal：性能仅次于qsbr的实现，不需要改动代码，代价是需要牺牲一个signal给urcu实现。
3. rcu-generic：性能最差的rcu实现（但也比mutex强多了），不需要改动代码，可以作为初始的第一选择。

本文会详细剖析qsbr（quiescent-state-based RCU）的实现。

# 一个例子



  [1]: http://liburcu.org/
  [2]: http://lttng.org/
