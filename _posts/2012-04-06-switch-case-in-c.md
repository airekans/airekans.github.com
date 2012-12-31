---
layout: post
title: "深入理解C里面的switch"
description: ""
category: C
tags: [c, Programming Language]
---
{% include JB/setup %}

事情的起因是这样的，在wx的源码里面看到了下面一段比较诡异的代码：

{% highlight cpp linenos %}
switch ( level ) {
	case wxLOG_Info:
		if ( GetVerbose() )  // ***** Note here ****
	case wxLOG_Message:
		{
			m_aMessages.Add(szString);
			m_aSeverity.Add(wxLOG_Message);
			m_aTimes.Add((long)t);
			m_bHgasessages = true;
		}
		break;
	...
}{% endhighlight %}

上面的`switch`活生生的把一个if的条件部分和主体部分给分开到两个case里面，最诡异的是这竟然是合法的。  
到底是怎么一回事呢？  
那就需要来看看switch到底是怎么实现的。

下面我就用C来写一个类似的程序，然后看看它的汇编代码是怎么一回事。  
下面的C程序用下面的命令就可以得到汇编输出：

    gcc -S -o switch.s switch.c

{% highlight cpp linenos %}
int main(int argc, const char *argv[])
{
	int i = 1;

	switch (i)
	{
		case 1:
			if (i == 1)
		case 2:
			{
				i = 2;
			}

			i = 3;
			break;
		default:
			i = 4;
	}

	return ;
}{% endhighlight %}

来看一下汇编输出，注意这里的汇编是[GAS][1]的语法：

{% highlight gas linenos %}
.file	"switch.c"
	.text
.globl main
	.type	main, @function
main:
	pushl	%ebp
	movl	%esp, %ebp
	subl	$8, %esp
	andl	$-16, %esp
	movl	$, %eax
	addl	$15, %eax
	addl	$15, %eax
	shrl	$4, %eax
	sall	$4, %eax
	subl	%eax, %esp
	movl	$1, -4(%ebp)
	movl	-4(%ebp), %eax
	movl	%eax, -8(%ebp)
	cmpl	$1, -8(%ebp)
	je	.L3
	cmpl	$2, -8(%ebp)
	je	.L5
	jmp	.L6
.L3:
	cmpl	$1, -4(%ebp)
	jne	.L4
.L5:
	movl	$2, -4(%ebp)
.L4:
	movl	$3, -4(%ebp)
	jmp	.L2
.L6:
	movl	$4, -4(%ebp)
.L2:
	movl	$, %eax
	leave
	ret
	.size	main, .-main
	.section	.note.GNU-stack,"",@progbits
	.ident	"GCC: (GNU) 3.4.4 20050721 (Red Hat 3.4.4-2)"{% endhighlight %}

注意一下最主要的部分：

{% highlight gas linenos %}
movl	$1, -4(%ebp)
movl	-4(%ebp), %eax{% endhighlight %}

这两句相当于：

接下来的那段就是switch：

{% highlight gas linenos %}
	movl	%eax, -8(%ebp)  # 判断部分 switch (i)
	cmpl	$1, -8(%ebp)
	je	.L3
	cmpl	$2, -8(%ebp)
	je	.L5
	jmp	.L6
.L3:   # case 1:
	cmpl	$1, -4(%ebp)
	jne	.L4
.L5:   # case 2:
	movl	$2, -4(%ebp)
.L4:
	movl	$3, -4(%ebp)
	jmp	.L2
.L6:   # default:
	movl	$4, -4(%ebp)
.L2:{% endhighlight %}

可以看到，实际上switch是在开始的部分用一系列的cmp来判断变量i是不是与case中的几个值相等，如果等于就jmp到对应的lable。这里的逻辑相当于使用了goto语句。  
而几个case的地方，实际上汇编代码是连接起来的，所以像开头所说的那部分condition和body分开的情况是可以存在的。  
实际上C里面的switch完全等价于goto语句，如下面的switch：

{% highlight cpp linenos %}
switch (i)
{
	case 1:
		if (i == 1)
	case 2:
		{
			i = 2;
		}

		i = 3;
		break;
	default:
		i = 4;
}{% endhighlight %}

等价于下面的goto语句实现：

{% highlight gas linenos %}
if (i == 1)
	goto L1;
else if (i == 2)
	goto L2;
else
	goto Ldefault;

L1:
	if (i == 1)
L2:
	{
		i = 2;
	}

	i = 3;
	goto Lend;
Ldefault:
	i = 4;

Lend:{% endhighlight %}

当然goto实际上就是C语言版本的jmp指令了。

PS: 有兴趣的童鞋可以去看看[Duff’s device][2]，你就知道switch是多么强大、多么tricky的一个语句了。


[1]: http://en.wikipedia.org/wiki/GNU_Assembler
[2]: http://en.wikipedia.org/wiki/Duff%27s_device
 
