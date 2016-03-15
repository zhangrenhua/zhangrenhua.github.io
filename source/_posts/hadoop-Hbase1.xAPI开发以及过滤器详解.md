---
title: Hbase1.0API开发以及过滤器详解
description: 在 2015 年，HBase 迎来了一个里程碑——HBase 1.0 release，这也代表着 HBase 走向了稳定。本文主要针对Hbase1.0 API进行讲解，并对原有的过滤器进行详细解释。
tags: [hadoop生态圈, hbase]
categories: hbase
date: 2016-1-15 21:31:14
---

## 背景

在 2015 年，HBase 迎来了一个里程碑——HBase 1.0 release，这也代表着 HBase 走向了稳定。本文主要针对Hbase1.0 API进行讲解，并对原有的过滤器进行详细解释。


## 常用表操作

### lib引用

```
commons-codec-1.4.jar
commons-collections-3.2.1.jar
commons-configuration-1.6.jar
commons-lang-2.6.jar
commons-logging-1.2.jar
guava-12.0.1.jar
hadoop-auth.jar
hadoop-common.jar
hbase-annotations-1.0.0-cdh5.4.1.jar
hbase-client-1.0.0-cdh5.4.1.jar
hbase-common-1.0.0-cdh5.4.1.jar
hbase-hadoop2-compat-1.0.0-cdh5.4.1.jar
hbase-it-1.0.0-cdh5.4.1.jar
hbase-prefix-tree-1.0.0-cdh5.4.1.jar
hbase-protocol-1.0.0-cdh5.4.1.jar
htrace-core-3.1.0-incubating.jar
htrace-core.jar
log4j-1.2.17.jar
netty-3.6.6.Final.jar
protobuf-java-2.5.0.jar
slf4j-api-1.7.5.jar
slf4j-log4j12.jar
zookeeper.jar
```

上面jar包对应的集群环境中就能找到。

### 示例代码
```
import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.HColumnDescriptor;
import org.apache.hadoop.hbase.HConstants;
import org.apache.hadoop.hbase.HTableDescriptor;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Admin;
import org.apache.hadoop.hbase.client.BufferedMutator;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.hbase.client.Delete;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.io.compress.Compression.Algorithm;

public class Demo {

    /**
     * 
     * @@Description: 创建表
     * @param config
     * @throws IOException
     */
    public static void createSchemaTables(Configuration config, String tableName, String familyName)
            throws IOException {
        try (Connection connection = ConnectionFactory.createConnection(config); Admin admin = connection.getAdmin()) {

            HTableDescriptor table = new HTableDescriptor(TableName.valueOf(tableName));
            table.addFamily(new HColumnDescriptor(familyName).setCompressionType(Algorithm.SNAPPY));

            System.out.print("Creating table. ");
            createOrOverwrite(admin, table);
            System.out.println(" Done.");
        }

    }

    public static void createOrOverwrite(Admin admin, HTableDescriptor table) throws IOException {
        if (admin.tableExists(table.getTableName())) {
            admin.disableTable(table.getTableName());
            admin.deleteTable(table.getTableName());
        }
        admin.createTable(table);
    }

    public static void modifySchema(Configuration config, String tableNameStr, String familyName) throws IOException {
        try (Connection connection = ConnectionFactory.createConnection(config); Admin admin = connection.getAdmin()) {

            TableName tableName = TableName.valueOf(tableNameStr);
            if (admin.tableExists(tableName)) {
                System.out.println("Table does not exist.");
                System.exit(-1);
            }

            HTableDescriptor table = new HTableDescriptor(tableName);

            // Update existing table
            HColumnDescriptor newColumn = new HColumnDescriptor("NEWCF");
            newColumn.setCompactionCompressionType(Algorithm.GZ);
            newColumn.setMaxVersions(HConstants.ALL_VERSIONS);
            admin.addColumn(tableName, newColumn);

            // Update existing column family
            HColumnDescriptor existingColumn = new HColumnDescriptor(familyName);
            existingColumn.setCompactionCompressionType(Algorithm.GZ);
            existingColumn.setMaxVersions(HConstants.ALL_VERSIONS);
            table.modifyFamily(existingColumn);
            admin.modifyTable(tableName, table);

            // Disable an existing table
            admin.disableTable(tableName);

            // Delete an existing column family
            admin.deleteColumn(tableName, familyName.getBytes("UTF-8"));

            // Delete a table (Need to be disabled first)
            admin.deleteTable(tableName);
        }
    }

    /**
     * 
     * @@Description: put
     * @param connection
     * @param tableName
     * @throws IOException
     */
    public static void put(Connection connection, String tableName) throws IOException {
        // flush 大小，默认2m
        connection.getConfiguration().set("hbase.client.write.buffer", "2097152");
        BufferedMutator bufferedMutator = connection.getBufferedMutator(TableName.valueOf(tableName));

        // 单行，也可以输出list<Mutation>
        bufferedMutator.mutate(new Put("test".getBytes("UTF-8")));

        // 删除
        bufferedMutator.mutate(new Delete("test".getBytes("UTF-8")));
    }

    public static void main(String... args) throws IOException {
        /**
         * hbase API操作，只需要设置前面3个参数即可。
         */
        Configuration config = HBaseConfiguration.create();
        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum", "hynamet01,hynamet02");
        conf.set("hbase.zookeeper.property.clientPort", "2181");
        // 默认是/hbase,根据配置文件指定
        conf.set("zookeeper.znode.parent", "/hbase");

        try (Connection connection = ConnectionFactory.createConnection(config);
                // 使用自定义的线程池
                // ConnectionFactory.createConnection(conf, pool);
                Admin admin = connection.getAdmin()) {

        }
    }

}
```

倒序查询：
hbase0.98版本之后的Scan支持`setReversed(true)`函数。
有意思的是，扫描时设置的startRow和stopRow必须正好反过来。

Hbase远程API操作，最核心的三个参数：
```
#zookeeper集群机器组，可多个
hbase.zookeeper.quorum=hynamet01,hynamet02

#zookeeper集群端口
hbase.zookeeper.property.clientPort

#zookeeper目录
zookeeper.znode.parent=/hbase
```

其它优化参数，后续我会写个总结。

### HBase系统架构

HBase Client使用HBase的RPC机制与HMaster和HRegionServer进行通信，对于管理类操作，Client与HMaster进行RPC；对于数据读写类操作，Client与HRegionServer进行RPC。

所以要保证客户端机器能访问hbase集群的所有机器，并配置本地hosts

## 常用过滤器

HBase为筛选数据提供了一组过滤器，通过这个过滤器可以在HBase中的数据的多个维度（行，列，数据版本）上进行对数据的筛选操作，也就是说过滤器最终能够筛选的数据能够细化到具体的一个存储单元格上（由行键，列明，时间戳定位）。通常来说，通过行键，值来筛选数据的应用场景较多。

### 过滤器示例

```
import java.io.IOException;
import java.io.UnsupportedEncodingException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Admin;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.hbase.client.Get;
import org.apache.hadoop.hbase.client.Result;
import org.apache.hadoop.hbase.client.ResultScanner;
import org.apache.hadoop.hbase.client.Scan;
import org.apache.hadoop.hbase.client.Table;
import org.apache.hadoop.hbase.filter.BinaryComparator;
import org.apache.hadoop.hbase.filter.CompareFilter;
import org.apache.hadoop.hbase.filter.FamilyFilter;
import org.apache.hadoop.hbase.filter.Filter;
import org.apache.hadoop.hbase.filter.FilterList;
import org.apache.hadoop.hbase.filter.QualifierFilter;
import org.apache.hadoop.hbase.filter.SingleColumnValueFilter;
import org.apache.hadoop.hbase.filter.SubstringComparator;
import org.apache.hadoop.hbase.filter.ValueFilter;
import org.apache.hadoop.hbase.util.Bytes;

public class FilterDemo {

    /**
     * 
     * @@Description: 根据row起始字符来查询
     * @param table
     * @throws UnsupportedEncodingException
     * @throws IOException
     */
    public static void scanByRow(Table table) throws UnsupportedEncodingException, IOException {
        /**
         * 通过rowKey的开始结束字符好scan
         */
        Scan scan = new Scan();
        scan.setStartRow("1".getBytes("UTF-8"));
        scan.setStopRow("2".getBytes("UTF-8"));

        // 指定最大的版本个数。如果不带任何参数调用setMaxVersions，表示取所有的版本。如果不掉用setMaxVersions，只会取到最新的版本.
        // scan.setMaxVersions(maxVersions);

        // setStartRow: >= ,setStopRow： < ，所有要到达<=效果等加上一个大于9的字符比如：
        // scan.setStopRow("2~".getBytes("UTF-8"));
        ResultScanner scanner = table.getScanner(scan);
        System.out.println(scanner);
    }

    /**
     * 
     * @@Description: 根据rowKey获取行
     * @param table
     * @throws Exception
     */
    public static void scanFilterRow(Table table) throws Exception {
        // 也可以传入,list<Get>来获取多行
        Result result = table.get(new Get("test".getBytes("UTF-8")));

        // 指定最大的版本个数。如果不带任何参数调用setMaxVersions，表示取所有的版本。如果不掉用setMaxVersions，只会取到最新的版本.
        // new Get("test".getBytes("UTF-8")).setMaxVersions();

        System.out.println(result);
    }

    /**
     * 
     * @@Description:FamilyFilter用于对列族名称进行匹配，找出匹配的列族下的数据项
     * @param table
     * @param tableName
     * @throws IOException
     */
    public static void scanFamilyFilter(Table table, byte[] tableName) throws IOException {
        Scan scan = new Scan();
        Filter filter = new FamilyFilter(CompareFilter.CompareOp.EQUAL, new BinaryComparator(Bytes.toBytes("main")));
        scan.setFilter(filter);
        ResultScanner scanner = table.getScanner(scan);
    }

    /**
     * 
     * @@Description: QualifierFilter用于对列名称进行匹配，找出匹配的列下的数据项
     * @param table
     * @param tableName
     * @throws IOException
     */
    public static void scanQualifierFilter(Table table, byte[] tableName) throws IOException {
        Scan scan = new Scan();
        Filter filter = new QualifierFilter(CompareFilter.CompareOp.EQUAL,
                new BinaryComparator(Bytes.toBytes("works")));

        scan.setFilter(filter);
        ResultScanner scanner = table.getScanner(scan);
    }

    /**
     * 
     * @@Description: ValueFilter用于对所有列的数据值进行比较，将匹配上的值表进行返回。如，如果字段包含20，则该列返回。
     * @param table
     * @throws IOException
     */
    public static void scanValueFilter(Table table) throws IOException {
        Scan scan = new Scan();
        SubstringComparator valueComparator = new SubstringComparator("20");
        Filter filter = new ValueFilter(CompareFilter.CompareOp.EQUAL, valueComparator);

        // like模糊查询
        // SingleColumnValueFilter filter = new
        // SingleColumnValueFilter(Bytes.toBytes(colFamily),
        // Bytes.toBytes(columnName), CompareFilter.CompareOp.EQUAL, new
        // SubstringComparator(value));
        // filter.setFilterIfMissing(true);

        scan.setFilter(filter);
        ResultScanner scanner = table.getScanner(scan);

        // 同样的，过滤器还可以应用到GET方法，只不过是只能对指定行的数据做再过滤
        // Get get = new Get(Bytes.toBytes("科技部"));
        // get.setFilter(filter);
        // Result result = table.get(get);
        // printResult(result);
    }

    /**
     * 
     * @@Description: SingleColumnValueFilter用于对指定列的数据值进行比较，将匹配上的值表进行返回。如，如果字段包含
     *                ”zhang“，则该列返回。
     * @param table
     * @throws IOException
     */
    public static void scanQualifierFilter(Table table) throws IOException {
        Scan scan = new Scan();
        // like模糊查询
        SingleColumnValueFilter filter = new SingleColumnValueFilter(Bytes.toBytes("INFO"), Bytes.toBytes("name"),
                CompareFilter.CompareOp.EQUAL, new SubstringComparator("zhang"));
        filter.setFilterIfMissing(true);

        scan.setFilter(filter);
        ResultScanner scanner = table.getScanner(scan);

    }

    /**
     * 
     * @@Description: FilterList 过滤器列表，用于对数据进行组合过滤查询,可实现多个过滤条件的OR和AND关系
     * @param table
     * @throws IOException
     */
    public static void scanMultColumnValueFilter(Table table) throws IOException {
        Scan scan = new Scan();
        // 查找创建时间为2010-10-01的记录
        Filter filter1 = new SingleColumnValueFilter(Bytes.toBytes("main"), Bytes.toBytes("parentdept"),
                CompareFilter.CompareOp.EQUAL, new BinaryComparator(Bytes.toBytes("总行")));
        Filter filter2 = new SingleColumnValueFilter(Bytes.toBytes("info"), Bytes.toBytes("createdate"),
                CompareFilter.CompareOp.EQUAL, new BinaryComparator(Bytes.toBytes("2011-10-01")));

        // MUST_PASS_ONE 表示只要任一条件通过则返回 相关于OR关系
        // MUST_PASS_ALL 表示必须所有条件都满足才返回 相当于AND关系
        FilterList list = new FilterList(FilterList.Operator.MUST_PASS_ALL);
        list.addFilter(filter1);
        list.addFilter(filter2);
        scan.setFilter(list);

        ResultScanner scanner = table.getScanner(scan);
    }

    public static void main(String... args) throws IOException {
        /**
         * hbase API操作，只需要设置前面3个参数即可。
         */
        Configuration config = HBaseConfiguration.create();
        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum", "hynamet01,hynamet02");
        conf.set("hbase.zookeeper.property.clientPort", "2181");
        // 默认是/hbase,根据配置文件指定
        conf.set("zookeeper.znode.parent", "/hbase");

        try (Connection connection = ConnectionFactory.createConnection(config);
                // 使用自定义的线程池
                // ConnectionFactory.createConnection(conf, pool);
                Admin admin = connection.getAdmin();
                Table table = connection.getTable(TableName.valueOf("test"));) {

        }
    }

}

```


### 过滤器详解

1、RowFilter：筛选出匹配的所有的行，对于这个过滤器的应用场景，是非常直观的：使用BinaryComparator可以筛选出具有某个行键的行，或者通过改变比较运算符（下面的例子中是CompareFilter.CompareOp.EQUAL）来筛选出符合某一条件的多条数据，以下就是筛选出行键为row1的一行数据：

```
Filter rf = new RowFilter(CompareFilter.CompareOp.EQUAL, new BinaryComparator(Bytes.toBytes("row1"))); // OK 筛选出匹配的所有的行
```

2、PrefixFilter：筛选出具有特定前缀的行键的数据。这个过滤器所实现的功能其实也可以由RowFilter结合RegexComparator来实现，不过这里提供了一种简便的使用方法，以下过滤器就是筛选出行键以row为前缀的所有的行：

```
Filter pf = new PrefixFilter(Bytes.toBytes("row")); // OK 筛选匹配行键的前缀成功的行
```

3、KeyOnlyFilter：这个过滤器唯一的功能就是只返回每行的行键，值全部为空，这对于只关注于行键的应用场景来说非常合适，这样忽略掉其值就可以减少传递到客户端的数据量，能起到一定的优化作用：

```
Filter kof = new KeyOnlyFilter(); // OK 返回所有的行，但值全是空
```

4、RandomRowFilter：从名字上就可以看出其大概的用法，本过滤器的作用就是按照一定的几率（<=0会过滤掉所有的行，>=1会包含所有的行）来返回随机的结果集，对于同样的数据集，多次使用同一个RandomRowFilter会返回不通的结果集，对于需要随机抽取一部分数据的应用场景，可以使用此过滤器：

```
Filter rrf = new RandomRowFilter((float) 0.8); // OK 随机选出一部分的行
```

5、InclusiveStopFilter：扫描的时候，我们可以设置一个开始行键和一个终止行键，默认情况下，这个行键的返回是前闭后开区间，即包含起始行，单不包含中指行，如果我们想要同时包含起始行和终止行，那么我们可以使用此过滤器：

```
Filter isf = new InclusiveStopFilter(Bytes.toBytes("row1")); // OK 包含了扫描的上限在结果之内
```

6、FirstKeyOnlyFilter：如果你只想返回的结果集中只包含第一列的数据，那么这个过滤器能够满足你的要求。它在找到每行的第一列之后会停止扫描，从而使扫描的性能也得到了一定的提升：

```
Filter fkof = new FirstKeyOnlyFilter(); // OK 筛选出第一个每个第一个单元格
```

7、ColumnPrefixFilter：顾名思义，它是按照列名的前缀来筛选单元格的，如果我们想要对返回的列的前缀加以限制的话，可以使用这个过滤器：

```
Filter cpf = new ColumnPrefixFilter(Bytes.toBytes("qual1")); // OK 筛选出前缀匹配的列
```

8、ValueFilter：按照具体的值来筛选单元格的过滤器，这会把一行中值不能满足的单元格过滤掉，如下面的构造器，对于每一行的一个列，如果其对应的值不包含ROW2_QUAL1，那么这个列就不会返回给客户端：

```
Filter vf = new ValueFilter(CompareFilter.CompareOp.EQUAL, new SubstringComparator("ROW2_QUAL1")); // OK 筛选某个（值的条件满足的）特定的单元格
```

9、ColumnCountGetFilter：这个过滤器来返回每行最多返回多少列，并在遇到一行的列数超过我们所设置的限制值的时候，结束扫描操作：

```
Filter ccf = new ColumnCountGetFilter(2); // OK 如果突然发现一行中的列数超过设定的最大值时，整个扫描操作会停止
```

10、SingleColumnValueFilter：用一列的值决定这一行的数据是否被过滤。在它的具体对象上，可以调用setFilterIfMissing(true)或者setFilterIfMissing(false)，默认的值是false，其作用是，对于咱们要使用作为条件的列，如果这一列本身就不存在，那么如果为true，这样的行将会被过滤掉，如果为false，这样的行会包含在结果集中。

```
SingleColumnValueFilter scvf = new SingleColumnValueFilter(
        Bytes.toBytes("colfam1"), 
        Bytes.toBytes("qual2"), 
        CompareFilter.CompareOp.NOT_EQUAL, 
        new SubstringComparator("BOGUS"));
scvf.setFilterIfMissing(false);
scvf.setLatestVersionOnly(true); // OK
```

<font color="red">使用SingleColumnValueFilter会要求COLUMNS中包含该字段，COLUMNS没有包含该字段时会返回错误的结果，设置`scvf.setFilterIfMissing(true)`即可解决该问题</font>

11、SingleColumnValueExcludeFilter：这个与10种的过滤器唯一的区别就是，作为筛选条件的列的不会包含在返回的结果中。

12、SkipFilter：这是一种附加过滤器，其与ValueFilter结合使用，如果发现一行中的某一列不符合条件，那么整行就会被过滤掉：

```
Filter skf = new SkipFilter(vf); // OK 发现某一行中的一列需要过滤时，整个行就会被过滤掉
```

13、WhileMatchFilter：这个过滤器的应用场景也很简单，如果你想要在遇到某种条件数据之前的数据时，就可以使用这个过滤器；当遇到不符合设定条件的数据的时候，整个扫描也就结束了：

```
Filter wmf = new WhileMatchFilter(rf); // OK 类似于Python itertools中的takewhile
```

14、FilterList：用于综合使用多个过滤器。其有两种关系：FilterList.Operator.MUST_PASS_ONE和FilterList.Operator.MUST_PASS_ALL，默认的是FilterList.Operator.MUST_PASS_ALL，顾名思义，它们分别是AND和OR的关系，并且FilterList可以嵌套使用FilterList，使我们能够表达更多的需求：

```
List<Filter> filters = new ArrayList<Filter>();
filters.add(rf);
filters.add(vf);
FilterList fl = new FilterList(FilterList.Operator.MUST_PASS_ALL, filters); // OK 综合使用多个过滤器， AND 和 OR 两种关系

```


## 参考资料
- http://blog.csdn.net/cnweike/article/details/42920547
- http://hbase.apache.org/book.html