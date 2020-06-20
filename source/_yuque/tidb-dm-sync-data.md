
---

title: 041-使用DM进行同步上游数据到tidb

date: 2019-08-22 19:35:21 +0800

tags: [mysql,tidb,tikv,dm,数据同步]

categories: 数据库

---

> 这是坚持技术写作计划（含翻译）的第41篇，定个小目标999，每周最少2篇。


回想一下之前数据同步一般会使用一些ETL工具，例如 [DataX](https://github.com/alibaba/DataX) ,[StreamSets](http://streamsets.com/) ,[Kettle](https://community.hitachivantara.com/s/article/data-integration-kettle),[sqoop](http://sqoop.apache.org/),[nifi](http://nifi.apache.org/),当然也可能使用原始的mysqldump进行人肉苦逼同步。因为每个团队的背景不一样，没法简单的说哪种工具更好，更优，还是需要落地才行。所以本文就不过多介绍不同ETL工具间的优劣了，毕竟PHP是世界上最好的语言。

本文主要讲解，如何使用tidb官方的同步工具DM进行数据同步。先搬运一下tidb官方对dm的简介
> DM (Data Migration) 是一体化的数据同步任务管理平台，支持从 MySQL 或 MariaDB 到 TiDB 的全量数据迁移和增量数据同步。使用 DM 工具有利于简化错误处理流程，降低运维成本。

略微吐槽下，tidb的官方技术栈略有点复杂，比如，广义的tidb，一般是指pd,tidb,tikv这类核心组件(参考 [TiDB 整体架构](https://pingcap.com/docs-cn/v3.0/architecture/))，但是，部署的话，得用ansible部署，监控呢，得学会看官方提供的 Prometheus+Grafana,同步数据的话，又的看 [mydumper](https://pingcap.com/docs-cn/v3.0/reference/tools/mydumper/) , [loader](https://pingcap.com/docs-cn/v3.0/reference/tools/loader/),[syncer](https://pingcap.com/docs-cn/v3.0/reference/tools/syncer/), [Data Migration](https://pingcap.com/docs-cn/v3.0/reference/tools/data-migration/overview/) ,[TiDB Lightning](https://pingcap.com/docs-cn/v3.0/reference/tools/tidb-lightning/overview/) , 管理tidb集群的话，又会用到一些工具，比如，pd control,pd recover,tikv control,tidb controller,如果要给tidb开启binlog，用于同步到其他tidb或者mysql集群，又要研究 Pump,Drainer,binlogctl ... 

就感觉tidb团队，一看就是出身大户人家，看官方建议的集群配置吧。

- tidb集群最低配置要求

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566446890363-012769a1-301d-4504-b265-426b5aa564c3.png#align=left&display=inline&height=456&name=image.png&originHeight=456&originWidth=980&size=49012&status=done&width=980)

- dm实例配置

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566446936450-c3a1f644-7f96-4d3f-acff-4e7bb705fa51.png#align=left&display=inline&height=225&name=image.png&originHeight=225&originWidth=376&size=12816&status=done&width=376)

- tidb-lighting配置要求(超过200G以上的迁移，建议用tidb-lighting)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566446971353-736f2b20-bbfe-47ef-9c7f-d2077c3efb4c.png#align=left&display=inline&height=664&name=image.png&originHeight=664&originWidth=1035&size=88797&status=done&width=1035)

- tidb binlog的配置

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566447000079-99ece3b7-2ce8-4e50-a373-05d90a3551e2.png#align=left&display=inline&height=283&name=image.png&originHeight=283&originWidth=1226&size=48745&status=done&width=1226)

所以，几乎每一个用tidb的人，第一件事，都是，如何修改ansible的参数，绕过检测 [手动滑稽]，人嘛，都是，一边吐槽XX周边工具太少，又会吐槽XX太多，学不动，像是初恋的少女，等远方的男友，怕他乱来，又怕他不来，哈哈

对比一下友商的<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566457220555-8735c5a8-62a8-46c3-95c6-22e611cd623d.png#align=left&display=inline&height=434&name=image.png&originHeight=434&originWidth=837&size=72883&status=done&width=837)

其实tidb的官方文档，写的还挺详细的，就是不太像是给入门的人看的 [手动捂脸]，本文主要是结合我在使用DM过程中，写一下遇到的问题，以及群内大牛的解答

<!-- more -->

<a name="Zu2pv"></a>
## 简介
DM 简化了单独使用mysqldumper，loader，syncer的工作量，从易用性，健壮性和可观测等方面来看，建议使用DM。

注意一下[官方文档](https://pingcap.com/docs-cn/v3.0/reference/tools/data-migration/overview/)写的限制条件。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566447862468-9a9f37ab-cd91-4560-9532-81bc0c250c6b.png#align=left&display=inline&height=814&name=image.png&originHeight=814&originWidth=1011&size=106197&status=done&width=1011)

<a name="vCPLU"></a>
## 部署DM

参考  [使用 DM-Ansible 部署 DM 集群](https://pingcap.com/docs-cn/v3.0/how-to/deploy/data-migration-with-ansible/) 

如无特殊说明都按照官方文档操作。

第五步配置互信时，servers 是要部署DM的节点ip，注意当前登录名，确保是tidb(执行 `woami`  )

```bash
vi hosts.ini
[servers]
172.16.10.71
172.16.10.72
172.16.10.73

[all:vars]
username = tidb
```

执行 `ansible-playbook -i hosts.ini create_users.yml -u root -k` 时，如果是使用 ssh key的话，可以<br />`ansible-playbook -i hosts.ini create_users.yml -u root -k --private-key /path/to/your/keyfile`

第7步配置worker时，需要注意，如果要增量或者全量，并且上游服务的binlog被删过，并且是gtid格式的，需要执行 `show VARIABLES like 'gtid_purged'` 如果有值，则需要指定  `relay_binlog_gtid`  ,否则会报 `close sync with err: ERROR 1236 (HY000): The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.` 此时停掉worker后，修改 `relay_binlog_gtid` 重启无效，是需要修改meta文件的 `/tidb/deploy/dm_worker_/relay_log/f5df-11e7-a1dd.000001/relay.meta` 感谢 军军![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566451401212-2b73ea3f-809f-40b4-b572-798152bb4dec.png#align=left&display=inline&height=807&name=image.png&originHeight=807&originWidth=520&size=82610&status=done&width=520)<br />另外需要配置  `enable_gtid=true` <br />如果不是gtid格式的，则需要修改这个 `relay_binlog_name` （在mysql执行 `show BINARY logs` ）<br />`mysql_password` 需要使用 `dmctl -encrypt 你的密码` 如果找不到dmctl，确保执行了 `ansible-playbook local_prepare.yml` 后在 `/path/to/dm-ansible/resources/bin/dmctl` 

dm的worker支持单机多实例，也支持单机单实例(推荐) ,如果因为资源问题，要开启单机多实例的话，

```bash
[dm_worker_servers]
dm_worker1_1 ansible_host=172.16.10.72 server_id=101 deploy_dir=/data1/dm_worker dm_worker_port=8262 mysql_host=172.16.10.81 mysql_user=root mysql_password='VjX8cEeTX+qcvZ3bPaO4h0C80pe/1aU=' mysql_port=3306
dm_worker1_2 ansible_host=172.16.10.72 server_id=102 deploy_dir=/data2/dm_worker dm_worker_port=8263 mysql_host=172.16.10.82 mysql_user=root mysql_password='VjX8cEeTX+qcvZ3bPaO4h0C80pe/1aU=' mysql_port=3306

dm_worker2_1 ansible_host=172.16.10.73 server_id=103 deploy_dir=/data1/dm_worker dm_worker_port=8262 mysql_host=172.16.10.83 mysql_user=root mysql_password='VjX8cEeTX+qcvZ3bPaO4h0C80pe/1aU=' mysql_port=3306
dm_worker2_2 ansible_host=172.16.10.73 server_id=104 deploy_dir=/data2/dm_worker dm_worker_port=8263 mysql_host=172.16.10.84 mysql_user=root mysql_password='VjX8cEeTX+qcvZ3bPaO4h0C80pe/1aU=' mysql_port=3306
```

注意 `server_id=101 deploy_dir=/data1/dm_worker dm_worker_port=8262` 别冲突，尤其是 `deploy_dir`  和 `dm_worker_port` 

第九步,如果部署dm的节点数太多，可以提升并发数 `ansible-playbook deploy.yml -f 10` <br />第十步,启动 `ansible-playbook start.yml` 

上述是简单操作， 如果涉及到复杂的，例如，扩容，缩容dm节点，重启dm-master或者dm-worker，可以参考 [DM 集群操作](https://pingcap.com/docs-cn/v3.0/reference/tools/data-migration/cluster-operations/)

<a name="SSt0y"></a>
## 配置DM
<a name="vGJ15"></a>
### 如何看文档
配置dm-worker的tasks，三段文档结合着看<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566452687610-157bf15c-8438-4e56-a2e0-a5e503b9f5e6.png#align=left&display=inline&height=183&name=image.png&originHeight=183&originWidth=193&size=5688&status=done&width=193)<br />一般场景，使用 [Data Migration 简单使用场景](https://pingcap.com/docs-cn/v3.0/reference/tools/data-migration/usage-scenarios/simple-synchronization/) 即可满足。
<a name="IFrX2"></a>
### 库重命名
库重命名,将上游的user，备份成user_north库。另外，不支持实例内databases批量加前缀或者后缀。所以，有多少个需要重命名的，就乖乖写多少个吧

```yaml
routes:
  ...
  instance-1-user-rule:
    schema-pattern: "user"
    target-schema: "user_north"
```

<a name="RZfbR"></a>
### 忽略库或者表

```yaml
black-white-list:
  log-ignored:
    ignore-dbs: ["log"] # 忽略同步log库
    ignore-tables:
    - db-name: "test" # 忽略同步log库内的test表
      tbl-name: "log"
```

<a name="REr42"></a>
### 尽量使用白名单
个人建议尽量使用白名单进行同步，防止因为新增库dm校验不通过，导致task被pause掉。此时的假设是，同步任务是严谨的，不应该出现不可控因素。当然这只是建议。<br />如果 上游数据库有，a,b,c三个库，前期白名单只写了a,b，进行全量+增量同步(all模式),并且task的unit已经是sync(非dump)，如果此时要同步库，此时如果只是简单改白名单，然后pause-task,update-task,resume-task,会报表不存在的错。具体的解决办法，可以参见 我在tug上提的问题 [https://asktug.com/t/db/616](https://asktug.com/t/db/616) ，感谢 wangxj@pingCAP
```yaml
black-white-list:
  rule-1:
    do-dbs: ["~^test-*"] 同步所有test-开头的库
```

<a name="6Dhk0"></a>
### 忽略drop和truncate操作
毕竟使用DM是用于同步数据，在一定程度上也可以用于灾备场景使用，万一业务库被人drop，truncate了，tidb这还可以救命，所以建议忽略这些危险操作，有必要的，可以人工去执行。

```yaml
filters:
  ...
  store-filter-rule:
    schema-pattern: "store"
    events: ["drop database", "truncate table", "drop table"]
    action: Ignore
```

使用DM Portal生成配置文件，但是Portal生成的是不支持gtid的，详见  军军 的解释，详细使用，参见 [DM Portal 简介](https://pingcap.com/docs-cn/v3.0/reference/tools/data-migration/dm-portal/)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566451855483-cd157a57-8a54-430f-a416-3fb610ee930e.png#align=left&display=inline&height=691&name=image.png&originHeight=691&originWidth=1243&size=105929&status=done&width=1243)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566451876609-53fd8a8a-bd6e-4ebc-8597-66dd754fbe75.png#align=left&display=inline&height=766&name=image.png&originHeight=766&originWidth=506&size=76630&status=done&width=506)

<a name="AX9xH"></a>
## 启动DM
<a name="IKnLH"></a>
### 常规操作
参考 [管理数据同步任务](https://pingcap.com/docs-cn/v3.0/reference/tools/data-migration/manage-tasks/) <br />使用 `./dmctl -master-addr 172.16.30.14` 进入交互式命令界面(不支持非交互式的，导致我在使用中遇到，当报错信息特别大时，超过缓冲区，会导致看不到有效的报错信息，check-task xx-task >result.log 这种的不支持，希望后边能改进下 )

- 启动任务 `start-task /path/to/task.yaml`  注意是task文件，而不是任务名
- 查询任务 `query-status [task-name]` task-name是可选的，不填查所有，填了，只查指定的
- 暂停任务 `pause-task task-name` ,如果要更新task文件(update-task) task一定要处于pause状态(报错导致的pause也行)
- 恢复同步 `resume-task task-name` 处于暂停(pause)的任务要恢复，需要使用resume-task ,如果是 `full` 或者 `all` 时unit处于dump状态的(非load)，resume-task时会清空已经dump到本地的文件，重新拉取(想想200多G的数据库，到99%了，突然pause了，就肝儿颤)
- 更新同步任务 `update-task /path/to/task.yaml` 注意是task文件，不是任务名,执行更新操作，必须是pause状态，所以，尽量别再dump时执行update，要执行也是在前期执行。执行后，需要使用 `resume-task` 启动已pause的任务
- 停止任务 `stop-task task-name` 

<a name="ySF6E"></a>
### 常见问题

1. check通过，start时报错，或者start也正常，query时报错，这种错，有跟没有区别不大，只能ssh到worker节点，看日志 `/path/to/deploy/log/` 

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566454565615-d005ec59-0f1d-4bc8-99e3-8c9dc7696c2a.png#align=left&display=inline&height=374&name=image.png&originHeight=374&originWidth=988&size=23249&status=done&width=988)

2. `Couldn't acquire LOCK BINLOG FOR BACKUP, snapshots will not be consistent:Access denied; you need (at least one of) the SUPER privilege(s) for this operation` 

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566454666623-50b2ef14-1093-4c93-b02d-3785ed9bca54.png#align=left&display=inline&height=145&name=image.png&originHeight=145&originWidth=1530&size=40218&status=done&width=1530)<br />如果没有reload权限，会报错，但是不会终止操作，可以忽略<br />注意，如果是阿里云的rds的话，默认是把reload权限给去掉的。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566454840223-ba2f3f4b-8943-4c1c-a421-ff14cc7d6716.png#align=left&display=inline&height=440&name=image.png&originHeight=440&originWidth=915&size=52226&status=done&width=915)

3. sql-mode 不一致引起的问题，mysql默认的sql-mode是空字符串,参考 [SQL 模式](https://pingcap.com/docs-cn/v3.0/reference/sql/sql-mode) ，排查方式 `SELECT @@sql_mode` ,如果是tidb是新库，可以 `set global sql_mode='';` 如果要改mysql的话，需要写到 `my.ini` 里，防止重启失效。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566455511112-5409abd2-560a-4d8c-8a44-56d47fea9faa.png#align=left&display=inline&height=755&name=image.png&originHeight=755&originWidth=516&size=77674&status=done&width=516)

4. all模式同步数据，如果task状态已经是sync，此时这个task白名单新增库或者表会导致报错，表不存在，然后task被pause。要么使用full-task+incremental-task两个文件，每次白名单新增时，先更新full-task，再更新incremental-task，要么直接新启一个task，用于full更新白名单库，然后改all的task，重启即可。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566455585953-ef546b3c-d3b2-4fde-aade-27e6a4dcab6c.png#align=left&display=inline&height=550&name=image.png&originHeight=550&originWidth=535&size=60166&status=done&width=535)

5. 处于dump的任务，一旦pause了，再次resume，会删除已dump的数据文件，重新拉取

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1566455624234-f4832af6-0363-4df5-b204-2beab0e9f3f9.png#align=left&display=inline&height=855&name=image.png&originHeight=855&originWidth=528&size=103391&status=done&width=528)
<a name="ZfXWN"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。<br />长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。

<a name="35808e79"></a>
## 参考资料

- [我的博客](http://anjia0532.github.io/2019/08/21/tidb-dm-sync-data)
- [我的掘金](https://juejin.im/post/5d5e3d2bf265da03e921d0ea)
- [DataX在有赞大数据平台的实践](https://tech.youzan.com/datax-in-action/)
- [Data Migration 常见问题](https://pingcap.com/docs-cn/v3.0/faq/data-migration/)
- [使用 DM-Ansible 部署 DM 集群](https://pingcap.com/docs-cn/v3.0/how-to/deploy/data-migration-with-ansible/)

