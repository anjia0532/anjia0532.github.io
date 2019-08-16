
---

title: 035-解决streamsets jdbc全量模式数据重复问题

date: 2019-07-22 21:00:00 +0800

tags: [streamsets,sdc,mysql,binlog,etl,数据分析,数据处理]

categories: 大数据

---

> 这是坚持技术写作计划（含翻译）的第35篇，定个小目标999，每周最少2篇。


本文主要解决在使用streamsets的JDBC Query Consumer Origin组件消费时，使用全量模式(Full Mode)，数据重复问题。

<!-- more -->

在之前一篇《[033-史上最全-mysql迁移到clickhouse的5种办法](https://anjia0532.github.io/2019/07/17/mysql-to-clickhouse/#streamsets)》中，有介绍如何使用JDBC Query Consumer全量导出，但是有人反馈因为streamsets的管道(pipeline)一直在重复运行，导致最后数据是重复的。

实际上在官方文档有讲 [Full and Incremental Mode](https://streamsets.com/documentation/datacollector/latest/help/datacollector/UserGuide/Origins/JDBCConsumer.html#ariaid-title6)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563874566018-7fd172c4-91aa-4c76-a6af-cf7211443aca.png#align=left&display=inline&height=476&name=image.png&originHeight=476&originWidth=1187&size=73254&status=done&width=1187)

主要看提示(Tip)部分，如果只想执行一次查询后就停止pipeline，应该配置origin的generate events 并且使用Pipeline Finisher来自动停止pipeline，更多信息参见 Event Generation.

在jdbc origin勾选 Produce Events

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563875657778-70e75154-a57b-444d-bf73-4315d05e9fc4.png#align=left&display=inline&height=692&name=image.png&originHeight=692&originWidth=821&size=56065&status=done&width=821)<br />从组件选则Pipeline Finisher，并且配置 Preconditions 为 `${record:eventType() == 'no-more-data'}` 即可<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563875726527-ed22dd7a-cff4-4b10-ac57-2e29ce2f0075.png#align=left&display=inline&height=618&name=image.png&originHeight=618&originWidth=1351&size=70288&status=done&width=1351)




<a name="5OqvA"></a>
## 参考资料

- [我的博客](https://anjia0532.github.io/2019/07/22/sdc-jdbc-full-mode)
- [我的掘金](https://juejin.im/post/5d36dbca5188257b775d4b40)
- [Full and Incremental Mode](https://streamsets.com/documentation/datacollector/latest/help/datacollector/UserGuide/Origins/JDBCConsumer.html#ariaid-title6)
- [Event Generation](https://streamsets.com/documentation/datacollector/latest/help/datacollector/UserGuide/Origins/JDBCConsumer.html#concept_o1c_kwr_kz)
- [Case Study: Stop the Pipeline](https://streamsets.com/documentation/datacollector/latest/help/datacollector/UserGuide/Event_Handling/EventFramework-Title.html#concept_kff_ykv_lz)

