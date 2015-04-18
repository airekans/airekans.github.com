---
layout: post
title: "Google benchmark：一个简单易容的C++ benchmark库"
description: "这篇文章介绍了Google的benchmark库，一个简单易用的C++库，相对于自己手写这些架子代码来说，做benchmark要简化很多。"
category: cpp
tags: [cpp, benchmark]
---
{% include JB/setup %}

在写C++程序的时候，经常需要对某些函数或者某些类的方法进行benchmark。一般来说，我们可以写一些简单的程序来进行测试，
然后跑一定的次数(比如10w次)，看看跑了多久。

比如我写了下面这个从`int`到`string`的转换程序：

{% highlight cpp linenos=table %}
string uint2str(unsigned int num)
{
    ostringstream oss;
    oss << num;
    return oss.str();
}{% endhighlight %}

那么我们可以写下面这个程序：

{% highlight cpp linenos=table %}
int main()
{
    for (int i = 0; i < 1000000; ++i) {
        (void) uint2str(i);
    }
    return 0;
}{% endhighlight %}

然后在命令用time跑，看看跑了多少时间，但是这样做有一个问题，如果我们需要和另外一个函数做比较，
则main函数需要写一个分支来跑这个函数，或者干脆重新写一个程序。另外如果我们需要比较在不同的数据规模下函数会跑多快，
则这个benchmark程序写起来就比较麻烦了。

正好最近看见Google开源的[benchmark C++库](https://github.com/google/benchmark)，且自己也在写`HashMap`，所以也就实践了用benchmark库来进行benchmark，
发现它有下面几个不错的feature：

1. 简单易容，如果用过gtest的人，写起来会非常熟悉。
2. 对于不同的data size进行benchmark支持很好，可以很简单的用同一个代码段跑不同的data size。
3. 输出的benchmark结果直接就是真实时间和CPU时间，且很方便的导入excel进行数据分析。
4. 支持多线程benchmark(这个我还没用到)。

这篇文章就会简单介绍一下如果用benchmark来写我们自己的benchmark程序。
    
# 简单使用

其实在benchmark这个库的README就已经有比较详细的介绍了，这里还是以上面的例子来做benchmark。
首先我们把benchmark下载下来，然后用cmake进行编译。然后我们在c++里面写下面的代码：


{% highlight cpp linenos=table %}
#include <benchmark/benchmark.h>

static void BM_uint2str(benchmark::State& state) {
    unsigned int num = 1234;
    while (state.KeepRunning())
        (void) uint2str(num);
}
// Register the function as a benchmark
BENCHMARK(BM_uint2str);

BENCHMARK_MAIN();{% endhighlight %}

有了上面的程序，然后编译链接，就可以直接跑了。需要注意在链接的时候要把`-lpthread`也加上，否则可能会有runtime exception。
跑这个程序，会有下面的输出：

    Run on (4 X 2504.66 MHz CPU s)
    2015-04-18 19:55:26
    Benchmark     Time(ns)    CPU(ns) Iterations
    --------------------------------------------
    BM_uint2str        428        425    1617472

怎么样，很直观吧？

有一个小地方需要注意的是，benchmark需要跑在一个循环里面，因为一般来说函数的时间会有一定的波动，
所以benchmark需要用一个state来表示是不是需要继续跑，一般来说，耗时短的函数会跑的多一些，
耗时长的函数会跑的少一些，总体来说每个benchmark都会跑差不多时间。

# 使用不同的参数跑benchmark

假设我们写了下面的函数:

{% highlight cpp linenos=table %}
void vuint2vstr(const vector<unsigned int>& vint, vector<string>& vstr) {
    vstr.clear();
    for (std::size_t i = 0; i < vint.size(); ++i) {
        vstr.push_back(uint2str(vint[i]));
    }
}{% endhighlight %}

我们可以用类似之前提到的方法来写benchmark，但是如果我想从不同的vector大小来测试上面的函数的性能呢？
直接用Range函数就可以了：

{% highlight cpp linenos=table %}
static void BM_vuint2vstr(benchmark::State& state) {
    vector<unsigned int> vuint;
    for (std::size_t i = 0; i < state.range_x(); ++i) {
        vuint.push_back(i);
    }
    
    vector<string> vstr;
    while (state.KeepRunning())
        vuint2vstr(vuint, vstr);
}
// Register the function as a benchmark
BENCHMARK(BM_vuint2vstr)->Range(8, 8 << 10);

BENCHMARK_MAIN();{% endhighlight %}

对！就是直接在`BENCHMARK`宏后面加上Range就可以了！第一个参数是起始值，第二个参数是终止值。
而在benchmark里面通过`state.range_x()`来获取实际的值。

用法非常简单，极大的简化了程序员的工作啊。

# 一个小Tips

其实上面的例子，都可以在benchmark的README里面找到，而且还有更多的例子，比如说模版支持，线程支持等。
不过在实际的使用中，我自己是发现了一个使用上的tips。

在benchmark里面，如果每个迭代会有一些额外的setup，我们可能会需要在循环里面做。
但是一般来说我们想要在benchmark时间统计里面把这部分去掉。
而在benchmark里面，刚好有两个函数可以做这个事情：`PauseTiming()`和`ResumeTiming()`。
咋一看好像不错，有builtin支持。
不过如果你真的在循环里面用了的话，那么在输出结果里面你可能会看到意外的结果——时间额外多了很多。

如果翻看benchmark的代码的话，你会发现在这两个函数的注释里写着这两个函数非常heavy weight，
最好不要在benchmark的循环里面用。
这是因为这两个函数里面有加锁和读`/proc`文件系统的操作，相对与纯CPU的操作，overhead还是有不少的。
所以在循环里面最好还是不要使用这两个函数。
