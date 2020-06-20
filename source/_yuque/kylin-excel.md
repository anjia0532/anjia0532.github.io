---
title: 022-会Excel就会数据分析(Kylin2.6.1+Excel+Power BI)
urlname: kylin-excel
date: 2019-05-04 19:30:00 +0800
tags: [CDH,hadoop,大数据,KYLIN,OLAP,麒麟,Excel,PowerBI]
categories: [大,数,据]
---

> 这是坚持技术写作计划（含翻译）的第 22 篇，定个小目标 999，每周最少 2 篇。

本文主要介绍如何使用 excel/power BI 连接 kylin2.6.1

<!-- more -->

## 前提准备

### 下载 kylin odbc

apache 官网已经停止提供 odbc 驱动了，只能从 csdn 等下载别人上传的，安全性毫无保证。后来转念一想，kylin 的商业公司[https://kyligence.io](https://kyligence.io/)，应该会提供吧，一番打探，果然有。

下载地址：[Download Kyligence ODBC Driver for Apache Kylin](https://kyligence.io/resources/kyligence-odbc-driver-for-apache-kylin-2/)

需要填写邮箱进行下载

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557487637088-183279e0-8761-48b5-81d2-8fd69bce8e36.png#align=left&display=inline&height=381&name=image.png&originHeight=381&originWidth=500&size=30999&status=done&width=500)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557487684886-c42c67f8-5dee-4af2-a47d-2bdce29db157.png#align=left&display=inline&height=381&name=image.png&originHeight=381&originWidth=500&size=47222&status=done&width=500)

### 配置 odbc 数据源

Win+R-> Control -> 管理工具 -> ODBC 数据源(64 位)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557487993106-7a08b90d-f53d-40f0-a3c9-8fdbff8611c8.png#align=left&display=inline&height=635&name=image.png&originHeight=635&originWidth=694&size=79547&status=done&width=694)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557488108032-bddd3f2f-fbfc-4e1d-a803-5c3f7a56f8f4.png#align=left&display=inline&height=425&name=image.png&originHeight=425&originWidth=400&size=17622&status=done&width=400)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557488114771-0324f0b0-c3a7-46fa-beea-170dff5e4b2d.png#align=left&display=inline&height=537&name=image.png&originHeight=537&originWidth=691&size=37086&status=done&width=691)

### 下载 powerQuery

Excel 2016 及以后版本，自带 PowerQuery
![](https://cdn.nlark.com/yuque/0/2019/png/226273/1557488219530-ad5c7194-e6ea-4601-b243-fe8cc45a99ab.png#align=left&display=inline&height=169&originHeight=169&originWidth=368&status=done&width=368)

之前版本需要自行下载 PowerQuery [https://www.microsoft.com/zh-CN/download/details.aspx?id=39379](https://www.microsoft.com/zh-CN/download/details.aspx?id=39379)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557488170637-adb0b9c1-ee8f-4be4-a9b3-ad31bda9b8db.png#align=left&display=inline&height=530&name=image.png&originHeight=530&originWidth=841&size=43855&status=done&width=841)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557488185981-9b81bcfa-3ddc-4695-9b3f-7d899748c779.png#align=left&display=inline&height=584&name=image.png&originHeight=584&originWidth=1268&size=44789&status=done&width=1268)

## Excel 连接 Kylin

### Excel 配置 Kylin ODBC 数据源

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557488301678-c1039340-d621-4384-a12e-a0b2ba8fab3f.png#align=left&display=inline&height=804&name=image.png&originHeight=804&originWidth=531&size=81563&status=done&width=531)

选择刚刚配置的 KylinDSN

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557488319201-dbb80bdd-e95e-4d10-8bd0-5d13ffd14837.png#align=left&display=inline&height=215&name=image.png&originHeight=215&originWidth=698&size=13399&status=done&width=698)
填入用户名密码

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557488426650-7fd3f229-c5d3-4daa-a495-787e6c946a08.png#align=left&display=inline&height=323&name=image.png&originHeight=323&originWidth=698&size=23403&status=done&width=698)

可以选择表进行预览
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557488441858-b8aabe15-7ac5-43c9-a7d1-154fbe70eafb.png#align=left&display=inline&height=698&name=image.png&originHeight=698&originWidth=878&size=76847&status=done&width=878)
点击加载按钮进行加载
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557488523780-b064229f-f003-4e0f-b125-08c8779b9eef.png#align=left&display=inline&height=640&name=image.png&originHeight=640&originWidth=1889&size=150632&status=done&width=1889)

## PowerBI 连接 Kylin

### 下载并安装 PowerBI

下载地址  [https://www.microsoft.com/zh-CN/download/details.aspx?id=45331](https://www.microsoft.com/zh-CN/download/details.aspx?id=45331)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558174065280-896bf454-a7cf-4ef0-af43-3053909c9b2e.png#align=left&display=inline&height=578&name=image.png&originHeight=578&originWidth=1098&size=82713&status=done&width=1098)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558174108812-155f3b80-e2ad-4e9d-90b6-07ebe2a55c43.png#align=left&display=inline&height=663&name=image.png&originHeight=663&originWidth=610&size=47135&status=done&width=610)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558174120855-4ec58da8-296b-446d-a2aa-45845dd377d9.png#align=left&display=inline&height=221&name=image.png&originHeight=221&originWidth=700&size=11194&status=done&width=700)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558174179132-030a7763-fc1a-48fa-bb1a-8de1b5ff060e.png#align=left&display=inline&height=699&name=image.png&originHeight=699&originWidth=879&size=53708&status=done&width=879)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558175521076-b79af7e2-3efb-48b5-9768-36d6d38102ca.png#align=left&display=inline&height=729&name=image.png&originHeight=729&originWidth=1850&size=101592&status=done&width=1850)

## 参考资料

- [Excel 及 Power BI 教程](http://kylin.apache.org/cn/docs/tutorial/powerbi.html)
- [Power BI Desktop 入门](https://docs.microsoft.com/zh-cn/power-bi/guided-learning/gettingdata?tutorial-step=2)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
