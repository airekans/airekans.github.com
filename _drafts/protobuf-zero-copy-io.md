# Protobuf中的IO

Protobuf中，Message的序列化和反序列化的效率一直是同类的库中比较高的。
一般来说，如果我们拿到一个Message之后，会用下面两个方法来进行序列化/反序列化：

{% highlight cpp %}
ofstream ofs("out.txt");
msg.SerailizeToOstream(&ofs);

// get the message from file.
ifstream ifs("out.txt");
msg.ParseFromIstream(&ifs);
{% endhighlight %}

也就是说protobuf中的`SerailizeToOstream`和`ParseFromIstream`实现的比较高效。
而在这两个函数的实现中，核心是使用了`ZeroCopyInputStream`和`ZeroCopyOutputStream`这两个IO类。

