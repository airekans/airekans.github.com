---
layout: post
title: "git clone from github failed"
description: ""
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

因为我用的是HTTPS协议，所以HTTPS有一个证书是需要验证的
