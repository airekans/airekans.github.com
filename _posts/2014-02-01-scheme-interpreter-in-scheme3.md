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
很少有语言会有专门的类型处理Symbol(Ruby算是主流语言里有Symbol的一个了)。

在Scheme里面，Symbol就是
