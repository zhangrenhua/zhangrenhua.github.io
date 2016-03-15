---
title: hadoop2.x-Eclipse开发环境搭建
description: 对于初学者来说，搭建一个开发环境是非常有必要的。由于对Hadoop API的熟练度不够，很难做到不需要调试。当然对于作者我来说早就不需要debug来调试代码了。下面我将《eclipse开发环境搭建》献给那些刚入门的hadoop开发者和那些已经忘记怎么搭建的老手同志们。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-12-05 18:20:00
---

## 背景

对于初学者来说，搭建一个开发环境是非常有必要的。由于对Hadoop API的熟练度不够，很难做到不需要调试。当然对于作者我来说早就不需要debug来调试代码了。下面我将《Windows下eclipse开发环境搭建》献给那些刚入门的hadoop开发者和那些已经忘记怎么搭建的老手同志们。

## Hadoop本地部署
下面只是简单的描述部署hadoop本地运行环境，而不是Windows下部署hadoop集群环境。然而eclipse本地调试只需要这些。

### 下载hadoop安装包
http://hadoop.apache.org/#Download+Hadoop
我下载的是hadoop-2.2.0.tar.gz

### 配置hadoop环境变量
1、解压hadoop-2.2.0.tar.gz
2、配置环境变量：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-eclipse-hjbl.png)

HADOOP_HOME=G:\javaTools\hadoop-2.2.0
PATH=G:\javaTools\hadoop-2.2.0\bin
注：<font color="red">这两个环境变量一定要配置，尽量保证路径中不要有中文和空格。</font>

### 下载windows开发工具并替换
1、下载包含windows端开发Hadoop2.2需要的winutils.exe
[hadoop-common-2.2.0-bin-master.zip](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-common-2.2.0-bin-master.zip)

2、将hadoop-common-2.2.0-bin-master.zip压缩包中所有文件覆盖至
$HADOOP_HOME\bin目录
注：<font color="red">$HADOOP_HOME是你自己hadoop安装的目录。</font>

### eclipse插件安装
1、下载eclipse插件包
[hadoop-eclipse-plugin-2.2.0.jar](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-eclipse-plugin-2.2.0.jar)

2、将hadoop-eclipse-plugin-2.2.0.jar放到eclipse安装目录下的plugins下，并重启eclipse
比如：G:\StudyTools\java\IDE\eclipse-jee-mars-R-win32-x86_64\eclipse\plugins

### eclipse环境配置

1、打开Map/Reduce窗口Window -> Show View -> Map/Reduce Locations：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-eclipse-newHadoopLocation.png)
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-eclipse-mrconfig.png)

根据上图只要配置下面几个属性即可：
Host：hadoop集群中NameNode机器IP
<font color="red">如果做了HA则选择状态为Active NameNode的机器</font>
Port：NameNode  RPC交互端口，默认8020
以上2个port都填8020也没关系，因为跑MapReduce用我们集群环境的配置文件即可
User Name：访问用户，随意填写

2、关闭集群文件权限检查
如果想操作HDFS上的文件还得把Hadoop集群环境下的文件权限检查关闭：
修改hdfs-site.xml配置文件中dfs.permissions.enabled = false
修改之后必须重启集群。

3、根据上面配置好后，可以直接使用eclipse来远程操作HDFS文件系统
打开Project Explorer窗口Window -> Show View -> Project Explorer：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-eclipse-hdfs.png)

### 执行MapReduce
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-eclipse-mapreduce.png)
1、必须将集群环境中的几个配置文件放到项目的类路径下：
core-site.xml
hdfs-site.xml
建议将log4j.properties文件也加入，可以在hadoop安装包中随意找一个。
注：<font color="red">上面2个xml一定要是hadoop环境中的真实配置文件</font>

2、只要将上面几个配置文件放入到项目中，直接运行自己写的Main函数即可。

### 错误解决
1、缺少文件：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-eclipse-errwinutils.png)
如果出现这个错误肯定是《下载windows开发工具并替换》这一步骤没做。

2、文件IO异常
```
Exception in thread "main" java.lang.UnsatisfiedLinkError: org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Ljava/lang/String;I)Z
```

原因：缺少hadoop.dll文件
解决方案：
将hadoop-common-2.2.0-bin-master.zip包下的hadoop.dll文件放在C:\Windows\System32目录下。

3、权限错误
```
15/10/28 16:05:53 INFO mapred.JobClient: Running job: job_201510281103_0003
15/10/28 16:05:54 INFO mapred.JobClient: map 0% reduce 0%
15/10/28 16:06:05 INFO mapred.JobClient: Task Id : attempt_201510281103_0003_m_000002_0, Status : FAILED
org.apache.hadoop.security.AccessControlException: org.apache.hadoop.security.AccessControlException: Permission denied: user=DrWho, access=WRITE, inode="hadoop":hadoop:supergroup:rwxr-xr-x 
at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:39)
```

解决方案：关闭集群文件权限检查
修改hadoop集群环境hdfs-site.xml配置文件中
dfs.permissions.enabled = false
修改之后必须重启集群。