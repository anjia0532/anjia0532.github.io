---
title: 033-史上最全-mysql迁移到clickhouse的5种办法
urlname: mysql-to-clickhouse
date: '2019-07-17 22:15:38 +0800'
tags:
  - 数据分析
  - 数据处理
  - mysql
  - 数据库
  - clickhouse
categories: 大数据
---

> 这是坚持技术写作计划（含翻译）的第 33 篇，定个小目标 999，每周最少 2 篇。

数据迁移需要从 mysql 导入 clickhouse, 总结方案如下，包括 clickhouse 自身支持的三种方式，第三方工具两种。

<!-- more -->

## create table engin mysql

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
    ...
    INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
    INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
) ENGINE = MySQL('host:port', 'database', 'table', 'user', 'password'[, replace_query, 'on_duplicate_clause']);
```

> 官方文档: [https://clickhouse.yandex/docs/en/operations/table_engines/mysql/](https://clickhouse.yandex/docs/en/operations/table_engines/mysql/)

注意，实际数据存储在远端 mysql 数据库中，可以理解成外表。
可以通过在 mysql 增删数据进行验证。

## insert into select from

```sql
-- 先建表
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = engine
-- 导入数据
INSERT INTO [db.]table [(c1, c2, c3)] select 列或者* from mysql('host:port', 'db', 'table_name', 'user', 'password')
```

可以自定义列类型，列数，使用 clickhouse 函数对数据进行处理，比如 `select toDate(xx) from mysql("host:port","db","table_name","user_name","password")`

## create table as select from

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name
ENGINE =Log
AS
SELECT *
FROM mysql('host:port', 'db', 'article_clientuser_sum', 'user', 'password')
```

> 网友文章: [http://jackpgao.github.io/2018/02/04/ClickHouse-Use-MySQL-Data/](http://jackpgao.github.io/2018/02/04/ClickHouse-Use-MySQL-Data/)

不支持自定义列，参考资料里的博主写的 `ENGIN=MergeTree`  测试失败。
可以理解成 create table 和 insert into select 的组合

## Altinity/clickhouse-mysql-data-reader

Altinity 公司开源的一个 python 工具，用来从 mysql 迁移数据到 clickhouse(支持 binlog 增量更新和全量导入)，但是官方 readme 和代码脱节，根据 quick start 跑不通。

```bash
## 创建表
clickhouse-mysql \
    --src-host=127.0.0.1 \
    --src-user=reader \
    --src-password=Qwerty1# \
    --table-templates-with-create-database \
    --src-table=airline.ontime > create_clickhouse_table_template.sql
## 修改脚本
vim create_clickhouse_table_template.sql

## 导入建表
clickhouse-client -mn < create_clickhouse_table_template.sql

## 数据导入
clickhouse-mysql \
     --src-host=127.0.0.1 \
     --src-user=reader \
     --src-password=Qwerty1# \
     --table-migrate \
     --dst-host=127.0.0.1 \
     --dst-table=logunified \
     --csvpool
```

> 官方文档: [https://github.com/Altinity/clickhouse-mysql-data-reader#mysql-migration-case-1---migrate-existing-data](https://github.com/Altinity/clickhouse-mysql-data-reader#mysql-migration-case-1---migrate-existing-data)

注意，上述三种都是从 mysql 导入 clickhouse，如果数据量大，对于 mysql 压力还是挺大的。下面介绍两种离线方式(streamsets 支持实时，也支持离线)
csv

```bash
## 忽略建表
clickhouse-client \
  -h host \
  --query="INSERT INTO [db].table FORMAT CSV" < test.csv
```

但是如果源数据质量不高，往往会有问题，比如包含特殊字符(分隔符，转义符)，或者换行。被坑的很惨。

- 自定义分隔符, `--format_csv_delimiter="|"`
- 遇到错误跳过而不中止， `--input_format_allow_errors_num=10`  最多允许 10 行错误, `--input_format_allow_errors_ratio=0.1`  允许 10%的错误
- csv 跳过空值(null) ，报 `Code: 27. DB::Exception: Cannot parse input: expected , before: xxxx: (at row 69) ERROR: garbage after Nullable(Date): "8,002<LINE FEED>0205"`  `sed ' :a;s/,,/,\\N,/g;ta' |clickhouse-client -h host --query "INSERT INTO [db].table FORMAT CSV"`  将 `,,`  替换成 `,\N,`

`python clean_csv.py --src=src.csv --dest=dest.csv --chunksize=50000 --cols --encoding=utf-8 --delimiter=,`

clean_csv.py 参考我另外一篇  [032-csv 文件容错处理](https://anjia0532.github.io/2019/07/16/clean-csv/)

## streamsets

streamsets 支持从 mysql 或者读 csv 全量导入，也支持订阅 binlog 增量插入，参考我另外一篇 [025-大数据 ETL 工具之 StreamSets 安装及订阅 mysql binlog](https://anjia0532.github.io/2019/06/10/cdh-streamsets/)。
本文只展示从 mysql 全量导入 clickhouse
本文假设你已经搭建起 streamsets 服务
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563437517163-af02c2db-f03b-4884-8f16-4850918ddc0d.png#align=left&display=inline&height=850&originHeight=850&originWidth=1911&size=97265&status=done&width=1911)
启用并重启服务
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563437777046-970dff85-960f-48f3-a3e9-0a10002f34b4.png#align=left&display=inline&height=860&originHeight=860&originWidth=1843&size=89828&status=done&width=1843)
上传 mysql 和 clickhouse 的 jdbc jar 和依赖包
便捷方式，创建 pom.xml，使用 maven 统一下载

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.anjia</groupId>
  <artifactId>demo</artifactId>
  <packaging>jar</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>demo</name>
  <url>http://maven.apache.org</url>
  <dependencies>
    <dependency>
        <groupId>ru.yandex.clickhouse</groupId>
        <artifactId>clickhouse-jdbc</artifactId>
        <version>0.1.54</version>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.47</version>
  </dependency>
  </dependencies>
</project>
```

如果本地装有 maven，执行如下命令
`mvn dependency:copy-dependencies -DoutputDirectory=lib -DincludeScope=compile` 
所有需要的 jar 会下载并复制到 lib 目录下
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563438052063-1f073ee5-1c50-4842-8f9e-5f9974867895.png#align=left&display=inline&height=298&originHeight=298&originWidth=391&size=34034&status=done&width=391)
然后拷贝到 streamsets `/opt/streamsets-datacollector-3.9.1/streamsets-libs-extras/streamsets-datacollector-jdbc-lib/lib/`  目录下
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563442430751-7889c386-014a-417e-bc0c-069414f08b89.png#align=left&display=inline&height=740&originHeight=740&originWidth=751&size=59852&status=done&width=751)
重启 streamsets 服务
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563443529858-b23e198f-a097-4359-ad0b-85768a5c68ec.png#align=left&display=inline&height=320&originHeight=320&originWidth=194&size=13854&status=done&width=194)![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563443568159-590b99dc-af61-45d2-8724-b33d08ba7df2.png#align=left&display=inline&height=235&originHeight=235&originWidth=189&size=9943&status=done&width=189)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563444143877-060e4c52-53af-42f3-99e2-38cfc2add983.png#align=left&display=inline&height=726&originHeight=726&originWidth=1253&size=83069&status=done&width=1253)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563444315768-4295a91d-b610-4df8-8796-196358662c68.png#align=left&display=inline&height=776&originHeight=776&originWidth=1378&size=80345&status=done&width=1378)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563444358478-d8bd7c56-c0c0-49a6-903e-1d284d812e5d.png#align=left&display=inline&height=773&originHeight=773&originWidth=1344&size=81650&status=done&width=1344)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1563444395375-690becdc-1f48-4cfb-b6bd-13d03554cbc1.png#align=left&display=inline&height=785&originHeight=785&originWidth=1629&size=69971&status=done&width=1629)

## 参考资料

- [我的博客](https://anjia0532.github.io/2019/07/17/mysql-to-clickhouse)
- [我的掘金](https://juejin.im/post/5d30454ce51d4510bf1d6737)
- [Building data stream pipelines with CrateDB and StreamSets data collector](https://crate.io/docs/crate/guide/en/latest/tools/streamsets.html)
- [JDBC Query Consumer](https://streamsets.com/documentation/datacollector/latest/help/datacollector/UserGuide/Origins/JDBCConsumer.html)
- [Data Flow Pipeline Using StreamSets](https://dzone.com/articles/data-flow-pipeline-using-streamsets)
