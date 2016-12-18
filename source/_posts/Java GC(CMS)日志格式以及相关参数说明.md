---
title: Java GC(CMS)日志格式以及相关参数说明
description: 阅读GC日志是处理Java虚拟机内存问题的基本技能，它只是一些人为确定的规则，没有太多技术含量。</br>每种收集器的日志形式都是由它们自身的实现所决定的，换而言之，每个收集器的日志格式都可以不一样。但虚拟机设计者为了方便用户阅读，将各个收集器的日志都维持一定的共性。</br>本文通过示例来描述gc日志的基本格式，并整理了Java虚拟机的一些常用参数。
tags: "jvm"
categories: java
date: 2016年11月30日 21:43:25
---

## 背景

阅读GC日志是处理Java虚拟机内存问题的基本技能，它只是一些人为确定的规则，没有太多技术含量。

每种收集器的日志形式都是由它们自身的实现所决定的，换而言之，每个收集器的日志格式都可以不一样。但虚拟机设计者为了方便用户阅读，将各个收集器的日志都维持一定的共性，例如以下两段典型的GC日志：
```
4.065: [GC4.065: [ParNew: 279616K->15816K(314560K), 0.0592280 secs] 279616K->15816K(1013632K), 0.0595130 secs] [Times: user=0.72 sys=0.39, real=0.06 secs]]
[Full GC 283.736:[ParNew:261599K-＞261599K（261952K），0.0000288 secs]
```

**第一行gc日志：**
`4.065:` 代表GC发生的时间，这个值是Java虚拟机启动的以来经过的秒数。
`[GC4.065: [ParNew:` 代表此次是`ParNew`新生代在进行回收
`279616K->15816K(314560K)` “GC前该内存区域已使用容量” -> “GC后该内存区域已使用容量” （该内存区域总容量）
`0.0592280 secs` 该区域GC所占用时间
`[Times: user=0.72 sys=0.39, real=0.06 secs]` 这里面的user、sys和real与Linux的time命令所输出的时间含义一致，分别代表用户消耗的CPU时间、内核态消耗的CPU事件和操作从开始到结束所经过的墙钟时间（Wall Clock Time）。CPU时间与墙钟时间的区别是，墙钟时间包括各种非运算的等待耗时，例如等待磁盘I/O、等待线程阻塞，而CPU时间不包括这些耗时，但当系统有多CPU或者多核的话，多线程操作会叠加这些CPU时间，所以读者看到user或sys时间超过real时间是完全正常的。

**第二行gc日志：**
`[Full GC（System）` 是由System.gc()方法所触发的收集，full说明这次垃圾收集是停顿类型“stop-the-world”
`[Full GC 283.736:[ParNew:` 在新生代收集器ParNew的日志也会出现[Full GC，折衣板是因为出现了分配担保失败之类的问题，所以才导致STW)


## 垃圾收集相关的常用参数

|参数 | 描述
|------|-------
| UseSerialGC                     |  虚拟机运行在Client 模式下的默认值，打开此开关后，使用Serial + Serial Old 的收集器组合进行内存回收
| UseParNewGC                     |  打开此开关后，使用ParNew + Serial Old 的收集器组合进行内存回收
| UseConcMarkSweepGC              |  打开此开关后，使用ParNew + CMS + Serial Old 的收集器组合进行内存回收。Serial Old 收集器将作为CMS 收集器出现Concurrent Mode Failure失败后的后备收集器使用
| UseParallelGC                   |  虚拟机运行在Server 模式下的默认值，打开此开关后，使用Parallel Scavenge + Serial Old（PS MarkSweep）的收集器组合进行内存回收
| UseParallelOldGC                |  打开此开关后，使用Parallel Scavenge + Parallel Old 的收集器组合进行内存回收
| SurvivorRatio                   |  新生代中Eden 区域与Survivor 区域的容量比值， 默认为8， 代表Eden ：Survivor=8∶1:1，Survivor分S0和S1
| NewRatio                        |  -XX:NewRatio=4表示年轻代与年老代所占比值为1:4
| PretenureSizeThreshold          |  直接晋升到老年代的对象大小，设置这个参数后，大于这个参数的对象将直接在老年代分配
| MaxTenuringThreshold            |  晋升到老年代的对象年龄。每个对象在坚持过一次Minor GC 之后，年龄就加1，当超过这个参数值时就进入老年代
| UseAdaptiveSizePolicy           |  动态调整Java 堆中各个区域的大小以及进入老年代的年龄
| HandlePromotionFailure          |  是否允许分配担保失败，即老年代的剩余空间不足以应付新生代的整个Eden 和Survivor 区的所有对象都存活的极端情况
| ParallelGCThreads               |  设置并行GC 时进行内存回收的线程数
| GCTimeRatio                     |  GC 时间占总时间的比率，默认值为99，即允许1% 的GC 时间。仅在使用Parallel Scavenge 收集器时生效
| MaxGCPauseMillis                |  设置GC 的最大停顿时间。仅在使用Parallel Scavenge 收集器时生效
| CMSInitiatingOccupancyFraction  |  设置CMS 收集器在老年代空间被使用多少后触发垃圾收集。默认值为68%，仅在使用CMS 收集器时生效
| UseCMSCompactAtFullCollection   |  设置CMS 收集器在完成垃圾收集后是否要进行一次内存碎片整理。仅在使用CMS 收集器时生效
| CMSFullGCsBeforeCompaction      |  设置CMS 收集器在进行若干次垃圾收集后再启动一次内存碎片整理。仅在使用CMS 收集器时生效



### 功能选项

|参数 | 默认值/限制 | 说明
|------|-------|-------
|-XX:-AllowUserSignalHandlers | 限于Linux和Solaris，默认不启用                                                |允许为java进程安装信号处理器,信号处理参见类:sun.misc.Signal, sun.misc.SignalHandler
|-XX:+DisableExplicitGC       | 默认启用                                                                      |禁止在运行期显式地调用System.gc()
|-XX:+FailOverToOldVerifier   | Java6新引入选项，默认启用                                                     |如果新的Class校验器检查失败，则使用老的校验器(失败原因:因为JDK6最高向下兼容到JDK1.2，而JDK1.2的class info 与JDK6的info存在较大的差异，所以新校验器可能会出现校验失败的情况)
|-XX:+HandlePromotionFailure  | java5以前是默认不启用，java6默认启用                                          |关闭新生代收集担保
|-XX:+MaxFDLimit              | 限于Solaris，默认启用                                                         |设置java进程可用文件描述符为操作系统允许的最大值。
|-XX:PreBlockSpin=10          | -XX:+UseSpinning 必须先启用，对于java6来说已经默认启用了，这里默认自旋10次    |控制多线程自旋锁优化的自旋次数
|-XX:-RelaxAccessControlCheck | 默认不启用                                                                    |在Class校验器中，放松对访问控制的检查,作用与reflection里的setAccessible类似
|-XX:+ScavengeBeforeFullGC    | 默认启用                                                                      |在Full GC前触发一次Minor GC
|-XX:+UseAltSigs              | 限于Solaris，默认启用                                                         |为了防止与其他发送信号的应用程序冲突，允许使用候补信号替代 SIGUSR1和SIGUSR2
|-XX:+UseBoundThreads         | 限于Solaris, 默认启用                                                         |绑定所有的用户线程到内核线程, 减少线程进入饥饿状态（得不到任何cpu time）的次数
|-XX:-UseConcMarkSweepGC      | 默认不启用                                                                    |启用CMS低停顿垃圾收集器,减少FGC的暂停时间
|-XX:+UseGCOverheadLimit      | 默认启用                                                                      |限制GC的运行时间。如果GC耗时过长，就抛OOM
|-XX:+UseLWPSynchronization   | 限于solaris，默认启用                                                         |使用轻量级进程（内核线程）替换线程同步
|-XX:-UseParallelGC           | -server时启用,其他情况下，默认不启用                                          |策略为新生代使用并行清除，年老代使用单线程Mark-Sweep-Compact的垃圾收集器
|-XX:-UseParallelOldGC        | 默认不启用                                                                    |策略为老年代和新生代都使用并行清除的垃圾收集器
|-XX:-UseSerialGC             | -client时启用,其他情况下，默认不启用                                          |使用串行垃圾收集器
|-XX:-UseSpinning             | java1.4.2和1.5需要手动启用, java6默认已启用                                   |启用多线程自旋锁优化
|-XX:+UseTLAB                 | 1.4.2以前和使用-client选项时，默认不启用，其余版本默认启用                    |启用线程本地缓存区
|-XX:+UseSplitVerifier        | java5默认不启用, java6默认启用                                                |使用新的Class类型校验器
|-XX:+UseThreadPriorities     | 默认启用                                                                      |使用本地线程的优先级
|-XX:+UseVMInterruptibleIO    | 限于solaris，默认启用                                                         |在solaris中，允许运行时中断线程


### 性能选项

|参数 | 默认值/限制 | 说明
|------|-------|-------
|-XX:+AggressiveOpts          | JDK 5 update 6后引入，但需要手动启用, JDK6默认启用                                                                | 启用JVM开发团队最新的调优成果。例如编译优化，偏向锁，并行年老代收集等
|-XX:CompileThreshold=10000   | 1000                                                                                                              | 通过JIT编译器，将方法编译成机器码的触发阀值，可以理解为调用方法的次数，例如调1000次，将方法编译为机器码
|-XX:LargePageSizeInBytes=4m  | 默认4m, amd64位：2m                                                                                               | 设置堆内存的内存页大小
|-XX:MaxHeapFreeRatio=70      | 70                                                                                                                | GC后，如果发现空闲堆内存占到整个预估上限值的70%，则收缩预估上限值
|-XX:MaxNewSize=size          | 1.3.1 Sparc: 32m, 1.3.1 x86: 2.5m                                                                                 | 新生代占整个堆内存的最大值
|-XX:MaxPermSize=64m          | 5.0以后: 64 bit VMs会增大预设值的30%, 1.4 amd64: 96m, 1.3.1 -client: 32m, 其他默认 64m                            | Perm（俗称方法区）占整个堆内存的最大值
|-XX:MinHeapFreeRatio=40      | 40                                                                                                                | GC后，如果发现空闲堆内存占到整个预估上限值的40%，则增大上限值
|-XX:NewRatio=2               | Sparc -client: 8, x86 -server: 8, x86 -client: 12, -client: 4 (1.3),                                              | 新生代和年老代的堆内存占用比例, 例如2表示新生代占年老代的1/2，占整个堆内存的1/3
|                             | 8 (1.3.1+), x86: 12, 其他默认 2                                                                                   |
|-XX:NewSize=2.125m           | 5.0以后: 64 bit Vms 会增大预设值的30%, x86: 1m, x86, 5.0以后: 640k, 其他默认 2.125m                               | 新生代预估上限的默认值
|-XX:ReservedCodeCacheSize=32m| Solaris 64-bit, amd64, -server x86: 48m, 1.5.0_06之前, Solaris 64-bit amd64: 1024m, 其他默认 32m                  | 设置代码缓存的最大值，编译时用
|-XX:SurvivorRatio=8          | Solaris amd64: 6, Sparc in 1.3.1: 25, Solaris platforms 5.0以前: 32, 其他默认 8                                   | Eden与Survivor的占用比例。例如8表示，一个survivor区占用 1/8 的Eden内存，即1/10的新生代内存，为什么不是1/9？
|                             |                                                                                                                   | 因为我们的新生代有2个survivor，即S0和S1。所以survivor总共是占用新生代内存的 2/10，Eden与新生代的占比则为 8/10
|-XX:TargetSurvivorRatio=50   | 50                                                                                                                | 实际使用的survivor空间大小占比。默认是50%，最高90%
|-XX:ThreadStackSize=512      | Sparc: 512, Solaris x86: 320 (5.0以前 256), Sparc 64 bit: 1024, Linux amd64: 1024 (5.0 以前 0), 其他默认 512.     | 线程堆栈大小
|-XX:+UseBiasedLocking        | JDK 5 update 6后引入，但需要手动启用, JDK6默认启用                                                                | 启用偏向锁
|-XX:+UseFastAccessorMethods  | 默认启用                                                                                                          | 优化原始类型的getter方法性能(get/set:Primitive Type)
|-XX:-UseISM                  | 默认启用                                                                                                          | 启用solaris的ISM
|-XX:+UseLargePages           | JDK 5 update 5后引入，但需要手动启用, JDK6默认启用                                                                | 启用大内存分页
|-XX:+UseMPSS                 | 1.4.1 之前: 不启用, 其余版本默认启用                                                                              | 启用solaris的MPSS，不能与ISM同时使用
|-XX:+UseStringCache          | 默认开启                                                                                                          | 启用缓存常用的字符串。
|-XX:AllocatePrefetchLines=1  | 1                                                                                                                 | Number of cache lines to load after the last object allocation using prefetch instructions generated in JIT compiled code. Default values are 1 if the last allocated object was an instance and 3 if it was an array.
|-XX:AllocatePrefetchStyle=1  | 1                                                                                                                 | Generated code style for prefetch instructions. 0 – no prefetch instructions are generate*d*, 1 – execute prefetch instructions after each allocation, 2 – use TLAB allocation watermark pointer to gate when prefetch instructions are executed.
|-XX:+UseCompressedStrings    | Java 6 update 21有一选项                                                                                          | 其中，对于不需要16位字符的字符串，可以使用byte[] 而非char[]。对于许多应用，这可以节省内存，但速度较慢（5％-10％）
|-XX:+OptimizeStringConcat    | 在Java 6更新20中引入                                                                                              | 优化字符串连接操作在可能的情况下

### G1垃圾收集器选项

|参数 | 说明
|------|-------
|-XX:+UseG1GC                         | 使用Garbage First（G1）收集器
|-XX:MaxGCPauseMillis=n               | 为所需的最长暂停时间设置目标值。默认值是 200 毫秒。指定的值不适用于您的堆大小。
|-XX:InitiatingHeapOccupancyPercent=n | 设置触发标记周期的 Java 堆占用率阈值。默认占用率是整个 Java 堆的 45%。
|-XX:NewRatio=n                       | 旧/新生代空间大小比率。默认值为2。
|-XX:SurvivorRatio=n                  | eden / survivor空间大小比例。默认值为8。
|-XX:MaxTenuringThreshold=n           | 对象能经历多少次Minor GC才晋升到旧生代。默认值为15。
|-XX:ParallelGCThreads=n              | "设置 STW 工作线程数的值。将 n 的值设置为逻辑处理器的数量。n 的值与逻辑处理器的数量相同，最多为 8。
如果逻辑处理器不止八个，则将 n 的值设置为逻辑处理器数的 5/8 左右。这适用于大多数情况，除非是较大的 SPARC 系统，其中 n 的值可以是逻辑处理器数的 5/16 左右。"
|-XX:ConcGCThreads=n                  | 设置并行标记的线程数。将 n 设置为并行垃圾回收线程数 (ParallelGCThreads) 的 1/4 左右。
|-XX:G1ReservePercent=n               | 设置作为空闲空间的预留内存百分比，以降低目标空间溢出的风险。默认值是 10%。增加或减少百分比时，请确保对总的 Java 堆调整相同的量。Java HotSpot VM build 23 中没有此设置。
|-XX:G1HeapRegionSize=n               | 设置的 G1 区域的大小。值是 2 的幂，范围是 1 MB 到 32 MB 之间。目标是根据最小的 Java 堆大小划分出约 2048 个区域。


参考文档：
- G1垃圾收集器参数调优参考： http://www.oracle.com/technetwork/cn/articles/java/g1gc-1984535-zhs.html
- oracle官网 http://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html#G1Options
