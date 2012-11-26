---
layout: post
title: "Scheme Interpreter In Scheme(2)"
description: "Basic structure of the interpreter"
category: scheme
tags: [scheme, Programming Language]
---
{% include JB/setup %}

在之前的[介绍](scheme/2012/11/18/scheme-in-scheme-1/)里面，
我讲了我们想要实现的Scheme语言的定义，并且用这个定义好的语言写了一些例子程序。
那么在这篇文章里面，我会讲讲大概的解析器是什么样子的。

# 前提：Lexer

在编译原理里面，介绍编译器的时候，一般都会介绍前端的一个重要的组成部分是Lexer的模块。
Lexer是词法分析器，也就是讲输入的字符流转换成语法定义的Token流。
一般的实现都是用状态机来实现，而在我们的解析器里面，为了简化实现的难度，我们利用Scheme
内置的`read`函数，它相当与Scheme的Lexer。它每次都从input-stream输入一个S表达式。

举个例子，看下面的代码：

{% highlight scheme linenos %}
(read (open-input-string "(define a 1)"))  ; read from stdin
;;; 上面的表达式返回(define a 1),
;;; 这个表达式也可以用下面的表达式来获得
(cons (quote define) (quote a) 1){% endhighlight %}

上面的代码也能看出一个Lisp的重要特性——代码即数据。在Lisp里面，
Lisp代码可以很容易的看成是Lisp里面的数据，基本不用什么特别的处理。
这个特性让Lisp语言的拓展性相比起其他语言来有很大的优势。

接下来我们的解析器，都用`read`来进行输入的转换。基于`read`，
我们就能假设输入进来的Lisp代码，可以用相对于atom或者list的操作来进行处理，
而不用用字符操作来进行处理。

上面的说明是什么意思？用下面的代码来说明一下应该最好：

{% highlight scheme linenos %}
(define l (read (open-input-string "(define a 1)")))
(if (eq? (quote define) (car l))
    (display "It's definition!")
	(display "It's not definition!")){% endhighlight %}

上面的代码里面，我将用`read`读进来的表达式用`car`取出第一个symbol，
然后用`eq?`来进行比对。

看了上面的代码，估计你心中已经大概有了一点概念了吧？

# 解析器的基本结构

有了前面的说明，接下来我们就要想想怎么写解析器才可以实现之前说的语言了。

既然我们是写解析器，解析器实际上就是一个evaluate表达式的过程，
我就把这个解析器的函数命名为eval。

假设现在我们只需要解析最基本的atom，比如`1`, `a`, `define`的话，
那么`eval`要怎么写呢？首先在scheme里面，有一个函数是`pair?`，
是用来判断一个表达式是不是list的。

比如：

{% highlight scheme %}
(pair? 1) ; false
(pair? (cons 1 2)) ; true{% endhighlight %}

