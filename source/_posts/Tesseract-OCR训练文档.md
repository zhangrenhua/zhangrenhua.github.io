---
title: Tesseract-OCR训练文档
description: Tesseract-OCR是一个图像只能字符识别技术（支持中文），对于一些比较标准的图片字符，识别率还是很高的，但是对于一些比较复杂（比如：手写字符），识别率就很低。下面主要讲解，如何提高一些不太标准或者相似的字符识别率。
tags: [ocr]
categories: ocr
date: 2017-4-12 13:39:29
---


## 背景

Tesseract-OCR是一个图像只能字符识别技术（支持中文），对于一些比较标准的图片字符，识别率还是很高的，但是对于一些比较复杂（比如：手写字符），识别率就很低。下面主要讲解，如何提高一些不太标准或者相似的字符识别率。


## 工作环境

- [Tesseract-OCR引擎（3.02.02）](http://7xoqbc.com1.z0.glb.clouddn.com/tesseract-ocr-setup-3.02.02.exe)
- [jTessBoxEditor用来训练字库](http://7xoqbc.com1.z0.glb.clouddn.com/jTessBoxEditor-1.4.zip)
- [java jdk（1.7或更高版本）](http://www.oracle.com/technetwork/java/javase/archive-139210.html)

以上程序都是基于windows操作系统，exe文件下载直接运行安装，zip文件直接解压就行

### linux安装

在yum安装前，需要epel源：
```
[root@root yum.repos.d]# yum install epel-release
```
/etc/yum.repos.d目录下就多了一个epel.repo文件

开始yum安装Tesseract：
```
[root@root yum.repos.d]# yum install tesseract
```

测试是否安装成功：
```
[root@root tesseract]# tesseract

Usage:
  tesseract imagename|stdin outputbase|stdout [options...] [configfile...]

OCR options:
  --tessdata-dir /path  specify the location of tessdata path
  --user-words /path/to/file    specify the location of user words file
  --user-patterns /path/to/file specify the location of user patterns file
  -l lang[+lang]        specify language(s) used for OCR
  -c configvar=value    set value for control parameter.
                        Multiple -c arguments are allowed.
  -psm pagesegmode      specify page segmentation mode.
These options must occur before any configfile.

pagesegmode values are:
  0 = Orientation and script detection (OSD) only.
  1 = Automatic page segmentation with OSD.
  2 = Automatic page segmentation, but no OSD, or OCR
  3 = Fully automatic page segmentation, but no OSD. (Default)
  4 = Assume a single column of text of variable sizes.
  5 = Assume a single uniform block of vertically aligned text.
  6 = Assume a single uniform block of text.
  7 = Treat the image as a single text line.
  8 = Treat the image as a single word.
  9 = Treat the image as a single word in a circle.
  10 = Treat the image as a single character.

Single options:
  -v --version: version info
  --list-langs: list available languages for tesseract engine. Can be used with --tessdata-dir.
  --print-parameters: print tesseract parameters to the stdout.
```


## 识别图片

![jTessBoxEditor images](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor3.jpg)

先下载[测试图片](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor-images.zip)，解压，通过命令行进入到图片目录，执行下面命令
```
tesseract 1.png result -l eng
```

![jTessBoxEditor test](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor4.jpg)

发现上面两张图片，识别都出错了，下面就开始进行字符训练。

## jTessBoxEditor训练

### jTessBoxEditor安装
jTessBoxEditor需要jre7（Java Runtime Environment）以上的版本支持。
安装完jre后，下载jTessBoxEditor，解压，运行train.bat文件即可运行
![jTessBoxEditor start](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor1.jpg)

运行界面
![jTessBoxEditor started](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor2.jpg)

### 合同图片

运行jTessBoxEditor工具，把所有图片合成一张.tif格式的图片
![jTessBoxEditor merge-tiff1](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor5.jpg)

打开所有要合成的图片(刚才在下的图片)
![jTessBoxEditor merge-tiff2](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor6.jpg)

将合成的.tif图片命名为`LAN.test.exp0`
![jTessBoxEditor merge-tiff2](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor7.jpg)

必须要按以下格式命名
```
tif命名规范：
[lang].[fontname].exp[num].tif
其中lang为语言名称，fontname为字体名称，num为序号，可以随便定义。
```

提示创建成功，在图片目录下生成一个`LAN.test.exp0.tif`的文件
![jTessBoxEditor merge-tiff2](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor8.jpg)

### 生成box文件

在`LAN.test.exp0.tif`所在的目录下打开一个命令行，产生相应的Box文件（*.box），该文件记录了tesseract识别出来的每一个字和其位置坐标。
```
$ tesseract LAN.test.exp0.tif LAN.test.exp0 batch.nochop makebox
Tesseract Open Source OCR Engine v3.02 with Leptonica
Page 1 of 2
Page 2 of 2
```

这时会在目录下生成一个`LAN.test.exp0.box`文件（可以用记事本打开，查看文字识别的坐标信息）

### 修正文字

使用jTessBoxEditor开始修正文字
![jTessBoxEditor boxeditor1](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor9.jpg)

打开后，可以看到第一张图片识别正确
![jTessBoxEditor boxeditor1](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor10.jpg)

这样的话，我们点击下一页，处理后面的图片
![jTessBoxEditor boxeditor1](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor11.jpg)

后面这张图片识别出错，我们手动调整并保存（如果）
![jTessBoxEditor boxeditor1](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor12.jpg)

注：<font color="red">有可能打开的tif文件并不能识别出字符，所以直接跳过继续下一张图片处理，如果出现多个字符未分割，则可通过界面上的`split`进行分割。</font>

### 产生字符特征文件（*.tr）

```
$ tesseract LAN.test.exp0.tif LAN.test.exp0 nobatch box.train
Tesseract Open Source OCR Engine v3.02 with Leptonica
Page 1 of 2
APPLY_BOXES:
   Boxes read from boxfile:       4
   Found 4 good blobs.
TRAINING ... Font name = test
Generated training data for 2 words
Page 2 of 2
APPLY_BOXES:
   Boxes read from boxfile:       4
   Found 4 good blobs.
Generated training data for 1 words
```

出现上面提示信息，则成功创建`LAN.test.exp0.tr`文件

### 产生计算字符集（unicharset）

```
unicharset_extractor LAN.test.exp0.box
Extracting unicharset from LAN.test.exp0.box
Wrote unicharset file ./unicharset.
```

创建unicharset文件，如果有多个box文件，可在命令行后面追加空格隔开（如：unicharset_extractor LAN.test.exp0.box LAN.test.exp1.box）

### 定义字体特征文件并聚集字符特征

**新建`font_properties`文件**
设置编码为UTF-8无BOM编码格式，内容如下
```
test 0 0 0 0 0
```

注意:这里test必须与训练名中的字体名称保持一致,填入下面内容 ,这里全取值为0，表示字体不是粗体、斜体等等。

**聚类shape**
```
$ shapeclustering –F font_properties  -U unicharset –O LAN.unicharset LAN.test.exp0.tr
Reading LAN.test.exp0.tr ...
Building master shape table
Computing shape distances...
Stopped with 0 merged, min dist 999.000000
Computing shape distances... 0
Stopped with 0 merged, min dist 999.000000
Computing shape distances... 0
Stopped with 0 merged, min dist 999.000000
Computing shape distances... 0
Stopped with 0 merged, min dist 999.000000
Computing shape distances... 0
Stopped with 0 merged, min dist 999.000000
Computing shape distances... 0
Stopped with 0 merged, min dist 999.000000
Computing shape distances... 0
Stopped with 0 merged, min dist 999.000000
Computing shape distances... 0
Stopped with 0 merged, min dist 999.000000
Computing shape distances...
Stopped with 0 merged, min dist 999.000000
Computing shape distances...
Stopped with 0 merged, min dist 999.000000
Computing shape distances... 0 1 2 3 4 5 6
Stopped with 0 merged, min dist 0.492188
Master shape_table:Number of shapes = 7 max unichars = 1 number with multiple unichars = 0
```

**聚集字符特征**
```
$ mftraining -F font_properties  -U unicharset LAN.test.exp0.tr
Read shape table shapetable of 7 shapes
Reading LAN.test.exp0.tr ...
Done!
```

**成字符形状正常化特征文件normproto**

```
$ cntraining LAN.test.exp0.tr
Reading LAN.test.exp0.tr ...
Clustering ...

Writing normproto ...
```

**合并训练文件**
把unicharset, inttemp, normproto, pffmtable,shapetable这四个文件加上前缀"test."
```
rename unicharset test.unicharset
rename inttemp test.inttemp
rename normproto test.normproto
rename pffmtable test.pffmtable
rename shapetable test.shapetable
```

生成语言包
```
combine_tessdata test.
Combining tessdata files
TessdataManager combined tesseract data files.
Offset for type 0 is -1
Offset for type 1 is 140
Offset for type 2 is -1
Offset for type 3 is 603
Offset for type 4 is 127965
Offset for type 5 is 128027
Offset for type 6 is -1
Offset for type 7 is -1
Offset for type 8 is -1
Offset for type 9 is -1
Offset for type 10 is -1
Offset for type 11 is -1
Offset for type 12 is -1
Offset for type 13 is 129164
Offset for type 14 is -1
Offset for type 15 is -1
Offset for type 16 is -1
```

此时语言包已生成，整个目录下的文件列表如下
![jTessBoxEditor all](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor13.jpg)

### 验证

将`test.traineddata`文件复制到tesseract-OCR安装目录下的tessdata目录下
![jTessBoxEditor all](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor14.jpg)

执行命令验证
```
$ tesseract 2.png result -l test
```

`-l`：指定刚才生成的语言包test

验证结果
![jTessBoxEditor test](http://7xoqbc.com1.z0.glb.clouddn.com/jtessboxeditor15.jpg)

通过上图可以看出，训练之后第二张图片已经识别成功

## 总结

如果想通过这种训练方式来解决图片识别率问题，那么训练库就得比较全，得把所有可能出现的字符内容图片都要训练处理。

## 帮助文档

- http://vietocr.sourceforge.net/training.html
- https://github.com/tesseract-ocr/tesseract