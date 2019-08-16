
---

title: 005-blocked by：[FORBIDDEN/12/index read-only / allow delete (api)]

date: 2019-02-19 08:31:34 +0800

tags: [es,elasticsearch,运维]

categories: 运维

---

> 这是坚持技术写作计划（含翻译）的第5篇，定个小目标999，每周最少2篇。


吐槽一下，最近有点招bug，前两天磁盘异常爆满，今天es又挂了。

<a name="11e8ce1c"></a>
## Logstash ClusterBlockException

logstash 日志中周期性出现 `FORBIDDEN/12/index read-only / allow delete (api)]` ，并且es中无法写入新数据。<br />原因是 ES 主动保护功能，防止es集群状态变成红色(RED)或者黄色(YELLOW)<br />原因有两个:

- 内存不足：JVMMemoryPressure 超过92%并持续30分钟时，ES触发保护机制，并且阻止写入操作，以防止集群达到红色状态，启用写保护后，写入操作将失败，并且抛出 `ClusterBlockException` ，无法创建新索引，并且抛出 `IndexCreateBlockException` ,当五分钟内恢复到88%以下时，将禁用写保护
- 磁盘空间不足：es的默认磁盘水位警戒线是85%，一旦磁盘使用率超过85%，es不会再为该节点分配分片，es还有一个磁盘水位警戒线是90%，超过后，将尝试将分片重定位到其他节点。

<a name="de842a6c"></a>
## 解决方案

- 磁盘扩容
- 删除无用索引
- 将旧索引的副本数调小
- 增加数据节点
- 手动将 `index.blocks.read_only_allow_delete` 改成false

<a name="4d13249a"></a>
## 另
还有一种报错是 `blocked by: [FORBIDDEN/8/index write (api)];` 后续再补充

<a name="35808e79"></a>
## 参考资料

- [AWS elastic-search. FORBIDDEN/8/index write (api). Unable to write to index](https://stackoverflow.com/questions/44383601/aws-elastic-search-forbidden-8-index-write-api-unable-to-write-to-index)
- [Dynamic Index Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#dynamic-index-settings)

<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。


