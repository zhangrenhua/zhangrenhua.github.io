---
title: TextInputFormat如何处理跨split的那行数据
description: 刚接触hadoop，首先学习的必然是MapReduce编程。根据《hadoop权威指南》中的步骤走下去，必然是NCDC天气数据的统计分析了。当时我看着书籍里面的自定义Map只需要继承Mapper就能获取到一行一行的数据，我相信大多数人都会想为什么能获取到一行行的数据呢？为什么获取到的不是整个文件？或者是文件中的一部分？
tags: [hadoop生态圈, java]
categories: hadoop
date: 2015-11-21 20:00:00
---

## 背景

刚接触hadoop，首先学习的必然是MapReduce编程。根据《hadoop权威指南》中的步骤走下去，必然是NCDC天气数据的统计分析了。当时我看着书籍里面的自定义Map只需要继承Mapper就能获取到一行一行的数据，我相信大多数人都会想为什么能获取到一行行的数据呢？为什么获取到的不是整个文件？或者是文件中的一部分？

于是，我通过继续阅读《hadoop权威指南》并在网上找资料终于搞懂了，原来是InputFormat搞得鬼。hadoop将数据给到map进行处理前会使用InputFormat对数据进行两方面的预处理：
1、对输入数据进行切分，生成一组split，一个split会分发给一个mapper进行处理；
2、针对每个split，再创建一个RecordReader读取Split内的数据，并按照<key,value>的形式组织成一条record传给map函数进行处理。

其实说到这里问题就来了，注意hadoop在进行map之前会对数据进行split，而数据的split也是inputformat处理的，如果对split流程不懂的好可以去看我的另一博文[《mapreduce执行流程分析》](/2015/11/27/hadoop-mapreduce%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)。

本文主要针对TextInputFormat处理细节进行讲解。在split的读取方面，它是将给到的Split按行读取，以行首字节在文件中的偏移做key，以行数据做value传给map函数处理，这部分的逻辑是由它所创建并使用的LineRecordReader封装和实现的。关于这部分逻辑，在一开始接触hadoop时会有一个常见的疑问：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/hadoop-textinputformat-split.png)

如果一个行被切分到两个split里（这几乎是一定会发生的情况），TextInputFormat是如何处理的？如果是生硬地把一行切割到两个split里，是对数据的一种破坏，可能会影响数据分析的正确性(比如word count就是一个例子)。

## 数据行读取
当时我还在想HDFS文件系统是按块进行存储的，默认每块大小为128M。是不是HDFS文件系统是按行存储的？要不根据块进行split一行数据很有可能再个split上。后来经过网上找资料并阅读源码，终于理解了。下面我就简单地分析TextInputFormat是如何处理跨split的行数据的：
```
public class TextInputFormat extends FileInputFormat<LongWritable, Text> {

  @Override
  public RecordReader<LongWritable, Text> 
    createRecordReader(InputSplit split,
                       TaskAttemptContext context) {
    String delimiter = context.getConfiguration().get(
        "textinputformat.record.delimiter");
    byte[] recordDelimiterBytes = null;
    if (null != delimiter)
      recordDelimiterBytes = delimiter.getBytes(Charsets.UTF_8);
    return new LineRecordReader(recordDelimiterBytes);
  }

  @Override
  protected boolean isSplitable(JobContext context, Path file) {
    final CompressionCodec codec =
      new CompressionCodecFactory(context.getConfiguration()).getCodec(file);
    if (null == codec) {
      return true;
    }
    return codec instanceof SplittableCompressionCodec;
  }

}
```

1、TextInputFormat中其实使用了LineRecordReader来进行数据的读取，LineRecordReader会创建一个org.apache.hadoop.util.LineReader实例，并依赖这个LineReader的readLine方法来读取一行记录，具体可参考LineRecordReader类的185行：
```
    while (getFilePosition() <= end || in.needAdditionalRecordAfterSplit()) {
      if (pos == 0) {
        newSize = skipUtfByteOrderMark();
      } else {
        newSize = in.readLine(value, maxLineLength, maxBytesToConsume(pos));
        pos += newSize;
      }

      if ((newSize == 0) || (newSize < maxLineLength)) {
        break;
      }

      // line too long. try again
      LOG.info("Skipped line of size " + newSize + " at pos " + 
               (pos - newSize));
    }
```

那么关键的逻辑就在这个readLine方法里了,下面是我添加了中文注释的源码。这个方法主要的逻辑归纳起来是3点:
1、总是是从buffer里读取数据，如果buffer里的数据读完了，先加载下一批数据到buffer；
2、在buffer中查找“行尾”，将开始位置至行尾处的数据拷贝给str(也就是Value)，如果为遇到“行尾”，继续加载新的数据到buffer进行查找；
3、关键点在于：<font color="red">给到buffer的数据是直接从文件中读取的,完全不会考虑是否超过了split的界限,而是一直读取到当前行结束为止</font>

新版的api，readLine下面其实是有2个实现的：readDefaultLine方法按操作系统默认的换行符来处理，readCustomLine方法则是自定义的换行符。这里我们主要讲解readDefaultLine方法：
```
/**
   * Read a line terminated by one of CR, LF, or CRLF.
   */
  private int readDefaultLine(Text str, int maxLineLength, int maxBytesToConsume)
  throws IOException {
    /* We're reading data from in, but the head of the stream may be
     * already buffered in buffer, so we have several cases:
     * 1. No newline characters are in the buffer, so we need to copy
     *    everything and read another buffer from the stream.
     * 2. An unambiguously terminated line is in buffer, so we just
     *    copy to str.
     * 3. Ambiguously terminated line is in buffer, i.e. buffer ends
     *    in CR.  In this case we copy everything up to CR to str, but
     *    we also need to see what follows CR: if it's LF, then we
     *    need consume LF as well, so next call to readLine will read
     *    from after that.
     * We use a flag prevCharCR to signal if previous character was CR
     * and, if it happens to be at the end of the buffer, delay
     * consuming it until we have a chance to look at the char that
     * follows.
     */
    str.clear();
    int txtLength = 0; //tracks str.getLength(), as an optimization
    int newlineLength = 0; //length of terminating newline
    boolean prevCharCR = false; //true of prev char was CR
    long bytesConsumed = 0;
    do {
      int startPosn = bufferPosn; //starting from where we left off the last time

      //如果buffer中的数据读完了，先加载一批数据到buffer里
      if (bufferPosn >= bufferLength) {
        startPosn = bufferPosn = 0;
        if (prevCharCR) {
          ++bytesConsumed; //account for CR from previous read
        }
        bufferLength = in.read(buffer);
        if (bufferLength <= 0) {
          break; // EOF
        }
      }

    /*
     *注意：这里的逻辑有点复杂,由于不同操作系统对“行结束符“的定义不同：  
     *UNIX: '\n'  (LF)  
     *Mac:  '\r'  (CR)  
     *Windows: '\r\n'  (CR)(LF)  
     *为了准确判断一行的结尾，程序的判定逻辑是：
     *1.如果当前符号是LF，可以确定一定是到了行尾，但是需要参考一下前一个字符，因为如果前一个字符是CR，那就是windows文件，“行结束符的长度”  
     *(即变量：newlineLength,这个变量名起的有点糟糕)应该是2，否则就是UNIX文件，“行结束符的长度”为1。  
     *2.如果当前符号不是LF，看一下前一个符号是不是CR，如果是也可以确定一定上个字符就是行尾了，这是一个mac文件。  
     *3.如果当前符号是CR的话，还需要根据下一个字符是不是LF判断“行结束符的长度”，所以只是标记一下prevCharCR=true，供读取下个字符时参考。
     */
      for (; bufferPosn < bufferLength; ++bufferPosn) { //search for newline
        if (buffer[bufferPosn] == LF) {
          newlineLength = (prevCharCR) ? 2 : 1;
          ++bufferPosn; // at next invocation proceed from following byte
          break;
        }
        if (prevCharCR) { //CR + notLF, we are at notLF
          newlineLength = 1;
          break;
        }
        prevCharCR = (buffer[bufferPosn] == CR);
      }
      int readLength = bufferPosn - startPosn;
      if (prevCharCR && newlineLength == 0) {
        --readLength; //CR at the end of the buffer
      }
      bytesConsumed += readLength;
      int appendLength = readLength - newlineLength;
      if (appendLength > maxLineLength - txtLength) {
        appendLength = maxLineLength - txtLength;
      }
      if (appendLength > 0) {
        str.append(buffer, startPosn, appendLength);
        txtLength += appendLength;
      }

      //newlineLength == 0就意味着始终没有读到行尾，程序会继续通过文件输入流继续从文件里读取数据。
    } while (newlineLength == 0 && bytesConsumed < maxBytesToConsume);

    if (bytesConsumed > (long)Integer.MAX_VALUE) {
      throw new IOException("Too many bytes before newline: " + bytesConsumed);
    }
    return (int)bytesConsumed;
  }
```


这里有一个非常重要的地方：LineRecordReader的初始化函数（initialize）：
第85行：fileIn = fs.open(file);
我们看以看到：对于LineRecordReader：当它对取”一行“时，<font color="red">一定是读取到完整的行</font>，不会受filesplit的任何影响，因为它读取是filesplit所在的文件，而不是限定在filesplit的界限范围内。所以不会出现“断行”的问题！
```
  public void initialize(InputSplit genericSplit,
                         TaskAttemptContext context) throws IOException {
    FileSplit split = (FileSplit) genericSplit;
    Configuration job = context.getConfiguration();
    this.maxLineLength = job.getInt(MAX_LINE_LENGTH, Integer.MAX_VALUE);
    start = split.getStart();
    end = start + split.getLength();
    final Path file = split.getPath();

    // open the file and seek to the start of the split
    final FileSystem fs = file.getFileSystem(job);
    fileIn = fs.open(file);

    ....
  }
```

## 首行尾行处理
按照readLine的上述行为，在遇到跨split的行时，会到下一个split继续读取数据直至行尾，那么下一个split怎么判定开头的一行有没有被上一个split的LineRecordReader读取过从而避免漏读或重复读取开头一行呢？这方面LineRecordReader使用了一个简单而巧妙的方法：
既然无法断定每一个split开始的一行是独立的一行还是被切断的一行的一部分，那就跳过每个split的开始一行(当然要除第一个split之外)，从第二行开始读取，<font color="red">然后在到达split的结尾端时总是再多读一行</font>，这样数据既能接续起来又避开了断行带来的麻烦。以下是相关的源码：
```
  public void initialize(InputSplit genericSplit,
                         TaskAttemptContext context) throws IOException {

    ....

    // If this is not the first split, we always throw away first record
    // because we always (except the last split) read one extra line in
    // next() method.
    if (start != 0) {
      start += in.readLine(new Text(), 0, maxBytesToConsume(start));
    }
    this.pos = start;
  }
```
在LineRecordReader的初始化函数111到116行确定start位置时，明确注明：<font color="red">如果不是文件的开始位置则忽略掉该split的第一行数据</font>

```
  public boolean nextKeyValue() throws IOException {
    if (key == null) {
      key = new LongWritable();
    }
    key.set(pos);
    if (value == null) {
      value = new Text();
    }
    int newSize = 0;
    // We always read one extra line, which lies outside the upper
    // split limit i.e. (end - 1)
    while (getFilePosition() <= end || in.needAdditionalRecordAfterSplit()) {
      if (pos == 0) {
        newSize = skipUtfByteOrderMark();
      } else {
        newSize = in.readLine(value, maxLineLength, maxBytesToConsume(pos));
        pos += newSize;
      }

      if ((newSize == 0) || (newSize < maxLineLength)) {
        break;
      }

      // line too long. try again
      LOG.info("Skipped line of size " + newSize + " at pos " + 
               (pos - newSize));
    }
    if (newSize == 0) {
      key = null;
      value = null;
      return false;
    } else {
      return true;
    }
  }

  private long getFilePosition() throws IOException {
    long retVal;
    if (isCompressedInput && null != filePosition) {
      retVal = filePosition.getPos();
    } else {
      retVal = pos;
    }
    return retVal;
  }
```

相应地，在LineRecordReader判断是否还有下一行的方法:LineRecordReader.nextKeyValue()179到181行中，while使用的判定条件是：
当前位置小于或等于split的结尾位置，也就说：
<font color="red">当当前以处于split的结尾位置上时，while依然会执行一次，这一次读到显然已经是下一个split的开始行了！</font>


## 总结
至此，跨split的行读取的逻辑就完备了。如果引申地来看，这是map-reduce前期数据切分的一个普遍性问题，即不管我们用什么方式切分和读取一份大数据中的小部分，包括我们在实现自己的InputFormat时,都会面临在切分处数据时的连续性解析问题。对此我们应该深刻地认识到：split最直接的现实作用是取出大数据中的一小部分给mapper处理，但这只是一种“逻辑”上的，“宏观”上的切分，在“微观”上，在split的首尾切分处，为了确保数据连续性，跨越split接续并拼接数据也是完全正当和合理的。