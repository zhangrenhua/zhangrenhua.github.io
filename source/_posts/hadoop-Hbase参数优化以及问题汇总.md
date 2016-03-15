---
title: Hbase参数优化以及问题汇总
description: 在工作中由于涉及高并发Hbase写入项目，对Hbase参数调优有过深入研究。当数据到了一定的量的时候会出现各种错误导致写入失败，最普遍的有，连接超时、regionServe自动下线，然而regionServer自动下线是个最头疼的问题（有的时候宁愿写入速度慢一点也不想RegionServer当掉）。下面主要讲解Hbase的一些参数调优，以及GC调优。
tags: [hadoop生态圈, hbase]
categories: hbase
date: 2016-1-17 13:35:22
---

## 背景

在工作中由于涉及高并发Hbase写入项目，对Hbase参数调优有过深入研究。当数据到了一定的量的时候会出现各种错误导致写入失败，最普遍的有，连接超时、regionServe自动下线，然而regionServer自动下线是个最头疼的问题（有的时候宁愿写入速度慢一点也不想RegionServer当掉）。下面主要讲解Hbase的一些参数调优，以及GC调优。

## 参数调优详解

以下参数，除了前面两个是在hbase-env.sh文件中，其它的都是hbase-site.xml中。

**HBase Master Maximum Java heap size**

Hmaster进程最大使用堆空间大小，默认1G，如果内存充裕可调到2-4G

**RegionServers maximum Java heap size**

RegionServer进程最大使用堆空间大小，默认2G,Cloudera专家建议内存设置不要超过18GB，Region Server管理的Region个数不要超过300个。

**hbase.regionserver.handler.count=100**

该配置定义了每个Region Server上的RPC Handler的数量。Region Server通过RPC Handler接收外部请求并加以处理。所以提升RPC Handler的数量可以一定程度上提高HBase接收请求的能力。当然，handler数量也不是越大越好，这要取决于节点的硬件情况。

**hbase.hregion.max.filesize=10737418240**

配置项的含义是当region的大小大于设定值后hbase就会开始split，可根据存储内容进行适当调整。比如是存储文件的话那应该将此值设置更大。

**hbase.hregion.majorcompaction=0**
major、compaction的执行周期，默认是一天，可以设置为0禁用。利用后台调度，在业务不忙的时间点单独运行。

**hbase.hregion.memstore.block.multiplier=4**

容忍缓存中写入数据超过hbase.hregion.memstore.flush.size缓冲区最大的倍数；
每次写入时，判断超出缓存的倍数后，不能再写入，阻塞，等待flush完成后，会进行GC回收，regionserver会暂停完成flush，并gc回收后，则继续接收数据； 这个过程中容易出现regionserver服务暂停，与hmaster失败心跳超时，引起regionserver下线。

**hbase.hregion.memstore.flush.size=134217728**

当一个region里的memstore占用内存大小超过（memstore.flush * mestore.block.multiplier） 时执行flush。默认128M不建议调大（可根据硬盘的写入速度来适当调整）。

**hbase.balancer.period=1800000**

RegionServer负载均衡特定时间间隔，默认5分钟，可是稍微调长至半个小时。

**hfile.block.cache.size=0.4**

HFile文件的块缓存大小占堆内存大小的比例，如果对读性能要求很高，可稍微调大一点。

**hbase.regionserver.global.memstore.upperLimit=0.4**

当ReigonServer内所有region的memstores所占用内存总和达到heap的40%时，HBase会强制block所有的更新并flush这些region以释放所有memstore占用的内存。

**hbase.server.thread.wakefrequency=500**

每间隔hbase.server.thread.wakefrequency时间（默认10s）检查一次regionserver缓冲区大小，超过hbase.hregion.memstore.flush.size此大小则flush刷新到hdfs文件中；
若此参数间隔太久，且数据写入太多，会引起长时间的阻塞等待flush；因此高并发写入时，此参数要适当调小。

**hbase.hstore.blockingStoreFiles=20**

在flush时,当一个region中的Store(Coulmn Family)内有超过xx个storefile时,则block所有的写请求进行compaction。设置过小会使影响系统吞吐率（使吞吐率不高）。建议将该值调大一些，（但也不应过大，经验值是20左右吧。太大的话会在系统压力很大时使storefile过多，compact一直无法完成，扫库或者数据读取的性能会受到影响）。

**hbase.hstore.compactionThreshold=6**
hbase.hstore.compactionThreshold：HStore的storeFile数量>= compactionThreshold配置的值，则可能会进行compact，默认值为3，可以调大，比如设置为6，在定期的major compact中进行剩下文件的合并。

**hbase.hregion.memstore.mslab.enabled=true**

启用hbase0.90x版本引入的一种高级机制来缓解region服务器内存碎片文件。


### 客户端参数

**Write Buffer Size**

```
HTable htable = new HTable(config, tablename);   
htable.setWriteBufferSize(6 * 1024 * 1024);   
htable.setAutoFlush(false);
```

HBase Client会在数据累积到设置的阈值后才提交Region Server。这样做的好处在于可以减少RPC连接次数，达到批次效果。

**hbase.client.keyvalue.maxsize=10485760**

```
connection.getConfiguration().set("hbase.client.keyvalue.maxsize", "104857600");
```

列族的最大值，默认10M，如果存储内容超过10M则需更改次值。

**hfile.block.cache.size**

```
Get get = new Get(rowkey.getBytes());
get.setCacheBlocks(false); //不进行block缓存
```

**WAL**

```
Put put = new Put("test".getBytes("UTF-8"));
put.setDurability(Durability.SKIP_WAL);
```

其实不推荐关闭WAL，关闭WAl日志后，如果RegionServer当掉的话，会导致还没有flsh到HDFS上的数据丢失。<font color="red">除非有很好的数据重跑机制，否则不推荐关闭</font>

## GC调优

由于GC调优内容比较多，所以单独抽出来写，后续会补上。

## 表设计原理

**尽量只使用一个列族**

Hbase进行合并是会对所有列族进行合并

**列族采用压缩算法**

```
alter 'test',{NAME=>'INFO',COMPRESSION=>'SNAPPY'}
```

**rowKey设计尽量不要用时间或者自然顺序的字段作为开始**

这样能保证，写入性能。如果某些业务场景必须这样，可以使用Salt方式来实现。

Salt：
rowKey前面加上3位随机码，查询的时候多线程并行查询100次（rowKey前面加上0-99）。仅限于根据rowKey精确查询。

下面还介绍了通过timestamp来解决，用时间作为RowKey前缀写入性能慢的问题。

**预分区**

可以根据预先固定好的需求来进行预先分区，提高写入性能。

```
public static boolean createTable(Admin admin, HTableDescriptor table, byte[][] splits)
throws IOException {
  try {
    admin.createTable( table, splits );
    return true;
  } catch (TableExistsException e) {
    logger.info("table " + table.getNameAsString() + " already exists");
    // the table already exists...
    return false;
  }
}

public static byte[][] getHexSplits(String startKey, String endKey, int numRegions) {
  byte[][] splits = new byte[numRegions-1][];
  BigInteger lowestKey = new BigInteger(startKey, 16);
  BigInteger highestKey = new BigInteger(endKey, 16);
  BigInteger range = highestKey.subtract(lowestKey);
  BigInteger regionIncrement = range.divide(BigInteger.valueOf(numRegions));
  lowestKey = lowestKey.add(regionIncrement);
  for(int i=0; i < numRegions-1;i++) {
    BigInteger key = lowestKey.add(regionIncrement.multiply(BigInteger.valueOf(i)));
    byte[] b = String.format("%016x", key).getBytes();
    splits[i] = b;
  }
  return splits;
}
```


## Hbase的TTL字段超时设置测试


**首先disable这个表，否则记录会丢失。我们建立一个测试表test，有一个列簇fa**

```
hbase(main):011:0> create 'test','fa'
hbase(main):011:0> describe 'test'
Table test is ENABLED                                                                                                        
test                                                                                                                         
COLUMN FAMILIES DESCRIPTION                                                                                                  
{NAME => 'fa', DATA_BLOCK_ENCODING => 'NONE', BLOOMFILTER => 'ROW', REPLICATION_SCOPE => '0', VERSIONS => '1', COMPRESSION =>
 'NONE', MIN_VERSIONS => '0', TTL => 'FOREVER', KEEP_DELETED_CELLS => 'FALSE', BLOCKSIZE => '65536', IN_MEMORY => 'false', BL
OCKCACHE => 'true'}                                                                                                          
1 row(s) in 0.0230 seconds
```

表默认的`TTL => FOREVER`超时时间为永久存活（hbase1.0之前好像是多少年去了）。
那么对于某些列，我存储一段时间后，需要该列值失效。那如何设置？下面我们做一个测试：

**我们先向test表中插入一下记录：**

```
1           2015-11-11 14:09:40
2           2015-11-12 14:09:40
3           2015-11-13 14:09:40
4           2015-11-14 14:09:40
5           2015-11-15 14:09:40

hbase(main):082:0> put 'test',1,'fa:uptime','2015-11-11 14:09:40'
。。。。。。。。。。。。。
hbase(main):123:0>  scan 'test'
ROW                        COLUMN+CELL                                                              
 1                         column=fa:uptime, timestamp=1415688066815, value=2015-11-11 14:09:40     
 2                         column=fa:uptime, timestamp=1415688082648, value=2015-11-12 14:09:40     
 3                         column=fa:uptime, timestamp=1415688092101, value=2015-11-13 14:09:40     
 4                         column=fa:uptime, timestamp=1415688101473, value=2015-11-14 14:09:40     
 5                         column=fa:uptime, timestamp=1415688115318, value=2015-11-15 14:09:40     
5 row(s) in 0.0400 seconds
```

**我们修改过期时间为200s 查看记录**

```
hbase(main):126:0> disable 'test'           //修改表前，需要先disable 表
hbase(main):114:0> alter "test",NAME=>'fa',TTL=>'200'  //修改表的列簇超时时间为200s
hbase(main):114:0> enable  "test"          //使表可用，以供查询
hbase(main):129:0>  scan 'test'           //此时查询时，表中原来的5条，记录均存在，接下来我们更新其中一条。再查看。
ROW                        COLUMN+CELL                                                              
 1                         column=fa:uptime, timestamp=1415688066815, value=2015-11-11 14:09:40     
 2                         column=fa:uptime, timestamp=1415688082648, value=2015-11-12 14:09:40     
 3                         column=fa:uptime, timestamp=1415688092101, value=2015-11-13 14:09:40     
 4                         column=fa:uptime, timestamp=1415688101473, value=2015-11-14 14:09:40     
 5                         column=fa:uptime, timestamp=1415688115318, value=2015-11-15 14:09:40     
5 row(s) in 0.0420 seconds
```

由于200s时间很短，我们选择跟新第三条记录：跟新为：

```
3           2015-11-16 14:09:40
hbase(main):130:0> put 'test',3,'fa:uptime','2015-11-16 14:09:40'
0 row(s) in 0.0200 seconds
```

更新后等待200秒再去查看，发现表中的记录只剩下一条了：也就是我们最后跟新的这条。为什么前面的会消失呢，是因为TTL生效记录过期了。

```
hbase(main):131:0> scan 'test'
ROW                        COLUMN+CELL                                                              
 3                         column=fa:uptime, timestamp=1415688323548, value=2015-11-16 14:09:40     
1 row(s) in 0.0140 seconds
```

再过200秒后查看，表中的记录数为空，所有的记录都超时，被删除。

**总结**

测试说明，我们可用对hbase的某些列簇，做一些高级设置，例如超时，压索，等设置。注意在设置之前需要先disable表，否则，表中记录会被清空。另外，TTL=>的更新超时时间是指：该列最后更新的时间，到超时时间的限制，而不是第一次创建，到超时时间。

**通过设置属性可以禁用TTL：**
`hbase.store.delete.expired.storefile=false`

**hbase1.0中加入了行的ttl设置，可以通过代码设置：**
```
Put put = new Put("test".getBytes("UTF-8"));
// 单位：毫秒(milliseconds)
put.setTTL(200 * 100);
```

<font color="red">代码中TTLS表示单位毫秒而不是秒，如果代码中设置的TTL时间比列族设置的要长，以列族设置的为准。并且如果列族没有设置TTL超时时间，则以代码中设置为准。</font>

## 保留删除数据

默认情况下，删除标记延长回到时间的起点。因此，获取或扫描操作将无法看到已删除的单元格（行或列），甚至当获取或扫描操作指示的时间范围内删除标记放在前。列族可以选择保留已删除单元格。在这种情况下，删除的细胞仍然可以检索到的，只要这些操作指定结束之前的任何时间戳删除会影响细胞一个时间范围。这允许点在时间查询甚至在删除的存在。

删除单元格仍受到TTL和永远不会有更多的不是删除单元格“版本的最大数”。一种新的“原始”扫描选项返回所有被删除的行，并删除标记。

### 设置删除仍然保留

shell：
```
hbase> hbase> alter ‘t1′, NAME => ‘f1′, KEEP_DELETED_CELLS => true
```

java：
```
HColumnDescriptor.setKeepDeletedCells(true);
```

### 示例

**不设置删除保留属性**

```
create 'test', {NAME=>'e', VERSIONS=>2147483647}
put 'test', 'r1', 'e:c1', 'value', 10
put 'test', 'r1', 'e:c1', 'value', 12
put 'test', 'r1', 'e:c1', 'value', 14
delete 'test', 'r1', 'e:c1',  11

hbase(main):017:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                              COLUMN+CELL
 r1                                              column=e:c1, timestamp=14, value=value
 r1                                              column=e:c1, timestamp=12, value=value
 r1                                              column=e:c1, timestamp=11, type=DeleteColumn
 r1                                              column=e:c1, timestamp=10, value=value
1 row(s) in 0.0120 seconds

hbase(main):018:0> flush 'test'
0 row(s) in 0.0350 seconds

hbase(main):019:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                              COLUMN+CELL
 r1                                              column=e:c1, timestamp=14, value=value
 r1                                              column=e:c1, timestamp=12, value=value
 r1                                              column=e:c1, timestamp=11, type=DeleteColumn
 r1                                              column=e:c1, timestamp=10, value=value
1 row(s) in 0.0120 seconds

hbase(main):020:0> major_compact 'test'
0 row(s) in 0.0260 seconds

hbase(main):021:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                              COLUMN+CELL
 r1                                              column=e:c1, timestamp=14, value=value
 r1                                              column=e:c1, timestamp=12, value=value
 r1                                              column=e:c1, timestamp=10, value=value
1 row(s) in 0.0120 seconds
```


**设置删除保留属性**

```
hbase(main):005:0> create 'test', {NAME=>'e', VERSIONS=>2147483647, KEEP_DELETED_CELLS => true}
0 row(s) in 0.2160 seconds

=> Hbase::Table - test
hbase(main):006:0> put 'test', 'r1', 'e:c1', 'value', 10
0 row(s) in 0.1070 seconds

hbase(main):007:0> put 'test', 'r1', 'e:c1', 'value', 12
0 row(s) in 0.0140 seconds

hbase(main):008:0> put 'test', 'r1', 'e:c1', 'value', 14
0 row(s) in 0.0160 seconds

hbase(main):009:0> delete 'test', 'r1', 'e:c1',  11
0 row(s) in 0.0290 seconds

hbase(main):010:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                                                                          COLUMN+CELL
 r1                                                                                          column=e:c1, timestamp=14, value=value
 r1                                                                                          column=e:c1, timestamp=12, value=value
 r1                                                                                          column=e:c1, timestamp=11, type=DeleteColumn
 r1                                                                                          column=e:c1, timestamp=10, value=value
1 row(s) in 0.0550 seconds

hbase(main):011:0> flush 'test'
0 row(s) in 0.2780 seconds

hbase(main):012:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                                                                          COLUMN+CELL
 r1                                                                                          column=e:c1, timestamp=14, value=value
 r1                                                                                          column=e:c1, timestamp=12, value=value
 r1                                                                                          column=e:c1, timestamp=11, type=DeleteColumn
 r1                                                                                          column=e:c1, timestamp=10, value=value
1 row(s) in 0.0620 seconds

hbase(main):013:0> major_compact 'test'
0 row(s) in 0.0530 seconds

hbase(main):014:0> scan 'test', {RAW=>true, VERSIONS=>1000}
ROW                                                                                          COLUMN+CELL
 r1                                                                                          column=e:c1, timestamp=14, value=value
 r1                                                                                          column=e:c1, timestamp=12, value=value
 r1                                                                                          column=e:c1, timestamp=11, type=DeleteColumn
 r1                                                                                          column=e:c1, timestamp=10, value=value
1 row(s) in 0.0650 seconds
```

### 总结

保留已删除值是为了避免HBase的删除单元格时将其删除的唯一原因是删除标记。因此，与KEEP_DELETED_CELLS启用，<font color='red'>只有超过版本个数或则TTL超时，否则这条记录将不会删除。</font>

## 利用TimeStamp来解决RowKey前缀为时间写入性能慢问题

### RowKey设计

由时间前缀替换成TimeStamp，如：
```
#时间前缀
rowKey=20160125215720_zrh

#修改后
rowKey=zrh
timeStamp=1453730240000
```

`timeStamp=1453730240000`为20160125215720表示的毫秒数。

### 通过代码设置timeStamp

```
Put put = new Put("test".getBytes("UTF-8"));
SimpleDateFormat format = new SimpleDateFormat("yyyyMMddHHmmss");
long ts = format.parse("20160125215720").getTime();
System.out.println(ts);
put.addColumn("INFO".getBytes(), "sex".getBytes(), 1l, "男".getBytes());
```

### 代码查询

```
SimpleDateFormat format = new SimpleDateFormat("yyyyMMddHHmmss");
long ts = format.parse("20160125215720").getTime();
Scan scan = new Scan();
scan.setTimeStamp(ts);
// 或者
// scan.setTimeRange(ts, ts + 1);
```

通过setTimeRange设置时间范围。

### 需要注意的两个地方

在hbase scan中为了提高效率，一般采用setStartRow和setStopRow或者采用filter的方式来减少结果集。还有一种方式就是用setTimeRange，它可以指定返回指定时间范围内的columns数据。对setTimeRange函数的使用，有两个特别需要注意的地方：

(1)  setTimeRange指定的是columns（列，也叫qualify）的时间范围，而不是columns family（列族）的范围。

这样如果一个表的多个columns的时间不一致，那么scan的返回结果就可能只包含一个columns的数据，其他columns的数据并不会返回。原因是：hbase中插入数据时，数据的列和值是存在KeyValue中，KeyValue的时间就是当前时间。KeyValue会被加入到columns family中，而columns family是不带时间戳的。

对于这种情况，在程序中需要特别注意不要遗漏某个column的数据。


(2)  scan如果指定了TimeRange，那么可能返回旧版本的数据。

先创建一张表，注意在创建表的时候故意只指定了一个版本：
```
create 'zhangrenhua',{NAME=>'cf',VERSION=>1} // 一个版本
put 'zhangrenhua','row1','cf:a','value1'
put 'zhangrenhua','row2','cf:a','value2'
put 'zhangrenhua','row3','cf:a','value3'
put 'zhangrenhua','row3','cf:a','value4'

hbase(main):045:0> scan 'zhangrenhua'
ROW                                      COLUMN+CELL
 row1                                    column=cf:a, timestamp=1362663065599, value=value1
 row2                                    column=cf:a, timestamp=1362663072949, value=value2
 row3                                    column=cf:a, timestamp=1362663086941, value=value4
3 row(s) in 0.0170 seconds
```


如果指定了时间范围呢？试一下：

```
hbase(main):046:0> scan 'zhangrenhua',{TIMERANGE=>[1362663065599,1362663086941]}
ROW                                      COLUMN+CELL                                                                                                         
 row1                                    column=cf:a, timestamp=1362663065599, value=value1                                                                  
 row2                                    column=cf:a, timestamp=1362663072949, value=value2                                                                  
 row3                                    column=cf:a, timestamp=1362663084573, value=value3                                                                  
3 row(s) in 0.0110 seconds
```


可以看到，row3这条记录返回的值是value3。再试一下：

```
hbase(main):048:0> scan 'zhangrenhua',{TIMERANGE=>[1362663065599,1362663086942]}
ROW                                      COLUMN+CELL                                                                                                         
 row1                                    column=cf:a, timestamp=1362663065599, value=value1                                                                  
 row2                                    column=cf:a, timestamp=1362663072949, value=value2                                                                  
 row3                                    column=cf:a, timestamp=1362663086941, value=value4                                                                  
3 row(s) in 0.0100 seconds
```

这时是和预期的一样，row3这条记录返回的是最新的值value4了。

为什么明明只指定了一个版本，但是设置不同的时间条件，竟然可以分别返回两个版本的数据？这是因为(row3, value3)这行记录虽然被hbase标记为”需要删除”，但是真正的删除在compaction阶段才会进行。(row3, value3)这条记录通过get和普通的scan都是无法访问到的，只有当设置了TimeRange才能访问到。

所以，对于scan时设置了TimeRange的程序来说，要注意可能会得到一些旧版本的数据。

此外：scan的时候可以通过`setMaxVersions()`来对每个column返回指定版本个数的数据
`Result.getColumn()`是返回指定列上的所有版本数据
`Result.getColumnLatest()`可以返回最新版本的数据


## 如何将一个HBase CF指定为IN_MEMORY

通过指定列族的`IN_MEMORY=true`属性，来让该列族的数据缓存在内存中，以提高查询性能。<font color="red">一些保存元数据的小表可以使用该属性来提升性能。</font>

### 创建table的时候可以指定列族的属性

shell：
```
create 'test', {NAME => 'edp', IN_MEMORY => true}
```

Java：
```
HColumnDescriptor.setInMemory(true);
```


### 使用N_MEMORY需注意的地方

1、标记`IN_MEMORY=>'true'`的column family的总体积最好不要超过in-memory cache的大小（in-memory cache = heap size * hfile.block.cache.size * 0.85 * 0.25），特别是当总体积远远大于了in-memory cache时，会在in-memory cache上发生严重的颠簸。

2、换个角度再看，普遍提到的使用in-memory cache的场景是把元数据表的column family声明为`IN_MEMORY=>true`。实际上这里的潜台词是：元数据表都很小。其时我们也可以大胆地把一些需要经常访问的，总体积不会超过in-memory cache的column family都设为IN_MEMORY=>'true'从而更加充分地利用cache空间。就像前面提到的，普通的block永远是不会被放入in-memory cache的，只存放少量metadata是对in-memory cache资源的浪费（未来的版本应该提供三种区段的比例配置功能）。

##  总结

**建表的一些默认参数值**
```
hbase> describe 'test'
DESCRIPTION                                          ENABLED
 'test', {NAME => 'cf', DATA_BLOCK_ENCODING => 'NONE false
 ', BLOOMFILTER => 'ROW', REPLICATION_SCOPE => '0',
 VERSIONS => '1', COMPRESSION => 'GZ', MIN_VERSIONS
 => '0', TTL => 'FOREVER', KEEP_DELETED_CELLS => 'fa
 lse', BLOCKSIZE => '65536', IN_MEMORY => 'false', B
 LOCKCACHE => 'true'}
1 row(s) in 0.1070 seconds
```

1、 客户端以表格的形式读取数据
2、 一张表是被划分成多个Hregion区域
3、 Hregion是被Hregion服务器管理的，当客户端需要访问某行数据的时候，需要访问对应的Hregion服务器。
4、 Hregions服务器里面有三种方式保存数据：
A、 Hmemcache高速缓存，保留是最新写入的数据
B、 Hlog记录文件，保留的是提交成功了，但未被写入文件的数据
C、 Hstores文件，数据的物理存放形式。

## 参考
- http://hbase.apache.org/book.html