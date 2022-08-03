---
title: '072-解决 Jetbrains 启动失败报 BindException: Address already in use 错误'
urlname: jetbrains-starting-failed-address-already-in-use
date: '2022-04-17 11:35:21 +0800'
tags:
  - ide
  - idea
categories:
  - ide
  - idea
  - jetbrains
---

> 这是坚持技术写作计划（含翻译）的第 72 篇，定个小目标 999，每周最少 2 篇。

本文主要介绍遇到 Jetbrains 全家桶( Goland , Idea , Webstorm , Pycharm 等)启动时，报 `BindException: Address already in use` 的解决办法。

报错截图类似

![](https://cdn.nlark.com/yuque/0/2022/png/226273/1650166229031-ff7bff3f-380f-447b-b7b0-a5702f1820ef.png#clientId=u22807112-5082-4&crop=0&crop=0&crop=1&crop=1&from=paste&id=u2288a3bf&margin=%5Bobject%20Object%5D&originHeight=736&originWidth=1395&originalType=url∶=1&rotation=0&showTitle=false&status=done&style=none&taskId=uf8a35234-bf8d-4da8-b87c-44c14b2d116&title=)

<!-- more -->

## 方案 1

对我有用

```bash
net stop winnat
net start winnat
```

## 方案 2

对我没用

```bash
netsh winsock reset
```

## 方案 3

对我没用

```bash
netsh int ipv4 set dynamicport tcp start=49152 num=16383
netsh int ipv4 set dynamicport udp start=49152 num=16383
```

## 方案 4

我没试过

关闭 `Hyper-V`

## 题外话

[TCPView](https://docs.microsoft.com/en-us/sysinternals/downloads/tcpview) 一个 Windows 小工具，用于查看本机的 TCP 连接及使用和监听的端口号，常用于排查端口占用问题
[火绒安全](https://www.huorong.cn/person5.html) `安全工具`->`流量监控`->`查看所有连接` 功能类似 TCPView ，如果已经装了火绒安全的话，就不用装 TCPView 了
不想装软件，可以用 `Win+R`->`cmd` `netstat -ano | findstr 8080` 来排查端口使用

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/e78fa84f96fef4e733J60tq8EA~~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2022/04/17/jetbrains-starting-failed-address-already-in-use/)
- [我的掘金](https://juejin.cn/post/7088269133023805447/)
- [IDEA Start Failed: Address already in use](https://intellij-support.jetbrains.com/hc/en-us/community/posts/360006880600-IDEA-Start-Failed-Address-already-in-use)
- [Revise IDE folders locking mechanism (don't fail startup if all ports in range are taken, limited network due to firewall/VPN)](https://youtrack.jetbrains.com/issue/IDEA-238995)
