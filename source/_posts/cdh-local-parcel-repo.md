
---

title: 018-CDH6.2构建本地源加速CDH安装

date: 2019-04-28 19:10:00 +0800

tags: [CDH,hadoop,大数据]

categories: 大数据

---
> 这是坚持技术写作计划（含翻译）的第18篇，定个小目标999，每周最少2篇。


目前国内还没有机构或者个人提供CDH的公共加速源，导致CDH安装时超慢，并且一旦失败后，还得不支持断点安装(linux机制)，配置CDH本地repo是学习cdh的第一步，否则单是安装就需要以小时为单位。

<!-- more -->

本文以centos7.6为例（其余发行版类似），介绍CDH 自定义 parcel和package 镜像源（parcel是cdh自定义格式）

<a name="22xMQ"></a>
## 创建内网repo
<a name="4pKfU"></a>
### 配置web服务器
可以用apache2，也可以用nginx，任何提供http服务的都可以
```bash
$ sudo apt-get install -y httpd
$ sudo systemctl start httpd
$ sudo systemctl enable httpd
```

<a name="D9i4s"></a>
### 下载packages
这是给centos安装cm6用的

```bash
$ sudo mkdir -p /var/www/html/cloudera-repos
$ sudo wget --recursive --no-parent --no-host-directories https://archive.cloudera.com/cm6/6.2.0/redhat7/ -P /var/www/html/cloudera-repos
$ sudo wget https://archive.cloudera.com/cm6/6.2.0/allkeys.asc -P /var/www/html/cloudera-repos/cm6/6.2.0/
$ sudo chmod -R ugo+rX /var/www/html/cloudera-repos/cm6
```

<a name="POHSQ"></a>
### 下载和发布parcel repo

1. 下载 `manifest.json` 和parcel 文件<br />**CDH6** 

CDH 6 parcel中包含 Apache Impala, Apache Kudu, Apache Spark 2, and Cloudera Search等组件，以6.2.0为例，在web服务器上运行下面指令，用来下载最新版的cdh 6.2，如果要换成cdh6.x的其他版本，只需要替换命令中的 `6.2.0` 即可。更多6.x版本信息参见 [CDH 6 Download Information](https://www.cloudera.com/documentation/enterprise/latest/topics/rg_cdh_6_download.html#cdh_download_info) 。
```bash
$ sudo mkdir -p /var/www/html/cloudera-repos
$ sudo wget --recursive --no-parent --no-host-directories https://archive.cloudera.com/cdh6/6.2.0/parcels/ -P /var/www/html/cloudera-repos
$ sudo wget --recursive --no-parent --no-host-directories https://archive.cloudera.com/gplextras6/6.2.0/parcels/ -P /var/www/html/cloudera-repos
$ sudo chmod -R ugo+rX /var/www/html/cloudera-repos/cdh6
$ sudo chmod -R ugo+rX /var/www/html/cloudera-repos/gplextras6
```

     **CDH5** <br />CDH 5 parcel中包含 Impala, Kudu, Spark 1, and Search 等组件，以5.14.4为例，在web服务器上运行以下指令，如果要换成cdh5.x的其他版本,需要替换命令中的 `5.14.4` 为指定版本号，更多5.x版本信息参见 [CDH Download Information](https://www.cloudera.com/documentation/enterprise/release-notes/topics/cdh_vd_cdh_download.html)
```bash
$ sudo mkdir -p /var/www/html/cloudera-repos
$ sudo wget --recursive --no-parent --no-host-directories https://archive.cloudera.com/cdh5/parcels/5.14.4/ -P /var/www/html/cloudera-repos
$ sudo wget --recursive --no-parent --no-host-directories https://archive.cloudera.com/gplextras5/parcels/5.14.4/ -P /var/www/html/cloudera-repos
$ sudo chmod -R ugo+rX /var/www/html/cloudera-repos/cdh5
$ sudo chmod -R ugo+rX /var/www/html/cloudera-repos/gplextras5
```

如果像本文实例一样，只需支持单一版本（centos7.6）cdh即可，为了节省时间，可以只下载具体版本。<br />以CDH6的为例，增加 `--accept-regex "el7|manifest"` ,代表只下载包含xenial和maifest的文件
```bash
# 官方命令
sudo wget --recursive --no-parent --no-host-directories https://archive.cloudera.com/cdh6/6.2.0/parcels/ -P /var/www/html/cloudera-repos
# 改后命令
sudo wget --recursive --no-parent --accept-regex "el7|manifest" --no-host-directories https://archive.cloudera.com/cdh6/6.2.0/parcels/ -P /var/www/html/cloudera-repos
```

如果想再快点，可以使用迅雷，axel，aria2等多线程工具快速下载后，上传到web服务器。<br />      **Apache Accumulo for CDH** <br />以下载Accumulo1.7.2为例,如果换成别的版本，替换命令中1.7.2即可

```bash
$ sudo mkdir -p /var/www/html/cloudera-repos
$ sudo wget --recursive --no-parent --no-host-directories https://archive.cloudera.com/accumulo-c5/parcels/1.7.2/ -P /var/www/html/cloudera-repos
$ sudo chmod -R ugo+rX /var/www/html/cloudera-repos/accumulo-c5
```

     **CDS Powered By Apache Spark 2 for CDH** <br />以下载CDS2.3.0.cloudera3为例,更多版本信息参见  [CDS Powered By Apache Spark Version Information](https://www.cloudera.com/documentation/spark2/latest/topics/spark2_packaging.html#versions)
```bash
$ sudo mkdir -p /var/www/html/cloudera-repos
$ sudo wget --recursive --no-parent --no-host-directories https://archive.cloudera.com/spark2/parcels/2.3.0.cloudera3/ -P /var/www/html/cloudera-repos
$ sudo chmod -R ugo+rX /var/www/html/cloudera-repos/spark2
```

     **Cloudera Navigator Key Trustee Server** <br /> Key Trustee KMS parcel中包含  Cloudera Navigator HSM KMS ，从 [download page](http://www.cloudera.com/content/www/en-us/downloads/navigator/key-trustee-kms.html) 下载Key Trustee KMS，选择指定Version，比如 `Navigator Key Trustee KMS 6.2.0` ,选择Package or Parcel,选择 `Parcel` ,选择 `DOWNLOAD NOW` ,将下载Key Trustee KMS parcels 和 manifest.json ，将下载的 `.tar.gz` 上传到web服务器上，并解压，以Key Trustee KMS 6.2.0为例

```bash
$ sudo mkdir -p /var/www/html/cloudera-repos/keytrustee-kms
$ sudo tar xvfz /path/to/keytrustee-kms-6.2.0-parcels.tar.gz -C /var/www/html/cloudera-repos/keytrustee-kms --strip-components=1
$ sudo chmod -R ugo+rX /var/www/html/cloudera-repos/keytrustee-kms
```

     **Sqoop Connectors** <br />以下载最新版Sqoop为例
```bash
$ sudo mkdir -p /var/www/html/cloudera-repos
$ sudo wget --recursive --no-parent --no-host-directories http://archive.cloudera.com/sqoop-connectors/parcels/latest/ -P /var/www/html/cloudera-repos
$ sudo chmod -R ugo+rX /var/www/html/cloudera-repos/sqoop-connectors
```


2. 访问repo地址 `http://<Web_server>/cloudera-repos/` 确保你下载的文件能够正常访问。

<a name="IlmKd"></a>
### 配置Cloudera Manager 使用Parcel repo

1. 两种方法二选一，配置parcel
  1. Navigation bar - 导航条
    1. 点击navigation bar 的parcel图标或者点击 `Hosts` 然后点击 `Parcels` 标签
    1. 点击 `Configuration` 按钮
  2. Menu - 菜单
    1. 选择 `Administration` (管理) -> `Settings` (设置)
    1. 选择 `Category`  > `Parcels` 
2. 在 `Remote Pacel Respository URLs` 点击添加按钮，并添加。
2. 填上parcel地址，比如  `http://<web_server>/cloudera-parcels/cdh6/6.2.0/` 
2. 填写 `Reason for change`  变更原因,点击 `Save Changes` 提交保存。

<a name="4qw5J"></a>
## 国内镜像源
本文写完后，发现中科大有一个CDH的反代，速度还挺快，可以按需使用。参考 [ustclug/mirrorrequest#56](https://github.com/ustclug/mirrorrequest/issues/56) ，经测试，特别不稳定，持续两天，访问不通。

<a name="9OziU"></a>
## 参考资料

- [Configuring a Local Package Repository](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_create_local_package_repo.html?cdoc-os=ubuntu+1604+xenial#internal_package_repo)
- [Configuring a Local Parcel Repository](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_create_local_parcel_repo.html)

