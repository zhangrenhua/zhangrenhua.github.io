---
title: Yarn任务提交流程
description: Yarn是随着hadoop发展而催生的新框架，全称是Yet Another Resource Negotiator，可以翻译为“另一个资源管理器”。yarn在`hadoop2.0`开始取代了以前hadoop中jobtracker（后面简写JT）的角色，因为以前JT的 任务过重，负责任务的调度、跟踪、失败重启等过程，而且只能运行mapreduce作业，不支持其他编程模式，这也限制了JT使用范围，而yarn应运而 生，解决了这两个问题。
tags: [hadoop生态圈, java]
categories: yarn
date: 2015-12-30 20:00:33
---

## 背景

Yarn是随着hadoop发展而催生的新框架，全称是Yet Another Resource Negotiator，可以翻译为“另一个资源管理器”。yarn在`hadoop2.0`开始取代了以前hadoop中jobtracker（后面简写JT）的角色，因为以前JT的 任务过重，负责任务的调度、跟踪、失败重启等过程，而且只能运行mapreduce作业，不支持其他编程模式，这也限制了JT使用范围，而yarn应运而 生，解决了这两个问题。

## Yarn(MRv2)

Yarn把jobtracker的任务分解开来，分为：

ResourceManager（简写RM）负责管理分配全局资源
ApplicationMaster（简写AM），AM与每个具体任务对应，负责管理任务的整个生命周期内的所有事宜
除了上面两个以外，tasktracker被NodeManager（简写NM）替代，RM与NM构成了集群的计算平台。这种设计允许NM上长期运行一些辅助服务，这些辅助服务一般都是应用相关的，通过配置项指定，在NM启动时加载。例如在yarn上运行mapreduce程序时，shuffle就 是一个由NM加载起来的辅助服务。需要注意的是，在hadoop 0.23之前的版本，shuffle是tasktracker的一部分。

与每个应用相关的AM是一个框架类库，它与RM沟通协商如何分配资源，与NM协同执行并且监测应用的执行情况。在yarn的设计 中，mapreduce只是一种编程模式，yarn还允许像MPI(message passing interface)，Spark等应用构架部署在yarn上运行。

## Yarn设计

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-yarn-jgt.jpg)

上图是一个典型的YARN集群。可以看到RM有两个主要服务：

可插拔的Scheduler，只负责用户提交任务的调度
ApplicationsMaster的（简写AsM）负责管理集群中每个任务的ApplicationMaster（简写AM），负责任务的监控、失败重起等
在hadoop1.0时，资源分配的单位是slot，再具体分为map的slot与reduce的slot，而且这些slot的个数是在任务运行前 事先定义的，在任务运行过程中不能改变，很明显，这会造成资源的分配不均问题。在haodop2.0中，yarn采用了container的概念来分配资 源。每个container由一些可以动态改变的属性组成，到现在为止，仅支持内存、cpu两种。但是yarn的这种资源管理方式是通用的，社区以后会加 入更多的属性，比如网络带宽，本地硬盘大小等等。

## Yarn主要组件

在这小节里，主要介绍yarn各个组件，以及他们之间是如何通信的。

### Client<—>RM

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-yarn-rm.jpg)

上面这个图是Client向RM提交任务时的流程。
(1) Client通过New Application Request来通知RM中的AsM组建
(2) AsM一般会返回一个新生成的全局ID，除此之外，传递的信息还有集群的资源状况，这样Client就可以在需要时请求资源来运行任务的第一个container即AM。
(3) 之后，Client就可以构造并发送ASC了。ASC中包括了调度队列，优先级，用户认证信息，除了这些基本的信息之外，还包括用来启动AM的CLC信息，一个CLC中包括jar包、依赖文件、安全token，以及运行任务过程中需要的其他文件。

经过上面这三步，一个Client就完成了一次任务的提交。之后，Client可以直接通过RM查询任务的状态，在必要时，可以要求RM杀死这个应用。如下图：

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-yarn-rm-1.jpg)

### RM<—>AM

RM在收到Client端发送的ASC后，它会查询是否有满足其资源要求的container来运行AM，找到后，RM会与那个container所在机器上的NM通信，来启动AM。下面这个图描述了这其中的细节。

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-yarn-am-1.png)

(1) AM向RM注册，这个过程包括handshaking过程，并且传递一些信息，包括AM监听的RPC端口、用于监测任务运行状态的URL等。
(2) RM中的Scheduler部件做回应。这个过程会传递AM所需的信息，比如这个集群的最大与最小资源使用情况等。AM利用这些信息来计算并请求任务所需的资源。
(3) 这个过程是AM向RM请求资源。传递的信息主要包含请求container的列表，还有可能包含这个AM已经释放的container的列表。
(4) 在AM经过(3)请求资源之后，在稍微晚些时候，会把心跳包与任务进度信息发送给RM
(5)Scheduler在收到AM的资源请求后，会根据调度策略，来分配container以满足AM的请求。
(6) 在任务完成后，AM会给RM发送一个结束消息，然后退出。

<font color="red">在上面(5)与(6)之间，AM在收到RM返回的container列表后，会与每个container所在机器的NM通信，来启动这个container</font>，下面就说说这个过程。

### AM<—>NM

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-yarn-an-1.png)

(1) AM向container所在机器的NM发送CLC来启动container
(2)(3) 在container运行过程中，AM可以查询它的运行状态

## API

通过上面的描述，开发者在开发YARN上的应用时主要需要关注以下接口：
ApplicationClientProtocol，Client使用这个协议来与RM通信，来启动一个新应用，检查任务的运行状态或杀死任务

ApplicationMasterProtocol，AM使用这个协议来向RM注册/撤销，请求资源来运行任务。

ContainerManagementProtocol，AM使用这个协议来与NM通信，来启动/停止container，查询container的状态。

## 总结

用户在使用hadoop1.0 API编写的MapReduce可以不用修改直接运行在yarn上，不过随着yarn的发展，向后兼容性还不知道怎么样。不管怎样，新的yarn平台绝对值得我们使用。类似的资源管理框架还有`mesos`、`coraca`、`Torca`、`Omega`。

## 参考
- http://www.mamicode.com/info-detail-852263.html