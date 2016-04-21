# Read-Copy Update，向无锁编程进发！

----

在无锁编程的世界里，ABA问题是一个没有办法回避的实现问题。就看看实现一个最简单的[基于单链表的stack都有这么多的坑][1]，就知道无锁编程有多难。
难道我们追求高性能的道路就被这个拦路虎挡住了？
No，我们有Read-Copy Update（RCU）这个法宝，帮助我们方便的实现很多的无锁算法数据结构。

本文会首先简略介绍RCU的基本概念，然后讲解liburcu里面的qsrb实现，让读者可以了解RCU的实现细节。

# 什么是RCU？

引用一下这篇[著名的RCU科普文][2]的开头：

> Read-copy update (RCU) is a synchronization mechanism that was added to the Linux kernel in October of 2002. RCU achieves scalability improvements by allowing reads to occur concurrently with updates.

首先，RCU是一种同步机制；其次RCU实现了读写的并行；最后，2002开始被Linux kernel所使用。
RCU利用一种Publish-Subscribe的机制，在Writer端增加一定负担，使得Reader端几乎可以**Zero-overhead**。

RCU适合用于同步基于指针实现的数据结构（例如链表，哈希表等），同时由于他的Reader 0 overhead的特性，特别适用用读操作远远大与写操作的场景。例如在Linux内核中的routing模块（与DNS非常相关）则用到来RCU来实现高性能。

## 一个链表的例子

假设我们有下面这个结构定义：

```c
struct foo {
    int a;
    int b;
    int c;
};
struct foo *gp = NULL;
```

那么在不考虑`gp`这个指针会被改变的情况下，我们可以这样的去进行读操作：

```c
struct foo* p = gp;
if (p != NULL) {
    do_something_with(p->a, p->b, p->c);
}
```

一切看起来都很简单。如果我们现在有一个写者是像下面这样去改变`gp`的呢？

```c
struct foo* p = kmalloc(sizeof(*p), GFP_KERNEL);
struct foo* tmp_gp = gp;
p->a = 1;
p->b = 2;
p->c = 3;
gp = p;
free(tmp_gp);
```

读者们知道会发生什么吗？



  [1]: https://en.wikipedia.org/wiki/ABA_problem#Examples
  [2]: http://lwn.net/Articles/262464/
