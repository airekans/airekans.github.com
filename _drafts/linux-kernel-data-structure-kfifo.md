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

需要注意的是，kfifo要求队列的size是2的幂，这样在后面操作的时候求余操作可以通过与运算来完成，从而更高效。

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



