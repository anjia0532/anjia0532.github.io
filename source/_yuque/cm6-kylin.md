---
title: 021-cdh6.2+kylin2.6.2
urlname: cm6-kylin
date: 2019-05-03 20:00:01 +0800
tags: [CDH,hadoop,大数据,KYLIN,OLAP,麒麟]
categories: [大,数,据]
---

> 这是坚持技术写作计划（含翻译）的第 21 篇，定个小目标 999，每周最少 2 篇。

本文主要介绍，如何使用大数据神兽 Kylin(2.6.2)连接 cdh6.2。

<!-- more -->

## 提示

- 因为 cdh6.2 使用的是 hadoop3，而目前的 kylin3.0beta 版本只是 hadoop2,所以只能安装 kylin2.5+,此处选择 kylin2.6.2-cdh60（cdh6.0 版）

## 安装 kylin

### 下载 kylin2.6.2 二进制包

```bash
wget http://mirrors.tuna.tsinghua.edu.cn/apache/kylin/apache-kylin-2.6.2/apache-kylin-2.6.2-bin-cdh60.tar.gz
tar zxf apache-kylin-2.6.2-bin-cdh60.tar.gz -C /usr/local/
ln -s /usr/local/apache-kylin-2.6.2-bin-cdh60 /usr/local/kylin
```

### 配置 kylin 环境变量

```bash
cat << EOF | sudo tee -a /etc/profile
#设置java环境
export JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera/
export CLASSPATH=.:\$JAVA_HOME/lib:\$JAVA_HOME/jre/lib:\$CLASSPATH
export KYLIN_HOME=/usr/local/kylin
export PATH=\$JAVA_HOME/bin:\$JAVA_HOME/jre/bin:\$PATH
export CDH_HOME=/opt/cloudera/parcels/CDH
export HBASE_HOME=\${CDH_HOME}/lib/hbase
export HBASE_CLASSPATH=\${HBASE_HOME}/lib/hbase-common-2.1.0-cdh6.2.0.jar
EOF
source /etc/profile
```

如果不加 `$HBASE_HOME`  会报 `hbase-common lib not found`

```
Retrieving hadoop conf dir...
KYLIN_HOME is set to /usr/local/kylin
Retrieving hive dependency...
Retrieving hbase dependency...
Error: Could not find or load main class org.apache.hadoop.hbase.util.GetJavaProperty
hbase-common lib not found
```

### 在 hdfs 创建 kylin 和 spark 目录

```bash
export HADOOP_USER_NAME=hdfs
```

否则会报

```bash
$KYLIN_HOME/bin/check-env.sh
Retrieving hadoop conf dir...
Error: Could not find or load main class org.apache.hadoop.hbase.util.GetJavaProperty
KYLIN_HOME is set to /usr/local/kylin
mkdir: Permission denied: user=root, access=WRITE, inode="/kylin":hdfs:supergroup:drwxr-xr-x
Failed to create hdfs:///kylin/spark-history. Please make sure the user has right to access hdfs:///kylin/spark-history
```

```bash
yum install -y net-tools
```

否则会报

```bash
$KYLIN_HOME/bin/check-env.sh
Retrieving hadoop conf dir...
Error: Could not find or load main class org.apache.hadoop.hbase.util.GetJavaProperty
KYLIN_HOME is set to /usr/local/kylin
/usr/local/kylin/bin/check-port-availability.sh: line 27: netstat: command not found
```

### 下载 spark

```bash
$KYLIN_HOME/bin/download-spark.sh
```

否则会报

```bash
$KYLIN_HOME/bin/kylin.sh start
Retrieving hadoop conf dir...
错误: 找不到或无法加载主类 org.apache.hadoop.hbase.util.GetJavaProperty
KYLIN_HOME is set to /usr/local/kylin
Retrieving hive dependency...
Retrieving hbase dependency...
错误: 找不到或无法加载主类 org.apache.hadoop.hbase.util.GetJavaProperty
Retrieving hadoop conf dir...
错误: 找不到或无法加载主类 org.apache.hadoop.hbase.util.GetJavaProperty
Retrieving kafka dependency...
Retrieving Spark dependency...
spark not found, set SPARK_HOME, or run bin/download-spark.sh
```

如果知己指定了不兼容的 spark 版本，可能会导致 404，参考  [Kylin web UI http 404 error](https://issues.apache.org/jira/browse/KYLIN-3872)

### 启动 kylin

```bash
$KYLIN_HOME/bin/kylin.sh start
```

如果成功会输出

```
A new Kylin instance is started by root. To stop it, run 'kylin.sh stop'
Check the log at /usr/local/kylin/logs/kylin.log
Web UI is at http://<hostname>:7070/kylin
```

浏览器打开  http://IP:7070/kylin ，用户名密码是 `ADMIN/KYLIN` 
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557482549401-ccd81caf-6b9f-4dad-8899-1262307ef09a.png#align=left&display=inline&height=445&name=image.png&originHeight=445&originWidth=822&size=19890&status=done&width=822)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557482608635-8a90d635-862f-4695-b3b7-93c634267a5c.png#align=left&display=inline&height=738&name=image.png&originHeight=738&originWidth=979&size=46329&status=done&width=979)

## 使用 kylin(以官方 demo 演示)

### 导入数据

```bash
$KYLIN_HOME/bin/sample.sh

Retrieving hadoop conf dir...
Error: Could not find or load main class org.apache.hadoop.hbase.util.GetJavaProperty
Loading sample data into HDFS tmp path: /tmp/kylin/sample_cube/data
Going to create sample tables in hive to database DEFAULT by cli
WARNING: Use "yarn jar" to launch YARN applications.
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.2.0-1.cdh6.2.0.p0.967373/jars/log4j-slf4j-impl-2.8.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.2.0-1.cdh6.2.0.p0.967373/jars/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]

Logging initialized using configuration in jar:file:/opt/cloudera/parcels/CDH-6.2.0-1.cdh6.2.0.p0.967373/jars/hive-common-2.1.1-cdh6.2.0.jar!/hive-log4j2.properties Async: false
OK
//....
Sample cube is created successfully in project 'learn_kylin'.

** Restart Kylin Server or click Web UI => System Tab => Reload Metadata to take effect **

```

### 重新加载元数据

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557483583704-c21d7c5b-00b8-4d77-a50f-dc8fd534fe86.png#align=left&display=inline&height=581&name=image.png&originHeight=581&originWidth=1641&size=124786&status=done&width=1641)
选择 learn_kylin

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557483605102-3911751e-a861-4754-9dca-3b0311657f55.png#align=left&display=inline&height=149&name=image.png&originHeight=149&originWidth=222&size=7898&status=done&width=222)

### 构建 Cube

选择 Model，选择 kylin_sales_model,选择 build
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557483714167-c3f0370c-7a1a-4f46-a532-8123b4d4b9f6.png#align=left&display=inline&height=642&name=image.png&originHeight=642&originWidth=1901&size=80161&status=done&width=1901)
此处选择起止日期。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557483733469-dae19d83-b08a-4d72-9a05-1de253703e9f.png#align=left&display=inline&height=365&name=image.png&originHeight=365&originWidth=1512&size=26543&status=done&width=1512)
如果没关闭 hdfs 权限校验，此处肯定会 build 失败。可以通过右侧 `>`  图标点击查看进度。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557486437790-8e1c97a3-aebb-4866-857a-8ef3f0848d0c.png#align=left&display=inline&height=427&name=image.png&originHeight=427&originWidth=1871&size=53965&status=done&width=1871)
build 成功后，回到 Insight 界面，此时已经成功构建出 5 张表了。

### 讲解 demo 表

Kylin 的示例是销售业务分析

- KYLIN_SALES 事实表，存有销售订单的详细信息(卖家，商品分类，订单金额，商品数量等)
- KYLIN_COUNTRY 维度表，存有国家信息(简写，名称等)
- KYLIN_CATEGORY_GROUPINGS 维度表，存有商品分类的详细介绍(分类名称等)
- KYLIN_CAL_DT 维度表，存有时间扩展信息(日期所在年始，月始，周始，年份，月份等)
- KYLIN_ACCOUNT 维度表，存有账户信息(账户 id，卖家等级，买家等级，国家等)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557486460421-1d88684e-825c-420d-9fdd-9e9717c28cb6.png#align=left&display=inline&height=576&name=image.png&originHeight=576&originWidth=1102&size=47232&status=done&width=1102)

### 运行查询语句

执行 `select count(1) from kylin_sales`  点击 submit，下方会显示执行结果，以及执行耗时(此处是 1.8 秒)。kylin 会缓存执行结果，再次执行发现变成了 0.18 秒
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1557486561989-c58f3430-7336-49dc-9b47-14c68fb8c6d6.png#align=left&display=inline&height=771&name=image.png&originHeight=771&originWidth=1870&size=80062&status=done&width=1870)
执行稍微复杂的 SQL 语句

```sql
select sum(KYLIN_SALES.PRICE)
as price_sum,KYLIN_CATEGORY_GROUPINGS.META_CATEG_NAME,KYLIN_CATEGORY_GROUPINGS.CATEG_LVL2_NAME
from KYLIN_SALES inner join KYLIN_CATEGORY_GROUPINGS
on KYLIN_SALES.LEAF_CATEG_ID = KYLIN_CATEGORY_GROUPINGS.LEAF_CATEG_ID and
KYLIN_SALES.LSTG_SITE_ID = KYLIN_CATEGORY_GROUPINGS.SITE_ID
group by KYLIN_CATEGORY_GROUPINGS.META_CATEG_NAME,KYLIN_CATEGORY_GROUPINGS.CATEG_LVL2_NAME
order by KYLIN_CATEGORY_GROUPINGS.META_CATEG_NAME asc,KYLIN_CATEGORY_GROUPINGS.CATEG_LVL2_NAME desc
```

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558172952282-191a44d2-d434-4aad-8eff-da8cb6f8fc76.png#align=left&display=inline&height=586&name=image.png&originHeight=586&originWidth=1456&size=63114&status=done&width=1456)
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1558172990808-29ba9fd6-dfab-4835-84df-c98c6fa80fb7.png#align=left&display=inline&height=638&name=image.png&originHeight=638&originWidth=1536&size=68562&status=done&width=1536)自带简单的可视化。

## 参考资料

- [如何在 CDH 中部署及使用 Kylin](https://mp.weixin.qq.com/s?__biz=MzI4OTY3MTUyNg==∣=2247489540&idx=1&sn=a9a2c9bbb065987cd8756635c146800d)
- [Kylin web UI http 404 error](https://issues.apache.org/jira/browse/KYLIN-3872)
- [Kylin 2.6.1 on Ambari 2.7.1.0 花式踩坑集锦](https://zhuanlan.zhihu.com/p/62187552)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
