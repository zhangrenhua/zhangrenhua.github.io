---
title: Spark Eclipse本地运行以及远程debug调试
description: 以前使用java自带的远程端口调试Spark，感觉很累。今天突然心血来潮，决定要在eclipse上运行spark，但是在网络上面找不到这样的文章，几乎都是eclipse debug远程调试的。下面我将eclipse本地运行Spark与eclipse远程调试两种方式记录下来。
tags: [hadoop生态圈, spark]
categories: spark
date: 2015-12-24 14:54:20
---

## 背景

以前使用java自带的远程端口调试Spark，感觉很累。今天突然心血来潮，决定要在eclipse上运行spark，但是在网络上面找不到这样的文章，几乎都是eclipse debug远程调试的。下面我将eclipse本地运行Spark与eclipse远程调试两种方式记录下来。

## 本地运行Spark

本地模式运行Spark其实就是将启动模式设置为local即可，与IDE无关，使用eclipse、intellij、netbeans都可以。初学者和验证性程序可以使用此方法。

### 数据文件准备

data.txt：
```
张仁华，你好
```

将data.txt文件放入到D盘下，记得文件要UTF-8编码格式的哦（否则会乱码）。

### jar包引用
```
spark-assembly-1.4.1-hadoop2.4.0.jar
```

本人使用Spark1.4.1作为测试例子，根据实际情况来选择你自己版本的jar文件。


### 示例代码

Demo.java：
```
import java.util.ArrayList;
import java.util.List;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.sql.DataFrame;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.RowFactory;
import org.apache.spark.sql.SQLContext;
import org.apache.spark.sql.types.DataTypes;
import org.apache.spark.sql.types.StructField;
import org.apache.spark.sql.types.StructType;

public class Demo {

    public static void main(String[] args) {
        SparkConf sparkConf = new SparkConf();
        sparkConf.setMaster("local[2]").setAppName("demo");
        try (JavaSparkContext ctx = new JavaSparkContext(sparkConf);) {
            SQLContext sql = new SQLContext(ctx.sc());

            String input = "d:/data.txt";

            // 使用ctx读取文件
            JavaRDD<String> textFile = ctx.textFile(input);
            System.out.println("textFile:" + textFile.collect());

            // 使用hadoop API读取文件
            JavaPairRDD<LongWritable, Text> newAPIHadoopFile = ctx.newAPIHadoopFile(input, TextInputFormat.class,
                    LongWritable.class, Text.class, new Configuration());
            System.out.println("hadoop api:" + newAPIHadoopFile.count());

            /**
             * 使用sqlContext 来操作DataFrame
             */
            // Generate the schema based on the string of schema
            List<StructField> fields = new ArrayList<StructField>();
            fields.add(DataTypes.createStructField("txt", DataTypes.StringType, true));
            StructType schema = DataTypes.createStructType(fields);

            DataFrame df = sql.createDataFrame(textFile.map(new Function<String, Row>() {

                /**
                 * 
                 */
                private static final long serialVersionUID = -7045738234651396459L;

                @Override
                public Row call(String arg0) throws Exception {
                    return RowFactory.create(arg0);
                }
            }), schema);
            sql.registerDataFrameAsTable(df, "tmp");

            DataFrame query = sql.sql("select * from tmp");
            List<Row> collectAsList = query.collectAsList();
            for (Row row : collectAsList) {
                System.out.println(row.getString(0));
            }
        }
    }
}
```

本地运行必须设置为local模式，local[x] x代表使用cpu的核个数。
`sparkConf.setMaster("local[2]")`

### 传递给spark的master url可以有如下几种：
**local** 本地单线程
**local[K]** 本地多线程（指定K个内核）
**local[*]** 本地多线程（指定所有可用内核）
**spark://HOST:PORT**  连接到指定的 Spark standalone cluster master，需要指定端口。
**mesos://HOST:PORT**  连接到指定的  Mesos 集群，需要指定端口。
**yarn-client** 客户端模式 连接到 YARN 集群。需要配置 HADOOP_CONF_DIR。
**yarn-cluster** 集群模式 连接到 YARN 集群。需要配置 HADOOP_CONF_DIR。

### 项目结构

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-spark-debug-jgt.png)

### 运行结果
```
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
15/12/24 20:21:42 INFO SparkContext: Running Spark version 1.4.1
15/12/24 20:21:43 INFO SecurityManager: Changing view acls to: hua
15/12/24 20:21:43 INFO SecurityManager: Changing modify acls to: hua
15/12/24 20:21:43 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(hua); users with modify permissions: Set(hua)
15/12/24 20:21:43 INFO Slf4jLogger: Slf4jLogger started
15/12/24 20:21:43 INFO Remoting: Starting remoting
15/12/24 20:21:43 INFO Remoting: Remoting started; listening on addresses :[akka.tcp://sparkDriver@192.168.10.102:58083]
15/12/24 20:21:43 INFO Utils: Successfully started service 'sparkDriver' on port 58083.
15/12/24 20:21:43 INFO SparkEnv: Registering MapOutputTracker
15/12/24 20:21:43 INFO SparkEnv: Registering BlockManagerMaster
15/12/24 20:21:43 INFO DiskBlockManager: Created local directory at C:\Users\hua\AppData\Local\Temp\spark-4d6396b2-f348-41a4-b2cd-2e88ea8513f6\blockmgr-45825136-42e1-4ecb-8ef7-416de1892d24
15/12/24 20:21:43 INFO MemoryStore: MemoryStore started with capacity 1952.6 MB
15/12/24 20:21:43 INFO HttpFileServer: HTTP File server directory is C:\Users\hua\AppData\Local\Temp\spark-4d6396b2-f348-41a4-b2cd-2e88ea8513f6\httpd-0ad48254-d056-4619-8c6c-185b6f9358ca
15/12/24 20:21:43 INFO HttpServer: Starting HTTP Server
15/12/24 20:21:44 INFO Utils: Successfully started service 'HTTP file server' on port 58084.
15/12/24 20:21:44 INFO SparkEnv: Registering OutputCommitCoordinator
15/12/24 20:21:44 INFO Utils: Successfully started service 'SparkUI' on port 4040.
15/12/24 20:21:44 INFO SparkUI: Started SparkUI at http://192.168.10.102:4040
15/12/24 20:21:44 INFO Executor: Starting executor ID driver on host localhost
15/12/24 20:21:44 INFO Utils: Successfully started service 'org.apache.spark.network.netty.NettyBlockTransferService' on port 58121.
15/12/24 20:21:44 INFO NettyBlockTransferService: Server created on 58121
15/12/24 20:21:44 INFO BlockManagerMaster: Trying to register BlockManager
15/12/24 20:21:44 INFO BlockManagerMasterEndpoint: Registering block manager localhost:58121 with 1952.6 MB RAM, BlockManagerId(driver, localhost, 58121)
15/12/24 20:21:44 INFO BlockManagerMaster: Registered BlockManager
15/12/24 20:21:44 INFO MemoryStore: ensureFreeSpace(143840) called with curMem=0, maxMem=2047491440
15/12/24 20:21:45 INFO MemoryStore: Block broadcast_0 stored as values in memory (estimated size 140.5 KB, free 1952.5 MB)
15/12/24 20:21:45 INFO MemoryStore: ensureFreeSpace(12633) called with curMem=143840, maxMem=2047491440
15/12/24 20:21:45 INFO MemoryStore: Block broadcast_0_piece0 stored as bytes in memory (estimated size 12.3 KB, free 1952.5 MB)
15/12/24 20:21:45 INFO BlockManagerInfo: Added broadcast_0_piece0 in memory on localhost:58121 (size: 12.3 KB, free: 1952.6 MB)
15/12/24 20:21:45 INFO SparkContext: Created broadcast 0 from textFile at Demo.java:34
15/12/24 20:21:45 INFO FileInputFormat: Total input paths to process : 1
15/12/24 20:21:45 INFO SparkContext: Starting job: collect at Demo.java:35
15/12/24 20:21:45 INFO DAGScheduler: Got job 0 (collect at Demo.java:35) with 2 output partitions (allowLocal=false)
15/12/24 20:21:45 INFO DAGScheduler: Final stage: ResultStage 0(collect at Demo.java:35)
15/12/24 20:21:45 INFO DAGScheduler: Parents of final stage: List()
15/12/24 20:21:45 INFO DAGScheduler: Missing parents: List()
15/12/24 20:21:45 INFO DAGScheduler: Submitting ResultStage 0 (MapPartitionsRDD[1] at textFile at Demo.java:34), which has no missing parents
15/12/24 20:21:45 INFO MemoryStore: ensureFreeSpace(3112) called with curMem=156473, maxMem=2047491440
15/12/24 20:21:45 INFO MemoryStore: Block broadcast_1 stored as values in memory (estimated size 3.0 KB, free 1952.5 MB)
15/12/24 20:21:45 INFO MemoryStore: ensureFreeSpace(1779) called with curMem=159585, maxMem=2047491440
15/12/24 20:21:45 INFO MemoryStore: Block broadcast_1_piece0 stored as bytes in memory (estimated size 1779.0 B, free 1952.5 MB)
15/12/24 20:21:45 INFO BlockManagerInfo: Added broadcast_1_piece0 in memory on localhost:58121 (size: 1779.0 B, free: 1952.6 MB)
15/12/24 20:21:45 INFO SparkContext: Created broadcast 1 from broadcast at DAGScheduler.scala:874
15/12/24 20:21:45 INFO DAGScheduler: Submitting 2 missing tasks from ResultStage 0 (MapPartitionsRDD[1] at textFile at Demo.java:34)
15/12/24 20:21:45 INFO TaskSchedulerImpl: Adding task set 0.0 with 2 tasks
15/12/24 20:21:45 INFO TaskSetManager: Starting task 0.0 in stage 0.0 (TID 0, localhost, PROCESS_LOCAL, 1390 bytes)
15/12/24 20:21:45 INFO TaskSetManager: Starting task 1.0 in stage 0.0 (TID 1, localhost, PROCESS_LOCAL, 1390 bytes)
15/12/24 20:21:45 INFO Executor: Running task 0.0 in stage 0.0 (TID 0)
15/12/24 20:21:45 INFO Executor: Running task 1.0 in stage 0.0 (TID 1)
15/12/24 20:21:45 INFO HadoopRDD: Input split: file:/d:/data.txt:9+9
15/12/24 20:21:45 INFO HadoopRDD: Input split: file:/d:/data.txt:0+9
15/12/24 20:21:45 INFO deprecation: mapred.tip.id is deprecated. Instead, use mapreduce.task.id
15/12/24 20:21:45 INFO deprecation: mapred.task.id is deprecated. Instead, use mapreduce.task.attempt.id
15/12/24 20:21:45 INFO deprecation: mapred.task.is.map is deprecated. Instead, use mapreduce.task.ismap
15/12/24 20:21:45 INFO deprecation: mapred.task.partition is deprecated. Instead, use mapreduce.task.partition
15/12/24 20:21:45 INFO deprecation: mapred.job.id is deprecated. Instead, use mapreduce.job.id
15/12/24 20:21:45 INFO Executor: Finished task 0.0 in stage 0.0 (TID 0). 1813 bytes result sent to driver
15/12/24 20:21:45 INFO Executor: Finished task 1.0 in stage 0.0 (TID 1). 1792 bytes result sent to driver
15/12/24 20:21:45 INFO TaskSetManager: Finished task 0.0 in stage 0.0 (TID 0) in 60 ms on localhost (1/2)
15/12/24 20:21:45 INFO TaskSetManager: Finished task 1.0 in stage 0.0 (TID 1) in 54 ms on localhost (2/2)
15/12/24 20:21:45 INFO TaskSchedulerImpl: Removed TaskSet 0.0, whose tasks have all completed, from pool 
15/12/24 20:21:45 INFO DAGScheduler: ResultStage 0 (collect at Demo.java:35) finished in 0.070 s
15/12/24 20:21:45 INFO DAGScheduler: Job 0 finished: collect at Demo.java:35, took 0.114840 s
textFile:[张仁华，你好]
15/12/24 20:21:45 INFO MemoryStore: ensureFreeSpace(182176) called with curMem=161364, maxMem=2047491440
15/12/24 20:21:45 INFO MemoryStore: Block broadcast_2 stored as values in memory (estimated size 177.9 KB, free 1952.3 MB)
15/12/24 20:21:45 INFO MemoryStore: ensureFreeSpace(17237) called with curMem=343540, maxMem=2047491440
15/12/24 20:21:45 INFO MemoryStore: Block broadcast_2_piece0 stored as bytes in memory (estimated size 16.8 KB, free 1952.3 MB)
15/12/24 20:21:45 INFO BlockManagerInfo: Added broadcast_2_piece0 in memory on localhost:58121 (size: 16.8 KB, free: 1952.6 MB)
15/12/24 20:21:45 INFO SparkContext: Created broadcast 2 from newAPIHadoopFile at Demo.java:38
15/12/24 20:21:45 INFO FileInputFormat: Total input paths to process : 1
15/12/24 20:21:45 INFO SparkContext: Starting job: count at Demo.java:40
15/12/24 20:21:45 INFO DAGScheduler: Got job 1 (count at Demo.java:40) with 1 output partitions (allowLocal=false)
15/12/24 20:21:45 INFO DAGScheduler: Final stage: ResultStage 1(count at Demo.java:40)
15/12/24 20:21:45 INFO DAGScheduler: Parents of final stage: List()
15/12/24 20:21:45 INFO DAGScheduler: Missing parents: List()
15/12/24 20:21:45 INFO DAGScheduler: Submitting ResultStage 1 (d:/data.txt NewHadoopRDD[2] at newAPIHadoopFile at Demo.java:38), which has no missing parents
15/12/24 20:21:45 INFO MemoryStore: ensureFreeSpace(1696) called with curMem=360777, maxMem=2047491440
15/12/24 20:21:45 INFO MemoryStore: Block broadcast_3 stored as values in memory (estimated size 1696.0 B, free 1952.3 MB)
15/12/24 20:21:45 INFO MemoryStore: ensureFreeSpace(1054) called with curMem=362473, maxMem=2047491440
15/12/24 20:21:45 INFO MemoryStore: Block broadcast_3_piece0 stored as bytes in memory (estimated size 1054.0 B, free 1952.3 MB)
15/12/24 20:21:45 INFO BlockManagerInfo: Added broadcast_3_piece0 in memory on localhost:58121 (size: 1054.0 B, free: 1952.6 MB)
15/12/24 20:21:45 INFO SparkContext: Created broadcast 3 from broadcast at DAGScheduler.scala:874
15/12/24 20:21:45 INFO DAGScheduler: Submitting 1 missing tasks from ResultStage 1 (d:/data.txt NewHadoopRDD[2] at newAPIHadoopFile at Demo.java:38)
15/12/24 20:21:45 INFO TaskSchedulerImpl: Adding task set 1.0 with 1 tasks
15/12/24 20:21:45 INFO TaskSetManager: Starting task 0.0 in stage 1.0 (TID 2, localhost, PROCESS_LOCAL, 1422 bytes)
15/12/24 20:21:45 INFO Executor: Running task 0.0 in stage 1.0 (TID 2)
15/12/24 20:21:45 INFO NewHadoopRDD: Input split: file:/d:/data.txt:0+18
15/12/24 20:21:45 INFO Executor: Finished task 0.0 in stage 1.0 (TID 2). 1830 bytes result sent to driver
15/12/24 20:21:45 INFO TaskSetManager: Finished task 0.0 in stage 1.0 (TID 2) in 11 ms on localhost (1/1)
15/12/24 20:21:45 INFO DAGScheduler: ResultStage 1 (count at Demo.java:40) finished in 0.012 s
15/12/24 20:21:45 INFO TaskSchedulerImpl: Removed TaskSet 1.0, whose tasks have all completed, from pool 
15/12/24 20:21:45 INFO DAGScheduler: Job 1 finished: count at Demo.java:40, took 0.020939 s
hadoop api:1
15/12/24 20:21:45 INFO BlockManagerInfo: Removed broadcast_3_piece0 on localhost:58121 in memory (size: 1054.0 B, free: 1952.6 MB)
15/12/24 20:21:45 INFO BlockManagerInfo: Removed broadcast_1_piece0 on localhost:58121 in memory (size: 1779.0 B, free: 1952.6 MB)
15/12/24 20:21:46 INFO SparkContext: Starting job: collectAsList at Demo.java:65
15/12/24 20:21:46 INFO DAGScheduler: Got job 2 (collectAsList at Demo.java:65) with 2 output partitions (allowLocal=false)
15/12/24 20:21:46 INFO DAGScheduler: Final stage: ResultStage 2(collectAsList at Demo.java:65)
15/12/24 20:21:46 INFO DAGScheduler: Parents of final stage: List()
15/12/24 20:21:46 INFO DAGScheduler: Missing parents: List()
15/12/24 20:21:46 INFO DAGScheduler: Submitting ResultStage 2 (MapPartitionsRDD[5] at collectAsList at Demo.java:65), which has no missing parents
15/12/24 20:21:46 INFO MemoryStore: ensureFreeSpace(5128) called with curMem=355886, maxMem=2047491440
15/12/24 20:21:46 INFO MemoryStore: Block broadcast_4 stored as values in memory (estimated size 5.0 KB, free 1952.3 MB)
15/12/24 20:21:46 INFO MemoryStore: ensureFreeSpace(2786) called with curMem=361014, maxMem=2047491440
15/12/24 20:21:46 INFO MemoryStore: Block broadcast_4_piece0 stored as bytes in memory (estimated size 2.7 KB, free 1952.3 MB)
15/12/24 20:21:46 INFO BlockManagerInfo: Added broadcast_4_piece0 in memory on localhost:58121 (size: 2.7 KB, free: 1952.6 MB)
15/12/24 20:21:46 INFO SparkContext: Created broadcast 4 from broadcast at DAGScheduler.scala:874
15/12/24 20:21:46 INFO DAGScheduler: Submitting 2 missing tasks from ResultStage 2 (MapPartitionsRDD[5] at collectAsList at Demo.java:65)
15/12/24 20:21:46 INFO TaskSchedulerImpl: Adding task set 2.0 with 2 tasks
15/12/24 20:21:46 INFO TaskSetManager: Starting task 0.0 in stage 2.0 (TID 3, localhost, PROCESS_LOCAL, 1390 bytes)
15/12/24 20:21:46 INFO TaskSetManager: Starting task 1.0 in stage 2.0 (TID 4, localhost, PROCESS_LOCAL, 1390 bytes)
15/12/24 20:21:46 INFO Executor: Running task 0.0 in stage 2.0 (TID 3)
15/12/24 20:21:46 INFO Executor: Running task 1.0 in stage 2.0 (TID 4)
15/12/24 20:21:46 INFO HadoopRDD: Input split: file:/d:/data.txt:9+9
15/12/24 20:21:46 INFO HadoopRDD: Input split: file:/d:/data.txt:0+9
15/12/24 20:21:46 INFO Executor: Finished task 1.0 in stage 2.0 (TID 4). 1800 bytes result sent to driver
15/12/24 20:21:46 INFO TaskSetManager: Finished task 1.0 in stage 2.0 (TID 4) in 8 ms on localhost (1/2)
15/12/24 20:21:46 INFO Executor: Finished task 0.0 in stage 2.0 (TID 3). 2775 bytes result sent to driver
15/12/24 20:21:46 INFO TaskSetManager: Finished task 0.0 in stage 2.0 (TID 3) in 12 ms on localhost (2/2)
15/12/24 20:21:46 INFO DAGScheduler: ResultStage 2 (collectAsList at Demo.java:65) finished in 0.013 s
15/12/24 20:21:46 INFO TaskSchedulerImpl: Removed TaskSet 2.0, whose tasks have all completed, from pool 
15/12/24 20:21:46 INFO DAGScheduler: Job 2 finished: collectAsList at Demo.java:65, took 0.021384 s
张仁华，你好
15/12/24 20:21:46 INFO SparkUI: Stopped Spark web UI at http://192.168.10.102:4040
15/12/24 20:21:46 INFO DAGScheduler: Stopping DAGScheduler
15/12/24 20:21:46 INFO MapOutputTrackerMasterEndpoint: MapOutputTrackerMasterEndpoint stopped!
15/12/24 20:21:46 INFO Utils: path = C:\Users\hua\AppData\Local\Temp\spark-4d6396b2-f348-41a4-b2cd-2e88ea8513f6\blockmgr-45825136-42e1-4ecb-8ef7-416de1892d24, already present as root for deletion.
15/12/24 20:21:46 INFO MemoryStore: MemoryStore cleared
15/12/24 20:21:46 INFO BlockManager: BlockManager stopped
15/12/24 20:21:46 INFO BlockManagerMaster: BlockManagerMaster stopped
15/12/24 20:21:46 INFO OutputCommitCoordinator$OutputCommitCoordinatorEndpoint: OutputCommitCoordinator stopped!
15/12/24 20:21:46 INFO SparkContext: Successfully stopped SparkContext
15/12/24 20:21:46 INFO Utils: Shutdown hook called
15/12/24 20:21:46 INFO Utils: Deleting directory C:\Users\hua\AppData\Local\Temp\spark-4d6396b2-f348-41a4-b2cd-2e88ea8513f6
15/12/24 20:21:46 INFO RemoteActorRefProvider$RemotingTerminator: Shutting down remote daemon.
15/12/24 20:21:46 INFO RemoteActorRefProvider$RemotingTerminator: Remote daemon shut down; proceeding with flushing remote transports.

```

## Eclipse远程调试

Eclipse远程调试可以使用多种模式启动，如：local、yarn-client。其原理就是使用JVM自带的远程调试方法。

### 修改Spark JVM启动参数

**这里对上面的几个参数进行说明：**
**-Xdebug** 启用调试特性
**-Xrunjdwp** 启用JDWP实现，包含若干子选项：
transport=dt_socket JPDA front-end和back-end之间的传输方法。dt_socket表示使用套接字传输。
address=8888 JVM在8888端口上监听请求，这个设定为一个不冲突的端口即可。
server=y y表示启动的JVM是被调试者。如果为n，则表示启动的JVM是调试器。
suspend=y y表示启动的JVM会暂停等待，直到调试器连接上才继续执行。suspend=n，则JVM不会暂停等待。

#### Spark1.4以前的版本修改

修改$SPARK_HOME/bin/spark-class，大约在128行左右加入下面代码：
```
JAVA_OPTS="-XX:MaxPermSize=128m -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=8888 $JAVA_OPTS"
```

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-spark-debug-upt.png)

#### Spark1.4或者更新的版本
修改$SPARK_HOME/bin/spark-class，大约在倒数5行左右修改原来的代码：
```
done < <("$RUNNER" -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=8888 "$@")
```

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-spark-debug-uplt.png)

### spark-submit启动

启动Spark Application的时候我们可以在终端看到监听启动的信息，并且会卡住直到Eclipse启动调试(必须使用yarn-client or local模式启动)：
```
spark-submit --class spark.Demo --master yarn-client /home/hadoop/apptest/spark-demo.jar /tmp/data.txt --executor-memory 1024m
```

### 在Eclipse设置远程调试的IP和Port 

点击左上角：Run -> Debug Configurations，新建一个Remote Java Application启动配置

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-spark-debug-eclipse.png)

**Project：**选择Spark工程
**Host：**远程Spark启动的机器ip
**Port：**Spark监听端口，如：8888

配置好后点击debug，运行即可断点联调。运行的jar文件必须要和Eclipse上的源码保持一致，如有疑问直接回复。