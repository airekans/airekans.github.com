---
layout: post
title: "利用jekyll搭建中文博客"
description: "how to setup a blog by jekyll"
category: jekyll
tags: [jekyll]
---
{% include JB/setup %}

好吧，今天终于开始在[github](https://github.com)上面开始写博客了。

之前在CSDN上面写点技术文章，感觉也还过得去。无奈CSDN的样子实在太丑了点，
而且对于代码的显示也不太友好，而且很多地方都不能自己配置，所以后来转到了
在SAE上面搭一个wordpress。

SAE嘛，刚开始吸引我的地方是他是一个Paas，而且有许多PHP应用已经port到了SAE上面，
其中就包括了Wordpress。而且SAE也是一段时间内免费的，所以我就开始了往SAE上面迁移博客。
说是迁移，其实也没有把CSDN那里的文章搬过来……Wordpress好是好，可以当过了半年之后，
注册SAE时候送的那500云豆就花完了，所以我的博客在某一天就上不去了，
而我就花了10RMB暂时的买了几个云豆。所以嘛，本着不想花钱，又想自己折腾的精神，
我就开始在Github上面搭博客了。

目前在Github上面打博客一般就是通过Github Pages或者是jekyll来搭建了。
Github Pages其实引擎也都是用的jekyll，所以最终我就决定自己用jekyll来搭了。
下面就开始进入主题，介绍一下我的这个博客是怎么搭建起来的。

# [jekyll](https://github.com/mojombo/jekyll)是个啥东西？

jekyll实际上是由github开发出来的用于在github上面放置静态页面的一个页面生成工具。
它不是像wordpress那样的一个博客web程序。它是一个从markdown文件生成静态的HTML的工具。

实际上，用jekyll不单只可以做博客，也可以做一些其他的动态内容不太多的网站。
不过一般来说，动态内容不多的也就博客了。所以下面先来讲讲用jekyll写博客是个怎么样的流程。

假设已经装好了jekyll，如果我现在要写一篇新的博客，那么就会有下面的流程：

1. `rake post title="test"` 这个命令在_post目录下面创建了一个
   "datetime-test.md"的文件了.
1. `jekyll --server` 这个命令启动了jekyll的server程序，监听4000
   端口。这样就可以打开浏览器进行浏览了，网址是"localhost:4000"。
1. 重复1。

有了上面的流程，写一个博客就很方便了，开着jekyll server，然后用你最喜欢的编辑器，
写markdown，就是这么简单。

# 搭建jekyll博客环境

要搭建jekyll环境很简单，你只需要一个安装好ruby 1.9.3，然后执行下面的命令：

    gem install jekyll

接着在github创建一个叫做`USERNAME.github.com`的repo。

然后去用下面的命令把一个jekyll的博客模板下载下来：

    git clone https://github.com/plusjade/jekyll-bootstrap.git USERNAME.github.com
	cd USERNAME.github.com
	git remote set-url origin git@github.com:USERNAME/USERNAME.github.com.git
	git push origin master


这样，你就创建好了一个jekyll博客了。你现在可以打开`USERNAME.github.com`看看。

有了上面的步骤，接着你就是修改一下repo里面的index.md，
还有创建博客的时候按照上面描述的顺序去创建就可以了。

# 代码语法高亮

在jekyll的文档里面，说到代码的语法高亮是通过[pygment](http://pygments.org/)来实现的。
按照文档上面说，用下面的格式就可以实现语法高亮了：

    {{ "{% highlight ruby linenos " }}%}
    def foo
        puts 'foo'
    end
    {{ "{% endhighlight " }}%}

但是如果不小心的话，jekyll是只会将代码块区分开来，但是并没有将其语法高亮。
后来仔细看文档，发现了下面的话：

> In order for the highlighting to show up, you’ll need to include a highlighting stylesheet. For an example stylesheet you can look at [syntax.css](http://github.com/mojombo/tpw/tree/master/css/syntax.css). These are the same styles as used by GitHub and you are free to use them for your own site.

所以关键的就是把上面提到的那个syntax.css文件加到默认的css加载里面去。
由于我默认用的是twitter主题，所以就做如下的改动：

1. 将syntax.css放到assert/themes/twitter/css/里面去。
1. 在_include/themes/twitter/default.html里面的head节点里面把上面的syntax.css给加载上去。

用了上面的方法，就可以实现和Github一样的语法高亮了。
对于默认的语法高亮设置，[这篇文章](http://www.stehem.net/2012/02/14/how-to-get-pygments-to-work-with-jekyll.html)也讲到如何设置，如果有什么问题也可以参考一下。

# Building Github Pages

其实这个过程还是很简单的，就只需要将你本地的commit push到github，
那么github马上会帮你自动的build。成功之后，你的邮箱就会收到一个
page build successful的邮件。

但是在某些情况下，你的commit可能会build不成功。In this case，你的page并不会更新，
而是停留在上一次好的版本。当出现这种情况，有下面几种问题的解决方案：

1. 仔细看github上面关于用jekyll来build pages的[文档](https://help.github.com/articles/using-jekyll-with-pages)。
1. 参考github上关于jekyll的[trouble shooting](https://help.github.com/articles/pages-don-t-build-unable-to-run-jekyll)。
1. Last but not the least, 发邮件问github的support.

为什么会说到这个呢，因为在我的pages里面，就出现过这个情况。
原因是我用了liquid里面的raw tag，而github的jekyll是不支持这个tag的。
所以在我push了commit之后，page一直没有更新。后来问support才知道，原来是page
的build出现问题了。然后看了一下trouble shooting，才知道原来我用了raw tag……
总之，问题解决就是好事。

# 主题

至于主题这个事，我现在还在慢慢的研究，暂时还是用回默认的twitter主题。

以后有什么补充的话，我会继续在这个文章里面进行补充。
