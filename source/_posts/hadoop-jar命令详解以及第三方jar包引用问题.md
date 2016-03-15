---
title: hadoop jar命令详解以及第三方jar包引用问题
description: 在学习hadoop时，我们往往会执行hadoop jar命令来运行官方自带的word count例子。这时候我们就应该了解下hadoop jar命令的运行原理，下面我会详细介绍怎么运行hadoop自带的例子。以及我们在以后自己编写MapReduce时引用第三方jar包，需要注意的问题。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-12-05 19:10:00
---

## 背景

在学习hadoop时，我们往往会执行hadoop jar命令来运行官方自带的word count例子。这时候我们就应该了解下hadoop jar命令的运行原理，下面我会详细介绍怎么运行hadoop自带的例子。以及我们在以后自己编写MapReduce时引用第三方jar包，需要注意的问题。

## Hadoop官方例子
在我们hadoop集群搭建好之后，我们就可以运行hadoop自带的例子来验证集群。

1、创建测试文件vi word-count.txt，并写入内容
vi word-count.txt
```
hello renhua.zhang
java python hadoop
```

2、将文件上传至HDFS
```
#创建文件目录
hadoop fs -mkdir -p /user/hadoop/test-examples/

#上传文件至HDFS
hadoop fs -put word-count.txt /user/hadoop/test-examples/
```

3、运行word count
```
hadoop jar hadoop-mapreduce-examples-2.x.jar org.apache.hadoop.examples.WordCount /user/hadoop/test-examples/ /out

#注意hadoop-mapreduce-examples-2.x.jar文件在$HADOOP_HOME/share/hadoop/mapreduce/目录下
```

4、执行完成后，查看结果
```
hadoop fs -cat /out/part-r-00000
```

## hadoop jar命令详解
hadoop jar hadoop-mapreduce-examples-2.x.jar做了什么？
hadoop jar后面的参数又是什么含义？
hadoop这命令为什么能直接运行？

首先我们得了解下操作系统PATH环境变量的用途，我们在部署hadoop的时候已经将$HADOOP_HOME/bin目录配置到了操作系统的PATH环境变量中。所以$HADOOP_HOME/bin目录下的hadoop文件是可以直接运行的。

当我们知道hadoop命名其实是执行了hadoop这个文件脚本之后，我们就来探讨下`hadoop jar`命令到底做了什么，jar这个参数的真正含义，以及是否还有其他参数可以传。

### 使用文本编辑器查看hadoop脚本文件，并搜索Jar关键字
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-jar-cmd.png)

注：<font color="red">hadoop脚本文件默认在$HADOOP_HOME/bin目录下，但是如果用是商业发行版的话(比如：Cloudera)，则文件在vi /usr/lib/hadoop/bin/hadoop</font>

可以看到参数为jar的时候CLASS=org.apache.hadoop.util.RunJar，这个CLASS又是什么东西呢？查看文件的倒数第4行：
```
exec "$JAVA" $JAVA_HEAP_MAX $HADOOP_OPTS $CLASS "$@"
```

看这命令行感觉是不是说明RunJar就是启动类？

### 我们将hadoop文件最后第4行的exec改成echo的然后再执行
vi $HADOOP_HOME/bin/hadoop
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-jar-hadoop.png)

再执行word count:
```
hadoop jar hadoop-mapreduce-examples-2.x.jar org.apache.hadoop.examples.WordCount /user/hadoop/test-examples/ /out

#输出
hadoop jar hadoop-mapreduce-examples-2.x.jar org.apache.hadoop.examples.WordCount /user/hadoop/test-examples/ /out
/usr/java/jdk1.7.0_67/bin/java -Xmx1000m -Dhadoop.log.dir=/usr/lib/hadoop/logs -Dhadoop.log.file=hadoop.log -Dhadoop.home.dir=/usr/lib/hadoop -Dhadoop.id.str= -Dhadoop.root.logger=INFO,console -Djava.library.path=/usr/lib/hadoop/lib/native -Dhadoop.policy.file=hadoop-policy.xml -Djava.net.preferIPv4Stack=true -Dhadoop.security.logger=INFO,NullAppender org.apache.hadoop.util.RunJar hadoop-mapreduce-examples-2.x.jar org.apache.hadoop.examples.WordCount /user/hadoop/test-examples/ /out
```
现在看到这个输出明了吧？其实运行的就是org.apache.hadoop.util.RunJar这个类

### hadoop jar命令各参数含义
hadoop jar各段的含义：
1、hadoop：$HADOOP_HOME/bin下的shell脚本名。
2、jar：hadoop脚本需要的command参数。
3、hadoop-mapreduce-examples-2.x.jar：要执行的jar包在本地文件系统中的完整路径，参递给RunJar类。
4、org.apache.hadoop.examples.WordCount：main方法所在的类，参递给RunJar类。
5、/user/hadoop/test-examples/：传递给WordCount类，作为DFS文件系统的路径，指示输入数据来源。
6、/out：传递给WordCount类，作为DFS文件系统的路径，指示输出数据路径。

### RunJar.java详解，
通过上面对hadoop jar命令解释后，现在我们来进一步了解RunJar做了些什么，
查看hadoop源码RunJar.java如下：
```
/**
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.apache.hadoop.util;

import java.lang.reflect.Array;
import java.lang.reflect.Method;
import java.lang.reflect.InvocationTargetException;
import java.net.URL;
import java.net.URLClassLoader;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.io.File;
import java.util.regex.Pattern;
import java.util.Arrays;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.jar.JarFile;
import java.util.jar.JarEntry;
import java.util.jar.Manifest;

import org.apache.hadoop.classification.InterfaceAudience;
import org.apache.hadoop.classification.InterfaceStability;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileUtil;
import org.apache.hadoop.io.IOUtils;

/** Run a Hadoop job jar. */
@InterfaceAudience.Private
@InterfaceStability.Unstable
public class RunJar {

  /** Pattern that matches any string */
  public static final Pattern MATCH_ANY = Pattern.compile(".*");

  /**
   * Priority of the RunJar shutdown hook.
   */
  public static final int SHUTDOWN_HOOK_PRIORITY = 10;

  /**
   * Unpack a jar file into a directory.
   *
   * This version unpacks all files inside the jar regardless of filename.
   */
  public static void unJar(File jarFile, File toDir) throws IOException {
    unJar(jarFile, toDir, MATCH_ANY);
  }

  /**
   * Unpack matching files from a jar. Entries inside the jar that do
   * not match the given pattern will be skipped.
   *
   * @param jarFile the .jar file to unpack
   * @param toDir the destination directory into which to unpack the jar
   * @param unpackRegex the pattern to match jar entries against
   */
  public static void unJar(File jarFile, File toDir, Pattern unpackRegex)
    throws IOException {
    JarFile jar = new JarFile(jarFile);
    try {
      Enumeration<JarEntry> entries = jar.entries();
      while (entries.hasMoreElements()) {
        JarEntry entry = (JarEntry)entries.nextElement();
        if (!entry.isDirectory() &&
            unpackRegex.matcher(entry.getName()).matches()) {
          InputStream in = jar.getInputStream(entry);
          try {
            File file = new File(toDir, entry.getName());
            ensureDirectory(file.getParentFile());
            OutputStream out = new FileOutputStream(file);
            try {
              IOUtils.copyBytes(in, out, 8192);
            } finally {
              out.close();
            }
          } finally {
            in.close();
          }
        }
      }
    } finally {
      jar.close();
    }
  }

  /**
   * Ensure the existence of a given directory.
   *
   * @throws IOException if it cannot be created and does not already exist
   */
  private static void ensureDirectory(File dir) throws IOException {
    if (!dir.mkdirs() && !dir.isDirectory()) {
      throw new IOException("Mkdirs failed to create " +
                            dir.toString());
    }
  }

  /** Run a Hadoop job jar.  If the main class is not in the jar's manifest,
   * then it must be provided on the command line. */
  public static void main(String[] args) throws Throwable {
    String usage = "RunJar jarFile [mainClass] args...";

    if (args.length < 1) {
      System.err.println(usage);
      System.exit(-1);
    }

    int firstArg = 0;
    String fileName = args[firstArg++];
    File file = new File(fileName);
    if (!file.exists() || !file.isFile()) {
      System.err.println("Not a valid JAR: " + file.getCanonicalPath());
      System.exit(-1);
    }
    String mainClassName = null;

    JarFile jarFile;
    try {
      jarFile = new JarFile(fileName);
    } catch(IOException io) {
      throw new IOException("Error opening job jar: " + fileName)
        .initCause(io);
    }

    Manifest manifest = jarFile.getManifest();
    if (manifest != null) {
      mainClassName = manifest.getMainAttributes().getValue("Main-Class");
    }
    jarFile.close();

    if (mainClassName == null) {
      if (args.length < 2) {
        System.err.println(usage);
        System.exit(-1);
      }
      mainClassName = args[firstArg++];
    }
    mainClassName = mainClassName.replaceAll("/", ".");

    File tmpDir = new File(new Configuration().get("hadoop.tmp.dir"));
    ensureDirectory(tmpDir);

    final File workDir;
    try { 
      workDir = File.createTempFile("hadoop-unjar", "", tmpDir);
    } catch (IOException ioe) {
      // If user has insufficient perms to write to tmpDir, default  
      // "Permission denied" message doesn't specify a filename. 
      System.err.println("Error creating temp dir in hadoop.tmp.dir "
                         + tmpDir + " due to " + ioe.getMessage());
      System.exit(-1);
      return;
    }

    if (!workDir.delete()) {
      System.err.println("Delete failed for " + workDir);
      System.exit(-1);
    }
    ensureDirectory(workDir);

    ShutdownHookManager.get().addShutdownHook(
      new Runnable() {
        @Override
        public void run() {
          FileUtil.fullyDelete(workDir);
        }
      }, SHUTDOWN_HOOK_PRIORITY);


    unJar(file, workDir);

    ArrayList<URL> classPath = new ArrayList<URL>();
    classPath.add(new File(workDir+"/").toURI().toURL());
    classPath.add(file.toURI().toURL());
    classPath.add(new File(workDir, "classes/").toURI().toURL());
    File[] libs = new File(workDir, "lib").listFiles();
    if (libs != null) {
      for (int i = 0; i < libs.length; i++) {
        classPath.add(libs[i].toURI().toURL());
      }
    }
    
    ClassLoader loader =
      new URLClassLoader(classPath.toArray(new URL[0]));

    Thread.currentThread().setContextClassLoader(loader);
    Class<?> mainClass = Class.forName(mainClassName, true, loader);
    Method main = mainClass.getMethod("main", new Class[] {
      Array.newInstance(String.class, 0).getClass()
    });
    String[] newArgs = Arrays.asList(args)
      .subList(firstArg, args.length).toArray(new String[0]);
    try {
      main.invoke(null, new Object[] { newArgs });
    } catch (InvocationTargetException e) {
      throw e.getTargetException();
    }
  }
  
}
```

查看源码的133-156行：
```
    String mainClassName = null;

    JarFile jarFile;
    try {
      jarFile = new JarFile(fileName);
    } catch(IOException io) {
      throw new IOException("Error opening job jar: " + fileName)
        .initCause(io);
    }

    Manifest manifest = jarFile.getManifest();
    if (manifest != null) {
      mainClassName = manifest.getMainAttributes().getValue("Main-Class");
    }
    jarFile.close();

    if (mainClassName == null) {
      if (args.length < 2) {
        System.err.println(usage);
        System.exit(-1);
      }
      mainClassName = args[firstArg++];
    }
    mainClassName = mainClassName.replaceAll("/", ".");

```
首先从jar包META-INF\MANIFEST.MF文件中查找Main-Class属性获取要执行的class类，149-155行如果没有找到则从hadoop jar命令的第2个参数获取，这里也就是org.apache.hadoop.examples.WordCount

查看205-215行，将mainClassName通过java反射获取[mainClassName]类的静态Main方法，执行并注入命令行参数
[mainClassName]：这里指的是org.apache.hadoop.examples.WordCount
```
    Class<?> mainClass = Class.forName(mainClassName, true, loader);
    Method main = mainClass.getMethod("main", new Class[] {
      Array.newInstance(String.class, 0).getClass()
    });
    String[] newArgs = Arrays.asList(args)
      .subList(firstArg, args.length).toArray(new String[0]);
    try {
      main.invoke(null, new Object[] { newArgs });
    } catch (InvocationTargetException e) {
      throw e.getTargetException();
    }
```

说到这里大家应该都懂了，根据word count最终运行的是org.apache.hadoop.examples.WordCount的Main方法。WordCount.java具体做什么我就不讲了。

### 根据Runjar.java来分析第三方jar包引用问题

如果MapReduce中引用了第三方jar文件，运行时会报类似异常【Error: java.lang.ClassNotFoundException:***】。那么我们改怎么解决这个问题呢？
其实有几种解决方案，我下面介绍一种最为优雅的解决方案。

查看190-202行代码：
```
    ArrayList<URL> classPath = new ArrayList<URL>();
    classPath.add(new File(workDir+"/").toURI().toURL());
    classPath.add(file.toURI().toURL());
    classPath.add(new File(workDir, "classes/").toURI().toURL());
    File[] libs = new File(workDir, "lib").listFiles();
    if (libs != null) {
      for (int i = 0; i < libs.length; i++) {
        classPath.add(libs[i].toURI().toURL());
      }
    }

    ClassLoader loader =
      new URLClassLoader(classPath.toArray(new URL[0]));
```

代码中明确写死，将wordDir（运行jar的目录，这里表示hadoop-mapreduce-examples-2.x.jar）下面的classes目录和lib目录下所有.class和.jar文件将会被ClassLoader加载。
<font color="red">所以将第三方jar文件打包到lib目录是最佳的第三方引用解决方案</font>
下面是打包好的一个示例jar文件：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-jar-build.png)

