---
title: kafka基本操作与参数详解
description: 早在前段时间就准备把kafka的一些资料整理下的，忙于私事实在是分身乏术。今天乘上班空余时间，把kafka的一些基本资料记录下来。前一篇博客已经大概的介绍了kafka是什么东西，用来做什么，以及一些特性，本文通过参考官方文档，对kafka的一些基本操作与概念进行解释，并将broker与topic参数列举出来，对一些常用的参数给出参考值。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2016-3-10 15:42:58
---

## 背景

早在前段时间就准备把kafka的一些资料整理下的，忙于私事实在是分身乏术。今天乘上班空余时间，把kafka的一些基本资料记录下来。前一篇博客已经大概的介绍了kafka是什么东西，用来做什么，以及一些特性，本文通过参考官方文档，对kafka的一些基本操作与概念进行解释，并将broker与topic参数列举出来，对一些常用的参数给出参考值。

笔者使用kafka0.8.1.1


## 命令行操作


### 创建topic
```
kafka-topics --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
```

`zookeeper`：指定zookeeper集群
`replication-factor`：副本数
`partitions`：分区数
`topic`：topic名称

### 修改topic配置参数

覆盖已经有topic参数，下面例子修改“test”的`max message`属性
```
kafka-topics --zookeeper localhost:2181 --alter --topic test  --config max.message.bytes=128000
```

多个参数，则多个`--config`


### 删除topic配置参数
```
kafka-topics --zookeeper localhost:2181 --alter --topic test --deleteConfig max.message.bytes
```

### topic删除

1、删除zookeeper元数据：
```
kafka-topics --delete --zookeeper localhost:2181 --topic test
```

2、删除kafka存储目录：
删除server.properties文件log.dirs配置，默认为"/tmp/kafka-logs"下相关topic目录（所有与topic相关的broker机器）。

建议不要轻易删除topic，手动删除数据文件可能存在误操作。


## topic级别配置属性表

以下是topic级别配置， kafak server中默认配置为下表“Server Default Property”列，当需要设置topic级别配置时，属性设置为“Property(属性)”列

<table><tbody><tr><th>Property(属性)<br>
</th><th>Default(默认值)<br>
</th><th>Server Default Property(server.properties)<br>
</th><th>说明(解释)<br>
</th></tr>
<tr><td>cleanup.policy<br>
</td><td>delete<br>
</td><td>log.cleanup.policy<br>
</td><td>日志清理策略选择有：delete和compact主要针对过期数据的处理，或是日志文件达到限制的额度，会被 topic创建时的指定参数覆盖<br>
</td></tr><tr><td>delete.retention.ms<br>
</td><td>86400000 (24 hours)<br>
</td><td>log.cleaner.delete.retention.ms<br>
</td><td>对于压缩的日志保留的最长时间，也是客户端消费消息的最长时间，同log.retention.minutes的区别在于一个控制未压缩数据，一个控制压缩后的数据。会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>flush.messages<br>
</td><td>None<br>
</td><td>log.flush.interval.messages<br>
</td><td>log文件”sync”到磁盘之前累积的消息条数,因为磁盘IO操作是一个慢操作,但又是一个”数据可靠性"的必要手段,所以此参数的设置,需要在"数据可靠性"与"性能"之间做必要的权衡.如果此值过大,将会导致每次"fsync"的时间较长(IO阻塞),如果此值过小,将会导致"fsync"的次数较多,这也意味着整体的client请求有一定的延迟.物理server故障,将会导致没有fsync的消息丢失.<br>
</td></tr><tr><td>flush.ms<br>
</td><td>None<br>
</td><td>log.flush.interval.ms<br>
</td><td>仅仅通过interval来控制消息的磁盘写入时机,是不足的.此参数用于控制"fsync"的时间间隔,如果消息量始终没有达到阀值,但是离上一次磁盘同步的时间间隔达到阀值,也将触发.<br>
</td></tr><tr><td>index.interval.bytes<br>
</td><td>4096<br>
</td><td>log.index.interval.bytes<br>
</td><td>当执行一个fetch操作后，需要一定的空间来扫描最近的offset大小，设置越大，代表扫描速度越快，但是也更好内存，一般情况下不需要搭理这个参数<br>
</td></tr><tr><td>max.message.bytes<br>
</td><td>1,000,000<br>
</td><td>message.max.bytes<br>
</td><td>表示消息的最大大小，单位是字节<br>
</td></tr><tr><td>min.cleanable.dirty.ratio<br>
</td><td>0.5<br>
</td><td>log.cleaner.min.cleanable.ratio<br>
</td><td>日志清理的频率控制，越大意味着更高效的清理，同时会存在一些空间上的浪费，会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>retention.bytes<br>
</td><td>None<br>
</td><td>log.retention.bytes<br>
</td><td>topic每个分区的最大文件大小，一个topic的大小限制 = 分区数*log.retention.bytes。-1没有大小限log.retention.bytes和log.retention.minutes任意一个达到要求，都会执行删除，会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>retention.ms<br>
</td><td>7 days<br>
</td><td>log.retention.minutes<br>
</td><td>数据存储的最大时间超过这个时间会根据log.cleanup.policy设置的策略处理数据，也就是消费端能够多久去消费数据<br>
log.retention.bytes和log.retention.minutes达到要求，都会执行删除，会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>segment.bytes<br>
</td><td>1 GB<br>
</td><td>log.segment.bytes<br>
</td><td>topic的分区是以一堆segment文件存储的，这个控制每个segment的大小，会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>segment.index.bytes<br>
</td><td>10 MB<br>
</td><td>log.index.size.max.bytes<br>
</td><td>对于segment日志的索引文件大小限制，会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>segment.ms<br>
</td><td>7 days<br>
</td><td>log.roll.hours<br>
</td><td>这个参数会在日志segment没有达到log.segment.bytes设置的大小，也会强制新建一个segment会被 topic创建时的指定参数覆盖<br>
</td></tr></tbody></table>


在kafka0.9版本中加入了下面两个参数：

`segment.jitter.ms`=>`log.roll.jitter.{ms,hours}`：The maximum jitter to subtract from logRollTimeMillis。 默认值=0

`min.insync.replicas`=>`min.insync.replicas`：当一个生产者集request.required.acks - 1，min.insync.replicas指定最小数量的副本，必须承认一个写被认为是成功的。如果这个最低不能达到，那么生产者将抛出一个异常（无论是notenoughreplicas或notenoughreplicasafterappend）。当一起使用时，min.insync.replicas和request.required.acks允许你执行更大的耐久性保证。一个典型的情况是与3个复制因子创建主题，即min.insync.replicas 2，生产request.required.acks - 1。这将确保生产者提出了一个例外，如果大多数副本不收到一个写。


## broker参数配置

每个kafka broker中配置文件server.properties默认必须配置的属性如下：
```
broker.id=0
num.network.threads=2
num.io.threads=8
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576
socket.request.max.bytes=104857600
log.dirs=/tmp/kafka-logs
num.partitions=2
log.retention.hours=168

log.segment.bytes=536870912
log.retention.check.interval.ms=60000
log.cleaner.enable=false

zookeeper.connect=localhost:2181
zookeeper.connection.timeout.ms=1000000
```

server.properties中所有配置参数说明(解释)如下列表：

<table cellspacing="0" class="t_table"><tbody><tr><td>参数<br>
</td><td>说明(解释)<br>
</td></tr><tr><td>broker.id =0<br>
</td><td>每一个broker在集群中的唯一表示，要求是正数。当该服务器的IP地址发生改变时，broker.id没有变化，则不会影响consumers的消息情况<br>
</td></tr><tr><td>log.dirs=/data/kafka-logs<br>
</td><td>kafka数据的存放地址，多个地址的话用逗号分割,多个目录分布在不同磁盘上可以提高读写性能&nbsp;&nbsp;/data/kafka-logs-1，/data/kafka-logs-2<br>
</td></tr><tr><td>port =9092<br>
</td><td>broker server服务端口<br>
</td></tr><tr><td>message.max.bytes =6525000<br>
</td><td>表示消息体的最大大小，单位是字节<br>
</td></tr><tr><td>num.network.threads =4<br>
</td><td>broker处理消息的最大线程数，一般情况下不需要去修改<br>
</td></tr><tr><td>num.io.threads =8<br>
</td><td>broker处理磁盘IO的线程数，数值应该大于你的硬盘数<br>
</td></tr><tr><td>background.threads =4<br>
</td><td>一些后台任务处理的线程数，例如过期消息文件的删除等，一般情况下不需要去做修改<br>
</td></tr><tr><td>queued.max.requests =500<br>
</td><td>等待IO线程处理的请求队列最大数，若是等待IO的请求超过这个数值，那么会停止接受外部消息，应该是一种自我保护机制。<br>
</td></tr><tr><td>host.name<br>
</td><td>broker的主机地址，若是设置了，那么会绑定到这个地址上，若是没有，会绑定到所有的接口上，并将其中之一发送到ZK，一般不设置<br>
</td></tr><tr><td>socket.send.buffer.bytes=100*1024<br>
</td><td>socket的发送缓冲区，socket的调优参数SO_SNDBUFF<br>
</td></tr><tr><td>socket.receive.buffer.bytes =100*1024<br>
</td><td>socket的接受缓冲区，socket的调优参数SO_RCVBUFF<br>
</td></tr><tr><td>socket.request.max.bytes =100*1024*1024<br>
</td><td>socket请求的最大数值，防止serverOOM，message.max.bytes必然要小于socket.request.max.bytes，会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>log.segment.bytes =1024*1024*1024<br>
</td><td>topic的分区是以一堆segment文件存储的，这个控制每个segment的大小，会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>log.roll.hours =24*7<br>
</td><td>这个参数会在日志segment没有达到log.segment.bytes设置的大小，也会强制新建一个segment会被 topic创建时的指定参数覆盖<br>
</td></tr><tr><td>log.cleanup.policy = delete<br>
</td><td>日志清理策略选择有：delete和compact主要针对过期数据的处理，或是日志文件达到限制的额度，会被 topic创建时的指定参数覆盖<br>
</td></tr><tr><td>log.retention.minutes=3days<br>
</td><td>数据存储的最大时间超过这个时间会根据log.cleanup.policy设置的策略处理数据，也就是消费端能够多久去消费数据<br>
log.retention.bytes和log.retention.minutes任意一个达到要求，都会执行删除，会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>log.retention.bytes=-1<br>
</td><td>topic每个分区的最大文件大小，一个topic的大小限制 =分区数*log.retention.bytes。-1没有大小限log.retention.bytes和log.retention.minutes任意一个达到要求，都会执行删除，会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>log.retention.check.interval.ms=5minutes<br>
</td><td>文件大小检查的周期时间，是否处罚 log.cleanup.policy中设置的策略<br>
</td></tr><tr><td>log.cleaner.enable=false<br>
</td><td>是否开启日志压缩<br>
</td></tr><tr><td>log.cleaner.threads = 2<br>
</td><td>日志压缩运行的线程数<br>
</td></tr><tr><td>log.cleaner.io.max.bytes.per.second=None<br>
</td><td>日志压缩时候处理的最大大小<br>
</td></tr><tr><td>log.cleaner.dedupe.buffer.size=500*1024*1024<br>
</td><td>日志压缩去重时候的缓存空间，在空间允许的情况下，越大越好<br>
</td></tr><tr><td>log.cleaner.io.buffer.size=512*1024<br>
</td><td>日志清理时候用到的IO块大小一般不需要修改<br>
</td></tr><tr><td>log.cleaner.io.buffer.load.factor =0.9<br>
</td><td>日志清理中hash表的扩大因子一般不需要修改<br>
</td></tr><tr><td>log.cleaner.backoff.ms =15000<br>
</td><td>检查是否处罚日志清理的间隔<br>
</td></tr><tr><td>log.cleaner.min.cleanable.ratio=0.5<br>
</td><td>日志清理的频率控制，越大意味着更高效的清理，同时会存在一些空间上的浪费，会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>log.cleaner.delete.retention.ms =1day<br>
</td><td>对于压缩的日志保留的最长时间，也是客户端消费消息的最长时间，同log.retention.minutes的区别在于一个控制未压缩数据，一个控制压缩后的数据。会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>log.index.size.max.bytes =10*1024*1024<br>
</td><td>对于segment日志的索引文件大小限制，会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td>log.index.interval.bytes =4096<br>
</td><td>当执行一个fetch操作后，需要一定的空间来扫描最近的offset大小，设置越大，代表扫描速度越快，但是也更好内存，一般情况下不需要搭理这个参数<br>
</td></tr><tr><td>log.flush.interval.messages=None<br>
</td><td>log文件”sync”到磁盘之前累积的消息条数,因为磁盘IO操作是一个慢操作,但又是一个”数据可靠性"的必要手段,所以此参数的设置,需要在"数据可靠性"与"性能"之间做必要的权衡.如果此值过大,将会导致每次"fsync"的时间较长(IO阻塞),如果此值过小,将会导致"fsync"的次数较多,这也意味着整体的client请求有一定的延迟.物理server故障,将会导致没有fsync的消息丢失.<br>
</td></tr><tr><td>log.flush.scheduler.interval.ms =3000<br>
</td><td>检查是否需要固化到硬盘的时间间隔<br>
</td></tr><tr><td>log.flush.interval.ms = None<br>
</td><td>仅仅通过interval来控制消息的磁盘写入时机,是不足的.此参数用于控制"fsync"的时间间隔,如果消息量始终没有达到阀值,但是离上一次磁盘同步的时间间隔达到阀值,也将触发.<br>
</td></tr><tr><td>log.delete.delay.ms =60000<br>
</td><td>文件在索引中清除后保留的时间一般不需要去修改<br>
</td></tr><tr><td>log.flush.offset.checkpoint.interval.ms =60000<br>
</td><td>控制上次固化硬盘的时间点，以便于数据恢复一般不需要去修改<br>
</td></tr><tr><td>auto.create.topics.enable =true<br>
</td><td>是否允许自动创建topic，若是false，就需要通过命令创建topic<br>
</td></tr><tr><td>default.replication.factor =1<br>
</td><td>是否允许自动创建topic，若是false，就需要通过命令创建topic<br>
</td></tr><tr><td>num.partitions =1<br>
</td><td>每个topic的分区个数，若是在topic创建时候没有指定的话会被topic创建时的指定参数覆盖<br>
</td></tr><tr><td><br>
</td><td><br>
</td></tr><tr><td>以下是kafka中Leader,replicas配置参数<br>
</td><td><br>
</td></tr><tr><td>controller.socket.timeout.ms =30000<br>
</td><td>partition leader与replicas之间通讯时,socket的超时时间<br>
</td></tr><tr><td>controller.message.queue.size=10<br>
</td><td>partition leader与replicas数据同步时,消息的队列尺寸<br>
</td></tr><tr><td>replica.lag.time.max.ms =10000<br>
</td><td>replicas响应partition leader的最长等待时间，若是超过这个时间，就将replicas列入ISR(in-sync replicas)，并认为它是死的，不会再加入管理中<br>
</td></tr><tr><td>replica.lag.max.messages =4000<br>
</td><td>如果follower落后与leader太多,将会认为此follower[或者说partition relicas]已经失效<br>
##通常,在follower与leader通讯时,因为网络延迟或者链接断开,总会导致replicas中消息同步滞后<br>
##如果消息之后太多,leader将认为此follower网络延迟较大或者消息吞吐能力有限,将会把此replicas迁移<br>
##到其他follower中.<br>
##在broker数量较少,或者网络不足的环境中,建议提高此值.<br>
</td></tr><tr><td>replica.socket.timeout.ms=30*1000<br>
</td><td>follower与leader之间的socket超时时间<br>
</td></tr><tr><td>replica.socket.receive.buffer.bytes=64*1024<br>
</td><td>leader复制时候的socket缓存大小<br>
</td></tr><tr><td>replica.fetch.max.bytes =1024*1024<br>
</td><td>replicas每次获取数据的最大大小<br>
</td></tr><tr><td>replica.fetch.wait.max.ms =500<br>
</td><td>replicas同leader之间通信的最大等待时间，失败了会重试<br>
</td></tr><tr><td>replica.fetch.min.bytes =1<br>
</td><td>fetch的最小数据尺寸,如果leader中尚未同步的数据不足此值,将会阻塞,直到满足条件<br>
</td></tr><tr><td>num.replica.fetchers=1<br>
</td><td>leader进行复制的线程数，增大这个数值会增加follower的IO<br>
</td></tr><tr><td>replica.high.watermark.checkpoint.interval.ms =5000<br>
</td><td>每个replica检查是否将最高水位进行固化的频率<br>
</td></tr><tr><td>controlled.shutdown.enable =false<br>
</td><td>是否允许控制器关闭broker ,若是设置为true,会关闭所有在这个broker上的leader，并转移到其他broker<br>
</td></tr><tr><td>controlled.shutdown.max.retries =3<br>
</td><td>控制器关闭的尝试次数<br>
</td></tr><tr><td>controlled.shutdown.retry.backoff.ms =5000<br>
</td><td>每次关闭尝试的时间间隔<br>
</td></tr><tr><td>leader.imbalance.per.broker.percentage =10<br>
</td><td>leader的不平衡比例，若是超过这个数值，会对分区进行重新的平衡<br>
</td></tr><tr><td>leader.imbalance.check.interval.seconds =300<br>
</td><td>检查leader是否不平衡的时间间隔<br>
</td></tr><tr><td>offset.metadata.max.bytes<br>
</td><td>客户端保留offset信息的最大空间大小<br>
</td></tr><tr><td>kafka中zookeeper参数配置<br>
</td><td><br>
</td></tr><tr><td>zookeeper.connect = localhost:2181<br>
</td><td>zookeeper集群的地址，可以是多个，多个之间用逗号分割hostname1:port1,hostname2:port2,hostname3:port3<br>
</td></tr><tr><td>zookeeper.session.timeout.ms=6000<br>
</td><td>ZooKeeper的最大超时时间，就是心跳的间隔，若是没有反映，那么认为已经死了，不易过大<br>
</td></tr><tr><td>zookeeper.connection.timeout.ms =6000<br>
</td><td>ZooKeeper的连接超时时间<br>
</td></tr><tr><td>zookeeper.sync.time.ms =2000<br>
</td><td>ZooKeeper集群中leader和follower之间的同步实际那<br>
</td></tr></tbody></table>

### broker服务端参数优化

配置优化都是修改server.properties文件中参数值

1、网络和io操作线程配置优化
```
# broker处理消息的最大线程数
num.network.threads=xxx
# broker处理磁盘IO的线程数
num.io.threads=xxx
```

建议配置：
一般num.network.threads主要处理网络io，读写缓冲区数据，基本没有io等待，配置线程数量为cpu核数加1。
num.io.threads主要进行磁盘io操作，高峰期可能有些io等待，broker处理磁盘IO的线程数，数值应该大于你的硬盘数。

2、log数据文件刷新策略
为了大幅度提高producer写入吞吐量，需要定期批量写文件。

建议配置：
```
# 每当producer写入10000条消息时，刷数据到磁盘
log.flush.interval.messages=10000
# 每间隔1秒钟时间，刷数据到磁盘
log.flush.interval.ms=1000
```

3、日志保留策略配置
当kafka server的被写入海量消息后，会生成很多数据文件，且占用大量磁盘空间，如果不及时清理，可能磁盘空间不够用，kafka默认是保留7天。

建议配置：
```
# 保留三天，也可以更短 
log.retention.hours=72
# 段文件配置1GB，有利于快速回收磁盘空间，重启kafka加载也会加快(如果文件过小，则文件数量比较多，
# kafka启动时是单线程扫描目录(log.dir)下所有数据文件)
log.segment.bytes=1073741824
```


可以配置jmx监控服务：
kafka server中默认是不启动jmx端口的，需要用户自己配置
```
vim bin/kafka-run-class.sh
#最前面添加一行
JMX_PORT=8060
```

## 参考

- http://kafka.apache.org/081/documentation.html#configuration
- http://www.aboutyun.com/thread-10882-1-1.html
