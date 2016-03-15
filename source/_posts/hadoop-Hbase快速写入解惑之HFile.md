---
title: Hbase快速写入解惑之HFile
description: 在网上经常看到关于Hbase高速写入的一些文章，其中大部分就是利用Hbase提供的工具类生成HFile。并且一些文章说HFile只适合初始导入（或者表中数据为空），还有一些文章中说不管初始化还是增量添加数据都是用HFile，其实上面两种说法都不完全正确。下面我会写出Mapreduce和非MapReduce方式生成HFle并导入Hbase表中的实现。并且解释在什么时候使用这种方式最合适。
tags: [hadoop生态圈, hbase]
categories: hbase
date: 2016-1-28 19:44:47
---

## 背景

在网上经常看到关于Hbase高速写入的一些文章，其中大部分就是利用Hbase提供的工具类生成HFile。并且一些文章说HFile只适合初始导入（或者表中数据为空），还有一些文章中说不管初始化还是增量添加数据都是用HFile，其实上面两种说法都不完全正确。下面我会写出Mapreduce和非MapReduce方式生成HFle并导入Hbase表中的实现。并且解释在什么时候使用这种方式最合适。


## MapReduce生成HFile

先建好Hbase表：
```
create 'hfiletable','fm1','fm2'
```


测试数据，并上传至/tmp目录：
```
key1,fm1:col1,value1
key1,fm1:col2,value2
key1,fm2:col1,value3
key4,fm1:col1,value4
```


示例代码：
```
import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.FsShell;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.client.HTable;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
import org.apache.hadoop.hbase.mapreduce.HFileOutputFormat2;
import org.apache.hadoop.hbase.mapreduce.LoadIncrementalHFiles;
import org.apache.hadoop.hbase.util.Bytes;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class BulkLoadJob {
    static Logger logger = LoggerFactory.getLogger(BulkLoadJob.class);

    public static class BulkLoadMap
            extends Mapper<LongWritable, Text, ImmutableBytesWritable, Put> {

        public void map(LongWritable key, Text value, Context context)
                throws IOException, InterruptedException {

            String[] valueStrSplit = value.toString().split(",");
            String hkey = valueStrSplit[0];
            String family = valueStrSplit[1].split(":")[0];
            String column = valueStrSplit[1].split(":")[1];
            String hvalue = valueStrSplit[2];

            /**
             * 只支持单列族并发写入，如果是多个列族得多write一次
             */
            final byte[] rowKey = Bytes.toBytes(hkey);
            final ImmutableBytesWritable HKey = new ImmutableBytesWritable(
                    rowKey);
            Put put = new Put(rowKey);
            byte[] cell = Bytes.toBytes(hvalue);
            put.add(Bytes.toBytes(family), Bytes.toBytes(column), cell);
            context.write(HKey, put);

        }
    }

    public static void main(String[] args) throws Exception {

        /**
         * 此方法运行不宜小文件使用，每执行一次就会hfile就会增加，可以采用major_compact 'tablename' 进行合并
         */
        Configuration conf = HBaseConfiguration.create();
        String inputPath = "/tmp/data.txt";
        String outputPath = "/tmp/hfiledemo";
        String tableName = "hfiletable";

        /**
         * 删除输出目录
         */
        FileSystem fileSystem = FileSystem.get(conf);
        Path outPath = new Path(outputPath);
        if (fileSystem.exists(outPath)) {
            fileSystem.delete(outPath, true);
        }

        Job job = Job.getInstance(conf, "HFile bulk load");
        job.setJarByClass(BulkLoadJob.class);
        job.setMapperClass(BulkLoadJob.BulkLoadMap.class);
        job.setMapOutputKeyClass(ImmutableBytesWritable.class);
        job.setMapOutputValueClass(Put.class);
        // Disable speculative execution
        job.setSpeculativeExecution(false);
        job.setReduceSpeculativeExecution(false);

        // in/out format
        job.setInputFormatClass(TextInputFormat.class);
        job.setOutputFormatClass(HFileOutputFormat2.class);

        FileInputFormat.setInputPaths(job, inputPath);
        FileOutputFormat.setOutputPath(job, outPath);

        HTable hTable = new HTable(conf, tableName);
        HFileOutputFormat2.configureIncrementalLoad(job, hTable, hTable);

        if (job.waitForCompletion(true)) {

            /**
             * 解决权限问题
             */
            FsShell shell = new FsShell(conf);
            try {
                shell.run(new String[] { "-chmod", "-R", "777", outputPath });
            } catch (Exception e) {
                logger.error("Couldnt change the file permissions ", e);
                throw new IOException(e);
            }

            /**
             * 加载到hbase表
             */
            LoadIncrementalHFiles loader = new LoadIncrementalHFiles(conf);
            loader.doBulkLoad(outPath, hTable);
        }
    }
}
```

## 非MapReduce生成HFile

示例代码：
```
import java.text.DecimalFormat;
import java.util.UUID;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.HConstants;
import org.apache.hadoop.hbase.KeyValue;
import org.apache.hadoop.hbase.client.HTable;
import org.apache.hadoop.hbase.io.compress.Compression;
import org.apache.hadoop.hbase.io.hfile.CacheConfig;
import org.apache.hadoop.hbase.io.hfile.HFile;
import org.apache.hadoop.hbase.io.hfile.HFile.Writer;
import org.apache.hadoop.hbase.io.hfile.HFileContext;
import org.apache.hadoop.hbase.mapreduce.LoadIncrementalHFiles;
import org.apache.hadoop.hbase.util.Bytes;


public class HFileGenertor {

    // 建表语句，数据文件参考data.txt
    // create 'hfiletable','fm1','fm2'
    public static void main(String[] args) throws Exception {
        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum", "hynamet01,hynamet02");
        conf.set("hbase.zookeeper.property.clientPort", "2181");
        // 默认是/hbase,根据配置文件指定
        conf.set("zookeeper.znode.parent", "/hbase");

        String tableName = "hfiletable";
        byte[] family = Bytes.toBytes("fm1");
        String outputdir = "/tmp/zhangrenhua/hbase/";

        /**
         * hfile 随机文件名
         */
        Path dir = new Path(outputdir);
        Path familydir = new Path(outputdir + Bytes.toString(family),
                UUID.randomUUID().toString());

        /**
         * 创建HFile操作对象
         */
        Configuration tempConf = new Configuration(conf);
        // tempConf.set("hbase.metrics.showTableName", "false");
        tempConf.setFloat(HConstants.HFILE_BLOCK_CACHE_SIZE_KEY, 1.0f);

        // 文件属性对象，比如压缩算法之类的
        HFileContext fileContext = new HFileContext();
        fileContext.setCompression(Compression.Algorithm.NONE);

        // HFile文件操作对象
        Writer writer = HFile.getWriterFactory(conf, new CacheConfig(tempConf))
                .withPath(FileSystem.get(conf), familydir)
                .withFileContext(fileContext).create();

        DecimalFormat df = new DecimalFormat("0000000");

        KeyValue kv1 = null;
        KeyValue kv2 = null;
        KeyValue kv3 = null;
        KeyValue kv4 = null;
        KeyValue kv5 = null;
        KeyValue kv6 = null;
        KeyValue kv7 = null;
        KeyValue kv8 = null;

        byte[] cn = Bytes.toBytes("cn");
        byte[] dt = Bytes.toBytes("dt");
        byte[] ic = Bytes.toBytes("ic");
        byte[] ifs = Bytes.toBytes("if");
        byte[] ip = Bytes.toBytes("ip");
        byte[] le = Bytes.toBytes("le");
        byte[] mn = Bytes.toBytes("mn");
        byte[] pi = Bytes.toBytes("pi");

        int maxLength = 100;
        for (int i = 0; i < maxLength; i++) {
            String currentTime = System.currentTimeMillis() + df.format(i);
            long current = System.currentTimeMillis();
            // rowkey和列都要按照字典序的方式顺序写入，否则会报错的
            kv1 = new KeyValue(Bytes.toBytes(currentTime), family, cn, current,
                    KeyValue.Type.Put, Bytes.toBytes("3"));

            kv2 = new KeyValue(Bytes.toBytes(currentTime), family, dt, current,
                    KeyValue.Type.Put, Bytes.toBytes("6"));

            kv3 = new KeyValue(Bytes.toBytes(currentTime), family, ic, current,
                    KeyValue.Type.Put, Bytes.toBytes("8"));

            kv4 = new KeyValue(Bytes.toBytes(currentTime), family, ifs, current,
                    KeyValue.Type.Put, Bytes.toBytes("7"));

            kv5 = new KeyValue(Bytes.toBytes(currentTime), family, ip, current,
                    KeyValue.Type.Put, Bytes.toBytes("4"));

            kv6 = new KeyValue(Bytes.toBytes(currentTime), family, le, current,
                    KeyValue.Type.Put, Bytes.toBytes("2"));

            kv7 = new KeyValue(Bytes.toBytes(currentTime), family, mn, current,
                    KeyValue.Type.Put, Bytes.toBytes("5"));

            kv8 = new KeyValue(Bytes.toBytes(currentTime), family, pi, current,
                    KeyValue.Type.Put, Bytes.toBytes("1"));

            writer.append(kv1);

            writer.append(kv2);
            writer.append(kv3);
            writer.append(kv4);
            writer.append(kv5);
            writer.append(kv6);
            writer.append(kv7);
            writer.append(kv8);
        }

        writer.close();

        /**
         * 把生成的HFile导入到hbase当中
         */
        HTable table = new HTable(conf, tableName);
        LoadIncrementalHFiles loader = new LoadIncrementalHFiles(conf);
        loader.doBulkLoad(dir, table);

    }
}
```


## 总结

上面代码利用hbase的数据信息按照特定格式存储在hdfs内这一原理，直接生成这种hdfs内存储的数据格式文件，然后上传至合适位置，即完成巨量数据快速入库的办法。配合mapreduce完成，高效便捷，而且不占用region资源，增添负载。

### 注意事项

1、生成的hfile文件`RowKey`必须是有序的。
2、在最后一步将HFile加载到hbase中时，会检查rowkey，并会根据rowkey来进行split（如果数据集中的rowkey分散在多个region的多个hfile中，那么split的动静就会很大），所有一般情况下只适合用来做表初始化。
3、每执行一次都会生成一到多个HFile，如果数据量小，小文件太多，如不调用合并方法，则会影响Hbase查询性能。

那么在哪些情况下使用此方法写入Hbase最为合适？
1、数据初始化导入表中，使用此方法效率最好。
2、如果表的rowkey是以时间戳或自然顺序持续增长的大量数据则使用此方法`批量`写入数据最为合适。
3、如果数据量很大也可以使用此方法导入，但是会造成大量的HFile split。

