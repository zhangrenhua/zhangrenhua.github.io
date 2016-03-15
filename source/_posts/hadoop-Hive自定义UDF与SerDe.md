---
title: Hive自定义函数（UDF、UDAF、UDTF）与SerDe
description: 时间匆匆，一晃又一年，今天特意抽一下午时间来整理博客。当Hive提供的内置函数无法满足我们的业务处理需要时，此时就可以考虑使用自定义函数，自定义函数有三种（UDF、UDAF、UDTF）下面我会描述这三种自定义函数的作用并提供示例代码。本次除了整理自定义函数外还将在github上看到的一位牛人写的自定义serde记录下来。
tags: [hadoop生态圈, hive]
categories: hive
date: 2016-2-21 16:49:05
---

## 背景

时间匆匆，一晃又一年，今天特意抽一下午时间来整理博客。当Hive提供的内置函数无法满足我们的业务处理需要时，此时就可以考虑使用自定义函数，自定义函数有三种（UDF、UDAF、UDTF）下面我会描述这三种自定义函数的作用并提供示例代码。本次除了整理自定义函数外还将在github上看到的一位牛人写的自定义serde记录下来。

Hive支持三种自定义函数，我们逐个讲解。


笔者使用hive1.1进行测试。

## 自定义函数

### UDF

普通的用户自定义函数。接受单行输入，并产生单行输出。

```
package com.example.hive.udf;
 
import org.apache.hadoop.hive.ql.exec.UDF;
import org.apache.hadoop.io.Text;

public final class Lower extends UDF {
  public Text evaluate(final Text s) {
    if (s == null) { return null; }
    return new Text(s.toString().toLowerCase());
  }
}
```

继承UDF，重写evaluate方法即可。

注：可以在class中加入@Description注解，并添加信息，这样在命令行中执行`DESCRIBE FUNCTION xxx`时能显示注解内容。

根据后面步骤部署后测试：
```
hive> select my_lower(title), sum(freq) from titles group by my_lower(title);
```


### UDAF

用户定义聚集函数（User-defined aggregate function）。接受多行输入，并产生单行输出。比如MAX，COUNT函数。

步骤：
1.继承AbstractGenericUDAFResolver
2.内部类实现UDAFEvaluator

具体步骤参考：
https://cwiki.apache.org/confluence/display/Hive/GenericUDAFCaseStudy

### UDTF

用户定义表生成函数（User-defined table-generating function）。接受单行输入，并产生多行输出（即一个表）。

不是特别常用，具体步骤参考：
https://cwiki.apache.org/confluence/display/Hive/DeveloperGuide+UDTF


### 部署函数

将代码打包，比如my_jar.jar，进入hive命令行执行：
```
hive> add jar /home/xxx/my_jar.jar;
Added my_jar.jar to class path
```

创建函数：
```
create function my_db.my_lower as 'com.example.hive.udf.Lower';
```

**在hive0.13版本中，可以直接支持创建函数时指定hdfs目录上的jar文件：**
```
CREATE FUNCTION myfunc AS 'myclass' USING JAR 'hdfs:///path/to/jar';
```


## 自定义SerDe

Serde主要用来处理hive中一行数据字段的切分逻辑，在低版本中hive是不能解析csv格式的文本数据的，直到0.14版本后hive引入OpenCSVSerde用来支持。

### CSV Serde

```
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hive.serde.serdeConstants;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;
import org.apache.hadoop.hive.serde2.objectinspector.StructField;
import org.apache.hadoop.hive.serde2.objectinspector.StructObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.StringObjectInspector;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.Writable;

import java.io.CharArrayReader;
import java.io.IOException;
import java.io.Reader;
import java.io.StringWriter;
import java.io.Writer;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Properties;

import au.com.bytecode.opencsv.CSVReader;
import au.com.bytecode.opencsv.CSVWriter;

/**
 * OpenCSVSerde use opencsv to deserialize CSV format.
 * Users can specify custom separator, quote or escape characters. And the default separator(\),
 * quote("), and escape characters(\) are the same as the opencsv library.
 *
 */
@SerDeSpec(schemaProps = {
    serdeConstants.LIST_COLUMNS,
    OpenCSVSerde.SEPARATORCHAR, OpenCSVSerde.QUOTECHAR, OpenCSVSerde.ESCAPECHAR})
public final class OpenCSVSerde extends AbstractSerDe {

  public static final Log LOG = LogFactory.getLog(OpenCSVSerde.class.getName());
  private ObjectInspector inspector;
  private String[] outputFields;
  private int numCols;
  private List<String> row;

  private char separatorChar;
  private char quoteChar;
  private char escapeChar;

  public static final String SEPARATORCHAR = "separatorChar";
  public static final String QUOTECHAR = "quoteChar";
  public static final String ESCAPECHAR = "escapeChar";

  @Override
  public void initialize(final Configuration conf, final Properties tbl) throws SerDeException {

    final List<String> columnNames = Arrays.asList(tbl.getProperty(serdeConstants.LIST_COLUMNS)
        .split(","));

    numCols = columnNames.size();

    final List<ObjectInspector> columnOIs = new ArrayList<ObjectInspector>(numCols);

    for (int i = 0; i < numCols; i++) {
      columnOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
    }

    inspector = ObjectInspectorFactory.getStandardStructObjectInspector(columnNames, columnOIs);
    outputFields = new String[numCols];
    row = new ArrayList<String>(numCols);

    for (int i = 0; i < numCols; i++) {
      row.add(null);
    }

    separatorChar = getProperty(tbl, SEPARATORCHAR, CSVWriter.DEFAULT_SEPARATOR);
    quoteChar = getProperty(tbl, QUOTECHAR, CSVWriter.DEFAULT_QUOTE_CHARACTER);
    escapeChar = getProperty(tbl, ESCAPECHAR, CSVWriter.DEFAULT_ESCAPE_CHARACTER);
  }

  private char getProperty(final Properties tbl, final String property, final char def) {
    final String val = tbl.getProperty(property);

    if (val != null) {
      return val.charAt(0);
    }

    return def;
  }

  @Override
  public Writable serialize(Object obj, ObjectInspector objInspector) throws SerDeException {
    final StructObjectInspector outputRowOI = (StructObjectInspector) objInspector;
    final List<? extends StructField> outputFieldRefs = outputRowOI.getAllStructFieldRefs();

    if (outputFieldRefs.size() != numCols) {
      throw new SerDeException("Cannot serialize the object because there are "
              + outputFieldRefs.size() + " fields but the table has " + numCols + " columns.");
    }

    // Get all data out.
    for (int c = 0; c < numCols; c++) {
      final Object field = outputRowOI.getStructFieldData(obj, outputFieldRefs.get(c));
      final ObjectInspector fieldOI = outputFieldRefs.get(c).getFieldObjectInspector();

      // The data must be of type String
      final StringObjectInspector fieldStringOI = (StringObjectInspector) fieldOI;

      // Convert the field to Java class String, because objects of String type
      // can be stored in String, Text, or some other classes.
      outputFields[c] = fieldStringOI.getPrimitiveJavaObject(field);
    }

    final StringWriter writer = new StringWriter();
    final CSVWriter csv = newWriter(writer, separatorChar, quoteChar, escapeChar);

    try {
      csv.writeNext(outputFields);
      csv.close();

      return new Text(writer.toString());
    } catch (final IOException ioe) {
      throw new SerDeException(ioe);
    }
  }

  @Override
  public Object deserialize(final Writable blob) throws SerDeException {
    Text rowText = (Text) blob;

    CSVReader csv = null;
    try {
      csv = newReader(new CharArrayReader(rowText.toString().toCharArray()), separatorChar,
              quoteChar, escapeChar);
      final String[] read = csv.readNext();

      for (int i = 0; i < numCols; i++) {
        if (read != null && i < read.length) {
          row.set(i, read[i]);
        } else {
          row.set(i, null);
        }
      }

      return row;
    } catch (final Exception e) {
      throw new SerDeException(e);
    } finally {
      if (csv != null) {
        try {
          csv.close();
        } catch (final Exception e) {
          LOG.error("fail to close csv writer ", e);
        }
      }
    }
  }

  private CSVReader newReader(final Reader reader, char separator, char quote, char escape) {
    // CSVReader will throw an exception if any of separator, quote, or escape is the same, but
    // the CSV format specifies that the escape character and quote char are the same... very weird
    if (CSVWriter.DEFAULT_ESCAPE_CHARACTER == escape) {
      return new CSVReader(reader, separator, quote);
    } else {
      return new CSVReader(reader, separator, quote, escape);
    }
  }

  private CSVWriter newWriter(final Writer writer, char separator, char quote, char escape) {
    if (CSVWriter.DEFAULT_ESCAPE_CHARACTER == escape) {
      return new CSVWriter(writer, separator, quote, "");
    } else {
      return new CSVWriter(writer, separator, quote, escape, "");
    }
  }

  @Override
  public ObjectInspector getObjectInspector() throws SerDeException {
    return inspector;
  }

  @Override
  public Class<? extends Writable> getSerializedClass() {
    return Text.class;
  }

  @Override
  public SerDeStats getSerDeStats() {
    return null;
  }
}

```

将代码打包，比如csv-serde.jar：
```
add jar path/to/csv-serde.jar;

create table my_table(a string, b string, ...)
  row format serde 'com.bizo.hive.serde.csv.CSVSerde'
  stored as textfile
;
```

### Custom formatting

The default separator, quote, and escape characters from the opencsv library are:
```
DEFAULT_ESCAPE_CHARACTER \
DEFAULT_QUOTE_CHARACTER  "
DEFAULT_SEPARATOR        ,
```

You can also specify custom separator, quote, or escape characters.
```
add jar path/to/csv-serde.jar;

create table my_table(a string, b string, ...)
 row format serde 'com.bizo.hive.serde.csv.CSVSerde'
 with serdeproperties (
   "separatorChar" = "\t",
   "quoteChar"     = "'",
   "escapeChar"    = "\\"
  )   
 stored as textfile
;
```

注：比如一些定长格式的数据我们也可以自定义一个serde用来能支持将数据直接导入hive表中。

## 参考资料

- https://cwiki.apache.org/confluence/display/Hive/LanguageManual
- https://github.com/ogrodnek/csv-serde/blob/master/readme.md
