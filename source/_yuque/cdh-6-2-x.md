---
title: 017-Centos7.6+CDH 6.2 安装和使用
urlname: cdh-6-2-x
date: 2019-04-25 19:10:00 +0800
tags: [大数据,hadoop,spark,cdh,hdp]
categories: [大,数,据]
---

> 这是坚持技术写作计划（含翻译）的第 17 篇，定个小目标 999，每周最少 2 篇。

本文主要介绍 hadoop 发行版 CDH 最新版(6.2)的安装。

<!-- more -->

## 准备

### 硬件配置

| IP            | HostName        | OS             | Cores | Mem | Disk | Rules         |
| ------------- | --------------- | -------------- | ----- | --- | ---- | ------------- |
| 192.168.20.24 | cdh01.anjia.com | centos7.6 mini | 4     | 16G | 100G | CM6,CDH agent |
| 192.168.20.25 | cdh02.anjia.com | centos7.6 mini | 4     | 16G | 100G | CDH agent     |
| 192.168.20.28 | cdh03.anjia.com | centos7.6 mini | 4     | 16G | 100G | CDH agent     |

### 软件包

本地搭建镜像源，加速构建。参考  [018-CDH6.2 构建本地源加速 CDH 安装](https://juejin.im/post/5cc57a01f265da036c57929b)

### 安装前准备

#### CM 存储空间规划

参考  [Storage Space Planning for Cloudera Manager](https://www.cloudera.com/documentation/enterprise/6/6.2/topics/cm_ig_reqs_space.html)

#### 配置网络名

```bash
# cdh01.anjia.com 192.168.20.24
sudo hostnamectl set-hostname cdh01.anjia.com
cat << EOF | sudo tee -a /etc/hosts
192.168.20.24  cdh01.anjia.com  cdh01
192.168.20.25  cdh02.anjia.com  cdh02
192.168.20.28  cdh03.anjia.com  cdh03
EOF
echo "HOSTNAME=cdh01.anjia.com" >> /etc/sysconfig/network

# cdh02.anjia.com 192.168.20.25
sudo hostnamectl set-hostname cdh02.anjia.com
cat << EOF | sudo tee -a /etc/hosts
192.168.20.24  cdh01.anjia.com  cdh01
192.168.20.25  cdh02.anjia.com  cdh02
192.168.20.28  cdh03.anjia.com  cdh03
EOF
echo "HOSTNAME=cdh02.anjia.com" >> /etc/sysconfig/network

# cdh03.anjia.com 192.168.20.28
sudo hostnamectl set-hostname cdh03.anjia.com
cat << EOF | sudo tee -a /etc/hosts
192.168.20.24  cdh01.anjia.com  cdh01
192.168.20.25  cdh02.anjia.com  cdh02
192.168.20.28  cdh03.anjia.com  cdh03
EOF
echo "HOSTNAME=cdh03.anjia.com" >> /etc/sysconfig/network
```

参考  [Configure Network Names](https://www.cloudera.com/documentation/enterprise/6/6.2/topics/configure_network_names.html)

#### 禁用防火墙

每台都执行

```bash
sudo iptables-save > ~/firewall.rules
sudo systemctl disable firewalld
sudo systemctl stop firewalld
```

参考  [Disabling the Firewall](https://www.cloudera.com/documentation/enterprise/6/6.2/topics/install_cdh_disable_iptables.html)

#### 开启 ntp 同步时钟

每台都执行

```bash
yum install -y ntp
sed -i "/^server/ d" /etc/ntp.conf
cat << EOF | sudo tee -a /etc/ntp.conf
server ntp1.aliyun.com
server ntp2.aliyun.com
server ntp3.aliyun.com
server ntp4.aliyun.com
EOF
sudo systemctl start ntpd
sudo systemctl enable ntpd
ntpdate -u ntp1.aliyun.com
hwclock --systohc
```

#### 修改 repo

每台都执行

```bash
sudo wget https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/cloudera-manager.repo -P /etc/yum.repos.d/
sed -i "s/https\:\/\/archive.cloudera.com/http\:\/\/192.168.20.24\/cloudera-repos/g"  /etc/yum.repos.d/cloudera-manager.repo
sudo rpm --import https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/RPM-GPG-KEY-cloudera
```

#### 优化虚拟内存需求率

每台都执行

```bash
echo 'vm.swappiness = 10' >> /etc/sysctl.conf
sysctl -p
```

#### 解决透明大页面问题

```bash
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

#### 安装 jdk

每台都执行

```bash
sudo yum install -y oracle-j2sdk1.8

cat << EOF | sudo tee -a /etc/profile
#设置java环境
export JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera/
export CLASSPATH=.:\$JAVA_HOME/lib:\$JAVA_HOME/jre/lib:\$CLASSPATH
export PATH=\$JAVA_HOME/bin:\$JAVA_HOME/jre/bin:\$PATH
EOF
source /etc/profile
```

### 安装 CM 和 CDH

#### 安装 CM

在 cdh01.anjia.com 执行

```bash
sudo yum install -y cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server
```

启用 Auto-TLS,在 CM6.2.0 人工不启用 auto-tls 会导致登陆不成功

```bash
sudo JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera /opt/cloudera/cm-agent/bin/certmanager --location /opt/cloudera/CMCA setup --configure-services
```

#### 安装数据库

本文选择 MariaDB
在 cdh01.anjia.com 执行

```bash
sudo yum install -y mariadb-server
sudo systemctl stop mariadb
cat << EOF | sudo tee -a /etc/my.cnf.d/cdh-db.cnf
[mysqld]
transaction-isolation = READ-COMMITTED
# Disabling symbolic-links is recommended to prevent assorted security risks;
# to do so, uncomment this line:
symbolic-links = 0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd

key_buffer = 16M
key_buffer_size = 32M
max_allowed_packet = 32M
thread_stack = 256K
thread_cache_size = 64
query_cache_limit = 8M
query_cache_size = 64M
query_cache_type = 1

max_connections = 550
#expire_logs_days = 10
#max_binlog_size = 100M

#log_bin should be on a disk with enough free space.
#Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your
#system and chown the specified folder to the mysql user.
log_bin=/var/lib/mysql/mysql_binary_log

#In later versions of MariaDB, if you enable the binary log and do not set
#a server_id, MariaDB will not start. The server_id must be unique within
#the replicating group.
server_id=1

binlog_format = mixed

read_buffer_size = 2M
read_rnd_buffer_size = 16M
sort_buffer_size = 8M
join_buffer_size = 8M

# InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit  = 2
innodb_log_buffer_size = 64M
innodb_buffer_pool_size = 4G
innodb_thread_concurrency = 8
innodb_flush_method = O_DIRECT
innodb_log_file_size = 512M
EOF


sudo systemctl enable mariadb
sudo systemctl start mariadb

sudo /usr/bin/mysql_secure_installation

[...]
Enter current password for root (enter for none):
OK, successfully used password, moving on...
[...]
Set root password? [Y/n] Y
New password:
Re-enter new password:
[...]
Remove anonymous users? [Y/n] Y
[...]
Disallow root login remotely? [Y/n] N
[...]
Remove test database and access to it [Y/n] Y
[...]
Reload privilege tables now? [Y/n] Y
[...]
All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!

```

#### 安装数据库驱动

每台都执行

```bash
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.46.tar.gz
tar zxvf mysql-connector-java-5.1.46.tar.gz
sudo mkdir -p /usr/share/java/
cd mysql-connector-java-5.1.46
sudo cp mysql-connector-java-5.1.46-bin.jar /usr/share/java/mysql-connector-java.jar
```

#### 创建数据库

在 cdh01.anjia.com 执行

```bash
mysql -u root -p<password>

CREATE DATABASE <database> DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON <database>.* TO '<user>'@'%' IDENTIFIED BY '<password>';
```

这一步 scm 是必须的，其余的数据库可以等后边真实使用时再创建。

|              Service               | Database  |  User  |
| :--------------------------------: | :-------: | :----: |
|      Cloudera Manager Server       |    scm    |  scm   |
|          Activity Monitor          |   amon    |  amon  |
|          Reports Manager           |   rman    |  rman  |
|                Hue                 |    hue    |  hue   |
|       Hive Metastore Server        | metastore |  hive  |
|           Sentry Server            |  sentry   | sentry |
|  Cloudera Navigator Audit Server   |    nav    |  nav   |
| Cloudera Navigator Metadata Server |   navms   | navms  |
|               Oozie                |   oozie   | oozie  |

#### 配置 CM 数据库

```bash
sudo /opt/cloudera/cm/schema/scm_prepare_database.sh [options] <databaseType> <databaseName> <databaseUser> <password>
# 输出
JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera
Verifying that we can write to /etc/cloudera-scm-server
Creating SCM configuration file in /etc/cloudera-scm-server
Executing:  /usr/java/jdk1.8.0_181-cloudera/bin/java -cp /usr/share/java/mysql-connector-java.jar:/usr/share/java/oracle-connector-java.jar:/usr/share/java/postgresql-connector-java.jar:/opt/cloudera/cm/schema/../lib/* com.cloudera.enterprise.dbutil.DbCommandExecutor /etc/cloudera-scm-server/db.properties com.cloudera.cmf.db.
[                          main] DbCommandExecutor              INFO  Successfully connected to database.
All done, your SCM database is configured correctly!
```

详细参数，参考  [Syntax for scm_prepare_database.sh](https://www.cloudera.com/documentation/enterprise/latest/topics/prepare_cm_database.html#scm_prepare_syntax)

#### 安装 CDH 和其他软件

```bash
sudo systemctl start cloudera-scm-server
# 启动后，监听server日志，大约1-3分钟左右
sudo tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log
# 直到日志出现，如下内容，即启动成功。
2019-05-08 13:12:26,523 INFO WebServerImpl:com.cloudera.server.cmf.WebServerImpl: Started Jetty server.
```

参考资料

- [CDH 6.2.x Packaging](https://www.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_cdh_62_packaging.html)
