---
title: kafka生产者消费者示例详解
description: kafka生产者消费者示例，通过官方例子自定义高级查询，并对一些主要参数进行详细描述。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2016-3-16 20:04:29
---

## 背景

kafka生产者消费者示例，通过官方例子自定义高级查询，并对一些主要参数进行详细描述。

## Producer

生产者：
```
import kafka.javaapi.producer.Producer;
import kafka.producer.KeyedMessage;
import kafka.producer.ProducerConfig;

import java.util.Properties;

/**
 * Created by hua on 2016/2/21.
 */

public class KafkaProducer {
    private final Producer<String, String> producer;

    private String topic;
    private String brokerList;
    private String acks;

    public KafkaProducer(String topic, String brokerList, String acks) {
        this.topic = topic;
        this.brokerList = brokerList;
        this.acks = acks;

        Properties props = new Properties();
        props.put("metadata.broker.list", brokerList);

        // 指定序列化处理类，默认为kafka.serializer.DefaultEncoder,即byte[]
        props.put("serializer.class", "kafka.serializer.StringEncoder");
        props.put("key.serializer.class", "kafka.serializer.StringEncoder");

        // 同步还是异步，默认aync同步，async异步。异步可以提高发送吞吐量，但是也可能导致丢失未发送过去的消息
        props.put("producer.type", "sync");

        // 是否压缩，默认0表示不压缩，1表示用gzip压缩，2表示用snappy压缩。压缩后消息中会有头来指明消息压缩类型，故在消费者端消息解压是透明的无需指定。
        props.put("compression.codec", "1");
        
        // 设置Partition类, 对队列进行合理的划分(默认是Hash分区)
        //props.put("partitioner.class", "kafka.producer.DefaultPartitioner");

        //request.required.acks
        // 0，这意味着，生产者从不等待一个确认从经纪人（相同的行为为0.7）。此选项提供了最低的延迟，但最弱的耐久性保证（一些数据将丢失，当服务器失败）。
        // 1，这意味着生产者得到一个确认后的领导者副本已收到的数据。此选项提供了更好的耐久性，因为客户端等待，直到服务器确认请求成功（只有消息被写入到现在的死亡的领导者，但尚未复制将被丢失）。
        //-1，这意味着，生产者得到一个确认后，所有的同步副本已收到的数据。此选项提供了最佳的耐久性，我们保证，没有任何消息会丢失，只要至少有一个在同步副本仍然。
        props.put("request.required.acks", acks);
        producer = new Producer<String, String>(new ProducerConfig(props));
    }

    public void send(String key, String data) {
        System.out.println(data);
        // send(List<KeyedMessage<K, V>> messages);
        producer.send(new KeyedMessage<String, String>(topic, key, data));
    }

    public static void main(String[] args) {
        KafkaProducer producer = new KafkaProducer("test1", "hynamet02:9082", "-1");
        producer.send("0", "hello,world!");
    }
}
```


## Consumer

消费者：
```
import kafka.consumer.ConsumerConfig;
import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;
import kafka.message.MessageAndMetadata;
import kafka.serializer.StringDecoder;
import kafka.utils.VerifiableProperties;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;

/**
 * Created by hua on 2016/2/21.
 */
public class KafkaConsumer {

    private final ConsumerConnector consumer;

    private KafkaConsumer() {
        Properties props = new Properties();
        //zookeeper 配置
        props.put("zookeeper.connect", "hynamet01:2181");

        // group 代表一个消费组
        props.put("group.id", "hua-group");

        // 连接zk的session超时时间
        props.put("zookeeper.session.timeout.ms", "4000");
        // 默认是会自动commit offset到zk的，
        // 如果要自己commit，设auto.commit.enable为false
        props.put("auto.commit.interval.ms", "1000");
        // 往zookeeper上写offset的频率,如果offset出了返回，则 smallest: 自动设置reset到最小的offset. largest : 自动设置offset到最大的offset. 其它值不允许，会抛出异常
        props.put("auto.offset.reset", "smallest");
        props.put("zookeeper.sync.time.ms", "200");
        //序列化类
        props.put("serializer.class", "kafka.serializer.StringEncoder");

        ConsumerConfig config = new ConsumerConfig(props);
        consumer = kafka.consumer.Consumer.createJavaConsumerConnector(config);
    }

    void consume() {
        Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
        topicCountMap.put("test", new Integer(1));

        StringDecoder keyDecoder = new StringDecoder(new VerifiableProperties());
        StringDecoder valueDecoder = new StringDecoder(new VerifiableProperties());

        Map<String, List<KafkaStream<String, String>>> consumerMap =
                consumer.createMessageStreams(topicCountMap, keyDecoder, valueDecoder);
        KafkaStream<String, String> stream = consumerMap.get("test").get(0);

        ConsumerIterator<String, String> it = stream.iterator();
        while (it.hasNext()) {
            MessageAndMetadata<String, String> next = it.next();
            System.out.println(next.key() + "\t" + next.message());
        }
    }

    public static void main(String[] args) {
        new KafkaConsumer().consume();
    }
}


```

参数说明：
`group.id`：一个消费组ID，每个消费组都有自己的消费偏移量，不支持多线程用同一group.id（一个行数据只会被一个线程读取）


**分区、Offset、消费线程、group.id的关系**
1）一组（类）消息通常由某个topic来归类，我们可以把这组消息“分发”给若干个分区（partition），每个分区的消息各不相同；
2）每个分区都维护着他自己的偏移量（Offset），记录着该分区的消息此时被消费的位置；
3）一个消费线程可以对应若干个分区，但一个分区只能被具体某一个消费线程消费；
4）group.id用于标记某一个消费组，每一个消费组都会被记录他在某一个分区的Offset，即不同consumer group针对同一个分区，都有“各自”的偏移量。

## 自定义高级查询

 Kafka JAVA客户端代码示例,使用场景：
 * 针对一个消息读取多次
 * 在一个process中，仅仅处理一个topic中的一组partitions
 * 使用事务，确保每个消息只被处理一次

```
package com.hua.kafka;

import kafka.api.FetchRequest;
import kafka.api.FetchRequestBuilder;
import kafka.api.PartitionOffsetRequestInfo;
import kafka.common.ErrorMapping;
import kafka.common.TopicAndPartition;
import kafka.javaapi.*;
import kafka.javaapi.consumer.SimpleConsumer;
import kafka.message.MessageAndOffset;

import java.nio.ByteBuffer;
import java.util.*;


/**
 * 
 * <p/>
 * Kafka JAVA客户端代码示例,使用场景：
 * 针对一个消息读取多次
 * 在一个process中，仅仅处理一个topic中的一组partitions
 * 使用事务，确保每个消息只被处理一次
 */
public class SimpleExample {
    public static void main(String args[]) {
        SimpleExample example = new SimpleExample();
        long maxReads = 20;
        String topic = "test1";
        int partition = 2;
        List<String> seeds = new ArrayList<String>();
        seeds.add("cdh-s1");
        int port = 9092;
        try {
            example.run(maxReads, topic, partition, seeds, port);
        } catch (Exception e) {
            System.out.println("Oops:" + e);
            e.printStackTrace();
        }
    }

    private List<String> m_replicaBrokers = new ArrayList<String>();

    public SimpleExample() {
        m_replicaBrokers = new ArrayList<String>();
    }

    public void run(long a_maxReads, String a_topic, int a_partition,
                    List<String> a_seedBrokers, int a_port) throws Exception {
        // find the meta data about the topic and partition we are interested in
        //
        PartitionMetadata metadata = findLeader(a_seedBrokers, a_port, a_topic,
                a_partition);
        if (metadata == null) {
            System.out
                    .println("Can't find metadata for Topic and Partition. Exiting");
            return;
        }
        if (metadata.leader() == null) {
            System.out
                    .println("Can't find Leader for Topic and Partition. Exiting");
            return;
        }
        String leadBroker = metadata.leader().host();
        String clientName = "Client_" + a_topic + "_" + a_partition;

        SimpleConsumer consumer = new SimpleConsumer(leadBroker, a_port,
                100000, 64 * 1024, clientName);
        long readOffset = getLastOffset(consumer, a_topic, a_partition,
                kafka.api.OffsetRequest.EarliestTime(), clientName);

        int numErrors = 0;
        while (a_maxReads > 0) {
            if (consumer == null) {
                consumer = new SimpleConsumer(leadBroker, a_port, 100000,
                        64 * 1024, clientName);
            }
            FetchRequest req = new FetchRequestBuilder().clientId(clientName)
                    .addFetch(a_topic, a_partition, readOffset, 100000) // Note:
                    // this
                    // fetchSize
                    // of
                    // 100000
                    // might
                    // need
                    // to be
                    // increased
                    // if
                    // large
                    // batches
                    // are
                    // written
                    // to
                    // Kafka
                    .build();
            FetchResponse fetchResponse = consumer.fetch(req);

            if (fetchResponse.hasError()) {
                numErrors++;
                // Something went wrong!
                short code = fetchResponse.errorCode(a_topic, a_partition);
                System.out.println("Error fetching data from the Broker:"
                        + leadBroker + " Reason: " + code);
                if (numErrors > 5)
                    break;
                if (code == ErrorMapping.OffsetOutOfRangeCode()) {
                    // We asked for an invalid offset. For simple case ask for
                    // the last element to reset
                    readOffset = getLastOffset(consumer, a_topic, a_partition,
                            kafka.api.OffsetRequest.LatestTime(), clientName);
                    continue;
                }
                consumer.close();
                consumer = null;
                leadBroker = findNewLeader(leadBroker, a_topic, a_partition,
                        a_port);
                continue;
            }
            numErrors = 0;

            long numRead = 0;
            for (MessageAndOffset messageAndOffset : fetchResponse.messageSet(
                    a_topic, a_partition)) {
                long currentOffset = messageAndOffset.offset();
                if (currentOffset < readOffset) {
                    System.out.println("Found an old offset: " + currentOffset
                            + " Expecting: " + readOffset);
                    continue;
                }
                readOffset = messageAndOffset.nextOffset();
                ByteBuffer payload = messageAndOffset.message().payload();

                byte[] bytes = new byte[payload.limit()];
                payload.get(bytes);
                System.out.println(String.valueOf(messageAndOffset.offset())
                        + ": " + new String(bytes, "UTF-8"));
                numRead++;
                a_maxReads--;
            }

            if (numRead == 0) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ie) {
                }
            }
        }
        if (consumer != null)
            consumer.close();
    }

    public static long getLastOffset(SimpleConsumer consumer, String topic,
                                     int partition, long whichTime, String clientName) {
        TopicAndPartition topicAndPartition = new TopicAndPartition(topic,
                partition);
        Map<TopicAndPartition, PartitionOffsetRequestInfo> requestInfo = new HashMap<TopicAndPartition, PartitionOffsetRequestInfo>();
        requestInfo.put(topicAndPartition, new PartitionOffsetRequestInfo(
                whichTime, 1));
        kafka.javaapi.OffsetRequest request = new kafka.javaapi.OffsetRequest(
                requestInfo, kafka.api.OffsetRequest.CurrentVersion(),
                clientName);
        OffsetResponse response = consumer.getOffsetsBefore(request);

        if (response.hasError()) {
            System.out
                    .println("Error fetching data Offset Data the Broker. Reason: "
                            + response.errorCode(topic, partition));
            return 0;
        }
        long[] offsets = response.offsets(topic, partition);
        return offsets[0];
    }

    private String findNewLeader(String a_oldLeader, String a_topic,
                                 int a_partition, int a_port) throws Exception {
        for (int i = 0; i < 3; i++) {
            boolean goToSleep = false;
            PartitionMetadata metadata = findLeader(m_replicaBrokers, a_port,
                    a_topic, a_partition);
            if (metadata == null) {
                goToSleep = true;
            } else if (metadata.leader() == null) {
                goToSleep = true;
            } else if (a_oldLeader.equalsIgnoreCase(metadata.leader().host())
                    && i == 0) {
                // first time through if the leader hasn't changed give
                // ZooKeeper a second to recover
                // second time, assume the broker did recover before failover,
                // or it was a non-Broker issue
                //
                goToSleep = true;
            } else {
                return metadata.leader().host();
            }
            if (goToSleep) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ie) {
                }
            }
        }
        System.out
                .println("Unable to find new leader after Broker failure. Exiting");
        throw new Exception(
                "Unable to find new leader after Broker failure. Exiting");
    }

    private PartitionMetadata findLeader(List<String> a_seedBrokers,
                                         int a_port, String a_topic, int a_partition) {
        PartitionMetadata returnMetaData = null;
        loop:
        for (String seed : a_seedBrokers) {
            SimpleConsumer consumer = null;
            try {
                consumer = new SimpleConsumer(seed, a_port, 100000, 64 * 1024,
                        "leaderLookup");
                List<String> topics = Collections.singletonList(a_topic);
                TopicMetadataRequest req = new TopicMetadataRequest(topics);
                kafka.javaapi.TopicMetadataResponse resp = consumer.send(req);

                List<TopicMetadata> metaData = resp.topicsMetadata();
                for (TopicMetadata item : metaData) {
                    for (PartitionMetadata part : item.partitionsMetadata()) {
                        if (part.partitionId() == a_partition) {
                            returnMetaData = part;
                            break loop;
                        }
                    }
                }
            } catch (Exception e) {
                System.out.println("Error communicating with Broker [" + seed
                        + "] to find Leader for [" + a_topic + ", "
                        + a_partition + "] Reason: " + e);
            } finally {
                if (consumer != null)
                    consumer.close();
            }
        }
        if (returnMetaData != null) {
            m_replicaBrokers.clear();
            for (kafka.cluster.Broker replica : returnMetaData.replicas()) {
                m_replicaBrokers.add(replica.host());
            }
        }
        return returnMetaData;
    }
}

```

在后续一段时间，我会把kafka针对topic多分区多线程生产与消费代码整理出来。

## 参考资料

- https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example
- http://kafka.apache.org/081/documentation.html