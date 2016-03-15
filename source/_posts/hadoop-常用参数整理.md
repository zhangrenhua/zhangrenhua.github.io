---
title: Hadoop常用参数整理（HDFS/Yarn/MapReduce/GC）
description: 最近整理了一些常用的`Hadoop`集群相关配置参数，并对相关参数进行调优，其中有一些参数能影响到集群的性能等问题，需谨慎使用。本来想在这里把`Hbase`参数也带上的，由于`Hbase`参数调优细节太多，所以单独抽了出来，在未来的几天内我会补上。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2016-1-5 19:22:09
---

## 背景

最近整理了一些常用的`Hadoop`集群相关配置参数，并对相关参数进行调优，其中有一些参数能影响到集群的性能等问题，需谨慎使用。本来想在这里把`Hbase`参数也带上的，由于`Hbase`参数调优细节太多，所以单独抽了出来，在未来的几天内我会补上。


## HDFS

Hadoop HDFS组件的相关参数说明，本文没有说明某个参数对应哪个配置文件（笔者使用商业版集群，所以就没有一一去找）：

参数名称 | 参数默认值 | 描述 
--- | --- | --- 
dfs.block.size,dfs.blocksize | 134217728 |  以字节计算的新建 HDFS 文件默认块大小。请注意该值也用作 HBase 区域服务器 HLog 块大小。
dfs.replication | 3 |  HDFS文件的数据块复制份数。
dfs.webhdfs.enabled | TRUE |  启用 WebHDFS 界面，启动50070端口。
dfs.permissions | TRUE |  HDFS文件权限检查。
dfs.datanode.failed.volumes.tolerated | 0 | 能够导致DN挂掉的坏硬盘最大数，默认0就是只要有1个硬盘坏了，DN就会shutdown。
dfs.data.dir,dfs.datanode.data.dir | xxx,xxx | DataNode数据保存路径，可以写多块硬盘，逗号分隔
dfs.name.dir,dfs.namenode.name.dir | xxx,xxx |  NameNode本地元数据存储目录，可以写多块硬盘，逗号分隔
fs.trash.interval | 1 | 垃圾桶检查频度（分钟）。要禁用垃圾桶功能，请输入0。
dfs.safemode.min.datanodes | 0 | 指定在名称节点存在 safemode 前必须活动的 DataNodes 数量。输入小于或等于 0 的值，以在决定启动期间是否保留 safemode 时将活动的 DataNodes 数量考虑在内。值大于群集中 DataNodes 的数量时将永久保留 safemode。
dfs.client.read.shortcircuit | TRUE | 启用 HDFS short circuit read。该操作允许客户端直接利用 DataNode 读取 HDFS 文件块。这样可以提升本地化的分布式客户端的性能
dfs.datanode.handler.count | 3 | DataNode 服务器线程数。默认为3,较大集群,可适当调大些,比如8。
dfs.datanode.max.xcievers, dfs.datanode.max.transfer.threads | 256 | 指定在 DataNode 内外传输数据使用的最大线程数，datanode在进行文件传输时最大线程数
dfs.balance.bandwidthPerSec, dfs.datanode.balance.bandwidthPerSec | 1048576 | 每个 DataNode 可用于平衡的最大带宽。单位为字节/秒。

上面参数中可能有2个名称，前面一个是老版本1.x的后面的是新版本2.x的。

### 参数建议

**dfs.data.dir, dfs.datanode.data.dir**

DataNode 存储 HDFS 块数据的本地文件系统中的目录逗号分隔列表。值通常为 /data/N/dfs/dn for N = 1, 2, 3...，应使用 noatime 选项装载这些目录并使用 JBOD 配置磁盘。不推荐使用 RAID。


**dfs.name.dir, dfs.namenode.name.dir**

NameNode本地元数据存储目录,<font color="red">如果这个参数设置为多个目录，那么这些目录下都保存着元信息的多个备份。</font>


**dfs.client.read.shortcircuit**

启用 HDFS short circuit read。该操作允许客户端直接利用 DataNode 读取 HDFS 文件块。这样可以提升本地化的分布式客户端的性能


**dfs.datanode.handler.count**

DataNode 服务器线程数。默认为3,较大集群,可适当调大些,比如8


**dfs.datanode.max.xcievers, dfs.datanode.max.transfer.threads**

相当于linux下的打开文件最大数量，可适当调大些，比如4096。


**dfs.balance.bandwidthPerSec, dfs.datanode.balance.bandwidthPerSec**

balancer时，hdfs移动数据的速度，默认值为1M/S的速度。一般情况下设置为20M；设置的过大会影响当前job的运行。

### 进程内存设置

以下是机器128G内存下的参数配置：

DataNode 的 Java 堆栈大小：8G
Namenode 的 Java 堆栈大小：32GB，可以更大
dfs.datanode.max.locked.memory（DataNode 可用于缓存内存中的数据块的最大内存量。将其设置为零将禁用缓存。）：4G


## Yarn

Hadoop Yarn组件的相关参数说明，本文没有说明某个参数对应哪个配置文件（笔者使用商业版集群，所以就没有一一去找）：

参数名称 | 参考值 | 描述 
--- | --- | --- 
yarn.scheduler.minimum-allocation-mb | 1024 | 容器可以请求的最小物理内存量（以 MiB 为单位）。如果使用 Capacity 或 FIFO scheduler（或任何 CDH 5 之前的 scheduler），内存请求数量将四舍五入为该数字最接近的倍数。
yarn.scheduler.maximum-allocation-mb | 51200 | 可为容器请求的最大物理内存数量（以 MiB 为单位）。
yarn.nodemanager.resource.memory-mb | 51200 | 可分配给容器的物理内存数量（以 MiB 为单位）。
yarn.scheduler.fair.continuous-scheduling-enabled | TRUE | 启用公平调度
yarn.log-aggregation.retain-seconds | 604800 | 保存汇聚日志时间，秒，超过会删除，-1表示不删除。 注意，设置的过小，将导致NN垃圾碎片。建议3-7天 = 7 * 86400 = 604800
yarn.nodemanager.log.retain-seconds | 10800 | 保留用户日志的时间（以秒为单位）。仅适用在日志聚合已禁用的情况下。
yarn.app.mapreduce.am.job.client.port-range |  | MR AM能够绑定使用的端口范围。例如：50000-50050,50100-50200。 如果你先要全部的有用端口，可以留空（默认值null）。

关于Yarn组件主要介绍上面几个参数，参数配置需根据自己集群环境来决定。

### 参数建议

**yarn.nodemanager.resource.memory-mb**

表示该节点上YARN可使用的物理内存总量，默认是8192（MB），注意，如果你的节点内存资源不够8GB，则需要调减小这个值，而YARN不会智能的探测节点的物理内存总量


**yarn.scheduler.maximum-allocation-mb**

单个任务可申请的最多物理内存量，默认是8192（MB）。
默认情况下，YARN采用了线程监控的方法判断任务是否超量使用内存，一旦发现超量，则直接将其杀死。由于Cgroups对内存的控制缺乏灵活性（即任务任何时刻不能超过内存上限，如果超过，则直接将其杀死或者报OOM），而Java进程在创建瞬间内存将翻倍，之后骤降到正常值，这种情况下，采用线程监控的方式更加灵活（当发现进程树内存瞬间翻倍超过设定值时，可认为是正常现象，不会将任务杀死），因此YARN未提供Cgroups内存隔离机制。
可以使用如下命令在提交任务时动态设置：
```
hadoop jar <jarName> -D mapreduce.reduce.memory.mb=5120
```


### 进程内存设置

ResourceManager 的 Java 堆栈大小 24G
NodeManager 的 Java 堆栈大小 4096m

## MapReduce

参数名称 | 默认值 | 描述 
--- | --- | --- 
yarn.app.mapreduce.am.resource.mb | 1536 | ApplicationMaster占用的内存量
yarn.app.mapreduce.am.resource.cpu-vcores | 1 | MR ApplicationMaster占用的虚拟CPU个数
mapreduce.am.max-attempts | 2 | MR ApplicationMaster最大失败尝试次数
mapreduce.map.memory.mb | 1024 | 为作业的每个 Map 任务分配的物理内存量 (MiB)
mapreduce.map.cpu.vcores | 1 | 为作业的每个 Map 任务分配的虚拟 CPU 内核数
mapreduce.map.maxattempts | 4 | Map Task最大失败尝试次数
mapreduce.reduce.memory.mb | 1024 | 为作业的每个 Reduce 任务分配的物理内存量 (MiB)
mapreduce.reduce.cpu.vcores | 1 | 作业的每个 Reduce 任务的虚拟 CPU 内核数。
mapreduce.reduce.maxattempts | 4 | Reduce Task最大失败尝试次数
mapreduce.map.speculative | FALSE | 是否对Map Task启用推测执行机制
mapreduce.reduce.speculative | FALSE | 是否对Reduce Task启用推测执行机制
mapreduce.job.queuename | default | 作业提交到的队列
mapreduce.task.io.sort.mb | 100 | 任务内部排序缓冲区大小
mapreduce.map.sort.spill.percent | 0.8 | Map阶段溢写文件的阈值（排序缓冲区大小的百分比）
mapreduce.reduce.shuffle.parallelcopies | 5 | Reduce Task启动的并发拷贝数据的线程数目

### 参数建议

**mapreduce.map.memory.mb**

单个Map任务的最大内存可以根据实际业务情况来调整，比如有些任务很占内存，就应该调多一点。


**mapreduce.reduce.memory.mb**

单个Reduce任务的最大内存，同上。


**mapreduce.task.io.sort.mb**

Map端数据排序可用内存，如果内存充裕建议调大至256M或512M，可提升排序性能。


## 进程GC配置参考

建议Hadoop进程的GC参数加上如下选项，很多商业版都默认加上了，这对JDK性能提升有很大帮助：

- -XX:CMSInitiatingOccupancyFraction=70  
- -XX:+HeapDumpOnOutOfMemoryError 
- -XX:+UseConcMarkSweepGC 
- -XX:-CMSConcurrentMTEnabled 
- -XX:+CMSIncrementalMode 
- -Djava.net.preferIPv4Stack=true

**参数说明**

-XX:CMSInitiatingOccupancyFraction=70，年老代使用了70%时回收内存

-XX:+HeapDumpOnOutOfMemoryError，JVM会在遇到OutOfMemoryError时拍摄一个“堆转储快照”，并将其保存在一个文件中

-XX:+UseConcMarkSweepGC，启用CMS

-XX:-CMSConcurrentMTEnabled（-号代表否认）
    当该标志被启用时，并发的CMS阶段将以多线程执行(因此，多个GC线程会与所有的应用程序线程并行工作)。
    该标志已经默认开启，如果顺序执行更好，这取决于所使用的硬件，多线程执行可以通过-XX：-CMSConcurremntMTEnabled禁用。

-XX:+CMSIncrementalMode. 在增量模式下，CMS 收集器在并发阶段，不会独占整个周期，
    而会周期性的暂停，唤醒应用线程。收集器把并发阶段工作，划分为片段，安排在次级(minor) 回收之间运行。
    这对需要低延迟，运行在少量CPU服务器上的应用很有用。

-Djava.net.preferIPv4Stack=true，禁用IPv6


后面我会写一篇Hbase参数调优以及GC优化的文章，里面会详细介绍Java GC。


## 参考文档
- http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.0.6.0/bk_installing_manually_book/content/rpm-chap1-11.html

如有错误，还请提醒。
