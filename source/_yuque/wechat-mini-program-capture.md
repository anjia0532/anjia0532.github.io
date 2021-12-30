---
title: 056-微信小程序抓包的三种方式
urlname: wechat-mini-program-capture
date: 2021-01-24 00:28:30 +0800
tags: [python,爬虫,微信,小程序]
categories: [爬虫]
---

> 这是坚持技术写作计划（含翻译）的第 56 篇，定个小目标 999，每周最少 2 篇。

很多时候针对 app 和 h5 的分析都困难重重，不妨换个思路，去小程序碰碰运气。本文分别讲解了从 PC 端，Android 端，IOS 端对小程序抓包的三种方式

<!-- more -->

## 准备工作

下载 Fiddler 最新版 [https://www.telerik.com/download/fiddler](https://www.telerik.com/download/fiddler)
安装 Fiddler，运行并配置 Https
Tools -> Options...->Https 勾选捕获 Https 连接，并安装 https 证书
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611407168642-844c8526-f750-4c94-92df-5f519d3ae052.png#align=left&display=inline&height=491&margin=%5Bobject%20Object%5D&name=image.png&originHeight=491&originWidth=1089&size=84917&status=done&style=none&width=1089)
配置运行远程连接及代理端口
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611407326837-72f432c8-d8b7-46b7-a1b1-1f9dbef14560.png#align=left&display=inline&height=368&margin=%5Bobject%20Object%5D&name=image.png&originHeight=368&originWidth=542&size=26220&status=done&style=none&width=542)

## PC 端

![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611402955638-1346027e-1e74-45fa-a66b-0c523fea8c51.png#align=left&display=inline&height=409&margin=%5Bobject%20Object%5D&name=image.png&originHeight=409&originWidth=561&size=38001&status=done&style=none&width=561)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611403675533-f7c2ec50-4f08-42d8-b016-195a46a58a57.png#align=left&display=inline&height=933&margin=%5Bobject%20Object%5D&name=image.png&originHeight=933&originWidth=889&size=428635&status=done&style=none&width=889)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611406921161-93352001-fbaf-400f-9331-acacca1321f4.png#align=left&display=inline&height=902&margin=%5Bobject%20Object%5D&name=image.png&originHeight=902&originWidth=1557&size=88409&status=done&style=none&width=1557)
用 PC 端微信好处是调试方便，缺点是，部分小程序有兼容性问题，所以极端场景下还得使用移动端抓包

## IOS 端

优先推荐使用 ios 版抓包（因为安卓版本 7.0+版本安全策略问题处理起来略微麻烦些）
下载针对移动版的 [fiddlercertmaker](https://telerik-fiddler.s3.amazonaws.com/fiddler/addons/fiddlercertmaker.exe) （fiddler 默认带的对于 ios13.x 经常抓不到包）
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611416432887-32ab028d-5076-4e83-921b-2b772debf1aa.png#align=left&display=inline&height=378&margin=%5Bobject%20Object%5D&name=image.png&originHeight=378&originWidth=1271&size=63931&status=done&style=none&width=1271)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611416524191-36f854d0-835a-4a8a-9f2d-3ffcb2bbb13e.png#align=left&display=inline&height=883&margin=%5Bobject%20Object%5D&name=image.png&originHeight=883&originWidth=497&size=121227&status=done&style=none&width=497)
设置 ->通用-> 描述文件 -> DO_NOT_TRUST_FiddlerRoot ->安装
设置 ->通用-> 关于本机 -> 证书信任设置 ->DO_NOT_TRUST_FiddlerRoot 打开开关 （IOS 10+后需要手动信任）
设置 -> 无线局域网 -> wifi 名称后的 i 图标 ->HTTP 代理 ->手动 -> 设置局域网服务器 ip 和端口，不用填认证信息
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611416925605-4a40a4c6-3eca-439f-9d5e-e90cdf2ffa98.png#align=left&display=inline&height=887&margin=%5Bobject%20Object%5D&name=image.png&originHeight=887&originWidth=995&size=147989&status=done&style=none&width=995)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/226273/1611417096794-07446aaa-2870-4e7a-8abd-5308574536bf.png#align=left&display=inline&height=697&margin=%5Bobject%20Object%5D&name=image.png&originHeight=697&originWidth=1261&size=387185&status=done&style=none&width=1261)
如果抓不到包的话，可以试试 [charles](https://www.charlesproxy.com/) ，一般就能抓到

## Android 端

android 7+版本针对 ssl 安全性做了加强，简单来说就是不认用户自己安装的证书，针对安卓端的 https 抓包有四种方案

1. 降级到 android7.0 之前的版本，比如 5.x 6.x
1. 将设备 root，把证书写到系统证书里
1. 用虚拟机（比如网易 mumu 或者夜神）安装 7.0 之前的版本或者 7.0 以后的将证书放到系统证书里
1. 使用 [https://github.com/MegatronKing/HttpCanary](https://github.com/MegatronKing/HttpCanary) 进行抓包

前三种不管哪种方案，弄好后，都修改 wifi 的代理设置，设置好代理后，进行抓包就行了

## 最后

分享几个比较流行的抓包/流量分析工具

- [charles](https://www.charlesproxy.com/)
- [fiddler-everywhere](https://www.telerik.com/download/fiddler-everywhere)
- [fiddler](https://www.telerik.com/download/fiddler)
- [HttpCanary](https://github.com/MegatronKing/HttpCanary)
- [wireshark](https://www.wireshark.org/)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html?ka=search_list_jname_2_blank&lid=ak5J7ypLUb7.search.2) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/01/24/wechat-mini-program-capture/)
- [我的掘金](https://juejin.cn/post/6920993581758939150/)
- [推荐一款万能抓包神器：Fiddler Everywhere](https://www.cnblogs.com/jinjiangongzuoshi/p/13577025.html)
- [轻松搞定 Charles 的 HTTPS 抓包（iOS13 可用）](https://blog.csdn.net/y277an/article/details/103573163)
