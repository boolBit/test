---
title: jmeter+jprofiler性能压测
date: 2017-02-03 17:25:12
tags: [性能压测, jmeter, jprofiler]
categories: 教程
---

# jmeter

jmeter是一款开源的java编写的压测工具，非常轻量级，win平台下载运行bin目录下的jmeter.bat即可启动，支持web请求，jdbc，java接口等方面的压测。

主要组件，配置介绍

1、Test Plan (测试计划)

用来描述一个性能测试，包含与本次性能测试所有相关的功能。也就说本的性能测试的所有内容是于基于一个计划的。

2、Threads （Users）线程

虽然有三个添加线程组的选项，名字不一样， 创建之后，其界面是完全一样的。之前的版本只有一个线程组的名字。现在多一个setUp theread Group 与terDown Thread Group

* setup thread group 

一种特殊类型的ThreadGroup的，可用于执行预测试操作。这些线程的行为完全像一个正常的线程组元件。不同的是，这些类型的线程执行测试前进行定期线程组的执行。

可用于执行预测试操作,每次线程执行均执行一次。

* teardown thread group. 

一种特殊类型的ThreadGroup的，可用于执行测试后动作。这些线程的行为完全像一个正常的线程组元件。不同的是，这些类型的线程执行测试结束后执行定期的线程组。

可用于执行测试后动作,每次线程执行均执行一次。

* thread group(线程组).

 这个就是我们通常添加运行的线程。通俗的讲一个线程组,，可以看做一个虚拟用户组，线程组中的每个线程都可以理解为一个虚拟用户。线程组中包含的线程数量在测试执行过程中是不会发生改变的。

```
    线程数：这里选择5

    Ramp-Up Period：单位是秒，默认时间是1秒。它指定了启动所有线程所花费的时间，比如，当前的设定表示“在5秒内启动5个线程，每个线程的间隔时间为1秒”。如果你需要Jmeter立即启动所有线程，将此设定为0即可

    循环次数：表示每个线程执行多少次请求。
```


3、测试片段（Test Fragment）

测试片段元素是控制器上的一个种特殊的线程组，它在测试树上与线程组处于一个层级。它与线程组有所不同，因为它不被执行，除非它是一个模块控制器或者是被控制器所引用时才会被执行。

控制器

JMeter有两种类型的控制器：取样器（sample）和逻辑控制器（Logic Controller），用这些原件来驱动处理一个测试。

4、取样器（Sampler）

取样器（Sampler）是性能测试中向服务器发送请求，记录响应信息，记录响应时间的最小单元，JMeter 原生支持多种不同的sampler ， 如 HTTP Request Sampler 、 FTP  Request Sampler 、TCP  Request Sampler 、 JDBC Request Sampler 等，每一种不同类型的 sampler 可以根据设置的参数向服务器发出不同类型的请求。

在Jmeter的所有Sampler中，Java Request Sampler与BeanShell Requst Sampler是两种特殊的可定制的Sampler.


5、逻辑控制器（Logic Controller）

逻辑控制器，包括两类无件，一类是用于控制test plan 中 sampler 节点发送请求的逻辑顺序的控制器，常用的有 如果（If）控制器 、 switch Controller 、Runtime Controller、循环控制器等。另一类是用来组织可控制 sampler 来节点的， 如 事务控制器、吞吐量控制器。


6、配置元件（Config Element）

配置元件（config element）用于提供对静态数据配置的支持。CSV Data Set config 可以将本地数据文件形成数据池 （Data Pool），而对应于HTTP Request Sampler和 TCP Request Sampler等类型的配制无件则可以修改 Sampler的默认数据。


7、定时器（Timer）

定时器（Timer）用于操作之间设置等待时间，等待时间是性能测试中常用的控制客户端QPS的手段。类似于LoadRunner里面的“思考时间”。 JMeter 定义了Bean Shell Timer、Constant Throughput Timer、固定定时器等不同类型的Timer。


8、前置处理器（Per Processors）

前置处理器用于在实际的请求发出之前对即将发出的请求进行特殊处理。例如，HTTP URL重写修复符则可以实现URL重写，当RUL中有sessionID 一类的session信息时，可以通过该处理器填充发出请求的实际的sessionID 。



9、后置处理器（Post Processors）

后置处理器是用于对Sampler 发出请求后得到的服务器响应进行处理。一般用来提取响应中的特定数据（类似LoadRunner测试工具中的关联概念）。例如，XPath  Extractor 则可以用于提取响应数据中通过给定XPath 值获得的数据;正则表达式提取器，则可以提取响应数据中通过正则表达式获得的数据。


10、断言（Assertions）

断言用于检查测试中得到的相应数据等是否符合预期，断言一般用来设置检查点，用以保证性能测试过程中的数据交互是否与预期一致。

11、监听器（Listener）

这个监听器可不是用来监听系统资源的元件。它是用来对测试结果数据进行处理和可视化展示的一系列元件。 图形结果、查看结果树、聚合报告、用表格察看结果都是我们经常用到的元件。




# 使用badboy录制jmeter执行计划脚本

# jprofiler

jprofiler是一个性能监控分析工具，支持cpu，内存，java方法调用栈消耗等功能，能够连接本地或者远程的java程序。

下载9.2版本，输入注册码安装后，在开始中心点击new remote integration，前三步看自己情况选择第四部选择第二个选项，输入远程地址，jprofiler目录地址，端口，结束。

按照perform modifications 阶段的提示，将本地指定目录的config.xml目录拷贝到远程jprofiler的主目录下.

修改服务器java应用所在的tomcat bin目录下，修改startup.sh 在ｅｘｅｃ最后执行前添加
CATALINA_OPTS="-agentpath:/mnt/server/jprofiler9/bin/linux-x64/libjprofilerti.so=port=8849,nowait  $CATALINA_OPTS"
export CATALINA_OPTS

服务器启动tomcat　java应用，jprofiler　attach上去即可，看到数据.


[参考](http://www.cnblogs.com/yangxia-test/p/3964881.html)
