---
title: 法院公开信息爬取
description: htmlunit + tess4j爬取法院公开信息(被执行人、失信被执行人、审判流程等信息)<br/><br/>htmlunit 是一款开源的java 页面分析工具，读取页面后，可以有效的使用htmlunit分析页面上的内容。项目可以模拟浏览器运行，被誉为java浏览器的开源实现。是一个没有界面的浏览器，运行速度迅速。<br/><br/>tess4j安装请参考[Tesseract-OCR图片识别 + tess4j](/2016/05/26/Tesseract-OCR图片识别/)<br/><br/>项目地址：https://github.com/zhangrenhua/crawler-court
tags: [htmlunit, ocr]
categories: 爬虫
date: 2016-6-1 14:49:35
---

## 背景


htmlunit + tess4j爬取法院公开信息(被执行人、失信被执行人、审判流程等信息)

htmlunit 是一款开源的java 页面分析工具，读取页面后，可以有效的使用htmlunit分析页面上的内容。项目可以模拟浏览器运行，被誉为java浏览器的开源实现。是一个没有界面的浏览器，运行速度迅速。

tess4j安装请参考[Tesseract-OCR图片识别 + tess4j](/2016/05/26/Tesseract-OCR图片识别/)

项目地址：https://github.com/zhangrenhua/crawler-court

## 项目功能

本项目采用HtmlUnit + Tess4j，主要用来爬取法院公开信息。其中功能细节如下：

**数据层面**
1、法院被执行人 - http://zhixing.court.gov.cn/search/
2、法院失信被执行人 - http://shixin.court.gov.cn/

**图片解析**
1、无噪音纯字母数字组合
2、有背景噪音纯字母数字组合

**数据格式**
返回json格式

**待开发**
审判流程信息 - http://splcgk.court.gov.cn/splcgk/

## 测试

### 运行测试

法院被执行人测试结果：</br>
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/crawler-court-beexecuted.png)

法院失信被执行人测试结果：</br>
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/crawler-court-dishonesty.png)