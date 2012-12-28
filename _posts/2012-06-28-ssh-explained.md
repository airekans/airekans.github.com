---
layout: post
title: "SSH协议详解"
description: "Detailed explanation of SSH protocol"
category: protocol
tags: [Internet, protocol]
---
{% include JB/setup %}

作为程序员，一定不会没有用过ssh吧。当我们需要远程登录到服务器上进行操作的时候，一般就会用ssh。
ssh是secure shell的简称，它相对于早起的telnet和rsh的明文传输，提供了加密、校验和压缩，使得我们可以很安全的远程操作，
而不用担心信息泄露(当然不是绝对的，加密总有可能被破解，只是比起明文来说那是强了不少)。

本文会详细的讲解SSH协议是怎么定义的，以及他是怎么实现安全的加密。

# 几个基本概念

在介绍ssh协议之前，有几个涉及到的基本概念首先需要介绍，它们对于理解ssh协议本身有非常重要和关键的作用。

## 加密

加密的意思是将一段数据经过处理之后，输出为一段外人无法或者很难破译的数据，除了指定的人可以解密之外。
一般来说，加密的输入还会有一个key，这个key作为加密的参数，
而在解密的时候也会用一个相关联(有可能是相同)的key作为输入。粗略来说是下面的流程：
    
{% highlight py linenos %}
# 加密方
encrypted_data = encrypt(raw_data, key)
# 解密方
raw_data = decrypt(encrypted_data, key1){% endhighlight %}

目前主流的加密算法一般分为下面两类：

1. [私钥(secret key)加密][13]，也称为对称加密
2. [公钥(public key)加密][14]

## 私钥加密

所谓的私钥加密，是说加密方和解密方用的都是同一个key，这个key对于加密方和解密方来说是保密的，第三方是不能知道的。在第三方不知道私钥的情况下，是很难将加密的数据解密的。一般来说是加密方先产生私钥，然后通过一个安全的途径来告知解密方这个私钥。

## 公钥加密

公钥加密，是说解密的一方首先生成一对密钥，一个私钥一个公钥，私钥不会泄漏出去，而公钥则是可以任意的对外发布的。用公钥进行加密的数据，只能用私钥才能解密。加密方首先从解密方获取公钥，然后利用这个公钥进行加密，把数据发送给解密方。解密方利用私钥进行解密。如果解密的数据在传输的过程中被第三方截获，也不用担心，因为第三方没有私钥，没有办法进行解密。

公钥加密的问题还包括获取了公钥之后，加密方如何保证公钥来自于确定的一方，而不是某个冒充的机器。假设公钥不是来自我们信任的机器，那么就算我们用公钥加密也没有用，因为加密之后的数据是发送给了冒充的机器，该机器就可以利用它产生的私钥进行解密了。所以公钥加密里面比较重要的一步是身份认证。

需要说明一下，一般的私钥加密都会比公钥加密快，所以大数据量的加密一般都会使用私钥加密，而公钥加密会作为身份验证和交换私钥的一个手段。

## 数据一致性/完整性

数据一致性说得是如何保证一段数据在传输的过程中没有遗漏、破坏或者修改过。一般来说，目前流行的做法是对数据进行hash，得到的hash值和数据一起传输，然后在收到数据的时候也对数据进行hash，将得到的hash值和传输过来的hash值进行比对，如果是不一样的，说明数据已经被修改过；如果是一样的，则说明极有可能是完整的。

目前流行的hash算法有[MD5][15]和[SHA-1][16]算法。

## 身份验证

身份验证说的是，判断一个人或者机器是不是就是你想要联系的。也就是说如果A想要和B通信，一般来说开始的时候会交换一些数据，A怎么可以判断发送回来的数据就真的是B发送的呢？现实中有很多方法可以假冒一个机器。

在SSH里面，这主要是通过公钥来完成的。首先客户端会有一个公钥列表，保存的是它信任的机器上面的公钥。在开始SSH连接之后，服务器会发送过来一个公钥，然后客户端就会进行查找，如果这个公钥在这个列表里面，就说明这个机器是真的服务器。

当然实际的情况会复杂一些。实际上服务器不是真的发送公钥过来，因为这很容易被第三方盗取。这个在下面会详细的讲述。

#  SSH2协议概况

理解一个协议最好是从他的大概信息交流流程来了解。这个在《[SSH: The Secure][17]》里面有很详细的说明，我从中摘取了几个主要的图来说明一下。

首先是一个主要的脉络图：

![SSH overview][18]

可以看到，里面有几个关键的key：

1.  session key: 这个是用来作为secret key加密用的一个key，同时也作为每个ssh连接的标识ID。
2.  host key: 这个是用来作为server的身份验证用的。
3.  known-hosts: 这个是存在客户端的一个可信server的public key列表。
4.  user key: 这个是用来作为client的身份验证用的。

当server和client交换了session key之后，所有的数据都会使用这个session来进行私钥加密。

上面的图是一个很粗略的描述，下面这个图是对SSH2协议的一个详细的描述：

![SSH2 protocol details][19]

上面这幅图大致的说明了SSH2协议的全景。首先SSH2协议分为3个子协议，分别是SSH-TRANS, SSH-AUTH和SSH-CONN。其中SSH-TRANS是传输协议，定义了传输的包和加密通道，其他两个协议是建立在这个协议之上的。

SSH-AUTH是SSH里面用于验证客户端身份的协议。我们在用ssh命令输入密码的那一步实际上就是在这个阶段。可以看到的是，虽然传输的是用户名和密码，但是由于这个协议建立在SSH-TRANS之上，所以内容都是加密的，可以放心的传输。

而SSH-CONN是真正的应用协议。在这里可以定义各种不同的协议，其中我们经常使用的scp、sftp还有正常的remote shell都是定义在这里的一种协议实现。这里的各种应用协议都要首先经过SSH-AUTH的验证之后才可以使用。

这个三个协议之间的关系可以用下面这幅图来说明：

![SSH protocol relationship][20]

其中SSH-TRANS是基本的协议，SSH-AUTH和SSH-CONN都是通过这个协议来实现安全加密的。虽然在时序上，SSH-CONN发生在SSH-AUTH之后，但是SSH-CONN并不依赖于SSH-AUTH。

# SSH Protocol

## SSH-TRANS

首先介绍一下SSH-TRANS的基本结构。在客户端连接上SSH服务器之后，会进行下面协议通信：

1.  客户端和服务端都向对方发送一个ssh版本字符串。字符串的格式如下：

        SSH-protoversion-softwareversion SP comments CR LF

    其中comment是可选的。
    一般来说，目前用的ssh服务器和客户端一般都是支持SSH2，所以一个开始的version string一般就像下面这样：
    
        SSH-2.0-OpenSSH CR LF

1.  接下来的通信都用SSH自身定义的一个Binary Packet Protocol进行通信。这个Binary Packet Protocol其实就是将所有的用户数据都加上长度头，然后再进行加密。一个Packet的定义如下： 

        uint32    packet_length
        byte      padding_length
        byte[n1]  payload; n1 = packet_length - padding_length - 1
        byte[n2]  random padding; n2 = padding_length
        byte[m]   mac (Message Authentication Code - MAC); m = mac_length
    
    实际上所有的数据都放在payload里面。最后的mac是用来给数据计算校验码用的。
	
1.  在传输完ssh version string之后，客户端和服务端会开始进行key exchange，简称kex。Kex是用来让客户端和服务器生成本次通信的密钥和session ID的。
在kex之后，服务器和客户端都有一个key和hash，而私钥加密用的secret key就是通过这两个值来生成的。
具体的算法这里就不阐述了，可以去看SSH-TRANS的RFC\[2\]。在kex的最后一步，服务器会给客户端发送他自己的public key。
而客户端会通过在自己的known_hosts里面查找这个public key来验证服务器的身份。 
至此，服务器和客户端都用来secret key，所以接下来所以数据都会进行加密，而不用担心信息泄露。
在kex之后，客户端就可以开始进行SSH-AUTH，也就是叫服务器验证自己的身份。


## SSH-AUTH
    
在客户端的身份认证中，有3种预先定义好的方法可以用。
    
1. public key
2. password
3. hostbased
    
其中前两种是我们平常最常用的：password就是一般的密码验证，而public key就是一般的无密码验证。
当服务器成功的验证了客户端的身份之后，就会开始客户端请求的服务(service)了。
需要注意的是，服务器的验证方式并不是说3种方式任选其一，而是可以组合的。也就是说，服务器可以要求客户端同时通过Password和public key两种方式的认证。


## SSH-CONN
    
这个也就是我们最后用到的一个服务的协议定义了。最常用的包括shell， port forwarding，X11 forwarding等等。

在SSH-CONN里面最重要的就是Channel的机制了。在SSH-CONN里面，和服务器的通信基本上都是通过建立channel来通信的。
多个channel共享同一个ssh session。SSH协议自身定义如何负责多个channel之间消息的分发。
对于使用者来说只需要开多个channel就可以了。
比如说普通在ssh的客户端开启port forwarding的时候，就会开启一个shell channel和一个forwarding channel。
这一part对于程序员来说都是比较熟悉的。


# Library

目前看的ssh的库主要有[libssh][21]和[libssh2][22]。其中的比较可以在[这里][23]找到。从接口上来说，
libssh2的接口定义比较清晰，不过libssh2只能用于client端的开发，而libssh可以进行server和client端的开发。
而且libssh2的文档比libssh的文档要差些。在做开发的时候文档是一个很关键的因素。


# References
    
1.  [SSH: The Secure Shell][17]
1.  [SSH-TRANS][24]
1.  [SSH-ARCH][25]
1.  [SSH-AUTH][26]
1.  [SSH-CONN][27]


[13]: http://en.wikipedia.org/wiki/Symmetric-key_algorithm
[14]: http://en.wikipedia.org/wiki/Public-key_encryption
[15]: http://en.wikipedia.org/wiki/MD5
[16]: http://en.wikipedia.org/wiki/Sha1
[17]: http://docstore.mik.ua/orelly/networking_2ndEd/ssh/index.htm
[18]: http://docstore.mik.ua/orelly/networking_2ndEd/ssh/figs/ssh_0301.gif "SSH overview"
[19]: http://docstore.mik.ua/orelly/networking_2ndEd/ssh/figs/ssh_0304.gif "SSH2 protocol details"
[20]: http://docstore.mik.ua/orelly/networking_2ndEd/ssh/figs/ssh_0305.gif "SSH protocol relationship"
[21]: http://www.libssh.org/
[22]: http://www.libssh2.org/
[23]: http://www.libssh2.org/libssh2-vs-libssh.html
[24]: http://tools.ietf.org/html/rfc4253 "SSH-TRANS"
[25]: http://tools.ietf.org/html/rfc4251 "SSH-ARCH"
[26]: http://tools.ietf.org/html/rfc4252 "SSH-AUTH"
[27]: http://tools.ietf.org/html/rfc4254 "SSH-CONN"
