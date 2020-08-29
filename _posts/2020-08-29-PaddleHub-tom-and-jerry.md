---
layout: post
title: "PaddleHub创意项目 猫和老鼠相声专场"
date: 2020-08-29
excerpt: 
tags: [图像处理, paddle, OpenCV]
feature: 
comments: false
---

之前报名了百度的一个零基础深度学习的线上课程，百度的课程肯定是用自家的飞桨那一套，这篇文章主要记录一下课程里的一个PaddleHub创意项目实现过程中的思路和踩到的坑。

##### 1、关于PaddleHub

PaddleHub是基于飞桨生态的一系列预训练好了的模型，可以在此基础上进行微调然后部署成服务，包括一些图像和自然语言处理的东西，图像分类、目标检测、文本相似度计算等等，具体的可以去[github](https://github.com/PaddlePaddle/PaddleHub)上看，最重要的是不需要写很多代码就可以直接调用模型。

##### 2、关于动机

本来打算弄一段柯南把凶手框出来，但是月底时间不足，只能考虑直接拿过模型来用。一开始打算全员狗头，想了想好像没什么意思，之前在b站上看见有人做猫和老鼠版的京剧，于是决定弄段对口相声换脸成猫和老鼠。相声的优势在于场景和人物位置变化不大，~~比较好找脸~~（事实上小小地被模型打了回脸），找到以后把两个头像贴上去就行了。

##### 3、设计思路

实现的思路其实很简单：

1. 人脸识别部分用PaddleHub的模型
2. 换脸用OpenCV，逐帧读取视频并处理，换脸的时候掩膜然后前景后景叠加起来替换原图
3. 用`moviepy`这个包来整合视频和音频

##### 4、一些细节

- OpenCV读进来的图像是RGB或者RGBA的，直接用matplotlib可视化的话看起来色调不正常，可以转成BGR的

  ```python
  import cv2
  import matplotlib.pyplot as plt
  img_bgr = cv2.cvtColor(img_rgb, cv2.COLOR_RGBA2BGR)
  plt.imshow(img_bgr)
  plt.axis('off') 
  ```

- 一张图片的大小可以大致表示为$$H \times W \times C$$，也就是说选取图像的区域应该选前两维，最后一维是各个通道的值

- 有时候模型会把手识别成脸，这时候选择置信度最高的两个“脸”，按照左右顺序贴猫和老鼠头像，当然在极端的情况下，<font color="red">手的置信分数很高</font>。

- 为了能把脸盖住，以硬编码的方式稍微放大了一下覆盖的区域，但是这样存在一个<font color="red">问题</font>：如果脸的位置过于靠近边界，这个放大有可能导致越界，会在mask的时候因为大小的问题报一个错。

##### 4、结果

项目地址：https://aistudio.baidu.com/aistudio/projectdetail/764990

处理后的成品视频（本人抠图水平一般，带有白边属于正常情况，不影响观看）：

<video controls  style="width= 50%; height=50%">
<source src="https://dle.oss-cn-beijing.aliyuncs.com/18-7-21/result%20%283%29.mp4" type="video/mp4">
</video>

##### 5、其它

最后来一点杂谈吧，首先告诫自己心里应该有点b树，上回折腾OpenCV还是快两年以前了，菜就不要折腾图像[捂脸]；然后飞桨，中文文档好评，但是看起来总感觉少了点什么，可能字不够多吧；最后期待一下PaddleHub的语音模型，有点想给韩剧做字幕。

