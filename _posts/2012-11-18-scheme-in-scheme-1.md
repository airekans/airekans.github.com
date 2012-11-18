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
----

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
----

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
