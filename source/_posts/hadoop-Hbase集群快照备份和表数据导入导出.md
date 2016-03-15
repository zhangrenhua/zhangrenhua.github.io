---
title: Hbase集群快照备份和表数据导入导出的三种方式
description: HBase Snapshots允许你对一个表进行快照（即可用副本），它不会对Region Servers产生很大的影响，它进行复制和恢复操作的时候不包括数据拷贝。导出快照到另外的集群也不会对Region Servers产生影响。下面告诉你如何使用Snapshots功能以及表数据导入导出的三种方式。
tags: [hadoop生态圈, hbase]
categories: hbase
date: 2016-1-26 19:33:25
---

## 背景

HBase Snapshots允许你对一个表进行快照（即可用副本），它不会对Region Servers产生很大的影响，它进行复制和恢复操作的时候不包括数据拷贝。导出快照到另外的集群也不会对Region Servers产生影响。下面告诉你如何使用Snapshots功能以及表数据导入导出的三种方式。


## Snapshots快照

1、开启快照支持功能，在0.95+之后的版本都是默认开启的，在0.94.6+是默认关闭
```
<property>
    <name>hbase.snapshot.enabled</name>
    <value>true</value>
</property>
```

2、给表建立快照，不管表是启用或者禁用状态，这个操作不会进行数据拷贝
```
$ ./bin/hbase shell 
hbase> snapshot 'myTable', 'myTableSnapshot-122112'
```

3、列出已经存在的快照
```
$ ./bin/hbase shell 
hbase> list_snapshots
```

4、删除快照
```
$ ./bin/hbase shell 
hbase> delete_snapshot 'myTableSnapshot-122112'
```

5、从快照复制生成一个新表
```
$ ./bin/hbase shell 
hbase> clone_snapshot 'myTableSnapshot-122112', 'myNewTestTable'
```

6、用快照恢复数据，它需要先禁用表，再进行恢复
```
$ ./bin/hbase shell
hbase> disable 'myTable' 
hbase> restore_snapshot 'myTableSnapshot-122112'
```

提示：因为备份（replication）是系统日志级别的，而快照是文件系统级别的，当使用快照恢复之后，副本会和master处于不同的状态，如果你需要使用恢复的话，你要停止备份，并且重置bootstrap。

如果是因为不正确的客户端行为导致数据丢失，全表恢复又需要表被禁用，可以采用快照生成一个新表，然后从新表中把需要的数据用map-reduce拷贝到主表当中。

7、复制到别的集群当中
该操作要用hbase的账户执行，并且在hdfs当中要有hbase的账户建立的临时目录（hbase.tmp.dir参数控制）

采用16个mappers来把一个名为MySnapshot的快照复制到一个名为srv2的集群当中
```
$ bin/hbase class org.apache.hadoop.hbase.snapshot.ExportSnapshot -snapshot MySnapshot -copy-to hdfs://srv2:8020/hbase -mappers 16
```

**限制带宽消耗**
导出的快照，通过指定-bandwidth参数，它需要代表每秒兆字节的整数时，可以限制带宽消耗。下面的例子在上述实施例限制为200 MB /秒。
```
$ bin/hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot -snapshot MySnapshot -copy-to hdfs://srv2:8082/hbase -mappers 16 -bandwidth 200
```


## 表数据导入导出

### 表数据导出

导出是一个实用工具，将转储表的内容，HDFS的序列文件。通过调用：
```
$ bin/hbase org.apache.hadoop.hbase.mapreduce.Export <tablename> <outputdir> [<versions> [<starttime> [<endtime>]]]
```

**参数说明：**
`tablename（必选）`：表名
`outputdir（必选）`：导出的hdfs的路径
`versions（可选）`：导出版本数
`starttime（可选）`：导出数据的起始时间（注意这里的起始时间是指数据的timestamp，比如我传入的的昨天的日期，则代表导出昨天以后数据）
`endtime（可选）`：导出数据的结束时间（同样是指数据的timestamp）

默认情况下，导出工具仅导出而不必考虑存储版本数给定cell的最新版本。要导出多个版本，替换<versions> 与版本的所需数量。

注：缓存输入扫描通过hbase.client.scanner.caching在任务配置中配置。


### 表数据导入

导入是一种实用工具，将加载已导出回HBase的数据。通过调用：
```
$ bin/hbase org.apache.hadoop.hbase.mapreduce.Import <tablename> <inputdir>
```

要导入在0.96集群或起0.94导出的文件，运行导入命令时，如下需要设置系统属性“base.import.version”：
```
$ bin/hbase -Dhbase.import.version=0.94 org.apache.hadoop.hbase.mapreduce.Import <tablename> <inputdir>
```

**参数说明:**
`tablename`：表名
`inputdir`：数据存储的路径

<font color="red">如果表中存在数据，会以追加的方式导入。</font>

## ImportTsv导入

ImportTsv是Hbase提供的一个命令行工具，可以将存储在HDFS上的自定义分隔符（默认\t）的数据文件，通过一条命令方便的导入到HBase表中，对于大数据量导入非常实用，其中包含两种方式将数据导入到HBase表中：
第一种是使用TableOutputformat在reduce中插入数据；
第二种是先生成HFile格式的文件，再执行一个叫做CompleteBulkLoad的命令，将文件move到HBase表空间目录下，同时提供给client查询。

例如，假设我们将数据装载到一个名为“test”用的ColumnFamily名为“D”两列“C1”和“C2”的表。
```
$ bin/hbase
$ create 'test','d'
```

假定一个输入文件内容为：
```
row1,c1,c2
row2,c1,c2
row3,c1,c2
row4,c1,c2
row5,c1,c2
row6,c1,c2
row7,c1,c2
row8,c1,c2
row9,c1,c2
row10,c1,c2
```

将文件上传至hdfs的/tmp目录，并命名为data.txt

对于ImportTsv使用该输入文件，在命令行必须是这样的：
```
 hbase org.apache.hadoop.hbase.mapreduce.ImportTsv  -Dimporttsv.separator=',' -Dimporttsv.columns=HBASE_ROW_KEY,d:c1,d:c2 test /tmp/data.txt
```


**其他包含 -D的可指定的选项包括：**
`-Dimporttsv.skip.bad.lines=false` – 若遇到无效行则失败
`-Dimporttsv.separator=|` – 文件中代替tabs的分隔符
`-Dimporttsv.timestamp=currentTimeAsLong` – 导入时使用指定的时间戳
`-Dimporttsv.mapper.class=my.Mapper` – 使用用户指定的Mapper类来代替默认的org.apache.hadoop.hbase.mapreduce.TsvImporterMapper


并且在这个例子中，第一列是rowkey，这就是为什么HBASE_ROW_KEY被使用。第二、三列为：“d:c1” and “d:c2”，如果你已经准备了大量的数据进行批量加载，确保目标HBase的表是预分区的。

<font color="red">如果表中存在数据，会以追加的方式导入。</font>

### 利用bulkload 将数据导入到

利用importTSV生成HFile：
```
hbase org.apache.hadoop.hbase.mapreduce.ImportTsv -Dimporttsv.separator="," -Dimporttsv.bulk.output=/tmp/zhangrenhua/hbase -Dimporttsv.columns=HBASE_ROW_KEY,d:c1,d:c2 test /tmp/data.txt
```

将HFile导入到Hbase：
```
hbase org.apache.hadoop.hbase.mapreduce.LoadIncrementalHFiles /tmp/zhangrenhua/hbase test
```

## 总结

在使用ImportTsv时，一定要注意参数importtsv.bulk.output的配置，通常来说使用Bulk output的方式对Regionserver来说更加友好一些，这种方式加载数据几乎不占用Regionserver的计算资源，因为只是在HDFS上移动了HFile文件，然后通知HMaster将该Regionserver的一个或多个region上线。

## 参考资料
- http://hbase.apache.org/book.html#tools