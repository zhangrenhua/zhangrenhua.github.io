---
title: 实时计算UV《基数估计算法》
description: <br/><br/>在大数据分布式计算的时候，PV(Page View)可以很方便相加合并，但UV(Unique Visitor)不能。<br/><br/>分布式计算的情况下，几百个业务、数十万URL同时统计UV，如果还要分时段统计(每分钟/每5分钟合并/每小时合并/每天合并)，内存的消耗是不可接受的。<br/><br/>这个时候，概率的力量就体现了出来。我们在Probabilistic Data Structures for Web Analytics and Data Mining可以看到，精确的哈希表统计UV和基数计数的内存比较，并不是一个数量级的。基数计数可以让你实现UV的合并，内存消耗极小，并且误差完全在可接受范围内。<br/><br/>UV概念可参考<a href="/2016/06/07/IP、网站浏览量（PV）、访问次数（VV）、独立访客（UV）有什么区别？/">《网站浏览量（PV）、访问次数（VV）、独立访客（UV）区别》</a>具体算法是Adaptive Counting，使用的计算库是stream-2.7.0.jar。
tags: [java]
categories: 网站分析
date: 2016-6-8 10:59:14
---

## 背景

在大数据分布式计算的时候，PV(Page View)可以很方便相加合并，但UV(Unique Visitor)不能。

分布式计算的情况下，几百个业务、数十万URL同时统计UV，如果还要分时段统计(每分钟/每5分钟合并/每小时合并/每天合并)，内存的消耗是不可接受的。

这个时候，概率的力量就体现了出来。我们在Probabilistic Data Structures for Web Analytics and Data Mining可以看到，精确的哈希表统计UV和基数计数的内存比较，并不是一个数量级的。基数计数可以让你实现UV的合并，内存消耗极小，并且误差完全在可接受范围内。

UV概念可参考[《网站浏览量（PV）、访问次数（VV）、独立访客（UV）区别》](/2016/06/07/IP、网站浏览量（PV）、访问次数（VV）、独立访客（UV）有什么区别？/)
具体算法是Adaptive Counting，使用的计算库是stream-2.7.0.jar。


## 基数的计算有两个难点

### 一是不利于实时流计算的实现。

例如我们的一些产品中经常会提供实时UV，也就是从某个时间点开始（例如今天零点）到目前的独立访客数。为了做到这点，需在内存中为每一个UV数值维护一个查找性能高的数据结构（例如B树），这样当实时流中新来一个访问时，能快速查找这个访客是否已经来过，由此确定UV值是增加1还是不变。如果我们要为100万家店铺同时提供这种服务，就要在内存中维护100万个B树，而如果还要分不同来源维度计算UV的话，这个数量还会迅速膨胀。这对我们的服务器计算资源和内存资源都是一个很大的挑战。

### 第二点就是传统的基数计算方法无法有效合并

例如，前一小时和这一小时的UV虽然分别计算出来了，但是要看这两个小时的总UV依然要重新进行一遍复杂的计算。使用bitmap数据结构的方案虽然可以快速合并，但是空间复杂度太高，因为时间段的任意组合数量与时间段数量呈幂级关系，所以不论是B树还是简单的bitmap在大数据面前都不是一个有效的方案。


我们使用的算法主要是Adaptive Counting算法，这个算法出现在 “Fast and accurate traffic matrix measurement using adaptive cardinality counting” 这篇论文里，但是我同时在ccard-lib里也实现了Linear Counting、LogLog Counting和HyperLogLog Counting等常见的基数估计算法。


这些算法是概率算法，就是通过牺牲一定的准确性（但是精度可控，并可以通过数学分析给出控制精度的方法），来大幅节省计算的资源使用。例如我们仅仅使用8k的内存就可以对一个数亿量级的UV进行估计，而误差不超过2%，这比使用B树或原始bitmap要大幅节省内存。同时基数估计算法用到了经过哈希变换的bitmap空间，在大幅节省内存的同时依然可以实现高效合并，这就同时解决了上面提到的两个难点。


## 示例

### 算法包下载

maven:
```
<dependency>
  <groupId>com.clearspring.analytics</groupId>
  <artifactId>stream</artifactId>
  <version>2.7.0</version>
</dependency>
```

github:`https://github.com/chaoslawful/ccard-lib#reference`

### 示例文件

ip1.txt:
```
127.0.0.1
127.0.0.1
127.0.0.5
127.0.0.7
127.0.0.6
127.0.0.2
```

ip2.txt:
```
127.0.0.1
127.0.0.1
127.0.0.3
127.0.0.4
127.0.0.1
127.0.0.2
```


### 示例代码

```
import java.io.BufferedReader;
import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileReader;
import java.io.IOException;

import com.clearspring.analytics.stream.cardinality.AdaptiveCounting;
import com.clearspring.analytics.stream.cardinality.CardinalityMergeException;
import com.clearspring.analytics.stream.cardinality.ICardinality;

public class TestCardinality {

    private static Long count = 0l;

    public static void main(String[] args) throws IOException {
        long start = System.currentTimeMillis();

        // 计算基数
        ICardinality c1 = cardinality("d:/ip1.txt");
        ICardinality c2 = cardinality("d:/ip2.txt");

        System.out.println("c1 unique count:" + c1.cardinality());
        System.out.println("c2 unique count:" + c2.cardinality());

        /*
         * 合并两个数据集
         */
        try {
            System.out.println("c1 c2 merge unique count:" + c1.merge(c2).cardinality());
        } catch (CardinalityMergeException e) {
            e.printStackTrace();
        }

        System.out.println("Total count:" + count);
        System.out.println("Total cost:" + (System.currentTimeMillis() - start) + " ms ..");
        // System.out.println(Runtime.getRuntime().freeMemory());

    }

    /**
     * 
     * 估算uv
     * 
     * @param filePath
     *            文件路径
     * @return
     * @throws FileNotFoundException
     */
    public static ICardinality cardinality(String filePath) throws IOException {

        // 初始化
        ICardinality card = AdaptiveCounting.Builder.obyCount(Integer.MAX_VALUE).build();

        /*
         * 读取文件并添加到card
         */
        File file = new File(filePath);
        BufferedReader reader = null;
        try {
            reader = new BufferedReader(new FileReader(file));
            String tempString = null;
            while ((tempString = reader.readLine()) != null) {
                card.offer(tempString);
                count++;
            }

        } finally {
            if (reader != null) {
                try {
                    reader.close();
                } catch (IOException e1) {
                }
            }
        }
        return card;
    }
```

### 运行结果

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/cardinality-test.png)

## 参考

- http://m.oschina.net/blog/315457