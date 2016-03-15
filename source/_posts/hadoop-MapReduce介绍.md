---
title: hadoop-MapReduce介绍
description: Hadoop MapReduce是基于HDFS的MapReduce编程框架，MapReduce是一个能够在大量的普通配置的计算机上处理和生成超大数据集的编程模型的</br>具体实现：“Map（映射）”和“Reduce（归约）”</br>采用MapReduce架构可以使那些没有并行计算和分布式处理系统开发经验的程序员有效利</br>用分布式系统的丰富资源。</br>将整个计算的过程称为作业 (Job),作业分解为任务 (Task),有两种任务：</br>1）Map把输入的键/值对转换成一组中间结果的键/值对</br>2）Reduce把Map任务产生的一组具有相同键的中间结果根据逻辑转换生成较小的最终结果</br><img alt="Alt text" src="http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-jg.png">
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-11-09 20:10:00
---

## 背景

Hadoop MapReduce是基于HDFS的MapReduce编程框架，MapReduce是一个能够在大量的普通配置的计算机上处理和生成超大数据集的编程模型的
具体实现：“Map（映射）”和“Reduce（归约）”
采用MapReduce架构可以使那些没有并行计算和分布式处理系统开发经验的程序员有效利
用分布式系统的丰富资源。

将整个计算的过程称为作业 (Job),作业分解为任务 (Task),有两种任务：
1）Map把输入的键/值对转换成一组中间结果的键/值对
2）Reduce把Map任务产生的一组具有相同键的中间结果根据逻辑转换生成较小的最终结果

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-jg.png)

## MapReduce示例

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-sl.png)

1、输入：文件将以每行一个key/value的形式分割输入
2、Map：每个文件处理都开启一个Map线程，Map方法将按单词规则对文本进行分割，每个单词成形成一个key/value
3、排序：在对单词进行划分后，会按key值规则进行排序
4、Combine合并：在完成Map操作后，在本地先对结果按key进行合并，Value形成List列表。以减少reduce的工作量
5、合并排序：将所有Map的结果进行合并、排序。基开启多个Reduce线程时，需要进行分区，形成新的Reduce输入。
6、Reduce：Reduce线程开启，每个Reduce处理一部分Key/Value。此处统计每个单词出现的次数即可。Reduce过程对每个键值对进行合并、逻辑计算处理。
7、输出结果。

## MapReduce处理流程

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-cllc.png)

### Map端
1．每个输入分片会让一个map任务来处理，默认情况下，以HDFS的一个块的大小（默认为64M）为一个分片。Map的工作就是按业务逻辑对数据进行拆分，形成key/value的键值对数据。map输出的结果会暂且放在一个环形内存缓冲区中（该缓冲区的大小默认为100M，由mapreduce.task.io.sort.mb属性控制），当该缓冲区快要溢出会在本地文件系统中创建一个溢出文件，将该缓冲区中的数据写入这个文件。

2、在做Map数据拆分的同时，map线程会将本地负责的key/value数据列表按key值进行排序，并且会进行分区平均。分区的数量是根据reduce线程的个数来确定的，以便在完成map操作之后，将后续的工作平均分配给每个reduce线程，这样做是为了避免有些reduce任务分配到大量数据，而有些reduce任务却分到很少数据，甚至没有分到数据的尴尬局面。

3、如果此时设置了Combiner方法，会将排序后的结果进行Combine操作，也就是将相同key值的数据进行合并，合并的过程中会不断地进行排序和combine操作，目的有两个：
1.尽量减少每次写入磁盘的数据量；
2.尽量减少下一复制阶段网络传输的数据量。最后合并成了一个已分区且已排序的文件。

4．Shuffle “洗牌” ：一个map产生的数据，一般按照key值进行hash算法，平均的将各个map节点上的数据分别分配给reduce线程对应的节点。

### Reduce端
1．Reduce会接收到不同map任务传来的数据，并且每个map传来的数据都是有序的。如果reduce端接受的数据量相当小，则直接存储在内存中
如果数据量超过了该缓冲区大小的一定比例（由mapreduce.map.sort.spill.percent决定），则对数据合并后溢写到磁盘中。

2、Reduce线程在接收到各节点的key/value数据之后，会按照预先编写的reduce算法，对每个key/value值进行处理计算。

3．最终reduce端会将所有处理过的数据同样的以key/value的形式合并写入同一个文件存放hdfs系统。

在我们进行map/reduce操作时，一般只需要写map和reduce的算法；并且对key/value值进行合理的定义，即可以实现各种业务逻辑的分布式算法。

## MapReduce工作原理

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-gzyl.png)

1.在客户端启动一个作业。 

2.向JobTracker请求一个Job ID。 

3.将运行作业所需要的资源文件复制到HDFS上，包括MapReduce程序打包的JAR文件、配置文件和客户端计算所得的输入划分信息。
这些文件都存放在JobTracker专门为该作业创建的文件夹中。文件夹名为该作业的Job ID。输入划分信息告诉了JobTracker应该为这个作业启动多少个map/reduce任务等信息。

4.JobTracker接收到作业后，将其放在一个作业队列里，等待作业调度器对其进行调度，当作业调度器根据自己的调度算法调度到该作业时，会根据输入划分信息创建多个map/reduce任务，并将任务分配给不同节点上的TaskTracker执行。根据数据本地化（Data-Local）原则，一般会将map任务分配给含有该map处理的数据块的TaskTracker节点上，同时将程序JAR包复制到该TaskTracker上来运行，这叫“运算移动，数据不移动”。而分配reduce任务时并不考虑数据本地化。 

5.TaskTracker每隔一段时间会给JobTracker发送一个心跳，告诉JobTracker它依然在运行，同时心跳中还携带着很多的信息，比如当前map任务完成的进度等信息。当JobTracker收到作业的最后一个任务完成信息时，便把该作业设置成“成功”。当JobClient查询状态时，它将得知任务已完成，便返回一条消息给用户。

## 下一代MapReduce - Yarn

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-yarn.png)

在Hadoop1.x中，由于MapReduce模块的主服务（Jobtracker）任务太多，当集群中MR任务非常多时，会造成大量内存开销。于是在hadoop2之后，将 JobTracker 两个主要功能进行拆分（资源管理、任务调度 / 监控）。
1、新的资源管理器全局管理所有应用程序计算资源的分配(Resource Manager)
2、任务调度协调被分配给hadoop集群的每个节点单独进行管理(Node Manager)

其他：
首先客户端不变，其调用 API 及接口大部分保持兼容
但是原框架中核心的 JobTracker 和 TaskTracker 不见了，取而代之的是 ResourceManager, ApplicationMaster 与 NodeManager 三个部分。
首先 ResourceManager 是一个中心的服务，它做的事情是调度、启动每一个 Job 所属的 ApplicationMaster、另外监控 ApplicationMaster 的存在情况

NodeManager 功能比较专一，就是负责 Container 状态的维护，并向 RM 保持心跳。
ApplicationMaster 负责一个 Job 生命周期内的所有工作，类似老的框架中 JobTracker。
这个设计大大减小了 JobTracker（也就是现在的 ResourceManager）的资源消耗，并且让监测每一个 Job 子任务 (tasks) 状态的程序分布式化了，更安全、更优美。 

Container 是 Yarn 为了将来作资源隔离而提出的一个框架。

## 参考资料
帮胜 ——《hadoop技术介绍》