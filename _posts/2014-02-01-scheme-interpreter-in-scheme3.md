---
layout: post
title: "Scheme Interpreter In Scheme(3)"
description: "用Scheme实现Scheme解析器系列第3篇，介绍了如何实现变量的定义。"
category: scheme
tags: [scheme, Programming Language]
---
{% include JB/setup %}

[上一篇](scheme/2012/11/26/scheme-in-scheme-2/)我介绍了如何用Scheme实现atom的解析。
目前为止我们可以解析`Number`，`String`和`bool`类型的值。而在接下来的这篇文章里，
我会讲述如何实现变量的定义。

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

而一个Symbol的__表示形式__就是它本身，但是他的__输入形式__是这样的:

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

## define表达式的判断

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

对于`define`的判断，首先判断这个表达式是不是一个非空`list`，
然后判断它的第一个元素是不是symbol `define`。
当表达式满足上面的条件，就用`eval-definition`来解析整个`define`表达式。

既然`define`是用来定义变量的，那么定义变量需要做些什么呢？

## 程序的运行environment

在一个程序运行的时候，会有一个运行时环境伴随著它变化，我们可以称之为environment。
而这个environment里面，其实就是包含着所有的变量定义。
而如何表示environment，是每个解析器都需要解决的核心问题之一。

就当前来说，我们可以假设，所有的变量定义都是全局的。
那么我们可以用一个有两个元素的列表来表示environment，其中第一个元素是变量的symbol，
而第二个元素是变量的当前值。

所以我们可以用下面的代码来定义environment：

{% highlight scheme linenos=table %}
(define the-global-environment (cons (quote ()) (quote ())))

(define (define-variable! var val env)
  (set-car! env (cons var (car env)))
  (set-cdr! env (cons val (cdr env))))
{% endhighlight %}

我们用`the-global-environment`来表示全局的environment，而它的初始值是一个
包含了两个空列表的`cons cell`。
我们也定义了`define-variable!`来给解析器定义一个新的变量。
这里出现了两个新的函数`set-car!`和`set-cdr!`，分别用来设置一个`cons cell`的
第一个元素和第二个元素的值。

## define表达式的解析

因为定义一个变量需要修改environment，所以我们在`eval-definition`里面肯定需要用到它。
下面我们看看怎么定义`eval-definition`。

{% highlight scheme linenos=table %}
(define (eval-definition exp)
  (define-variable! (car (cdr exp)) (car (cdr (cdr exp)))
                    the-global-environment)
  (quote ok))
{% endhighlight %}

在`eval-definition`里面，我们只是简单的调用了一下`define-variable!`，
并返回一个`ok`。而在调用的`define-variable!`的时候，
我们从表达式里面取出第二个元素作为变量名，取出第三个元素作为变量值，
并把`the-global-environment`传递进去作为environment。

而返回`ok`，其实只是表示这个定义的表达式成功了，并没有太多的意义。
返回什么都是可以的，因为定义变量这个表达式本身的值不应该被使用。

最后我们把所有的代码都串起来，看看是什么样子。

{% highlight scheme linenos=table %}
(define the-global-environment (cons (quote ()) (quote ())))

(define (define-variable! var val env)
  (set-car! env (cons var (car env)))
  (set-cdr! env (cons val (cdr env))))

(define (eval exp)
  (cond ((number? exp) exp)
        ((string? exp) exp)
        ((or (eq? (quote true) exp) (eq? (quote false) exp)) exp)
        ((and (pair? exp) (eq? (car exp) (quote define)))
         (eval-definition exp))
        (else (display "Unknown type"))))

(define (eval-definition exp)
  (define-variable! (car (cdr exp)) (car (cdr (cdr exp)))
                    the-global-environment)
  (quote ok))
{% endhighlight %}

Wow，看起来非常高大上啊！！我们现在试试用这个解析器解析一些变量定义看看：

{% highlight scheme %}
(eval (read (open-input-string "(define a 1)"))) ; returns "ok"
(eval (read (open-input-string "(define b \"abc\")"))) ; returns "ok"
{% endhighlight %}

嗯，看起来不错，运行非常良好，可惜还不能引用这些定义了的变量。
接下来我会在第4篇里面讲述如何实现变量引用的解析。
