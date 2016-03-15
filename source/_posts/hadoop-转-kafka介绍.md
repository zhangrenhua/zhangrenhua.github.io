---
title: 转-kafka介绍
description: Kafka是Linkedin于2010年12月份开源的消息系统，它主要用于处理活跃的流式数据。活跃的流式数据在web网站应用中非常常见，这些数据包括网站的pv、用户访问了什么内容，搜索了什么内容等。这些数据通常以日志的形式记录下来，然后每隔一段时间进行一次统计处理。</br>传统的日志分析系统提供了一种离线处理日志信息的可扩展方案，但若要进行实时处理，通常会有较大延迟。而现有的消息（队列）系统能够很好的处理实时或者近似实时的应用，但未处理的数据通常不会写到磁盘上，这对于Hadoop之类（一小时或者一天只处理一部分数据）的离线应用而言，可能存在问题。Kafka正是为了解决以上问题而设计的，它能够很好地离线和在线应用。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2016-2-29 21:32:11
---

## 概述

Kafka是Linkedin于2010年12月份开源的消息系统，它主要用于处理活跃的流式数据。活跃的流式数据在web网站应用中非常常见，这些数据包括网站的pv、用户访问了什么内容，搜索了什么内容等。这些数据通常以日志的形式记录下来，然后每隔一段时间进行一次统计处理。

传统的日志分析系统提供了一种离线处理日志信息的可扩展方案，但若要进行实时处理，通常会有较大延迟。而现有的消息（队列）系统能够很好的处理实时或者近似实时的应用，但未处理的数据通常不会写到磁盘上，这对于Hadoop之类（一小时或者一天只处理一部分数据）的离线应用而言，可能存在问题。Kafka正是为了解决以上问题而设计的，它能够很好地离线和在线应用。

## 介绍

Kafka是LinkedIn开发并开源出来的一个高吞吐的分布式消息系统。其具有以下特点：
1）支持高Throughput的应用
2）scale out：无需停机即可扩展机器
3）持久化：通过将数据持久化到硬盘以及replication防止数据丢失
4）持online和offline的场景。

kafka使用scala开发，支持多语言客户端（c++、java、python、go等）其架构如下：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-kafka-jg.jpg)

Producer：消息发布者
Broker：消息中间件处理结点，一个kafka节点就是一个broker
Consumer：消息订阅者

kafka的消息分几个层次：
1）Topic：一类消息，例如page view日志，click日志等都可以以topic的形式存在，kafka集群能够同时负责多个topic的分发

2）Partition： Topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。partition中的每条消息都会被分配一个有序的id（offset）

3）Message：消息，最小订阅单元


具体流程：

1、Producer根据指定的partition方法（round-robin、hash等），将消息发布到指定topic的partition里面；
2、kafka集群接收到Producer发过来的消息后，将其持久化到硬盘，并保留消息指定时长（可配置），而不关注消息是否被消费；
3、Consumer从kafka集群pull数据，并控制获取消息的offset。

## 设计

**ThroughPut**
High Throughput是kafka需要实现的核心目标之一，为此kafka做了以下一些设计：
1）数据磁盘持久化：消息不在内存中cache，直接写入到磁盘，充分利用磁盘的顺序读写性能
2）zero-copy：减少IO操作步骤
3）数据批量发送
4）数据压缩
5）Topic划分为多个partition，提高parallelism

**load balance&HA**
1）producer根据用户指定的算法，将消息发送到指定的partition
2）存在多个partiiton，每个partition有自己的replica，每个replica分布在不同的Broker节点上
3）多个partition需要选取出lead partition，lead partition负责读写，并由zookeeper负责fail over
4）通过zookeeper管理broker与consumer的动态加入与离开

**pull-based system**
由于kafka broker会持久化数据，broker没有内存压力，因此，consumer非常适合采取pull的方式消费数据，具有以下几点好处：
1）简化kafka设计
2）consumer根据消费能力自主控制消息拉取速度
3）consumer根据自身情况自主选择消费模式，例如批量，重复消费，从尾端开始消费等

**Scale Out**
当需要增加broker结点时，新增的broker会向zookeeper注册，而producer及consumer会根据注册在zookeeper上的watcher感知这些变化，并及时作出调整。

## 设计目标

（1）数据在磁盘上存取代价为O(1）。一般数据在磁盘上是使用BTree存储的，存取代价为O（lgn）。
（2）高吞吐率。即使在普通的节点上每秒钟也能处理成百上千的message。
（3）显式分布式，即所有的producer、broker和consumer都会有多个，均为分布式的。
（4）支持数据并行加载到Hadoop中。

## KafKa部署结构

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-kafka-deployment.jpg)

kafka是显式分布式架构，producer、broker（Kafka）和consumer都可以有多个。Kafka的作用类似于缓存，即活跃的数据和离线处理系统之间的缓存。几个基本概念：
（1）message（消息）是通信的基本单位，每个producer可以向一个topic（主题）发布一些消息。如果consumer订阅了这个主题，那么新发布的消息就会广播给这些consumer。
（2）Kafka是显式分布式的，多个producer、consumer和broker可以运行在一个大的集群上，作为一个逻辑整体对外提供服务。对于consumer，多个consumer可以组成一个group，这个message只能传输给某个group中的某一个consumer.

## KafKa关键技术点

### zero-copy

在Kafka上，有两个原因可能导致低效：1）太多的网络请求 2）过多的字节拷贝。为了提高效率，Kafka把message分成一组一组的，每次请求会把一组message发给相应的consumer。 此外， 为了减少字节拷贝，采用了sendfile系统调用。为了理解sendfile原理，先说一下传统的利用socket发送文件要进行拷贝：

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-kafka-zero-copy02.gif)

Sendfile系统调用：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-kafka-zero-copy11.gif)

###  Exactly once message transfer

怎样记录每个consumer处理的信息的状态？在Kafka中仅保存了每个consumer已经处理数据的offset。这样有两个好处：1）保存的数据量少 2）当consumer出错时，重新启动consumer处理数据时，只需从最近的offset开始处理数据即可。

### Push/pull

Producer 向Kafka（push）推数据，consumer 从kafka 拉（pull）数据。

### 负载均衡和容错

Producer和broker之间没有负载均衡机制。
broker和consumer之间利用zookeeper进行负载均衡。所有broker和consumer都会在zookeeper中进行注册，且zookeeper会保存他们的一些元数据信息。如果某个broker和consumer发生了变化，所有其他的broker和consumer都会得到通知。

## 参考资料

- Kafka主页：http://sna-projects.com/kafka/design.php
- Zero-copy原理：https://www.ibm.com/developerworks/linux/library/j-zerocopy/
- Kafka与Hadoop：http://sna-projects.com/sna/media/kafka_hadoop.pdf


转自：http://dongxicheng.org/search-engine/kafka/