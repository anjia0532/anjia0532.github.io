---
title: 027-解决cdh6.2中datanode无法启动问题
urlname: hdfs-datanode-start-failed
date: '2019-06-10 21:00:01 +0800'
tags:
  - CDH
  - hadoop
  - 大数据
  - hdfs
  - datanode
categories: 大数据
---

> 这是坚持技术写作计划（含翻译）的第 27 篇，定个小目标 999，每周最少 2 篇。

今天安装 cdh 时遇到 hdfs 启动失败问题，解决起来倒是不麻烦，简单记录下。

<!-- more -->

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560907215000-3fe2af8a-139e-4fc6-ae67-1abb7d5e19e7.png#align=left&display=inline&height=767&originHeight=767&originWidth=1596&size=260810&status=done&width=1596)
关键信息 `java.io.IOException: Incompatible clusterIDs in /dfs/dn: namenode clusterID = cluster74; datanode clusterID = cluster14` 
关键信息，namenode 的 clusterID 和 datanode 的不一致。

原因：执行 hdfs namenode -format 后，current 目录会删除并重新生成，其中 VERSION 文件中的 clusterID 也会随之变化，而 datanode 的 VERSION 文件中的 clusterID 保持不变，造成两个 clusterID 不一致。

方案：

1. 如果是新建的集群，则直接主机目录 `/dfs/dn` （cdh->hdfs->配置-> `dfs.datanode.data.dir` ）下的 current 目录下的文件删除 `rm -rf /dfs/dn/current/*`
2. 如果集群内有数据，则只改 /dfs/dn/current/VERSION  中的 `clusterID=clusterXX`  XX 为正确的 namenode 的 clusterID

重启 datanode 即可
