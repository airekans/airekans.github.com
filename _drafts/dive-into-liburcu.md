# liburcu，一个用户态的RCU实现

----

在上一篇RCU的介绍里面，我们基本了解了RCU是如何实现Reader无锁的。
而由于RCU最开始是从Linux kernel里面实现的，kernel里面的实现非常依赖于整个内核的运行机制（比如Scheduler，软中断等），所以要把它port出来在用户态使用的话，难度并不小。
所幸目前已经有个开源的Userspace RCU实现——[liburcu][1]，不单只实现了RCU算法，而且有几种实现方案，从侵入式的到非侵入式的。而且这个库已经在比较多的项目中用到，比如比较出名的[LTTng][2]。

liburcu提供了以下几种RCU实现：

1. rcu-qsbr：性能最好的RCU实现，可以做到reader 0 zerohead，但是需要改动代码，侵入式。
2. rcu-signal：性能仅次于qsbr的实现，不需要改动代码，代价是需要牺牲一个signal给urcu实现。
3. rcu-generic：性能最差的rcu实现（但也比mutex强多了），不需要改动代码，可以作为初始的第一选择。

本文会详细剖析qsbr（quiescent-state-based RCU）的实现。本文的代码来自liburcu，所以代码的协议和liburcu保持一致，使用LGPL协议。

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
   1. 线程开始的时候需要调用`rcu_register_thread()`进行注册，线程结束的时候需要调用`rcu_unregister_thread()`进行注销。
   2. 对于共享数据区的访问需要用`rcu_read_lock()`和`rcu_read_unlock()`来表示临界区。
   3. 对于共享数据的指针，需要用`rcu_dereference()`来获取。
   4. 线程时不时需要调用`rcu_quiescent_state()`来生命线程在quiescent state。
 - 对于写者
   1. 新的数据初始化需要在替换指针之前就完成。
   2. 指针替换需要调用`rcu_xchg_pointer()`来完成。
   3. 替换完数据之后，需要调用`synchronize_rcu()`来等待[Grace Period][3]的结束。
   4. 在`synchronize_rcu()`结束之后，我们就可以放心的删除旧数据了。

接下来我们来看看这些函数是怎么实现的。

# QSBR关键数据结构

在RCU里面，最核心的就是Grace Period了。在qsbr里面，Grace Period是用一个全局的`unsigned long`(64 bits)的counter——`rcu_gp`来表示。
每新开始一个Grace Period，就往这个counter上加一。所以这个数值我们可以称之为gp号。

而对于每个读线程，都会有一个`rcu_reader`结构，这个结构里面存着最近一次的gp号缓存，以及一些额外的数据。

```c
struct rcu_gp {
    unsigned long ctr;
    int32_t futex;
} __attribute__((aligned(CAA_CACHE_LINE_SIZE)));
extern struct rcu_gp rcu_gp;

struct rcu_reader {
    /* Data used by both reader and synchronize_rcu() */
    unsigned long ctr;
    struct cds_list_head node 
        __attribute__((aligned(CAA_CACHE_LINE_SIZE)));
    int waiting;
    pthread_t tid;
    unsigned int registered:1;
};
extern DECLARE_URCU_TLS(struct rcu_reader, rcu_reader);
```

在qsbr里面，`read_lock`和`read_unlock`都不会改变本线程的gp缓存，只有在`rcu_quiescent_state()`调用的时候，会从全局的`rcu_gp`里面获取最新的gp号，更新到本线程缓存。

当写线程执行到`synchronize_rcu()`的时候，实际上就会先把`rcu_gp`加一，然后等待所有的读线程的gp缓存都等于最新的gp号，然后才返回。这也就是qsbr实现的Grace Period机制。

# 读线程函数

接下来我们来看看对于读线程来说，几个关键函数是怎么实现的。

## 线程注册、注销

在qsbr里面，每一个读线程都需要调用`rcu_register_thread()`进行注册，否则写线程并不知道该读线程的存在。而在线程结束之前也必须调用`rcu_unregister_thread()`进行注销，否则会造成写线程死锁。

```c
DEFINE_URCU_TLS(struct rcu_reader, rcu_reader);
static CDS_LIST_HEAD(registry);
static pthread_mutex_t rcu_registry_lock = PTHREAD_MUTEX_INITIALIZER;

void rcu_register_thread(void) {
	URCU_TLS(rcu_reader).tid = pthread_self();
	assert(URCU_TLS(rcu_reader).ctr == 0);

	mutex_lock(&rcu_registry_lock);
	assert(!URCU_TLS(rcu_reader).registered);
	URCU_TLS(rcu_reader).registered = 1;
	cds_list_add(&URCU_TLS(rcu_reader).node, &registry);
	mutex_unlock(&rcu_registry_lock);
	_rcu_thread_online();
}

void rcu_unregister_thread(void) {
	_rcu_thread_offline();
	assert(URCU_TLS(rcu_reader).registered);
	URCU_TLS(rcu_reader).registered = 0;
	mutex_lock(&rcu_registry_lock);
	cds_list_del(&URCU_TLS(rcu_reader).node);
	mutex_unlock(&rcu_registry_lock);
}
```

上面的代码首先定义了TLS变量`rcu_reader`，使得每个读线程都有一个`rcu_reader`。然后定义一个双向链表`registry`，用来保存所有读线程的`rcu_reader`。这会在写线程的`synchronize_rcu()`用到。还有一个`mutex`来保护这个链表。

在`rcu_register_thread`里面，主要就是往这个链表里面加入本线程的`rcu_reader`。接着调用`_rcu_thread_online`来缓存最新的`rcu_gp`。
`rcu_unregister_thread`则做相反的事，先清除一些标记，然后把本线程的`rcu_reader`从链表里面删除。

这两个函数用到了锁，但是由于这两个函数只在线程的开始和结束才会调用，所以对性能基本没有影响。

## 读临界区

读线程在对公共数据做操作的时候，需要调用`rcu_read_lock`和`rcu_read_unlock`来标记临界区：

```c
inline void rcu_read_lock(void) {
	urcu_assert(URCU_TLS(rcu_reader).ctr);
}

inline void rcu_read_unlock(void) {
	urcu_assert(URCU_TLS(rcu_reader).ctr);
}
```

可以看到在这两个函数里面实际上什么都没有做，只是assert，说明在O2优化下这就是个空的函数，也就是




  [1]: http://liburcu.org/
  [2]: http://lttng.org/
  [3]: http://airekans.github.io/c/2016/04/23/rcu-intro#grace-period
