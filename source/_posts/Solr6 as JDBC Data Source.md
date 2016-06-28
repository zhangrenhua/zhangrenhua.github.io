---
title: Solr6 as JDBC Data Source
description: 这里主要讲解solr jdbc如何使用，solr概念和安装请参考<a href="/2016/06/17/Solr6.0.1概念和集群部署/">Solr6.0.1概念和集群部署</a>。<br/>在这篇文章中我将展示如何使用Solr的JDBC驱动程序，Solr可以在Apache Zeppelin或任何其他数据探索和可视化工具中的JDBC支持。
tags: [hadoop生态圈, solr]
categories: solr
date: 2016-6-18 09:39:55
---

## 简介

这里主要讲解solr jdbc如何使用，solr概念和安装请参考[Solr6.0.1概念和集群部署](/2016/06/17/Solr6.0.1概念和集群部署/)。

在这篇文章中我将展示如何使用Solr的JDBC驱动程序，Solr可以在Apache Zeppelin或任何其他数据探索和可视化工具中的JDBC支持。


我们将开始建立一个非常简单的Maven项目将包括所有需要的库：
```
<dependency>
    <groupId>org.apache.solr</groupId>
    <artifactId>solr-solrj</artifactId>
    <version>6.0.1</version>
</dependency>
```


示例代码：
```
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.Properties;

public class AppTest {

    public static void main(String[] args) {
        Connection connection = null;
        Statement statement = null;
        ResultSet resultSet = null;
        try {
            Properties props = new Properties();
            props.put("aggregationMode", "map_reduce");
            props.put("numWorkers", "1");

            // define connection string - point to ZooKeeper and collection
            String connectionString = "jdbc:solr://192.168.31.135:2181/solr?collection=solr";
            // get the connection
            connection = DriverManager.getConnection(connectionString, props);
            // create statement
            statement = connection.createStatement();
            // run a simple query
            resultSet = statement.executeQuery("SELECT id,name FROM solr order by id");
            // iterate through the results
            while (resultSet.next()) {
                // read two fields
                String id = resultSet.getString("id");
                String name = resultSet.getString("name");
                // print out results
                System.out.println(id + ": " + name);
            }
        } catch (SQLException sqlEx) {
            System.out.println("Error while running SQL query to Solr: " + sqlEx.getMessage());
        } finally {
            // close the objects
            if (resultSet != null) {
                try {
                    resultSet.close();
                } catch (Exception ex) {
                }
            }
            if (statement != null) {
                try {
                    statement.close();
                } catch (Exception ex) {
                }
            }
            if (connection != null) {
                try {
                    connection.close();
                } catch (Exception ex) {
                }
            }
        }
    }
}
```


注意：如果查询字段不是DocValues格式，则会报错：`java.io.IOException: id must have DocValues to use this feature.`
所以现在jdbc驱动还是有限制的。
