---
title: 使用jupyter(IPython)开发opencv
date: 2017-07-16 17:09:12
tags: [opencv,ipython,jupyter]
---


opencv 的默认使用highgui显示图片，
用命令行运行，可以正常显示
```
cv.namedWindow("Image")
cv.imshow("Image",img)
```

用jupyter则有无反应

<!--more-->

## 本文环境
```python
import sys
import cv2
print("python版本:%s"% sys.version)
print("opencv版本:%s"% cv2.__version__)
```

输出
```
python版本:3.5.3 |Continuum Analytics, Inc.| (default, May 15 2017, 10:43:23) [MSC v.1900 64 bit (AMD64)]
opencv版本:3.2.0
```

## 安装opencv
如果使用Anaconda,则打开 `Anaconda Prompt`,`activate python35`切换到相应的python环境

```bash
pip install --upgrade setuptools
pip install numpy Matplotlib
pip install opencv-python
```

参考 [Python环境搭建之OpenCV][],但是在jupyter中，运行该博文下一段demo代码，无反应

经过一番google，已解决，现整理如下

## jupyter显示opencv图片
以[lenna][] 图为例

```python

import cv2
from matplotlib import pyplot as plt

img = cv2.imread('lenna1.png')
 
show_img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB) 

plt.imshow(show_img)
plt.show()
```

参考自 [Quickie: Mix up OpenCV and Jupyter (iPython Notebook)][linkQuickie:MixUpOpencvAndJupyter_ipython] 和官方 [Using Matplotlib][linkUsingMatplotlib]

## opencv读取网络图片
```python
%matplotlib inline
import numpy as np
import cv2
from matplotlib import pyplot as plt
import urllib.request as ul

data = None

try:
    
    data = ul.urlopen('http://www.mupin.it/wp-content/uploads/2012/06/lenna1.png').read()
    
except Exception as e:
    
    print("Could not download the image: %s " %( e.message))
    
else:
    data =  np.fromstring(data, np.uint8)
    img_data =  cv2.imdecode(data, cv2.IMREAD_COLOR )
    img_data = cv2.cvtColor(img_data, cv2.COLOR_BGR2RGB)
    plt.imshow(img_data)
    plt.show()
```

本示例用的环境是python:3.5.3 和 opencv:3.2.0，在opencv3.x中已经不存在`cv2.CV_LOAD_IMAGE_COLOR`,根据 [Python OpenCV load image from byte string][linkPythonOpencvLoadImageFromByteString] ，改成`cv2.IMREAD_COLOR`

大部分代码 参考自 [Quickie: Grab an image from the web with urllib2 and OpenCV][linkQuickie:GrabAnImageFromTheWebWith]


## [OpenCV入门教程][opencv-tutorial]


[opencv-tutorial]: http://blog.csdn.net/column/details/opencv-tutorial.html
[lenna]: https://en.wikipedia.org/wiki/Lenna
[Python环境搭建之OpenCV]: http://www.cnblogs.com/lclblack/p/6377710.html
[linkQuickie:MixUpOpencvAndJupyter_ipython]: https://giusedroid.blogspot.jp/2015/04/blog-post.html
[linkQuickie:GrabAnImageFromTheWebWith]: https://giusedroid.blogspot.jp/2015/04/quickie-download-and-show-image-with.html
[linkPythonOpencvLoadImageFromByteString]: https://stackoverflow.com/a/17170855/7001350
[linkUsingMatplotlib]: http://docs.opencv.org/3.2.0/dc/d2e/tutorial_py_image_display.html
