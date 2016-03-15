---
title: Hadoop中序列化（Writable）接口
description: 所谓序列化（Serialization）使用官方的定义就是指将数据采用流的形式存储，以便于数据在网络传输或者写入磁盘上。</br>我们知道，目前Hadoop采用运算模型是多个分节点共同进行Map任务，然后将分节点上的已经计算完毕的任务发送给Reduce任务节点共同进行任务操作，Reduce任务节点收到信息后开始计算。但是这里产生的问题就是有些数据无法进行远程传输，或者无法保证传输质量，因此，为了解决这些问题，Hadoop提出了一种消息的序列化格式，将消息在发送前人为或者自动修改成特定格式进行传输，在收到消息后，根据传来的序列化格式自动对其进行反序列化。这里反序列化的意思就是将数据从流的形式转化为原始的数据格式。</br>序列化好处主要有已下几点：</br>（1）格式确定：存为特定的格式后，双方可以根据约定对数据进行可逆的操作；</br>（2）便于传输：Hadoop的运算模式是基于Map端与Reduce端的分离与整合，其中有需要大量的数据传输，对数据进行序列化后，可以更好的在两端进行传输；</br>（3）更易于程序后期管理：程序在运行过程中，后期往往会加入更多的新功能，Hadoop本身是一个“云计算框架”，实质上可以把她理解成运行在多个计算机上的分布式大型程序框架，只不过物理位置是分开的。而程序在后期进行维护更新过程中，往往会加入大量新的功能与方法，采用既有的数据格式不便于后期的维护开发。而采用序列化后，程序在后期进行变更与升级，而只需要对双方提供新的序列与反序列化约定即可。
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-12-20 21:29:00
---

## 背景

所谓序列化（Serialization）使用官方的定义就是指将数据采用流的形式存储，以便于数据在网络传输或者写入磁盘上。
我们知道，目前Hadoop采用运算模型是多个分节点共同进行Map任务，然后将分节点上的已经计算完毕的任务发送给Reduce任务节点共同进行任务操作，Reduce任务节点收到信息后开始计算。但是这里产生的问题就是有些数据无法进行远程传输，或者无法保证传输质量，因此，为了解决这些问题，Hadoop提出了一种消息的序列化格式，将消息在发送前人为或者自动修改成特定格式进行传输，在收到消息后，根据传来的序列化格式自动对其进行反序列化。这里反序列化的意思就是将数据从流的形式转化为原始的数据格式。

序列化好处主要有已下几点：
（1）格式确定：存为特定的格式后，双方可以根据约定对数据进行可逆的操作；

（2）便于传输：Hadoop的运算模式是基于Map端与Reduce端的分离与整合，其中有需要大量的数据传输，对数据进行序列化后，可以更好的在两端进行传输；

（3）更易于程序后期管理：程序在运行过程中，后期往往会加入更多的新功能，Hadoop本身是一个“云计算框架”，实质上可以把她理解成运行在多个计算机上的分布式大型程序框架，只不过物理位置是分开的。而程序在后期进行维护更新过程中，往往会加入大量新的功能与方法，采用既有的数据格式不便于后期的维护开发。而采用序列化后，程序在后期进行变更与升级，而只需要对双方提供新的序列与反序列化约定即可。

## 自定义序列化

Hadoop提供很多种默认实现，下面对常用的几种进行简要说明：
（1）Text -> 字符串操作
（2）IntWritable -> int操作
（3）ObjectWritable -> java基础类型操作
（4）NullWritable -> Null操作
（5）ByteWritable -> 字节操作

上面几种实现其实不能满足实际工作中的一些需求，比如：Map中的value我们可能要传多个字段（可以用MapWritable）或者是说一个自定义的对象。

自定义序列化接口（Writable），只需要继承WritableComparable类，并实现其中3个方法即可。

示例：
```
import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

import org.apache.hadoop.io.WritableComparable;

/**
 * 
 * @Description:
 * @version 1.0 2014年8月11日 下午2:33:39 by 张仁华 创建
 */
public class TextAndIntPair implements WritableComparable<TextAndIntPair> {

    private String text;
    private Integer ints;

    public TextAndIntPair() {
        super();
        text = "";
        ints = 0;
    }

    @Override
    public void write(DataOutput out) throws IOException {
        out.writeUTF(text);
        out.write(ints);
    }

    @Override
    public void readFields(DataInput in) throws IOException {
        text = in.readUTF();
        ints = in.readInt();
    }

    @Override
    public int compareTo(TextAndIntPair o) {
        int compareText = text.compareTo(o.getText());
        if (compareText != 0) {
            return compareText;
        }
        if (ints == null) {
            ints = 0;
        }
        if (o.getInts() == null) {
            ints = 0;
        }

        return ints.compareTo(o.getInts());
    }

    @Override
    public int hashCode() {
        final int prime = 31;
        int result = 1;
        result = prime * result + ((ints == null) ? 0 : ints.hashCode());
        result = prime * result + ((text == null) ? 0 : text.hashCode());
        return result;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj)
            return true;
        if (obj == null)
            return false;
        if (getClass() != obj.getClass())
            return false;
        TextAndIntPair other = (TextAndIntPair) obj;
        if (ints == null) {
            if (other.ints != null)
                return false;
        } else if (!ints.equals(other.ints))
            return false;
        if (text == null) {
            if (other.text != null)
                return false;
        } else if (!text.equals(other.text))
            return false;
        return true;
    }

    public String getText() {
        return text;
    }

    public void setText(String text) {
        this.text = text;
    }

    public Integer getInts() {
        return ints;
    }

    public void setInts(Integer ints) {
        this.ints = ints;
    }

}


```

这是自定义的一个Writable类，实现非常简单，首先使用Hadoop基于数据类型String与Integer进行基础构建，并提供set与get方法对其进行赋值与取值。通过其构造方法的调用对之进行初始化。同时因为我们自定义的Writable类继承自WritableComparable类，必须实现三个默认的方法。对于write(DataOutput out)与readFields(DataInput int)方法，我们调用其数据类型对于的write与read方法来实现数据的写于读操作，我们不用再直接对DataOutput与DataInput进行操作。compareTo方法我们使用Java语言自带的比较器。这也是Java所提倡的尽量使用原生的数据类型与方法。

注：<font color="red">如果使用基本数据类型，则要保证每个字段不能为Null，建议使用默认值，Hadoop中也是这样做的。</font>

hive源码中有一个比较不错的类，推荐大家去看下：
```
org.apache.hive.hcatalog.data.DefaultHCatRecord
```

## 参考资料

《MapReduce2.0源码分析与实战编程》