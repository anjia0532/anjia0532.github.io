
---

title: 036-win10搭建python的linux开发环境(pycharm+vagrant+virtualbox)

date: 2019-07-24 19:35:21 +0800

tags: [虚拟机,kvm,vagrant,virtualbox,jumpserver,堡垒机]

categories: python

---

> 这是坚持技术写作计划（含翻译）的第36篇，定个小目标999，每周最少2篇。


本文以jumpserver为例，介绍如何在windows环境下进行jumpserver开发(jumpserver依赖的一些库，只有linux环境才能用)，其实不局限于jumpserver，其他项目也适用(不限于python)。参考 [打造跨平台一致性开发环境](https://juejin.im/entry/5c6a6da5f265da2de52d7d7c/detail)

python+vagrant+virtualbox系列文章<br /> 

- [036-win10搭建python的linux开发环境(pycharm+vagrant+virtualbox)](https://juejin.im/post/5d3a55ece51d454f71439dd2) 
- [037-vagrant启动(up)后自动同步文件(rsync-auto)](https://juejin.im/post/5d562b5e5188252d43756db8) 
- [040-解决Linux使用virtualbox共享文件夹问题](https://juejin.im/post/5d5695056fb9a06afd6600f0)
- [042-解决win10 VirtualBox无法启动(VERR_NEM_VM_CREATE_FAILED)](https://juejin.im/post/5d63869a51882559c41612c6)
- [043-解决vagrant访问virtualbox共享文件夹报无权限问题(Permission denied)](https://juejin.im/post/5d6493d6e51d456206115a2c)

<!-- more -->

<a name="FBqFH"></a>
## 初始化环境
<a name="DzvF8"></a>
### 安装vagrant+virtualbox
参考 [Vagrant系列(一)----win10搭建Vagrant+VirtualBox环境](https://blog.csdn.net/u011781521/article/details/80275212)
<a name="LDM8N"></a>
### 下载centos镜像
可以使用境外服务器或者迅雷下载 `https://cloud.centos.org/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-1905_01.VirtualBox.box` 
```bash
vagrant box add centos/7 \
    https://cloud.centos.org/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-1905_01.VirtualBox.box

## 或者

vagrant box add centos/7 /path/to/CentOS-7-x86_64-Vagrant-1905_01.VirtualBox.box
```

<a name="3o5Z3"></a>
### 下载jumpserver源代码并创建vagrant
如果下载速度慢，可以使用码云(gitee.com)自己创建个镜像项目，clone码云上的jumpserver
```bash
git clone --depth=1 https://github.com/jumpserver/jumpserver.git
cd jumpserver
```
在jumpserver下创建Vagrantfile
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box_check_update = false
  config.vm.box = "centos/7"
  config.vm.hostname = "jumpserver"
  config.vm.network "private_network", ip: "172.17.8.101"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "4096"
    vb.cpus = 2
    vb.name = "jumpserver"
  end

  config.vm.synced_folder ".", "/vagrant", type: "rsync",
    rsync__verbose: true,
    rsync__exclude: ['.git*', 'node_modules*','*.log','*.box','Vagrantfile']

  config.vm.provision "shell", inline: <<-SHELL
sudo sed -e "/mirrorlist/d" -e "s/#baseurl/baseurl/g" -e "s/mirror\.centos\.org/mirrors\.tuna\.tsinghua\.edu\.cn/g" -i /etc/yum.repos.d/CentOS-Base.repo
sudo yum makecache
sudo yum install -y epel-release

sudo yum install -y python36 python36-devel python36-pip \
		 libtiff-devel libjpeg-devel libzip-devel freetype-devel \
     lcms2-devel libwebp-devel tcl-devel tk-devel sshpass \
     openldap-devel mariadb-devel mysql-devel libffi-devel \
     openssh-clients telnet openldap-clients gcc

mkdir /home/vagrant/.pip
cat << EOF | sudo tee -a /home/vagrant/.pip/pip.conf
[global]
timeout = 6000
index-url = https://mirrors.aliyun.com/pypi/simple/

[install]
use-mirrors = true
mirrors = https://mirrors.aliyun.com/pypi/simple/
trusted-host=mirrors.aliyun.com
EOF

python3.6 -m venv /home/vagrant/venv
source /home/vagrant/venv/bin/activate
echo "source /home/vagrant/venv/bin/activate" >> /home/vagrant/.bash_profile
  SHELL
end

```

```bash
cd jumpserver
vagrant up
vagrant ssh
```

<a name="gn90K"></a>
### ~~安装python3.6及配置pip源~~
如果使用我的Vargrantfile，已经自动配置阿里云源和清华源并且安装必要依赖包了，不需要重复配置
```bash
## 设置yum的清华源
sudo sed -e "/mirrorlist/d" -e "s/#baseurl/baseurl/g" -e "s/mirror\.centos\.org/mirrors\.tuna\.tsinghua\.edu\.cn/g" -i /etc/yum.repos.d/CentOS-Base.repo
sudo yum makecache
sudo yum install -y epel-release

## 安装依赖包
sudo yum install -y python36 python36-devel python36-pip \
		 libtiff-devel libjpeg-devel libzip-devel freetype-devel \
     lcms2-devel libwebp-devel tcl-devel tk-devel sshpass \
     openldap-devel mariadb-devel mysql-devel libffi-devel \
     openssh-clients telnet openldap-clients gcc

## 配置pip阿里云源
mkdir ~/.pip
cat << EOF | sudo tee -a ~/.pip/pip.conf
[global]
timeout = 6000
index-url = https://mirrors.aliyun.com/pypi/simple/

[install]
use-mirrors = true
mirrors = https://mirrors.aliyun.com/pypi/simple/
trusted-host=mirrors.aliyun.com
EOF
```

<a name="YFYr7"></a>
### 安装依赖
如果使用我的Vargrantfile，不需要配置python env了，已经配置好了
```bash
vagrant ssh
python3.6 -m venv /home/vagrant/venv
source /home/vagrant/venv/bin/activate
```

只需要执行这个即可
```bash
pip3 install -r /vagrant/requirements/requirements.txt
```

参考 [安装文档](http://docs.jumpserver.org/zh/docs/step_by_step.html)

<a name="KRTaO"></a>
## 配置pycharm并启动jumpserver
<a name="Mk0Vl"></a>
### 配置pycharm

<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>S</kbd>打开设置

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564102215460-c6a602ed-86e6-49fc-b2b1-d43689243daa.png#align=left&display=inline&height=706&name=image.png&originHeight=706&originWidth=1009&size=70204&status=done&width=1009)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564102389714-c0a5c982-1597-4007-8719-c4f316457bf8.png#align=left&display=inline&height=678&name=image.png&originHeight=678&originWidth=841&size=65908&status=done&width=841)<br />如果正常，会显示已经安装的pip包，如果是空，或者只有两个，那意味着python的env设置的不对。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564102637247-a8c69ed0-d6a2-4d00-ad3b-feebe98a0589.png#align=left&display=inline&height=674&name=image.png&originHeight=674&originWidth=1007&size=68920&status=done&width=1007)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564102989073-8062df66-c4cc-45dd-b574-db6e6a39cb6d.png#align=left&display=inline&height=739&name=image.png&originHeight=739&originWidth=1068&size=108292&status=done&width=1068)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564110989520-3f8ac63d-5b58-46b7-89bc-d24f6352785c.png#align=left&display=inline&height=706&name=image.png&originHeight=706&originWidth=1009&size=63412&status=done&width=1009)

[使用 PyCharm专业版和vagrant进行同步开发](https://blog.csdn.net/weixin_42393089/article/details/83211456)

<a name="TVFTG"></a>
### 修改配置并初始化数据库

```bash
cd jumpserver
cp config_example.yml config.yml
## 修改config.yml的相关配置
vagrant ssh
source /home/vagrant/venv/bin/activate
cd /vagrant/
./utils/make_migrations.sh
```

<a name="avGGO"></a>
### 启动项目
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564103536157-efbb24a4-c29d-45b4-991f-b314544ed548.png#align=left&display=inline&height=79&name=image.png&originHeight=79&originWidth=353&size=5532&status=done&width=353)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564103574288-28ffdb6f-25f2-4a67-b8ba-936cca59ccf3.png#align=left&display=inline&height=273&name=image.png&originHeight=273&originWidth=407&size=25404&status=done&width=407)<br />浏览器打开 [http://172.17.8.101:8080/auth/login/?next=/](http://172.17.8.101:8080/auth/login/?next=/)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564103612991-53990854-279f-41c9-8250-6647b7889279.png#align=left&display=inline&height=368&name=image.png&originHeight=368&originWidth=889&size=55638&status=done&width=889)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1564103694901-b41968af-0830-4fc3-beab-4a2393a76279.png#align=left&display=inline&height=240&name=image.png&originHeight=240&originWidth=596&size=56566&status=done&width=596)


<a name="Kgbm8"></a>
## 其他
已经提交PR，如果有需要的朋友， 可以在PR上投票，可以提高通过率[added Vagrantfile to support windows dev#3036](https://github.com/jumpserver/jumpserver/pull/3036) （目前已合并到jumpserver repo中）

jumpserver virtualbox 已上传百度云盘 <br />链接：[https://pan.baidu.com/s/1mr6xM7UVkPJy3TPTyoM_NQ](https://pan.baidu.com/s/1mr6xM7UVkPJy3TPTyoM_NQ)  密码：rci6

<a name="Xjcpr"></a>
## 2019-08-28 更新
如果遇到ansible无法执行的问题是因为上述方案只起了django 的server，没有启动celery

```bash
cd /path/to/host/jumpserver/
vagrant ssh
# 分别启动 celery和beat
/vagrant/jms start celery
/vagrant/jms start beat
```

至于为啥不直接 `jms start all` 而是使用pycharm启动server，因为可以debug jumpserver 。而用 `jms start all` 则不可以

<a name="sNYV4"></a>
## 参考资料

- [我的博客](http://anjia0532.github.io/2019/07/24/jumpserver-vagrant-virtualbox)
- [我的掘金](https://juejin.im/post/5d3a55ece51d454f71439dd2)
- [Win10 10月更新 VirtualBox VT-x is not available (VERR_VMX_NO_VMX). 解决](https://blog.csdn.net/imilano/article/details/83038682)
- [Vagrant系列(一)----win10搭建Vagrant+VirtualBox环境](https://blog.csdn.net/u011781521/article/details/80275212)
- [安装文档](http://docs.jumpserver.org/zh/docs/step_by_step.html)
- [使用 PyCharm专业版和vagrant进行同步开发](https://blog.csdn.net/weixin_42393089/article/details/83211456)
- [打造跨平台一致性开发环境](https://juejin.im/entry/5c6a6da5f265da2de52d7d7c/detail)


