
---

title: 028-解决cdh6.2 报 Failed to install Oozie ShareLib

date: 2019-06-12 12:30:53 +0800

tags: [CDH,hadoop,oozie]

categories: 大数据

---

> 这是坚持技术写作计划（含翻译）的第28篇，定个小目标999，每周最少2篇。


本文主要参考 [0590-6.1.0-C6升级过程中Oozie共享库的问题分析](https://cloud.tencent.com/developer/article/1419295) ，这个问题是，cdh6.2的通病，只要安装oozie就会出现(无论是升级，还是新装)。<br />

1. 纠正解决方案里的软连接问题

```bash
cd /opt/cloudera/parcels/CDH/lib/oozie/libtools
ln -s ../../../jars/logredactor-2.0.7.jar logredactor-2.0.7.jar 
```

2. 此方案需要在部署oozie的主机上执行(我一开始执行错节点了)，执行完后重启oozie。

