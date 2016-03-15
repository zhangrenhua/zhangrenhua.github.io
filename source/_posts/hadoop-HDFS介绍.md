---
title: hadoop-HDFS介绍
description: Hadoop Distributed File System，简称HDFS，是一个分布式文件系统。HDFS是一个主/从（Mater/Slave）体系结构，从最终用户的角度来看，它就像传统的文件系统一样，可以通过目录路径对文件执行CRUD（Create、Read、Update和Delete）操作。<br/><br/>由于分布式存储的性质，HDFS集群拥有一个NameNode和多个DataNode。NameNode管理文件系统的元数据，DataNode存储实际的数据。客户端通过同NameNode和DataNodes的交互访问文件系统。客户端联系NameNode以获取文件的元数据，而真正的文件I/O操作是直接和DataNode进行交互的。<br/><img alt="Alt text" src="http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hdfs-jgt.png">
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-12-10 21:00:00
---

## 背景
Hadoop Distributed File System，简称HDFS，是一个分布式文件系统。HDFS是一个主/从（Mater/Slave）体系结构，从最终用户的角度来看，它就像传统的文件系统一样，可以通过目录路径对文件执行CRUD（Create、Read、Update和Delete）操作。

由于分布式存储的性质，HDFS集群拥有一个NameNode和多个DataNode。NameNode管理文件系统的元数据，DataNode存储实际的数据。客户端通过同NameNode和DataNodes的交互访问文件系统。客户端联系NameNode以获取文件的元数据，而真正的文件I/O操作是直接和DataNode进行交互的。
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hdfs-jgt.png)

Hadoop的设计思想受到Google公司的GFS设计思想的启示，基于一种开源的理念实现的分布式分布式文件系统。HDFS的设计基础与目标如下：

1、硬件错误（Hardware Failure）是常态，因而需要数据冗余技术。

2、流失数据访问（Streaming Data Access），即数据批量读取而非随机读写，Hadoop擅长做数据分析而不是事务处理。

3、大规模数据集（Large Data Sets）。

4、简单一致性模型（Simple Coherency Model），即为了降低系统复杂度，对文件采用一次性写多次读的逻辑设计，也就是文件一经写入，关闭，就再不要修改。

5、“Moving Computation is Cheaper than Moving Data”，通俗理解，程序采用“数据就近”原则分配节点执行。

6、Portability Across Heterogeneous Hardware and Software Platforms，即有着很强的可扩展性。

## HDFS特点
**基于廉价的普通硬件，可以容忍硬件出错**
系统中的某一台或几台服务器出现故障的时候，系统仍可用且数据保持完整

**大数据集（大文件）**
HDFS适合存储大量文件，总存储量可以达到PB,EB级
HDFS适合存储大文件，单个文件大小一般在百MB级之上
文件数目适中，不适合处理大量的小文件

**简单的一致性模型**
HDFS应用程序需要一次写入，多次读取一个文件的访问模式
支持追加(append)操作，但无法更改已写入数据

**顺序的数据流访问**
HDFS适合用于处理批量数据，而不适合用于随机定位访问
侧重高吞吐量的数据访问，可以容忍数据访问的高延迟
为把“计算”移动到“数据”提供基础和便利
实现分布式文件存储的分布式计算

## HDFS一些概念

**元数据 - NameNode**
HDFS通过NameNode存储文件系统的元数据，元数据一般常驻内存，以便提供高速的查询交互
1G内存大约可存放100万个数据块的元数据信息
文件系统的信息包括：文件系统的目录信息、文件和块的对应关系、块的存放位置和块信息

**NameNode的元数据持久化**：fsImage文件保存文件系统的持久化信息；edits保存更改记录日志（类似数据库中的RedoLog）

**块-DataNode**
HDFS通过DataNode存储数据块信息，每个文件将按固定大小划分成多个块（默认64M）
文件在HDFS中存储时，被划分成多个数据块，分别存储到集群多个不同的DataNode节点中
每个数据块会存在多个数据副本，存放到不同的DataNode中（默认3份）
每个块会在本地文件系统产生两个文件，一个是实际的数据文件，另一个是块的附加信息文件，其中包括数据的校验和，生成时间

**客户端**
命令行客户端
API客户端
    - Java接口
    - 其他语言访问接口

## HDFS体系架构

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hdfs-jgt.png)

**NameNode**
NameNode可以看作是分布式文件系统中的管理者，主要负责管理文件系统的命名空间、集群配置信息和存储块的复制等。
NameNode会将文件系统的Meta-data存储在内存中，这些信息主要包括了文件信息、每一个文件对应的文件块的信息和每一个文件块在DataNode的信息等。

**Secondary NameNode**
Secondary NameNode`不是备份节点`
Secondary NameNode 的主要的工作是`阶段性的合并fsimage`和`edits`文件，并将fsImage进行回传。以此来控制`edits`的`文件大小在合理的范围`。
在NameNode硬盘损坏的情况下，Secondary NameNode也可用作数据恢复，但绝不是全部。只能手工恢复。


**DataNode**
DataNode是文件存储的基本单元，它将Block存储在本地文件系统中，保存了Block的Meta-data，同时周期性地将所有存在的Block信息发送给NameNode。

**文件写入**
1) Client向NameNode发起文件写入的请求。 
2) NameNode根据文件大小和文件块配置情况，返回给Client它所管理部分DataNode的信息。
3) Client将文件划分为多个Block，根据DataNode的地址信息，按顺序写入到每一个DataNode块中。

**文件读取**
1) Client向NameNode发起文件读取的请求。 
2) NameNode返回文件存储的DataNode的信息。 
3) Client读取文件信息。 

## HDFS文件冗余存储技术

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hdfs-rycc.png)

NameNode中记录文件的存储路径、记录每个数据文件会被划分成N个数块每个数据块的副本数量、每个数据块存放到哪些DataNode节点中。
一般数据文件将按64M的大小划分数据块，每个数据块会存放3份副本，每个副本都分别存放到不同的DataNode节点中。
当进行数据该问时，可以通过拼装不同结点上的数据块，组装成完整的数据文件。
好处：
1、采用这种方式，可以保证在DataNode某个节点损坏时，数据不会丢失
2、当进行分布计算时，可以在每个DataNode上启用多个数据处理进程，同时进行数据处理。

## HDFS高可用-NFS方式

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hdfs-ha-nfs.png)

NFS，是Network File System的简写，即网络文件系统
通过共享NameNode的数据存储，实现高可用

通过zookeeper监视NameNode状态，实现动态热切换

## HDFS高可用-QJM方式

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hdfs-ha-qjm.png)

通过JournalNode实现NameNode间的日志传输和同步，保证命名节点的数据备份
Quorum Journal Manage：仲裁日志管理。

## 参考资料
帮胜 ——《hadoop技术介绍》