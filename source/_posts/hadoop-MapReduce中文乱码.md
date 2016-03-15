---
title: MapReduce中文乱码
description: 刚接触MapReduce编程时，大多数同学都会遇到中文乱码问题。在hadoop中默认都以UTF-8来进行文件处理。所以如果我们上传到HDFS上的数据文件不是UTF-8格式，则在运行MapReduce时就会出现乱码，下面我将提供3种中文乱码的解决方案：</br>1、修改数据文件</br>2、编写文件上传客户端程序</br>3、在Map输入时对Text转码
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-12-14 14:30:00
---

## 背景

刚接触MapReduce编程时，大多数同学都会遇到中文乱码问题。在hadoop中默认都以UTF-8来进行文件处理。所以如果我们上传到HDFS上的数据文件不是UTF-8格式，则在运行MapReduce时就会出现乱码，下面我将提供3种中文乱码的解决方案：
1、修改数据文件
2、编写文件上传客户端程序
3、在Map输入时对Text转码

## 修改数据文件
如果是测试程序，文件小，使用这种方式那是最简单不过了。如果是生产程序：**要么让数据提供方那边做处理，要么就只能使用后两种方法。**

修改文件编码，这里以Notepad++作为例子：点击程序左上角“格式(M)”按钮，选择“转换为UTF-8无BOM编码格式”，保存即可。
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-mapreduce-zwlm.png)

## 编写上传客户端

使用这种方式，可以更加灵活，更加快速的的上传文件。我们可以在文件上传时进行数据的格式转换，并发上传，压缩等处理（<font color="red">如果文件很大，数据量大建议采用第三种方法[在Map输入时对Text转码](#u5728Map_u8F93_u5165_u65F6_u5BF9Text_u8F6C_u7801)</font>）。这里只写编码转换代码，如不知道怎么上传HDFS可参考我另外一篇博客[《hadoop-HDFS编程》](/2015/12/11/hadoop-HDFS%E7%BC%96%E7%A8%8B/)。

```
public byte[] decodeByteArray(byte[] value, String encodeCharset, String decodeCharset) throws UnsupportedEncodingException {
    if (isNull(encodeCharset) || isNull(decodeCharset)) {
        return value;
    }
    return new String(new String(new String(value).getBytes(encodeCharset), decodeCharset)).getBytes('UTF-8');
}
```

一行一行读取数据，将读取到的数据进行decodeByteArray编码转换即可。**也可以多行读取，自定义readLine即可。**

## 在Map输入时对Text转码

采用这种方式，**可以高效的利用集群分布式处理**，在数据读取的时候进行转换。**但是需要固定编码格式**：
```
public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {

    /**
     * LongWritable, IntWritable, Text 均是 Hadoop 中实现的用于封装 Java
     * 数据类型的类，这些类实现了WritableComparable接口，
     * 都能够被串行化从而便于在分布式环境中进行数据交换，你可以将它们分别视为long,int,String 的替代品。
     */
    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();// Text 实现了BinaryComparable类可以作为key值

    /**
     * Mapper接口中的map方法： void map(K1 key, V1 value, OutputCollector
     * <K2,V2> output, Reporter reporter) 映射一个单个的输入k/v对到一个中间的k/v对
     * 输出对不需要和输入对是相同的类型，输入对可以映射到0个或多个输出对。
     * OutputCollector接口：收集Mapper和Reducer输出的<k,v>对。
     * OutputCollector接口的collect(k, v)方法:增加一个(k,v)对到output
     */

    public void map(Object key, Text value, Context context) throws IOException, InterruptedException {

        String decode_value = new String(value.getBytes(), "GBK");
        StringTokenizer itr = new StringTokenizer(decode_value);// 得到什么值
        System.out.println("value什么东西 ： " + decode_value);
        System.out.println("key什么东西 ： " + key.toString());

        while (itr.hasMoreTokens()) {
            word.set(itr.nextToken());

            context.write(word, one);
        }
    }
}
```

注：使用new String的形式来进行编码转换，代码中的“GBK”得改成你自己的文件编码。
```
 new String(value.getBytes(), "GBK"));
```