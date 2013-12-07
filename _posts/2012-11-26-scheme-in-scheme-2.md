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

{% highlight scheme linenos=table %}
(define l (read (open-input-string "(define a 1)")))
(if (eq? (quote define) (car l))
    (display "It's definition!")
    (display "It's not definition!"))

(if (number? (car (car (car l))))
    (display "It's number!")
    (display "It's not number!")){% endhighlight %}

上面的代码里面，我将用`read`读进来的表达式用`car`取出第一个symbol，
然后用`eq?`来进行比对。`eq?`是一个用来判断两个symbol是否一样的函数。
而`number?`就是一个用来判断参数是不是Number类型的函数。
除了上面两个函数之外，还有`string?`函数，
它可以用来判断参数是不是String类型的。

看了上面的代码，估计你心中已经大概有了一点概念了吧？

# 解析器的基本结构

有了前面的说明，接下来我们就要想想怎么写解析器才可以实现之前说的语言了。

既然我们是写解析器，解析器实际上就是一个evaluate表达式的过程，
我就把这个解析器的函数命名为eval。

假设现在我们只需要解析最基本的atom，比如`1`, `a`, `define`的话，
那么`eval`要怎么写呢？首先在scheme里面，有一个函数是`pair?`，
是用来判断一个表达式是不是list的。

比如：

{% highlight scheme linenos=table %}
(pair? 1) ; false
(pair? (cons 1 2)) ; true{% endhighlight %}

有了`pair?`之后，我们就可以很方便判断一个S表达式是不是atom了。
下面是一个只解析atom的解析器：

{% highlight scheme linenos=table %}
(define (eval exp)
  (if (not (pair? exp))
      (if (number? exp)
          exp
          (display "Unknown type"))
      (display "Unknown type")))

(eval 1) ; returns 1
(eval 10) ; returns 10
(eval "hello") ; display "Unknown type"{% endhighlight %}

看到上面的代码中，实际上`eval`的定义可以简化成只用一个`number?`判断，
因为`number?`就是一个类型检查。如下：

{% highlight scheme linenos=table %}
(define (eval exp)
  (if (number? exp)
      exp
      (display "Unknown type"))){% endhighlight %}

如果现在加入对字符类型的atom进行解析的话，要怎么写呢？还记得之前我们有`string?`
来对参数进行String的类型判断么？对，我们就用`string?`就可以了，如下：

{% highlight scheme linenos=table %}
(define (eval exp)
  (if (number? exp)
      exp
      (if (string? exp)
          exp
          (display "Unknown type"))))

(eval 11) ; returns 11
(eval "hello") ; returns "hello"
(eval (quote a)) ; display "Unknown type"{% endhighlight %}

# cond表达式

在上面的eval里面，我们用了两个if，而if越多，嵌套就越多，
那么想想我们如果要处理的表达式类型越多，那么我们嵌套不就……
在C里面，可以用`switch`或者连续的`if`来避免深层的嵌套，比如：

{% highlight cpp linenos=table %}
if (i == 1)
{
    i++;
}
else if (i == 2)
{
    i--;
}
else
{
    i += 2;
}{% endhighlight %}

其实在Scheme里面，有一个`cond`表达式，它的作用和上面C里面的`if`类似。

{% highlight scheme linenos=table %}
(cond ((= a 1) a)
      ((> a 1) (+ a 1))
      (else (- a 1))){% endhighlight %}

上面的表达式应该不难看懂吧？我们用C来表示一次，你应该就是明白了：

{% highlight cpp linenos=table %}
if (a == 1)
{
    a;
}
else if (a > 1)
{
    a + 1;
}
else
{
    a - 1;
}{% endhighlight %}

有了`cond`表达式，那么我们用`cond`来“重构”一下我们的解析器吧。

{% highlight scheme linenos=table %}
(define (eval exp)
  (cond ((number? exp) exp)
        ((string? exp) exp)
        (else (display "Unknown type")))){% endhighlight %}

现在我们的解析器已经可以处理Number和String了，那么还有什么是atom呢？
还有Boolean我们没有处理。那我们现在来分析一下怎么处理Boolean吧。首先需要注意的是，
在mit-scheme里面，boolean的值是`#t`和`#f`，而在我们要实现的解析器里面，
boolean的值是`true`和`false`。这里的关系，和用C来实现Scheme是类似的，
实现语言C里面的boolean是1和0，而被实现的语言里面的boolean是`true`和`false`。
而这里`true`和`false`从解析器的角度来说是symbol类型的，所以我们可以用`eq?`
来进行判断。

有了上面的说明之后，那么我们现在来加入对boolean的解析吧。

{% highlight scheme linenos=table %}
(define (eval exp)
  (cond ((number? exp) exp)
        ((string? exp) exp)
        ((or (eq? (quote true) exp) (eq? (quote false) exp)) exp)
        (else (display "Unknown type")))){% endhighlight %}

上面的`or`和C里面的`||`或者Python里面的`or`是一样的作用的。
OK，有了上面解析器，那么现在我们玩一玩吧！

{% highlight scheme linenos=table %}
(eval (read (open-input-string "1"))) ; 等同于(eval 1)
(eval "hello") ; returns "hello"
(eval (read (open-input-string "true"))) ; returns true
(eval (read (open-input-string "false"))) ; return false{% endhighlight %}

好了，现在我们已经有了一个能用的解析器了，虽然它现在只能解析atom，但是在接下来的几节中，
我们还会继续的丰富这个解析器，它就能慢慢地解析更多的东西啦。
