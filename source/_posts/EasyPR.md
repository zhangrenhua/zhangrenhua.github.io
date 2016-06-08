---
title: EasyPR是一个中文的开源车牌识别系统
description: EasyPR是一个中文的开源车牌识别系统，其目标是成为一个简单、高效、准确的车牌识别引擎。<br/><br/>相比于其他的车牌识别系统，EasyPR有如下特点：<br/><ul><li>它基于openCV这个开源库。这意味着你可以获取全部源代码，并且移植到opencv支持的所有平台。</li><li>它能够识别中文。例如车牌为苏EUK722的图片，它可以准确地输出std:string类型的”苏EUK722”的结果。</li><li>它的识别率较高。图片清晰情况下，车牌检测与字符识别可以达到80%以上的精度。</li></ul>
tags: [车牌识别]
categories: ocr
date: 2016-6-8 14:35:45
---


## EasyPR

EasyPR是一个中文的开源车牌识别系统，其目标是成为一个简单、高效、准确的车牌识别引擎。

相比于其他的车牌识别系统，EasyPR有如下特点：

* 它基于openCV这个开源库。这意味着你可以获取全部源代码，并且移植到opencv支持的所有平台。
* 它能够识别中文。例如车牌为苏EUK722的图片，它可以准确地输出std:string类型的"苏EUK722"的结果。
* 它的识别率较高。图片清晰情况下，车牌检测与字符识别可以达到80%以上的精度。


### 跨平台

目前除了windows平台以外，还有以下其他平台的EasyPR版本。一些平台的版本可能会暂时落后于主平台。

|版本 | 开发者 | 版本 | 地址 
|------|-------|-------|-------
| android |  goldriver  |  1.3  |  [linuxxx/EasyPR_Android](https://github.com/linuxxx/EasyPR_Android)
| linux | Micooz  |  1.4  |  已跟EasyPR整合
| ios | zhoushiwei |  1.3  |  [zhoushiwei/EasyPR-iOS](https://github.com/zhoushiwei/EasyPR-iOS)
| mac | zhoushiwei,Micooz |  1.4  | 已跟EasyPR整合
| java | fan-wenjie |  1.2  | [fan-wenjie/EasyPR-Java](https://github.com/fan-wenjie/EasyPR-Java)

### 兼容性

当前EasyPR是基于opencv3.0版本开发的，3.0及以上的版本应该可以兼容，以前的版本可能会存在不兼容的现象。

### 运行示例

本次测试在windows上执行，采用jdk1.8 + opencv-3.0 + javacv-0.11，请先确认机器上是否安装。

下载EasyPR-Java，导入eclipse，运行EasyPrTest.java：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/opencv-easypr-test.png)


本文不涉及opencv的安装，如有需要可联系我。