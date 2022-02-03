---
title: 040-解决Linux使用virtualbox共享文件夹问题
urlname: linux-virtualbox-shared-folder
date: '2019-08-16 18:01:57 +0800'
tags: []
categories: []
---

date: 2019-08-16 19:05:03
tags: [虚拟机,kvm,vagrant,virtualbox,python]
categories: python

---

> 这是坚持技术写作计划（含翻译）的第 40 篇，定个小目标 999，每周最少 2 篇。

本文主要介绍，在使用 virtualbox 时，如何共享文件夹

- rsync  是单向(宿主机修改了，定时同步到虚拟机里，但是虚拟机修改的不会对宿主造成影响)
- nfs  官方文档说 `Windows users: NFS folders do not work on Windows hosts. Vagrant will ignore your request for NFS synced folders on Windows.`  而且需要下载插件，新手十有八九会被坑
- smb  兼容性比较好，支持 mac,linux,windows 访问(虚拟机),宿主机只限 mac 和 win，但是 win 需要管理员权限，mac 下操作挺复杂，还得进行配置，防止自动超时
- VirtualBox  综合来看，virtualbox 不错，当然，如果文件量太多的话，也有性能问题,意思是别想着用来构建前端项目(一个  `node_modules`  搞死你啊)，可以结合 rsync 使用，rsync 可以设置排除目录，然后定时同步到虚拟机，需要双向的，再把文件复制到挂载为 virtualbox 的目录下，宿主机就可以访问了。

python+vagrant+virtualbox 系列文章



- [036-win10 搭建 python 的 linux 开发环境(pycharm+vagrant+virtualbox)](https://juejin.im/post/5d3a55ece51d454f71439dd2)
- [037-vagrant 启动(up)后自动同步文件(rsync-auto)](https://juejin.im/post/5d562b5e5188252d43756db8)
- [040-解决 Linux 使用 virtualbox 共享文件夹问题](https://juejin.im/post/5d5695056fb9a06afd6600f0)
- [042-解决 win10 VirtualBox 无法启动(VERR_NEM_VM_CREATE_FAILED)](https://juejin.im/post/5d63869a51882559c41612c6)
- [043-解决 vagrant 访问 virtualbox 共享文件夹报无权限问题(Permission denied)](https://juejin.im/post/5d6493d6e51d456206115a2c)

<!-- more -->

```bash
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box_check_update = false
  config.vm.box = "centos/7"
  config.vm.hostname = "ansible"
  config.vm.network "private_network", ip: "172.17.8.102"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "4096"
    vb.cpus = 2
    vb.name = config.vm.hostname
  end
  ## 单向同步
  config.vm.synced_folder ".", "/vagrant", type: "rsync",
    rsync__verbose: true,
    rsync__auto: true,
    rsync__exclude: ['.git*', 'node_modules*','*.log','*.box','Vagrantfile']
    config.trigger.after :up do |t|
      t.info = "rsync auto"
      t.run = {inline: "vagrant rsync-auto"}
    end

  config.vm.provision "shell", inline: <<-SHELL
## 配置xshell等可以使用密码登录
sed -e "s/#PasswordAuthentication yes/PasswordAuthentication yes/g" -e "s/PasswordAuthentication no/PasswordAuthentication yes/g" -i  /etc/ssh/sshd_config
service sshd restart

## 设置yum的清华源（阿里云源不稳定）
sudo sed -e "/mirrorlist/d" -e "s/#baseurl/baseurl/g" -e "s/mirror\.centos\.org/mirrors\.tuna\.tsinghua\.edu\.cn/g" -i /etc/yum.repos.d/CentOS-Base.repo
sudo yum makecache
sudo yum install -y epel-release

## 安装virtualbox需要kernel-headers
yum install -y gcc make kernel-headers-$(uname -r) kernel-devel-$( uname -r)

## 可以使用rsync同步目录，不用每次都联网下载
curl -O http://download.virtualbox.org/virtualbox/6.0.10/VBoxGuestAdditions_6.0.10.iso
sudo mkdir /media/VBoxGuestAdditions
sudo mount -o loop,ro VBoxGuestAdditions_6.0.10.iso /media/VBoxGuestAdditions
sudo sh /media/VBoxGuestAdditions/VBoxLinuxAdditions.run
rm VBoxGuestAdditions_6.0.10.iso
sudo umount /media/VBoxGuestAdditions
sudo rmdir /media/VBoxGuestAdditions
  SHELL
end
```

![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1565955156510-abc85d92-1f70-4e0e-b8c1-f4eabf080a76.png#align=left&display=inline&height=519&name=image.png&originHeight=519&originWidth=850&size=64745&status=done&width=850)
控制台输出如下所示，即为挂载成功。
![image.png](https://cdn.nlark.com/yuque/0/2019/png/226273/1565955232571-19356d03-5e24-47f2-99a9-d861b94c05a9.png#align=left&display=inline&height=44&name=image.png&originHeight=44&originWidth=875&size=11098&status=done&width=875)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2019/08/16/linux-virtualbox-shared-folder)
- [我的掘金](https://juejin.im/post/5d5695056fb9a06afd6600f0)
- [larsar/shared_folder_centos_virtualbox.txt](https://gist.github.com/larsar/1687725)
- [Synced Folders](https://www.vagrantup.com/docs/synced-folders/)
- [How to sync folder with Vagrant in Windows?](https://stackoverflow.com/a/44171136)
- [Automatically download and install VirtualBox guest additions in Vagrant](https://blog.csanchez.org/2012/05/03/automatically-download-and-install-virtualbox-guest-additions-in-vagrant/)
