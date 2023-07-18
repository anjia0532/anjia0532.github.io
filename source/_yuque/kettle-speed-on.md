---
title: 026-Kettle表输入表输出提升50倍的秘诀
urlname: kettle-speed-on
date: '2019-06-12 12:30:53 +0800'
tags:
  - Kettle
  - ETL
  - PDI
  - pentaho
categories: 大数据
---

> 这是坚持技术写作计划（含翻译）的第 26 篇，定个小目标 999，每周最少 2 篇。

最近工作需要，需要从 Oracle 导数据到 Mysql，并且需要进行适当的清洗，转换。
数据量在 5 亿条左右，硬件环境为 Winserver 2008R2 64 位 ，64G，48 核，1T hdd，kettle 是 8.2，从 Oracle（11G,linux 服务器，局域网连接）抽到 mysql(5.7,本机，win server)。
优化前的速度是读 1000r/s(Oracle)左右，写 1000r/s 左右。
优化后的速度是读 8Wr/s(Oracle)左右，写 4Wr/s 左右。
因为表的字段大小和类型以及是否有索引都有关系，所以总体来说，提升了 20-50 倍左右。

<!-- more -->

## mysql 优化

mysql 此处只是为了迁移数据用，实际上用 csv，或者 clickhouse 也行。但是担心 csv 在处理日期时可能有问题，而 clickhouse 不能在 win 下跑，而条件所限，没有多余的 linux 资源，而 mysql 第三方开源框架(不管是导入 hdfs)，还是作为 clickhouse 的外表，还是数据展示(supserset,metabase 等)，还是迁移到 tidb，都很方便。所以最终决定用 mysql。

> Note:此处的 mysql 只做临时数据迁移用，所以可以随便重启跟修改 mysqld 参数。如果是跟业务混用时，需要咨询 dba，确保不会影响其他业务。

### mysql 安装及配置优化

- 从  [https://dev.mysql.com/downloads/mysql/5.7.html#downloads](https://dev.mysql.com/downloads/mysql/5.7.html#downloads)  下载 64 位 zip mysql
- 解压  mysql-5.7.26-winx64.zip 到目录，比如 D:\mysql-5.7.26-winx64
- 创建 D:\mysql-5.7.26-winx64\my.ini

```
[mysqld]
port=3306
basedir=D:\mysql-5.7.26-winx64\
datadir=D:\mysql-5.7.26-winx64\data
net_buffer_length=5242880
max_allowed_packet=104857600
bulk_insert_buffer_size=104857600
max_connections = 1000
innodb_flush_log_at_trx_commit = 2
# 本场景下测试MyISAM比InnoDB 提升1倍左右
default-storage-engine=MyISAM
general_log = 1
general_log_file=D:\mysql-5.7.26-winx64\logs\mysql.log
innodb_buffer_pool_size = 36G
innodb_log_files_in_group=2
innodb_log_file_size = 500M
innodb_log_buffer_size = 50M
sync_binlog=1
innodb_lock_wait_timeout = 50
innodb_thread_concurrency = 16
key_buffer_size=82M
read_buffer_size=64K
read_rnd_buffer_size=256K
sort_buffer_size=256K
myisam_max_sort_file_size=100G
myisam_sort_buffer_size=100M
transaction_isolation=READ-COMMITTED
```

- 参考  [2.3.5 Installing MySQL on Microsoft Windows Using a noinstall ZIP Archive](https://dev.mysql.com/doc/refman/5.7/en/windows-install-archive.html)  进行安装
- 启动 mysql 服务

## Kettle 优化

### 启动参数优化

本机内存较大，为了防止 OOM，所以调大内存参数，创建环境变量 `PENTAHO_DI_JAVA_OPTIONS` = `-Xms20480m -Xmx30720m -XX:MaxPermSize=1024m`  起始 20G，最大 30G

### 表输入和表输出开启多线程

表输入如果开启多线程的话，会导致数据重复。比如 `select * from test` ,起 3 个线程，就会查 3 遍，最后的数据就是 3 份。肯定不行，没达到优化的目的。
因为 source 是 oracle，利用 oracle 的特性： `rownum`  和函数： `mod` ,以及 kettle 的参数: `Internal.Step.Unique.Count,Internal.Step.Unique.Number`

```sql
select * from (SELECT test.*,rownum rn FROM test ) where mod(rn,${Internal.Step.Unique.Count}) = ${Internal.Step.Unique.Number}
```

解释一下

- rownum 是 oracle 系统顺序分配为从查询返回的行的编号,返回的第一行分配的是 1,第二行是 2，意味着，如果排序字段或者数据有变化的话，rownum 也会变（也就是跟物理数据没有对应关系,如果要对应关系的话，应该用 rowid,但是 rowid 不是数字，而是类似 AAAR3sAAEAAAACXAAA  的编码），所以需要对 rownum 进行固化，所以将 `SELECT test.*,rownum rn FROM test`  作为子查询
- mod 是 oracle 的取模运算函数，比如， `mod(5,3)`  意即 `5%3=2` ，就是 `5/3=1...2`  中的 2,也就是如果能获取到总线程数，以及当前线程数，取模，就可以对结果集进行拆分了。 `mod(行号,总线程数)=当前线程序号`
- kettle 内置函数 `${Internal.Step.Unique.Count}`  和 `${Internal.Step.Unique.Number}`  分别代表线程总数和当前线程序号

而表输出就无所谓了，开多少线程，kettle 都会求总数然后平摊的。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560314746879-dda9ca2b-4796-4a40-bef3-445afc7bfa39.png#align=left&display=inline&height=200&originHeight=200&originWidth=383&size=15147&status=done&width=383)
右键选择表输入或者表输出，选择 `改变开始复制的数量...`  注意，不是一味的调大就一定能提升效率，要进行测试的。
表输入时，注意勾选替换变量
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560314826857-199c854f-e240-4094-b112-c0b0c9a8bc8a.png#align=left&display=inline&height=261&originHeight=261&originWidth=744&size=12391&status=done&width=744)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560314887846-35838541-cab9-484c-859a-968d3566241c.png#align=left&display=inline&height=732&originHeight=732&originWidth=505&size=28983&status=done&width=505)

- 修改提交数量(默认 100，但是不是越大越好)
- 去掉裁剪表，因为是多线程，你肯定不希望，A 线程刚插入的，B 给删掉。
- 必须要指定数据库字段，因为表输入的时候，会多一个行号字段。会导致插入失败。当然如果你在创建表时，多加了行号字段，当做自增 id 的话，那就不需要这一步了。
- 开启批量插入

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560315449676-93873a23-b85d-4446-98be-589792105ec1.png#align=left&display=inline&height=538&originHeight=538&originWidth=529&size=33262&status=done&width=529)

> Note: 通过开启多线程，速度能提升 5 倍以上。

### 开启线程池及优化 jdbc 参数

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560315569302-4a129a71-4921-4c32-96c8-80a423811f46.png#align=left&display=inline&height=338&originHeight=338&originWidth=440&size=17388&status=done&width=440)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560315593202-4f39a1a4-6340-4dc1-8915-70428c1c1a34.png#align=left&display=inline&height=408&originHeight=408&originWidth=1005&size=19781&status=done&width=1005)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560315610579-7ce37a68-6bc9-4041-b67d-a3d5c6162291.png#align=left&display=inline&height=253&originHeight=253&originWidth=1015&size=14437&status=done&width=1015)

## 运行观察结果

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560315664223-5d94f858-ef86-42a3-ae31-86b83d189490.png#align=left&display=inline&height=399&originHeight=399&originWidth=708&size=34982&status=done&width=708)
注意调整不同的参数(线程数，提交数),观察速度。

## 其余提升空间

1. 换 ssd
2. 继续优化 mysql 参数
3. 换引擎，比如，tokudb
4. 换抽取工具，比如 streamsets,datax
5. 换数据库，比如 clickhouse,tidb,Cassandra
6. kettle 集群

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
