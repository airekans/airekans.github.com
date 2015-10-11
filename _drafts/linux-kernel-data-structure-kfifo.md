# Linux内核中的队列 kfifo

在内核中经常会有需要用到队列来传递数据的时候，而在内核中就有一个轻量而且实现非常巧妙的队列实现——kfifo。
简单来说kfifo是一个有限定大小的环形buffer，借用网络上的一个图片来说明一下是最清楚的：

![kfifo-diagram](http://blog.chinaunix.net/attachment/201404/10/18770639_1397093507W9w9.bmp)

`kfifo`本身并没有队列元素的概念，其内部只是一个buffer。在使用的时候需要用户知道其内部存储的内容，所以最好是用来存储定长对象。

`kfifo`有一个重要的特性，就是当使用场景是单写单读的情况下，不需要加锁，所以在这种情况下的性能较高。

# 定义及API

kfifo主要定义在`include/linux/kfifo.h`里面：

```c
struct kfifo {
	unsigned char *buffer;	/* the buffer holding the data */
	unsigned int size;	/* the size of the allocated buffer */
	unsigned int in;	/* data is added at offset (in % size) */
	unsigned int out;	/* data is extracted from off. (out % size) */
	spinlock_t *lock;	/* protects concurrent modifications */
};

extern struct kfifo *kfifo_init(unsigned char *buffer, unsigned int size,
				gfp_t gfp_mask, spinlock_t *lock);
extern struct kfifo *kfifo_alloc(unsigned int size, gfp_t gfp_mask,
				 spinlock_t *lock);
extern void kfifo_free(struct kfifo *fifo);
extern unsigned int __kfifo_put(struct kfifo *fifo,
				const unsigned char *buffer, unsigned int len);
extern unsigned int __kfifo_get(struct kfifo *fifo,
				unsigned char *buffer, unsigned int len);
```

可以看到在kfifo本身的定义里面，有一个`spinlock_t`，这是用来在多线程同时修改队列的时候加锁的。而其余的成员就很明显了，是用来表示队列的当前状态的。队列本身的内容存储在`buffer`里面。

需要注意的是，kfifo要求队列的size是2的幂(2^n)，这样在后面操作的时候求余操作可以通过与运算来完成，从而更高效。

初始化通过`kfifo_init`和`kfifo_alloc`完成。而对于队列操作的主要函数的是`kfifo_put`和`kfifo_get`。这两个函数会先加锁，然后调用`__kfifo_put`或者`__kfifo_get`。也就是说真正的逻辑是实现在这两个函数里。
之前也说过`kfifo`在单读单写的情况下是不需要加锁的，所以这里我们会着重看看这两个函数。

# 入队

`__kfifo_put`的定义很短：

```c
unsigned int __kfifo_put(struct kfifo *fifo,
			const unsigned char *buffer, unsigned int len)
{
	unsigned int l;
	len = min(len, fifo->size - fifo->in + fifo->out);

	/*
	 * Ensure that we sample the fifo->out index -before- we
	 * start putting bytes into the kfifo.
	 */
	smp_mb();

	/* first put the data starting from fifo->in to buffer end */
	l = min(len, fifo->size - (fifo->in & (fifo->size - 1)));
	memcpy(fifo->buffer + (fifo->in & (fifo->size - 1)), buffer, l);

	/* then put the rest (if any) at the beginning of the buffer */
	memcpy(fifo->buffer, buffer + l, len - l);

	/*
	 * Ensure that we add the bytes to the kfifo -before-
	 * we update the fifo->in index.
	 */
	smp_wmb();
	fifo->in += len;

	return len;
}
```

可以看到里面加了一些memory barrier来确保单读单写场景的正确，这里我们可以暂时忽略。

主要的步骤如下：

1. 计算len和队列余下容量的较小值，如果队列容量不足，则只会拷贝剩余容量的大小。
2. 先拷贝一部分内容到队列的尾部。
3. 如果队列尾部并不能容下所有的内容，则再在队列的头部空闲空间继续拷贝。
4. 把队列内容长度加上len
5. 返回新增内容的长度len

这里注意到in只有在`__kfifo_put`里面才会修改，而这个函数里面只会对in增加，所以in的值只会增加，不会减少。而in本身是`unsigned int`类型的，所以当in超出了2^32的时候，会自动从0开始继续。

同时前面也说过，`kfifo`的size是2^n。所以当`in > 2^n`的时候，`(in & 2^n - 1) == (in % 2^n)`，所以这里可以用与操作替代求余来获取in在队列中实际的位置。

# 出队

`__kfifo_get`的定义和`__kfifo_put`长度差不多：

```c
unsigned int __kfifo_get(struct kfifo *fifo,
			 unsigned char *buffer, unsigned int len)
{
	unsigned int l;
	len = min(len, fifo->in - fifo->out);

	/*
	 * Ensure that we sample the fifo->in index -before- we
	 * start removing bytes from the kfifo.
	 */
	smp_rmb();

	/* first get the data from fifo->out until the end of the buffer */
	l = min(len, fifo->size - (fifo->out & (fifo->size - 1)));
	memcpy(buffer, fifo->buffer + (fifo->out & (fifo->size - 1)), l);

	/* then get the rest (if any) from the beginning of the buffer */
	memcpy(buffer + l, fifo->buffer, len - l);

	/*
	 * Ensure that we remove the bytes from the kfifo -before-
	 * we update the fifo->out index.
	 */
	smp_mb();
	fifo->out += len;

	return len;
}
```

忽略掉memory barrier之后，主要步骤如下：

1. 计算len和队列长度的较小值，如果队列内容不够，则只拷贝较小值的大小。
2. 拷贝队列尾部的内容到输出buffer里面。
3. 如果仍然有部分内容没有拷贝的话，则从队列头部拷贝余下的内容。
4. 队列内容长度减少len(也就是`out += len`)。
5. 返回拷贝内容的长度。

其实基本就是`__kfifo_put`的逆过程。

那这里就有一个问题了，其实队列的长度并不一定要用`in`和`out`两个变量来表示啊，也可以用一个`len`变量来表示啊。那这里就涉及到了多线程的互斥问题了。

# 多线程互斥

这里我们只考虑最简单的多线程场景——单读单写。如果我们只用一个`len`来表示队列长度的话，那么看看`__kfifo_put`和`__kfifo_get`里面对这个变量都需要做修改，而且一个是`+=`操作，一个是`-=`。如果在不加锁的情况下，这两个操作并不是原子操作，所以如果只用一个`len`，我们必须用锁来保护，无论是多么简单的多线程场景。

如果我们用`in`和`out`来表示队列的读边界和写边界的话，那么队列的长度可以用`in - out`来表示。而且就像我们看到的那样，`in`只会在`__kfifo_put`里面修改，而`out`也只会在`__kfifo_get`里面修改，所以无论是`in`或`out`都只会有一个线程修改，所以不会有互斥的问题。

那是不是这样就线程安全了呢？并不是。

还记得之前忽略掉的那些memory barrier吗？如果没有了那些barrier的话，代码仍然是不安全的。

