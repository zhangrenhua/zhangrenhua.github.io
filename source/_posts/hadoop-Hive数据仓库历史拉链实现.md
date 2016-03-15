---
title: Hive数据仓库历史拉链实现
description: 发现最近写博客的频率越来越低了，是太忙了？或者是不想写？呵呵！革命尚未成功，同志还需努力。今天利用下午一点空余时间整理如何基于hive上实现数据仓库中的历史拉链算法。
tags: [hadoop生态圈, hive]
categories: hive
date: 2016-2-28 14:55:31
---

## 背景

发现最近写博客的频率越来越低了，是太忙了？或者是不想写？呵呵！革命尚未成功，同志还需努力。今天利用下午一点空余时间整理如何基于hive上实现数据仓库中的历史拉链算法。

笔者使用hive1.1进行测试。

## 历史存储方式

下面先来了解一下历史数据存储的几种方法。

数据历史的存储方式
1、切片存储：通过数据日期记录数据的入库时点
2、拉链存储：通过开始日期与结束日期记录每一个主键记录的变化过程

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hive-ods.png)
两种存储方式没有好坏之分，都可以完整的保留数据历史，只是各自适合于不同的数据情况。

### 拉链存储方式--新增：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hive-ods-add.png)

### 拉链存储方式--删除：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hive-ods-del.png)

### 拉链存储方式--修改：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hive-ods-update.png)

## 拉链存储的优点
1、节省存储空间
2、记录数据变化
3、便于获取数据

## 为什么我们要大规模使用拉链表
1、数据平台的建设原则是保留数据历史
2、遵循实际的数据规律，搭建标准、灵活、可用行强的数据平台

## 存储方式的选择
1、记录对象：除特殊情况，一般使用拉链存储
2、记录时间：一般默认使用切片存储，实际选择时，根据实际数据情况（是否存在属性变化，是否存在数据物理删除）进行选择

拉链存储的事件表：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hive-ods-sjb.png)


## 拉链算法实现

### 创建测试表
```
-- 技术缓冲层
DROP TABLE
    t_account;
CREATE TABLE
    t_account
    (
        id VARCHAR(30),
        name VARCHAR(60),
        balance DECIMAL(14,4)
    )
    row format delimited fields terminated BY '\177';

-- 近源模型层 临时表
DROP TABLE
    n_account;
CREATE TABLE
    n_account
    (
        id VARCHAR(30),
        name VARCHAR(60),
        balance DECIMAL(14,4),
        START_DT VARCHAR(8),
        END_DT VARCHAR(8)
    )
    STORED AS parquet TBLPROPERTIES
    (
        'COMPRESSION_CODEC'='snappy'
    );
-- 近源模型层 数据表
DROP TABLE
    o_account;
CREATE TABLE
    o_account
    (
        id VARCHAR(30),
        name VARCHAR(60),
        balance DECIMAL(14,4),
        START_DT VARCHAR(8)
    )
    PARTITIONED BY
    (
        END_DT VARCHAR(8)
    )
    STORED AS parquet TBLPROPERTIES
    (
        'COMPRESSION_CODEC'='snappy'
    );
```

本次使用账户表进行测试，技术缓冲层`t_account`使用默认的txt格式存储，有便于将格式符合的原始文件能直接加载到hive中。近源模型层使用`parquet`格式存储，`parquet`格式号称为计算而生。

### 拉链步骤

从技术缓冲层向近源模型层加载状态类无删除拉链算法(LEFT JOIN方式)
步骤1，清空临时表：
```
TRUNCATE TABLE
    default.n_account;
```

步骤2，取出增量数据(新增|修改)到临时表：
```
INSERT
INTO
    TABLE default.n_account
SELECT DISTINCT
    T.id,
    T.name,
    T.balance,
    '${data_date}' AS START_DT,
    '30001231'     AS END_DT
FROM
    default.t_account T
LEFT JOIN
    default.o_account O
ON
    O.id=COALESCE(RTRIM(T.id),'')
AND O.START_DT<'${data_date}'
AND O.END_DT>='${data_date}'
WHERE
    (
        O.id IS NULL)
OR  O.id<>COALESCE(RTRIM(T.id),'')
OR  O.name<>COALESCE(RTRIM(T.name),'')
OR  O.balance<>COALESCE(T.balance,'') ;
```

步骤3，取当前有效数据到临时表,并对其中部分数据进行闭链：
```
INSERT
INTO
    TABLE default.n_account
SELECT DISTINCT
    O.id ,
    O.name ,
    O.balance ,
    O.START_DT ,
    CASE
        WHEN N.id IS NOT NULL
        THEN N.START_DT
        ELSE '30001231'
    END AS END_DT
FROM
    default.o_account O
LEFT JOIN
    default.n_account N
ON
    O.id=N.id
WHERE
    O.START_DT<'${data_date}'
AND O.END_DT>='${data_date}';
```

步骤4，取当前有效数据到临时表,并对其中部分数据进行闭链：
```
ALTER TABLE
    default.o_account DROP PARTITION(END_DT>='${data_date}');
```


步骤5，将临时表数据全部插入目标表：
```
// 开启动态分区
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;
INSERT
    OVERWRITE TABLE default.o_account PARTITION
    (
        END_DT
    )
SELECT
    id,
    name,
    balance,
    START_DT,
    END_DT
FROM
    default.n_account;
```


### 拉链测试

将数据加载到技术缓冲层（次步骤可以由pig、mapreduce、spark、sqoop等工具来替代）：
```
INSERT
INTO
    TABLE default.t_account VALUES
    (
        '001',
        'zhangsan',
        2000
    );
```


根据拉链步骤一一执行，并将sql中的`${data_date}`变量替换成测试日期如（20140101）

模拟下一次数据：清空技术缓冲层表数据（如需备份可以采用分区的形式来保存）,并插入测试数据：
```
TRUNCATE TABLE
    default.t_account ;
INSERT
INTO
    TABLE default.t_account VALUES
    (
        '001',
        'zhangsan',
        4000
    );
```

根据拉链步骤一一执行，并将sql中的`${data_date}`变量替换成测试日期如（20140102）

查看测试结果：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hive-ods-test-sel.png)

此时可以看到，zhangsan余额为2000时的数据已经闭链。

## APPEND算法

直接将当日增量数据覆盖到目标表：
```
// 开启动态分区
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;

INSERT OVERWRITE TABLE default.o_account PARTITION(ETL_DT) 
SELECT
    id,
    name,
    balance,
    END_DT
FROM
    default.n_account T
WHERE T.ETL_DT='${data_date}';
```


至此拉链算法实现测试完成。