
---

title: 038-拯救大兵瑞恩之Tidb如何在Tikv损坏的情况下恢复

date: 2019-08-01 22:35:41 +0800

tags: [数据库,mysql,database,tidb,tikv,dba]

categories: 数据库

---
> 这是坚持技术写作计划（含翻译）的第38篇，定个小目标999，每周最少2篇。


很喜欢TiDB的设计哲学，比如，数据库就活该要么OLAP，要么OLTP么，为嘛就不能兼顾一下？数据量大了后，就一定要反人类的分库分表么？你当分库分表好玩么？分库一时爽，维护火葬场！

尤其在一些中小型团队里，为了数据分析搞一套hadoop，约等于为了喝牛奶，从牛崽开始养一头奶牛。一路上明坑暗坑不断。

考虑到学习成本，迁移成本(高度兼容MySQL-但不是100%)，运维成本(支持Ansible-团队有ansible运维经验)，使用成本(相比hadoop)，硬件成本(相比hadoop)，收益(不用分开分表，支持olap和oltp，支持分布式事务,支持TiSpark,支持tikv，自带同步工具)等。

好了，疑似广告的一段话说完了，回归正题，介绍是如何悲催的遇上整栋大厦停电，并且恰好tikv文件损坏，以及如何在Tidb各位大佬的远程文字指导下，一步步把心态从删库跑路，转变成说不定还有救，以及，我擦，真救回来的坎坷心路旅程。

因为Tidb之前是别的同事负责，刚接过来不久，对Tidb整个的掌握还很初级。真·面向故障学习！

全文主要是对本次事故的回溯，琐碎细节较多，介意的可以直接看最后。

<!-- more -->

集群环境

| name | ip | service |
| --- | --- | --- |
| tikv-1 | 192.168.1.200 | tikv |
| tikv-2 | 192.168.1.201 | tikv |
| tikv-3 | 192.168.1.202 | tikv(有坏region) |
| pd | 192.168.1.203 | pd |
| tidb | 192.168.1.204 | tidb,monitoring |
| sn-data-node-1 | 192.168.1.216 | tidb |
| sn-data-node-2 | 192.168.1.217 | tidb |
| tikv-4 | 192.168.1.218 | tikv(有坏region) |


<a name="H7w7e"></a>
## 悲催之始

上午coding正嗨，突发性停电<br />来电后 ssh 到 tidb的ansible主控机 `ansible-playbook start.yml` <br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564644740298-54b52606-d594-4ba7-b98c-0cb6c21137df.png#align=left&display=inline&height=527&name=image.png&originHeight=527&originWidth=1329&size=82800&status=done&width=1329)

事情有点不妙，但是扔不死心的， `stop and start` 一通后，仍是这个结果。事情有点大条。

好在之前偷偷潜伏到TiDB的官方群里，没事就听各位大佬吹水打屁，多少受了点熏陶。撸起袖子，开搞。

<a name="TCaY3"></a>
## 定位问题
<a name="ckulg"></a>
### Round 1 懵逼树上懵逼果
先看官方文档<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564648950258-956bb3fb-25de-467f-b6cf-82a7df480f49.png#align=left&display=inline&height=604&name=image.png&originHeight=604&originWidth=1261&size=102094&status=done&width=1261)<br />好吧，跟没看区别不大。

既然是tidb起不来，就先看tidb的日志(实际上应该看http://prometheus:9090/targets ，因为不太熟悉，所以走了弯路，为嘛不看grafana,是因为tidb那卡到后，ansible就自动退出了，没有起grafana)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564648871805-b912a3d1-b5bd-456a-9d4c-8e7db9a47678.png#align=left&display=inline&height=558&name=image.png&originHeight=558&originWidth=1206&size=122785&status=done&width=1206)<br />暴露的是连接两个tikv报错，这是前期比较关键的线索，起码有初步排查方向了。<br />另外从日志看到，疑似报空间不足，实际上没意义，在两台tikv执行 `df -i` `df -h` 来看，都很充足。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564649022668-c8585d09-5958-4143-8314-05fd56deccb4.png#align=left&display=inline&height=88&name=image.png&originHeight=88&originWidth=1538&size=21191&status=done&width=1538)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564649493786-8d8b7428-0cde-447a-a69f-8b4eeeff6f19.png#align=left&display=inline&height=806&name=image.png&originHeight=806&originWidth=556&size=109795&status=done&width=556)<br />群内 @张曾钧@PingCAP 大佬开始介入，并且开始了将近8个小时的细心和耐心的指导，讲真，PingCAP团队是我见过最热心耐心的团队，素未蒙面，但乐于助人[呲牙]

此时，通过看官方文档，尝试性，试了下Prometheus可以打开，<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564649685360-a05e8e44-b756-4fdd-ab74-1032893b8820.png#align=left&display=inline&height=522&name=image.png&originHeight=522&originWidth=497&size=38639&status=done&width=497)<br />能看出有两个TiKV down掉，正好是上面两个。PS ,我是事后才发现的，当时我一直认为TiKV是起来的[捂脸]<br />插播一下，[TiDB 整体架构](https://pingcap.com/docs-cn/v3.0/architecture/) ，不多解释。建议看看，可以了解一下TiDB的架构原理，比如，TiDB,TiKV,PD等的职责。<br />![](https://cdn.nlark.com/yuque/0/2019/png/226273/1564649874173-be2c66cd-022a-446b-bed8-fadd92e8c0fa.png#align=left&display=inline&height=1112&originHeight=1112&originWidth=1996&size=0&status=done&width=1996)
> 小结： Round 1 以找出两个TiKV down结束，效率低到羞愧。


<a name="wnRsx"></a>
### Round 2 懵逼树前排排坐
注意，下面如无特殊说明，一律是TiKV关掉状态下，执行命令。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564650252340-21759119-1843-4434-867a-9963ea765209.png#align=left&display=inline&height=905&name=image.png&originHeight=905&originWidth=547&size=98917&status=done&width=547)![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564650290157-89fc6eec-843d-4e59-a186-c45f21487c44.png#align=left&display=inline&height=856&name=image.png&originHeight=856&originWidth=552&size=112176&status=done&width=552)<br />使用@唐刘@PingCAP的方法 `grep -B 50 Welcome` ,开始接触事发原因了。
> 更多的grep的方法(-A -B -C)，参考 man grep 或者 [GREP(1)](http://man7.org/linux/man-pages/man1/grep.1.html) ，因为tikv启动会打印Welcome，所以有理由认为，每次的Welcome之前的，肯定是上次退出的原因。


![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564650861003-ced69111-e25b-4e7a-91fa-1e07eeead844.png#align=left&display=inline&height=850&name=image.png&originHeight=850&originWidth=1530&size=211744&status=done&width=1530)<br />至此，出现了第一个坏掉的region。<br />当时执行了 `./pd-ctl store -d -u http://127.0.0.1:2379` <br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564651001470-549e9680-0b9f-47dd-9713-3e4833167f6e.png#align=left&display=inline&height=554&name=image.png&originHeight=554&originWidth=1059&size=55248&status=done&width=1059)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564650935734-5b346583-f35d-4b50-a788-0bc7840a30d3.png#align=left&display=inline&height=448&name=image.png&originHeight=448&originWidth=399&size=18168&status=done&width=399)<br />找到挂掉的两个TiKV的store id，跟上图的![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564651146956-35cba4a9-55f6-4577-b434-264f20faea47.png#align=left&display=inline&height=46&name=image.png&originHeight=46&originWidth=125&size=3516&status=done&width=125) 68935能对起来。

此时救苦救难的  大佬 提供了 [TiKV Control 使用说明#恢复损坏的-mvcc-数据](https://pingcap.com/docs-cn/v3.0/reference/tools/tikv-control/#恢复损坏的-mvcc-数据)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564651249500-2b1a9f82-1b58-4969-8601-1bc01e2402e8.png#align=left&display=inline&height=675&name=image.png&originHeight=675&originWidth=551&size=81077&status=done&width=551)![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564651219522-f6219985-567d-487c-bafd-71983748cb57.png#align=left&display=inline&height=682&name=image.png&originHeight=682&originWidth=1248&size=108238&status=done&width=1248)<br />实际上执行后，没啥效果，后来发现是因为此region超过一半副本出问题了，recover-mvcc 没法恢复。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564652211716-e27ec913-9262-4429-a3bb-9a9043625b65.png#align=left&display=inline&height=49&name=image.png&originHeight=49&originWidth=857&size=7616&status=done&width=857)
> Round 2 结束，找到了救命稻草，TiKV Control 和PD Control,但是，事情远没这么简单。


<a name="EyoeN"></a>
### Round 3 渐入佳境
中间出了个差点搞死自己的小插曲<br />`/home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d store delete 10` 自作聪明的以为，TiKV 已经没救了，执行了store delete 操作。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564651626772-b211f752-e5e4-4786-b1bd-0ba2b23557cb.png#align=left&display=inline&height=602&name=image.png&originHeight=602&originWidth=1301&size=92940&status=done&width=1301)<br />但是实际上还有救，所以又变成了，如何把已经delete掉的store，再度挂上去。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564651566804-822ec7d1-84f8-41f5-936f-7ed676bd83d0.png#align=left&display=inline&height=503&name=image.png&originHeight=503&originWidth=552&size=62399&status=done&width=552)<br />根据  大佬的 `curl -X POST http://${pd_ip}:2379/pd/api/v1/store/${store_id}/state?state=Up` 成功挂上，当然还是down的状态。

根据![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564652020901-07018aee-c4fa-49b8-bc3c-fbf3e6c79338.png#align=left&display=inline&height=352&name=image.png&originHeight=352&originWidth=1269&size=68592&status=done&width=1269)<br />`tikv-ctl --db /path/to/tikv/db bad-regions` 两台坏的，分别如下<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564651955786-839ddeb4-24c4-48ea-ac4c-5388070ad861.png#align=left&display=inline&height=84&name=image.png&originHeight=84&originWidth=1008&size=15053&status=done&width=1008)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564652396271-4b1eb843-e26f-49e4-99c3-b69dfb1ca5f3.png#align=left&display=inline&height=53&name=image.png&originHeight=53&originWidth=813&size=9564&status=done&width=813)<br />发现坏掉的 region是 31101（实际上因为用的是2.1.4，每次只显示一个，处理完后，才会显示下一个，效率很低，后来在 [@戚铮](#) 大佬的指导下，换用tikv-ctl 3.x ，每次可以显示全部的坏的region ）<br />在好的节点上执行，也不是文中的all regions are healthy ，实际上是因为，数据文件被占用，没法获取句柄，停掉就行 `ansible-playbook stop.yml  -l tikv_servers` 停掉全部tikv节点<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564652278072-088aa64f-abf6-422e-bb3d-8f48911b1bcc.png#align=left&display=inline&height=150&name=image.png&originHeight=150&originWidth=1328&size=26658&status=done&width=1328)

此时@张曾钧@PingCAP 提示用 `tikv-ctl --db /path/to/tikv/db tombstone -p 127.0.0.1:2379 -r`   把坏的 region 设置为tombstone ，但是报错<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564652514513-9392a0eb-14f1-4db0-9b53-913755ab9c2e.png#align=left&display=inline&height=55&name=image.png&originHeight=55&originWidth=984&size=12889&status=done&width=984)

通过执行 pd-ctl region 31101 发现<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564652666941-f219b6a8-177d-49fe-97cd-0c4b598c9b6f.png#align=left&display=inline&height=471&name=image.png&originHeight=471&originWidth=920&size=44057&status=done&width=920)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564652692433-3691b53b-baf4-4a07-b48b-04baffb296a5.png#align=left&display=inline&height=252&name=image.png&originHeight=252&originWidth=339&size=8996&status=done&width=339)<br />这个region有两个副本是在坏节点上，超过一半损坏（剧透一下，实际上最后发现，出问题的都是损坏超过1半的，1/3的都自己恢复了）

尝试执行 operator add remove-peer 发现删不掉。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564652808996-cc279c2f-ba16-411e-8eb5-c00c43839a64.png#align=left&display=inline&height=501&name=image.png&originHeight=501&originWidth=992&size=89355&status=done&width=992)

此时 戚铮 大佬出场。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564652991806-2c3f7356-46a3-4661-8d94-fe002d67e420.png#align=left&display=inline&height=724&name=image.png&originHeight=724&originWidth=530&size=86534&status=done&width=530)<br />经过一番测试，发现 region31101很坚挺，使用 `recover-mvcc`  恢复不了，前面说了是因为损失超过一半副本的原因，使用 `operator add remove-peer` 删不了，估计也是。

<a name="zvbGH"></a>
### Round 4 貌似解决
不能因为一颗老鼠屎，坏了一锅汤，部分region坏掉了，先尝试强制恢复试试，保证别的正常吧。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564653280296-469366c4-a6ae-47a8-a497-40bd7ce7fa06.png#align=left&display=inline&height=582&name=image.png&originHeight=582&originWidth=1256&size=110748&status=done&width=1256)<br />注意此命令是在好的store上执行<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564653489333-def691f2-726e-475c-a4ca-8cd7ee40e793.png#align=left&display=inline&height=45&name=image.png&originHeight=45&originWidth=610&size=6467&status=done&width=610)<br />此时启动TiKV集群，执行region 31101，坏掉的已经删掉了，但是服务还是起不来。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564653530458-02c011c8-20fd-4a2f-a05f-8c93515c609f.png#align=left&display=inline&height=495&name=image.png&originHeight=495&originWidth=523&size=27235&status=done&width=523)<br />此时执行 ![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564653616113-ab0e4831-4813-4e58-b7d5-ee49b9e86b87.png#align=left&display=inline&height=78&name=image.png&originHeight=78&originWidth=1035&size=12329&status=done&width=1035)

此时在 [@戚铮](#) 老大的指导下，升级 tikv-ctl ,<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564653856567-a2abc9d6-4f8f-4cf6-bd5f-96fa3a2f6e0a.png#align=left&display=inline&height=695&name=image.png&originHeight=695&originWidth=1322&size=117796&status=done&width=1322)<br />结果202这台，一共三个region坏了，处理了俩，感觉遥遥无期，下了tikv-ctl 3.x后发现，就还有一个坏的。胜利在望。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564653924392-b0be58c8-e5f3-4d82-97f7-f5e9e21e6340.png#align=left&display=inline&height=83&name=image.png&originHeight=83&originWidth=682&size=9300&status=done&width=682)

重复上述操作后，此节点终于up了<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564653986410-df4091d6-436f-4f1c-ba65-0b86d4012df0.png#align=left&display=inline&height=369&name=image.png&originHeight=369&originWidth=488&size=23247&status=done&width=488)

218这个有6个坏的<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564654011573-d22aa7a1-cddd-4e14-9537-31dd02f76212.png#align=left&display=inline&height=154&name=image.png&originHeight=154&originWidth=749&size=20993&status=done&width=749)<br />`unsafe-recover remove-fail-stores`  一通后，终于起来服务了。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564654127217-6f3318f9-1f93-4cee-b64f-649125144a59.png#align=left&display=inline&height=783&name=image.png&originHeight=783&originWidth=697&size=76233&status=done&width=697)

> Round 4 服务已可以启动，总结一下
> 1. 先stop tikv
> 1. 如果坏的region少于一半，可以尝试 recover-mvcc
> 1. 如果超过一半，就玄乎了，是在不行就 unsafe-recover remove-fail-stores,然后再 tikv-ctl --db /path/to/tikv/db tombstone -p 127.0.0.1:2379 -r 31101,xx,xx,xx 
> 1. 再start tikv


<a name="58Wno"></a>
### Round 5 最终局
你以为万事大吉了？命运就是爱捉弄人。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564654457090-8b4b7bc1-c86b-46f9-bd8c-770334b1581a.png#align=left&display=inline&height=520&name=image.png&originHeight=520&originWidth=549&size=59936&status=done&width=549)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564654496584-e2f090a0-2837-4979-9d1e-ba278925441f.png#align=left&display=inline&height=637&name=image.png&originHeight=637&originWidth=1459&size=107875&status=done&width=1459)<br />回到原点。

最后还是损失了部分，但是量不大。

<a name="mYLTJ"></a>
## 总结

1. 对TiDB不够熟悉，很多流于表面
1. 对TiDB的文档和工具使用不熟练
1. TiDB的文档不太清晰，比如在故障处理里，没有内链像是pd-ctl,tikv-ctl，甚至都没有提，在pd-ctl和tikv-ctl等工具没有提如何下载，在工具下载里，没有提包含啥工具。很佛系
1. 多亏了群内各位大佬的热心指导
1. 如果是tikv有问题，先stop tikv
1. 如果对于损坏数小于半数的，可以尝试 recover-mvcc
1. 对于超过半数的，可以尝试 unsafe-recover remove-fail-stores ，再 将store设置 tombstone
1. 再start tikv
1. 可以结合 [tidb损坏tikv节点怎么恢复集群](https://www.cnblogs.com/vansky/p/9415551.html) 来做。
1. 多试验，尤其是做极限测试，并且尝试处理，会积累很多经验。
1. 虽然没有瑞恩没有全须全尾的拯救回来，但是缺胳膊少腿总好过没命啊。

<a name="Gf6U8"></a>
## 一点小广告
其实如果不考虑OLTP的场景，还可以尝试使用clickhouse。这是之前整理的clickhouse的一些文章。

- [031-数据可视化之redash(支持43种数据源)](https://anjia0532.github.io/2019/07/08/redash/)
- [033-史上最全-mysql迁移到clickhouse的5种办法](https://anjia0532.github.io/2019/07/17/mysql-to-clickhouse/)
- [035-解决streamsets jdbc全量模式数据重复问题](https://anjia0532.github.io/2019/07/22/sdc-jdbc-full-mode/)

<a name="7y7Px"></a>
## 参考资料

- [我的博客](http://anjia0532.github.io/2019/08/01/tidb-tikv-bad-regions)
- [我的掘金](https://juejin.im/post/5d42f942f265da03d60ee060)
- [failed to write to engine #10596](https://github.com/pingcap/tidb/issues/10596)
- [使用 TiDB-Ansible 扩容缩容 TiDB 集群](https://pingcap.com/docs-cn/v3.0/how-to/scale/with-ansible/)
- [TiDB 集群故障诊断](https://pingcap.com/docs-cn/v3.0/how-to/troubleshoot/cluster-setup/)
- [PD Control 使用说明](https://pingcap.com/docs-cn/v3.0/reference/tools/pd-control/)
- [TiKV Control 使用说明](https://pingcap.com/docs-cn/v3.0/reference/tools/tikv-control/)
- [TiDB 工具下载](https://pingcap.com/docs-cn/v3.0/reference/tools/download/)
- [tidb损坏tikv节点怎么恢复集群](https://www.cnblogs.com/vansky/p/9415551.html)
- [Tidb用户案例](https://pingcap.com/cases-cn/)

