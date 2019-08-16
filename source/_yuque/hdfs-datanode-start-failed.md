
---

title: 027-解决cdh6.2中datanode无法启动问题

date: 2019-06-10 21:00:01 +0800

tags: [CDH,hadoop,大数据,hdfs,datanode]

categories: 大数据

---

> 这是坚持技术写作计划（含翻译）的第27篇，定个小目标999，每周最少2篇。


今天安装cdh时遇到hdfs启动失败问题，解决起来倒是不麻烦，简单记录下。

<!-- more -->

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560907215000-3fe2af8a-139e-4fc6-ae67-1abb7d5e19e7.png#align=left&display=inline&height=767&name=image.png&originHeight=767&originWidth=1596&size=260810&status=done&width=1596)<br />关键信息 `java.io.IOException: Incompatible clusterIDs in /dfs/dn: namenode clusterID = cluster74; datanode clusterID = cluster14` <br />关键信息，namenode的 clusterID和datanode的不一致。

原因：执行hdfs namenode -format后，current目录会删除并重新生成，其中VERSION文件中的clusterID也会随之变化，而datanode的VERSION文件中的clusterID保持不变，造成两个clusterID不一致。

方案：

1. 如果是新建的集群，则直接主机目录 `/dfs/dn` （cdh->hdfs->配置-> `dfs.datanode.data.dir` ）下的current目录下的文件删除 `rm -rf /dfs/dn/current/*` 
1. 如果集群内有数据，则只改 /dfs/dn/current/VERSION 中的 `clusterID=clusterXX`  XX为正确的namenode的clusterID

重启datanode即可

