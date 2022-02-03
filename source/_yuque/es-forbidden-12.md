---
title: '005-blocked by：[FORBIDDEN/12/index read-only / allow delete (api)]'
urlname: es-forbidden-12
date: '2019-02-19 08:31:34 +0800'
tags:
  - es
  - elasticsearch
  - 运维
categories: 运维
---

> 这是坚持技术写作计划（含翻译）的第 5 篇，定个小目标 999，每周最少 2 篇。

吐槽一下，最近有点招 bug，前两天磁盘异常爆满，今天 es 又挂了。

## Logstash ClusterBlockException

logstash 日志中周期性出现 `FORBIDDEN/12/index read-only / allow delete (api)]` ，并且 es 中无法写入新数据。
原因是 ES 主动保护功能，防止 es 集群状态变成红色(RED)或者黄色(YELLOW)
原因有两个:

- 内存不足：JVMMemoryPressure 超过 92%并持续 30 分钟时，ES 触发保护机制，并且阻止写入操作，以防止集群达到红色状态，启用写保护后，写入操作将失败，并且抛出 `ClusterBlockException` ，无法创建新索引，并且抛出 `IndexCreateBlockException` ,当五分钟内恢复到 88%以下时，将禁用写保护
- 磁盘空间不足：es 的默认磁盘水位警戒线是 85%，一旦磁盘使用率超过 85%，es 不会再为该节点分配分片，es 还有一个磁盘水位警戒线是 90%，超过后，将尝试将分片重定位到其他节点。

## 解决方案

- 磁盘扩容
- 删除无用索引
- 将旧索引的副本数调小
- 增加数据节点
- 手动将 `index.blocks.read_only_allow_delete`  改成 false

## 另

还有一种报错是 `blocked by: [FORBIDDEN/8/index write (api)];`  后续再补充

## 参考资料

- [AWS elastic-search. FORBIDDEN/8/index write (api). Unable to write to index](https://stackoverflow.com/questions/44383601/aws-elastic-search-forbidden-8-index-write-api-unable-to-write-to-index)
- [Dynamic Index Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#dynamic-index-settings)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
