---
title: Spark DataFrme操作Hive
description: 从spark1.2 到spark1.3，spark SQL中的SchemaRDD变为了DataFrame，DataFrame相对于SchemaRDD有了较大改变，同时提供了更多好用且方便的API。<br>本文主要演示如何在spark1.5.2中使用DataFrame将数据写入hive中以及DataFrame的一些其他API，仅供参考。
tags: [hadoop生态圈, java]
categories: spark
date: 2015-11-28 20:10:00
---

## 背景
从spark1.3起，spark SQL中的SchemaRDD变为了DataFrame，DataFrame相对于SchemaRDD有了较大改变，同时提供了更多好用且方便的API。

本文主要演示如何在spark1.5.2中使用DataFrame将数据写入hive中以及DataFrame的一些其他API，仅供参考。

## DataFrame SaveAsTable

### 示例
新建数据文件test.txt，并上次至HDFS的/test/zhangrenhua目录下:
```
zrh,19
z,20
r,21
h,90
```

创建hive表：
```
CREATE DATABASE test;
DROP TABLE
    test.test;
CREATE TABLE
    test.test
    (
        t_string string,
        t_integer INT,
        t_boolean BOOLEAN,
        t_double DOUBLE,
        t_decimal DECIMAL(20,8)
    )
    stored AS orc;
```


Demo.java：
```
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

import org.apache.spark.SparkConf;
import org.apache.spark.SparkContext;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.sql.DataFrame;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SaveMode;
import org.apache.spark.sql.hive.HiveContext;
import org.apache.spark.sql.types.DataTypes;
import org.apache.spark.sql.types.Decimal;
import org.apache.spark.sql.types.StructField;
import org.apache.spark.sql.types.StructType;

/**
 * spark程序入口
 */
public class Demo {

    public static void main(String[] args) {

        String input = "/test/zhangrenhua";
        SparkConf sparkConf = new SparkConf().setAppName("Demo");
        SparkContext sc = new SparkContext(sparkConf);
        HiveContext hiveContext = new HiveContext(sc);
        hiveContext.setConf("spark.sql.hive.metastore.version", "0.13.0.2.1");

        // 创建spark Context
        try (JavaSparkContext ctx = new JavaSparkContext(sc);) {
            // 读取测试文件
            JavaRDD<String> textFile = ctx.textFile(input);
            JavaRDD<Row> map = textFile.map(new Function<String, Row>() {

                /**
                 * 
                 */
                private static final long serialVersionUID = 8745604304589989962L;

                @Override
                public Row call(String v1) throws Exception {

                    // 解析测试数据，并setting默认值
                    String[] array = v1.split(",");
                    TestBean result = new TestBean();
                    result.setT_string(array[0]);
                    result.setT_integer(Integer.parseInt(array[1]));
                    result.setT_boolean(true);
                    result.setT_double(12.12);
                    Decimal t_decimal = new Decimal();
                    t_decimal.set(new scala.math.BigDecimal(new BigDecimal("11111111.11111111")));
                    result.setT_decimal(t_decimal);

                    // 不能使用hiveContext.createDataFrame(map, TestBean.class);
                    // 字段顺序不一致，不知道是bug还是什么。所以只能选择row
                    return result.toRow();
                }
            });

            // Generate the schema based on the string of schema
            List<StructField> fields = new ArrayList<StructField>();
            fields.add(DataTypes.createStructField("t_string", DataTypes.StringType, true));
            fields.add(DataTypes.createStructField("t_integer", DataTypes.IntegerType, true));
            fields.add(DataTypes.createStructField("t_boolean", DataTypes.BooleanType, true));
            fields.add(DataTypes.createStructField("t_double", DataTypes.DoubleType, true));
            fields.add(DataTypes.createStructField("t_decimal", DataTypes.createDecimalType(20, 8), true));

            StructType schema = DataTypes.createStructType(fields);

            DataFrame df = hiveContext.createDataFrame(map, schema);
            df.write().format("orc").mode(SaveMode.Append).saveAsTable("test.test");
        }
    }

}
```

TestBean.java：
```
import java.io.Serializable;

import org.apache.spark.sql.Row;
import org.apache.spark.sql.RowFactory;
import org.apache.spark.sql.types.Decimal;

public class TestBean implements Serializable {

    /**
     * 
     */
    private static final long serialVersionUID = -5868257956951746438L;
    private String t_string;
    private Integer t_integer;
    private Boolean t_boolean;
    private Double t_double;
    private Decimal t_decimal;

    public String getT_string() {
        return t_string;
    }

    public void setT_string(String t_string) {
        this.t_string = t_string;
    }

    public Integer getT_integer() {
        return t_integer;
    }

    public void setT_integer(Integer t_integer) {
        this.t_integer = t_integer;
    }

    public Boolean getT_boolean() {
        return t_boolean;
    }

    public void setT_boolean(Boolean t_boolean) {
        this.t_boolean = t_boolean;
    }

    public Double getT_double() {
        return t_double;
    }

    public void setT_double(Double t_double) {
        this.t_double = t_double;
    }

    public Decimal getT_decimal() {
        return t_decimal;
    }

    public void setT_decimal(Decimal t_decimal) {
        this.t_decimal = t_decimal;
    }

    public Row toRow() {
        return RowFactory.create(t_string, t_integer, t_boolean, t_double, t_decimal);
    }
}

```

根据以上代码，编译打包即刻使用spark-submit命令执行：
```
spark-submit --master yarn-client --class Demo xx.jar
```

数据查询：</br>
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/spark-hive-insert-query.png)

### 要点记录

1、根据官方例子中使用hiveContext.createDataFrame(map, TestBean.class);是可以将对象转换成DataFrame的，<font color="red">但是我在测试中发现如果使用这种方式创建DataFrame，最终存到hive表中的数据字段是不对的。</font>，所以还是自定义schema，返回Row对象。

```
RowFactory.create(t_string, t_integer, t_boolean, t_double, t_decimal);

// Generate the schema based on the string of schema
List<StructField> fields = new ArrayList<StructField>();
fields.add(DataTypes.createStructField("t_string", DataTypes.StringType, true));
fields.add(DataTypes.createStructField("t_integer", DataTypes.IntegerType, true));
fields.add(DataTypes.createStructField("t_boolean", DataTypes.BooleanType, true));
fields.add(DataTypes.createStructField("t_double", DataTypes.DoubleType, true));
fields.add(DataTypes.createStructField("t_decimal", DataTypes.createDecimalType(20, 8), true)); 
```
<font color="red">创建List<StructField> fields和Row对象时，传入的字段顺序一定要和表中的顺序保持一致。</font>

2、将DataFrame存入自定义数据库的表中
```
df.saveAsTable("test.test");
```
从spark1.5.1开始支持显示指定数据库名，在spark1.5.1之前的版本需要使用下面方法：
```
hiveContext.sql("use test");
```


## DataFrame Save As 分区表
上面演示了如何将常用数据类型（string、integer、boolean、double、decimal）的DataFrame存入到hive表中。但是没有描述如何将数据存入<font color="red">分区表</font>中，下面我会给出如何写入分区表中的具体逻辑。

1、动态分区
```
// 设置动态分区
hiveContext.sql("set hive.exec.dynamic.partition=true");
hiveContext.sql("set hive.exec.dynamic.partition.mode=nonstrict");
```

创建Row对象时，**将分区字段放到最后**：
```
RowFactory.create(t_string, t_integer, t_boolean, t_double, t_decimal, '分区字段');
```
df.saveAsTable("test.test");

使用动态分区会影响写入性能，如果分区字段是可以固定的，则建议使用下面 **固定分区**方法。

2、固定分区
```
df.registerTempTable("demo");
hiveContext.sql("insert into test.test partition(date='20151205') select * from demo");
```
注：date为分区字段，20151205为分区值，**可由参数传入**。

## Metadata Refreshing
Spark SQL caches Parquet metadata for better performance. When Hive metastore Parquet table conversion is enabled, metadata of those converted tables are also cached. If these tables are updated by Hive or other external tools, you need to refresh them manually to ensure consistent metadata.
```
// sqlContext is an existing HiveContext
sqlContext.refreshTable("my_table")
```

## DataFrame Operations
DataFrame提供一个结构化的数据，下面是使用Java操作DataFrame的一些基本例子：
```
JavaSparkContext sc // An existing SparkContext.
SQLContext sqlContext = new org.apache.spark.sql.SQLContext(sc)

// Create the DataFrame
DataFrame df = sqlContext.read().json("examples/src/main/resources/people.json");

// Show the content of the DataFrame
df.show();
// age  name
// null Michael
// 30   Andy
// 19   Justin

// Print the schema in a tree format
df.printSchema();
// root
// |-- age: long (nullable = true)
// |-- name: string (nullable = true)

// Select only the "name" column
df.select("name").show();
// name
// Michael
// Andy
// Justin

// Select everybody, but increment the age by 1
df.select(df.col("name"), df.col("age").plus(1)).show();
// name    (age + 1)
// Michael null
// Andy    31
// Justin  20

// Select people older than 21
df.filter(df.col("age").gt(21)).show();
// age name
// 30  Andy

// Count people by age
df.groupBy("age").count().show();
// age  count
// null 1
// 19   1
// 30   1
```

## Parquet Configuration
Configuration of Parquet can be done using the `setConf` method on `SQLContext` or by running
`SET key=value` commands using SQL.

<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.sql.parquet.binaryAsString</code></td>
  <td>false</td>
  <td>
    Some other Parquet-producing systems, in particular Impala, Hive, and older versions of Spark SQL, do
    not differentiate between binary data and strings when writing out the Parquet schema. This
    flag tells Spark SQL to interpret binary data as a string to provide compatibility with these systems.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.int96AsTimestamp</code></td>
  <td>true</td>
  <td>
    Some Parquet-producing systems, in particular Impala and Hive, store Timestamp into INT96.  This
    flag tells Spark SQL to interpret INT96 data as a timestamp to provide compatibility with these systems.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.cacheMetadata</code></td>
  <td>true</td>
  <td>
    Turns on caching of Parquet schema metadata. Can speed up querying of static data.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.compression.codec</code></td>
  <td>gzip</td>
  <td>
    Sets the compression codec use when writing Parquet files. Acceptable values include:
    uncompressed, snappy, gzip, lzo.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.filterPushdown</code></td>
  <td>true</td>
  <td>Enables Parquet filter push-down optimization when set to true.</td>
</tr>
<tr>
  <td><code>spark.sql.hive.convertMetastoreParquet</code></td>
  <td>true</td>
  <td>
    When set to false, Spark SQL will use the Hive SerDe for parquet tables instead of the built in
    support.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.output.committer.class</code></td>
  <td><code>org.apache.parquet.hadoop.<br />ParquetOutputCommitter</code></td>
  <td>
    <p>
      The output committer class used by Parquet. The specified class needs to be a subclass of
      <code>org.apache.hadoop.<br />mapreduce.OutputCommitter</code>.  Typically, it's also a
      subclass of <code>org.apache.parquet.hadoop.ParquetOutputCommitter</code>.
    </p>
    <p>
      <b>Note:</b>
      <ul>
        <li>
          This option is automatically ignored if <code>spark.speculation</code> is turned on.
        </li>
        <li>
          This option must be set via Hadoop <code>Configuration</code> rather than Spark
          <code>SQLConf</code>.
        </li>
        <li>
          This option overrides <code>spark.sql.sources.<br />outputCommitterClass</code>.
        </li>
      </ul>
    </p>
    <p>
      Spark SQL comes with a builtin
      <code>org.apache.spark.sql.<br />parquet.DirectParquetOutputCommitter</code>, which can be more
      efficient then the default Parquet output committer when writing data to S3.
    </p>
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.mergeSchema</code></td>
  <td><code>false</code></td>
  <td>
    <p>
      When true, the Parquet data source merges schemas collected from all data files, otherwise the
      schema is picked from the summary file or a random data file if no summary file is available.
    </p>
  </td>
</tr>
</table>

## Interacting with Different Versions of Hive Metastore
One of the most important pieces of Spark SQL’s Hive support is interaction with Hive metastore, which enables Spark SQL to access metadata of Hive tables. Starting from Spark 1.4.0, a single binary build of Spark SQL can be used to query different versions of Hive metastores, using the configuration described below. Note that independent of the version of Hive that is being used to talk to the metastore, internally Spark SQL will compile against Hive 1.2.1 and use those classes for internal execution (serdes, UDFs, UDAFs, etc).

The following options can be used to configure the version of Hive that is used to retrieve metadata:

<table class="table">
  <tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
  <tr>
    <td><code>spark.sql.hive.metastore.version</code></td>
    <td><code>1.2.1</code></td>
    <td>
      Version of the Hive metastore. Available
      options are <code>0.12.0</code> through <code>1.2.1</code>.
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.hive.metastore.jars</code></td>
    <td><code>builtin</code></td>
    <td>
      Location of the jars that should be used to instantiate the HiveMetastoreClient. This
      property can be one of three options:
      <ol>
        <li><code>builtin</code></li>
        Use Hive 1.2.1, which is bundled with the Spark assembly jar when <code>-Phive</code> is
        enabled. When this option is chosen, <code>spark.sql.hive.metastore.version</code> must be
        either <code>1.2.1</code> or not defined.
        <li><code>maven</code></li>
        Use Hive jars of specified version downloaded from Maven repositories.  This configuration
        is not generally recommended for production deployments. 
        <li>A classpath in the standard format for the JVM.  This classpath must include all of Hive 
        and its dependencies, including the correct version of Hadoop.  These jars only need to be
        present on the driver, but if you are running in yarn cluster mode then you must ensure
        they are packaged with you application.</li>
      </ol>
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.hive.metastore.sharedPrefixes</code></td>
    <td><code>com.mysql.jdbc,<br/>org.postgresql,<br/>com.microsoft.sqlserver,<br/>oracle.jdbc</code></td>
    <td>
      <p>
        A comma separated list of class prefixes that should be loaded using the classloader that is
        shared between Spark SQL and a specific version of Hive. An example of classes that should
        be shared is JDBC drivers that are needed to talk to the metastore. Other classes that need
        to be shared are those that interact with classes that are already shared. For example,
        custom appenders that are used by log4j.
      </p>
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.hive.metastore.barrierPrefixes</code></td>
    <td><code>(empty)</code></td>
    <td>
      <p>
        A comma separated list of class prefixes that should explicitly be reloaded for each version
        of Hive that Spark SQL is communicating with. For example, Hive UDFs that are declared in a
        prefix that typically would be shared (i.e. <code>org.apache.spark.*</code>).
      </p>
    </td>
  </tr>
</table>

## JDBC To Other Databases
Spark SQL also includes a data source that can read data from other databases using JDBC. This functionality should be preferred over using JdbcRDD. This is because the results are returned as a DataFrame and they can easily be processed in Spark SQL or joined with other data sources. The JDBC data source is also easier to use from Java or Python as it does not require the user to provide a ClassTag. (Note that this is different than the Spark SQL JDBC server, which allows other applications to run queries using Spark SQL).

To get started you will need to include the JDBC driver for you particular database on the spark classpath. For example, to connect to postgres from the Spark Shell you would run the following command:

```
SPARK_CLASSPATH=postgresql-9.3-1102-jdbc41.jar bin/spark-shell
```

Tables from the remote database can be loaded as a DataFrame or Spark SQL Temporary table using the Data Sources API. The following options are supported:

<table class="table">
  <tr><th>Property Name</th><th>Meaning</th></tr>
  <tr>
    <td><code>url</code></td>
    <td>
      The JDBC URL to connect to.
    </td>
  </tr>
  <tr>
    <td><code>dbtable</code></td>
    <td>
      The JDBC table that should be read.  Note that anything that is valid in a <code>FROM</code> clause of
      a SQL query can be used.  For example, instead of a full table you could also use a
      subquery in parentheses.
    </td>
  </tr>

  <tr>
    <td><code>driver</code></td>
    <td>
      The class name of the JDBC driver needed to connect to this URL.  This class will be loaded
      on the master and workers before running an JDBC commands to allow the driver to
      register itself with the JDBC subsystem.
    </td>
  </tr>
  <tr>
    <td><code>partitionColumn, lowerBound, upperBound, numPartitions</code></td>
    <td>
      These options must all be specified if any of them is specified.  They describe how to
      partition the table when reading in parallel from multiple workers.
      <code>partitionColumn</code> must be a numeric column from the table in question. Notice
      that <code>lowerBound</code> and <code>upperBound</code> are just used to decide the
      partition stride, not for filtering the rows in table. So all rows in the table will be
      partitioned and returned.
    </td>
  </tr>
</table>

示例：
```
Map<String, String> options = new HashMap<String, String>();
options.put("url", "jdbc:postgresql:dbserver");
options.put("dbtable", "schema.tablename");

DataFrame jdbcDF = sqlContext.read().format("jdbc"). options(options).load();
```

## 参考资料
- http://spark.apache.org/docs/latest/sql-programming-guide.html#interacting-with-different-versions-of-hive-metastore