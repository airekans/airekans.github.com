---
layout: post
title: "OpenStack Tempest整体剖析"
description: "从源码角度对OpenStack里面的测试项目Tempest进行整体的剖析"
category: cloud-computing
tags: [cloud computing, OpenStack, distributed computing]
---
{% include JB/setup %}

# Tempest是什么项目

Tempest是一个OpenStack的测试集，主要是用来对OpenStack的API做smoke test以及压力测试，也包含了对CLI client的测试和场景测试。

Tempest使用nose来驱动，其测试的主要风格是按照pyunit来写的，同时使用了testtools和testresources等几个测试工具库。

# 如何使用Tempest

要使用tempest来测试一个搭建好的OpenStack环境，首先要有一个设置了各个OpenStack参数的配置文件供tempest使用，在etc/文件夹下有个tempest.conf.sample供参考使用。
有了配置文件之后，就可以直接通过nosetests tempest命令来跑所有的测试了，也可以通过指定tempest包里某个测试类来单独跑某一个测试。

## Tempest的结构

Tempest的文件结构主要是下面这样：

    tempest
    ├── api   # API的测试集
    ├── cli  # OpenStack的命令行工具测试集
    ├── common  # 一些公共的工具类和函数
    ├── scenario   # 对OpenStack的常用场景进行测试，包括基本的启动VM，挂载volumn和网络配置等
    ├── services  # tempest自己实现的OpenStack API Client，自己实现是为了不让一些bug隐藏在官方实现的Client里面。
    ├── stress  # 压力测试集，利用multiprocessing来启动多个进程来同时对OpenStack发起请求。
    ├── thirdparty  # EC2兼容的API测试集
    ├── whitebox   # 白盒测试集，主要是对DB操作，然后发起请求，然后比对结果

其中tempest是一个顶层目录，下面各个目录包含的文件主要是上面说的功能。

`tempest.api`、`tempest.scenario`、`tempest.thirdparty`和`tempest.whitebox`里面的测试类都是基于`tempest.test.BaseTestCase`。
`BaseTestCase`声明了`config`属性，也就是读取配置文件类，还声明了`setUpClass`方法，在类初始化的时候调用。
`BaseTestCase`的子类`tempest.test.TestCase`就声明了很多工具函数，供它的子类调用。包括`setUpClass`(初始化OpenStack的各个服务的Client并设置成类的属性)，资源管理函数(`get/set/remove_resource`)和`status_timeout`(等待资源到达某个期望的状态)。

有了上面的工具，测试就可以比较方便的编写。

下面介绍一下tempest里面主要的几个package。

### tempest.api

这个package包含了OpenStack几乎所有native API的测试。每个一个服务都自己有一个独立的包，比如`tempest.api.compute`。
下面以`tempest.api.compute`作为例子。

每一个测试，都有两个实现，一个是测试JSON格式，一个是测试XML格式的。这个是通过类的`_interface`属性类设置。而在基类`BaseComputeTest`里面，会利用这个属性构造不同的API实现。不过目前XML格式的测试基本上都是空的实现，所以主要的测试都是在JSON格式上。

以`tempest.api.compute.flavors.test_flavors`为例，`FlavorTestJson`继承了`BaseComputeTest`，所以在类初始化的时候，就会把tempest自己实现的API client赋值给类的属性。然后在具体的测试函数里面，`FlavorTestJson`就利用这个client的函数来对OpenStack进行查询，并且验证查询的结果。
如下面的函数：

{% highlight python %}@attr(type='smoke')
def test_list_flavors_with_detail(self):
    # Detailed list of all flavors should contain the expected flavor
    resp, flavors = self.client.list_flavors_with_detail()
    resp, flavor = self.client.get_flavor_details(self.flavor_ref)
    self.assertTrue(flavor in flavors){% endhighlight %}

上面就是利用flavorclient来获取所有的flavor列表，然后再取具体的某个flavor，然后验证这个flavor的确是在所有的flavor里面的。
注意到这个函数是用`attr`修饰器进行修饰的，这个修饰器是`tempest.test.attr`，它利用nose和testtools里面类似的功能，给不同的test打上tag，这样在跑测试的时候可以通过tag来进行筛选，如跑gate测试，或者跑smoke测试。

而上面用的Client是tempest自己实现的RESTful API client，他们实现在`tempest.services`里面，是利用`httplib2`来实现的简单RESTful client。

### tempest.scenario

`scenario`包含了几个简单的OpenStack完整的使用场景，来对OpenStack进行集成测试。也是初学者对于整个OpenStack的使用进行初步了解的一个入口。

每个场景测试类都继承于`tempest.scenario.manager.OfficialClientTest`，而`OfficialClientTest`本身又继承于`tempest.test.TestCase`。`OfficialClientTest`的特殊之处在于他的所有API Client都是官方的client而不是tempest自己实现的client。而且它声明了`tearDownClass`，在类销毁的时候会将所有已经申请的资源都删除掉，以达到每个测试集都是独立的效果。
而每个测试集都会在申请资源之后利用`TestCase`的接口向类里面注册资源，这样`OfficialClientTest`就可以自动的将注册过的资源释放了。

一个典型的场景测试是测试创建VM，然后挂载volumn，然后ssh上VM取看看是否挂载成功：

{% highlight python %}def test_minimum_basic_scenario(self):
    self.glance_image_create()
    self.nova_keypair_add()
    self.nova_boot()
    self.nova_list()
    self.nova_show()
    self.cinder_create()
    self.cinder_list()
    self.cinder_show()
    self.nova_volume_attach()
    self.cinder_show()
    self.nova_reboot()

    self.nova_floating_ip_create()
    self.nova_floating_ip_add()
    self.nova_security_group_rule_create()
    self.ssh_to_server()
    self.check_partitions()

    self.nova_volume_detach(){% endhighlight %}

