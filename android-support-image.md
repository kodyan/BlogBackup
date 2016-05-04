title: Android支持的图片格式
date: 2016/4/28 11:45:37
categories: 
tags: Android
---
Android官方文档中[Supported Media Formats](http://developer.android.com/guide/appendix/media-formats.html)部分介绍了Android支持的多媒体格式，Android支持的图片格式如下图。

![android-supported-image](http://7xoze0.com1.z0.glb.clouddn.com/android-supported-image.png)

本文对这几种图片格式做个学习总结

<!--more-->

----

## JPEG
[JPEG](https://zh.wikipedia.org/wiki/JPEG)（发音为jay-peg, IPA：[ˈdʒeɪpɛg]）是一种针对照片视频而广泛使用的一种压缩标准方法。这个名称代表Joint Photographic Experts Group（联合图像专家小组）。

- 常用的.jpg文件是有损压缩
- 不支持背景透明
- 适用于照片等色彩丰富的大图压缩
- 不适用于logo，线图

## GIF
[GIF](https://zh.wikipedia.org/wiki/GIF)，图像互换格式（GIF，Graphics Interchange Format）是一种位图图形文件格式，以8位色（即256种颜色）重现真彩色的图像。它采用无损压缩技术，只要图像不多于256色，则可既减少文件的大小，又保持成像的质量。

- 优秀的压缩算法使其在一定程度上保证图像质量的同时将体积变得很小。
- 可插入多帧，从而实现动画效果。
- 可设置透明色。
- 由于采用了8位压缩，最多只能处理256种颜色，故不宜应用于真彩色图片。

## PNG
[PNG](https://zh.wikipedia.org/wiki/PNG)，便携式网络图形（Portable Network Graphics，PNG）是一种无损压缩的位图图形格式，支持索引、灰度、RGB三种颜色方案以及Alpha通道等特性。

- 支持256色调色板技术以产生小体积文件
- 最高支持48位真彩色图像以及16位灰度图像。
- 支持Alpha通道的透明/半透明特性。
- 支持图像亮度的Gamma校准信息。
- 支持存储附加文本信息，以保留图像名称、作者、版权、创作时间、注释等信息。
- 使用无损压缩。
- 渐近显示和流式读写，适合在网络传输中快速显示预览效果后再展示全貌。
- 使用CRC防止文件出错。
- 最新的PNG标准允许在一个文件内存储多幅图像。

Android开发中的切图素材多为.png格式。

## BPM
BMP（全称Bitmap）是Windows操作系统中的标准图像文件格式，可以分成两类：设备相关位图（DDB）和设备无关位图（DIB），使用非常广。它采用位映射存储格式，除了图像深度可选以外，不采用其他任何压缩，因此，BMP文件所占用的空间很大。

BMP文件的图像深度可选lbit、4bit、8bit及24bit。BMP文件存储数据时，图像的扫描方式是按从左到右、从下到上的顺序。

## WebP
[WebP](https://zh.wikipedia.org/wiki/WebP)（发音weppy），是一种同时提供了有损压缩与无损压缩的图片文件格式，派生自视频编码格式VP8，是由Google在购买On2 Technologies后发展出来，以BSD授权条款发布。

从[官网](https://developers.google.com/speed/webp/)介绍来看，无损的WebP图片比PNG小26%，有损的WebP图片比JPEG小25-34%，同时，无损WebP支持透明及alpha通道，有损在一定条件下同样支持。

>WebP 的优势体现在它具有更优的图像数据压缩算法，能带来更小的图片体积，而且拥有肉眼识别无差异的图像质量；同时具备了无损和有损的压缩模式、Alpha 透明以及动画的特性，在 JPEG 和 PNG 上的转化效果都非常优秀、稳定和统一。

相较编码JPEG文件，编码同样质量的WebP文件也需要占用更多的计算资源。

Android 4.0+默认支持WebP，Android 4.2.1+开始支持无损WebP和带alpha通道的WebP。

----
参考资料：

[A new image format for the Web](https://developers.google.com/speed/webp/)

[WebP 探寻之路](http://isux.tencent.com/introduction-of-webp.html)

[PNG vs. GIF vs. JPEG vs. SVG - When best to use?](http://stackoverflow.com/questions/2336522/png-vs-gif-vs-jpeg-vs-svg-when-best-to-use/7752936#7752936)




