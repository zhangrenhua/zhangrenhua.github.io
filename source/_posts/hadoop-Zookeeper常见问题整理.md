---
title: hadoop-Zookeeper常见问题整理
description: 最近测试环境发现因为zookeeper挂掉而导致hadoop集群服务都挂掉的情况，zookeeper是集群中不可或缺的组件，大部分的组件都依赖zookeeper，在此将zookeeper的一些概念性的东西整理出来，后面也会提及如何恢复zookeeper数据。
tags: [hadoop生态圈, zookeeper]
categories: zookeeper
date: 2016-6-14 13:52:14
---

## 背景

最近测试环境发现因为zookeeper挂掉而导致hadoop集群服务都挂掉的情况，zookeeper是集群中不可或缺的组件，大部分的组件都依赖zookeeper，在此将zookeeper的一些概念性的东西整理出来，后面也会提及如何恢复zookeeper数据。


## ZK选举过程

当leader崩溃或者leader失去大多数的follower，这时候zk进入恢复模式，恢复模式需要重新选举出一个新的leader，让所有的Server都恢复到一个正确的状态。Zk的选举算法使用ZAB协议：
选举线程由当前Server发起选举的线程担任，其主要功能是对投票结果进行统计，并选出推荐的Server；
选举线程首先向所有Server发起一次询问(包括自己)；
选举线程收到回复后，验证是否是自己发起的询问(验证zxid是否一致)，然后获取对方的id(myid)，并存储到当前询问对象列表中，最后获取对方提议的leader相关信息(id,zxid)，并将这些信息存储到当次选举的投票记录表中；
收到所有Server回复以后，就计算出zxid最大的那个Server，并将这个Server相关信息设置成下一次要投票的Server；
线程将当前zxid最大的Server设置为当前Server要推荐的Leader，如果此时获胜的Server获得n/2 + 1的Server票数， 设置当前推荐的leader为获胜的Server，将根据获胜的Server相关信息设置自己的状态，否则，继续这个过程，直到leader被选举出来。
通过流程分析我们可以得出：要使Leader获得多数Server的支持，则Server总数最好是奇数2n+1，且存活的Server的数目不得少于n+1

## master/slave之间通信

Storm：定期扫描 
PtBalancer：节点监听

## 节点变多时，PtBalancer速度变慢

类似问题：根据Netflix的Curator作者所说，ZooKeeper真心不适合做Queue，或者说ZK没有实现一个好的Queue，详细内容可以看 https://cwiki.apache.org/confluence/display/CURATOR/TN4， 
原因有五：
ZK有1MB 的传输限制。 实践中ZNode必须相对较小，而队列包含成千上万的消息，非常的大。
如果有很多节点，ZK启动时相当的慢。 而使用queue会导致好多ZNode. 你需要显著增大 initLimit 和 syncLimit.
ZNode很大的时候很难清理。Netflix不得不创建了一个专门的程序做这事。
当很大量的包含成千上万的子节点的ZNode时， ZK的性能变得不好
ZK的数据库完全放在内存中。 大量的Queue意味着会占用很多的内存空间。
尽管如此， Curator还是创建了各种Queue的实现。 如果Queue的数据量不太多，数据量不太大的情况下，酌情考虑，还是可以使用的。

## 客户端对ServerList的轮询机制是什么

随机，客户端在初始化( new ZooKeeper(String connectString, int sessionTimeout, Watcher watcher) )的过程中，将所有Server保存在一个List中，然后随机打散，形成一个环。之后从0号位开始一个一个使用。 
两个注意点：
Server地址能够重复配置，这样能够弥补客户端无法设置Server权重的缺陷，但是也会加大风险。（比如: 192.168.1.1:2181,192.168.1.1:2181,192.168.1.2:2181).
如果客户端在进行Server切换过程中耗时过长，那么将会收到SESSION_EXPIRED. 这也是上面第1点中的加大风险之处。更多关于客户端地址列表相关的。

## 客户端如何正确处理CONNECTIONLOSS(连接断开) 和 SESSIONEXPIRED(Session 过期)两类连接异常

在ZooKeeper中，服务器和客户端之间维持的是一个长连接，在 SESSION_TIMEOUT 时间内，服务器会确定客户端是否正常连接(客户端会定时向服务器发送heart_beat),服务器重置下次SESSION_TIMEOUT时间。因此，在正常情况下，Session一直有效，并且zk集群所有机器上都保存这个Session信息。在出现问题情况下，客户端与服务器之间连接断了（客户端所连接的那台zk机器挂了，或是其它原因的网络闪断），这个时候客户端会主动在地址列表（初始化的时候传入构造方法的那个参数connectString）中选择新的地址进行连接。
好了，上面基本就是服务器与客户端之间维持长连接的过程了。在这个过程中，用户可能会看到两类异常CONNECTIONLOSS(连接断开) 和SESSIONEXPIRED(Session 过期)。
CONNECTIONLOSS发生在上面红色文字部分，应用在进行操作A时，发生了CONNECTIONLOSS，此时用户不需要关心我的会话是否可用，应用所要做的就是等待客户端帮我们自动连接上新的zk机器，一旦成功连接上新的zk机器后，确认刚刚的操作A是否执行成功了。

## 一个客户端修改了某个节点的数据，其它客户端能够马上获取到这个最新数据吗

ZooKeeper不能确保任何客户端能够获取（即Read Request）到一样的数据，除非客户端自己要求：方法是客户端在获取数据之前调用org.apache.zookeeper.AsyncCallback.VoidCallback, java.lang.Object) sync. 
通常情况下（这里所说的通常情况满足：1. 对获取的数据是否是最新版本不敏感，2. 一个客户端修改了数据，其它客户端需要不需要立即能够获取最新），可以不关心这点。 
在其它情况下，最清晰的场景是这样：ZK客户端A对 /my_test 的内容从 v1->v2, 但是ZK客户端B对 /my_test 的内容获取，依然得到的是 v1. 请注意，这个是实际存在的现象，当然延时很短。解决的方法是客户端B先调用 sync(), 再调用 getData().

## ZK为什么不提供一个永久性的Watcher注册机制

不支持用持久Watcher的原因很简单，ZK无法保证性能。 
使用watch需要注意的几点
Watches通知是一次性的，必须重复注册.
发生CONNECTIONLOSS之后，只要在session_timeout之内再次连接上（即不发生SESSIONEXPIRED），那么这个连接注册的watches依然在。
节点数据的版本变化会触发NodeDataChanged，注意，这里特意说明了是版本变化。存在这样的情况，只要成功执行了setData()方法，无论内容是否和之前一致，都会触发NodeDataChanged。
对某个节点注册了watch，但是节点被删除了，那么注册在这个节点上的watches都会被移除。
同一个zk客户端对某一个节点注册相同的watch，只会收到一次通知。即
Watcher对象只会保存在客户端，不会传递到服务端。

## 我能否收到每次节点变化的通知

如果节点数据的更新频率很高的话，不能。 
原因在于：当一次数据修改，通知客户端，客户端再次注册watch，在这个过程中，可能数据已经发生了许多次数据修改，因此，千万不要做这样的测试：”数据被修改了n次，一定会收到n次通知”来测试server是否正常工作。（我曾经就做过这样的傻事，发现Server一直工作不正常？其实不是）。即使你使用了GitHub上这个客户端也一样。

## 能为临时节点创建子节点吗

不能。

## 是否可以拒绝单个IP对ZK的访问,操作

ZK本身不提供这样的功能，它仅仅提供了对单个IP的连接数的限制。你可以通过修改iptables来实现对单个ip的限制，当然，你也可以通过这样的方式来解决。 https://issues.apache.org/jira/browse/ZOOKEEPER-1320

## 在getChildren(String path, boolean watch)是注册了对节点子节点的变化，那么子节点的子节点变化能通知吗

不能

## 创建的临时节点什么时候会被删除，是连接一断就删除吗？延时是多少？

连接断了之后，ZK不会马上移除临时数据，只有当SESSIONEXPIRED之后，才会把这个会话建立的临时数据移除。因此，用户需要谨慎设置Session_TimeOut

## zookeeper是否支持动态进行机器扩容？如果目前不支持，那么要如何扩容呢？

截止3.4.3版本的zookeeper，还不支持这个功能，在3.5.0版本开始，支持动态加机器了，期待下吧: https://issues.apache.org/jira/browse/ZOOKEEPER-107

## ZooKeeper集群中个服务器之间是怎样通信的？

Leader服务器会和每一个Follower/Observer服务器都建立TCP连接，同时为每个F/O都创建一个叫做LearnerHandler的实体。LearnerHandler主要负责Leader和F/O之间的网络通讯，包括数据同步，请求转发和Proposal提议的投票等。Leader服务器保存了所有F/O的LearnerHandler。

## zookeeper是否会自动进行日志清理？如何进行日志清理？

zk自己不会进行日志清理，需要运维人员进行日志清理


## 数据文件管理

默认情况下，ZK的数据文件和事务日志是保存在同一个目录中，建议是将事务日志存储到单独的磁盘上。
1 数据目录
ZK的数据目录包含两类文件：
    A、myid – 这个文件只包含一个数字，和serverid对应。
    B、snapshot. - 按zxid先后顺序的生成的数据快照。
集群中的每台ZK server都会有一个用于惟一标识自己的id，有两个地方会使用到这个id：myid文件和zoo.cfg文件中。myid文件存储在dataDir目录中，指定了当前server的server id。在zoo.cfg文件中，根据server id，配置了每个server的ip和相应端口。Zookeeper启动的时候，读取myid文件中的server id，然后去zoo.cfg 中查找对应的配置。
zookeeper在进行数据快照过程中，会生成snapshot文件，存储在dataDir目录中。文件后缀是zxid，也就是事务id。（这个zxid代表了zk触发快照那个瞬间，提交的最后一个事务id）。注意，一个快照文件中的数据内容和提交第zxid个事务时内存中数据近似相同。仅管如此，由于更新操作的幂等性，ZK还是能够从快照文件中恢复数据。数据恢复过程中，将事务日志和快照文件中的数据对应起来，就能够恢复最后一次更新后的数据了。
2 事务日志目录
dataLogDir目录是ZK的事务日志目录，包含了所有ZK的事务日志。正常运行过程中，针对所有更新操作，在返回客户端“更新成功”的响应前，ZK会确保已经将本次更新操作的事务日志写到磁盘上，只有这样，整个更新操作才会生效。每触发一次数据快照，就会生成一个新的事务日志。事务日志的文件名是log.，zxid是写入这个文件的第一个事务id。
3 文件管理
不同的zookeeper server生成的snapshot文件和事务日志文件的格式都是一致的（无论是什么环境，或是什么样的zoo.cfg 配置）。因此，如果某一天生产环境中出现一些古怪的问题，你就可以把这些文件下载到开发环境的zookeeper中加载起来，便于调试发现问题，而不会影响生产运行。另外，使用这些较旧的snapshot和事务日志，我们还能够方便的让ZK回滚到一个历史状态。
另外，ZK提供的工具类LogFormatter能够帮助可视化ZK的事务日志，帮助我们排查问题，关于事务日志的可以化，请查看这个文章《可视化zookeeper的事务日志》.
需要注意的一点是，zookeeper在运行过程中，不断地生成snapshot文件和事务日志，但是不会自动清理它们，需要管理员来处理。（ZK本身只需要使用最新的snapshot和事务日志即可）关于如何清理文件，上面章节“日常运维”有提到。

### 通过数据文件和事物日志文件恢复zookeeper

选择数据文件和事物文件，并复制到新的zookeeper服务数据目录下(datadir)：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/zookeeper-sel-log-demo.png)

启动新的zookeeper服务：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/zookeeper-start-demo.png)

查看zookeeper数据：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/zookeeper-ls-demo.png)

从图中可以看出数据已经恢复。


**注意**
如果集群中有一台zookeeper服务能正常启动那就直接将其它的zookeeper服务重置即可（记得要备份数据目录哦）。

### 日志清理

在使用zookeeper过程中，我们知道，会有dataDir和dataLogDir两个目录，分别用于snapshot和事务日志的输出（默认情况下只有dataDir目录，snapshot和事务日志都保存在这个目录中）。

ZK在完成若干次事务日志之后（在ZK中，凡是对数据有更新的操作，比如创建节点，删除节点或是对节点数据内容进行更新等，都会记录事务日志），ZK会触发一次快照（snapshot），将当前server上所有节点的状态以快照文件的形式dump到磁盘上去，即snapshot文件。这里的若干次事务日志是可以配置的，默认是100000，具体参看下文中关于配置参数“snapCount”的介绍。

考虑到ZK运行环境的差异性，以及对于这些历史文件，不同的管理员可能有自己的用途（例如作为数据备份），因此默认ZK是不会自动清理快照和事务日志，需要交给管理员自己来处理。

这里介绍4种清理日志的方法。在这4种方法中，推荐使用第一种方法，对于运维人员来说，将日志清理工作独立出来，便于统一管理也更可控。毕竟zk自带的一些工具并不怎么给力，这里是社区反映的两个问题：
https://issues.apache.org/jira/browse/ZOOKEEPER-957
http://zookeeper-user.578899.n2.nabble.com/PurgeTxnLog-td6304244.html

第一种，也是运维人员最常用的，写一个删除日志脚本，每天定时执行即可：

```
#!/bin/bash
           
#snapshot file dir
dataDir=/home/nileader/taokeeper/zk_data/version-2
#tran log dir
dataLogDir=/home/nileader/taokeeper/zk_log/version-2
#zk log dir
logDir=/home/nileader/taokeeper/logs
#Leave 60 files
count=60
count=$[$count+1]
ls -t $dataLogDir/log.* | tail -n +$count | xargs rm -f
ls -t $dataDir/snapshot.* | tail -n +$count | xargs rm -f
ls -t $logDir/zookeeper.log.* | tail -n +$count | xargs rm -f
```

以上这个脚本定义了删除对应两个目录中的文件，保留最新的60个文件，可以将他写到crontab中，设置为每天凌晨2点执行一次就可以了。

第二种，使用ZK的工具类PurgeTxnLog，它的实现了一种简单的历史文件清理策略，可以在这里看一下他的使用方法：http://zookeeper.apache.org/doc/r3.4.3/api/index.html，可以指定要清理的目录和需要保留的文件数目，简单使用如下：

```
java -cp zookeeper.jar:lib/slf4j-api-1.6.1.jar:lib/slf4j-log4j12-1.6.1.jar:lib/log4j-1.2.15.jar:conf org.apache.zookeeper.server.PurgeTxnLog <dataDir><snapDir> -n <count>
```

第三种，对于上面这个Java类的执行，ZK自己已经写好了脚本，在bin/zkCleanup.sh中，所以直接使用这个脚本也是可以执行清理工作的。
第四种，从3.4.0开始，zookeeper提供了自动清理snapshot和事务日志的功能，通过配置 autopurge.snapRetainCount 和 autopurge.purgeInterval 这两个参数能够实现定时清理了。这两个参数都是在zoo.cfg中配置的：
autopurge.purgeInterval  这个参数指定了清理频率，单位是小时，需要填写一个1或更大的整数，默认是0，表示不开启自己清理功能。
autopurge.snapRetainCount 这个参数和上面的参数搭配使用，这个参数指定了需要保留的文件数目。默认是保留3个。


