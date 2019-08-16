
---

title: 010-cloudboot批量安装rancheros

date: 2019-03-10 18:41:55 +0800

tags: []

---
> 这是坚持技术写作计划（含翻译）的第10篇，定个小目标999，每周最少2篇。


本文主要讲解如何使用[cloudboot](http://www.idcos.com/opensource/cloudboot-open-source)简单批量安装[rancheros](https://rancher.com/rancher-os/)。


<a name="61a3ec66"></a>
## 介绍


<a name="cloudboot"></a>
### cloudboot

[cloudboot](http://www.idcos.com/opensource/cloudboot-open-source)是 [云霁科技](http://www.idcos.com/) 科技开源的一款简单易用的装机系统，类似 [cobbler](http://cobbler.github.io) ,但是功能更强大，更易用。(可参考我之前写的 [007-Cobbler批量自动化部署Windows10和Server 2019](https://juejin.im/post/5c748b2af265da2d9262ed0f) 和 [006-Cobbler批量自动化部署CentOS/Ubuntu/Windows](https://juejin.im/post/5c748ae2f265da2d84108d71))

<a name="rancheros"></a>
### rancheros

[rancheros](https://rancher.com/rancher-os/) 是 [rancher lab](https://rancher.com) 开源的一款容器操作系统，类似[coreos](https://coreos.com/),RancherOS是RancherLab设计的小巧、专用的容器操作系统，可用安装到服务器本地硬盘中，也可以部署到公有云上，或者配合DockerMachine使用。与Ubuntu和CentOS不同，RancherOS使用cloud-config.yml配置文件来管理机器的配置信息，包括系统启动时的服务、网络相关的配置信息、存储配置、容器配置等等，都可以放到配置文件中进行管理。

<a name="46679853"></a>
## 安装cloudboot

参考 [cloudboot一键部署](http://idcos.github.io/osinstall-doc/environment/%E4%B8%80%E9%94%AE%E9%83%A8%E7%BD%B2.html) 不多赘述

<a name="334359aa"></a>
### 挂载rancheros镜像

```bash
wget -P /tmp/ http://releases.rancher.com/os/latest/rancheros.iso
mkdir -p $PWD/cloudboot/deploy/iso/rancheros/1.5.1/
mount -o loop /tmp/rancheros.iso /media
rsync -a /media/ $PWD/cloudboot/deploy/iso/rancheros/1.5.1/
umount /media
```

<a name="483e5673"></a>
### 创建软连接

```bash
docker exec -it cloudboot /bin/sh
ln -s /data/iso/rancheros /home/www/rancheros
```

<a name="ba8d1dca"></a>
### 注意：

- cloudboot默认用户名密码是 admin/admin
- 登陆后需要配置dhcp(【系统管理】-> 【系统设置】)
- 需要配置网段(【网段管理】->【应用网段】)
- 本文讲的是vmware，所以不需要配置OOB
- 需要配置设备位置(【模板管理】->【位置管理】)
- 如果cloudboot和rancheros都装在vmware虚拟机里，需要把vmware的网络设置中的dhcp去掉，否则会冲突

<a name="375e9611"></a>
## pxe安装rancheros

参考 [rancheros#docs#iPXE](https://rancher.com/docs/os/v1.x/en/installation/running-rancheros/server/pxe/) 和 [cloudboot PXE模板定制规范](http://idcos.github.io/osinstall-doc/os/PXE%E6%A8%A1%E6%9D%BF%E5%AE%9A%E5%88%B6%E8%A7%84%E8%8C%83.html)

<a name="c51df3a1"></a>
### PXE模板管理

从【模板管理】->【PXE模板管理】 新增rancheros-1.5.1

```bash
DEFAULT rancheros
LABEL rancheros
KERNEL http://osinstall.idcos.com/rancheros/1.5.1/boot/vmlinuz-4.14.85-rancher
APPEND initrd=http://osinstall.idcos.com/rancheros/1.5.1/boot/initrd-v1.5.1  rancher.cloud_init.datasources=[url:http://osinstall.idcos.com/api/osinstall/v1/device/getSystemBySn?sn={sn}] rancher.autologin=tty1 rancher.autologin=ttyS0 rancher.autologin=ttyS1 rancher.autologin=ttyS1 console=tty1 console=ttyS0 console=ttyS1 printk.devkmsg=on panic=10 
IPAPPEND 2
```

<a name="b3df8665"></a>
### 系统模板管理

从【模板管理】->【系统模板管理】 新增rancheros-1.5.1

把docker  mirror换成实际的加速器，如果不需要，可以删除，[ssh_authorized_keys 换成真实的ssh key](https://rancher.com/docs/os/v1.x/en/installation/configuration/ssh-keys/)

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

<a name="95dfb265"></a>
### 自定义脚本

在cloudboot宿主机上，运行 `docker exec -it cloudboot /bin/sh` ,然后运行 `vim /home/www/scripts/rancheros.sh`

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

<a name="b67e6b7d"></a>
### 自动化安装rancheros

从vmware创建 空盘 -> 其他Linux4.x或更高版本内核64位，2核2G虚拟机，然后上电<br />虚拟机会从PXE拉取[cloudboot的bootos](http://idcos.github.io/osinstall-doc/bootos/) 安装到内存中，并且往cloudboot上注册待录入的设备（待屏幕变蓝色）

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079937-7895b3bf-3100-47c6-9d89-d1069a5a420c.png#align=left&display=inline&height=400&originHeight=400&originWidth=720&size=0&status=done&width=720)<br />从 `http://${cloudboot host}/#/dashboard/device/scan/list` 会发现新设备，选中后，点击录入新设备<br />![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079846-e0a498fc-7070-40b2-b67b-b4dc3c16b04d.png#align=left&display=inline&height=321&originHeight=321&originWidth=691&size=0&status=done&width=691)

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079934-c5031ae9-b299-4b91-bcde-90d607704755.png#align=left&display=inline&height=222&originHeight=379&originWidth=1276&size=0&status=done&width=746)

bootos会自动轮询是否有自动装机任务，所以静候即可。如果等不及，可以在录入成功后，手动重启虚拟机。<br />在【正在安装的设备】中，会自动出现要安装的设备

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079912-f3e907b7-188d-4b82-aeff-1e577a4aba40.png#align=left&display=inline&height=98&originHeight=236&originWidth=1801&size=0&status=done&width=746)

点击【详情】会在滚动模式下试试看到安装进度

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079852-3baec0a2-0627-4fe9-8233-0459d3e5c20d.png#align=left&display=inline&height=157&originHeight=157&originWidth=440&size=0&status=done&width=440)

在【设备列表】可以看到已安装成功的设备

![](https://cdn.nlark.com/yuque/0/2019/png/226273/1552641079889-990adf01-3db1-4cd2-9791-4fc4d6b36e57.png#align=left&display=inline&height=72&originHeight=176&originWidth=1826&size=0&status=done&width=746)

<a name="138a6766"></a>
### 注意

~~不知道为嘛，安装后需要重启一下虚拟机后，才能使用ssh进行连接。~~

根据rancher labs 大神腩哥指点

> booting from ISO 首次启动，整个系统都在内存中。
> 
> 执行ros install后，安装bootloader和initrd/vmlinuz到磁盘。
> 
> 再次启动后，就是完整的运行在硬盘上的操作系统。


<a name="c9665e1f"></a>
## 脑洞

其实是正规操作，可以在 cloud config 配置自定义服务，这样装机后，就可以直接启动服务，不需要ssh到ros上，手动执行命令，例如配置rancher client的添加主机的命令，这样就可以直接添加到已有集群。 更多参考 [Custom System Services](https://rancher.com/docs/os/v1.x/en/installation/system-services/custom-system-services/)

```yaml
#cloud-config
rancher:
  services:
    nginxapp:
      image: nginx
      restart: always
```

<a name="35808e79"></a>
## 参考资料

- [cloudboot一键部署](http://idcos.github.io/osinstall-doc/environment/%E4%B8%80%E9%94%AE%E9%83%A8%E7%BD%B2.html)
- [cloudboot PXE模板定制规范](http://idcos.github.io/osinstall-doc/os/PXE%E6%A8%A1%E6%9D%BF%E5%AE%9A%E5%88%B6%E8%A7%84%E8%8C%83.html)
- [rancheros#docs#iPXE](https://rancher.com/docs/os/v1.x/en/installation/running-rancheros/server/pxe/)
- [006-Cobbler批量自动化部署CentOS/Ubuntu/Windows](https://juejin.im/post/5c748ae2f265da2d84108d71)
- [007-Cobbler批量自动化部署Windows10和Server 2019](https://juejin.im/post/5c748b2af265da2d9262ed0f)


<a name="fb674066"></a>
## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.shunnengnet.com/index.php/Home/Contact/join.html) , 一起搞事情。

长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。


