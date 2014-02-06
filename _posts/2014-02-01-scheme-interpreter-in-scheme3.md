---
layout: post
title: "Scheme Interpreter In Scheme(3)"
description: "用Scheme实现Scheme解析器系列第3篇，介绍了如何实现变量的定义和引用。"
category: scheme
tags: [scheme, Programming Language]
---
{% include JB/setup %}

[上一篇](scheme/2012/11/26/scheme-in-scheme-2/)我介绍了如何用Scheme实现atom的解析。
目前为止我们可以解析`Number`，`String`和`bool`类型的值。而在接下来的这篇文章里，
我会讲述如何实现变量的定义和引用。

# 变量定义

首先我们需要明确一下如何定义一个变量。在之前的文章里面，我已经提到过在Scheme里面的变量定义如下：

{% highlight scheme %}
(define a 1)
(define b "abc") 
{% endhighlight %}

上面的代码里面分别定义变量`a`和`b`为数字`1`和字符串`"abc"`。

如果从数据的角度来看，一个定义就是一个`list`，这个list包含3个元素：
其中第一个元素是symbol `define`；第二个元素也是symbol，不过是表示变量名字；
而第三个元素是这个变量的初始值，是一个atom。

而atom的解析，我们已经在前面的文章搞定了，但是symbol的解析我们还没有弄。
接下来我们先把symbol的解析搞定！

## 解析Symbol

在之前的讲解里面，我一直没有很明确的讲到底Symbol是什么。实际上在除了Scheme/Lisp语言之外，
很少有语言会有专门的类型处理Symbol(Ruby算是主流语言里有Symbol类型的一个了)。

在Scheme里面，Symbol就是一个”没有用引号的字符串“。实际上在`(define a 1)`里面，
`define`和`a`都是symbol，而他们是一个list里面的第一和第二个元素。

而一个Symbol的_表示形式_就是它本身，但是他的_输入形式_是这样的:

{% highlight scheme %}
(quote a) ; This is symbol a.
a ; This is variable a reference.
{% endhighlight %}

也就是说上面的表达式表示`a`这个symbol。为什么我们不能用`a`直接表示symbol呢？
原因是Scheme会默认把一个symbol解析成变量引用，就像上面的第二行。

而对于symbol，要判断两个symbol是否相等，可以用`eq?`函数(没错，Scheme里面`?`是合法的变量字符)。
就像下面这样：

{% highlight scheme %}
(define a (quote b))
(eq? a (quote b)) ; returns #t, which means true.
{% endhighlight %}

有了`eq?`，我们就可以判断一个symbol是不是我们想要的。

## define表达式的解析

重温一下，一个变量定义最简单是下面的形式：

{% highlight scheme %}
(define a 1)
(define b "abc")
{% endhighlight %}

那么我们可以用下面的方式来判断一个list是不是`define`表达式：

{% highlight scheme linenos=table %}
(define (eval exp)
  (cond ; eval other expression types mentioned before
        ((and (pair? exp) (eq? (car exp) (quote define)))
         (eval-definition exp))
        (else (display "Unknown type"))))
{% endhighlight %}



