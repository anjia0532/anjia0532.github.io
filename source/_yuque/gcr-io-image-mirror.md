---
title: Google Container Registry(gcr.io) 中国可用镜像(长期维护)
date: 2017-11-15 12:04:14
tags: [k8s,kubernetes,rancher,gcr.io]
---

google镜像库Google Container Registry([gcr.io][]) 被gfw墙了。花了点时间用github + travis ci + docker hub成功将gcr.io的全部镜像同步到docker hub了。配合 国内各种加速器 [Docker 中国官方镜像加速][Docker中国官方镜像加速] ,[加速器 DaoCloud - 业界领先的容器云平台][加速器Daocloud-业界领先的容器云平台]速度还是很快的

<!--more-->

![](http://ww1.sinaimg.cn/large/afaffa71ly1fliqwie5huj20qh0dit9y.jpg)

## 何时同步

使用了 travis ci 的定时构建功能，每天同步一次，同步成功后，会将结果更新到 [https://github.com/anjia0532/gcr.io_mirror][] , 注意，同步时间为 UTC 时间，换成北京时间+8小时即可

## 如何使用

将 `gcr.io/google-containers` 替换为 `anjia0532` 即可.

## 缺少镜像

如果本项目中缺少 `gcr.io` 的镜像,[请提issues][]



[gcr.io]: https://cloud.google.com/container-registry/
[Docker中国官方镜像加速]: https://www.docker-cn.com/registry-mirror
[加速器Daocloud-业界领先的容器云平台]: https://www.daocloud.io/mirror
[https://github.com/anjia0532/gcr.io_mirror]: https://github.com/anjia0532/gcr.io_mirror
[请提issues]: https://github.com/anjia0532/gcr.io_mirror/issues/new
