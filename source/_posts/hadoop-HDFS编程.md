---
title: HDFS编程
description: Hadoop有一个抽象的文件系统概念，HDFS只是其中的一个实现。Java抽象类org.apache.hadoop.fs.FileSystem定义了Hadoop中的一个文件系统接口，并且该抽象类有几个具体实现：</br><img alt="Alt text" src="http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hdfs-filesystem-list.jpg"></br>Hadoop使用java写的，通过Java API可以调用所有Hadoop文件系统的交互操作。例如，文件系统的命令解释器就是一个Java应用，它使用Java的FileSystem类来提供文件系统操作。其他一些接口还有Thrift，为C语言提供的libhdfs库。这些接口通常与HDFS一同使用，因为Hadoop中的其他文件系统一般都有访问基本文件系统的工具（对于Ftp，有FTP客户端；对于S3，有S3工具，等等），但它们大多数都能和任意一个Hadoop文件系统写作。</br>本文主要通过示例讲解HDFS API中的一些常用方法（采用jdk1.7+hadoop2.6，语法支持2.x）。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-12-11 20:31:00
---

## 背景
Hadoop有一个抽象的文件系统概念，HDFS只是其中的一个实现。Java抽象类org.apache.hadoop.fs.FileSystem定义了Hadoop中的一个文件系统接口，并且该抽象类有几个具体实现：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hdfs-filesystem-list.jpg)

Hadoop使用java写的，通过Java API可以调用所有Hadoop文件系统的交互操作。例如，文件系统的命令解释器就是一个Java应用，它使用Java的FileSystem类来提供文件系统操作。其他一些接口还有Thrift，为C语言提供的libhdfs库。这些接口通常与HDFS一同使用，因为Hadoop中的其他文件系统一般都有访问基本文件系统的工具（对于Ftp，有FTP客户端；对于S3，有S3工具，等等），但它们大多数都能和任意一个Hadoop文件系统写作。

本文主要通过示例讲解HDFS API中的一些常用方法（采用jdk1.7+hadoop2.6，语法支持2.x）。

## 示例

```
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.BlockLocation;
import org.apache.hadoop.fs.FSDataOutputStream;
import org.apache.hadoop.fs.FileStatus;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.hdfs.DistributedFileSystem;
import org.apache.hadoop.hdfs.protocol.DatanodeInfo;

public class Demo {

    public static void main(String[] args) throws IOException {

        // 创建conf
        Configuration conf = new Configuration();
        // 根据配置文件信息，创建出不同的实例（hdfs、ftp、local、http等）
        FileSystem fs = FileSystem.get(conf);

        createFile(fs, "/tmp/0000.txt");
    }

    /**
     * 
     * @@Description: 创建文件或者文件夹
     * @param fs
     * @param path
     * @throws IOException
     */
    public static void createFile(FileSystem fs, String path) throws IOException {
        Path f = new Path(path);
        fs.create(f).close();
    }

    /**
     * 
     * @@Description: 删除文件或递归删除文件夹
     * @param fs
     * @param path
     * @param recursive
     * @return
     * @throws IOException
     */
    public static boolean deleteFile(FileSystem fs, String path, boolean recursive) throws IOException {
        Path f = new Path(path);
        return fs.delete(f, recursive);
    }

    /**
     * 
     * @@Description:读取文件返回string
     * @param fs
     * @param path
     * @return
     * @throws IOException
     */
    public static String readDFSFileToString(FileSystem fs, String path) throws IOException {
        Path f = new Path(path);
        if (!fs.exists(f)) {
            return null;
        }

        String str = null;
        StringBuilder sb = new StringBuilder(1024);
        try (InputStream in = fs.open(f); BufferedReader bf = new BufferedReader(new InputStreamReader(in));) {

            long time = System.currentTimeMillis();
            while ((str = bf.readLine()) != null) {
                sb.append(str);
                sb.append("\n");
            }
            System.out.println(System.currentTimeMillis() - time);
            return sb.toString();
        }
    }

    /**
     * 
     * @@Description:将string写出到hdfs，覆盖
     * @param fs
     * @param path
     * @param string
     * @throws IOException
     */
    public static void writeStringToDFSFile(FileSystem fs, String path, String string) throws IOException {
        Path f = new Path(path);

        try (FSDataOutputStream os = fs.create(f, true);) {
            os.writeBytes(string);
        }
    }

    /**
     * 
     * @@Description:上传本地文件至hdfs
     * @param fs
     * @param filePath
     * @param uploadPath
     * @throws IOException
     */
    public static void upload(FileSystem fs, String filePath, String uploadPath) throws IOException {
        Path src = new Path(filePath);
        Path dst = new Path(uploadPath);

        fs.copyFromLocalFile(src, dst);
        System.out.println("upload to" + fs.getConf().get("fs.default.name"));

        FileStatus files[] = fs.listStatus(dst);

        for (FileStatus file : files) {
            System.out.println(file.getPath());
        }

    }

    /**
     * 
     * @@Description:创建文件夹
     * @param fs
     * @param dir
     * @return
     * @throws IOException
     */
    public static boolean mkdir(FileSystem fs, String dir) throws IOException {
        Path path = new Path(dir);
        return fs.mkdirs(path);
    }

    /**
     * 
     * @@Description: 删除文件夹
     * @param fs
     * @param dir
     * @return
     * @throws IOException
     */
    public static boolean deleteDir(FileSystem fs, String dir) throws IOException {
        Path path = new Path(dir);
        return fs.delete(path, true);
    }

    /**
     * 
     * @@Description: move
     * @param fs
     * @param oldPath
     * @param newPath
     * @return
     * @throws IOException
     */
    public static boolean rename(FileSystem fs, String oldPath, String newPath) throws IOException {
        Path old = new Path(oldPath);
        Path nw = new Path(newPath);
        return fs.rename(old, nw);
    }

    /**
     * 
     * @@Description: 获取文件最后更新时间
     * @param fs
     * @param filePath
     * @throws IOException
     */
    public static void lastTime(FileSystem fs, String filePath) throws IOException {
        Path path = new Path(filePath);
        FileStatus fileStatus = fs.getFileStatus(path);
        System.out.println("Modification time is：" + fileStatus.getModificationTime());

    }

    /**
     * 
     * @@Description: 查看block分布主机
     * @param fs
     * @param filePath
     * @throws IOException
     */
    public static void fileLoc(FileSystem fs, String filePath) throws IOException {
        Path path = new Path(filePath);
        FileStatus filestatus = fs.getFileStatus(path);
        BlockLocation[] blkLocations = fs.getFileBlockLocations(filestatus, 0, filestatus.getLen());

        for (int i = 0; i < blkLocations.length; i++) {
            String[] hosts = blkLocations[i].getHosts();

            for (String str : hosts) {
                System.out.println("block" + i + "location:" + str);
            }

        }
    }

    /**
     * 
     * @@Description:获取所有dataNode节点信息
     * @param fs
     * @throws IOException
     */
    public static void getList(FileSystem fs) throws IOException {
        DatanodeInfo[] dataNodeInfo = ((DistributedFileSystem) fs).getDataNodeStats();
        String[] names = new String[dataNodeInfo.length];

        for (int i = 0; i < dataNodeInfo.length; i++) {
            names[i] = dataNodeInfo[i].getHostName();
            System.out.println("node " + i + " hostname: " + names[i]);
        }
    }

    /**
     * 
     * @@Description: 文件追加 
     * @param fs
     * @throws IOException
     */
    public static void append(FileSystem fs) throws IOException{
        Path f = new Path("/tmp/ab.txt");
        Configuration conf = new Configuration();
        conf .set("dfs.client.block.write.replace-datanode-on-failure.policy" ,"NEVER" );
        conf .set("dfs.client.block.write.replace-datanode-on-failure.enable" ,"true" );
        
        FSDataOutputStream fos = fs.append(f);
        fos.write(" world".getBytes());
        fos.close();
    }

}
```

需要导入jar包如下：
```
commons-cli-1.2.jar
commons-collections-3.2.1.jar
commons-compress-1.4.1.jar
commons-configuration-1.6.jar
commons-lang-2.6.jar
commons-logging-1.1.3.jar
guava-11.0.2.jar
hadoop-auth-2.6.0.jar
hadoop-common-2.6.0.jar
hadoop-hdfs-2.6.0.jar
htrace-core-3.0.4.jar
protobuf-java-2.5.0.jar
slf4j-api-1.7.5.jar
```
上述jar文件皆在HADOOP_HOME/share/hadoop下的common\hdfs包中能找到。

将<font color="red">集群环境中core-site.xml和hdfs-site.xml两个配置文件复制到项目的src目录下，</font>最终项目结构如下：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-hdfs-example.jpg)


确保客户端机器能访问hadoop集群内的所有机器，并配置本地hosts

## Configuration简要概述

Hadoop没有使用java.util.Properties管理配置文件，也没有使用Apache Jakarta Commons Configuration管理配置文件，而是使用了一套独有的配置文件管理系统，并提供自己的API，即使用org.apache.hadoop.conf.Configuration处理配置信息。

Hadoop配置文件的格式：
```
<?xml version="1.0" encoding="UTF-8"?>

<!--Autogenerated by Cloudera Manager -->
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://nameservice1</value>
    </property>

</configuration>
```

Configuration初始化的时候默认加载项目类路径（classes）目录下的两个配置文件core-default.xml, core-site.xml，也可以手动添加其它自定义的符合格式的xml文件：
```
// 默认读取类路径下 core-default.xml, core-site.xml
Configuration conf = new Configuration();

// 手动添加外部xml
conf.addResource(new URL("file:///etc/hadoop/conf/core-site.xml"));
```

可以通过xml中的name属性get获取对应的属性值，并提供类型转换：
```
System.out.println(conf.get("fs.defaultFS"));

// 获取想要的类型值，如不能parse则抛出异常
conf.getInt(name, defaultValue)
conf.getBoolean(name, defaultValue)
conf.getDouble(name, defaultValue)
```

## FileSystem使用技巧

我们在编码时需要注意FileSystem不同子类文件系统之间的可移植性。这是非常有用的，比如说你可以非常方便的直接用同样的代码在你的本地文件系统上进行测试。
```
// 默认读取类路径下 hdfs-default.xml, hdfs-site.xml
FileSystem fs = FileSystem.get(conf);
```
上面代码会根据core-site、hdfs-site.xml两个配置里面的内容来决定创建具体的实现类，如果项目的类路径中有这两个配置文件（当然必须是格式正确并且是集群环境中的配置文件），则会返回DistributedFileSystem来操作HDFS上的文件系统。

也可以通过下面这种方式来显示的创建本地文件系统实现类LocalFileSystem：
```
LocalFileSystem fs = FileSystem.getLocal(conf);
```

通过这样的方式，**我们可能只需要改写一行代码就能操作不同的文件系统了**。这里只是描述如何获取DistributedFileSystem和LocalFileSystem两种实现方法，就如本文一开始就描述的FileSystem的所有实现类，我们一样可以通过类似的方法来获取这些具体实现类去操作这些文件系统(file、hdfs、hftp、hsftp、har、kfs、ftp、s3n、s3)。

<font color="red">Hadoop系统为了保证数据的一致性，会对文件生成相应的校验文件（.crc），并在读写的时候进行校验，确保数据的准确性。</font>
`fileSystem.setWriteChecksum(false);// 写文件，不写.crc`


## 参考资料
《Hadoop权威指南》