---
title: Yarn ResourceManager调度器
description: YARN是Hadoop新版（2.x）中的资源控制框架。本文旨在深入剖析ResourceManager的调度器，探讨三种调度器的设计侧重，最后给出一些配置建议和参数解释。<br/>为了方便查阅源代码，原代码位置使用[类名:行号]方式表示。<br/>名词解释：<br/>ResourceManager：以下简称RM。YARN的中控模块，负责统一规划资源的使用。<br/>NodeManager:以下简称NM。YARN的资源结点模块，负责启动管理container。<br/>ApplicationMaster:以下简称AM。YARN中每个应用都会启动一个AM，负责向RM申请资源，请求NM启动container，并告诉container做什么事情。<br/>Container：资源容器。YARN中所有的应用都是在container之上运行的。AM也是在container上运行的，不过AM的container是RM申请的。
tags: [hadoop生态圈, java]
categories: yarn
date: 2015-12-23 20:43:58
---

## 背景

YARN是Hadoop新版（2.x）中的资源控制框架。本文旨在深入剖析ResourceManager的调度器，探讨三种调度器的设计侧重，最后给出一些配置建议和参数解释。

为了方便查阅源代码，原代码位置使用[类名:行号]方式表示。

名词解释：

ResourceManager：以下简称RM。YARN的中控模块，负责统一规划资源的使用。

NodeManager:以下简称NM。YARN的资源结点模块，负责启动管理container。

ApplicationMaster:以下简称AM。YARN中每个应用都会启动一个AM，负责向RM申请资源，请求NM启动container，并告诉container做什么事情。

Container：资源容器。YARN中所有的应用都是在container之上运行的。AM也是在container上运行的，不过AM的container是RM申请的。

## RM中的调度器

ResourceManager是YARN资源控制框架的中心模块，负责集群中所有的资源的统一管理和分配。它接收来自NM的汇报，建立AM，并将资源派送给AM。整体的RM的框架可以参考：[RM总体架构](http://hortonworks.com/blog/apache-hadoop-YARN-resourcemanager/)

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/resource_manager.png)

最初的hadoop的版本只有FifoScheduler（先进先出调度器）。当Hadoop集群在大规模使用的时候，如何整合资源和分配资源就是一个迫切的需求。对此，Yahoo!和facebook先后开发了CapacityScheduler（容量调度器）和FairScheduler（公平调度器）。在新版本中，这两个调度器在保持核心算法的基础上，也被重新开发了一次。（实际上所有整个YARN的代码都是重写的…）

## 调度器的接口

首先来了解一下调度器的工作方式和对外暴露的接口：[ResourceScheduler:36]

一个完整的调度器在内存中都会维护一个队列，应用，NM，Container的关系。同时，一个调度器也是一个事件处理器，通过RM的异步的事件调用机制知晓外部的发生的事情。要跟外部交互也要发送相应的事件。调度器一共要处理6个调度事件。


事件 | 发送时机 | 处理逻辑
--- | --- | ---
NODE_REMOVED | 一个NM被添加 | 增加总资源池的大小，修改内存状态。
NODE_ADDED | 一个NM被移除 | 删除一个NM,减少总资源池的大小，回收内存状态中在这个NM上的Container，对每个container发送KILL事件。
NODE_UPDATE | 一个NM跟RM进行心跳 | 调度器会根据当前的NM状况，在这个NM上为某一个AM分配Container，并记录这个Container的信息，留待AM获取。这个部分是调度器真正分配container的部分。后面会重点描述。
APP_ADDED | 一个新的应用被提交 | 如果接受应用，发送APP_ACCEPTED事件，否则发送APP_REJECTED事件。
APP_REMOVED | 一个应用移除 | 可能是正常或者被杀死。清除内存中这个应用的所有的container，对每个container发送KILL事件。
CONTAINER_EXPIRED | 一个container过期未被使用 | 修改内存中这个contianer相关的内存状态。

除了这6个事件，还有一个函数会在AM跟RM心跳的时候会被调用。[YARNScheduler：105]
`Allocation allocate(ApplicationAttemptId appAttemptId,List<ResourceRequest> ask,List<ContainerId> release);`

AM会告诉调度器一些资源申请的请求和已经使用完的container的列表，然后获取到已经在NODE_UPDATE分配给这个应用的container的分配。

可以看到调度器接受资源的申请和分配资源这个动作是异步的。

## 资源分配模型

无论FifoScheduler，CapacityScheduler和FairScheduler的核心资源分配模型都是一样的。

调度器维护一群队列的信息。用户可以向一个或者多个队列提交应用。每次NM心跳的时候，调度器，根据一定的规则选择一个队列，再在队列上选择一个应用，尝试在这个应用上分配资源。不过，因为一些参数限制了分配失败，就会继续选择下一个应用。在选择了一个应用之后，这个应用对应也会有很多的资源申请的请求。调度器会优先匹配本地资源是申请请求，其次是同机架的，最后的任意机器的。

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/资源分配模型.jpg)

总的来说，3种调度器就是在回答如何选择一个队列，在一个队列上如何选择一个应用的问题。

当然，实际上，比起简单的FifoScheduler。CapacityScheduler和FairScheduler还有更多新奇好玩的特性。

## 调度器比较

我们先比较下3种调度器。

调度器 | FifoScheduler | CapacityScheduler | FairScheduler 
--- | --- | --- | --- 
设计目的 | 最简单的调度器，易于理解和上手 | 多用户的情况下，最大化集群的吞吐和利用率 | 多用户的情况下，强调用户公平地贡献资源
队列组织方式 | 单队列 | 树状组织队列。无论父队列还是子队列都会有资源参数限制，子队列的资源限制计算是基于父队列的。应用提交到叶子队列。 | 树状组织队列。但是父队列和子队列没有参数继承关系。父队列的资源限制对子队列没有影响。应用提交到叶子队列。
资源限制 | 无 | 父子队列之间有容量关系。每个队列限制了资源使用量，全局最大资源使用量，最大活跃应用数量等。 | 每个叶子队列有最小共享量，最大资源量和最大活跃应用数量。用户有最大活跃应用数量的全局配置。
队列ACL限制 | 可以限制应用提交权限 | 可以限制应用提交权限和队列开关权限，父子队列间的ACL会继承。 | 可以限制应用提交权限，父子队列间的ACL会继承。但是由于支持客户端动态创建队列，需要限制默认队列的应用数量。目前，还看不到关闭动态创建队列的选项。
队列排序算法 | 无 | 按照队列的资源使用量最小的优先 | 根据公平排序算法排序
应用选择算法 | 先进先出 | 先进先出 | 先进先出或者公平排序算法
本地优先分配 | 支持 | 支持 | 支持
延迟调度 | 不支持 | 不支持 |  支持
资源抢占 | 不支持 | 不支持 | 支持，看到代码中也有实现。但是，由于本特性还在开发阶段，本文没有真实试验。

简单总结下：

**FifoScheduler：**最简单的调度器，按照先进先出的方式处理应用。只有一个队列可提交应用，所有用户提交到这个队列。可以针对这个队列设置ACL。没有应用优先级可以配置。

**CapacityScheduler：**可以看作是FifoScheduler的多队列版本。每个队列可以限制资源使用量。但是，队列间的资源分配以使用量作排列依据，使得容量小的队列有竞争优势。集群整体吞吐较大。延迟调度机制使得应用可以放弃，夸机器或者夸机架的调度机会，争取本地调度。

**FairScheduler：**多队列，多用户共享资源。特有的客户端创建队列的特性，使得权限控制不太完美。根据队列设定的最小共享量或者权重等参数，按比例共享资源。延迟调度机制跟CapacityScheduler的目的类似，但是实现方式稍有不同。资源抢占特性，是指调度器能够依据公平资源共享算法，计算每个队列应得的资源，将超额资源的队列的部分容器释放掉的特性。

## 本地优化与延迟调度


我们解释一下什么是本地优化和延迟调度。

Hadoop是构建在以hdfs为基础的文件系统之上的。YARN所运行的应用的绝大部分输入都是hdfs上的文件。而hdfs上的文件的是分块多副本存储的。假设文件系统的备份因子是3。则每一个文件块都会在3个机器上有副本。在YARN运行应用的时候，AM会将输入文件进行切割，然后，AM向RM申请的container来运行task来处理这些被切割的文件段。


假如输入文件在ABC三个机器上有备份，那如果AM申请到的container在这3个机器上的其中一个，那这个task就无须从其它机器上传输要处理的文件段，节省网络传输。这就是Hadoop的本地优化。所以Hadoop的文件备份数量除了和数据安全有关，还对应用运行效率有莫大关系。

YARN的实现本地优化的方式是AM给RM提交的资源申请的时候，会同时发送本地申请，机架申请和任意申请。然后，RM的匹配这些资源申请的时候，会先匹配本地申请，再匹配机架申请，最后才匹配任意申请。关于AM资源申请机制可以参考：[ContainerAlloctor分析](http://dongxicheng.org/mapreduce-nextgen/yarnmrv2-mrappmaster-containerallocator/)

而延迟调度机制，就是调度器在匹配本地申请失败的时候，匹配机架申请或者任意申请成功的时候，允许略过这次的资源分配，直到达到延迟调度次数上限。CapacityScheduler和FairScheduler在延迟调度上的实现稍有不同，前者的调度次数是根据规则计算的，后者的调度次数通过配置指定的，但实际的含义是一样的。


## 参数配置

### 调度器的集群配置

学习中，请稍等。。。