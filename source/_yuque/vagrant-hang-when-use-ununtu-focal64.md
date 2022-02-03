---
title: 053-vagrant无法使用ubuntu20.04问题
urlname: vagrant-hang-when-use-ununtu-focal64
date: '2021-01-14 20:35:21 +0800'
tags:
  - vagrant
  - virtualbox
  - ubuntu
  - focal64
categories:
  - vagrant
---

> 这是坚持技术写作计划（含翻译）的第 53 篇，定个小目标 999，每周最少 2 篇。

最近使用 vagrant 安装 ubuntu20.04(ubuntu/focal64)是发现无法正常启动。

vagrant+virtualbox 系列文章



- [036-win10 搭建 python 的 linux 开发环境(pycharm+vagrant+virtualbox)](https://juejin.im/post/5d3a55ece51d454f71439dd2)
- [037-vagrant 启动(up)后自动同步文件(rsync-auto)](https://juejin.im/post/5d562b5e5188252d43756db8)
- [040-解决 Linux 使用 virtualbox 共享文件夹问题](https://juejin.im/post/5d5695056fb9a06afd6600f0)
- [042-解决 win10 VirtualBox 无法启动(VERR_NEM_VM_CREATE_FAILED)](https://juejin.im/post/5d63869a51882559c41612c6)
- [043-解决 vagrant 访问 virtualbox 共享文件夹报无权限问题(Permission denied)](https://juejin.im/post/5d6493d6e51d456206115a2c)[D)](https://juejin.im/post/5d63869a51882559c41612c6)
- [051-vagrant 无法使用 ubuntu20.04 问题](https://anjia0532.github.io/2021/01/14/vagrant-hang-when-use-ununtu-focal64/)

<!-- more -->

## 过程

```bash
# 离线下载 ubuntu/focal64 https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64-vagrant.box
# vagrant box add ubuntu/focal64 /path/to/focal-server-cloudimg-amd64-vagrant.box
```

```bash
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
    # Every Vagrant development environment requires a box. You can search for
    # boxes at https://vagrantcloud.com/search.
    config.vm.box_check_update = false
    config.vm.box = "ubuntu/xenial64"
    config.vm.hostname = "redis"
    config.vm.network "private_network", ip: "172.17.8.102"
    config.vm.provider "virtualbox" do |vb|
      vb.memory = "4096"
      vb.cpus = 2
      vb.name = "tdegine"
    end

    config.vm.synced_folder ".", "/vagrant", type: "rsync",
      rsync__verbose: true,
      rsync__exclude: ['.git*', 'node_modules*','*.log','*.box','Vagrantfile']

    config.vm.provision "shell", inline: <<-SHELL
sudo sed -i 's/archive.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list
sudo apt update
# 安装cmake
sudo apt-get install -y cmake build-essential

# 安装docker
sudo apt-get remove docker docker-engine docker.io
sudo apt-get install-y apt-transport-https ca-certificates curl gnupg2 software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo mkdir -p /etc/docker
sudo apt-get install -y docker-ce
    SHELL
  end


```

```bash
$ vagrant up

Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'ubuntu/focal64'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'ubuntu/focal64' version '20200804.0.0' is up to date...
==> default: Setting the name of the VM: jpc_default_1597236075101_37777
==> default: Clearing any previously set network interfaces...
==> default: Preparing network interfaces based on configuration...
default: Adapter 1: nat
default: Adapter 2: hostonly
==> default: Forwarding ports...
default: 22 (guest) => 2222 (host) (adapter 1)
==> default: Running 'pre-boot' VM customizations...
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
default: SSH address: 127.0.0.1:2222
default: SSH username: vagrant
default: SSH auth method: private key
```

一直卡在 `default: SSH auth method: private key`  并且 virtualbox 也会卡死，并且无法强制关闭或者释放此 vm，需要杀死进程才行。

## 解决办法

下面方法二选一就行

### 修改 vagrantfile 参数

增加 `vb.customize ["modifyvm", :id, "--uartmode1", "file", File::NULL]`

```bash
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
    # Every Vagrant development environment requires a box. You can search for
    # boxes at https://vagrantcloud.com/search.
    config.vm.box_check_update = false
    config.vm.box = "ubuntu/xenial64"
    config.vm.hostname = "redis"
    config.vm.network "private_network", ip: "172.17.8.102"
    config.vm.provider "virtualbox" do |vb|
      vb.memory = "4096"
      vb.cpus = 2
      vb.name = "tdegine"
      # 增加这行
      vb.customize ["modifyvm", :id, "--uartmode1", "file", File::NULL]
    end

    config.vm.synced_folder ".", "/vagrant", type: "rsync",
      rsync__verbose: true,
      rsync__exclude: ['.git*', 'node_modules*','*.log','*.box','Vagrantfile']

    config.vm.provision "shell", inline: <<-SHELL
sudo sed -i 's/archive.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list
sudo apt update
# 安装cmake
sudo apt-get install -y cmake build-essential

# 安装docker
sudo apt-get remove docker docker-engine docker.io
sudo apt-get install-y apt-transport-https ca-certificates curl gnupg2 software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo mkdir -p /etc/docker
sudo apt-get install -y docker-ce
    SHELL
  end


```

### 换镜像

[https://app.vagrantup.com/bento/boxes/ubuntu-20.04](https://app.vagrantup.com/bento/boxes/ubuntu-20.04)

```bash
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-20.04"
end
```

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/job_detail/20db89ac1adece6d3nZ-2tu1E1Q~.html?ka=search_list_jname_2_blank&lid=ak5J7ypLUb7.search.2) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2021/01/14/vagrant-hang-when-use-ununtu-focal64/)
- [我的掘金](https://juejin.cn/post/6917589318823821320)
- [hashicorp/vagrant#11817#SSH auth method: private key](https://github.com/hashicorp/vagrant/issues/11817)
- [Vagrant box startup timeout due to no serial port](https://bugs.launchpad.net/cloud-images/+bug/1829625)
