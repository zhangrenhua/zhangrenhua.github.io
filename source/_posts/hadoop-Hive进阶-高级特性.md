---
title: Hive进阶-高级特性
description: 闲着无聊，趁着周末在家整理了一些Hive的高级特性，其中大多特性只有在`hive0.13`之后的版本才能享有。由于时间原因，本次笔者主要讲解hive特性有：insert、update、delete、zookeeper HA、thrift client、streaming write，在后面一段时间内我会陆续将hive udf/自定义SerDe等调优细节进行详细说明。
tags: [hadoop生态圈, hive]
categories: hive
date: 2016-1-31 12:05:18
---

## 背景

闲着无聊，趁着周末在家整理了一些Hive的高级特性，其中大多特性只有在`hive0.13`之后的版本才能享有。由于时间原因，本次笔者主要讲解hive特性有：insert、update、delete、zookeeper HA、thrift client、streaming write，在后面一段时间内我会陆续将hive udf/自定义SerDe等调优细节进行详细说明。

笔者使用hive1.1版本作为测试。

## Hive DML操作

新版建议使用beeline登录hive命令行：
```
beeline -u jdbc:hive2://localhost:10000/default
```

老的hive命令也可以用，但不建议使用。

### insert

登录到命令行后，创建test表，并执行2次insert语句：
```
create table test(name string);
insert into test values('zrh');
insert into test values('zrh');
```

此时，我们执行show table语法，发现有2张临时表：
```
hive> show tables;
OK
test
values__tmp__table__1
values__tmp__table__2
Time taken: 0.038 seconds, Fetched: 3 row(s)
```

查询临时表，发现就是insert语句中的值：
```
hive> select * from values__tmp__table__1;
OK
zrh
Time taken: 0.079 seconds, Fetched: 1 row(s)
```

再看HDFS目录下也出现2个数据文件：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hive-insert-test.png)

由此发现，这种insert插入语法基本只能做来测试使用，不建议用来插入数据，如想大批量写入数据可以参考后面的`streaming write`;

创建临时表语法：
```
create TEMPORARY table tmp_table(name string);
```

会话级别的表，重新连接后，该表则消失。

## update/delete

若想使用update和delete语法，必须设置下面几个参数（hive-site.xml）：
```
hive.support.concurrency=true
hive.enforce.bucketing=true
hive.exec.dynamic.partition.mode=nonstrict
hive.txn.manager=org.apache.hadoop.hive.ql.lockmgr.DbTxnManager
hive.compactor.initiator.on=true
hive.compactor.worker.threads=15
```

详细参数说明见：https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions

修改后重启hive。

目前只有ORCFileformat支持AcidOutputFormat，并且必须使用桶，因为ORC文件支持修改，所有才只能使用ORC格式：
```
create table test(id int ,name string )clustered by (id) into 2 buckets stored as orc TBLPROPERTIES('transactional'='true');

insert into test values(30,'zhangrenhua');
```

如不按上面说明建表，执行update或delete语句时则会报错：
```
Error: Error while compiling statement: FAILED: SemanticException [Error 10297]: Attempt to do update or delete on table default.test that does not use an AcidOutputFormat or is not bucketed (state=42000,code=10297)
```

update：
```
update test set name = 'zrh' where id = 30;
```

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hive-update-test.png)


delete：
```
delete from test where id = 30;
```

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hive-delete-test.png)

再看HDFS目录下的数据文件，发现多了一些标记文件：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hive-ddl-test.png)

从文件目录中可以看出，不管是update/delete其实都是增加一些标识文件，当然最重要的是依赖文件的存储格式，ORC支持update与修改。

## Zookeeper HA

修改hive-site.xml，添加如下参数：
```
hive.zookeeper.quorum=cdh-m1:2181,cdh-m2:2181
hive.zookeeper.session.timeout=60000
hive.server2.support.dynamic.service.discovery=true
hive.server2.zookeeper.namespace=hive_zookeeper_namespace_hive
hive.zookeeper.namespace=hive_zookeeper_namespace_hive
```

`xxx.namespace`：zookeeper中存储目录
`xxx.quorum`：zookeeper集群
`hive.server2.support.dynamic.service.discovery`：开启动态服务

重启hive服务，JDBC连接时需要注意：
```
Connection con = DriverManager.getConnection(
                "jdbc:hive2://cdh-m1:2181,cdh-m2:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive",
                "xxx", "xx");
```

此时连接地址修改成zookeeper集群地址逗号分隔。


## streaming write

同update/delete一样，建表格式有要求：
```
CREATE TABLE
    alerts
    (
        id INT ,
        msg string
    )
    partitioned BY
    (
        continent string,
        country string
    )
    clustered BY
    (
        id
    )
INTO
    5 buckets stored AS orc tblproperties
    (
        "transactional"="true"
    );
```

示例代码：
```
import java.util.ArrayList;

import org.apache.hive.hcatalog.streaming.DelimitedInputWriter;
import org.apache.hive.hcatalog.streaming.HiveEndPoint;
import org.apache.hive.hcatalog.streaming.StreamingConnection;
import org.apache.hive.hcatalog.streaming.TransactionBatch;

public class StreamWrite {

    public static void main(String[] args) throws Exception {
        // ------- MAIN THREAD ------- //
        String dbName = "default";
        String tblName = "alerts";

        // 分区值
        ArrayList<String> partitionVals = new ArrayList<String>(2);
        partitionVals.add("Asia");
        partitionVals.add("India");

        // 字段列表
        String[] fieldNames = new String[] { "id", "msg" };
        // String serdeClass =
        // "org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe";

        StreamingConnection connection = null;
        TransactionBatch txnBatch = null;

        try {

            HiveEndPoint hiveEP = new HiveEndPoint("thrift://cdh-s1:9083", dbName, tblName, partitionVals);
            connection = hiveEP.newConnection(true);
            DelimitedInputWriter writer = new DelimitedInputWriter(fieldNames, ",", hiveEP);
            txnBatch = connection.fetchTransactionBatch(10, writer);

            // 一个批次
            txnBatch.beginNextTransaction();
            txnBatch.write("1,Hello streaming".getBytes());
            txnBatch.write("2,Welcome to streaming".getBytes());
            txnBatch.commit();

            if (txnBatch.remainingTransactions() > 0) {
                // 一个批次
                txnBatch.beginNextTransaction();
                txnBatch.write("3,Roshan Naik".getBytes());
                txnBatch.write("4,Alan Gates".getBytes());
                txnBatch.write("5,Owen O’Malley".getBytes());
                txnBatch.commit();
            }

        } finally {
            if (txnBatch != null) {
                txnBatch.close();
            }
            if (connection != null) {
                connection.close();
            }
        }
    }
}
```

建议打包放到集群环境中运行，依赖了很多hadoop jar：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hive-stream-write-test.png)

大批量数据，如果没有使用Spark or MapReduce，可以使用此方法刘使写入。

## Thrift Client

示例代码：
```
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import javax.security.sasl.SaslException;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.apache.hive.service.auth.PlainSaslHelper;
import org.apache.hive.service.cli.FetchOrientation;
import org.apache.hive.service.cli.FetchType;
import org.apache.hive.service.cli.thrift.TCLIService;
import org.apache.hive.service.cli.thrift.TCloseOperationReq;
import org.apache.hive.service.cli.thrift.TExecuteStatementReq;
import org.apache.hive.service.cli.thrift.TExecuteStatementResp;
import org.apache.hive.service.cli.thrift.TFetchOrientation;
import org.apache.hive.service.cli.thrift.TFetchResultsReq;
import org.apache.hive.service.cli.thrift.TFetchResultsResp;
import org.apache.hive.service.cli.thrift.TOpenSessionReq;
import org.apache.hive.service.cli.thrift.TOperationHandle;
import org.apache.hive.service.cli.thrift.TProtocolVersion;
import org.apache.hive.service.cli.thrift.TRow;
import org.apache.hive.service.cli.thrift.TRowSet;
import org.apache.hive.service.cli.thrift.TSessionHandle;
import org.apache.thrift.TException;
import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.transport.TSocket;
import org.apache.thrift.transport.TTransport;
import org.apache.thrift.transport.TTransportException;

public class ThriftUtil {

    private static Log log = LogFactory.getLog(ThriftUtil.class);

    // 初始化一个与thrift服务端通信的会话
    public static TOpenSessionReq openSession(String user, String password, String database) {
        TOpenSessionReq openReq = new TOpenSessionReq();
        // 设置客户端服务版本
        openReq.setClient_protocol(TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V1);
        openReq.setUsername(user);
        openReq.setPassword(password);
        // 设置初始化进入的数据库
        Map<String, String> openConf = new HashMap<String, String>();
        openConf.put("use:database", database);
        openReq.setConfiguration(openConf);
        return openReq;
    }

    // 创建一个连接指定地址的thrift客户端
    /**
     * 
     * @param isNoSasl
     *            服务器sasl验证类型，true则为none类型，false则为nosasl类型
     * @param host
     *            主机
     * @param port
     *            端口
     * @param timeout
     *            超时时间
     * @return
     * @throws SaslException
     * @throws TTransportException
     */
    public static TCLIService.Client createCliHiveClient(boolean isNoSasl, String host, int port, int timeout)
            throws SaslException, TTransportException {
        TSocket transport = new TSocket(host, port, timeout);
        TBinaryProtocol pro = null;
        // 如果sasl验证类型是none类型
        if (isNoSasl) {
            TTransport saslPort = PlainSaslHelper.getPlainTransport("hadoop", "abcd1234", transport);
            pro = new TBinaryProtocol(saslPort);
            saslPort.open();
        } else {
            // 如果sasl验证类型是noSasl类型
            pro = new TBinaryProtocol(transport);
            transport.open();
        }
        TCLIService.Client client = new TCLIService.Client(pro);
        return client;
    }

    /**
     * 
     * @param client
     *            连接thriftserver的客户端
     * @param operationHandle
     *            当前操作对应句柄
     * @param Orientation
     *            获取结果的起始位置 FetchOrientation.First 从结果第一行开始取2
     * @param fetchType
     *            获取结果的类型 FetchType.LOG 获取此操作对应日志
     * @param maxRows
     *            此次获取的最大行数
     * @return
     */
    public static TRowSet fetchResult(TCLIService.Client client, TOperationHandle operationHandle,
            FetchOrientation Orientation, FetchType fetchType, long maxRows) {

        TFetchResultsReq fetchReq = new TFetchResultsReq();
        fetchReq.setFetchType(fetchType.toTFetchType());
        fetchReq.setMaxRows(maxRows);
        fetchReq.setOperationHandle(operationHandle);
        fetchReq.setOrientation(Orientation.toTFetchOrientation());
        TRowSet rowSetLog = null;
        try {
            rowSetLog = client.FetchResults(fetchReq).getResults();
        } catch (TException e) {
            log.info("Exception:" + e.getStackTrace());
            e.printStackTrace();
        }
        return rowSetLog;
    }

    /**
     * 
     * @describe 向thrift接口提交sql
     * @param client
     *            thrift客户端
     * @param session
     *            会话句柄
     * @param sql
     *            sql
     * @param isRunAsync
     *            此sql是否异步执行
     * @param conf
     *            此次请求中的特殊配置
     * @return
     * @throws TException
     */
    public static TExecuteStatementResp excuteSql(TCLIService.Client client, TSessionHandle session, String sql,
            boolean isRunAsync, Map<String, String> conf) throws TException {
        TExecuteStatementReq req = new TExecuteStatementReq();
        req.setSessionHandle(session);
        req.setStatement(sql);
        req.setConfOverlay(conf);
        req.setRunAsync(isRunAsync);
        TExecuteStatementResp resp = client.ExecuteStatement(req);
        return resp;
    }

    public static void main(String[] args) throws Exception {
        createCliHiveClient(true, "cdh-s1", 10000, 60);
    }

    public static void getResult(TOperationHandle op, TCLIService.Client client) throws TException {
        TFetchResultsReq freq = new TFetchResultsReq(op, TFetchOrientation.FETCH_FIRST, 1000);
        TFetchResultsResp fresp = client.FetchResults(freq);
        System.out.println(op.isHasResultSet());
        TRowSet set = fresp.getResults();
        System.out.println(set.toString());
        System.out.println(set.getRows());
        List<TRow> rows = set.getRows();
        for (TRow tRow : rows) {
            System.out.println(tRow.toString());
        }
        TCloseOperationReq closeReq = new TCloseOperationReq();
        closeReq.setOperationHandle(op);
        client.CloseOperation(closeReq);
    }

}

```

JDBC在thrift之上封装。

## 参考资料
- https://cwiki.apache.org/confluence/display/Hive/Streaming+Data+Ingest
- https://cwiki.apache.org/confluence/display/Hive/LanguageManual