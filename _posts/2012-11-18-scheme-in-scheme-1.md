---
layout: post
title: "Scheme Interpreter In Scheme(1)"
description: "implement a scheme interpreter in scheme itself"
category: scheme
tags: [scheme,PL]
---
{% include JB/setup %}

在这个系列里面，我会用scheme语言来实现一个scheme语言的解析器。
我们会在实现中学习到很多程序语言相关的概念和相关的实现，
这对于我们理解我们常用的语言也有很大的帮助。

# Scheme: A little bit history

Scheme语言是lisp语言的其中一个变种。Lisp语言可以说是计算机历史上第二长寿的语言了，
第一是Fortran。Lisp语言早期主要是应用在人工智能方面，
70年代至80年代由于人工智能的大繁荣，Lisp得到了很大的发展，但是后来由于人工智能的冬天，
Lisp的应用也随之进入了冬天。而就在这段冬天里，Scheme就在MIT诞生了。

Scheme作为Lisp最大的两个变种之一（另外一个是Common Lisp），在最近得到了很多的关注，
因为最近Scheme的其中一个JVM方言[Clojure](http://clojure.org)在业界得到了比较多的
应用。Scheme在诞生之初就有很多的创新，而其中最大的特征的就是Scheme是一门以minimalist
为设计思想的语言，也就是说Scheme的核心非常的小，但是里面却包含了许多强大的语言思想。

简单来说，Scheme包含了以下的特性：

1. 鼓励函数式编程。与传统的Imperative Programming不同，
函数式编程鼓励无副作用的编程方式，整个计算的过程可以用数学函数来描述，
从而达到简介表达高级程序逻辑的目的。（关于FP我也还在学习中）
1. 使用Lexical scoping。由于使用了Lexical scoping，所以实现闭包是非常简单的一件事。
1. 函数的尾递归(Tail recursion)优化。在函数式编程里面，
循环是比较不鼓励的一种编程style，
取而代之的是递归调用。而递归调用在平常的语言里面的开销比循环要大，但是有了尾递归之后，
循环和递归某种程度上是等价的。
1. 函数是first class object。这个在目前的很多语言中也都已经实现了。

除了上面的特性之外，Scheme还有延续(continuation)等其他的高级特性，在这里就不多说了。
如果感兴趣的话，可以移步[维基百科](http://en.wikipedia.org/wiki/Scheme_programming_language)看详细的介绍。

# 我们要实现的语言——Scheme的定义

讲了那么多，那么我们要实现的语言到底是怎么样的一个语言呢？

接下来我会讲述我们实现的Scheme包含的特性。而实现这个解析器的语言同时也可以用它来描述。

## 语法：S表达式

一个具有下面性质的表达式，可以称之为S表达式：

1. 一个不包含括号的原子表达式，比如1、"hello"、true、false等。
1. 一个用括号"()"括住的表达式，其中括号之间包含0个或以上的S表达式。

可以看到S表达式是一个递归的定义，所以下面的几个表达式都是S表达式：

    1 "hello" () (1 2) (("hello") 2) (+ 1 2)

而在Scheme里面，所有的表达式都是S表达式。其中第一种形式的S表达式称为atom，
而有括号的S表达式称为列表（list）。其中当表达式是列表形式的时候，
这个列表表示函数调用，其中第一个元素是函数的名字，后面的就是这个函数调用的实参。
也就是说`(+ 1 2)`表示的是`1 + 2`的意思。这种表示形式称为前缀表达式。

作为函数调用的另一个例子，假设mod是一个取模函数，就是第一个参数除于第二个参数的余数。
那么这个函数调用在Scheme里面就是写作`(mod 4 3)`就是在C里面的对应写法就是`mod(4, 3)`。

## 基本类型

我们编写的基本类型一共有以下几种：

1. Number，包括interger、floating point number。例如2，2.1。
1. String，和C里面的string是一样的，如"hello"。
1. Symbol，这个类型在Lisp里面比较常见，如abc。在Scheme里面，
要得到abc这个symbol，就用(quote abc)表示。
1. Boolean， 包括两个值，true和false。
1. List，这个和Python里面的List是类似的，不过写法是`(1 2 a)`。
并且Scheme里面的List不是数组，是单链表。而构造list的写法有几种：
    1. `(quote (1 2 a))`。注意到空的list表示为`(quote ())`
	1. `(cons 1 (cons 2 (cons a (quote ()))))`。注意到，
	元素的添加是通过`cons`来构造的。`(cons a b)`表示构造一个2个元素的list，
	其中第一个元素是a，余下的元素是b。
	1. 元素的取出是两个操作：car和cdr。假设a的值是`(cons 1 2)`，
	那么`(car a)`的值是1，`(cdr a)`的值是2。所以那上面的例子来说，
	`(car (quote (1 2 a)))`的值是1，`(cdr (quote (1 2 a)))`的值是
	`(quote (2 a))`。

## lambda

用过Python的人都知道Python里面有个keyword叫做lambda。但是Python里面的lambda功能很弱，
只能写一行的匿名函数。而Scheme里面的lambda就要强大多了，是一个功能完备的函数。

Scheme里面的lambda定义语法如下：


{% highlight scheme linenos %}
(lambda (args)
  (body)){% endhighlight %}

比如说下面的都是lambda

{% highlight scheme linenos %}
(lambda (x y)
  (+ x y))

(lambda (x)
  x)

(lambda (p)
  (p p)){% endhighlight %}

## 定义

定义包括变量定义和函数定义。其中变量定义是的语法如下：

{% highlight scheme linenos %}
(define a 1){% endhighlight %}

上面的表达式是定义了一个名为a的变量，他的值是1。

而函数定义的语法如下：

{% highlight scheme linenos %}
(define (add x y)
  (+ x y)){% endhighlight %}

其中add是函数名，而x、y是这个函数的参数，而这个函数体是`(+ x y)`，
也就是求两个参数的和。

而实际上，函数定义和变量定义是一样的，也就是函数定义等价于下面的语句：

{% highlight scheme linenos %}
(define add
  (lambda (x y)
    (+ x y))){% endhighlight %}

也就是函数实际上是一个值为lambda的变量。

## 闭包与Lexical scoping

闭包的准确定义是包含了其环境的函数，但是但从这句话里面我们很难明白到底什么是闭包。
用例子来解释应该是最简单的了。

比如说下面的Scheme代码：

{% highlight scheme linenos %}
(lambda (x)
  (lambda (y)
    (+ x y))){% endhighlight %}

上面的例子里，lambda里面的函数体是另外一个lambda，而这个里面的lambda使用了x，
这个x的定义并不在里面的lambda里面，而在外面的lambda，这个外面的lambda就是里面的
lambda的一个lexical的环境。那么我们就称里面这个lambda是一个闭包。

这里还涉及了一个叫做lexical scope的概念，它是和dynamic scope相对应的一个概念。
lexical scope的意思是，闭包里面的变量的取值是根据其定义的地方的环境来进行取值。

比如说例子里面的x取值就是外面的lambda的参数x的值。

而dynamic scope的意思就是说，闭包里面的变量值是根据调用的时候的环境来进行取值。
比如说下面的例子里面，

{% highlight scheme linenos %}
(define (inc x)
  (lambda (y)
    (+ x y)))

(define (test x)
  ((inc 3) 4))

(test 2){% endhighlight %}

对于dynamic scope的语言来说，上面的`(test 2)`的值是6，
但是对于lexical scope的语言来说，他的值是7。

## if条件语句

最基本的if条件语句是下面这样的

{% highlight scheme linenos %}
(if (= x 1)
    (+ x 1)
    x){% endhighlight %}

上面的语句和下面的C语句是等价的。

{% highlight c %}
x == 1 ? x + 1 : x{% endhighlight %}



