---
title: hadoop及其生态圈介绍
description: 接触hadoop已经有2年时间了，从当时的“哈到普”到现在的“哈杜浦”经历了太多的坎坷。去年年初当时还在跳槽找工作，某公司的面试官问我hadoop有了解过没？我当时脑袋一空，本来自信满满的，突然之间感觉自己懂的东西确实不多。但是，对于极度热爱技术的我来说肯定是要去了解去学习的。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-11-22 19:20:00
---

## 背景

接触hadoop已经有2年时间了，从当时的“哈到普”到现在的“哈杜浦”经历了太多的坎坷。去年年初当时还在跳槽找工作，某公司的面试官问我hadoop有了解过没？我当时脑袋一空，本来自信满满的，突然之间感觉自己懂的东西确实不多。但是，对于极度热爱技术的我来说肯定是要去了解去学习的。

当时我就百度、google搜索hadoop，只是初步的了解到了hadoop包含`分布式文件系统`和`分布式计算框架`。然而对于从未接触分布式集群计算的我来说并不知道它（hadoop）能用来做什么😂
经过我们项目经理的推荐，果断买了本《Hadoop权威指南》，后来又不断阅读了《Hive编程指南》、《hbase权威指南》、《实战Hadoop》、《Spark实战》等书籍。然而机缘巧合就在我买书后半个月左右，公司接到了某银行的一个大数据项目。在领导的帮助下果断入行hadoop，进入大数据时代。在此非常感谢一路上帮助过我的人。

下面我会对hadoop以及hadoop生态圈的一些常用组件和hadoop商业发行版进行简要描述。
## 简介
古代，人们用牛来拉重物。当一头牛拉不动一根圆木时，他们不曾想过培养更大更壮的牛。同样，我们也不需要尝试开发超级计算机，而应该试着结合使用更多计算机系统。-- 格蕾斯.霍珀
Hadoop是一个由Apache基金会所开发的分布式系统基础架构。Hadoop的框架最核心的设计就是：HDFS和MapReduce。HDFS为海量的数据提供了存储，则MapReduce为海量的数据提供了计算。
具体细节参见：http://baike.baidu.com/link?url=7CylJxaFCmFqChIOxZfp9XW5j5NdTly4hgxUa-sam7Qe592WKKMUnkBzlkqnDGil0fPM9WqBVqCkAe3QnEa9Dq

### Hadoop是什么
hadoop是什么？
1、Hadoop是一个开源的框架，可编写和运行分布式应用处理大规模数据，是专为离线和大规模数据分析而设计的，并不适合那种对几个记录随机读写的在线事务处理模式。Hadoop=HDFS（文件系统，数据存储技术相关）+ Mapreduce（数据处理），Hadoop的数据来源可以是任何形式，在处理半结构化和非结构化数据上与关系型数据库相比有更好的性能，具有更灵活的处理能力，不管任何数据形式最终会转化为key/value，key/value是基本数据单元。用函数式变成Mapreduce代替SQL，SQL是查询语句，而Mapreduce则是使用脚本和代码，而对于适用于关系型数据库，习惯SQL的Hadoop有开源工具hive代替。

2、Hadoop就是一个分布式计算的解决方案。
 hadoop擅长日志分析，facebook就用Hive来进行日志分析，2009年时facebook就有非编程人员的30%的人使用HiveQL进行数据分析；淘宝搜索中的自定义筛选也使用的Hive；利用Pig还可以做高级的数据处理，包括Twitter、LinkedIn上用于发现您可能认识的人，可以实现类似Amazon.com的协同过滤的推荐效果。淘宝的商品推荐也是！在Yahoo！的40%的Hadoop作业是用pig运行的，包括垃圾邮件的识别和过滤，还有用户特征建模。（2012年8月25新更新，天猫的推荐系统是hive，少量尝试mahout！）

下面举例说明：
设想一下这样的应用场景. 我有一个100M的数据库备份的sql 文件.我现在想在不导入到数据库的情况下直接用grep操作通过正则过滤出我想要的内容。

例如：某个表中含有相同关键字的记录那么有几种方式，一种是直接用linux的命令grep还有一种就是通过编程来读取文件,然后对每行数据进行正则匹配得到结果好了 现在是100M 的数据库备份。

上述两种方法都可以轻松应对。那么如果是1G , 1T 甚至 1PB 的数据呢，上面2种方法还能行得通吗？ 
答案是不能毕竟单台服务器的性能总有其上限，那么对于这种 超大数据文件怎么得到我们想要的结果呢？

有种方法 就是分布式计算, 分布式计算的核心就在于利用分布式算法 把运行在单台机器上的程序扩展到多台机器上并行运行。从而使数据处理能力成倍增加.但是这种分布式计算一般对编程人员要求很高，而且对服务器也有要求。导致了成本变得非常高。

hadoop 就是为了解决这个问题诞生的。hadoop可以很轻易的把很多linux的廉价pc 组成 分布式结点，然后编程人员也不需要知道分布式算法之类，只需要根据mapreduce的规则定义好接口方法，剩下的就交给Haddop。它会自动把相关的计算分布到各个结点上去，然后得出结果。

例如上述的例子：Hadoop要做的事首先把 1PB的数据文件导入到HDFS中,，然后编程人员定义好map和reduce，也就是把文件的行定义为key，每行的内容定义为value，然后进行正则匹配，匹配成功则把结果通过reduce聚合起来返回。Hadoop就会把这个程序分布到N 个结点去并行的操作，那么原本可能需要计算好几天，在有了足够多的节点之后就可以把时间缩小到几小时之内。这就是hadoop分布式计算带来的效率。

### Hadoop优点
Hadoop是一个能够对大量数据进行分布式处理的软件框架。
Hadoop以一种可靠、高效、可伸缩的方式进行数据处理。
Hadoop 是可靠的，因为它假设计算元素和存储会失败，因此它维护多个工作数据副本，确保能够针对失败的节点重新分布处理。
Hadoop 是高效的，因为它以并行的方式工作，通过并行处理加快处理速度。
Hadoop 还是可伸缩的，能够处理 PB 级数据。此外，Hadoop依赖于社区服务，因此它的成本比较低，任何人都可以使用。
Hadoop是一个能够让用户轻松架构和使用的分布式计算平台。用户可以轻松地在Hadoop上

开发和运行处理海量数据的应用程序。它主要有以下几个优点：
**高可靠性**：Hadoop按位存储和处理数据的能力值得人们信赖。
**高扩展性**：Hadoop是在可用的计算机集簇间分配数据并完成计算任务的，这些集簇可以方便地扩展到数以千计的节点中。
**高效性**：Hadoop能够在节点之间动态地移动数据，并保证各个节点的动态平衡，因此处理速度非常快。
**高容错性**：Hadoop能够自动保存数据的多个副本，并且能够自动将失败的任务重新分配。
**低成本**：与一体机、商用数据仓库以及QlikView、YonghongZ-Suite等数据集市相比，hadoop是开源的，项目的软件成本因此会大大降低。
Hadoop带有用Java语言编写的框架，因此运行在 Linux生产平台上是非常理想的。Hadoop 上的应用程序也可以使用其他语言编写，比如 C++、python。

### 目前国内使用大数据的应用方向
目前Hadoop的开发，主要是为了更好地支持数据科学家。Hadoop提供了一个强大的计算平台，拥有高扩展性和并行执行能力，非常适合应用于新一代功能强大的数据科学和企业级应用。并且，Hadoop还提供了可伸缩的分布式存储和MapReduce编程模式。企业正在使用Hadoop解决相关业务问题，主要集中在以下几个方面：

**为银行和信用卡公司增强欺诈性检测**
公司正在利用Hadoop检测交易过程中的欺诈行为。银行通过使用Hadoop，建立大型集群，进行数据分析；并将分析模型应用于银行交易过程，从而提供实时的欺诈行为检测。
**社交媒体市场分析**
公司目前正在使用Hadoop进行品牌管理、市场推广活动和品牌保护。互联网充满了各种资源，例如博客、版面、新闻、推特和社会媒体数据等。公司利用Hadoop监测、收集、汇聚这些信息，并提取、汇总自身的产品和服务信息，以及竞争对手的相关信息，发掘内在商业模式，或者预测未来的可能趋势，从而更加了解自身的业务。
**零售行业购物模式分析**
在零售行业，通过使用Hadoop分析商店的位置和它周围人口的购物模式，来确定商店里哪些产品最畅销。
**城市发展的交通模式识别**
城市发展往往需要依赖交通模式，来确定道路网络扩展的需求。通过监控在一天内不同时间的交通状况，发掘交通模型，城市规划人员就可以确定交通瓶颈。从而决定是否需要增加街道或者车道，来避免在高峰时段的交通拥堵。
**内容优化和内容参与**
企业越来越专注于优化内容，将其呈现在不同的设备上，并支持不同格式。因此，许多媒体公司需要处理大量的不同的格式的内容。所以，必须规划内容参与模式，才能进行反馈和改进。
**网络分析和调解**
针对交易数据、网络性能数据、基站数据、设备数据以及其他形式的后台数据等，进行大数据实时分析，能够降低公司运营成本，增强用户体验。
**大数据转换**
纽约时报要将1100万篇文章（1851至1980年）转换成PDF文件，这些文章都是从报纸上扫描得到的图片。利用Hadoop技术，这家报社能够在24小时内，将4TB的扫描文章转换为1.5TB的PDF文档。
类似的例子数不胜数。企业正在逐步使用Hadoop进行数据分析，从而作出更好的战略决策。总而言之，数据科学已经进入了商界。


## Hadoop生态圈常用的组件

### Yarn/MapReduce
#### MapReduce
MapReduce是一种编程模型，用于大规模数据集（大于1TB）的并行运算。概念"Map（映射）"和"Reduce（归约）"，是它们的主要思想，都是从函数式编程语言里借来的，还有从矢量编程语言里借来的特性。它极大地方便了编程人员在不会分布式并行编程的情况下，将自己的程序运行在分布式系统上。 当前的软件实现是指定一个Map（映射）函数，用来把一组键值对映射成一组新的键值对，指定并发的Reduce（归约）函数，用来保证所有映射的键值对中的每一个共享相同的键组。

##### MapReduce v1存在的缺陷
MapReduce 的第一个版本既有优点也有缺点。MRv1 是目前使用的标准的大数据处理系统。但是，这种架构存在不足，主要表现在大型集群上。当集群包含的节点超过 4,000 个时（其中每个节点可能是多核的），就会表现出一定的不可预测性。其中一个最大的问题是级联故障，由于要尝试复制数据和重载活动的节点，所以一个故障会通过网络泛洪形式导致整个集群严重恶化。

但 MRv1 的最大问题是多租户。随着集群规模的增加，一种可取的方式是为这些集群采用各种不同的模型。MRv1 的节点专用于 Hadoop，所以可以改变它们的用途以用于其他应用程序和工作负载。当大数据和 Hadoop 成为云部署中一个更重要的使用模型时，这种能力也会增强，因为它允许在服务器上对 Hadoop 进行物理化，而无需虚拟化且不会增加管理、计算和输入/输出开销。[2] 

#### Yarn
Apache Hadoop YARN （Yet Another Resource Negotiator，另一种资源协调者）是一种新的 Hadoop 资源管理器，它是一个通用资源管理系统，可为上层应用提供统一的资源管理和调度，它的引入为集群在利用率、资源统一管理和数据共享等方面带来了巨大好处。

YARN最初是为了修复MapReduce实现里的明显不足，并对可伸缩性（支持一万个节点和二十万个内核的集群）、可靠性和集群利用率进行了提升。YARN实现这些需求的方式是，把Job Tracker的两个主要功能（资源管理和作业调度/监控）分成了两个独立的服务程序——全局的资源管理（RM）和针对每个应用的应用 Master（AM），这里说的应用要么是传统意义上的MapReduce任务，要么是任务的有向无环图（DAG）。

YARN从某种那个意义上来说应该算做是一个云操作系统，它负责集群的资源管理。在操作系统之上可以开发各类的应用程序，例如批处理MapReduce、流式作业Storm以及实时型服务Storm等。这些应用可以同时利用Hadoop集群的计算能力和丰富的数据存储模型，共享同一个Hadoop 集群和驻留在集群上的数据。此外，这些新的框架还可以利用YARN的资源管理器，提供新的应用管理器实现

##### Yarn的优点
<font color="red">Yarn是hadoop2.0开始引用的资源管理框架</font>
大大减小了 JobTracker（也就是现在的 ResourceManager）的资源消耗，并且让监测每一个 Job 子任务 (tasks) 状态的程序分布式化了，更安全、更优美。

在新的 Yarn 中，ApplicationMaster 是一个可变更的部分，用户可以对不同的编程模型写自己的 AppMst，让更多类型的编程模型能够跑在 Hadoop 集群中，可以参考 hadoop Yarn 官方配置模板中的 mapred-site.xml 配置。

对于资源的表示以内存为单位 ( 在目前版本的 Yarn 中，没有考虑 cpu 的占用 )，比之前以剩余 slot 数目更合理。

老的框架中，JobTracker 一个很大的负担就是监控 job 下的 tasks 的运行状况，现在，这个部分就扔给 ApplicationMaster 做了，而 ResourceManager 中有一个模块叫做 ApplicationsMasters( 注意不是 ApplicationMaster)，它是监测 ApplicationMaster 的运行状况，如果出问题，会将其在其他机器上重启。

Container 是 Yarn 为了将来作资源隔离而提出的一个框架。这一点应该借鉴了 Mesos 的工作，目前是一个框架，仅仅提供 java 虚拟机内存的隔离,hadoop 团队的设计思路应该后续能支持更多的资源调度和控制 , 既然资源表示成内存量，那就没有了之前的 map slot/reduce slot 分开造成集群资源闲置的尴尬情况。[1] 

##### YARN的核心思想
将JobTracker和TaskTacker进行分离，它由下面几大构成组件：
1、一个全局的资源管理器 ResourceManager
2、ResourceManager的每个节点代理 NodeManager
3、表示每个应用的 ApplicationMaster
4、每一个ApplicationMaster拥有多个Container在NodeManager上运行

### hbase
HBase是一个分布式的、面向列的开源数据库，该技术来源于 Fay Chang 所撰写的Google论文“Bigtable：一个结构化数据的分布式存储系统”。就像Bigtable利用了Google文件系统（File System）所提供的分布式数据存储一样，HBase在Hadoop之上提供了类似于Bigtable的能力。HBase是Apache的Hadoop项目的子项目。HBase不同于一般的关系数据库，它是一个适合于非结构化数据存储的数据库。另一个不同的是HBase基于列的而不是基于行的模式。

### phoenix
phoenix，由saleforce.com开源的一个项目，后又捐给了Apache。它相当于一个Java中间件，帮助开发者，像使用jdbc访问关系型数据库一些，访问NoSql数据库HBase。

### hive
hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供简单的sql查询功能，可以将sql语句转换为MapReduce任务进行运行。 其优点是学习成本低，可以通过类SQL语句快速实现简单的MapReduce统计，不必开发专门的MapReduce应用，十分适合数据仓库的统计分析。

### impala
Impala像Dremel一样，其借鉴了MPP（Massively Parallel Processing，大规模并行处理）并行数据库的思想，抛弃了MapReduce这个不太适合做SQL查询的范式，从而让Hadoop支持处理交互式的工作负载，Cloudera 公司采用C++编写并捐赠给apache。

Cloudera Impala 可以直接为存储在HDFS或HBase中的Hadoop数据提供快速，交互式的SQL查询。除了使用相同的存储平台外， Impala和Apache Hive一样也使用了相同的元数据，SQL语法（Hive SQL），ODBC驱动和用户接口（Hue Beeswax），这就很方便的为用户提供了一个相似并且统一的平台来进行批量或实时查询。

Cloudera Impala 是用来进行大数据查询的补充工具。 Impala 并没有取代像Hive这样基于MapReduce的分布式处理框架。Hive和其它基于MapReduce的计算框架非常适合长时间运行的批处理作业，例如那些涉及到批量 Extract、Transform、Load ，即需要进行ETL作业。

### spark
Spark是UC Berkeley AMP lab所开源的类Hadoop MapReduce的通用并行框架，Spark，拥有Hadoop MapReduce所具有的优点；但不同于MapReduce的是Job中间输出结果可以保存在内存中，从而不再需要读写HDFS，因此Spark能更好地适用于数据挖掘与机器学习等需要迭代的MapReduce的算法。

Spark 是一种与 Hadoop 相似的开源集群计算环境，但是两者之间还存在一些不同之处，这些有用的不同之处使 Spark 在某些工作负载方面表现得更加优越，换句话说，Spark 启用了内存分布数据集，除了能够提供交互式查询外，它还可以优化迭代工作负载。

Spark 是在 Scala 语言中实现的，它将 Scala 用作其应用程序框架。与 Hadoop 不同，Spark 和 Scala 能够紧密集成，其中的 Scala 可以像操作本地集合对象一样轻松地操作分布式数据集。
尽管创建 Spark 是为了支持分布式数据集上的迭代作业，但是实际上它是对 Hadoop 的补充，可以在 Hadoop 文件系统中并行运行。通过名为 Mesos 的第三方集群框架可以支持此行为。Spark 由加州大学伯克利分校 AMP 实验室 (Algorithms, Machines, and People Lab) 开发，可用来构建大型的、低延迟的数据分析应用程序。

Spark1.5具有以下几个重要模块：
SQL and DataFrames
Spark Streaming
MLib(machine learning)
GraphX(graph)
Third-party Packages

### Mesos
Apache Mesos是由加州大学伯克利分校的AMPLab首先开发的一款开源群集管理软件，支持Hadoop、ElasticSearch、Spark、Storm 和Kafka等应用架构。

### sqoop
Sqoop是一个用来将Hadoop和关系型数据库中的数据相互转移的工具，可以将一个关系型数据库（例如 ： MySQL ,Oracle ,Postgres等）中的数据导入到Hadoop的HDFS中，也可以将HDFS的数据导入到关系型数据库中。

### nutch
Nutch 是一个开源Java 实现的搜索引擎。它提供了我们运行自己
的搜索引擎所需的全部工具。包括全文搜索和Web爬虫。

### solr
Solr是一个独立的企业级搜索应用服务器，它对外提供类似于Web-service的API接口。用户可以通过http请求，向搜索引擎服务器提交一定格式的XML文件，生成索引；也可以通过Http Get操作提出查找请求，并得到XML格式的返回结果。

### zookeeper
ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，是Google的Chubby一个开源的实现，是Hadoop和Hbase的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、名字服务、分布式同步、组服务等。

### pig
Apache Pig 是一个高级过程语言，适合于使用 Hadoop 和 MapReduce 平台来查询大型半结构化数据集。通过允许对分布式数据集进行类似 SQL 的查询，Pig 可以简化 Hadoop 的使用。本文将探索 Pig 背后的语言，并在一个简单的 Hadoop 集群中发现其用途。

### oozie
在Hadoop中执行的任务有时候需要把多个Map/Reduce作业连接到一起，这样才能够达到目的。在Hadoop生态圈中，有一种相对比较新的组件叫做Oozie，它让我们可以把多个Map/Reduce作业组合到一个逻辑工作单元中，从而完成更大型的任务。

### kafka
Kafka是一种高吞吐量的分布式发布订阅消息系统，它可以处理消费者规模的网站中的所有动作流数据。 这种动作（网页浏览，搜索和其他用户的行动）是在现代网络上的许多社会功能的一个关键因素。这些数据通常是由于吞吐量的要求而通过处理日志和日志聚合来解决。 对于像Hadoop的一样的日志数据和离线分析系统，但又要求实时处理的限制，这是一个可行的解决方案。Kafka的目的是通过Hadoop的并行加载机制来统一线上和离线的消息处理，也是为了通过集群机来提供实时的消费。

### flume
Flume是Cloudera提供的一个高可用的，高可靠的，分布式的海量日志采集、聚合和传输的系统，Flume支持在日志系统中定制各类数据发送方，用于收集数据；同时，Flume提供对数据进行简单处理，并写到各种数据接受方（可定制）的能力。

当前Flume有两个版本Flume 0.9X版本的统称Flume-og，Flume1.X版本的统称Flume-ng。由于Flume-ng经过重大重构，与Flume-og有很大不同，使用时请注意区分。

### Storm
Storm用于处理高速、大型数据流的分布式实时计算系统。为Hadoop添加了可靠的实时数据处理功能

### tez
Tez是Apache最新的支持DAG作业的开源计算框架，它可以将多个有依赖的作业转换为一个作业从而大幅提升DAG作业的性能。Tez并不直接面向最终用户——事实上它允许开发者为最终用户构建性能更快、扩展性更好的应用程序。Hadoop传统上是一个大量数据批处理平台。但是，有很多用例需要近乎实时的查询处理性能。还有一些工作则不太适合MapReduce，例如机器学习。Tez的目的就是帮助Hadoop处理这些用例场景。

### Hue
Hue是一个开源的Apache Hadoop UI系统，最早是由Cloudera Desktop演化而来，由Cloudera贡献给开源社区，它是基于Python Web框架Django实现的。通过使用Hue我们可以在浏览器端的Web控制台上与Hadoop集群进行交互来分析处理数据，例如操作HDFS上的数据，运行MapReduce Job等等。Hue所支持的功能特性集合：
1、默认基于轻量级sqlite数据库管理会话数据，用户认证和授权，可以自定义为MySQL、Postgresql，以及Oracle
2、基于文件浏览器（File Browser）访问HDFS
3、基于Hive编辑器来开发和运行Hive查询
4、支持基于Solr进行搜索的应用，并提供可视化的数据视图，以及仪表板（Dashboard）
5、支持基于Impala的应用进行交互式查询
6、支持Spark编辑器和仪表板（Dashboard）
7、支持Pig编辑器，并能够提交脚本任务
8、支持Oozie编辑器，可以通过仪表板提交和监控Workflow、Coordinator和Bundle
9、支持HBase浏览器，能够可视化数据、查询数据、修改HBase表
10、支持Metastore浏览器，可以访问Hive的元数据，以及HCatalog
11、支持Job浏览器，能够访问MapReduce Job（MR1/MR2-YARN）
12、支持Job设计器，能够创建MapReduce/Streaming/Java Job
13、支持Sqoop 2编辑器和仪表板（Dashboard）
14、支持ZooKeeper浏览器和编辑器
15、支持MySql、PostGresql、Sqlite和Oracle数据库查询编辑器

### Mahout
Apache Mahout 是 Apache Software Foundation (ASF) 开发的一个全新的开源项目，其主要目标是创建一些可伸缩的机器学习算法，供开发人员在Apache的许可下免费使用。Mahout宣布新的算法基于Spark实现。

## Hadoop商业发行版
### Cloudera CDH
CDH基于Hadoop2.x，包括HDFS，YARN，HBas，MapReduce，Hive, Pig, Zookeeper, Oozie, Mahout, Hue以及其他开源工具（包括实时查询引擎Impala）。Cloudera的个人免费版包括所有CDH工具，和支持高达50个节点的集群管理器。Cloudera企业版提供了更复杂的管理器，支持无限数量的集群节点，能够主动监控，并额外提供了数据分析工具。

### Hortonworks HDP
发行版2.3基于Hadoop2，包括HDFS，YARN, HBase, MapReduce, Hive, Pig, HCatalog, Zookeeper, Oozie, Mahout, Hue, Ambari, Tez，实时版Hive（Stinger）和其他开源工具。Hortonworks提供了高可用性支持、高性能的Hive ODBC驱动和针对大数据的Talend Open Studio（100%开源）。

### 华为 Fusioninsight2
Fusioninsight Manager作为FusionInsight的运维管理界面，提供界面化的统一安装、告警、监控和集群管理。提供北向接口，实现与企业现有网管系统集成；支持单点登录。当前支持syslog接口，接口消息可通过配置适配现有系统；整个Hadoop集群采用统一的集中管理，未来北向接口可根据需求灵活扩展。提供自动化的二次开发助手和开发样例Demo，帮助软件开发人员快速上手。**提供数据加密以及用户权限等功能。**

### 星环 Transwarp
TRANSWARP DATA HUB(简称TDH)是国内落地案例最多的一站式Hadoop大数据平台。其采用c++重新编写hadoop以及其它组件（**还包括星环科技自己公司的一些产品**），使得性能上有一定优势。

### MapR
组件包括HDFS, HBase, MapReduce, Hive, Mahout, Oozie, Pig, ZooKeeper, Hue以及其他开源工具。它还包括直接NFS访问、快照、“高实用性”镜像、专有的HBase实现，与Apache完全兼容的API和一个MapR管理控制台。**Name Node实现了真正意义上的集群。**

### IBM InfoSphere BigInsights
提供了两个版本。基本版包括HDFS, Hbase, MapReduce, Hive, Mahout, Oozie, Pig, ZooKeeper, Hue以及其他一些开源工具。并提供IBM的安装程序和数据访问工具的基本版本。企业版增加了复杂的作业管理工具、集成了数据源的数据访问层和BigSheets（类似电子表格的界面，用来操作集群中的数据）。

### GreenPlum的Pivotal HD
组件包括HDFS, MapReduce, Hive, Pig, HBase, Zookeeper, Sqoop, Flume和其他开源工具。Pivotal HD企业级还增加了先进的HAWQ数据库服务（ADS），和丰富、成熟、并行的SQL处理工具。

### 亚马逊弹性MapReduce（EMR）
亚马逊EMR是一个web服务，能够使用户方便且经济高效地处理海量的数据。它采用Hadoop框架，运行在亚马逊弹性计算云EC2和简单存储服务S3之上。包括HDFS（S3支持），HBase（专有的备份恢复），MapReduce，, Hive (Dynamo的补充支持), Pig, and Zookeeper

### Windows Azure的HDlnsight
HDlnsight基于Hortonworks数据平台（Hadoop1），运行在Azure云。它集成了微软管理控制台，易于部署，易于System Center的集成。通过使用Excel插件，可以整合Excel数据。通过Hive开放式数据库连接（ODBC）驱动程序，可以集成Microsoft SQL Server分析服务（SSAS）、PowerPivot和Power View。Azure Marketplace授权客户连接数据、智能挖掘算法以及防火墙之外的人。Windows Azure Marketplace从受信任的第三方供应商中，提供了数百个数据集。

### 商业发行版的选择
大量的发行版让你疑惑“我应该使用哪个发行版？”当公司/部门决定采用一个具体的版本时，应该考虑以下几点：
**技术细节**——包括Hadoop的版本、组件、专有功能部件等等。
**易于部署**——使用工具箱来实现管理的部署、版本升级、打补丁等等。
**易于维护**——主要包括集群管理、多中心的支持、灾难恢复支持等等。
**成本**——包括针发行版的实施成本、计费模式和许可证。
**企业集成的支持**——Hadoop应用程序与企业中其他部分的集成。

版本的选择依赖于，你打算利用Hadoop来解决哪些问题。
以下是本人对hadoop商业发行版选择的个人意见：
1、个人比较倾向Cloudera CDH：CDH公司在国内有服务商，Coudera Manager管理工具以及自带的一些组件确实比较不错。
2、其次是华为 Fusioninsight2：毕竟是国内的产品（技术服务比较方便）,而且华为这产品几个比较不错的亮点（数据加密、统一用户权限管理）。
3、Hortonworks HDP：100%开源，这是我第一次接触的发行版(还是有点感情的)。
4、星环 Transwarp：这是一家比较有潜力的公司，选择这家公司产品意味着以后会有很多意向不到的功能。
