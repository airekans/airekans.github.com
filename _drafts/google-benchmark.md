在写C++程序的时候，经常需要对某些函数或者某些类的方法进行benchmark。一般来说，我们可以写一些简单的程序来进行测试，
然后跑一定的次数(比如10w次)，看看跑了多久。

比如我写了下面这个从`int`到`string`的转换程序：

    string uint2str(unsigned int num)
    {
        ostringstream oss;
        oss << num;
        return oss.str();
    }


    
    
    
