---
title: MapReduce中使用自定义Partitioner
description: <img alt="Alt text" src="http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-cllc.png"></br>在执行完Map任务后紧接的就是进行数据的平均分区，分区的数量是根据reduce线程的个数来确定的，以便在完成map操作之后，将后续的工作平均分配给每个reduce线程，这样做是为了避免有些reduce任务分配到大量数据，而有些reduce任务却分到很少数据，甚至没有分到数据的尴尬局面。每个分区内又调用job.setSortComparatorClass()设置的key比较函数类排序(如果没有通过job.setSortComparatorClass()设置key比较函数类，则使用key的实现的compareTo方法)，这是一个二次排序。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-12-17 22:24:00
---

## 背景

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-cllc.png)

在执行完Map任务后紧接的就是进行数据的平均分区，分区的数量是根据reduce线程的个数来确定的，以便在完成map操作之后，将后续的工作平均分配给每个reduce线程，这样做是为了避免有些reduce任务分配到大量数据，而有些reduce任务却分到很少数据，甚至没有分到数据的尴尬局面。每个分区内又调用job.setSortComparatorClass()设置的key比较函数类排序(如果没有通过job.setSortComparatorClass()设置key比较函数类，则使用key的实现的compareTo方法)，这是一个二次排序。

Partitioner组件可以让Map对Key进行分区，从而可以根据不同的key来分发到不同的reduce中去处理：
1、你可以自定义key的一个分发规则，如数据文件包含学校所有班级的学生成绩，而输出的要求是每个班级的学生成绩必须在同一个文件里；
2、提供了一个默认的HashPartitioner，根据key的hashcode来分区。

## 自定义Partitioner

自定义Partitioner只需两步即可完成：
1、继承抽象类Partitioner，实现自定义的getPartition（）方法；
2、通过job.setPartitionerClass（）来设置自定义的Partitioner；

下面基于，如数据文件包含学校所有班级的学生成绩，而输出的要求是每个班级的学生成绩必须在同一个文件里，根据这个需求来编写自定义的Partitioner：
```
/**
 * 
 * @@Description: 自定义分区
 * @@version 1.0 2015年12月17日 下午4:39:38 by 张仁华 创建
 */
public class MyPartitioner extends Partitioner<Text, IntWritable> {

    @Override
    public int getPartition(Text key, IntWritable value, int numPartitions) {
        // 假设key = 班级编号_学生编号 
        // value = 成绩

        /**
         * 根据假设，截取班级编号，然后将：班级编号hashcode % reduce个数
         */
        String strKey = key.toString();
        int index = strKey.indexOf("_");
        return Math.abs(strKey.substring(0, index).hashCode()) % numPartitions;
    }

}
```

根据假设，截取班级编号，然后将班级编号hashcode % reduce个数，这样能保证同一个班级的学生都会集中在一个Reduce里面进行处理（最终输出也能在同一文件中）。

自定义MyPartitioner后只需在原有写好的MapReduce上加入以下代码即可：
```
// Job job = Job.getInstance(conf);
job.setPartitionerClass(MyPartitioner.class);
```

如何编写MapReduce请参考：[《MapReduce开发-WordCount》](/2015/12/08/hadoop-MapReduce-WordCount/)