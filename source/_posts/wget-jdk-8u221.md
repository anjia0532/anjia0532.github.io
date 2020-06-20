
---

title: 044-wget免登陆下载jdk 8u221

date: 2019-09-18 19:35:21 +0800

tags: [linux,java,jdk,jdk8,wget]

categories: java

---

> 这是坚持技术写作计划（含翻译）的第44篇，定个小目标999，每周最少2篇。


本文主要介绍如何使用wget免登陆下载可用的jdk 8u221，介绍6种方式

<!-- more -->

<a name="XqNyt"></a>
### 下载最新的jre8
如果只是安装jre即可，则可以使用(长期有效)

```bash
$ url=$(curl -s https://www.java.com/en/download/linux_manual.jsp | grep -E ".*x64.*javadl" | grep -v "RPM" | sed "s/.*href=\"//g;s/\".*//g" | head -n 1)
$ wget -c --content-disposition $url
$ old=$(ls -hat | grep jre | head -n1)
$ mv $old $(echo $old | awk -F"?" '{print $1}')
```

<a name="pUie1"></a>
### 下载jdk8u221
从oracle官方下载，但是不保证长期可用
```bash
$ wget -c --content-disposition "https://javadl.oracle.com/webapps/download/AutoDL?BundleId=239835_230deb18db3e4014bb8e3e8324f81b43"
$ old=$(ls -hat | grep jre | head -n1)
$ mv $old $(echo $old | awk -F"?" '{print $1}')
```

windows jdk-8u221-windows-x64.exe 地址

```bash
https://javadl.oracle.com/webapps/download/AutoDL?BundleId=239842_230deb18db3e4014bb8e3e8324f81b43
```

<a name="gRz5N"></a>
### 下载jdk8u131

长期有效，也是oracle官方下载链接(8u131以后的都404了)

```bash
$ wget -O jdk-8u131-linux-x64.tar.gz --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u131-b11/d54c1d3a095b4ff2b6607d096fa80163/jdk-8u131-linux-x64.tar.gz
```

<a name="5tYNj"></a>
### 下载第三方jdk
自行判断校验码，不保证有效性和安全性
```bash
$ jdk_name=$(curl -s http://enos.itcollege.ee/~jpoial/allalaadimised/jdk8/ | grep tar.gz | grep -v demo |sed "s/.*href=\"//g;s/\".*//g"|head -n 1)
$ wget -O "$jdk_name" "http://enos.itcollege.ee/~jpoial/allalaadimised/jdk8/$jdk_name"
```

<a name="XmBCl"></a>
### 下载第三方openjdk

长期有效，不保证安全性

[Download Java SE Standard Compliant Liberica JDK 8u222](https://bell-sw.com/pages/java-8u222/)

```bash
$ wget -O bellsoft-jdk8u222-linux-amd64.tar.gz "https://download.bell-sw.com/java/8u222/bellsoft-jdk8u222-linux-amd64.tar.gz"
```

<a name="VPFA2"></a>
### 从官方下载
方法长期有效，但是AuthParam有时效性,无法写成脚本，也可以安装openjdk

1. 注册并登陆oracle账号
1. 打开 [https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
1. ![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1568853475035-279da7c9-a353-4be0-8515-e640e3cdb248.png#align=left&display=inline&height=423&name=image.png&originHeight=423&originWidth=703&size=75666&status=done&width=703)
1. ![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1568853516316-68e23846-09d1-47fc-83b5-bd0d8d55ffe5.png#align=left&display=inline&height=227&name=image.png&originHeight=227&originWidth=902&size=32289&status=done&width=902)
1. 复制的地址类似 [https://download.oracle.com/otn/java/jdk/8u221-b11/230deb18db3e4014bb8e3e8324f81b43/jdk-8u221-linux-x64.tar.gz?AuthParam=xxxxx_xxxxxxxxxxxxxxxxxx](https://download.oracle.com/otn/java/jdk/8u221-b11/230deb18db3e4014bb8e3e8324f81b43/jdk-8u221-linux-x64.tar.gz?AuthParam=xxxxx_xxxxxxxxxxxxxxxxxx) 其中AuthParam参数是有时效性的
1. `wget -O ``jdk-8u221-linux-x64.tar.gz`` --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" "``https://download.oracle.com/otn/java/jdk/8u221-b11/230deb18db3e4014bb8e3e8324f81b43/jdk-8u221-linux-x64.tar.gz?AuthParam=xxxxx_xxxxxxxxxxxxxxxxxx"`

<a name="ZTBsf"></a>
### 题外话
在网上找wget免密码下载jdk时，发现了一个有意思的项目<br />[The catalog may also be accessed using command-line tools, or through a simple HTTP API.](https://lv.binarybabel.org/catalog) <br />虽然给出的java相关的因为OTN的原因，都挂了，但是别的还是有些能用的。挺方便的。

<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。<br />长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。

<a name="35808e79"></a>
## 参考资料

- [我的博客](https://anjia0532.github.io/2019/09/18/wget-jdk-8u221)
- [我的掘金](https://juejin.im/post/5d82d3fbf265da039929a9d9)
- [适用于Linux的Java下载](https://www.java.com/en/download/linux_manual.jsp)
- [hgomez/download-java8.sh](https://gist.github.com/hgomez/9650687)

