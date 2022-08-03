---
title: 073-apisix的三种数据备份方案
urlname: apisix-data-backup
date: '2022-04-19 19:35:21 +0800'
tags:
  - apisix
  - etcd
categories:
  - apisix
  - etcd
---

> 这是坚持技术写作计划（含翻译）的第 73 篇，定个小目标 999，每周最少 2 篇。

本文主要讲解 apisix 的数据备份方案。包括 Dashboard 导出，导出 stand-alone yaml，备份 etcd 数据 三种方式。

<!-- more -->

## 灵魂拷问

你们团队有 ETCD 运维经验么？如果没有，你有打算短期内学习 ETCD 的打算么？如果没有，那你做好 ETCD 宕机数据丢失的心理准备了吗？

比如我之前提的 issues [request help: Etcd node high cpu and memory leak](https://github.com/apache/apisix/issues/6124)，建议看一下，里面有介绍自动压缩 etcd

## Dashboard 导出数据

算是官方提供的备份方案，但是只能导出路由，其余的上游(Upstream)，插件，证书等，没有。意味着，如果导入到新集群的话，可能会失败(比如路由里有别的插件，消费者等配置)。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1650363466522-265f1c8c-4dd8-44ac-81e1-e5415a8b8b51.png#clientId=u6a56cfc8-1b59-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=854&id=u5642d489&margin=%5Bobject%20Object%5D&name=image.png&originHeight=854&originWidth=816&originalType=binary∶=1&rotation=0&showTitle=false&size=43908&status=done&style=none&taskId=u82b7c03a-48e9-4a50-9a57-722b98d6493&title=&width=816)

## 导出 stand-alone yaml

根据官方文档 [Stand-alone mode](https://apisix.apache.org/zh/docs/apisix/stand-alone/) 写了个小工具 [https://github.com/anjia0532/discovery-syncer/](https://github.com/anjia0532/discovery-syncer/#api%E6%8E%A5%E5%8F%A3)。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/226273/1650363752599-74107140-f56d-4571-933e-684447e82292.png#clientId=u6a56cfc8-1b59-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=80&id=u7c4b3e52&margin=%5Bobject%20Object%5D&name=image.png&originHeight=80&originWidth=872&originalType=binary∶=1&rotation=0&showTitle=false&size=9131&status=done&style=none&taskId=u4c4286d0-6fa1-4f1a-904e-9769a221c3e&title=&width=872)
吐槽下 apisix 的数据结构稍微有点乱，同一个 key，可能是 数组 []，可能是 对象 {}，虽然是历史债务吧，但是还是显得不严谨。参考我之前提的 issues [Some Admin APIs' data structure are not unified #6105](https://github.com/apache/apisix/issues/6105)

目前支持 Admin 暴露的所有资源。

1. 可以作为数据备份用。
1. 可以结合 Git/SVN 等版本控制工具，用于审计等操作。

同时这个工具也支持同步 nacos/eureka 数据到 apisix 并自动创建 upstream ，以及主持主动上下线 nacos/eureka 服务，可以参考我提的 issues [usercase: golang toolkit-sync eureka/nacos instance info to apisix #5957](https://github.com/apache/apisix/issues/5957)

## 使用 etcdctl 备份还原数据

```bash
# apisix etcd 地址
export ENDPOINT=apisix-etcd:2379

# ETCDCTL_API=3 etcdctl --endpoints $ENDPOINT endpoint status -w=table
# 备份数据
ETCDCTL_API=3 etcdctl --endpoints $ENDPOINT snapshot save snapshot.db

# 使用 linux cronjob + docker run
# 备份文件名为 北京时间 yyyyMMddHHmmss.db
# 并且删除最后修改时间超过10天的老备份文件
docker run anjia0532/etcd-development.etcd:v3.5.3 -rm -v$(pwd)/snapshot:/tmp /bin/sh -c "/usr/local/bin/etcdctl --endpoints=${ENDPOINT} snapshot save /tmp/$(TZ=Asia/Shanghai date +%Y%m%d%H%M%S).db && find /tmp/ -mtime +10 -type f -delete"

# 更建议用 k8s cronjob 来备份，存储到共享存储里

# 查看备份状态
ETCDCTL_API=3 etcdctl --endpoints $ENDPOINT snapshot status snapshot.db -w=table

# 还原快照
# apisix-etcd-0
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
  --name apisix-etcd-0 \
  --initial-cluster apisix-etcd-0=http://apisix-etcd-0:2380,m2=http://apisix-etcd-1:2380,m3=http://apisix-etcd-2:2380 \
  --initial-advertise-peer-urls http://localhost:2380

# apisix-etcd-1
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
  --name apisix-etcd-1 \
  --initial-cluster apisix-etcd-0=http://apisix-etcd-0:2380,m2=http://apisix-etcd-1:2380,m3=http://apisix-etcd-2:2380 \
  --initial-advertise-peer-urls http://localhost:2380

# apisix-etcd-2
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
  --name apisix-etcd-2 \
  --initial-cluster apisix-etcd-0=http://apisix-etcd-0:2380,m2=http://apisix-etcd-1:2380,m3=http://apisix-etcd-2:2380 \
  --initial-advertise-peer-urls http://localhost:2380
```

## 其他方案

关注 [https://github.com/api7/etcd-adapter](https://github.com/api7/etcd-adapter) 项目，改用传统数据库 （比如 pgsql,mysql 等），或者 apisix 团队实现的 btree

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/e78fa84f96fef4e733J60tq8EA~~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2022/04/19/apisix-data-backup/)
- [我的掘金](https://juejin.cn/post/7088270781980868622/)
