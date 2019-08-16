
---

title: 025-大数据ETL工具之StreamSets安装及订阅mysql binlog

date: 2019-06-10 21:00:01 +0800

tags: [CDH,hadoop,大数据,ETL,StreamSets,SDC]

categories: 大数据

---
> 这是坚持技术写作计划（含翻译）的第25篇，定个小目标999，每周最少2篇。


本文主要介绍CDH6.2+StreamSets3.9。

StreamSets是一个大数据采集和数据处理工具。可以通过拖拽式的可视化操作，实现数据管道(Pipelines)的设计和调度。其特点有：

- 拖拽式的可视化界面操作，上手快。
- 对常见数据处理(数据源、数据操作、数据输出)支持较好。
- 内置监控，可以对数据流进行观测。

类似的开源产品还有 [Apache NiFi](http://nifi.apache.org/) , 网上有关于NiFi和StreamSets 的对比 [Open Source ETL: Apache NiFi vs Streamsets](https://statsbot.co/blog/open-source-etl/) (网上有中文翻译版版)

国内接触较多的ETL工具，可能是 [DataX](https://github.com/alibaba/DataX) 、 [Kettle](https://kettle.pentaho.com) 、[Sqoop](http://sqoop.apache.org/)。此处有个简单的对比，[数据集成之 kettle、sqoop、datax、streamSets 比较 ](https://my.oschina.net/peakfang/blog/2056426)


<a name="t85Ls"></a>
## 安装StreamSets 3.9
<a name="sMTmJ"></a>
### 下载parcel安装包

从 [https://archives.streamsets.com/index.html](https://archives.streamsets.com/index.html) 下载3.9的<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560930235577-b347118a-9af6-4e1e-b3c8-d76d7d388e95.png#align=left&display=inline&height=681&name=image.png&originHeight=681&originWidth=549&size=51810&status=done&width=549)<br />并上传到http服务器的www目录下，本文以centos7.6为例

```bash
wget -P /var/www/html/streamsets3.9.0/ https://archives.streamsets.com/datacollector/3.9.0/parcel/manifest.json
wget -P /var/www/html/streamsets3.9.0/ https://archives.streamsets.com/datacollector/3.9.0/parcel/STREAMSETS_DATACOLLECTOR-3.9.0-el7.parcel.sha
wget -P /var/www/html/streamsets3.9.0/ https://archives.streamsets.com/datacollector/3.9.0/parcel/STREAMSETS_DATACOLLECTOR-3.9.0-el7.parcel
```
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560930516379-4f6922d6-76ed-44cd-b5c7-1b8579743cdf.png#align=left&display=inline&height=269&name=image.png&originHeight=269&originWidth=553&size=25017&status=done&width=553)

<a name="QEaW1"></a>
### 配置csd
从 [https://streamsets.com/opensource](https://streamsets.com/opensource) 下载<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560930592344-14e9b2ee-6153-4c2b-8e6e-eafa305b11ca.png#align=left&display=inline&height=628&name=image.png&originHeight=628&originWidth=520&size=49861&status=done&width=520)

```bash
wget -P /opt/cloudera/csd/ https://archives.streamsets.com/datacollector/3.9.0/csd/STREAMSETS-3.9.0.jar
cd /opt/cloudera/csd/
sudo chown cloudera-scm:cloudera-scm STREAMSETS-3.9.0.jar && sudo chmod 644 STREAMSETS-3.9.0.jar
systemctl restart cloudera-scm-server
```

<a name="16S3C"></a>
### 下载分发Parcel包
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560931069258-56f0a704-4b7a-4765-8e19-6600d46f7f6f.png#align=left&display=inline&height=158&name=image.png&originHeight=158&originWidth=770&size=15171&status=done&width=770)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560931088736-68ef0972-74b2-4043-984c-b2b1630adca0.png#align=left&display=inline&height=98&name=image.png&originHeight=98&originWidth=394&size=5113&status=done&width=394)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560931127791-d2b173ce-ed55-4b65-99a7-2cd658468dc5.png#align=left&display=inline&height=346&name=image.png&originHeight=346&originWidth=1023&size=42564&status=done&width=1023)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560931177761-a0d04a4d-a502-4b7f-a84e-fb6323c0299e.png#align=left&display=inline&height=118&name=image.png&originHeight=118&originWidth=1029&size=14553&status=done&width=1029)<br />下载并激活，但是，我实际测试时，总大小，4.6G，实际下载后，5.2G，导致sha1sum 校验失败，报<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560931888700-b4afecab-a671-463b-9610-701e2a58b761.png#align=left&display=inline&height=227&name=image.png&originHeight=227&originWidth=628&size=18002&status=done&width=628)

在cm所在主机， `ls -lah /opt/cloudera/parcel-repo`  <br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560932061185-d64adfe2-5884-4967-b98b-c76a999c3024.png#align=left&display=inline&height=159&name=image.png&originHeight=159&originWidth=777&size=25904&status=done&width=777)

把下载的 [https://archives.streamsets.com/datacollector/3.9.0/parcel/STREAMSETS_DATACOLLECTOR-3.9.0-el7.parcel](https://archives.streamsets.com/datacollector/3.9.0/parcel/STREAMSETS_DATACOLLECTOR-3.9.0-el7.parcel) 复制到 /opt/cloudera/parcel-repo 下<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560932517429-eb7b8ece-4135-4a81-a751-8e8c0d267ef9.png#align=left&display=inline&height=105&name=image.png&originHeight=105&originWidth=1039&size=12588&status=done&width=1039)<br />如果已经不信邪，试过下载，并报hash错误后，直接替换后，这个页面还是提示hash，此时再次点击下载，就会变成分配。<br />激活后如下所示<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560936491543-f6e5539b-2396-4eb4-a616-ae2bafd02155.png#align=left&display=inline&height=156&name=image.png&originHeight=156&originWidth=1589&size=17337&status=done&width=1589)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560936642593-d20866d4-46d1-490e-9bff-8cf7c72587d5.png#align=left&display=inline&height=701&name=image.png&originHeight=701&originWidth=903&size=171790&status=done&width=903)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560936658975-a85eae7a-341d-4c68-8967-a670c23cb622.png#align=left&display=inline&height=378&name=image.png&originHeight=378&originWidth=1146&size=38226&status=done&width=1146)<br />创建完毕

<a name="ScFf4"></a>
### streamsets 简单使用
打开streamsets，默认用户名密码 admin/admin<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1560945744410-72401676-7384-4312-8859-c7b652c1caca.png#align=left&display=inline&height=516&name=image.png&originHeight=516&originWidth=1374&size=63461&status=done&width=1374)<br />
        ![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561003595012-472339dd-c7c0-49be-9be3-855d9fe21016.png)
          
  
  
          
      
    
  <br />

官方教程，参考 [Basic Tutorial](https://streamsets.com/documentation/datacollector/3.9.x/help/datacollector/UserGuide/Tutorial/BasicTutorial.html)

本文主要讲解订阅mysql binlog进行数据同步

<a name="fcYrG"></a>
## mysql binlog

<a name="nEogv"></a>
### 开启binlog
修改mysql配置文件，my.cnf，在mysqld下增加（注意5.7的不加server-id无法正常启动）

```
server-id=1
log-bin=mysql-bin
binlog_format=ROW
```

<a name="Q47aL"></a>
### 创建并配置同步账号

```sql
GRANT ALL on slave_test.* to 'slave_test'@'%' identified by 'slave_test';
GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE on *.* to 'slave_test'@'%';
FLUSH PRIVILEGES;
```

<a name="oWlz6"></a>
### 安装mysql jdbc驱动

```bash
wget -P /opt/cloudera/parcels/STREAMSETS_DATACOLLECTOR/streamsets-libs/streamsets-datacollector-mysql-binlog-lib/lib/ https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.47/mysql-connector-java-5.1.47.jar
```

重启streamsets


<a name="qXsmC"></a>
### 创建pipeline
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561022971274-512e79c5-11a2-4910-85e2-aabeba28edb2.png#align=left&display=inline&height=483&name=image.png&originHeight=483&originWidth=1245&size=57203&status=done&width=1245)
<a name="vDqzm"></a>
### 配置mysql binlog解析及处理
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561023067141-a97a2e75-8272-472e-8d31-812be2123206.png#align=left&display=inline&height=703&name=image.png&originHeight=703&originWidth=831&size=64114&status=done&width=831)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561023232872-f63802b3-7be6-4968-9fb9-2f7c368e7c3b.png#align=left&display=inline&height=649&name=image.png&originHeight=649&originWidth=781&size=44770&status=done&width=781)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561023290961-315616ad-faed-405c-af47-f1bed5816b07.png#align=left&display=inline&height=345&name=image.png&originHeight=345&originWidth=776&size=40429&status=done&width=776)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561023323874-819a18b1-42f9-404b-9eac-17d7b7190b3c.png#align=left&display=inline&height=211&name=image.png&originHeight=211&originWidth=658&size=15146&status=done&width=658)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561023405577-345c7f5d-e682-45ba-b684-450707e4d26d.png#align=left&display=inline&height=482&name=image.png&originHeight=482&originWidth=880&size=57665&status=done&width=880)<br />配置目标端<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561023517902-fcf0a0d8-7360-4ca9-aed4-2b2f88b07ce7.png#align=left&display=inline&height=428&name=image.png&originHeight=428&originWidth=303&size=22581&status=done&width=303)![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561023530221-53bf1e14-b792-40bf-9fcf-234b3c1ca097.png#align=left&display=inline&height=583&name=image.png&originHeight=583&originWidth=773&size=28852&status=done&width=773)


<a name="AEeSg"></a>
### 运行

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561023620269-fd10f7a4-f48d-44d7-a61f-d03a5fc8dead.png#align=left&display=inline&height=458&name=image.png&originHeight=458&originWidth=1808&size=69037&status=done&width=1808)

<a name="WoQS7"></a>
### 测试
此处使用mysql自带的压测工具 `mysqlslap.exe` 进行测试 

```bash
bin/mysqlslap --user=root --password=xxxxxx --concurrency=50 --number-int-cols=5 --number-char-cols=20 --auto-generate-sql --number-of-queries=100000 --auto-generate-sql-load-type=write --host=192.168.0.123 --port=3306
--user 用户(需要有建库建表权限)
--password 密码
--concurrency 并发数
--number-int-cols 表内有5个数字列
--number-char-cols 表内有20个字符串列
--auto-generate-sql 自动生成脚本
--number-of-queries 总执行次数
--auto-generate-sql-load-type=write 只执行写入操作
--host mysql 主机
--port 端口
```
下方有监控报表

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1561022869352-3d191209-41df-43ef-a7a6-5b7f097f4ba0.png#align=left&display=inline&height=883&originHeight=883&originWidth=1896&status=done&width=1896)


<a name="41HXC"></a>
## 常见错误

        ![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561021775509-fa60a34d-8e71-4e30-aa65-88a23521fb26.png)
          
  
  
          
      
    
  <br />同步不一致导致的错误，手动从<br />![](https://cdn.nlark.com/yuque/0/2019/png/226273/1561023290961-315616ad-faed-405c-af47-f1bed5816b07.png#align=left&display=inline&height=332&originHeight=345&originWidth=776&status=done&width=746)<br />设置偏移量<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1561022441978-aefea073-d2eb-41b6-863c-733229e35252.png#align=left&display=inline&height=709&name=image.png&originHeight=709&originWidth=900&size=124020&status=done&width=900)

如果报错 `Pipeline Status: RUNNING_ERROR: For input string: ""xxxx"`   ，把my.cnf改成

```bash
server-id=1
log-bin=mysql-bin
binlog_format=ROW
sync_binlog=1
binlog_gtid_simple_recovery=ON
log_slave_updates=ON
gtid_mode=ON
enforce_gtid_consistency=ON
```

<a name="ePxB0"></a>
## 参考资料

- [腾讯工程师带你深入解析 MySQL binlog](https://zhuanlan.zhihu.com/p/33504555)
- [Home/Origins/MySQL Binary Log](https://streamsets.com/documentation/datacollector/latest/help/datacollector/UserGuide/Origins/MySQLBinaryLog.html)
- [Home/Tutorial/Basic Tutorial](https://streamsets.com/documentation/datacollector/3.9.x/help/datacollector/UserGuide/Tutorial/BasicTutorial.html)
- [如何在CDH中安装和使用StreamSets](https://mp.weixin.qq.com/s?__biz=MzI4OTY3MTUyNg==∣=2247488566&idx=1&sn=8c4350eb654453b2317de2b347b6e525&chksm=ec2ac43fdb5d4d2964e2b074d02faaeebe202c2ba3565ac00f589b696644bdbfa8f6f4d9754e&scene=21#wechat_redirect)
- [如何使用StreamSets实现MySQL中变化数据实时写入HBase](https://cloud.tencent.com/developer/article/1158163)

