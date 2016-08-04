---
title: Solr运维管理
description: Solr运维管理，主要有Collection、Core、Shard、Replica的创建删除与更新，以及后续会添加一些应急处理方案。
tags: [hadoop生态圈, solr]
categories: solr
date: 2016-6-23 09:29:24
---

## 简介

Solr运维管理，主要有Collection、Core、Shard、Replica的创建删除与更新，以及后续会添加一些应急处理方案。

Solr概念与集群安装请参考《[Solr6.0.1概念和集群部署](/2016/06/17/Solr6.0.1概念和集群部署/)》


## Collection

### 创建Collection

参考《[Solr6.0.1概念和集群部署](/2016/06/17/Solr6.0.1概念和集群部署/)》

注意：一定要将collection配置文件上传到zookeeper

### 删除Collection

```
http://localhost:8983/solr/admin/collections?action=DELETE&name=newCollection
```

删除collection后，数据目录和zookeeper中的/solr/configs/`newCollection`目录不会被删除，建议手动删除。

### 重新加载配置

```
http://localhost:8983/solr/admin/collections?action=RELOAD&name=newCollection
```

修改配置文件后，可以重新上传至zookeeper中，并执行upload。


### 修改参数项

```
http://localhost:8983/solr/admin/collections?action=MODIFYCOLLECTION&name=newCollection&maxShardsPerNode=2
```

可以修改的属性有:
- `maxShardsPerNode`
- `replicationFactor`
- `autoAddReplicas`
- `rule`
- `snitch`
有关这些属性的详细信息，参见创建部分。

注意：此API只更新collection中的配置属性。

## Core

### 增加Core

#### RestFul API创建Core

**执行命令**

```
http://hynamet01:8983/solr/admin/cores?action=CREATE&name=collection1_shard3_replica1&instanceDir=collection1_shard3_replica1&dataDir=data&collection=collection1&shard=shard3'
```

**参数描述**
`collection`：collection名称
`name`：core名称，不能和已有的core名称冲突
`shard`：shard名称，不能和已有的shard名称冲突
`instanceDir`：本地配置目录，一般与name保持一致
`dataDir`：存储目录，相对路径，默认data即可

上面命令会创建一个新的（Shard/Core）
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr-operation-addCore.png)


创建好Core后，会默认在`solr.solr.home`目录下创建与`name`相同的目录，目录下会生成一个core.properties：
```
#Written by CorePropertiesLocator
#Thu Jun 23 15:06:13 CST 2016
name=collection1_shard3_replica1
config=solrconfig.xml
schema=schema.xml
shard=shard3
dataDir=data
collection=collection1
coreNodeName=core_node3
```


`solr.solr.home`目录指定：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr-operation-addCore-ps.png)

#### 命令行创建

**创建目录**
```
#在solr.solr.home目录下创建目录
mkdir /var/lib/solr/collection1_shard3_replica1
```

**创建core.properties文件，写入示例文件中的内容**
```
#Written by CorePropertiesLocator
#Thu Jun 23 15:06:13 CST 2016
name=collection1_shard3_replica1
config=solrconfig.xml
schema=schema.xml
shard=shard3
dataDir=data
collection=collection1
coreNodeName=core_node3
```

**创建core**

```
./bin/solr create_core -c collection1_shard3_replica1 -d /var/lib/solr/collection1_shard3_replica1
```

填写参数见“RestFul API创建Core”步骤。

#### 界面创建

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr-operation-addCore-page.png)

填写参数见“RestFul API创建Core”步骤。


#### 注意

1、<font color="red">url中指定哪台solr服务器，就会在那台机器上创建该Core</font>
2、建议使用Restful API或者前台界面创建


### 删除Core

```
http://hynamet01:8983/solr/admin/cores?action=UNLOAD&core=collection1_shard3_repl
```

Core删除后数据文件会删除，但是数据目录不会删除，建议手动删除。


## Replication 

Shard的一个拷贝。每个Replica存在于Solr的一个Core中。一个命名为“test”的collection以numShards=1创建，并且指定replicationFactor设置为2，这会产生2个replicas，也就是对应会有2个Core，每个在不同的机器或者Solr实例。一个会被命名为test_shard1_replica1，另一个命名为test_shard1_replica2。它们中的一个会被选举为Leader。

### 添加Replication

```
http://hynamet02:8983/solr/admin/collections?action=ADDREPLICA&collection=collection1&shard=shard2&node=hynamet01:8983_solr
```

**参数说明**
`collection`：collection名称
`shard`：要添加副本的shard名称，在Cloud界面上可以查看
`node`：在哪台机器上添加该副本，下面会说如何查看nodes
`async`：异步添加
`property.coreNodeName`：

界面查看zk live_nodes目录：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr-operation-addReplica-listNodes.png)

查看添加结果：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr-operation-addReplica-demo.png)

副本添加成功后会立即复制数据到新增的副本中。

### 删除Replication

```
http://hynamet02:8983/solr/admin/collections?action=DELETEREPLICA&collection=collection1&shard=shard2&replica=core_node3
```

参数说明：
`collection`：collection名称
`shard`：副本所属shardpro
`replica`：副本所属的node名称

界面查看zk /clusterstate.json文件来查找`replica`的值：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr-operation-RmReplica-demo.png)

通过所属collection、shard、副本core名称来查找node名称。

## Shard Split

SolrCloud不支持自动分片，但是支持手动分片，而且手动分片后得到的新的分片所包含的Document数量有一定的差异（还不清楚SolrCloud是否支持手动分片后大致均分一个分片）。

在执行Split的时候，该Shard就不能执行写入操作。


```
http://hynamet01:8983/solr/admin/collections?action=SPLITSHARD&collection=collection1&shard=shard1&async=1000
```

**参数说明：**
`collection`：collection名称
`shard`：要删除的shard1
`async`：由于删除操作时间可能会比较长，建议使用异步操作
可选参数
`ranges`：一个逗号分隔的列表的哈希范围进制例如范围= 0-1f4,1f5-3e8,3e9-5dc
`split.key`：拆分的建


此时已经将shard1分成了两个子分片：shard1_0和shard1_1，此时，在hynamet01节点上有3个分片处于“Active”状态。实际上，到目前为止，子分片shard1_0和shard1_1已经完全接管了shard1分片，只是没有从图中自动离线退出，这个就需要我们在管理页面你上手动“unload”掉不需要的shard。
这时，新得到的两个子分片，并没有处理之前shard1的两个副本，他们也需要进行分片（实际上是重新复制新分片的副本），这个过程，如图所示：

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr-operation-shard-split1.png)


通过API删除shard（也可以通过界面手动删除shard对应的所有core）：
```
http://hynamet01:8983/solr/admin/collections?action=DELETESHARD&collection=collection1&shard=shard1&async=1000
```

**参数说明：**
`collection`：collection名称
`shard`：要删除的shard1
`async`：由于删除操作时间可能会比较长，建议使用异步操作
可选参数
`deleteInstanceDir`：是否删除实例目录
`deleteDataDir`：是否删除数据目录
`deleteIndex`：默认会删除shard下面的每个副本索引数据


删除后的效果：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/solr-operation-shard-split2.png)





未完待续。。。




## 参考文档

- https://cwiki.apache.org/confluence/display/solr/Collections+API （restful api）
