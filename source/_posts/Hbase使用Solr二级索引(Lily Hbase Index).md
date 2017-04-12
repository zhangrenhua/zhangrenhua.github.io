---
title: Hbase使用Solr二级索引(Lily Hbase Index)
description: Hbase使用Solr二级索引(Lily Hbase Index)
tags: [hadoop生态圈, hbase]
categories: hbase
date: 2017-2-10 17:23:53
---

## Solr部署

### 安装Solr集群

如果是Cloudera或者其它hadoop商业发行版，可自行参考官方操作手册，
如果是自己手动离线安装，可以参考 [solr相关文档](http://www.zhangrenhua.com/categories/solr/)。

### 创建collection


#### 创建默认的collection配置文件

```
solrctl instancedir --generate enterprise_list
```

注：此命令会在当前目录下生成enterprise_list文件夹，下面对应默认的solr配置。


#### 修改solr schema.xml配置文件

```
<fields>
  <!-- 
       必须字段 
   -->   
  <!-- hbase rowKey -->
  <field name="id" type="string" indexed="true" stored="true" multiValued="false" /> 
  <!-- hbase version -->
  <field name="_version_" type="long" indexed="true" stored="true"/>
  <!-- solr默认需要字段 -->
  <field name="text" type="string" indexed="true" stored="true"/>
  
   <!-- 
       基本字段，使用string
   -->
   <!-- 统一社会信用代码 -->
   <field name="uniform_social_credit_code" required="true" type="string" stored="true" indexed="true"/>
   <!-- 注册号 -->
   <field name="registration_no" type="string" indexed="true" stored="false"/>
   <!-- 纳税人识别号 -->
   <field name="taxpayer_identification_no" type="string" indexed="true" stored="false"/>
   <!-- 税务登记证 -->
   <field name="tax_registration_cert" type="string" indexed="true" stored="false"/>
   <!-- 组织机构代码 -->
   <field name="organization_code" type="string" indexed="true" stored="false"/>

   <!-- 
       法人代表、股东、高管信息、联系方式，
       通过WhitespaceTokenizerFactory切分，索引。
   -->
   <!-- 法人代表 -->
   <field name="legal_representative_name" type="text_ws" indexed="true" stored="true"/>
   <!-- 股东 -->
   <field name="shareholders" type="text_ws" indexed="true" stored="false"/>
   <!-- 高管 -->
   <field name="senior_executive" type="text_ws" indexed="true" stored="false"/>
   <!-- 联系方式 -->
   <field name="contact" type="text_ws" indexed="true" stored="false"/>
   <!-- 企业类型 -->
   <field name="enterprise_type" type="string" indexed="false" stored="false"/>
   <!-- 企业状态 -->
   <field name="enterprise_status" type="string" indexed="false" stored="true"/>
   <!-- 注册日期 -->
   <field name="registration_date" type="date" indexed="true" stored="true"/>
   <!-- 注册资本 -->
   <field name="registered_capital" type="double" indexed="true" stored="true"/>
   <!-- 所属区域 -->
   <field name="region" type="string" indexed="true" stored="false"/>
   <!-- 所属行业 -->
   <field name="industry_type" type="string" indexed="true" stored="true"/>

   <!-- 
       利用中文分词技术，分词索引和查询。
   -->
   <!-- 企业名称 -->
   <field name="enterprise_name" type="text_ik" indexed="true" stored="true"/>
   <!-- 地址 -->
   <field name="address" type="text_ik" indexed="true" stored="false"/>
  
</fields>


<!-- Field to use to determine and enforce document uniqueness. 
     Unless this field is marked with required="false", it will be a required field
  -->
<uniqueKey>id</uniqueKey>
```

注：scheam.xml在刚才生成的enterprise_list/conf目录下。

#### IK分词配置

在scheam.xml的types标签下添加如下内容：
```
    <!--配置IK分词器-->
    <fieldType name="text_ik" class="solr.TextField">
        <!--索引时候的分词器-->
        <analyzer type="index" isMaxWordLength="false" class="org.wltea.analyzer.lucene.IKAnalyzer"/>
        <!--查询时候的分词器-->
        <analyzer type="query" isMaxWordLength="true" class="org.wltea.analyzer.lucene.IKAnalyzer"/>
    </fieldType>
```

1、在/opt/cloudera/parcels/CDH/lib/solr/webapps/solr/WEB-INF创建classes目录
2、把IKAnalyzer.cfg.xml 和 stopword.dic添加到classes目录（不配置也行）
3、把IKAnalyzer2012FF_u1.jar添加到/opt/cloudera/parcels/CDH/lib/solr/webapps/solr/WEB-INF/lib目录
4、使用scp命令将jar文件传至，其它solr集群机器上去（参考：`scp IKAnalyzer2012FF_u1.jar phfinance-s6:/opt/cloudera/parcels/CDH/lib/solr/webapps/solr/WEB-INF/lib`）

<font color="red">一定要将jar文件，复制到其它solr集群机器上去，重启solr集群</font>。


#### 上传配置文件至zookeeper

```
solrctl --zk cumulus-s1:2181/solr instancedir --create enterprise_list enterprise_list/
```

参数说明:
`create`：表示创建，如果是更新配置文件，可以使用update（建议先delete再create）
`enterprise_list`：collection名称
`enterprise_list/`：solr配置文件目录

使用`zookeeper-client -server cumulus-s1:2181`命令登录zookeeper命令行查看文件目录：
![zookeeper cli](http://7xoqbc.com1.z0.glb.clouddn.com/hbase-solr-index2.jpg)

#### 创建collection

```
solrctl --solr http://cumulus-s1:8983/solr/ collection --create enterprise_list -s 3 -r 2 -m 5 -a
```

参数说明：
`create`：表示创建，如果是更新配置文件，可以使用reload
`enterprise_list`：collection名称
`-s`：<numShards>
`-a`：Create collection with autoAddReplicas=true
`-c`：<collection.configName>
`-r`：<replicationFactor>
`-m`：<maxShardsPerNode>
`-n`：<createNodeSet>

可以使用`solrctl --help`命令，来查看帮助信息。


## hbase部署


###  Hbase表需要开启REPLICATION复制功能

新建表：
```
create 'hermes:c_enterprise_list','info',{NAME => 'info', REPLICATION_SCOPE =>1} 
#其中1表示开启replication功能，0表示不开启，默认为0 
```

对于已经创建的表可以使用如下命令：
```
disable 'hermes:c_enterprise_list' 
alter 'hermes:c_enterprise_list',{NAME => 'info', REPLICATION_SCOPE => 1} 
enable 'hermes:c_enterprise_list' 
```


### Lily HBase Indexer配置


#### 创建Lily HBase Indexer配置文件

morphline-hbase-mapper.xml
```
<?xml version="1.0"?>  
<indexer table="hermes:c_enterprise_list" mapper="com.ngdata.hbaseindexer.morphline.MorphlineResultToSolrMapper">  
   
   <!-- The relative or absolute path on the local file system to the morphline configuration file. -->  
   <!-- Use relative path "morphlines.conf" for morphlines managed by Cloudera Manager -->  
   <param name="morphlineFile" value="morphlines.conf"/>  
      
   <!-- The optional morphlineId identifies a morphline if there are multiple morphlines in morphlines.conf -->  
   <param name="morphlineId" value="enterprise_list"/>
   
</indexer>  
```

`morphlineFile`：使用cloudera则默认为morphlines.conf
`morphlineId`： 对应Key-Value Store Indexer 中配置文件Morphlines.conf 中morphlines 属性id值

参考：https://github.com/NGDATA/hbase-indexer/wiki/Indexer-configuration#table。

#### 修改Morphlines 文件

进入Key-Value Store Indexer面板->配置->查看和编辑->属性-Morphline文件：
![morphline配置](http://7xoqbc.com1.z0.glb.clouddn.com/hbase-solr-index1.jpg)

morphlines.conf
```
SOLR_LOCATOR : {
  # Name of solr collection
  collection : collection
  
  # ZooKeeper ensemble
  zkHost : "$ZK_HOST" 
}


morphlines : [
{
id : enterprise_list
importCommands : ["org.kitesdk.**", "com.ngdata.**"]

commands : [                    
  {
    extractHBaseCells {
        mappings : [  
            {  
              inputColumn : "info:uniform_social_credit_code"  
              outputField : "uniform_social_credit_code"  
              type : string   
              source : value  
            }
            {  
              inputColumn : "info:registration_no"  
              outputField : "registration_no"  
              type : string   
              source : value  
            }
            {  
              inputColumn : "info:taxpayer_identification_no"  
              outputField : "taxpayer_identification_no"  
              type : string   
              source : value  
            }
            {  
              inputColumn : "info:tax_registration_cert"  
              outputField : "tax_registration_cert"  
              type : string   
              source : value  
            }
            {  
              inputColumn : "info:organization_code"  
              outputField : "organization_code"  
              type : string   
              source : value  
            }
            {  
              inputColumn : "info:legal_representative_name"  
              outputField : "legal_representative_name"  
              type : string   
              source : value  
            } 
            {  
              inputColumn : "info:shareholders"  
              outputField : "shareholders"  
              type : string   
              source : value  
            } 
            {  
              inputColumn : "info:senior_executive"  
              outputField : "senior_executive"  
              type : string   
              source : value  
            }  
            {  
              inputColumn : "info:contact"  
              outputField : "contact"  
              type : string   
              source : value  
            }
            {  
              inputColumn : "info:enterprise_type"  
              outputField : "enterprise_type"  
              type : string   
              source : value  
            } 
            {  
              inputColumn : "info:enterprise_status"  
              outputField : "enterprise_status"  
              type : string   
              source : value  
            }
            {  
              inputColumn : "info:registration_date"  
              outputField : "registration_date"  
              type : string   
              source : value  
            } 
            {  
              inputColumn : "info:registered_capital"  
              outputField : "registered_capital"  
              type : string  
              source : value  
            } 
            {  
              inputColumn : "info:region"  
              outputField : "region"  
              type : string   
              source : value  
            } 
            {  
              inputColumn : "info:industry_type"  
              outputField : "industry_type"  
              type : string   
              source : value  
            } 
            {  
              inputColumn : "info:enterprise_name"  
              outputField : "enterprise_name"  
              type : string   
              source : value  
            } 
            {  
              inputColumn : "info:address"  
              outputField : "address"  
              type : string   
              source : value  
            }               
          ] 
    }
  }
  { convertTimestamp {
        field : registration_date
        inputFormats : ["yyyy年MM月dd日", "yyyyMMdd", "yyyy-MM-dd"]
        outputTimezone : UTC
        
  }}

  { logDebug { format : "output record: {}", args : ["@{}"] } }
]
}
]
```

`id`：表示当前morphlines文件的ID名称。
`importCommands`：需要引入的命令包地址。
`extractHBaseCells`：该命令用来读取HBase列数据并写入到SolrInputDocument对象中，该命令必须包含零个或者多个mappings命令对象。
`mappings`：用来指定HBase列限定符的字段映射。
`inputColumn`:需要写入到solr中的HBase列字段。值包含列族和列限定符，并用‘ : ’分开。其中列限定符也可以使用通配符‘*’来表示，譬如可以使用data:*表示读取只要列族为data的所有hbase列数据，也可以通过data:my*来表示读取列族为data列限定符已my开头的字段值。
`outputField`:用来表示morphline读取的记录需要输出的数据字段名称，<font color="red">该名称必须和solr中的schema.xml文件的字段名称保持一致，否则写入不正确</font>。
`type`:用来定义读取HBase数据的数据类型，我们知道HBase中的数据都是以byte[]的形式保存，但是所有的内容在Solr中索引为text形式，所以需要一个方法来把byte[]类型转换为实际的数据类型。type参数的值就是用来做这件事情的。现在支持的数据类型有：byte[](原封不动的拷贝hbase中的byte[]数据),int,long,string,boolean,float,double,short和bigdecimal。当然你也可以指定自定的数据类型，只需要实现com.ngdata.hbaseindexer.parse.ByteArrayValueMapper接口即可。
`source`:用来指定HBase的KeyValue那一部分作为索引输入数据，可选的有‘value’和'qualifier',当为value的时候表示使用HBase的列值作为索引输入，当为qualifier的时候表示使用HBase的列限定符作为索引输入。

#### 注册Lily HBase Indexer

注册Lily HBase Indexer configuration 和 Lily Hbase Indexer Service
```
hbase-indexer add-indexer \
    --name enterprise_list \
    --indexer-conf /home/hermes/hermes-solr/morphline-hbase-mapper.xml \
    --connection-param solr.zk=cumulus-s1:2181,cumulus-s2:2181,cumulus-s3:2181/solr \
    --connection-param solr.collection=enterprise_list \
    --zookeeper cumulus-s1:2181,cumulus-s2:2181,cumulus-s3:2181
```

`name`：indexer名称
`indexer-conf`：hbase映射配置文件目录
`connection-param`：solr配置信息
`zookeeper`：zookeeper配置信息


直接输入`hbase-indexer`命令，即可查看命令行帮助信息。


#### 验证Lily HBase Indexer

验证索引器是否成功创建，`hbase-indexer list-indexers`：
![list-indexers](http://7xoqbc.com1.z0.glb.clouddn.com/hbase-solr-index3.jpg)

hbase put数据：
```
put 'hermes:c_enterprise_list','abcd','info:uniform_social_credit_code','2017'
```

稍等一会（根据solr刷新间隔决定），查询solr，检查索引是否建好。

![solr query](http://7xoqbc.com1.z0.glb.clouddn.com/hbase-solr-index3.jpg)


### Hbase表数据全量导入到SOLR集群

```
hadoop --config /etc/hadoop/conf \
  jar /opt/cloudera/parcels/CDH/lib/hbase-solr/tools/hbase-indexer-mr-job.jar \
  --conf /etc/hbase/conf/hbase-site.xml \
  -D 'mapred.child.java.opts=-Xmx1g' \
  --hbase-indexer-file  /home/hermes/hermes-solr/morphline-hbase-mapper.xml  \
  --zk-host cumulus-s1:2181,cumulus-s2:2181,cumulus-s3:2181/solr \
  --collection enterprise_list \
  --reducers 0 
```

在执行上面命令的目录下面，创建`morphline-hbase-mapper.xml`和`morphlines.conf`文件，内容从上面章节复制。

参考：
- https://www.cloudera.com/documentation/enterprise/latest/topics/search_hbase_mapreduce_indexer_tool.html
- http://kitesdk.org/docs/1.1.0/morphlines/morphlines-reference-guide.html

## 总结

1、经验证，Lily HBase Indexer，能满足hbase数据实时索引至solr中，包括实时新增、更新、删除。
2、Lily HBase Indexer采用离线生成solr索引文件，对solr查询无任何负载。
3、可能会出现solr索引数据文件已生成，但是在solr中查询不到数据，这是solr加载索引文件的时间间隔导致的，耐心等待一会即可。

本文档，是在Cloudera5.9集群基础之上操作的。

[示例附件](http://7xoqbc.com1.z0.glb.clouddn.com/hermes-solr.tar.gz)

