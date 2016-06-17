---
title: Solr6.0.1概念和集群部署
description: <h3 id="solrcloud"><a name="user-content-solrcloud" href="#solrcloud" class="headeranchor-link" aria-hidden="true"><span class="headeranchor"></span></a>SolrCloud的特色功能</h3><p><strong>集中式的配置</strong><br />信息使用ZK进行集中配置。启动时可以指定把Solr的相关配置文件上传Zookeeper，多机器共用。这些ZK中的配置不会再拿到本地缓存，Solr直接读取ZK中的配置信息。配置文件的变动，所有机器都可以感知到。另外，Solr的一些任务也是通过ZK作为媒介发布的。目的是为了容错。接收到任务，但在执行任务时崩溃的机器，在重启后，或者集群选出候选者时，可以再次执行这个未完成的任务。</p><p><strong>自动容错</strong><br />SolrCloud对索引分片，并对每个分片创建多个Replication。每个Replication都可以对外提供服务。一个Replication挂掉不会影响索引服务。更强大的是，它还能自动的在其它机器上帮你把失败机器上的索引Replication重建并投入使用。</p><p><strong>近实时搜索</strong><br />立即推送式的replication（也支持慢推送）。可以在秒内检索到新加入索引。</p><p><strong>查询时自动负载均衡</strong><br />SolrCloud索引的多个Replication可以分布在多台机器上，均衡查询压力。如果查询压力大，可以通过扩展机器，增加Replication来减缓。<br />自动分发的索引和索引分片发送文档到任何节点，它都会转发到正确节点。</p><p><strong>事务日志</strong><br />事务日志确保更新无丢失，即使文档没有索引到磁盘。</p><p><strong>索引存储在HDFS上</strong><br />索引的大小通常在G和几十G，上百G的很少，这样的功能或许很难实用。但是，如果你有上亿数据来建索引的话，也是可以考虑一下的。我觉得这个功能最大的好处或许就是和下面这个“通过MR批量创建索引”联合实用。</p><p><strong>通过MR批量创建索引</strong><br />有了这个功能，你还担心创建索引慢吗？<br />强大的RESTful API通常你能想到的管理功能，都可以通过此API方式调用。这样写一些维护和管理脚本就方便多了。</p><p><strong>优秀的管理界面</strong><br />主要信息一目了然；可以清晰的以图形化方式看到SolrCloud的部署分布；当然还有不可或缺的Debug功能。</p>
tags: [hadoop生态圈, solr]
categories: solr
date: 2016-6-17 23:04:26
---

## 简介

### SolrCloud的特色功能

**集中式的配置**
信息使用ZK进行集中配置。启动时可以指定把Solr的相关配置文件上传Zookeeper，多机器共用。这些ZK中的配置不会再拿到本地缓存，Solr直接读取ZK中的配置信息。配置文件的变动，所有机器都可以感知到。另外，Solr的一些任务也是通过ZK作为媒介发布的。目的是为了容错。接收到任务，但在执行任务时崩溃的机器，在重启后，或者集群选出候选者时，可以再次执行这个未完成的任务。

**自动容错**
SolrCloud对索引分片，并对每个分片创建多个Replication。每个Replication都可以对外提供服务。一个Replication挂掉不会影响索引服务。更强大的是，它还能自动的在其它机器上帮你把失败机器上的索引Replication重建并投入使用。

**近实时搜索**
立即推送式的replication（也支持慢推送）。可以在秒内检索到新加入索引。

**查询时自动负载均衡**
SolrCloud索引的多个Replication可以分布在多台机器上，均衡查询压力。如果查询压力大，可以通过扩展机器，增加Replication来减缓。
自动分发的索引和索引分片发送文档到任何节点，它都会转发到正确节点。

**事务日志**
事务日志确保更新无丢失，即使文档没有索引到磁盘。

**索引存储在HDFS上**
索引的大小通常在G和几十G，上百G的很少，这样的功能或许很难实用。但是，如果你有上亿数据来建索引的话，也是可以考虑一下的。我觉得这个功能最大的好处或许就是和下面这个“通过MR批量创建索引”联合实用。

**通过MR批量创建索引**
有了这个功能，你还担心创建索引慢吗？
强大的RESTful API通常你能想到的管理功能，都可以通过此API方式调用。这样写一些维护和管理脚本就方便多了。

**优秀的管理界面**
主要信息一目了然；可以清晰的以图形化方式看到SolrCloud的部署分布；当然还有不可或缺的Debug功能。

solr6.0.1已经可以在界面管理集群的大部分功能了。

## 概念

**Collection**
在SolrCloud集群中逻辑意义上的完整的索引。它常常被划分为一个或多个Shard，它们使用相同的Config Set。如果Shard数超过一个，它就是分布式索引，SolrCloud让你通过Collection名称引用它，而不需要关心分布式检索时需要使用的和Shard相关参数。

**Config Set**
 Solr Core提供服务必须的一组配置文件。每个config set有一个名字。最小需要包括solrconfig.xml (SolrConfigXml)和schema.xml (SchemaXml)，除此之外，依据这两个文件的配置内容，可能还需要包含其它文件。它存储在Zookeeper中。Config sets可以重新上传或者使用upconfig命令更新，使用Solr的启动参数bootstrap_confdir指定可以初始化或更新它。

**Core**
 也就是Solr Core，一个Solr中包含一个或者多个Solr Core，每个Solr Core可以独立提供索引和查询功能，每个Solr Core对应一个索引或者Collection的Shard，Solr Core的提出是为了增加管理灵活性和共用资源。在SolrCloud中有个不同点是它使用的配置是在Zookeeper中的，传统的Solr core的配置文件是在磁盘上的配置目录中。

**Leader**
 赢得选举的Shard replicas。每个Shard有多个Replicas，这几个Replicas需要选举来确定一个Leader。选举可以发生在任何时间，但是通常他们仅在某个Solr实例发生故障时才会触发。当索引documents时，SolrCloud会传递它们到此Shard对应的leader，leader再分发它们到全部Shard的replicas。

**Replica**
 Shard的一个拷贝。每个Replica存在于Solr的一个Core中。一个命名为“test”的collection以numShards=1创建，并且指定replicationFactor设置为2，这会产生2个replicas，也就是对应会有2个Core，每个在不同的机器或者Solr实例。一个会被命名为test_shard1_replica1，另一个命名为test_shard1_replica2。它们中的一个会被选举为Leader。

**Shard**
 Collection的逻辑分片。每个Shard被化成一个或者多个replicas，通过选举确定哪个是Leader。

**Zookeeper**
 Zookeeper提供分布式锁功能，对SolrCloud是必须的。它处理Leader选举。Solr可以以内嵌的Zookeeper运行，但是建议用独立的，并且最好有3个以上的主机。
 

## 新特性

Solr 6.0发布，新特性解析

SOLR-8831: allow _version_ field to be retrievable via docValues
docValues看来是越来越成熟了，之前使用给性能造成不小的影响，这个有待测试；

继地理信息检索之后又一重大新特性，从描述上看是图数据结构检索。
SOLR-7543: Basic graph traversal query Example: {!graph from="node_id" to="edge_id"}id:doc_1

虽然join在分布式模式下不建议使用，但无法避免很多时候还是需要；
SOLR-7584: Adds Inner and LeftOuter Joins to the Streaming API and Streaming Expressions
SOLR-8188: Adds Hash and OuterHash Joins to the Streaming API and Streaming Expressions

管理界面功能日渐完善，增加jdbc驱动支持，增加多个sql特性，门槛越来越低了，也开始贴近大数据分析需求。

## 集群部署

### 软件需求

1、必须使用jdk1.8或更高版本
2、本次使用jdk1.8.0_92 + solr6.0.1 + zookeeper3.4.5集群模式
3、请先下载安装zookeeper，本文没写zookeeper安装

### solr集群安装

#### 初始化zookeeper目录

经测试必须先创建zookeeper目录，否则会报错，连接zookeeper创建/solr目录：
```
# 进入zookeeper后台命令，默认连接本地localhost:2181
./zookeeper/bin/zkCli.sh

# 创建目录
create /solr "solr"
```



#### 安装solr自带例子


**下载**

```
webget https://mirrors.tuna.tsinghua.edu.cn/apache/lucene/solr/6.1.0/solr-6.1.0.tgz

#解压
tar zxvf solr-6.0.1.tgz 
```


#### 初始化solr例子配置文件


**上传solr自带例子配置文件至zookeeper的/solr/configs目录下：**

```
./bin/solr zk -z localhost:2181/solr -upconfig -n solr -d example/example-DIH/solr/solr
```

**参数说明**
```
-upconfig to move a configset from the local machine to Zookeeper.
-n configName    Name of the configset in Zookeeper that will be the destinatino of 'upconfig' and the source for 'downconfig'.
-d confdir       The local directory the configuration will be uploaded from 
-z zkHost        Zookeeper connection string.
```

<font color="red">在Solr部署中，必须先将collection配置文件上传至zookeeper中，后面才能创建collection。如果你不是新手，你也可以上传你自己真正要使用的配置文件。</font>

以前的schema.xml换成了managed-schema文件，具体配置文档：https://cwiki.apache.org/confluence/display/solr/Documents%2C+Fields%2C+and+Schema+Design

#### 创建collection并导入测试数据

**进入solr-6.0.1目录，启动服务：**
```
cd solr-6.0.1
./bin/solr  start -c -m 1g -z localhost:2181/solr -p 8983
```

**参数说明：**
```
-c：cloud模式启动
-m：最大内存使用
-z：zookeeper集群地址
-p：启动服务端口
```

本次测试在一台机器上装三个solr示例，用端口区分（如多台机器可以将solr复制过去直接启动）：
```
./bin/solr  start -c -m 1g -z localhost:2181/solr -p 8984
./bin/solr  start -c -m 1g -z localhost:2181/solr -p 8985
```

命令行操作参数参考：https://cwiki.apache.org/confluence/display/solr/Command+Line+Utilities

**solr面板：**
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr6.0.1-dashboard.png)

**在页面创建collections集群**

笔者服务器ip为：`192.168.31.135`，你们可以将下面ip换成自己的。

1、浏览器访问http://192.168.31.135:8983，点击collections，点击Add Collection，示例如下：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr6.0.1-collections-addCollection.png)

具体参数描述在restful api接口中会说明。


2、注意有可能报“Connection to Solr lost”错误，不过没关系，重新刷新浏览器即可：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr6.0.1-collections-addCollection-cloud.png)

看到该页面则表示collection集群创建成功。


**使用restful api创建collections集群**

刚才在页面上创建collections集群是为了尝鲜一下solr的新功能。在实际部署中建议还是采用restful api的方式创建（可以传入更多参数）。

直接通过 REST 接口来创建 Collection，你也可以通过浏览器访问下面地址，如下所示：
```
#先删除刚才创建好的collections集群
curl 'http://192.168.31.135:8983/solr/admin/collections?action=DELETE&name=solr'

删除后可以刷新浏览器确认是否删除，确认后再执行下面操作。

#创建collections集群
curl 'http://192.168.31.135:8983/solr/admin/collections?action=CREATE&name=solr&numShards=3&replicationFactor=2&maxShardsPerNode=2'
```

上面链接中的几个参数的含义，说明如下：
```
•   name： 待创建Collection的名称
•   config set： collection在zookeeper中的配置目录
•   numShards： 分片的数量
•   replicationFactor： 复制副本的数量
•   maxShardsPerNode：默认值为1，注意三个数值：numShards、replicationFactor、liveSolrNode，一个正常的solrCloud集群不容许同一个liveSolrNode上部署同一个shard的多个replic，因此当maxShardsPerNode=1时，numShards*replicationFactor>liveSolrNode时，报错。因此正确时因满足以下条件：
numShards*replicationFactor<liveSolrNode*maxShardsPerNode
```


所有restful api接口文档参考：https://cwiki.apache.org/confluence/display/solr/Collections+API


集群截图如下：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr6.0.1-collections-addCollection-cloud.png)

**导入测试数据**

导入solr自带的测试数据：
```
./bin/post -c solr example/exampledocs/*.xml
```

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr6.0.1-collection-importdata.png)

**查询数据**

1、浏览器访问http://192.168.31.135:8983，选择solr 
collection，点击Query，示例如下：

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr6.0.1-collection-querydata.png)

#### 集群管理

collection和replia管理：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr6.0.1-collections-addCollection1.png)

schema管理：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr6.0.1-collections-addCollection2.png)

字段的热门关键字是什么？（提示：在Schema browser中选择某个字段，然后点击 Load Term Info 按钮）

这些都是通过页面来管理集群，想要更多操作请参考上面的restful api接口文档。

## 总结

Solr版本更新速度确实很快，从去年4.x到今年的6.x，今天看到6.0.1的功能，着实让我很兴奋。
Solr6.0.1中页面管理功能日渐完善，以及新的一些特性，docValues越来越成熟了，增加jdbc驱动支持，增加多个sql特性，门槛越来越低了，也开始贴近大数据分析需求。

后续我会整理如下几个方面的知识点：
1、jdbc客户端连接查询
2、MapReduce批量导入数据
3、Spark批量导入数据


## 参考文档

- https://cwiki.apache.org/confluence/display/solr/Collections+API （restful api）
- https://cwiki.apache.org/confluence/display/solr/- Command+Line+Utilities （命令行工具）
- https://cwiki.apache.org/confluence/display/solr/Running+Solr+on+HDFS#RunningSolronHDFS-autoAddReplicasSettings （replica自动复制）
- https://cwiki.apache.org/confluence/display/solr/Managing+Solr（solr管理）
- https://cwiki.apache.org/confluence/display/solr/The+Well-Configured+Solr+Instance （solr配置）
- https://cwiki.apache.org/confluence/display/solr/Documents%2C+Fields%2C+and+Schema+Design（schema配置）
