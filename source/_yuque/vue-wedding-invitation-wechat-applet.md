---
title: 047-零基础零成本使用vue开发婚礼请柬小程序
urlname: vue-wedding-invitation-wechat-applet
date: '2019-09-25 20:13:43 +0800'
tags:
  - 前端
  - js
  - vuejs
  - vue
  - 请柬
categories:
  - 前端
  - vuejs
---

> 这是坚持技术写作计划（含翻译）的第 47 篇，定个小目标 999，每周最少 2 篇。

婚期定在 10.1，毕竟是程序员，就自告奋勇的跟老婆说自己写个小程序做婚礼请柬。在[网友秋秋 QY](https://blog.csdn.net/qq_36070288/article/details/102392912)分享的基础上，略作改进，整理了这篇零基础零成本小程序教程。（如果嫌麻烦，婚礼纪   啥的网上随便找个交差也行）

贴一个原作者的   小程序二维码
![](https://cdn.nlark.com/yuque/0/2019/png/226273/1573086500589-e2c96412-ca51-44da-8980-faab974f849b.png#align=left&display=inline&height=324&originHeight=324&originWidth=355&size=0&status=done&width=355)

<!-- more -->

### 事前准备

- [Hbuilder X](https://www.dcloud.io/hbuilderx.html)
- [微信开发者工具](https://developers.weixin.qq.com/miniprogram/dev/devtools/stable.html)
- [注册小程序账号](https://developers.weixin.qq.com/miniprogram/introduction/)
- [版本控制软件-git](https://git-scm.com/downloads)

### 完善小程序设置

#### 补全信息获取小程序 ID

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573033130173-20b540b4-9c6a-4457-9978-4f20d301f5ee.png#align=left&display=inline&height=535&originHeight=535&originWidth=1679&size=78558&status=done&width=1679)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573033168673-e5f4aa01-4bd3-493b-b5bb-311a4dca3646.png#align=left&display=inline&height=832&originHeight=832&originWidth=1180&size=42814&status=done&width=1180)

### 配置 hbuilder 和 uniapp

配置 hbuilder

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573034148614-8b5ec86b-245c-4cd8-b84f-bb0d4923e91f.png#align=left&display=inline&height=332&originHeight=332&originWidth=242&size=20729&status=done&width=242)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573034171542-dd03a197-97a7-41ae-ae2d-883d8198dfa7.png#align=left&display=inline&height=75&originHeight=75&originWidth=830&size=7171&status=done&width=830)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573034189687-c203caea-daa1-4255-9974-a590857ca70f.png#align=left&display=inline&height=73&originHeight=73&originWidth=820&size=9005&status=done&width=820)

从  github 上 clone 下项目

`git clone https://github.com/anjia0532/jiayuan.git`

修改配置文件 `/common/js/metadata.js`  比如酒店坐标，新郎新娘电话等
修改小程序配置文件 `manifest.json` ![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573034066448-14353f22-20c6-4ef7-952d-8c3b347be550.png#align=left&display=inline&height=271&originHeight=271&originWidth=1110&size=46896&status=done&width=1110)

#### 运行微信小程序

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573034433907-0d1cb6a3-f126-4d96-9be1-2b266108917e.png#align=left&display=inline&height=353&originHeight=353&originWidth=586&size=43227&status=done&width=586)

#### 开通云开发环境

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573034487888-5403d846-60a7-40d2-a6d8-2551b4e6a14e.png#align=left&display=inline&height=664&originHeight=664&originWidth=910&size=154966&status=done&width=910)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573034507642-0520d381-6aef-4fec-9e4e-d23ea46efb2d.png#align=left&display=inline&height=639&originHeight=639&originWidth=602&size=44756&status=done&width=602)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573034570809-ed298f8d-6338-4cf8-acdb-983b891718da.png#align=left&display=inline&height=282&originHeight=282&originWidth=1147&size=25164&status=done&width=1147)

#### 创建云开发数据库

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573034702654-6d1d3471-771f-43b2-b6a9-c43af016f13a.png#align=left&display=inline&height=260&originHeight=260&originWidth=1055&size=38322&status=done&width=1055)
从 `static/databases`  复制文件名(去掉.json)  作为集合名称，创建。
分别导入数据，其中 message 和 user 是空集合，导入失败，不用管。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573034940830-44afbca4-858c-4b05-80d7-7e32526fbb69.png#align=left&display=inline&height=384&originHeight=384&originWidth=835&size=35357&status=done&width=835)

设置数据库的读写权限
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573035007788-5a8ba49f-a552-452d-af24-d25f2b316cb5.png#align=left&display=inline&height=309&originHeight=309&originWidth=982&size=35008&status=done&width=982)

#### 上传婚纱照

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573035187508-2f382c26-68e0-4b70-b335-c5291e6f1efd.png#align=left&display=inline&height=714&originHeight=714&originWidth=1203&size=106490&status=done&width=1203)
修改为婚纱照地址，也可以从 json 文件改完，重新导入。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573035278426-ea00c3d6-f7d1-447e-ba20-ca8598b53776.png#align=left&display=inline&height=320&originHeight=320&originWidth=927&size=31837&status=done&width=927)

#### 创建并上传云函数

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573035536387-9cacd79a-540b-4aaa-a8ef-0f3a4b881180.png#align=left&display=inline&height=267&originHeight=267&originWidth=954&size=30443&status=done&width=954)

#### 体验预览

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573036063500-f50a7a32-82d5-41e9-bde3-dec25fa99594.png#align=left&display=inline&height=490&originHeight=490&originWidth=872&size=69137&status=done&width=872)

#### 发给朋友体验

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573036101498-d2cbaccc-63a6-465f-9dbc-287d849915c8.png#align=left&display=inline&height=877&originHeight=877&originWidth=1644&size=88594&status=done&width=1644)

#### 发布

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573036143517-21528daa-9af9-445a-b0bf-db3ef978346a.png#align=left&display=inline&height=445&originHeight=445&originWidth=297&size=28710&status=done&width=297)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573036205532-a16e3b7a-191d-4941-a003-e9e07cd4daf3.png#align=left&display=inline&height=661&originHeight=661&originWidth=1090&size=96811&status=done&width=1090)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573036218419-fb1dc7e7-a554-45c9-95c3-569baf046def.png#align=left&display=inline&height=227&originHeight=227&originWidth=660&size=13144&status=done&width=660)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1573036263894-f4c0ba33-b4c3-476c-bb61-9f413275ef9f.png#align=left&display=inline&height=809&originHeight=809&originWidth=1792&size=73346&status=done&width=1792)

## 后话

本文主要是个人需要，结合网上大牛的开源项目，略作修改放出的，可能会有各种需要调整的地方，欢迎留言。我会尽量帮助各位新人。

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2019/09/25/vue-wedding-invitation-wechat-applet/)
- [我的掘金](https://juejin.im/post/5dc2a1a26fb9a04a9f11c176)
- [mpvue+小程序云开发，纯前端实现婚礼邀请函](https://juejin.im/post/5c341e1d6fb9a049f66c4876)
