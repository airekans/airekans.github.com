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

而在这两个函数的实现中，核心是使用了`ZeroCopyInputStream`和`ZeroCopyOutputStream`这两个IO类。
那么到底Parse和Serialize是如何做到高效的呢？下面我们主要看Serialize的过程，Parse的过程是类似的，这里就不重复了。
我们先看看Serialize的主要流程。

# 序列化流程

先来看看从Message的`SerializeToOstream`开始，到调用到最底层的输出有些什么样的流程。

    +---------+                              +-----------------------+ +-------------------+
    | Message |                              | ZeroCopyOutputStream  | | CodedOutputStream |
    +---------+                              +-----------------------+ +-------------------+
         | ----------------------------------\           |                       |
         |-| client calls SerializeToOstream |           |                       |
         | -----------------------------------           |                       |
         |                                               |                       |
         | create ZeroCopyOutputStream                   |                       |
         |---------------------------------------------->|                       |
         |                                               |                       |
         |                                        return |                       |
         |<----------------------------------------------|                       |
         |                                               |                       |
         | SerializeToZeroCopyStream                     |                       |
         |-------------------------                      |                       |
         |                        |                      |                       |
         |<------------------------                      |                       |
         |                                               |                       |
         | create CodedOutputStream                      |                       |
         |---------------------------------------------------------------------->|
         |                                               |                       |
         |                                               |                return |
         |<----------------------------------------------------------------------|
         |                                               |                       |
         | SerializeWithCachedSizes                      |                       |
         |------------------------                       |                       |
         |                       |                       |                       |
         |<-----------------------                       |                       |
         |                                               |                       |
         | WriteString                                   |                       |
         |---------------------------------------------------------------------->|
         |                                               |                       |
         |                                               |                  Next |
         |                                               |<----------------------|
         |                                               |                       |
         |                                               | return                |
         |                                               |---------------------->|
         |                                               |                       |
         |                                               |                return |
         |<----------------------------------------------------------------------|
         |                                               |                       |

看到这里涉及到的`CodedOutputStream`，其是对`ZeroCopyOutputStream`的一层封装，
用来将一个Message格式化的输出到`ZeroCopyOutputStream`。
其中`CodedOutputStream`封装了PB的二进制编码格式，如果对PB的编码有兴趣的话，
可以自行移步[官方的encoding文档](https://developers.google.com/protocol-buffers/docs/encoding)进行了解，这里就不展开了。

