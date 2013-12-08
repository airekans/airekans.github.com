---
layout: post
title: "Non blocking Queue的实现"
description: ""
category: multi-threaded
tags: [async]
---
{% include JB/setup %}

之前在实现Tpool的时候就实现过一个用pthread\_cond\_signal/wait的BlockingQueue。而在多线程程序里面，用到队列的地方无数，对队列的并发要求也各不相同。实现一个简单点的线程池，在吞吐量不高的情况下用BlockingQueue还是没有什么问题的。但是在吞吐量大的情况下，用锁实现的Queue会因为加锁/解锁的开销成为性能瓶颈。

为了解决这个问题，就出现了Lock-free的队列实现，也称为Non Blocking Queue。本文主要讲解实现算法的一些细节。

# 并发队列的实现形式

并发队列在实现上，一般有下面几种：

1.  Single Lock队列：用一把锁，锁住队列的Enqueue和Dequeue操作。
2.  Double Lock队列：用两把锁分别锁住Enqueue和Dequeue操作。
3.  Lock Free队列(Non Blocking Queue)：完全不用锁来进行Enqueue和Dequeue的同步。

可以看到，对于Single Lock来说，只要线程数量多了，Enqueue和Dequeue操作数量一上去，那么这个锁就会成为了瓶颈。

Double Lock则解决了一部分问题，使得Enqueue和Dequeue的锁分开，只会在多个Enqueue和多个Dequeue之间产生互斥。则使得在Enqueue和Dequeue的速率相差不大的情况下，吞吐量会提高不少。

但是Double Lock仍然在入队和出队操作本身之间存在着互斥，在多个消费者之间仍然会有瓶颈。

Lock free则完全将这些互斥减到最小的程度。

# Non Blocking Queue的实现

在实现上，Non Blocking Queue的数据结构的实现是和Double Lock的实现相同的，可以参照[冠诚的文章][12]去了解一下。

粗略的展示一下实现代码：

    
{% highlight cpp linenos=table %}
typedef struct node_t {
	TYPE value;
	node_t *next
} NODE;

typedef struct queue_t {
	NODE *head;
	NODE *tail;
	LOCK q_h_lock;
	LOCK q_t_lock;
} Q;

initialize(Q *q) {
   node = new_node()   // Allocate a free node
   node->next = NULL   // Make it the only node in the linked list
   q->head = q->tail = node   // Both head and tail point to it
   q->q_h_lock = q->q_t_lock = FREE   // Locks are initially free
}

enqueue(Q *q, TYPE value) {
   node = new_node()       // Allocate a new node from the free list
   node->value = value     // Copy enqueued value into node
   node->next = NULL       // Set next pointer of node to NULL
   lock(&q->q_t_lock)      // Acquire t_lock in order to access Tail
	  q->tail->next = node // Link node at the end of the queue
	  q->tail = node       // Swing Tail to node
   unlock(&q->q_t_lock)    // Release t_lock
｝

dequeue(Q *q, TYPE *pvalue) {
   lock(&q->q_h_lock)   // Acquire h_lock in order to access Head
	  node = q->head    // Read Head
	  new_head = node->next       // Read next pointer
	  if new_head == NULL         // Is queue empty?
		 unlock(&q->q_h_lock)     // Release h_lock before return
		 return FALSE             // Queue was empty
	  endif
	  *pvalue = new_head->value   // Queue not empty, read value
	  q->head = new_head  // Swing Head to next node
   unlock(&q->q_h_lock)   // Release h_lock
   free(node)             // Free node
   return TRUE            // Queue was not empty, dequeue succeeded
}{% endhighlight %}

而对于Non Blocking Queue，最核心的操作是一个叫做Compare And Swap(简称CAS)的操作。这个操作用C++来表示大概是下面的代码：

{% highlight cpp linenos=table %}
template <typename T>
bool CompareAndSwap(T* dest, T oldValue, T newValue)
{
  if (*dest == oldValue)
  {
	*dest = newValue;
	return true;
  }
  return false;
}{% endhighlight %}

咋一看好像没什么大不了的，但是要注意到这个操作上在某些硬件上是实现成一条指令的， 所以可以保证这个操作是原子的。在X86的CPU上，这个指令是CMPXCHG。

有了这条指令，我们就可以用它来实现很多原本必须在加锁的情况下才可以实现的并发算法，其中Non Block Queue也就是使用了它。

在著名的《Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms》论文里面，就有如下的Non Blocking Queue实现伪码：

{% highlight cpp linenos=table %}
structure pointer_t {ptr: pointer to node_t, count: unsigned integer}
  structure node_t {value: data type, next: pointer_t}
  structure queue_t {Head: pointer_t, Tail: pointer_t}

  initialize(Q: pointer to queue_t)
	 node = new_node()		// Allocate a free node
	 node->next.ptr = NULL	// Make it the only node in the linked list
	 Q->Head.ptr = Q->Tail.ptr = node	// Both Head and Tail point to it

  enqueue(Q: pointer to queue_t, value: data type)
   E1:   node = new_node()	// Allocate a new node from the free list
   E2:   node->value = value	// Copy enqueued value into node
   E3:   node->next.ptr = NULL	// Set next pointer of node to NULL
   E4:   loop			// Keep trying until Enqueue is done
   E5:      tail = Q->Tail	// Read Tail.ptr and Tail.count together
   E6:      next = tail.ptr->next	// Read next ptr and count fields together
   E7:      if tail == Q->Tail	// Are tail and next consistent?
			   // Was Tail pointing to the last node?
   E8:         if next.ptr == NULL
				  // Try to link node at the end of the linked list
   E9:            if CAS(&tail.ptr->next, next, <node, next.count%2B1>)
  E10:               break	// Enqueue is done.  Exit loop
  E11:            endif
  E12:         else		// Tail was not pointing to the last node
				  // Try to swing Tail to the next node
  E13:            CAS(&Q->Tail, tail, <next.ptr, tail.count%2B1>)
  E14:         endif
  E15:      endif
  E16:   endloop
		 // Enqueue is done.  Try to swing Tail to the inserted node
  E17:   CAS(&Q->Tail, tail, <node, tail.count%2B1>)

  dequeue(Q: pointer to queue_t, pvalue: pointer to data type): boolean
   D1:   loop			     // Keep trying until Dequeue is done
   D2:      head = Q->Head	     // Read Head
   D3:      tail = Q->Tail	     // Read Tail
   D4:      next = head.ptr->next    // Read Head.ptr->next
   D5:      if head == Q->Head	     // Are head, tail, and next consistent?
   D6:         if head.ptr == tail.ptr // Is queue empty or Tail falling behind?
   D7:            if next.ptr == NULL  // Is queue empty?
   D8:               return FALSE      // Queue is empty, couldn't dequeue
   D9:            endif
				  // Tail is falling behind.  Try to advance it
  D10:            CAS(&Q->Tail, tail, <next.ptr, tail.count%2B1>)
  D11:         else		     // No need to deal with Tail
				  // Read value before CAS
				  // Otherwise, another dequeue might free the next node
  D12:            *pvalue = next.ptr->value
				  // Try to swing Head to the next node
  D13:            if CAS(&Q->Head, head, <next.ptr, head.count%2B1>)
  D14:               break             // Dequeue is done.  Exit loop
  D15:            endif
  D16:         endif
  D17:      endif
  D18:   endloop
  D19:   free(head.ptr)		     // It is safe now to free the old node
  D20:   return TRUE                   // Queue was not empty, dequeue succeeded{% endhighlight %}

其中Enqueue操作最重要的是E9行，Dequeue操作最重要的是D13行。

# Enqueue

(E1-E3)首先，无论在Double lock还是Lock free的队列算法里面，enqueue操作都要求先把一个节点分配并设置好，然后再把这个节点放到队列里面，这样可以用尽量少的操作把节点完整的添加到队列里。

(E5-E6)然后，线程尝试从Q里面取出尾节点，并把next指针也一并取出来。需要注意的是，`Q->tail`总是指针队列里面的元素，但是并不总是指着尾节点，但是在操作中，`Q->tail`总是尝试尽可能的接近并指向尾节点。

(E7-E11)这几行主要是看CAS操作成功有些什么前提条件。首先CAS比较的是`tail.ptr->next`的值，而上面一行的if判断就表明，这个时候的`tail.ptr->next`一定是指向NULL，否则CAS操作是不能成功的。一旦CAS操作成功，也就意味着新节点已经被添加到队列的尾部。注意CAS保证了这个比较并设置的过程是原子性的。当添加成功之后，就可以跳出循环，准备结束enqueue。注意这个时候虽然插入了新的节点，但是没有更新`Q->tail`的值。

(E12-E14)这几行会在`tail == Q->tail`且`tail.ptr->next != NULL`的时候执行。这个条件意味着在取出tail的值之后，别的线程已经往队列里面添加了新的节点，但是`Q->tail`节点有可能没有更新。于是在这个条件下，线程就尝试更新`Q->tail`的值，使其往后挪动(利用CAS操作来更新`Q->tail`)，尽量的指向队列的尾节点。

(E17)这一行其实和上面类似，只不过这是在加入了新节点之后，该线程尝试更新`Q->tail`，使其指向尾节点。这里也需要利用CAS操作，因为有可能在E9行成功加入新节点之后，另一个线程则走到了E13行，这个时候另外这个线程成功更新了`Q->tail`。所以当当前线程走到E17行的时候，有可能`Q->tail`已经被更新了，所以就需要使用CAS来检查值并更新。

从上面可以知道，在插入新节点的时候，插入点总是在最后，并且在插入之后，会把`Q->tail`尽可能的往后挪。

# Dequeue

Dequeue函数有一个很重要的假设是`Q->head`总是指向队列的头结点。Dequeue的策略是，head节点指向的是一个假节点，实际的头结点是head的next节点。在dequeue的时候，首先将`head->next`的值取出，作为返回值，然后将head节点取出并释放，此时原本的next节点作为新的head节点。

(D2-D4)在开始其他操作之前，需要先把头结点`Q->head`和它的next节点取出来。这里还取了`Q->tail`节点，是因为需要判断队列是否为空，和`Q->tail`此时是否指向了尾节点。

(D5-D9)在`head.ptr == tail.ptr`并且`next.ptr == NULL`的时候，表示这个时候队列里面只有一个假节点，也就是说这个时候队列为空，所以这个时候就返回false。

(D10)走到了这一行，说明了这个时候`head.ptr == tail.ptr`但是`next.ptr != NULL`。也就是说，队列中这时不只一个节点，但是`tail.ptr`却和`head.ptr`指向同一个节点，所以这个时候`tail.ptr`的指向是落后于尾节点的。所以在这里就尝试将tail往后挪动，使其尽量的靠近尾节点。

(D11-D15)线程走到这个分支就表示队列此时有两个节点以上。这个时候先将next的节点的值取出，然后尝试将头结点指向next节点(通过CAS实现)。如果CAS操作成功了，就表示节点操作成功，这个时候就可以安全的返回值了。如果没有成功，就表示这个时候头结点已经被别的线程修改了，取值操作就失效了，所以就需要重新循环一次。

(D19)既然在D13的时候，CAS已经确保了原head节点不在队列里面，这个时候就可以把这个原来的节点删除。

从上面的讲述可以看书，Non Blocking Queue的实现上是通过轮询来解决竞态条件的。如果在之前取出的状态不满足队列操作当时的假设的话，就通过重新执行一次来继续进行操作。而CAS则保证了在执行队列操作过程中的原子性。

当然CAS操作是Lock free算法的很重要的一步，但是要实现Lock free算法是极其困难的一件事情。要保证其正确性，要从各个方面来进行测试和验证。冠诚曾经提到Doug Lea在实现java.util.concurrent里面的LinkedBlockingQueue的时候，是要用一个人年来实现的。所以在想要用Lock free算法的时候，应该尽量使用现有的算法，而不是重造轮子。

# References:

1.  [多线程队列的算法优化][12]
1.  [Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms][15]
1.  [https://www.ibm.com/developerworks/java/library/j-jtp11234/](https://www.ibm.com/developerworks/java/library/j-jtp11234/)
1.  [http://www.ibm.com/developerworks/java/library/j-jtp04186/index.html](http://www.ibm.com/developerworks/java/library/j-jtp04186/index.html)
1.  [http://www.codeproject.com/Articles/23317/Lock-Free-Queue-implementation-in-C-and-C](http://www.codeproject.com/Articles/23317/Lock-Free-Queue-implementation-in-C-and-C)


[12]: http://www.parallellabs.com/2010/10/25/practical-concurrent-queue-algorithm/ "多线程队列的算法优化"
[15]: http://www.cs.rochester.edu/research/synchronization/pseudocode/queues.html "Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms"

