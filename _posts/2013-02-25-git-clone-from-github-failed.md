---
layout: post
title: "Git Clone From Github Failed"
description: "在某些环境下，从Github clone repo的时候，会遇到失败。这篇文章讲述了其中的一个解决方案。"
category: git
tags: [git, github]
---
{% include JB/setup %}

# 缘由

这几天在电脑上从公司网络尝试从github pull代码下来的时候，遇到了下面的错误。我是用的https的连接。

    $ git pull origin
    error: SSL certificate problem, verify that the CA cert is OK. Details:
    error:14090086:SSL routines:SSL3_GET_SERVER_CERTIFICATE:certificate verify failed while accessing https://github.com/airekans/Reshaper.git/info/refs

    fatal: HTTP request failed

这导致我没有办法同步github上面的代码。所以上网找了一下原因和解决方案，在这里记录一下。

# 原因

因为我用的是HTTPS协议，而HTTPS有一个证书是需要验证的。HTTPS协议要求server给我们发回一个证书，
而客户端负责验证这个证书是否有效。这个验证可以防止假冒网站的问题。而这个验证过程一般是在浏览器里面完成的，
而浏览器对于证书的验证一般是通过第三方受信网站来进行证书验证的。

而对于git，他使用的是`curl`，一个Linux下面的命令行浏览器。而在`curl`在访问https的网站的时候，
进行证书验证的是在`/usr/ssl/certs`里面的证书(有一些命令行浏览器是在`/etc/ssl/certs`里面存证书的)。
所以为什么我们用浏览器比如chrome或者firefox，可以正常的访问github，而用git在命令行访问就出问题，
原因就是浏览器使用的网上的证书验证，而git使用的是本地的证书验证。而我的本地并没有github的证书，
所以不能验证，从而导致了上面的错误。

# 解决方案

根据[SO](http://stackoverflow.com/a/4454754)上面的答案，我暂时是用了下面的命令行来进行解决的：

    $ env GIT_SSL_NO_VERIFY=true git clone https://github...

上面命令的前提是，你能保证你访问到的网站没有问题(其实就是人工的进行验证罢了)。
不过这不是最好的解决办法，因为这样很容易被假冒的网站(比如DNS污染)骗过。

最好的办法是能在命令行的情况下也能和浏览器一样利用第三方可信授权方，
根据这个[SO答案](http://stackoverflow.com/a/13325898)，如果你是用ubuntu或者debian的话，
就安装`ca-certificates`这个包。

    $ sudo apt-get install ca-certificates
