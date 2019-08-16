
---

title: 034-解决streamsets订阅过期binlog导致无法启动问题

date: 2019-07-22 20:00:52 +0800

tags: [streamsets,sdc,mysql,binlog,etl,数据分析,数据处理]

categories: 大数据

---
> 这是坚持技术写作计划（含翻译）的第34篇，定个小目标999，每周最少2篇。


streamsets停机时间过长，mysql master设置了自动清除binlog，服务启动时报  `ERROR MysqlSource - BinaryLogClient server error: The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.` 

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

尝试了手动设置偏移量无果。(参考资料 [MySQL Binary Log#Initial Offset](https://streamsets.com/documentation/datacollector/3.9.x/help/datacollector/UserGuide/Origins/MySQLBinaryLog.html#ariaid-title5) ）<br />通过查看运行日志，![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563761388749-2ef53ec1-40d5-479b-852e-7dfb7261cd67.png#align=left&display=inline&height=101&name=image.png&originHeight=101&originWidth=304&size=5193&status=done&width=304) <br />发现后台还有在运行之前的offset，突然想起在官方文档.
> **Note:** If you change the GTID mode on the database server after you have run a pipeline with the MySQL Binary Log origin, you must reset the origin and change the format of the initial offset value. Otherwise, the origin cannot correctly read the offset.
> When the pipeline stops, the MySQL Binary Log origin notes the offset where it stops reading. When the pipeline starts again, the origin continues processing from the last saved offset. You can reset the origin to process all requested objects.


以及<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563761746217-5afefc26-937d-44b7-b4c1-ba114e64a154.png#align=left&display=inline&height=139&name=image.png&originHeight=139&originWidth=397&size=14704&status=done&width=397)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563761793852-caf1a2f7-1c81-4b3c-b046-19d2d1dda76e.png#align=left&display=inline&height=240&name=image.png&originHeight=240&originWidth=387&size=24451&status=done&width=387)<br />解决办法就简单了，直接在启动时，Reset Origin即可。会从server的当前binlog进行订阅。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563761663744-640cf864-f48f-4ffa-9a7a-811ce396caa3.png#align=left&display=inline&height=152&name=image.png&originHeight=152&originWidth=349&size=11373&status=done&width=349)

**总结:** <br />多看官方文档，仔细看，另外，尽量别用翻译工具，否则类似该案例中的origin会被翻译成原点，起源等，会无法联想到sdc的一些概念用词。
<a name="qheIM"></a>
## 参考资料

- [我的博客](https://anjia0532.github.io/2019/07/16/fixed-sdc-mysql-binlog-master-purged/)
- [我的掘金](https://juejin.im/post/5d3652f96fb9a07eda0355aa)
- [sdc官方文档>MySQL Binary Log](https://streamsets.com/documentation/datacollector/3.9.x/help/datacollector/UserGuide/Origins/MySQLBinaryLog.html)
- [MySQL · 答疑解惑 · GTID不一致分析](http://mysql.taobao.org/monthly/2016/01/08/)
- [Mysql binlog 查看方法](http://soft.dog/2016/06/13/dig-mysql-binlog/)
- [Mysql之binlog日志说明及利用binlog日志恢复数据操作记录](https://www.cnblogs.com/kevingrace/p/5907254.html)
- [MySQL的binlog日志](https://www.cnblogs.com/martinzhang/p/3454358.html)


