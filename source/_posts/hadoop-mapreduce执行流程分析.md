---
title: MapReduce执行流程分析
description: 当MapReduce启动的时候，我们可知hadoop内部做了些什么？下面将详细介绍MapReduce的执行流程。<br/><img alt="Alt text" src="http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-cllc.png">
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-11-27 20:20:00
---

## 背景
当MapReduce启动的时候，我们可知hadoop内部做了些什么？下面将详细介绍MapReduce的执行流程。

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-cllc.png)

## Block块
由于hadoop启动首先会对文件进行逻辑上的split，split的个数取决于block块大小，所以在此先介绍hadoop中的文件block。

当我们把文件上传到HDFS中，第一步就是数据的划分，这个是真实物理上的划分，数据文件上传到HDFS后，要把文件划分成一块一块，每块的大小可以有hadoop-default.xml里配置选项进行划分。现在默认每块128MB以前是(64M)，一个文件被分成多个256MB大小的小文件，最后一个可能小于128MB。注意：128MB只是默认，是可以更改的，修改hdfs-site.xml里dfs.block.size属性即可：
```
<property> 
   <name>dfs.block.size</name> 
   <value>134217728</value> 
   <description>The default block size for new files.</description>
</property>
```

数据的划分有冗余，冗余的概念来自哪儿？为了保证数据的安全，上传的文件是被复制成3份，当一份数据宕掉，其余的可以即刻补上。当然这只是默认，同样还是修改hdfs-site.xml:
```
<property> 
   <name>dfs.replication</name> 
   <value>3</value> 
   <description>
     Default block replication.The actual number of replications can be specified
     when the file is created.The default is used if replication is not specified in create time. 
   </description>
</property>
```
注：<font color="red">可手动修改文件系统副本数：hadoop fs -setrep -R 2 /input
如果客户端写文件时就指定副本数，可以在本地代码中conf.setLong("dfs.replication",2l)</font>

## Split块
MapReduce执行前首先会预先分配Map个数，Map个数默认取决于文件的大小和block块的大小。当然Map个数也是可以通过参数要强制指定的，这里我就不做解释（后面我会另写博文来描述怎么手动指定map reduce个数）。

本文我们以hadoop自带的TextInputFormat来说。由于Hadoop版本更新换代很快，不同版本中的split的划分是由不同的job任务来完成的。早先的版本split是由JobTracker端完成的，后来的版本是由JobClient完成的，JobClient划分好后，把split.file写入HDFS中，到时候Aplication Master端读这个文件，就知道split是怎样划分的了。这种数据的划分其实只是一种逻辑上的划分，目的是为了让Map Task更好的获取数据
例如：
```
File1：Block11，Block12，Block13，Block14，Block15
File2：Block21，Block22，Block23
```
上面有2个文件，File1有5个block,File2有3个block。首先会调用TextInputFormat的getSplits方法中，首先调用listStatus(job)
```
List<FileStatus> files = listStatus(job);
```

来获取需要执行的文件个数。然后循环每个文件，根据file.getLen()/blockSize来决定改文件由多少个Map指定
```
long blockSize = file.getBlockSize();
          long splitSize = computeSplitSize(blockSize, minSize, maxSize);

protected long computeSplitSize(long blockSize, long minSize,
                                  long maxSize) {
    return Math.max(minSize, Math.min(maxSize, blockSize));
}
```
computeSplitSize方法其实默认返回就是blockSize。

几个简单的结论：
1)一个split大于等于1的整数个Block
2) 一个split不会包含两个File的Block,不会跨越File边界
3) split和Block的关系是一对多的关系，默认一对一
4) maptasks的个数最终决定于splits的长度

在FileSplit类中，有一项是private String[] hosts;看上去是说明这个FileSplit是放在哪些机器上的，实际上hosts里只是存储了一个Block的冗余机器列表。比如上面例子中的Split 1: Block11, Block12, Block13,Block14,这个FileSplit中的hosts里最终存储的是Block11本身和其冗余所在的机器列表，也就是说 Block12,Block13,Block14这些块存在的那些机器上没有在FileSplit中记录，并包含Block与Split之间的关系记录。

FileSplit中的这个属性有利于调度作业时候的数据本地性问题（<font color="red">数据本地性：作用很大，更有利于MapReduce性能调优</font>）。如果一个NodeManger前来索取task，ApplicationMaster就会找个task给它，找到一个maptask，得先看这个task的输入的FileSplit里hosts是否包含NodeManager所在机器，也就是判断和该NodeManager同时存在一个机器上的datanode是否拥有FileSplit中某个Block的备份。

## Map流程

1．每个输入分片会让一个map任务来处理，默认情况下，以HDFS的一个块的大小（默认为64M）为一个分片。Map的工作就是按业务逻辑对数据进行拆分，形成key/value的键值对数据。map输出的结果会暂且放在一个环形内存缓冲区中（该缓冲区的大小默认为100M，由mapreduce.task.io.sort.mb属性控制），当该缓冲区快要溢出会在本地文件系统中创建一个溢出文件，将该缓冲区中的数据写入这个文件。

2、在做Map数据拆分的同时，map线程会将本地负责的key/value数据列表按key值进行排序，并且会进行分区平均。分区的数量是根据reduce线程的个数来确定的，以便在完成map操作之后，将后续的工作平均分配给每个reduce线程，这样做是为了避免有些reduce任务分配到大量数据，而有些reduce任务却分到很少数据，甚至没有分到数据的尴尬局面。

3、如果此时设置了Combiner方法，会将排序后的结果进行Combine操作，也就是将相同key值的数据进行合并，合并的过程中会不断地进行排序和combine操作，目的有两个：
1.尽量减少每次写入磁盘的数据量；
2.尽量减少下一复制阶段网络传输的数据量。最后合并成了一个已分区且已排序的文件。

4．Shuffle “洗牌” ：一个map产生的数据，一般按照key值进行hash算法，平均的将各个map节点上的数据分别分配给reduce线程对应的节点。

## Shuffle过程浅析
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-shuffle.png)

上图中分为Map任务和Reduce任务两个阶段，从map端输出到reduce端的红色和绿色的线表示数据流的一个过程，也我们所要了解的Shuffle过程。

### Map端
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-shuffle-map.jpg)

1、在map端首先接触的是InputSplit，在InputSplit中含有DataNode中的数据，每一个InputSplit都会分配一个Mapper任务，Mapper任务结束后产生<K2,V2>的输出， 这些输出先存放在缓存中，每个map有一个环形内存缓冲区，用于存储任务的输出。默认大小100MB（io.sort.mb属性），一旦达到阀值0.8(io.sort.spil l.percent)，一个后台线程就把内容写到(spill)Linux本地磁盘中的指定目录（mapred.local.dir）下的新建的一个溢出写文件。

总结： map过程的输出是写入本地磁盘而不是HDFS，但是一开始数据并不是直接写入磁盘而是缓冲在内存中，缓存的好处就是减少磁盘I/O的开销，提高合并和排序的速度 。又因为默认的内存缓冲大小是100M（当然这个是可以配置的），所以 在编写map函数的时候要尽量减少内存的使用，为shuffle过程预留更多的内存 ，因为该过程是最耗时的过程。

2、写磁盘前，要进行partition、sort和combine等操作。通过分区，将不同类型的数据分开处理，之后对不同分区的数据进行排序，如果有Combiner，还要对排序后的数据进行combine。等最后记录写完，将全部溢出文件合并为一个分区且排序的文件。

3、最后将磁盘中的数据送到Reduce中，从图中可以看出Map输出有三个分区，有一个分区数据被送到图示的Reduce任务中，剩下的两个分区被送到其他Reducer任务中。而图示的Reducer任务的其他的三个输入则来自其他节点的Map输出。

补充： 在写磁盘的时候采用压缩的方式将map的输出结果进行压缩是一个减少网络开销很有效的方法！关于如何使用压缩，在本文第三部分会有介绍。

### Reduce端
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-shuffle-reduce.jpg)

1、Copy阶段：Reducer通过Http方式得到输出文件的分区。
reduce端可能从n个map的结果中获取数据，而这些map的执行速度不尽相同，当其中一个map运行结束时，reduce就会从 JobTracker中获取该信息。map运行结束后TaskTracker会得到消息，进而将消息汇报给JobTracker，reduce定时从 JobTracker获取该信息，reduce端默认有5个数据复制线程从map端复制数据。

2、Merge阶段：如果形成多个磁盘文件会进行合并
从map端复制来的数据首先写到reduce端的缓存中，同样缓存占用到达一定阈值后会将数据写到磁盘中，同样会进行partition、 combine、排序等过程。如果形成了多个磁盘文件还会进行合并，最后一次合并的结果作为reduce的输入而不是写入到磁盘中。（3）Reducer的参数：最后将合并后的结果作为输入传入Reduce任务中。

总结： 当Reducer的输入文件确定后，整个Shuffle操作才最终结束。之后就是Reducer的执行了，最后Reducer会把结果存到HDFS上(可以通过自定义output来控制输出)。

## Reduce流程

1．Reduce会接收到不同map任务传来的数据，并且每个map传来的数据都是有序的。如果reduce端接受的数据量相当小，则直接存储在内存中
如果数据量超过了该缓冲区大小的一定比例（由mapreduce.map.sort.spill.percent决定），则对数据合并后溢写到磁盘中。

2、Reduce线程在接收到各节点的key/value数据之后，会按照预先编写的reduce算法，对每个key/value值进行处理计算。

3．最终reduce端会将所有处理过的数据同样的以key/value的形式合并写入同一个文件存放hdfs系统。

在我们进行map/reduce操作时，一般只需要写map和reduce的算法；并且对key/value值进行合理的定义，即可以实现各种业务逻辑的分布式算法。

## Hadoop中的压缩
刚刚我们在了解Shuffle过程中看到，map端在写磁盘的时候采用压缩的方式将map的输出结果进行压缩是一个减少网络开销很有效的方法。其实，在Hadoop中早已为我们提供了一些压缩算法的实现，我们不用重复造轮子了。

### 解压缩算法的实现：Codec
Codec是Hadoop中关于压缩，解压缩的算法的实现，在Hadoop中，codec由CompressionCode的实现来表示。下面是一些常见压缩算法实现，如下图所示：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-shuffle-compression.png)

### Map端压缩输出到Reduce
```
conf.setBoolean("mapreduce.map.output.compress", true);
conf.setClass("mapreduce.map.output.compress.codec", SnappyCodec.class, CompressionCodec.class);
```

### MapReduce输出文件进行压缩

```
conf.setBoolean("mapreduce.output.fileoutputformat.compress", true);
conf.setClass("mapreduce.output.fileoutputformat.compress.codec",SnappyCodec.class, CompressionCodec.class);
```

上面代码中在reduce端输出压缩使用了刚刚Codec中的Snappy算法，当然你也可以使用bzip2\lzo算法。
注：<font color="red">lzo/Snappy算法在开源hadoop中默认是没有的，得自己手动安装</font>

## 总结
主要有两个方面影响shuffle阶段的性能：
1、数据完全是远程拷贝 
2、采用HTTP协议进行数据传输。对于第一个方面，如果采用某种策略（修改框架），让你reduce task也能有locality就好了；对于第二个方面，用新的更快的数据传输协议替换HTTP，也许能更快些, 如UDT协议， 它在MapReduce的另一个C++开源实现Sector/Sphere中被使用，效果不错！

## 参考资料 
- http://www.tuicool.com/articles/2YjeIne
- http://dongxicheng.org/mapreduce/hadoop-shuffle-phase/
