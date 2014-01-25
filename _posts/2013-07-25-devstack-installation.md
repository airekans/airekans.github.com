---
layout: post
title: "Installing Devstack On Ubuntu"
description: "这篇文章介绍了Devstack在ubuntu server上面的安装步骤。大致步骤和官方文档上面的都差不多，不过有些小问题和对应的解决办法。"
category: cloud-computing
tags: [cloud computing, OpenStack, distributed system]
---
{% include JB/setup %}

# What Is Devstack?

[Devstack](http://www.openstack.org/)是开源云计算IaaS项目[OpenStack](http://www.openstack.org/)的一个开发者版本。
Devstack能够在单一的PC上面部署，使得开发者在开发的过程中能够方便的部署和测试。
这对于开发者或者是想入门的人来说都非常方便。本文将介绍如何在Ubuntu x86-64 server
上面安装Devstack。其实这样的文章网上也有一些，不过在我安装的过程中遇到一些问题，
在网上也有遇到过，而且还没有看到有人把解决方法详细讲解，所以在这里记录一下。

# 安装环境

我的环境在Ubuntu desktop 12.04上面装了VirtualBox，然后建了一个虚拟机。
整个Devstack是在虚拟机里面安装的。虚拟机的配置如下：

- 64位CPU
- 2G内存
- 40G硬盘
- Ubuntu 12.04.2 x64 Server
- 网络是使用的Bridge Adapter

系统安装什么的，全部按默认的来，硬盘分区也是用的最原始的原始分区，没有用LVM。安装Devstack的时候用户是一个有sudo权限的非root用户，同时网速应该保持比较好的水平。
装好之后。接下来就是Devstack了。

# 安装Devstack

按照Devstack的[官方文档](http://devstack.org/guides/single-vm.html)，其实就是下面的几个命令：

{% highlight bash %}
apt-get install -qqy git
git clone https://github.com/openstack-dev/devstack.git
cd devstack
./stack.sh
{% endhighlight %}

如上，在开始运行了`stack.sh`之后，会提示输入几个相关部件的密码。
这里我都输入同一个，假设是`123456`。

如果网络不错的话，在安装完一些默认的包之后，会走到keystone的设置。
这个时候，`stack.sh`抱了下面的错误：

    ++ keystone service-create --name keystone --type identity --description 'Keystone Identity Service'
    Unable to communicate with identity service: {"error": {"message": "An unexpected error prevented the server from fulfilling your request. (OperationalError) (1045, \"Access denied for user 'root'@'localhost' (using password: YES)\") None None", "code": 500, "title": "Internal Server Error"}}. (HTTP 500)

从错误信息里面大致能够看出，是一个权限错误。而`keystone`本身是一个OpenStack里面的身份验证服务，后台使用数据库作为数据的存储。
在我的环境里是用了MySQL作为DB Backend的。
所以尝试了一下用`root`用户登录MySQL，的确是没有办法登录，错误也和上面的错误是一样的。看来是我密码没有初始化好？总之`keystone`估计是没有办法登录数据库，从而造成了错误了。
所以我只有充值MySQL的root密码了。
从[这里](http://dev.mysql.com/doc/refman/5.0/en/resetting-permissions.html)查到了重置密码的方法，所以按照里面的方法重置了root密码，密码也是前面设置的`123456`。

重置完之后，重新跑一下`stack.sh`，等个10来分钟左右，就会看到下面的输出：

    Horizon is now available at http://192.168.1.108/
    Keystone is serving at http://192.168.1.108:5000/v2.0/
    Examples on using novaclient command line is in exercise.sh
    The default users are: admin and demo
    The password: 123456
    This is your host ip: 192.168.1.108
    stack.sh completed in 760 seconds.

如果你看到了这一行，说明devstack已经完全配置好，已经启动起来了。现在你用浏览器打开`http://192.168.1.108`就能够看到OpenStack的管理界面了。现在可以开始折腾了！

## 重新启动OpenStack

当你已经正常启动过一次Devstack之后，下次想启动Devstack就可以不用在下载需要的软件包了。
只要在`localrc`里面加入下面的语句就可以：

    OFFLINE=True

然后你再跑`stack.sh`，就可以完全在无网络的环境下启动Devstack了

最后，希望大家能玩得开心。
