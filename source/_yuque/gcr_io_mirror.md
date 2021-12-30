---
title: 067-Google Container Registry(gcr.io) 中国可用镜像(长期维护，王者归来)
urlname: gcr_io_mirror
date: 2011-12-30 19:35:21 +0800
tags: [k8s,kubernetes,gcr.io,k8s.gcr.io,docker]
categories: [docker]
---

> 这是坚持技术写作计划（含翻译）的第 67 篇，定个小目标 999，每周最少 2 篇。

17 年在 github 上搞了个项目，用来同步 gcr.io 的镜像 ,详见 [https://anjia0532.github.io/2017/11/15/gcr-io-image-mirror/](https://anjia0532.github.io/2017/11/15/gcr-io-image-mirror/) ,后来因为被 travis 检测到流量异常，认为滥用结果 travis 账号被禁。
​

被禁后，考虑到当时中科大和`*.azk8s.cn` 都提供了加速业务，gcr.io_mirror 已经完成了历史使命，所以一边申请解封 travis 账号，一边将原项目归档，不再提供同步 gcr.io 的任务。
​

前两天接到小伙伴私信，说是中科大和`*.azk8s.cn` 都不再提供服务了，所以就花了点时间，重新搞了下 gcr.io_mirror

<!-- more -->

之前老版本是根据命名空间全量同步，但是实际上在用的过程中，大多是不会有 N 年前的老版本的，所以转换下思路，改成按需拉取，基于这个思路配合 github action 搞了一版新的。
​

## 镜像对应关系

```bash
# 原镜像名称
gcr.io/namespace/image_name:image_tag

# 等同于
anjia0532/namespace.image_name:image_tag

# 特别的对于 k8s.gcr.io
k8s.gcr.io/{image}:{tag} <==> gcr.io/google-containers/{image}:{tag} <==> anjia0532/google-containers.{image}:{tag}
```

## 如何拉取新镜像

[创建 issues](https://github.com/anjia0532/gcr.io_mirror/issues/new?assignees=&labels=porter&template=gcr-io_porter.md&title=%5BPORTER%5D) ,将自动触发 github actions 进行拉取转推到 docker hub

**注意：**

issues 标题必须为 `[PORTER]镜像名:tag` 的格式，
例如 要拉取 `k8s.gcr.io/federation-controller-manager-arm64:v1.3.1-beta.1` 镜像,
则 issues 的标题为
`[PORTER]gcr.io/google-containers/federation-controller-manager-arm64:v1.3.1-beta.1`
issues 的内容无所谓，可以为空
可以参考 [已搬运镜像集锦](https://github.com/anjia0532/gcr.io_mirror/issues?q=is%3Aissue+label%3Aporter+)

**注意:**
本项目目前仅支持 gcr.io 和 k8s.gcr.io 镜像

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/12/30/gcr_io_mirror/)
- 我的掘金
