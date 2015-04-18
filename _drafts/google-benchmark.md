在写C++程序的时候，经常需要对某些函数或者某些类的方法进行benchmark。一般来说，我们可以写一些简单的程序来进行测试，
然后跑一定的次数(比如10w次)，看看跑了多久。

比如我写了下面这个从`int`到`string`的转换程序：

```cpp
string uint2str(unsigned int num)
{
    ostringstream oss;
    oss << num;
    return oss.str();
}
```

那么我们可以写下面这个程序：

```cpp
int main()
{
    for (int i = 0; i < 1000000; ++i) {
        (void) uint2str(i);
    }
    return 0;
}
```

然后在命令用time跑，看看跑了多少时间，但是这样做有一个问题，如果我们需要和另外一个函数做比较，
则main函数需要写一个分支来跑这个函数，或者干脆重新写一个程序。另外如果我们需要比较在不同的数据规模下函数会跑多快，
则这个benchmark程序写起来就比较麻烦了。

正好最近看见Google开源的benchmark C++库，且自己也在写`HashMap`，所以也就实践了用benchmark库来进行benchmark，
发现它有下面几个不错的feature：

1. 简单易容，如果用过gtest的人，写起来会非常熟悉。
2. 对于不同的data size进行benchmark支持很好，可以很简单的用同一个代码段跑不同的data size。
3. 输出的benchmark结果直接就是真实时间和CPU时间，且很方便的导入excel进行数据分析。

这篇文章就会简单介绍一下如果用benchmark来写我们自己的benchmark程序
    
