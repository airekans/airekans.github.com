---
layout: post
title: "用gperftools对C/C++程序进行profile"
description: "经历过gprof的难用，callgrind的高门槛之后，利用gperftools这一易用且低门槛的profiler对C++程序进行性能调优真是太爽了！"
category: cpp
tags: [cpp, profile, gperftools]
---
{% include JB/setup %}


# 什么是perftools

在Linux编程的世界里，性能调优一直是个让人头疼的事。最出名的`gprof`虽然大家都知道，
其用法比较单一(只支持程序从启动到结束的profile)，而且对程序的运行时间会有比较大的影响，
所以其profile不一定准确。

而`valgrind`功能十分强大，但profile也一般针对整个程序的运行，很难只对程序运行中的某段时间进行profile。
而且也多少会影响程序的运行，且使用的难度也较大，所以我目前还没尝试。

除去上面的两个常见的工具，之前在公司的项目见过使用Google的[gperftools](https://code.google.com/p/gperftools/)
进行profile的，当时就被他简单的使用方法吸引。而最近维护的服务器也有性能问题，需要做性能调优。
在尝试了多种原始的profile方式之后，我选择了`gperftools`。

# 如何使用gperftools进行profile

在gperftools的文档中，就简单的说了下面的方式来进行profile：

    gcc [...] -o myprogram -lprofiler
    CPUPROFILE=/tmp/profile ./myprogram

是的，在编译和安装了`gperftools`之后，只需要上面的步骤就可以进行profile了，非常简单。
而profile的结果就保存在`/tmp/profile`。查看结果只需要用`gperftools`自带的一个`pprof`脚本来看就可以：

    $ pprof --text ./myprogram /tmp/profile
    14   2.1%  17.2%       58   8.7% std::_Rb_tree::find

`pprof`的输出也很直观，不过也还不够好，从这个输出中还不好看出调用关系，包括caller和callee。
而pprof也可以输出图示，还可以输出callgrind兼容的格式，这样就可以用`kcachegrind`来看profile结果了。

    $ pprof --callgrind ./myprogram /tmp/profile > callgrind.res



