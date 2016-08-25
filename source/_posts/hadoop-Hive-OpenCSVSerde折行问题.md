---
title: Hive-OpenCSVSerde折行问题
description: 最近遇到一件比较有趣的事，</br>某个大牛部门，将数据库中的某张表导出成txt格式。恩，这看起来很正常。</br>为什么说有趣呢？</br>因为表的字段内容中有换行符（`\n`），行的分割符也是`\n`，那么我程序读取的时候怎么区分哪个`\n`才是真正的行分割符，而不是字段值呢？是否该换种导出格式（csv or serialize...）？</br></br>然而如果大牛部门态度非常强硬的说：“只能我们去适应他们的格式”</br>遇到这种情况，请问我们敲代码的同志们该怎么去处理？是否会先吐槽一下下？下面我就列举几种吐槽方式：</br>1、何蛋蛋（程序猿，专注调戏测试妹子）：**，这尼玛非得弄死他们部门老大。</br>2、菜地地（攻城狮，除了吃就是游戏）：老大，搞不定咋办呀？</br>3、龚开开（自由职业者，专注移动端项目开发）：太没节操了，领导我下周不干了。</br>哈哈，中途逗一下。。。</br>当然我不是说解析很难，只是随意吐槽一下，为什么要这么死板？为什么在设计的时候不能多考虑一点点。哦，好吧！（具体怎么解析我就不说了，有兴趣的同学可以一起讨论，互相学习）</br>其实我得感谢这些牛牛们，让我学习到了新的知识，在此谢过鸟。。。</br>言归正传，如标题“OpenCSVSerde折行问题”，折行是什么意思呢？请看下一章节
tags: [hadoop生态圈, hive]
categories: hive
date: 2016-8-14 00:10:51
---

## 背景

最近遇到一件比较有趣的事，
某个大牛部门，将数据库中的某张表导出成txt格式。恩，这看起来很正常。

为什么说有趣呢？
因为表的字段内容中有换行符（`\n`），行的分割符也是`\n`，那么我程序读取的时候怎么区分哪个`\n`才是真正的行分割符，而不是字段值呢？是否该换种导出格式（csv or serialize...）？

然而如果大牛部门态度非常强硬的说：“只能我们去适应他们的格式”
遇到这种情况，请问我们敲代码的同志们该怎么去处理？是否会先吐槽一下下？下面我就列举几种吐槽方式：
1、何蛋蛋（程序猿，专注调戏测试妹子）：**，这尼玛非得弄死他们部门老大。
2、菜地地（攻城狮，除了吃就是游戏）：老大，搞不定咋办呀？
3、龚开开（自由职业者，专注移动端项目开发）：太没节操了，领导我下周不干了。

哈哈，中途逗一下。。。

当然我不是说解析很难，只是随意吐槽一下，为什么要这么死板？为什么在设计的时候不能多考虑一点点。哦，好吧！（具体怎么解析我就不说了，有兴趣的同学可以一起讨论，互相学习）

其实我得感谢这些牛牛们，让我学习到了新的知识，在此谢过鸟。。。

言归正传，如标题“OpenCSVSerde折行问题”，折行是什么意思呢？请看下一章节

## OpenCSVSerde折行问题

**什么是折行？**
折行是指，数据内容中包含换行符，
假设有表test一张，三个字段（a,b,c）
CSV文件内容：`"张三\r三",李四,王五`，其中`\n`代表换行符，`,`字段分隔符。

a字段值为：张三\r三
b字段值为：李四
c字段值为：王五


**OpenCSVSerde为什么处理不了折行问题**

下面先通过实例来说明，然后再看源码分析

创建测试表：
```
CREATE TABLE
    csv1_table
    (
        a string,
        b string,
        c string
    )
    ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'WITH SERDEPROPERTIES
    (
        "separatorChar" = ",",
        "quoteChar" = '"',
        "escapeChar" = "\\"
    ) STORED AS TEXTFILE;

```

![表结构](http://7xoqbc.com1.z0.glb.clouddn.com/hive-opencsvserde-desc.png)


创建测试文件(test.csv)：
![测试数据](http://7xoqbc.com1.z0.glb.clouddn.com/hive-opencsvserde-testdata.png)

上传测试文件：
```
hadoop fs -put test.csv /user/hive/warehouse/csv1_table
```

查询测试：
![查询测试](http://7xoqbc.com1.z0.glb.clouddn.com/hive-opencsvserde-testquery.png)

为什么会查询出两行数据？这就是折行数据导致的，请看下面源码分析。


**OpenCSVSerde 源码分析，为什么不能读取折行*

默认在标准的csv规范中，上面的测试数据是和规则的，为什么在OpenCSVSerde中读取为NULL呢？

前面有一张表结构图，通过命名查看：`desc formatted csv1_table;`，
里面是用`TextInputFormat`来读取数据文件的。

问题就出在了`TextInputFormat`类，该类是只要读取到换行符的话就会把它当前一行数据返回，
根据测试数据中，只要读取到`"张三\r`，这里发现换行符`\n`，则会立刻返回，这样就把数据读取成两行了。

查看OpenCSVSerde源码的deserialize方法：
![OpenCSVSerde源码](http://7xoqbc.com1.z0.glb.clouddn.com/hive-opencsvserde-source.png)

改方法将一行数据，转换成CSVReader对象解析，这样`"张三\r`数据的格式就是错误的，所以返回NULL。




