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

假设我们有下面这个需求：

> 一个全局的gp指针指向一个结构体，有几个读线程不断的读这个结构体里面的数据，求和。
> 这个结构体可能在某些时刻被一个写线程更新。

用liburcu的qsbr实现的话，会是下面这样的代码：

```cpp
struct Foo { int a, b, c, d; };

void ReadThreadFunc() {
    struct Foo* foo = NULL;
    int sum = 0;
    rcu_register_thread();
    for (int i = 0; i < 100000000; ++i) {
        for (int j = 0; j < 1000; ++j) {
            rcu_read_lock();
            foo = rcu_dereference(gs_foo);
            if (foo) {
                sum += foo->a + foo->b + foo->c + foo->d;
            }
            rcu_read_unlock();
        }
        rcu_quiescent_state();
    }
    rcu_unregister_thread();
}

void WriteThreadFunc() {
    while (!gs_is_end) {
        for (int i = 0; i < 1000; ++i) {
            struct Foo* foo =
                (struct Foo*) malloc(sizeof(struct Foo));
            foo->a = 2; foo->b = 3; 
            foo->c = 4; foo->d = 5;
            rcu_xchg_pointer(&gs_foo, foo);
            synchronize_rcu();
            if (foo) {
                free(foo);
            }
        }
    }
}
```

这里可以看到几个关键点：

 - 对于读者
   1. 线程开始的时候需要调用`rcu_register_thread()`进行注册。
   2. 线程结束的时候需要调用`rcu_unregister_thread()`进行注销。
   3. 对于共享数据区的访问需要用`rcu_read_lock()`和`rcu_read_unlock()`来表示临界区。
   4. 对于共享数据的指针，需要用`rcu_dereference()`来获取。
   5. 线程时不时需要调用`rcu_quiescent_state()`来生命线程在quiescent state。
 - 对于写者
   1. 新的数据初始化需要在替换指针之前就完成。
   2. 指针替换需要调用`rcu_xchg_pointer()`来完成。
   3. 替换完数据之后，需要调用`synchronize_rcu()`来等待[Grace Period][3]的结束。
   4. 在`synchronize_rcu()`结束之后，我们就可以放心的删除旧数据了。

接下来我们来看看这些函数是怎么实现的。

# QSBR关键数据结构

在RCU里面，最核心的就是Grace Period了。在qsbr里面，Grace Period是用一个全局的`unsigned long`(64 bits)的counter来表示。
每新开始一个Grace Period，就往这个counter上加一。所以这个数值我们可以称之为gp号。

而对于每个读线，都会有一个`rcu_reader`结构，这个结构里面存着最近一次的gp号
   

  [1]: http://liburcu.org/
  [2]: http://lttng.org/
  [3]: http://airekans.github.io/c/2016/04/23/rcu-intro#grace-period
