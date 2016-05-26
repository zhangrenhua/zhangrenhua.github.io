---
title: Tesseract-OCR图片识别 + tess4j
description: 最近在公司有个爬虫需求，爬取的网站上有图片验证码，不过样式比较简单。在网上找到一些图片识别的OCR框架，最后选择Tesseract。</br></br>tesseact其实全称是tesseract-ocr，是个自动识别字符的程序，项目网址是：http://code.google.com/p/tesseract-ocr/。虽然其主流平台是三大系统（Win/Linux/Mac OS），但在android和iphone上也是可以跑的。</br></br>tesseract支持多种语言 - 你只需下载对应的训练过的语言文件即可，并且可以通过config文件来调整行为：比如只识别数字，比如只识别指定的words或者指定的pattern。另外提一下，tesseract只支持字符识别，不支持条形码(barcode)识别。</br></br>tess4j（http://tess4j.sourceforge.net/）是一个从Java角度使用JNA封闭的针对 Tesseract ORC 的开源项目,使用  Apache License, v2.0 协议.支持TIFF, JPEG, GIF, PNG, and BMP image formats,Multi-page TIFF images,PDF document format.(支持Tiff是一个很大的亮点)。
tags: 爬虫
categories: ocr
date: 2016-5-26 16:19:49
---

## 背景

最近在公司有个爬虫需求，爬取的网站上有图片验证码，不过样式比较简单。在网上找到一些图片识别的OCR框架，最后选择Tesseract。

tesseact其实全称是tesseract-ocr，是个自动识别字符的程序，项目网址是：http://code.google.com/p/tesseract-ocr/。虽然其主流平台是三大系统（Win/Linux/Mac OS），但在android和iphone上也是可以跑的。

tesseract支持多种语言 - 你只需下载对应的训练过的语言文件即可，并且可以通过config文件来调整行为：比如只识别数字，比如只识别指定的words或者指定的pattern。另外提一下，tesseract只支持字符识别，不支持条形码(barcode)识别。

tess4j（http://tess4j.sourceforge.net/）是一个从Java角度使用JNA封闭的针对 Tesseract ORC 的开源项目,使用  Apache License, v2.0 协议.支持TIFF, JPEG, GIF, PNG, and BMP image formats,Multi-page TIFF images,PDF document format.(支持Tiff是一个很大的亮点)。


## 安装

本次测试在windows上进行，所以先下载exe文件，并安装：
[tesseract-ocr-setup-3.02.02.exe](http://7xoqbc.com1.z0.glb.clouddn.com/tesseract-ocr-setup-3.02.02.exe)


### 验证安装

将下面图片保存到D盘：
![Alt text](http://zhixing.court.gov.cn/search/security/jcaptcha.jpg?84)

进入cmd命令行，执行下面命令：
```
d:  --进入d盘
tesseract jcaptcha.jpg result  --解析图片，结果输出到result.txt文件

```

打开d:/result.txt查看结果。


## tess4j

pom.xml：
```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.hua.tess4j</groupId>
    <artifactId>tess4j</artifactId>
    <version>0.0.1-SNAPSHOT</version>


    <!-- <repositories> -->
    <!-- <repository> -->
    <!-- <id>org.ghost4j.repository.releases</id> -->
    <!-- <name>Ghost4J releases</name> -->
    <!-- <url>http://repo.ghost4j.org/maven2/releases</url> -->
    <!-- </repository> -->
    <!-- needed for image-io -->
    <!-- <repository> -->
    <!-- <releases /> -->
    <!-- <snapshots> -->
    <!-- <enabled>false</enabled> -->
    <!-- </snapshots> -->
    <!-- <id>mygrid-repository</id> -->
    <!-- <name>myGrid Repository</name> -->
    <!-- <url>http://www.mygrid.org.uk/maven/repository</url> -->
    <!-- </repository> -->
    <!-- </repositories> -->

    <dependencies>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.11</version>
            <scope>test</scope>
        </dependency>

        <!-- http://mvnrepository.com/artifact/net.sourceforge.tess4j/tess4j -->
        <dependency>
            <groupId>net.sourceforge.tess4j</groupId>
            <artifactId>tess4j</artifactId>
            <version>1.5.0</version>
        </dependency>

    </dependencies>

    <build />
</project>
```


测试代码：
```
import static org.junit.Assert.assertArrayEquals;
import static org.junit.Assert.assertEquals;

import java.awt.Rectangle;
import java.awt.image.BufferedImage;
import java.io.File;
import java.nio.IntBuffer;
import java.util.ArrayList;
import java.util.List;

import javax.imageio.IIOImage;
import javax.imageio.ImageIO;

import net.sourceforge.tess4j.TessAPI1;
import net.sourceforge.tess4j.Tesseract1;
import net.sourceforge.vietocr.ImageHelper;
import net.sourceforge.vietocr.ImageIOHelper;

import org.junit.After;
import org.junit.AfterClass;
import org.junit.Before;
import org.junit.BeforeClass;
import org.junit.Test;

import com.recognition.software.jdeskew.ImageDeskew;
import com.sun.jna.Pointer;

public class Tesseract1Test {

    Tesseract1 instance;

    @Before
    public void setUp() {
        instance = new Tesseract1();
    }

    /**
     * Test of doOCR method, of class Tesseract1.
     */
    @Test
    public void testDoOCR_File() throws Exception {
        System.out.println("doOCR on a PNG image");
        File imageFile = new File("d:/jcaptcha.jpg");
        String result = instance.doOCR(imageFile);
        System.out.println(result);
    }
}
```


### 提高识别率

instance.setPageSegMode(mode)，设置识别模式：
```
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
-l lang and/or -psm pagesegmode must occur before anyconfigfile.
```

## 总结

tesseract-OCR默认的识别率其实已经很高了，但是对于一些加了噪声的图片还需做一些手动处理，比如：可以手动训练数据，在后续如果有时间我会写出来。

如果想要支持中文可以下载中文语言包，所有语言包下载链接：
https://codeload.github.com/tesseract-ocr/tessdata/zip/master

示例代码下载：
[tess4j.zip](http://7xoqbc.com1.z0.glb.clouddn.com/tess4j.zip)