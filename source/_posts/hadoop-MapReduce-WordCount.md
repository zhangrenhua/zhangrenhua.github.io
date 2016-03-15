---
title: MapReduce开发-WordCount
description: 前面已经对MapReduce处理流程以及运行原理做了比较细致的讲解，下面直接进入MapReduce编程阶段。本文主要讲解官方的一个简要例子（WordCount），从这个简单的例子中贯穿整个MapReduce执行的过程，从中理解MapReduce作业的工作原理。本文采用eclipse本地运行方式，并讲解如何debug调试。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-12-08 19:10:00
---

## 背景

前面已经对MapReduce处理流程以及运行原理做了比较细致的讲解，下面直接进入MapReduce编程阶段。本文主要讲解官方的一个简要例子（WordCount），从这个简单的例子中贯穿整个MapReduce执行的过程，从中理解MapReduce作业的工作原理。本文采用eclipse本地运行方式，并讲解如何debug调试。

## 数据文件

data.txt
```
hello world
b l c d
a e f h i g
k g m  
```

上传文件至hdfs:
```
hadoop fs -mkdir -p /tmp/zrh/input
hadoop fs -put data.txt /tmp/zrh/input
```

## jar包引用

```
avro-1.7.4.jar
commons-cli-1.2.jar
commons-configuration-1.6.jar
commons-httpclient-3.1.jar
commons-lang-2.6.jar
commons-logging-1.1.1.jar
guava-12.0.1.jar
hadoop-annotations-2.2.0.jar
hadoop-archives-2.2.0.jar
hadoop-auth-2.2.0.jar
hadoop-common-2.2.0.jar
hadoop-datajoin-2.2.0.jar
hadoop-distcp-2.2.0.jar
hadoop-extras-2.2.0.jar
hadoop-gridmix-2.2.0.jar
hadoop-hdfs-2.2.0.jar
hadoop-hdfs-nfs-2.2.0.jar
hadoop-mapreduce-client-app-2.2.0.jar
hadoop-mapreduce-client-common-2.2.0.jar
hadoop-mapreduce-client-core-2.2.0.jar
hadoop-mapreduce-client-hs-2.2.0.jar
hadoop-mapreduce-client-hs-plugins-2.2.0.jar
hadoop-mapreduce-client-jobclient-2.2.0-tests.jar
hadoop-mapreduce-client-jobclient-2.2.0.jar
hadoop-mapreduce-client-shuffle-2.2.0.jar
hadoop-mapreduce-examples-2.2.0.jar
hadoop-nfs-2.2.0.jar
hadoop-rumen-2.2.0.jar
hadoop-streaming-2.2.0.jar
hadoop-yarn-api-2.2.0.jar
hadoop-yarn-applications-distributedshell-2.2.0.jar
hadoop-yarn-applications-unmanaged-am-launcher-2.2.0.jar
hadoop-yarn-client-2.2.0.jar
hadoop-yarn-common-2.2.0.jar
hadoop-yarn-server-common-2.2.0.jar
hadoop-yarn-server-nodemanager-2.2.0.jar
hadoop-yarn-server-resourcemanager-2.2.0.jar
hadoop-yarn-server-web-proxy-2.2.0.jar
hadoop-yarn-site-2.2.0.jar
jackson-core-asl-1.8.8.jar
jackson-mapper-asl-1.8.8.jar
log4j-1.2.17.jar
protobuf-java-2.5.0.jar
slf4j-api-1.6.4.jar
slf4j-log4j12-1.6.4.jar
```

## 集群配置文件

```
core-site.xml
hdfs-site.xml
log4j.properties
```

注：将hadoop集群中的两个配置文件放到项目的src目录下，并编写log4j文件以便于执行的时候能查看到MapReduce中的日志信息（log4j可以从hadoop源码中随便拿个 比较全的）。
<font color="red">如果不加上面两个xml文件，则以本地模式运行</font>

## WordCountTest.java

```
import java.io.IOException;
import java.util.StringTokenizer;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.partition.HashPartitioner;
import org.apache.hadoop.util.GenericOptionsParser;

public class WordCountTest {

    /**
     * MapReduceBase类:实现了Mapper和Reducer接口的基类（其中的方法只是实现接口，而未作任何事情） Mapper接口：
     * WritableComparable接口：实现WritableComparable的类可以相互比较。所有被用作key的类应该实现此接口。
     * Reporter 则可用于报告整个应用的运行进度，本例中未使用。
     * 
     */
    public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {

        /**
         * LongWritable, IntWritable, Text 均是 Hadoop 中实现的用于封装 Java
         * 数据类型的类，这些类实现了WritableComparable接口，
         * 都能够被串行化从而便于在分布式环境中进行数据交换，你可以将它们分别视为long,int,String 的替代品。
         */
        private final static IntWritable one = new IntWritable(1);
        private Text word = new Text();// Text 实现了BinaryComparable类可以作为key值

        /**
         * Mapper接口中的map方法： void map(K1 key, V1 value, OutputCollector
         * <K2,V2> output, Reporter reporter) 映射一个单个的输入k/v对到一个中间的k/v对
         * 输出对不需要和输入对是相同的类型，输入对可以映射到0个或多个输出对。
         * OutputCollector接口：收集Mapper和Reducer输出的<k,v>对。
         * OutputCollector接口的collect(k, v)方法:增加一个(k,v)对到output
         */

        public void map(Object key, Text value, Context context) throws IOException, InterruptedException {

            /**
             * 原始数据： c++ java hello world java hello you me too
             * map阶段，数据如下形式作为map的输入值：key为偏移量 0 c++ java hello 16 world java
             * hello 34 you me too
             */

            /**
             * 以下解析键值对 解析后以键值对格式形成输出数据 格式如下：前者是键排好序的，后者数字是值 c++ 1 java 1 hello 1
             * world 1 java 1 hello 1 you 1 me 1 too 1 这些数据作为reduce的输出数据
             */
            StringTokenizer itr = new StringTokenizer(value.toString());// 得到什么值
            System.out.println("value什么东西 ： " + value.toString());
            System.out.println("key什么东西 ： " + key.toString());

            while (itr.hasMoreTokens()) {
                word.set(itr.nextToken());

                context.write(word, one);
            }
        }
    }

    public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
        private IntWritable result = new IntWritable();

        /**
         * reduce过程是对输入数据解析形成如下格式数据： (c++ [1]) (java [1,1]) (hello [1,1]) (world
         * [1]) (you [1]) (me [1]) (you [1]) 供接下来的实现的reduce程序分析数据数据
         * 
         */
        public void reduce(Text key, Iterable<IntWritable> values, Context context)
                throws IOException, InterruptedException {
            int sum = 0;
            /**
             * 自己的实现的reduce方法分析输入数据 形成数据格式如下并存储 c++ 1 hello 2 java 2 me 1 too 1
             * world 1 you 1
             * 
             */
            for (IntWritable val : values) {
                sum += val.get();
            }

            result.set(sum);
            context.write(key, result);
        }
    }

    public static void main(String[] args) throws Exception {

        /**
         * JobConf：map/reduce的job配置类，向hadoop框架描述map-reduce执行的工作
         * 构造方法：JobConf()、JobConf(Class exampleClass)、JobConf(Configuration
         * conf)等
         */
        // 根据自己的实际情况填写输入分析的目录和结果输出的目录
        args = new String[2];

        // args[0] = "hdfs://10.242.157.115:9000/tmp/zrh/input";
        // 使用上面这种方式指定文件位置，则不需要把core-site.xml和hdfs-site.xml放入src目录，建议将集群环境的配置文件放到src目录
        args[0] = "/tmp/zrh/input";
        args[1] = "/tmp/zrh/output";

        Configuration conf = new Configuration();
        String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();
        for (String s : otherArgs) {
            System.out.println(s);
        }

        // 这里需要配置参数即输入和输出的HDFS的文件路径
        if (otherArgs.length != 2) {
            System.err.println("Usage: wordcount <in> <out>");
            System.exit(2);
        }

        // JobConf conf1 = new JobConf(WordCount.class);
        Job job = Job.getInstance(conf);// Job(Configuration conf, String
        job.setJobName("word count"); // job名称

        job.setPartitionerClass(HashPartitioner.class);
        job.setJarByClass(WordCountTest.class);
        job.setMapperClass(TokenizerMapper.class); // 为job设置Mapper类
        job.setCombinerClass(IntSumReducer.class); // 为job设置Combiner类
        job.setReducerClass(IntSumReducer.class); // 为job设置Reduce类
        job.setOutputKeyClass(Text.class); // 设置输出key的类型
        job.setOutputValueClass(IntWritable.class);// 设置输出value的类型
        FileInputFormat.addInputPath(job, new Path(otherArgs[0])); // 为map-reduce任务设置InputFormat实现类
                                                                    // 设置输入路径

        FileOutputFormat.setOutputPath(job, new Path(otherArgs[1]));// 为map-reduce任务设置OutputFormat实现类
                                                                    // 设置输出路径
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }

}
```

注：在eclipse中执行main函数既可。

## 项目结构
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-wordcount-1.png)

## 运行程序
将数据文件上传到HDFS指定目录(/tmp/zrh/input)后，运行WordCountTest类中的main函数即可。

### debug调试

与普通的java程序一样，标记断点位置，debug模式运行即可。

## 可能发生出现的异常

输出目录已经存在，使用`hadoop fs -rmr /tmp/zrh/output`删除目录即可：
```
15/12/10 13:51:23 ERROR security.UserGroupInformation: PriviledgedActionException as:hua (auth:SIMPLE) cause:org.apache.hadoop.mapred.FileAlreadyExistsException: Output directory hdfs://nameservice1/tmp/zrh/output already exists
Exception in thread "main" org.apache.hadoop.mapred.FileAlreadyExistsException: Output directory hdfs://nameservice1/tmp/zrh/output already exists
    at org.apache.hadoop.mapreduce.lib.output.FileOutputFormat.checkOutputSpecs(FileOutputFormat.java:146)
    at org.apache.hadoop.mapreduce.JobSubmitter.checkSpecs(JobSubmitter.java:456)
    at org.apache.hadoop.mapreduce.JobSubmitter.submitJobInternal(JobSubmitter.java:342)
    at org.apache.hadoop.mapreduce.Job$10.run(Job.java:1268)
    at org.apache.hadoop.mapreduce.Job$10.run(Job.java:1265)
    at java.security.AccessController.doPrivileged(Native Method)
    at javax.security.auth.Subject.doAs(Subject.java:415)
    at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1491)
    at org.apache.hadoop.mapreduce.Job.submit(Job.java:1265)
    at org.apache.hadoop.mapreduce.Job.waitForCompletion(Job.java:1286)
    at com.pactera.hadoop.WordCountTest.main(WordCountTest.java:138)
```

**如出现其它错误**，请参考《hadoop2.x-Eclipse开发环境搭建》。

## 运行结果

执行日志信息：
```
/tmp/zrh/input
/tmp/zrh/output
15/12/10 13:52:25 INFO Configuration.deprecation: session.id is deprecated. Instead, use dfs.metrics.session-id
15/12/10 13:52:25 INFO jvm.JvmMetrics: Initializing JVM Metrics with processName=JobTracker, sessionId=
15/12/10 13:52:25 WARN mapreduce.JobSubmitter: No job jar file set.  User classes may not be found. See Job or Job#setJar(String).
15/12/10 13:52:25 INFO input.FileInputFormat: Total input paths to process : 1
15/12/10 13:52:25 INFO mapreduce.JobSubmitter: number of splits:1
15/12/10 13:52:25 INFO Configuration.deprecation: user.name is deprecated. Instead, use mapreduce.job.user.name
15/12/10 13:52:25 INFO Configuration.deprecation: mapreduce.partitioner.class is deprecated. Instead, use mapreduce.job.partitioner.class
15/12/10 13:52:25 INFO Configuration.deprecation: mapred.output.value.class is deprecated. Instead, use mapreduce.job.output.value.class
15/12/10 13:52:25 INFO Configuration.deprecation: mapreduce.combine.class is deprecated. Instead, use mapreduce.job.combine.class
15/12/10 13:52:25 INFO Configuration.deprecation: mapreduce.map.class is deprecated. Instead, use mapreduce.job.map.class
15/12/10 13:52:25 INFO Configuration.deprecation: mapred.job.name is deprecated. Instead, use mapreduce.job.name
15/12/10 13:52:25 INFO Configuration.deprecation: mapreduce.reduce.class is deprecated. Instead, use mapreduce.job.reduce.class
15/12/10 13:52:25 INFO Configuration.deprecation: mapred.input.dir is deprecated. Instead, use mapreduce.input.fileinputformat.inputdir
15/12/10 13:52:25 INFO Configuration.deprecation: mapred.output.dir is deprecated. Instead, use mapreduce.output.fileoutputformat.outputdir
15/12/10 13:52:25 INFO Configuration.deprecation: mapred.map.tasks is deprecated. Instead, use mapreduce.job.maps
15/12/10 13:52:25 INFO Configuration.deprecation: mapred.output.key.class is deprecated. Instead, use mapreduce.job.output.key.class
15/12/10 13:52:25 INFO Configuration.deprecation: mapred.working.dir is deprecated. Instead, use mapreduce.job.working.dir
15/12/10 13:52:25 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_local207690334_0001
15/12/10 13:52:25 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/staging/hua207690334/.staging/job_local207690334_0001/job.xml:an attempt to override final parameter: hadoop.ssl.keystores.factory.class;  Ignoring.
15/12/10 13:52:25 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/staging/hua207690334/.staging/job_local207690334_0001/job.xml:an attempt to override final parameter: hadoop.ssl.client.conf;  Ignoring.
15/12/10 13:52:25 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/staging/hua207690334/.staging/job_local207690334_0001/job.xml:an attempt to override final parameter: hadoop.ssl.server.conf;  Ignoring.
15/12/10 13:52:25 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/staging/hua207690334/.staging/job_local207690334_0001/job.xml:an attempt to override final parameter: mapreduce.job.end-notification.max.retry.interval;  Ignoring.
15/12/10 13:52:25 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/staging/hua207690334/.staging/job_local207690334_0001/job.xml:an attempt to override final parameter: mapreduce.job.end-notification.max.attempts;  Ignoring.
15/12/10 13:52:25 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/staging/hua207690334/.staging/job_local207690334_0001/job.xml:an attempt to override final parameter: hadoop.ssl.require.client.cert;  Ignoring.
15/12/10 13:52:26 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/local/localRunner/hua/job_local207690334_0001/job_local207690334_0001.xml:an attempt to override final parameter: hadoop.ssl.keystores.factory.class;  Ignoring.
15/12/10 13:52:26 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/local/localRunner/hua/job_local207690334_0001/job_local207690334_0001.xml:an attempt to override final parameter: hadoop.ssl.client.conf;  Ignoring.
15/12/10 13:52:26 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/local/localRunner/hua/job_local207690334_0001/job_local207690334_0001.xml:an attempt to override final parameter: hadoop.ssl.server.conf;  Ignoring.
15/12/10 13:52:26 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/local/localRunner/hua/job_local207690334_0001/job_local207690334_0001.xml:an attempt to override final parameter: mapreduce.job.end-notification.max.retry.interval;  Ignoring.
15/12/10 13:52:26 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/local/localRunner/hua/job_local207690334_0001/job_local207690334_0001.xml:an attempt to override final parameter: mapreduce.job.end-notification.max.attempts;  Ignoring.
15/12/10 13:52:26 WARN conf.Configuration: file:/tmp/hadoop-hua/mapred/local/localRunner/hua/job_local207690334_0001/job_local207690334_0001.xml:an attempt to override final parameter: hadoop.ssl.require.client.cert;  Ignoring.
15/12/10 13:52:26 INFO mapreduce.Job: The url to track the job: http://localhost:8080/
15/12/10 13:52:26 INFO mapreduce.Job: Running job: job_local207690334_0001
15/12/10 13:52:26 INFO mapred.LocalJobRunner: OutputCommitter set in config null
15/12/10 13:52:26 INFO mapred.LocalJobRunner: OutputCommitter is org.apache.hadoop.mapreduce.lib.output.FileOutputCommitter
15/12/10 13:52:26 INFO mapred.LocalJobRunner: Waiting for map tasks
15/12/10 13:52:26 INFO mapred.LocalJobRunner: Starting task: attempt_local207690334_0001_m_000000_0
15/12/10 13:52:26 INFO util.ProcfsBasedProcessTree: ProcfsBasedProcessTree currently is supported only on Linux.
15/12/10 13:52:26 INFO mapred.Task:  Using ResourceCalculatorProcessTree : org.apache.hadoop.yarn.util.WindowsBasedProcessTree@7ec3cdad
15/12/10 13:52:26 INFO mapred.MapTask: Processing split: hdfs://nameservice1/tmp/zrh/input/data:0+42
15/12/10 13:52:26 INFO mapred.MapTask: Map output collector class = org.apache.hadoop.mapred.MapTask$MapOutputBuffer
15/12/10 13:52:26 INFO mapred.MapTask: (EQUATOR) 0 kvi 26214396(104857584)
15/12/10 13:52:26 INFO mapred.MapTask: mapreduce.task.io.sort.mb: 100
15/12/10 13:52:26 INFO mapred.MapTask: soft limit at 83886080
15/12/10 13:52:26 INFO mapred.MapTask: bufstart = 0; bufvoid = 104857600
15/12/10 13:52:26 INFO mapred.MapTask: kvstart = 26214396; length = 6553600
value什么东西 ： hello world
key什么东西 ： 0
value什么东西 ： b l c d
key什么东西 ： 13
value什么东西 ： a e f h i g
key什么东西 ： 22
value什么东西 ： k g m  
key什么东西 ： 35
15/12/10 13:52:26 INFO mapred.LocalJobRunner: 
15/12/10 13:52:26 INFO mapred.MapTask: Starting flush of map output
15/12/10 13:52:26 INFO mapred.MapTask: Spilling map output
15/12/10 13:52:26 INFO mapred.MapTask: bufstart = 0; bufend = 98; bufvoid = 104857600
15/12/10 13:52:26 INFO mapred.MapTask: kvstart = 26214396(104857584); kvend = 26214340(104857360); length = 57/6553600
15/12/10 13:52:26 INFO mapred.MapTask: Finished spill 0
15/12/10 13:52:26 INFO mapred.Task: Task:attempt_local207690334_0001_m_000000_0 is done. And is in the process of committing
15/12/10 13:52:26 INFO mapred.LocalJobRunner: map
15/12/10 13:52:26 INFO mapred.Task: Task 'attempt_local207690334_0001_m_000000_0' done.
15/12/10 13:52:26 INFO mapred.LocalJobRunner: Finishing task: attempt_local207690334_0001_m_000000_0
15/12/10 13:52:26 INFO mapred.LocalJobRunner: Map task executor complete.
15/12/10 13:52:26 INFO util.ProcfsBasedProcessTree: ProcfsBasedProcessTree currently is supported only on Linux.
15/12/10 13:52:26 INFO mapred.Task:  Using ResourceCalculatorProcessTree : org.apache.hadoop.yarn.util.WindowsBasedProcessTree@1a121058
15/12/10 13:52:26 INFO mapred.Merger: Merging 1 sorted segments
15/12/10 13:52:26 INFO mapred.Merger: Down to the last merge-pass, with 1 segments left of total size: 118 bytes
15/12/10 13:52:26 INFO mapred.LocalJobRunner: 
15/12/10 13:52:26 INFO Configuration.deprecation: mapred.skip.on is deprecated. Instead, use mapreduce.job.skiprecords
15/12/10 13:52:27 INFO mapreduce.Job: Job job_local207690334_0001 running in uber mode : false
15/12/10 13:52:27 INFO mapreduce.Job:  map 100% reduce 0%
15/12/10 13:52:27 INFO mapred.Task: Task:attempt_local207690334_0001_r_000000_0 is done. And is in the process of committing
15/12/10 13:52:27 INFO mapred.LocalJobRunner: 
15/12/10 13:52:27 INFO mapred.Task: Task attempt_local207690334_0001_r_000000_0 is allowed to commit now
15/12/10 13:52:27 INFO output.FileOutputCommitter: Saved output of task 'attempt_local207690334_0001_r_000000_0' to hdfs://nameservice1/tmp/zrh/output/_temporary/0/task_local207690334_0001_r_000000
15/12/10 13:52:27 INFO mapred.LocalJobRunner: reduce > reduce
15/12/10 13:52:27 INFO mapred.Task: Task 'attempt_local207690334_0001_r_000000_0' done.
15/12/10 13:52:28 INFO mapreduce.Job:  map 100% reduce 100%
15/12/10 13:52:28 INFO mapreduce.Job: Job job_local207690334_0001 completed successfully
15/12/10 13:52:28 INFO mapreduce.Job: Counters: 32
    File System Counters
        FILE: Number of bytes read=480
        FILE: Number of bytes written=396350
        FILE: Number of read operations=0
        FILE: Number of large read operations=0
        FILE: Number of write operations=0
        HDFS: Number of bytes read=84
        HDFS: Number of bytes written=64
        HDFS: Number of read operations=15
        HDFS: Number of large read operations=0
        HDFS: Number of write operations=4
    Map-Reduce Framework
        Map input records=4
        Map output records=15
        Map output bytes=98
        Map output materialized bytes=126
        Input split bytes=103
        Combine input records=15
        Combine output records=14
        Reduce input groups=14
        Reduce shuffle bytes=0
        Reduce input records=14
        Reduce output records=14
        Spilled Records=28
        Shuffled Maps =0
        Failed Shuffles=0
        Merged Map outputs=0
        GC time elapsed (ms)=15
        CPU time spent (ms)=0
        Physical memory (bytes) snapshot=0
        Virtual memory (bytes) snapshot=0
        Total committed heap usage (bytes)=382730240
    File Input Format Counters 
        Bytes Read=42
    File Output Format Counters 
        Bytes Written=64
```

执行结果：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-output.jpg)

注：最终输入的结果是排序好的，因为MapReduce中，在进行Reduce之前会根据key进行排序。