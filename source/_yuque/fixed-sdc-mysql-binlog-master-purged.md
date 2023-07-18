---
title: 034-解决streamsets订阅过期binlog导致无法启动问题
urlname: fixed-sdc-mysql-binlog-master-purged
date: '2019-07-22 20:00:52 +0800'
tags:
  - streamsets
  - sdc
  - mysql
  - binlog
  - etl
  - 数据分析
  - 数据处理
categories: 大数据
---

> 这是坚持技术写作计划（含翻译）的第 34 篇，定个小目标 999，每周最少 2 篇。

streamsets 停机时间过长，mysql master 设置了自动清除 binlog，服务启动时报   `ERROR MysqlSource - BinaryLogClient server error: The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.`

<!-- more -->

```
2019-07-20 11:14:19,713 [user:*admin] [pipeline:demo/demo-e2cf-4f6d-b2dc-ce7fc5aeb041] [runner:] [thread:ProductionPipelineRunnable-demo-e2cf-4f6d-b2dc-ce7fc5aeb041-demo] ERROR MysqlSource - BinaryLogClient server error: The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.
com.github.shyiko.mysql.binlog.network.ServerException: The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.
	at com.github.shyiko.mysql.binlog.BinaryLogClient.listenForEventPackets(BinaryLogClient.java:882)
	at com.github.shyiko.mysql.binlog.BinaryLogClient.connect(BinaryLogClient.java:559)
	at com.github.shyiko.mysql.binlog.BinaryLogClient$7.run(BinaryLogClient.java:793)
	at java.lang.Thread.run(Thread.java:748)
2019-07-20 11:14:19,717 [user:*admin] [pipeline:demo/demo-e2cf-4f6d-b2dc-ce7fc5aeb041] [runner:] [thread:ProductionPipelineRunnable-demo-e2cf-4f6d-b2dc-ce7fc5aeb041-demo] ERROR ProductionPipelineRunner - Pipeline execution failed
com.streamsets.pipeline.api.StageException: MYSQL_006 - MySql server error: The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.
	at com.streamsets.pipeline.stage.origin.mysql.MysqlSource.handleErrors(MysqlSource.java:389)
	at com.streamsets.pipeline.stage.origin.mysql.MysqlSource.produce(MysqlSource.java:225)
	at com.streamsets.datacollector.runner.StageRuntime.lambda$execute$2(StageRuntime.java:295)
	at com.streamsets.pipeline.api.impl.CreateByRef.call(CreateByRef.java:40)
	at com.streamsets.datacollector.runner.StageRuntime.execute(StageRuntime.java:243)
	at com.streamsets.datacollector.runner.StageRuntime.execute(StageRuntime.java:310)
	at com.streamsets.datacollector.runner.StagePipe.process(StagePipe.java:219)
	at com.streamsets.datacollector.execution.runner.common.ProductionPipelineRunner.processPipe(ProductionPipelineRunner.java:817)
	at com.streamsets.datacollector.execution.runner.common.ProductionPipelineRunner.runPollSource(ProductionPipelineRunner.java:561)
	at com.streamsets.datacollector.execution.runner.common.ProductionPipelineRunner.run(ProductionPipelineRunner.java:385)
	at com.streamsets.datacollector.runner.Pipeline.run(Pipeline.java:529)
	at com.streamsets.datacollector.execution.runner.common.ProductionPipeline.run(ProductionPipeline.java:110)
	at com.streamsets.datacollector.execution.runner.common.ProductionPipelineRunnable.run(ProductionPipelineRunnable.java:75)
	at com.streamsets.datacollector.execution.runner.standalone.StandaloneRunner.start(StandaloneRunner.java:701)
	at com.streamsets.datacollector.execution.runner.common.AsyncRunner.lambda$start$3(AsyncRunner.java:151)
	at com.streamsets.pipeline.lib.executor.SafeScheduledExecutorService$SafeCallable.lambda$call$0(SafeScheduledExecutorService.java:226)
	at com.streamsets.datacollector.security.GroupsInScope.execute(GroupsInScope.java:33)
	at com.streamsets.pipeline.lib.executor.SafeScheduledExecutorService$SafeCallable.call(SafeScheduledExecutorService.java:222)
	at com.streamsets.pipeline.lib.executor.SafeScheduledExecutorService$SafeCallable.lambda$call$0(SafeScheduledExecutorService.java:226)
	at com.streamsets.datacollector.security.GroupsInScope.execute(GroupsInScope.java:33)
	at com.streamsets.pipeline.lib.executor.SafeScheduledExecutorService$SafeCallable.call(SafeScheduledExecutorService.java:222)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:180)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:293)
	at com.streamsets.datacollector.metrics.MetricSafeScheduledExecutorService$MetricsTask.run(MetricSafeScheduledExecutorService.java:100)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
Caused by: com.github.shyiko.mysql.binlog.network.ServerException: The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.
	at com.github.shyiko.mysql.binlog.BinaryLogClient.listenForEventPackets(BinaryLogClient.java:882)
	at com.github.shyiko.mysql.binlog.BinaryLogClient.connect(BinaryLogClient.java:559)
	at com.github.shyiko.mysql.binlog.BinaryLogClient$7.run(BinaryLogClient.java:793)
	... 1 more
```

尝试了手动设置偏移量无果。(参考资料  [MySQL Binary Log#Initial Offset](https://streamsets.com/documentation/datacollector/3.9.x/help/datacollector/UserGuide/Origins/MySQLBinaryLog.html#ariaid-title5) ）
通过查看运行日志，![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563761388749-2ef53ec1-40d5-479b-852e-7dfb7261cd67.png#align=left&display=inline&height=101&originHeight=101&originWidth=304&size=5193&status=done&width=304) 
发现后台还有在运行之前的 offset，突然想起在官方文档.

> **Note:** If you change the GTID mode on the database server after you have run a pipeline with the MySQL Binary Log origin, you must reset the origin and change the format of the initial offset value. Otherwise, the origin cannot correctly read the offset.
> When the pipeline stops, the MySQL Binary Log origin notes the offset where it stops reading. When the pipeline starts again, the origin continues processing from the last saved offset. You can reset the origin to process all requested objects.

以及
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563761746217-5afefc26-937d-44b7-b4c1-ba114e64a154.png#align=left&display=inline&height=139&originHeight=139&originWidth=397&size=14704&status=done&width=397)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563761793852-caf1a2f7-1c81-4b3c-b046-19d2d1dda76e.png#align=left&display=inline&height=240&originHeight=240&originWidth=387&size=24451&status=done&width=387)
解决办法就简单了，直接在启动时，Reset Origin 即可。会从 server 的当前 binlog 进行订阅。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563761663744-640cf864-f48f-4ffa-9a7a-811ce396caa3.png#align=left&display=inline&height=152&originHeight=152&originWidth=349&size=11373&status=done&width=349)

**总结:** 
多看官方文档，仔细看，另外，尽量别用翻译工具，否则类似该案例中的 origin 会被翻译成原点，起源等，会无法联想到 sdc 的一些概念用词。

## 参考资料

- [我的博客](https://anjia0532.github.io/2019/07/16/fixed-sdc-mysql-binlog-master-purged/)
- [我的掘金](https://juejin.im/post/5d3652f96fb9a07eda0355aa)
- [sdc 官方文档>MySQL Binary Log](https://streamsets.com/documentation/datacollector/3.9.x/help/datacollector/UserGuide/Origins/MySQLBinaryLog.html)
- [MySQL · 答疑解惑 · GTID 不一致分析](http://mysql.taobao.org/monthly/2016/01/08/)
- [Mysql binlog 查看方法](http://soft.dog/2016/06/13/dig-mysql-binlog/)
- [Mysql 之 binlog 日志说明及利用 binlog 日志恢复数据操作记录](https://www.cnblogs.com/kevingrace/p/5907254.html)
- [MySQL 的 binlog 日志](https://www.cnblogs.com/martinzhang/p/3454358.html)
