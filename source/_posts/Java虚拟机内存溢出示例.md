---
title: Java虚拟机内存溢出示例
description: 本文主要记录下，Java虚拟机经常会遇到的一些内存溢出异常，并且通过示例代码来演示。本文用以加深对Java虚拟机内存溢出的理解，当我们应用程序出现类似异常时，能快速分析出，是何种原因导致的。</br>本文还添加了堆外内存泄漏的示例代码，堆外内存是值：可以通过API来创建固定大小的内存空间，开发者自行编码控制对象回收。（在jdk1.7中，当full gc时，垃圾收集器也会检查堆外内存，并将无引用的对象回收掉）。<img src="http://7xoqbc.com1.z0.glb.clouddn.com/jvm-error-5.png" alt="堆外内存溢出">
tags: "jvm"
categories: java
date: 2016年11月23日 20:53:24
---

## 背景


本文主要记录下，Java虚拟机经常会遇到的一些内存溢出异常，并且通过示例代码来演示。本文用以加深对Java虚拟机内存溢出的理解，当我们应用程序出现类似异常时，能快速分析出，是何种原因导致的。

本文还添加了堆外内存泄漏的示例代码，堆外内存是值：可以通过API来创建固定大小的内存空间，开发者自行编码控制对象回收。（在jdk1.7中，当full gc时，垃圾收集器也会检查堆外内存，并将无引用的对象回收掉）。

## 堆内存溢出

```
import java.util.ArrayList;
import java.util.List;

/**
 * 堆内存溢出
 *
 * -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 * @author hua
 *
 */
public class YoungGenGC {

	public static void main(String[] args) {

		List<byte[]> list = new ArrayList<>();

		while (true) {
			list.add(new byte[1024 * 1024]);
		}

	}
}
```

注意：请在运行时，加上注释上的参数。

运行结果图：
![堆内存溢出](http://7xoqbc.com1.z0.glb.clouddn.com/jvm-error-1.png)

## 栈内存溢出

```
/**
 * 定义了大量的本地变量，增大此方法帧中本地变量表的长度。结果：抛出StackOverflowError异常时输出的堆栈深度相应缩小。
 *
 * 在单个线程下，无论是由于栈帧太大还是虚拟机栈容量太小，当内存无法分配的时候，虚拟机抛出的都是StackOverflowError异常。
 *
 * -Xss2M
 *
 * @author hua
 *
 */
public class JavaVMStackSOF {

	private int stackLength = 1;

	public void stackLeak() {
		stackLength++;
		stackLeak();
	}

	public static void main(String[] args) {
		JavaVMStackSOF oom = new JavaVMStackSOF();
		try {
			oom.stackLeak();
		} catch (Throwable e) {
			System.out.println("stack length:" + oom.stackLength);
			throw e;
		}
	}
}
```

运行结果图：
![栈内存溢出](http://7xoqbc.com1.z0.glb.clouddn.com/jvm-error-2.png)


## 栈内存溢出（创建线程）

```
/**
 * 建议不要测试
 *
 * @author hua
 *
 */
public class JavaVMStackOOM {

	private void dontStop() {
		while (true) {

		}
	}

	public void stackLeakByThread() {
		while (true) {
			Thread thread = new Thread(new Runnable() {

				@Override
				public void run() {
					dontStop();
				}
			});
			thread.start();
		}
	}

	public static void main(String[] args) {
		JavaVMStackOOM oom = new JavaVMStackOOM();
		oom.stackLeakByThread();
	}
}
```

强烈建议不要测试，可能导致电脑死机处于假死状态。

![栈内存溢出](http://7xoqbc.com1.z0.glb.clouddn.com/jvm-error-3.png)

## 常量池溢出

```
/**
 * 常量池和方法区OOM测试
 *
 * Args:-XX:PermSize=10M -XX:MaxPermSize=10M
 *
 *
 *
 * @author hua
 *
 */
public class RuntimeConstantPoolOOM {

	public static void main(String[] args) {
		List<String> list = new ArrayList<String>();

		/*
		 * 必须在jdk1.6中才有效
		 */
		// 10M的PremSize在integer范围内足够产生OOM
		int i = 0;
		while (true) {
			list.add(String.valueOf(i++).intern());
		}

	}

	public static void internTest() {
		/*
		 * 这段代码在JDK 1.6中运行，会得到两个false，而在JDK
		 * 1.7中运行，会得到一个true和一个false。产生差异的原因是：在JDK
		 * 1.6中，intern()方法会把首次遇到的字符串实例复制到永久区
		 */
		String str1 = new StringBuilder("计算机").append("软件").toString();

		System.out.println(str1.intern() == str1);

		String str2 = new StringBuilder("ja").append("va").toString();

		System.out.println(str2.intern() == str2);
	}
}
```

此代码必须在jdk1.6中才有效，HotSpot JDK1.7已经将常量池，移到堆内存了。

![常量池溢出](http://7xoqbc.com1.z0.glb.clouddn.com/jvm-error-4.png)

## 堆外内存溢出

```
import java.lang.reflect.Field;

import sun.misc.Unsafe;

/**
 *
 * 堆外内存测试
 *
 * Args:-Xmx20M-XX:MaxDirectMemorySize=10M
 *
 *
 * 由DirectMemory导致的内存溢出，一个明显的特征是在Heap
 * Dump文件中不会看见明显的异常，如果读者发现OOM之后Dump文件很小，而程序中又直接或间接使用了NIO，那就可以考虑检查一下是不
 *
 * @author hua
 *
 */
public class DirectMemoryOOM {

	public static void main(String[] args) throws Exception {

		// 可以通过DirectByteBuffer类，来操作
		Field field = Unsafe.class.getDeclaredFields()[0];
		field.setAccessible(true);

		Unsafe unsafe = (Unsafe) field.get(null);
		while (true) {
			unsafe.allocateMemory(1024 * 1024); // 分配内存
		}
	}
}
```

![堆外内存溢出](http://7xoqbc.com1.z0.glb.clouddn.com/jvm-error-5.png)
