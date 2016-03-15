---
title: hadoop-你该学习的常用命令
description: 从事hadoop工作中，一定要掌握的就是一些常用命令行。可能有段时间不用的话这些命令就被忘记，本文记录了一些常用的命令行，基本能满足开发。但是我建议对于命令行语法可以使用--help这样类似的语法来查看帮助，尽管有些帮助文档写的很烂！
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-11-22 20:00:00
---

## 背景
从事hadoop工作中，一定要掌握的就是一些常用命令行。可能有段时间不用的话这些命令就被忘记，本文记录了一些常用的命令行，基本能满足开发。但是我建议对于命令行语法可以使用--help这样类似的语法来查看帮助，尽管有些帮助文档写的很烂！

## Hadoop常用命令
查看指定目录下内容
```
hadoop dfs –ls [文件目录]
```

打开某个已存在文件
```
hadoop dfs –cat [file_path]
```

将本地文件存储至hadoop
```
hadoop fs –put [本地地址] [hadoop目录]
```

将本地文件夹存储至hadoop
```
hadoop fs –put [本地目录] [hadoop目录] 
```

将hadoop上某个文件down至本地已有目录下
```
hadoop fs -get [文件目录] [本地目录]
```

删除hadoop上指定文件
```
hadoop fs –rm [文件地址]
```

删除hadoop上指定文件夹（包含子目录等）
```
hadoop fs –rm [目录地址]
```

在hadoop指定目录内创建新目录
```
hadoop fs –mkdir /user/t
```

在hadoop指定目录下新建一个空文件，使用touchz命令：
```
hadoop fs -touchz /user/new.txt
```

将hadoop上某个文件重命名，使用mv命令：
```
hadoop fs –mv /user/test.txt /user/ok.txt （将test.txt重命名为ok.txt）
```

将hadoop指定目录下所有内容保存为一个文件，同时down至本地
```
hadoop dfs –getmerge /user /home/t
```

将正在运行的hadoop作业kill掉
```
hadoop job –kill [job-id]
```

将正在运行的Yarn作业kill掉，比如Spark
```
yarn application -kill [applicationId]
```

NameNode之间元数据同步命令
```
hdfs namenode -bootstrapStandby
```

查看HDFS基本统计信息
```
hadoop dfsadmin -report
```

进入和退出安全模式
```
hadoop dfsadmin -safemode enter
hadoop dfsadmin -safemode leave
```

检查hdfs文件目录
```
hadoop fsck / -files -blocks
```

修改hdfs副本个数，dfs.
replication参数不会修改已经存在的文件系统，可手动修改文件副本个数
```
hadoop fs -setrep -R 2 /
```

设置文件目录的大小限制
```
hadoop dfsadmin -setSpaceQuota 1t /user/username
```

## Linux常用命令

待整理