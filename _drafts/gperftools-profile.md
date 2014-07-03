---
layout: post
title: "用gperftools对C/C++程序进行profile"
description: "经历过gprof的难用，callgrind的高门槛之后，利用gperftools这一易用且低门槛的profiler对C++程序进行性能调优真是太爽了！"
category: cpp
tags: [cpp, profile, gperftools]
---
{% include JB/setup %}


# gperftools简介

在Linux编程的世界里，性能调优一直是个让人头疼的事。最出名的`gprof`虽然大家都知道，
其用法比较单一(只支持程序从启动到结束的profile)，而且对程序的运行时间会有比较大的影响，
所以其profile不一定准确。

而
