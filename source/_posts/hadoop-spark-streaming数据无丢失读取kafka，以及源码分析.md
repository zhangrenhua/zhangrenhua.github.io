---
title: Spark streaming数据无丢失读取kafka，以及源码分析
description: 忙于工作交接，忙于“吸毒”，导致一个多月没有更新博客了。也许是因为这边的工作环境原因吧！没有比较大一点的项目，没有大数据量，导致学习兴趣下降。逆水行舟不进则退，必须开始动起来，最近自己在规划一个apache日志实时处理，用于监控项目的负载情况和简单的网络攻击防御处理。</br>采用spark streaming + kafka实时处理数据，保证数据的无丢失读取。</br>上面所说的“吸毒”其实就是，守望先锋。。。</br><h2 id="_2"><a name="user-content-_2" href="#_2" class="headeranchor-link" aria-hidden="true"><span class="headeranchor"></span></a>流处理的概念</h2>先解释下流处理中的一些概念：</br>- At most once 每条数据最多被处理一次</br>- At least once 每条数据最少被处理一次</br>- Exactly once 每条数据只会被处理一次</br></br>kafka streaming中怎么能满足上面的其中一点呢？请看下面分析与总结。
tags: [hadoop生态圈, spark streaming, streaming-loganalysis]
categories: spark
date: 2016-8-2 16:06:10
---

## 背景

忙于工作交接，忙于“吸毒”，导致一个多月没有更新博客了。也许是因为这边的工作环境原因吧！没有比较大一点的项目，没有大数据量，导致学习兴趣下降。逆水行舟不进则退，必须开始动起来，最近自己在规划一个apache日志实时处理，用于监控项目的负载情况和简单的网络攻击防御处理。

采用spark streaming + kafka实时处理数据，保证数据的无丢失读取。
上面所说的“吸毒”其实就是，守望先锋。。。


## 流处理的概念

先解释下流处理中的一些概念：

- At most once 每条数据最多被处理一次
- At least once 每条数据最少被处理一次
- Exactly once 每条数据只会被处理一次

kafka streaming中怎么能满足上面的其中一点呢？请看下面分析与总结。


## kafka streaming创建的几种方式

### createStream（Receiver）

利用`KafkaUtils.createStream`就能创建一个Receiver方式的kafka streaming。


#### 源码分析

查看`KafkaUtils` 84行左右，当调用createStream的时候其实是创建了一个`KafkaInputDStream`对象:
![KafkaUtils source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source1.png)

查看`KafkaInputDStream`类，有个`getReceiver`方法，改方法是获取一个kafka数据接收器，根据`useReliableReceiver`参数来决定使用哪个接收器：
![KafkaInputDStream source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source2.png)

这里先来讲`KafkaReceiver`接收器，在`KafkaInputDStream`中的`KafkaReceiver`类中，可以看出，这是一个简单的kafka消息接收器，采用`ConsumerConnector`来接收数据：
![KafkaInputDStream source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source3.png)

上面这个类，就是简单的数据接收器，下面就来具体讲下`ReliableKafkaReceiver`接收器，从名字可以看出它是一个可靠的接收器，为什么呢？那就从开始阅读源码吧：
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source4.png)

从上面类的注释中可以看出，
1、它默认是关闭的，设置`spark.streaming.receiver.writeaheadlog.enable=true` 即可启用。
2、根据topic的偏移量来接收数据，并更新偏移信息。
3、只有当数据可靠地存储时，偏移量才会被更新，所以kafkareceiver潜在的数据丢失问题可以消除（等会验证）。
4、ReliableKafkaReceiver已将`auto.commit.enable`参数设置为false，外面设置为true不会生效，并会打印警告日志。

ReliableKafkaReceiver类，其实也是采用`ConsumerConnector`来获取数据，但是有一点不同的，就是ReliableKafkaReceiver是自动维护topic的偏移信息的，通过zkClient来手动的提交偏移量：
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source5.png)


查看streaming数据获取的启动方法`onStart`，该方法主要做了以下处理：
1、`topicPartitionOffsetMap`，初始化主题对应的Partition的偏移量
2、`blockOffsetMap`，初始化块对应的偏移量
3、`blockGenerator`，初始化块的处理器
4、`props.setProperty(AUTO_OFFSET_COMMIT, "false")`，强制设置`auto.commit.enable`为false
5、`consumerConnector`，初始化消息读取器
6、`zkClient`，初始化zkClient，用户读写kafka偏移信息
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source6.png)

根据topic数据量来创建线程池，启动MessageHandler线程来获取数据：
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source7.png)

一直接收数据，并调用`storeMessageAndMetadata`方法来存储数据：
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source8.png)

`storeMessageAndMetadata`方法中通过调用`blockGenerator.addDataWithCallback(data, metadata)`来添加数据：
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source9.png)

`addDataWithCallback`方法中其实是调用
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source10.png)

调用`waitToPush`控制接收速度，并将数据添加到`currentBuffer`中，然后调用GeneratedBlockHandler的`onAddData`方法将偏移量信息更新到`topicPartitionOffsetMap`中：
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source11.png)

现在数据放入`currentBuffer`了，数据接收的偏移量也更新了，那么数据什么时候被读取？数据偏移量又是什么时候提交呢？请看BlockGenerator类的`start`方法，此方法做了三件事情：
1、将状态设置为active
2、启动`blockIntervalTimer`线程
3、启动`blockPushingThread`线程
![BlockGenerator source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source12.png)

`blockIntervalTimer`对象RecurringTimer类，该类其实就是每隔一段时间调用GeneratorState类的`updateCurrentBuffer`方法，`updateCurrentBuffer`方法是将当前从kafka接收到的数据`currentBuffer`放到一个block中，重置`currentBuffer`数据，并将block阻塞put到`blocksForPushing`中：
![BlockGenerator source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source13.png)

当一个block被创建是调用`listener.onGenerateBlock(blockId)`更新偏移量：
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source14.png)
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source15.png)

这个线程的频度控制是有参数`spark.streaming.blockInterval`控制的：
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source17.png)

`blockPushingThread`线程调用`keepPushingBlocks`方法，
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source16.png)

该方法主要从`blocksForPushing`中获取block，然后调用`pushBlock`：
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source18.png)

`pushBlock`方法则调用GeneratedBlockHandler的`onPushBlock`方法，`onPushBlock`方法这调用`storeBlockAndCommitOffset`进行数据推送，重试次数为3，如果3次推送失败，则停止接收。调用`store`函数来将block数据推入spark内存中：
![ReliableKafkaReceiver source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source19.png)

#### 结论

使用`createStream`方法来创建kafka数据流，有两种数据接收的实现`KafkaReceiver`和`ReliableKafkaReceiver`。
KafkaReceiver是一个简单的数据读取类，根据用户传入参数创建ConsumerConnector对象，并读取数据。
ReliableKafkaReceiver是一个可靠的数据读取类，根据用户传入参数创建ConsumerConnector对象，但是该类是手动维护kafka数据读取的偏移信息的，只有当数据包装成block或则传入spark内存中时，才会提交偏移信息，这样有效的保证了数据的安全性。

**两者的区别**

1、两者都是通过ConsumerConnector的方式来读取数据，在数据读取方面无区别
2、KafkaReceiver每读取一条数据后，就将数据存入spark内存中，而ReliableKafkaReceiver是将数据缓存到一个数组中，通过后台线程处理成block再往spark内存中写
3、ReliableKafkaReceiver手动维护偏移信息

ReliableKafkaReceiver将数据放入缓存，通过重试机制保证写入到spark内存中才会更新偏移信息，而KafkaReceiver不能保证。

其实两者都会存在数据丢失的问题，就算放入spark内存中，当spark work进程被中断后，还是会有数据丢失的问题存在。、

那怎样才能保证数据无丢失？？请看下面章节。


### createDirectStream（Direct）

创建数据流，根据topic的分区个数，对每个分区启用一个线程进行数据读取，效率高。返回的RDD中带有偏移信息（offset），但是不保存偏移信息，需要我们人工的去处理。

#### 源码分析


利用KafkaUtils.createDirectStream来创建数据流，主要新增两个参数,
`fromOffsets`：topic下partition的偏移信息
`messageHandler`：消息处理器
创建DirectKafkaInputDStream对象：
![DirectKafkaInputDStream source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source20.png)

DirectKafkaInputDStream，在进行运算的时候会调用`compute`方法，此方法主要目的创建并返回KafkaRDD对象，真正的数据处理也在KafkaRDD中，使用Kafka Direct方式没有缓存，数据是一批批获取，
DirectKafkaInputDStream继承InputDStream，最大重试次数为1，来确保语义一致性：
![DirectKafkaInputDStream source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source21.png)

KafkaRDD类中的`getPartitions`方法，从传入的偏移量信息中获取到每个Topic的Partition的Leader信息，host和port，然后实例化KafkaRDDPartition对象，
KafkaRDDPartition继承Partition，封装了Partition的信息，如topic，偏移from和until，Broker的host和port信息。
![KafkaRDD source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source22.png)

`compute`方法，如果偏移量from和util相等，则直接返回空的Iterator (偏移量相等则表示无数据)，不等则实例化KafkaRDDIterator对象:
![KafkaRDD source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source23.png)

`clamp`方法。从maxMessagesPerPartition中获取。从配置文件中获取spark.streaming.kafka.maxRatePerPartition，比较maxRateLimitPerPartition和(limit / numPartitions)的值，并取最小值。其中sescPerBatch为传入的Duration的值，得到每一个BatchDuration处理的最大Offset值，当前的偏移量与允许每个Partition最大的消息处理量和该Partition当前的Offset值，这两个的最小值，作为untilOffsets。
![KafkaRDDIterator source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source26.png)

KafkaRDDIterator继承NextIterator，具有iterator的特性，有getNext和hasNext方法，可以对数据迭代操作并计算。KafkaRDDIterator内部，以传入的KafkaParams参数构造了一个和Kafka集合交互的KafkaCluster对象，处理Partition的Leader发生变化时的具体处理办法，重写了getNext方法。

162行，如果TaskContext中之前没有失败过，即attemptNumber为0，则直接以KafkaRDDPartition中的host和port信息来连接Kafka，返回SimpleConsumer(Kafak的简单消费者，备注Kafka中还有高级消费者的API)；如果之前失败过，则找到该Partition新的Leader Broker信息，然后再进行连接。
![KafkaRDDIterator source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source24.png)

如果迭代器iter为空或者没有数据，则调用consumer发送FetchRequestBuilder给Broker，获取到一批数据。接下来再次判断iter是否有数据，如果没有则表示以读取到指定的untilOffset了。如果iter有数据，也会对offset和untilOffset进行比较，如果当前消费的offset大于等于untilOffset，则返回。如果当消费的offset小于untilOffset，则更新当前请求的Offset值，并调用messageHandler来处理当前的数据。其中messageHandler就是用户传入的对读取到的数据具体操作的函数。
![KafkaRDDIterator source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source25.png)

Spark Streaming可以根据输入的数据流量和当前的处理流量进行比较。动态资源分配调整，可以通过spark.streaming.backpressure.enabled来设置：
![RateController source](http://7xoqbc.com1.z0.glb.clouddn.com/spark-streaming-kafka-source27.png)


### 结论

1、Kafka Direct底层采用SimpleConsumer对象来获取数据，根据传入的偏移信息来批次获取数据。
2、采用Kafka Direct方式，一批数据处理完，再去取一批数据。Kafka Direct可以办到语义一致性，确保数据消费。
3、kafka Receiver通过高级api，性能没有Direct多线程多分区并行读取高。

**如何确保数据无丢失？**

采用Direct的方式来读取kafka数据，可以指定offset，但是默认不会将当前已读取的位置存储到zookeeper。所以，如果不手动存储，那么，下次执行还是从默认的位置开始读取（或者指定的位置）

当然，也可以将读取到数据的offset，人工的存储在数据库或者其它组件中，每次执行前都读取数据库，设置offset。

还有一种方式，就是将当前已读取的位置更新到zookeeper中，这样，下次读取就会从新的位置开始。

利用direct方式能很轻松的满足每条消息，最多被处理一次、最少被处理一次、只会被处理一次，当然最后的“只会被处理一次”还是有点困难的。

下面通过示例代码来演示，kafka streaming数据无丢失读取，虽然不能保证数据只能被处理一次。

<font color="red">但是我们可以从代码逻辑方面处理，只有在将数据处理完成后再向zk中提交kafka偏移信息。就算程序发生异常退出，我们在批次每运行时都清理上次没有完成的状态即可。</font>

为什么我这里没有讲到使用checkpoint，因为checkpoint也不能完全保证数据不丢失，反而会降低streming的吞吐量。


## 示例代码

### pom.xml

```
<dependencies>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.1</version>
        <scope>test</scope>
    </dependency>

    <!--spark-->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_2.10</artifactId>
        <version>1.6.1</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming-kafka_2.10</artifactId>
        <version>1.6.1</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-yarn_2.10</artifactId>
        <version>1.6.1</version>
    </dependency>

    </dependencies>
```

### java代码

```
import kafka.common.TopicAndPartition;
import kafka.message.MessageAndMetadata;
import kafka.serializer.StringDecoder;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.VoidFunction;
import org.apache.spark.streaming.Durations;
import org.apache.spark.streaming.api.java.JavaInputDStream;
import org.apache.spark.streaming.api.java.JavaStreamingContext;
import org.apache.spark.streaming.kafka.HasOffsetRanges;
import org.apache.spark.streaming.kafka.KafkaCluster;
import org.apache.spark.streaming.kafka.KafkaUtils;
import org.apache.spark.streaming.kafka.OffsetRange;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import scala.Predef;
import scala.Tuple2;
import scala.collection.JavaConversions;

import java.util.Arrays;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;

/**
 * spark streaming使用direct方式读取kafka数据，并存储每个partition读取的offset
 */
public final class JavaDirectKafkaWordCount {

    private static final Logger LOG = LoggerFactory.getLogger(JavaDirectKafkaWordCount.class);

    public static void main(String[] args) {

        if (args.length < 2) {
            System.err.println("Usage: JavaDirectKafkaWordCount <brokers> <topics>\n" +
                    "  <brokers> is a list of one or more Kafka brokers\n" +
                    "  <topics> is a list of one or more kafka topics to consume from\n\n");
            System.exit(1);
        }

        //StreamingExamples.setStreamingLogLevels();

        String brokers = args[0]; // kafka brokers
        String topics = args[1]; // 主题
        long seconds = 10; // 批次时间（单位：秒）

        // Create context with a 2 seconds batch interval
        SparkConf sparkConf = new SparkConf().setAppName("JavaDirectKafkaWordCount");
        JavaStreamingContext jssc = new JavaStreamingContext(sparkConf, Durations.seconds(seconds));

        // 设置kafkaParams
        HashSet<String> topicsSet = new HashSet<>(Arrays.asList(topics.split(",")));
        HashMap<String, String> kafkaParams = new HashMap<>();
        kafkaParams.put("metadata.broker.list", brokers);
        final String groupId = kafkaParams.get("group.id");

        // 创建kafka管理对象
        final KafkaCluster kafkaCluster = getKafkaCluster(kafkaParams);

        // 初始化offsets
        Map<TopicAndPartition, Long> fromOffsets = fromOffsets(topicsSet, kafkaParams, groupId, kafkaCluster, null);

        // 创建kafkaStream
        JavaInputDStream<String> stream = KafkaUtils.createDirectStream(jssc,
                String.class, String.class, StringDecoder.class,
                StringDecoder.class, String.class, kafkaParams,
                fromOffsets,
                new Function<MessageAndMetadata<String, String>, String>() {
                    public String call(MessageAndMetadata<String, String> v1)
                            throws Exception {
                        return v1.message();
                    }
                });


        // print
        stream.print();

        // 存储offsets
        storeConsumerOffsets(groupId, kafkaCluster, stream);

        // Start the computation
        jssc.start();
        jssc.awaitTermination();
    }

    /**
     * @param groupId      消费者 组id
     * @param kafkaCluster kafka管理对象
     * @param stream       kafkaStreamRdd
     */
    private static <T> void storeConsumerOffsets(final String groupId, final KafkaCluster kafkaCluster, JavaInputDStream<T> stream) {

        long l = System.currentTimeMillis();

        stream.foreachRDD(new VoidFunction<JavaRDD<T>>() {
            @Override
            public void call(JavaRDD<T> javaRDD) throws Exception {

                // 根据group.id 存储每个partition消费的位置
                OffsetRange[] offsets = ((HasOffsetRanges) javaRDD.rdd()).offsetRanges();
                for (OffsetRange o : offsets) {
                    // 封装topic.partition 与 offset对应关系 java Map
                    TopicAndPartition topicAndPartition = new TopicAndPartition(o.topic(), o.partition());
                    Map<TopicAndPartition, Object> topicAndPartitionObjectMap = new HashMap<>();
                    topicAndPartitionObjectMap.put(topicAndPartition, o.untilOffset());

                    // 转换java map to scala immutable.map
                    scala.collection.immutable.Map<TopicAndPartition, Object> scalaTopicAndPartitionObjectMap =
                            JavaConversions.mapAsScalaMap(topicAndPartitionObjectMap).toMap(new Predef.$less$colon$less<Tuple2<TopicAndPartition, Object>, Tuple2<TopicAndPartition, Object>>() {
                                public Tuple2<TopicAndPartition, Object> apply(Tuple2<TopicAndPartition, Object> v1) {
                                    return v1;
                                }
                            });

                    // 更新offset到kafkaCluster
                    kafkaCluster.setConsumerOffsets(groupId, scalaTopicAndPartitionObjectMap);
                }
            }
        });

        // 记录处理时间
        LOG.info("storeConsumerOffsets time:" + (System.currentTimeMillis() - l));
    }

    /**
     * 获取partition信息，并设置各分区的offsets
     *
     * @param topicsSet    所有topic
     * @param kafkaParams  kafka参数配置
     * @param groupId      消费者 组id
     * @param kafkaCluster kafka管理对象
     * @param offset       自定义offset
     * @return offsets
     */
    private static Map<TopicAndPartition, Long> fromOffsets(HashSet<String> topicsSet, HashMap<String, String> kafkaParams, String groupId, KafkaCluster kafkaCluster, Long offset) {

        long l = System.currentTimeMillis();

        // 所有partition offset
        Map<TopicAndPartition, Long> fromOffsets = new HashMap<>();

        // util.set 转 scala.set
        scala.collection.immutable.Set<String> immutableTopics = JavaConversions
                .asScalaSet(topicsSet)
                .toSet();

        // 获取topic分区信息
        scala.collection.immutable.Set<TopicAndPartition> scalaTopicAndPartitionSet = kafkaCluster
                .getPartitions(immutableTopics)
                .right()
                .get();

        if (offset != null || kafkaCluster.getConsumerOffsets(kafkaParams.get("group.id"),
                scalaTopicAndPartitionSet).isLeft()) {

            // 等于空则设置为0
            offset = (offset == null ? 0L : offset);

            // 设置每个分区的offset
            scala.collection.Iterator<TopicAndPartition> iterator = scalaTopicAndPartitionSet.iterator();
            while (iterator.hasNext()) {
                fromOffsets.put(iterator.next(), offset);
            }
        } else {
            // 往后继续读取
            scala.collection.Map<TopicAndPartition, Object> consumerOffsets = kafkaCluster
                    .getConsumerOffsets(groupId,
                            scalaTopicAndPartitionSet).right().get();

            scala.collection.Iterator<Tuple2<TopicAndPartition, Object>> iterator = consumerOffsets.iterator();
            while (iterator.hasNext()) {
                Tuple2<TopicAndPartition, Object> next = iterator.next();
                offset = (long) next._2();
                fromOffsets.put(next._1(), offset);
            }
        }

        // 记录处理时间
        LOG.info("fromOffsets time:" + (System.currentTimeMillis() - l));

        return fromOffsets;
    }

    /**
     * 将kafkaParams转换成scala map，用于创建kafkaCluster
     *
     * @param kafkaParams kafka参数配置
     * @return kafkaCluster管理工具类
     */
    private static KafkaCluster getKafkaCluster(HashMap<String, String> kafkaParams) {
        // 类型转换
        scala.collection.immutable.Map<String, String> immutableKafkaParam = JavaConversions
                .mapAsScalaMap(kafkaParams)
                .toMap(new Predef.$less$colon$less<Tuple2<String, String>, Tuple2<String, String>>() {
                    public Tuple2<String, String> apply(
                            Tuple2<String, String> v1) {
                        return v1;
                    }
                });

        return new KafkaCluster(immutableKafkaParam);
    }
}
```

上面程序只是一个简单的读取kafka数据并打印的示例，采用`KafkaUtils.createDirectStream`来创建streaming，直接从kafka的broker服务上读取数据，存储并设置`offset`。

为了确保task不会重复执行请设置下面两个参数：
`spark.task.maxFailures=1`，Task重试次数为1，即不重试
`spark.speculation=false`，关闭推测执行

## 参考文档

- http://spark.apache.org/docs/latest/streaming-kafka-integration.html