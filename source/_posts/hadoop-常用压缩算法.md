---
title: hadoop常用压缩算法
description: 目前在Hadoop中用得比较多的有lzo、gzip、snappy、bzip2这4种压缩格式， 根据实践经验介绍一下这4种压缩格式的优缺点和应用场景，以便大家在实践中根据实际情况选择不同的压缩格式。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-11-20 21:00:00
---

## 背景
目前在Hadoop中用得比较多的有lzo、gzip、snappy、bzip2这4种压缩格式， 根据实践经验介绍一下这4种压缩格式的优缺点和应用场景，以便大家在实践中根据实际情况选择不同的压缩格式。

## Gzip压缩
### 优点
压缩率比较高，而且压缩/解压速度也比较快；hadoop本身支持，在应用中处理gzip格式的文件就和直接处理文本一样；有hadoop native库；大部分linux系统都自带gzip命令，使用方便。

### 缺点
不支持split。

### 应用场景
当每个文件压缩之后在130M以内的（1个块大小内），都可以考虑用gzip压缩格式。譬如说一天或者一个小时的日志压缩成一个gzip文件，运行mapreduce程序的时候通过多个gzip文件达到并发。hive程序，streaming程序，和java写的mapreduce程序完全和文本处理一样，压缩之后原来的程序不需要做任何修改。

## Lzo压缩
### 优点
压缩/解压速度也比较快，合理的压缩率；支持split，是hadoop中最流行的压缩格式；支持hadoop native库；可以在linux系统下安装lzop命令，使用方便。

### 缺点
压缩率比gzip要低一些；hadoop本身不支持，需要安装；在应用中对lzo格式的文件需要做一些特殊处理（为了支持split需要建索引，还需要指定inputformat为lzo格式）。

### 应用场景
一个很大的文本文件，压缩之后还大于200M以上的可以考虑，而且单个文件越大，lzo优点越越明显。

## Snappy
### 优点
高速压缩速度和合理的压缩率；支持hadoop native库。

### 缺点
不支持split；压缩率比gzip要低；hadoop本身不支持，需要安装；linux系统下没有对应的命令。

### 应用场景
当mapreduce作业的map输出的数据比较大的时候，作为map到reduce的中间数据的压缩格式；或者作为一个mapreduce作业的输出和另外一个mapreduce作业的输入。

## Bzip2
###优点
支持split；具有很高的压缩率，比gzip压缩率都高；hadoop本身支持，但不支持native；在linux系统下自带bzip2命令，使用方便。

### 缺点
压缩/解压速度慢；不支持native。

### 应用场景
适合对速度要求不高，但需要较高的压缩率的时候，可以作为mapreduce作业的输出格式；或者输出之后的数据比较大，处理之后的数据需要压缩存档减少磁盘空间并且以后数据用得比较少的情况；或者对单个很大的文本文件想压缩减少存储空间，同时又需要支持split，而且兼容之前的应用程序（即应用程序不需要修改）的情况。

## 压缩格式比较图
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-ys-all.png)

## 总结
在hadoop中如果是单文件压缩后在200M左右建议使用gzip压缩，如果是单文件压缩之后大于400M建议使用<font color="red">可split</font>的lzo or bzip2压缩，这样在进行mapreduce的时候可以并行处理这个压缩文件。像在hbase中各列族可以使用snappy这种压缩算法，压缩解压都很快。
