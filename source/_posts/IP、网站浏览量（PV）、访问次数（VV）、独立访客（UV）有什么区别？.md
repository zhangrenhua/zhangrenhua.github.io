---
title: IP、网站浏览量（PV）、访问次数（VV）、独立访客（UV）区别以及UV去重
description: <h3>指标之间有什么区别？</h3>PV、UV、VV、独立IP数是网站分析中最基础、最常见的指标。你清楚各指标的具体意义吗？你了解他们之间的区别吗？接下来，让腾讯分析和您一起对各项基础指标进行解读吧。<br/>访问次数（VV）：记录所有访客1天内访问了多少次您的网站，相同的访客有可能多次访问您的网站。<br/>独立访客（UV）：1天内相同访客多次访问网站，只计算为1个独立访客。<br/>网站浏览量（PV）：用户每打开一个页面便记录1次PV<br/>独立IP（IP）：同一IP无论访问了几个页面，独立IP数均为1<h3>如何在Java使用极小的内存完成对超大数据的去重计数，用于实时计算中统计UV？</h3>一直在想如何在 实时计算中 完成对海量数据去重计数的功能，即SELECT COUNT(DISTINCT) 的功能。比如：从每天零点开始，实时计算全站累计用户数（UV），以及某些组合维度上的用户数，这里的用户假设以Cookieid来计。<br/><br/>想想一般的解决办法，在内存中使用HaspMap、HashSet？或者是在Redis中以Cookieid为key？感觉都不合适，在数以亿计用户的业务场景下，内存显然也成了瓶颈。<br/><br/>如果说，实时计算的业务场景中，对UV的计算精度并不要求100%（比如：实时的监测某一网站的PV和UV），那么可以考虑采用基数估计算法来统计。这里有一个Java的实现版本 stream-lib： https://github.com/addthis/stream-lib<br/><br/>采用基数估计算法目的就是为了使用很小的内存，即可完成超大数据的去重计数。号称是只使用几KB的内存，就可以完成对数以条数据的去重计数。但基数估计算法都不是100%精确的，误差在0~2%之间，一般是1%左右。
tags: [java]
categories: 网站分析
date: 2016-6-7 14:13:35
---


### 指标之间有什么区别？

PV、UV、VV、独立IP数是网站分析中最基础、最常见的指标。你清楚各指标的具体意义吗？你了解他们之间的区别吗？接下来，让腾讯分析和您一起对各项基础指标进行解读吧。
访问次数（VV）：记录所有访客1天内访问了多少次您的网站，相同的访客有可能多次访问您的网站。
独立访客（UV）：1天内相同访客多次访问网站，只计算为1个独立访客。
网站浏览量（PV）：用户每打开一个页面便记录1次PV
独立IP（IP）：同一IP无论访问了几个页面，独立IP数均为1

#### 访问次数（VV）
名词：VV = Visit View（访问次数）
说明：从访客来到您网站到最终关闭网站的所有页面离开，计为1次访问。若访客连续30分钟没有新开和刷新页面，或者访客关闭了浏览器，则被计算为本次访问结束。

#### 独立访客（UV）
名词：UV= Unique Visitor（独立访客数）
说明：1天内相同的访客多次访问您的网站只计算1个UV。以cookie为依据

#### 网站浏览量（PV）
名词：PV=PageView (网站浏览量)
说明：指页面的浏览次数，用以衡量网站用户访问的网页数量。多次打开同一页面则浏览量累计；

#### 独立IP（IP）
名词：IP=独立IP数
说明：指1天内使用不同IP地址的用户访问网站的数量。

### 如何在Java使用极小的内存完成对超大数据的去重计数，用于实时计算中统计UV？

一直在想如何在 实时计算中 完成对海量数据去重计数的功能，即SELECT COUNT(DISTINCT) 的功能。比如：从每天零点开始，实时计算全站累计用户数（UV），以及某些组合维度上的用户数，这里的用户假设以Cookieid来计。

想想一般的解决办法，在内存中使用HaspMap、HashSet？或者是在Redis中以Cookieid为key？感觉都不合适，在数以亿计用户的业务场景下，内存显然也成了瓶颈。

如果说，实时计算的业务场景中，对UV的计算精度并不要求100%（比如：实时的监测某一网站的PV和UV），那么可以考虑采用基数估计算法来统计。这里有一个Java的实现版本 stream-lib： https://github.com/addthis/stream-lib

采用基数估计算法目的就是为了使用很小的内存，即可完成超大数据的去重计数。号称是只使用几KB的内存，就可以完成对数以条数据的去重计数。但基数估计算法都不是100%精确的，误差在0~2%之间，一般是1%左右。


#### 示例

maven:
```
<dependency>
  <groupId>com.clearspring.analytics</groupId>
  <artifactId>stream</artifactId>
  <version>2.7.0</version>
</dependency>
```

demo:
```
import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.IOException;

import com.clearspring.analytics.stream.cardinality.AdaptiveCounting;
import com.clearspring.analytics.stream.cardinality.ICardinality;

public class TestCardinality {

  public static void main(String[] args) {
    Runtime rt = Runtime.getRuntime();
    long start = System.currentTimeMillis();
    long updateRate = 1000000l;
    long count = 0l;
    ICardinality card = AdaptiveCounting.Builder.obyCount(Integer.MAX_VALUE).build();
    
    File file = new File(args[0]);
      BufferedReader reader = null;
      try {
        reader = new BufferedReader(new FileReader(file));
        String tempString = null;
        while ((tempString = reader.readLine()) != null) {
            card.offer(tempString);
            count++;
            if (updateRate > 0 && count % updateRate == 0) {
                  System.out.println("Total count:[" + count + "]  Unique count:[" + card.cardinality() + "] FreeMemory:[" + rt.freeMemory() + "] ..");
              }
        }
        reader.close();
      } catch (Exception e) {
        e.printStackTrace();
      } finally {
        if (reader != null) {
            try {
                reader.close();
            } catch (IOException e1) {}
        }
      }
      long end = System.currentTimeMillis();
      System.out.println("Total count:[" + count + "]  Unique count:[" + card.cardinality() + "] FreeMemory:[" + rt.freeMemory() + "] ..");
    System.out.println("Total cost:[" + (end - start) + "] ms ..");
  }
}
```