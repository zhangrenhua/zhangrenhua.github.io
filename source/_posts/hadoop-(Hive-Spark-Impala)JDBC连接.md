---
title: Hive、Spark、Impala使用JDBC连接
description: 在进行Hive、Spark Sql、Impala开发中，我们肯定是需要用到它们的JDBC接口的。在我使用了这3种JDBC接口后发现存在一个共同点，几乎可以说不需要改动代码就可以将连接转换成其它的运行驱动（Spark Sql、Impala、Hive）。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-12-14 11:10:00
---

## 背景

在进行Hive、Spark Sql、Impala开发中，我们肯定是需要用到它们的JDBC接口的。在我使用了这3种JDBC接口后发现存在一个共同点，几乎可以说不需要改动代码就可以将连接转换成其它的运行驱动（Spark Sql、Impala、Hive）。

##  Jar包准备
```
commons-configuration-1.6.jar
commons-logging-1.2.jar
hadoop-common.jar
hive-exec.jar
hive-jdbc.jar
hive-metastore.jar
hive-service.jar
httpclient-4.2.5.jar
httpcore-4.2.5.jar
log4j-1.2.17.jar
slf4j-api-1.7.5.jar
slf4j-log4j12.jar
```

**笔者基于hive0.14作为测试，如出现ClassNotFound这类似的异常，直接找到hive/lib目录下的对应jar文件既可**。

## 代码示例

```
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class Demo {

    public static void main(String[] args) throws Exception {

        Class.forName("org.apache.hive.jdbc.HiveDriver");
        runHive();
        runSparkSql();
        runImpala();

    }

    private static void runHive() throws Exception {
        Connection con = DriverManager.getConnection("jdbc:hive2://hynamet01:10000/default");
        Statement stmt = con.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT * FROM test");

        print(rs);
    }

    private static void runImpala() throws Exception {
        Connection con = DriverManager.getConnection("jdbc:hive2://hydatat01:21050/;auth=noSasl");
        Statement stmt = con.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT * FROM test");

        print(rs);
    }

    private static void runSparkSql() throws Exception {
        Connection con = DriverManager.getConnection("jdbc:hive2://hynamet02:10000/default");
        Statement stmt = con.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT * FROM test");

        print(rs);
    }

    private static void print(ResultSet rs) throws SQLException {
        // print the results to the console
        while (rs.next()) {
            System.out.println(rs.getString(1));
        }
    }

}
```

使用Hive连接JDBC，需要确认Hive的版本号。低版本HiveServer为（Hive1 Server）高版本的HiveServer为（Hive2 Server），上面是基于Hive2 Server编写的，Hive1 Server只需要将驱动类修改即可：
```
Class.forName("org.apache.hadoop.hive.jdbc.HiveDriver");  
```

注：这三种JDBC连接方式的区别在与jdbc连接url的不同，将上面jar、ip、端口、数据库修改成你的就可以运行。

**如果Hadoop集群开启了Kerberos or Sentry权限验证，则需要将对应的用户名密码以及对应的url参数进行修改**。