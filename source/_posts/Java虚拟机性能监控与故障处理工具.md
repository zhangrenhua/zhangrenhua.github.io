---
title: Java虚拟机性能监控与故障处理工具
description: 各一个系统定位问题的时候，知识、经验是关键基础，数据是依据，工具是运用知识处理数据的手段。这里说的数据包括：运行日志、异常堆栈、GC日志、线程快照、堆转储文件等。经常使用适当的虚拟机监控和分析的工具可以加快我们分析数据、定位解决问题的速度。</br><p>Sun JDK监控和故障处理工具表：</p><table><thead><tr><th>名称</th><th>主要作用</th></tr></thead><tbody><tr><td>jps</td><td>显示指定系统内所有的HotSpot虚拟机进程</td></tr><tr><td>jstat</td><td>用于收集HotSpot虚拟机各方面的运行数据</td></tr><tr><td>jinfo</td><td>显示虚拟机配置信息</td></tr><tr><td>jmap</td><td>生成虚拟机的内存转储快照</td></tr><tr><td>jhat</td><td>用于分析heapdump（jmap内存转储）文件，它会建立一个HTTP/HTML服务器，让用户可以在浏览器上查看分析结果</td></tr><tr><td>jstack</td><td>显示虚拟机的线程快照</td></tr></tbody></table>
tags: "jvm"
categories: java
date: 2016年12月10日 19:13:39
---

## 背景


各一个系统定位问题的时候，知识、经验是关键基础，数据是依据，工具是运用知识处理数据的手段。这里说的数据包括：运行日志、异常堆栈、GC日志、线程快照、堆转储文件等。经常使用适当的虚拟机监控和分析的工具可以加快我们分析数据、定位解决问题的速度。

## JDK命令行工具

Java开发人员肯定都知道JDK的bin目录中有"java.exe"、"javac.exe"这两个命令行工具，但并非所有程序员都了解过JDK的bin目录之中其他命令行程序的作用。每逢JDK更新版本之时，bin目录下命令行工具的数量和功能总会不知不觉地增加和增强。bin目录的内容如下图所示：
![jps示例1](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-2.png)

注意 如果读者在工作中需要监控运行于JDK 1.5的虚拟机之上的程序，在程序启动时请添加参数"-Dcom.sun.management.jmxremote"开启JMX管理功能，否则由于部分工具都是基于JMX，它们都将会无法使用，如果被监控程序运行于JDK 1.6的虚拟机之上，那JMX管理默认是开启的，虚拟机启动时无须再添加任何参数。


Sun JDK监控和故障处理工具表：

|名称 | 主要作用
|------|-------
|jps    | 显示指定系统内所有的HotSpot虚拟机进程
|jstat  | 用于收集HotSpot虚拟机各方面的运行数据
|jinfo  | 显示虚拟机配置信息
|jmap   | 生成虚拟机的内存转储快照
|jhat   | 用于分析heapdump（jmap内存转储）文件，它会建立一个HTTP/HTML服务器，让用户可以在浏览器上查看分析结果
|jstack | 显示虚拟机的线程快照


### jps

jps和linux的ps命令类似：可以列出正在运行的虚拟机进程，并显示虚拟机执行的主类（main）函数所在的类、名称以及这些进程的本地虚拟机唯一ID（Local Virtual Machine Identifier,LVMID）。

jps示例1：
![jps示例1](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-1.png)

jps示例2：
![jps示例2](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-2.png)

**jps工具主要选项**

|选项 | 作用
|------|-------
|-q    | 只输出LVMID
|-m    | 输出虚拟机进程启动时传递给主类main()函数的参数
|-l    | 输出主类的全名，如果进程执行的是jar包，输出Jar路径
|-v    | 输出虚拟机进程启动时JVM参数

### jstat

jstat(JVM Statistics Monitoring Tool)是用于监视虚拟机各种运行状态信息的命令行工具。它可以本地或远程虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据，在没有GUI图像界面，只提供了纯文本控制台环境的服务器上，它将是运行期定位虚拟机性能问题的首选工具。

示例：`jstat -gc 57118 1000 5`
![jstat示例](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-3.png)

此命令大概意思是，每个1000毫秒统计一次进程号为57118的JVM的GC信息，总共统计5次就退出。

jstat选项option代表着用户希望查询的虚拟机信息，主要分为3类：类装载、垃圾收集、运行期编译状况，具体选项及作用请参考下表：
|选项 | 作用
|------|-------
|-class           | 监视类装载、卸载数量、总空间以及类装载所耗费的时间
|-gc              | 监视Java堆状况，包括Eden区、两个survivor区、老年代、永久代等的容量、已用空间、GC时间合计等信息
|-gccapacity      | 监视内容与-gc基本相同，但输出主要关注Java堆各个区域使用到的最大、最小空间
|-gcutil          | 监视内容与-gc基本相同，但输出主要关注已使用空间占总空间的百分比
|-gccause         | 与-gcutil功能一样，但是会额外输出导致上一次GC产生的原因
|-gcnew           | 监视新生代GC状况
|-gcnewcapacity   | 监视内容与-gcnew基本相同，输出主要关注使用到的最大、最小空间
|-gcold           | 监视老年代GC状况
|-gcoldcapacity   | 监视内容与-gcold基本相同，输出主要关注使用到的最大、最小空间
|-gcpermcapacity  | 输出永久代使用到的最大、最小空间
|-compiler        | 输出JIT编译器编译过的方法、耗时等信息
|-printcompilation| 输出已经被JIT编译的方法


jstat根据不同的选项会列出不同的列，具体列描述参考下表：

|数据列 | 描述 | 支持的jstat选项
|------|-------|-------
|SOC   | Survivor0的当前容量 | -gc </br> -gccapacity </br> -gcnew </br> -gcnewcapacity
|S1C   | S1的当前容量        | -gc </br> -gccapacity </br> -gcnew </br> -gcnewcapacity
|S0U   | S0的使用量          | -gc </br> -gcnew
|S1U   | S1的使用量          | -gc </br> -gcnew
|EC    | Eden区的当前容量    | -gc </br> -gccapacity </br> -gcnew </br> -gcnewcapacity
|EU    | Eden区的使用量      | -gc </br> -gcnew
|OC    | old区的当前容量     | -gc </br> -gccapacity </br> -gcnew </br> -gcnewcapacity
|OU    | old区的使用量       | -gc </br> -gcnew
|PC    | 方法区的当前容量     | -gc </br> -gccapacity </br> -gcold </br> -gcoldcapacity </br> -gcpermcapacity
|PU    | 方法区的使用量      | -gc </br> -gcold
|YGC   | Young GC次数       | -gc </br> -gccapacity </br> -gcnew </br> -gcnewcapacity </br> -gcold </br> -gcoldcapacity </br> -gcpermcapacity </br> -gcutil </br> -gccause
|YGCT  | Young GC累计耗时   | -gc </br> -gcnew </br> -gcutil </br> -gccause
|FGC   | Full GC次数        |  -gc </br> -gccapacity </br> -gcnew </br> -gcnewcapacity </br> -gcold </br> -gcoldcapacity </br> -gcpermcapacity </br> -gcutil </br> -gccause
|FGCT  | Full GC积累耗时    | -gc </br> -gcold </br> -gcoldcapacity </br> -gcpermcapacity </br> -gcutil </br> -gccause
|GCT   | GC总的积累耗时     |	-gc </br> -gcold </br> -gcoldcapacity </br> -gccapacity </br> -gcpermcapacity </br> -gcutil </br> -gccause
|NGCMN | 新生代最小容量     | -gccapacity </br> -gcnewcapacity
|NGCMX | 新生代最大容量     | -gccapacity </br> -gcnewcapacity
|NGC   | 新生代当前容量     | -gccapacity </br> -gcnewcapacity
|OGCMN | 老年代最小容量     | -gccapacity </br> -gcoldcapacity
|OGCMX | 老年代最大容量     | -gccapacity </br> -gcoldcapacity
|OGC   | 老年代当前容量     | -gccapacity </br> -gcoldcapacity
|PGCMN | 方法区最小容量     | -gccapacity </br> -gcpermcapacity
|PGCMX | 方法区最大容量     | -gccapacity </br> -gcpermcapacity
|PGC   | 方法区当前容量     | -gccapacity </br> -gcpermcapacity
|PU    | 方法区使用量       | -gccapacity </br> -gcold
|LGCC  | 上一起GC发生的原因 | -gccapacity
|GCC   | 当前GC发生的原因   | -gccapacity
|TT    | 存活阀值，如果对象在新生代移动次数超过此阀值，则会被移到老年代  | -gcnew
|MTT   | 最大存活阀值，如果对象在新生代移动次数超过此阀值，则会被移到老年代 | -gcnew
|DSS   | survivor区的理想容量 | -gcnewr


### jinfo

jinfo(Configguration Info for Java)的作用是实时地查看和调整虚拟机各项参数。使用jps命令的-v参数可以查看虚拟机启动时显式指定的参数列表，但如果想知道未被显式指定的参数的系统默认值，就只能使用jinfo的-flag选项进行查询了（jdk1.6或可以使用：-XX:+PrintFlagsFinal）。


查看JVM启动参数：jinfo -flags 20241
![jinfo示例1](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-5.png)

查看XX启动参数：jinfo -flag CMSInitiatingOccupancyFraction 20241
![jinfo示例2](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-4.png)
可以通过jinfo -flag name=value 来修改可写属性；

查看System.getProperties()属性：jinfo -sysprops 20241
![jinfo示例3](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-6.png)

jinfo -help:
![jinfo示例3](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-7.png)

### jmap

jmap(Memory Map for Java)命令用于生产堆转储快照（一般称为heapdump或dump文件）。如果不使用jmap命令，要想获取java堆转储快照，还有一些比较“暴力”的手段，通过-XX:+HeapDumpOnOutOfMemoryError参数，可以让虚拟机在OOM异常出现之后自动生成dump文件，通过-XX:+HeapDumpOnCtrlBreak参数则可以使用[Ctrl + Break]键让虚拟机生成dump文件，又或者linux系统下通过kill -3命令发送进程退出信号，也能拿到dump文件。

jmap的作用并不仅仅是为了获取dump文件，它还可以查询finalize执行队列、Java堆和永久代的详细信息，如空间使用率、当前用的是哪种收集器等。

和jinfo命令一样，jmap有不少功能在windows平台下都是受限的，除了生成dump文件的-dump选项和用于查看每个类的实例、空间占用统计的-histo选项在所有操作系统提供之外，其余选项都只能在Linux/Solaris下使用。

**jmap工具选项表：**

|选项 | 描述
|------|-------
|-dump          | 生成java堆转储快照。格式为：-dump:[live,]format=b,file=<filename>，其中live子参数说明是否只dump出存活的对象
|-finalizerinfo | 显示在F-Queue中等待Finalizer线程执行finalize方法的对象。只在linux/Solaris中有效
|-heap          | 显示Java堆详细信息，如使用哪种回收器、参数配置、分代状况等。只在linux/Solaris中有效
|-histo         | 显示堆中对象统计信息，包括类、示例数量、合计容量
|-permstat      | 以Classloader为统计口径显示永久代内存状态。只在linux/Solaris中有效
|-F             | 当虚拟机进程堆-dump选项没有响应时，可使用这个选项强制生成dump快照。只在linux/Solaris中有效

**示例**

生成dump快照文件，2826为虚拟机进程号：jmap -dump:format=b,file=dump 2826
![jmap示例1](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-8.png)


### jhat

Sum JDk提供jhat（JVM Heap Analysis Tool）命令与jmap搭配使用，来分析jmap生成的堆转储快照。jhat内置了一个微型的HTTP/HTML服务器，生成dump文件的分析结果后，可以在浏览器中查看。在工作中一般不会直接使用jhat命令来分析dump文件，一般会使用比较高级的如：Eclipse Memory Analyzer、IBM HeapAnalyzer等工具来分析。

**示例**

jhat分析dump文件：jhat dump
```
Reading from dump...
Dump file created Sat Dec 10 19:42:37 CST 2016
Snapshot read, resolving...
Resolving 1863743 objects...
....
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```

屏幕显示“Server is ready.”的提示后，使用浏览器打开http://localhost:7000/
![jhat示例](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-9.png)

分析结果默认是已包围单位进行分组显示，分析内存泄漏问题主要会使用其中的“Heap Histogram”（与jmap-histo功能一样）与OQL，前者可以找到内存中总容量最大的对象，后者是标准的对象查询语言，使用类似SQL的语法对内存中的对象进行查询统计。


IBM HeapAnalyzer示例：
![IBM HeapAnalyzer](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-10.png)


### jstack

jstack(Stack Trace For Java)命令用于生成虚拟机当前时刻的线程快照（一般称为threaddump或者javacore文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的常见原因。线程出现停顿的时候通过jstack来查询各个线程的堆栈，就可以知道没有响应的线程到底在后台做些什么事情，或者等待着什么资源。

jstack工具主要选项：

|选项 | 作用
|------|-------
|-F    | 当正常输出的请求不被响应时，强制输出线程堆栈
|-l    | 除堆栈外，显示关于锁的附加信息
|-m    | 如果调用到本地方法的话，可以显示C/C++的堆栈


**示例**

查看线程堆栈：jstack -l 2826
![jstack示例1](http://7xoqbc.com1.z0.glb.clouddn.com/jdk-tools-11.png)


<font color="red">在JDk1.5中，java.lang.Thread类新增了一个getAllStackTraces()方法用于获取虚拟机中所有线程的StackTraceElement对象。使用这个方法可以通过简单的几行代码就完成jstack的大部分功能，在时间项目中不妨调用这个方法做个管理员界面，可以随时使用浏览器来查看线程堆栈。</font>
```
<!-- JSP示例代码 -->
＜%@page import="java.util.Map"%＞

＜html＞

＜head＞

＜title＞服务器线程信息＜/title＞

＜/head＞

＜body＞

＜pre＞
＜%
for（Map.Entry＜Thread,StackTraceElement[]＞stackTrace:Thread.getAllStackTraces().entrySet()）{

Thread thread=（Thread）stackTrace.getKey();

StackTraceElement[] stack=（StackTraceElement[]）stackTrace.getValue()；

if（thread.equals（Thread.currentThread()））{

continue;
}

System.out.print（"\n线程:"+thread.getName()+"\n"）;

for（StackTraceElement element:stack）{

System.out.print（"\t"+element+
}
}
%＞
＜/pre＞
＜/body＞
＜/html＞
```

## JDK可视化工具





## 参考

- http://www.cnblogs.com/zhangxiaoguang/p/5792468.html
- http://www.manongjc.com/article/451.html
