---
title: 010-cloudboot批量安装rancheros
urlname: cloudboot-ros
date: 2019-03-10 18:41:55 +0800
tags: []
categories: []
---

> 这是坚持技术写作计划（含翻译）的第 10 篇，定个小目标 999，每周最少 2 篇。

本文主要讲解如何使用[cloudboot](http://www.idcos.com/opensource/cloudboot-open-source)简单批量安装[rancheros](https://rancher.com/rancher-os/)。

## 介绍

### cloudboot

[cloudboot](http://www.idcos.com/opensource/cloudboot-open-source)是  [云霁科技](http://www.idcos.com/) 科技开源的一款简单易用的装机系统，类似 [cobbler](http://cobbler.github.io) ,但是功能更强大，更易用。(可参考我之前写的  [007-Cobbler 批量自动化部署 Windows10 和 Server 2019](https://juejin.im/post/5c748b2af265da2d9262ed0f)  和 [006-Cobbler 批量自动化部署 CentOS/Ubuntu/Windows](https://juejin.im/post/5c748ae2f265da2d84108d71))

### rancheros

[rancheros](https://rancher.com/rancher-os/) 是 [rancher lab](https://rancher.com)  开源的一款容器操作系统，类似[coreos](https://coreos.com/),RancherOS 是 RancherLab 设计的小巧、专用的容器操作系统，可用安装到服务器本地硬盘中，也可以部署到公有云上，或者配合 DockerMachine 使用。与 Ubuntu 和 CentOS 不同，RancherOS 使用 cloud-config.yml 配置文件来管理机器的配置信息，包括系统启动时的服务、网络相关的配置信息、存储配置、容器配置等等，都可以放到配置文件中进行管理。

## 安装 cloudboot

参考  [cloudboot 一键部署](http://idcos.github.io/osinstall-doc/environment/%E4%B8%80%E9%94%AE%E9%83%A8%E7%BD%B2.html)  不多赘述

### 挂载 rancheros 镜像

```bash
wget -P /tmp/ http://releases.rancher.com/os/latest/rancheros.iso
mkdir -p $PWD/cloudboot/deploy/iso/rancheros/1.5.1/
mount -o loop /tmp/rancheros.iso /media
rsync -a /media/ $PWD/cloudboot/deploy/iso/rancheros/1.5.1/
umount /media
```

### 创建软连接

```bash
docker exec -it cloudboot /bin/sh
ln -s /data/iso/rancheros /home/www/rancheros
```

### 注意：

- cloudboot 默认用户名密码是 admin/admin
- 登陆后需要配置 dhcp(【系统管理】-> 【系统设置】)
- 需要配置网段(【网段管理】->【应用网段】)
- 本文讲的是 vmware，所以不需要配置 OOB
- 需要配置设备位置(【模板管理】->【位置管理】)
- 如果 cloudboot 和 rancheros 都装在 vmware 虚拟机里，需要把 vmware 的网络设置中的 dhcp 去掉，否则会冲突

## pxe 安装 rancheros

参考 [rancheros#docs#iPXE](https://rancher.com/docs/os/v1.x/en/installation/running-rancheros/server/pxe/)  和  [cloudboot PXE 模板定制规范](http://idcos.github.io/osinstall-doc/os/PXE%E6%A8%A1%E6%9D%BF%E5%AE%9A%E5%88%B6%E8%A7%84%E8%8C%83.html)

### PXE 模板管理

从【模板管理】->【PXE 模板管理】 新增 rancheros-1.5.1

```bash
DEFAULT rancheros
LABEL rancheros
KERNEL http://osinstall.idcos.com/rancheros/1.5.1/boot/vmlinuz-4.14.85-rancher
APPEND initrd=http://osinstall.idcos.com/rancheros/1.5.1/boot/initrd-v1.5.1  rancher.cloud_init.datasources=[url:http://osinstall.idcos.com/api/osinstall/v1/device/getSystemBySn?sn={sn}] rancher.autologin=tty1 rancher.autologin=ttyS0 rancher.autologin=ttyS1 rancher.autologin=ttyS1 console=tty1 console=ttyS0 console=ttyS1 printk.devkmsg=on panic=10
IPAPPEND 2
```

### 系统模板管理

从【模板管理】->【系统模板管理】 新增 rancheros-1.5.1

把 docker  mirror 换成实际的加速器，如果不需要，可以删除，[ssh_authorized_keys 换成真实的 ssh key](https://rancher.com/docs/os/v1.x/en/installation/configuration/ssh-keys/)

```yaml
#cloud-config
rancher:
  console: alpine
  docker:
    registry_mirror: "https://xxx.mirror.aliyuncs.com"
runcmd:
  - sh -c 'curl http://osinstall.idcos.com/scripts/rancheros.sh | bash'
ssh_authorized_keys:
  - ssh-rsa AAAA....ZZZZ user@user
```

### 自定义脚本

在 cloudboot 宿主机上，运行 `docker exec -it cloudboot /bin/sh` ,然后运行 `vim /home/www/scripts/rancheros.sh`

```bash
#!/bin/bash
progress() {
    curl -H "Content-Type: application/json" -X POST -d "{\"Sn\":\"$_sn\",\"Title\":\"$1\",\"InstallProgress\":$2,\"InstallLog\":\"$3\"}" http://osinstall.idcos.com/api/osinstall/v1/report/deviceInstallInfo
}

_sn=$(sed q /sys/class/net/eth0/address)

progress "配置主机名和网络" 0.7 "6YWN572u5Li75py65ZCN5ZKM572R57uc"

# config network
curl -o /tmp/networkinfo "http://osinstall.idcos.com/api/osinstall/v1/device/getNetworkBySn?sn=${_sn}&type=raw"
source /tmp/networkinfo

cat > /etc/network/interfaces <<EOF
auto lo
iface lo inet loopback
auto eth0
iface eth0 inet static
address $IPADDR
netmask $NETMASK
gateway $GATEWAY
EOF

echo "$HOSTNAME" > /etc/hostname
sudo hostname "$HOSTNAME"

progress "配置alpine镜像源" 0.8 "6YWN572uYWxwaW5l6ZWc5YOP5rqQ"
sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories

progress "安装完成" 1 "5a6J6KOF5a6M5oiQ"
sudo ros install -c http://osinstall.idcos.com/api/osinstall/v1/device/getSystemBySn?sn=$_sn" -d /dev/sda -f
```

### 自动化安装 rancheros

从 vmware 创建 空盘 -> 其他 Linux4.x 或更高版本内核 64 位，2 核 2G 虚拟机，然后上电
虚拟机会从 PXE 拉取[cloudboot 的 bootos](http://idcos.github.io/osinstall-doc/bootos/)  安装到内存中，并且往 cloudboot 上注册待录入的设备（待屏幕变蓝色）

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079937-7895b3bf-3100-47c6-9d89-d1069a5a420c.png#align=left&display=inline&height=400&originHeight=400&originWidth=720&size=0&status=done&width=720)
从 `http://${cloudboot host}/#/dashboard/device/scan/list`  会发现新设备，选中后，点击录入新设备
![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079846-e0a498fc-7070-40b2-b67b-b4dc3c16b04d.png#align=left&display=inline&height=321&originHeight=321&originWidth=691&size=0&status=done&width=691)

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079934-c5031ae9-b299-4b91-bcde-90d607704755.png#align=left&display=inline&height=222&originHeight=379&originWidth=1276&size=0&status=done&width=746)

bootos 会自动轮询是否有自动装机任务，所以静候即可。如果等不及，可以在录入成功后，手动重启虚拟机。
在【正在安装的设备】中，会自动出现要安装的设备

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079912-f3e907b7-188d-4b82-aeff-1e577a4aba40.png#align=left&display=inline&height=98&originHeight=236&originWidth=1801&size=0&status=done&width=746)

点击【详情】会在滚动模式下试试看到安装进度

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079852-3baec0a2-0627-4fe9-8233-0459d3e5c20d.png#align=left&display=inline&height=157&originHeight=157&originWidth=440&size=0&status=done&width=440)

在【设备列表】可以看到已安装成功的设备

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079889-990adf01-3db1-4cd2-9791-4fc4d6b36e57.png#align=left&display=inline&height=72&originHeight=176&originWidth=1826&size=0&status=done&width=746)

### 注意

~~不知道为嘛，安装后需要重启一下虚拟机后，才能使用 ssh 进行连接。~~

根据 rancher labs 大神腩哥指点

> booting from ISO 首次启动，整个系统都在内存中。
>
> 执行 ros install 后，安装 bootloader 和 initrd/vmlinuz 到磁盘。
>
> 再次启动后，就是完整的运行在硬盘上的操作系统。

## 脑洞

其实是正规操作，可以在 cloud config 配置自定义服务，这样装机后，就可以直接启动服务，不需要 ssh 到 ros 上，手动执行命令，例如配置 rancher client 的添加主机的命令，这样就可以直接添加到已有集群。 更多参考 [Custom System Services](https://rancher.com/docs/os/v1.x/en/installation/system-services/custom-system-services/)

```yaml
#cloud-config
rancher:
  services:
    nginxapp:
      image: nginx
      restart: always
```

## 参考资料

- [cloudboot 一键部署](http://idcos.github.io/osinstall-doc/environment/%E4%B8%80%E9%94%AE%E9%83%A8%E7%BD%B2.html)
- [cloudboot PXE 模板定制规范](http://idcos.github.io/osinstall-doc/os/PXE%E6%A8%A1%E6%9D%BF%E5%AE%9A%E5%88%B6%E8%A7%84%E8%8C%83.html)
- [rancheros#docs#iPXE](https://rancher.com/docs/os/v1.x/en/installation/running-rancheros/server/pxe/)
- [006-Cobbler 批量自动化部署 CentOS/Ubuntu/Windows](https://juejin.im/post/5c748ae2f265da2d84108d71)
- [007-Cobbler 批量自动化部署 Windows10 和 Server 2019](https://juejin.im/post/5c748b2af265da2d9262ed0f)

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。
