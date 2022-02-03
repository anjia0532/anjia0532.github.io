---
title: 009-时间不同步导致Sentinel监控异常
urlname: sentinel-timestamp
date: '2019-03-07 19:10:00 +0800'
tags:
  - spring boot
  - spring cloud
  - sentinel
  - hystrix
  - 微服务
  - 熔断
categories: 微服务
---

> 这是坚持技术写作计划（含翻译）的第 9 篇，定个小目标 999，每周最少 2 篇。

## 背景描述

在公司测试服务器调试[ahas](https://www.aliyun.com/product/ahas)（Sentinel 商业版）时，发现频发性无规律的出现 Ahas 控制台【监控详情】不显示,甚至应用直接消失的问题。

开始以为是非 Spring boot 应用的问题(因为另外一个产品线是 spring boot 的，测试没问题)，反复翻看开源[sentinel 的 wiki](https://github.com/alibaba/Sentinel/wiki)和[商业 ahas 的帮助文档](https://help.aliyun.com/product/87450.html) ,并且结合 Sentinel 的日志排查，毫无头绪。但是换成开源的 Sentinel Dashboard 没问题

## 解决步骤

### 问题原因

上文提到的，Spring boot 可以，是因为其部署在阿里云 ecs 上，而阿里云主机默认都有 ntp 同步

而测试机连 Sentinel 的 Dashboard 没问题，换成 ahas 就有问题，是因为 Sentinel 的 client 和 dashboard，部署在同一台服务器，不存在时间差问题。

后来通过  [@乐有](#)  和  [@云寅](#)  的帮助，定位到时钟问题, 据 @乐有 介绍 Sentinel 允许的最大时间误差是 30s，而实验中，测试机和北京时间误差超过 55s。

### windows 自动同步时间及修改同步频率

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1551944444938-feeadc19-7e4d-4b5a-8783-b8758668e48e.png#align=left&display=inline&height=595&originHeight=595&originWidth=538&status=done&width=538)

如果同步出错，可以重启一下 `Windows Time`  服务，再次同步。

但是过了半天后，时钟又差 1 分钟，所以需要调整一下 NTP 同步频率
打开注册表，找到 `SpecialPollInterval` (
`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpClien\SpecialPollInterval` )

发现默认值是十六进制  `93a80` ，换成 10 进制是 `604800` (7 天*24 小时*60 分钟*60 秒=604800)，间隔太长了 ,改成 300(5 分钟*60 秒)即可。

## 参考资料

- [Sentinel#Wiki#FAQ](https://github.com/alibaba/Sentinel/wiki/FAQ#q-%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%92%8C%E6%8E%A7%E5%88%B6%E5%8F%B0%E4%B8%8D%E5%9C%A8%E4%B8%80%E5%8F%B0%E6%9C%BA%E5%99%A8%E4%B8%8A%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%88%90%E5%8A%9F%E6%8E%A5%E5%85%A5%E6%8E%A7%E5%88%B6%E5%8F%B0%E5%90%8E%E6%8E%A7%E5%88%B6%E5%8F%B0%E6%97%A0%E6%B3%95%E6%98%BE%E7%A4%BA%E5%AE%9E%E6%97%B6%E7%9A%84%E7%9B%91%E6%8E%A7%E6%95%B0%E6%8D%AE%E4%BD%86%E7%B0%87%E7%82%B9%E9%93%BE%E8%B7%AF%E9%A1%B5%E9%9D%A2%E6%9C%89%E5%AE%9E%E6%97%B6%E8%AF%B7%E6%B1%82%E6%95%B0%E6%8D%AE%E4%B8%8D%E4%B8%BA-0)
- [配置 Windows 实例 NTP 服务](https://help.aliyun.com/document_detail/51890.html?spm=a2c4g.11186623.6.664.6ab468b6AQsVAL)
- [使用阿里云 NTP 服务器](https://help.aliyun.com/document_detail/92704.html?spm=a2c4g.11186623.6.663.284f49eaBjyUPf)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
