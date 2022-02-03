---
title: 052-听说你爬回来的都是乱码？以某团为例讲解四种方案解决CSS字体反爬
urlname: python-web-font
date: '2021-01-13 19:40:21 +0800'
tags:
  - meituan
  - css
  - python
  - 反爬
categories:
  - python
---

> 这是坚持技术写作计划（含翻译）的第 52 篇，定个小目标 999，每周最少 2 篇。

本文以某团为例讲解如何获取 css 字体的真实内容

<!-- more -->

## 关于 css3 字体

在以前，前端工程师只能使用浏览器主机已经安装的字体，要实现一些比较风骚的字体效果，只能用 flash 或者图片，从 css3 以后，可以使用 [@font-face](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@font-face) 属性 实现，基本现代浏览器都支持，具体参考 [https://www.caniuse.com/?search=font-face](https://www.caniuse.com/?search=font-face) ，但是在人均爬虫时代，被玩出了新花样，包括但不限于 金额，数量，地址，名称等信息用 webfont 替代文本来抵挡一部分低级爬虫而又不影响用户阅读和查看。

访问 [https://fontdrop.info/](https://fontdrop.info/) ，下载 [http://s3plus.meituan.net/v1/mss_73a511b8f91f43d0bdae92584ea6330b/font/6cf535d2.woff](http://s3plus.meituan.net/v1/mss_73a511b8f91f43d0bdae92584ea6330b/font/6cf535d2.woff) ，并上传就能看到了
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1610536112771-3889a592-8c07-4043-9894-40b8673a85ac.png#align=left&display=inline&height=761&margin=%5Bobject%20Object%5D&name=image.png&originHeight=761&originWidth=800&size=61768&status=done&style=none&width=800)

## 方案一 OCR

这个思路是基于任何产品都不会牺牲用户阅读体验来提升反爬门槛，也就是说，用户肉眼能看到最终呈现效果，所以你把他当做是图片，用 OCR 产品去识别就可以了，除了占资源较多，识别效率较低外，鲁棒性更高。

常见的开源方案是 [tesseract](https://github.com/tesseract-ocr/tesseract)

## 方案二 映射

针对的是小站，自己费劲搞了一套 web font 万能不变，爬虫工程师定点爆破，假设有 100 套（这是往多了说的，实际上很可能就 1 套），把这些有限的 web font 都 down 下来，然后本地建立一个映射关系，爬取的内容 替换一下就行了

## 方案三 KNN/K 近邻/K 相邻

方案二的基础是 web font 有限且不变的前提，如果像某音，某团这种的机器随机生成的，那就没法了，这时候可以用 KNN
参考 [python 爬虫： 使用 knn 算法破解猫眼动态字体反爬](https://blog.csdn.net/qq_29570381/article/details/103035678)

## 方案四 欧氏距离

方案三 有点杀鸡用牛刀了 就为了个字体反爬，至于用 KNN 么，还得自己弄样本数据

计算欧氏距离只需要一个样本，找到映射关系，其余的只需要跟样本比较就行了

```python
# 下载 http://s3plus.meituan.net/v1/mss_73a511b8f91f43d0bdae92584ea6330b/font/6cf535d2.woff
base = TTFont(self.getFontFile(baseFontFile))
font = TTFont(self.getFontFile(fontFile))
# baseGlyf = base.get('glyf')
# glyf = font.get('glyf')

# np1 = np.array([j for j in baseGlyf[0].coordinates])
# np2 = np.array([j for j in glyf[0].coordinates])

np.linalg.norm(np1 -np2, 'constant', constant_values=0)

```

代码只贴出关键部分
通过 TTFont 解析 font，取 `glyf[n]coordinates`  取出点数组，使用 numpy 计算欧式距离，进行排序，最接近的就是符合的（两层循环），甚至可以观察下，欧式距离普遍小于几，假设是 1500，那内层循环小于 1500 就跳出就行了，注意 np1,np2 要补齐成一样长，否则会报错

补充一下，如果后期平台可能会同比缩放点位，这样简单计算欧氏距离就白搭了，需要把 xy 点位同比进行处理

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html?ka=search_list_jname_2_blank&lid=ak5J7ypLUb7.search.2) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/01/13/python-web-font/)
- [我的掘金](https://juejin.cn/post/6917200982505963533/)
- [CSS3 字体](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@font-face)
